---
name: graphql-design
description: Designing a GraphQL API — schema shape, resolver patterns, subscriptions, batching N+1 away with DataLoader, and typed errors
---

# Designing a GraphQL API

## Shaping the schema

```graphql
type Query {
  user(id: ID!): User
  users(filter: UserFilter, first: Int = 20, after: String): UserConnection!
}

type Mutation {
  createUser(input: CreateUserInput!): CreateUserPayload!
  updateUser(id: ID!, input: UpdateUserInput!): UpdateUserPayload!
}

type Subscription {
  orderStatusChanged(orderId: ID!): Order!
}

type User {
  id: ID!
  email: String!
  name: String!
  orders(first: Int = 10, after: String): OrderConnection!
  createdAt: DateTime!
}

input CreateUserInput {
  email: String!
  name: String!
}

type CreateUserPayload {
  user: User
  errors: [UserError!]!
}

type UserError {
  field: String!
  message: String!
}

type UserConnection {
  edges: [UserEdge!]!
  pageInfo: PageInfo!
  totalCount: Int!
}

type UserEdge {
  node: User!
  cursor: String!
}

type PageInfo {
  hasNextPage: Boolean!
  endCursor: String
}
```

Paginate with Relay-style connections, and have every mutation return a payload type that carries both the result and any errors.

## Writing resolvers

```typescript
const resolvers: Resolvers = {
  Query: {
    user: async (_, { id }, ctx) => {
      return ctx.dataloaders.user.load(id);
    },
    users: async (_, { filter, first, after }, ctx) => {
      const cursor = after ? decodeCursor(after) : undefined;
      const users = await ctx.db.user.findMany({
        where: buildFilter(filter),
        take: first + 1,
        cursor: cursor ? { id: cursor } : undefined,
        orderBy: { createdAt: "desc" },
      });

      const hasNextPage = users.length > first;
      const edges = users.slice(0, first).map(user => ({
        node: user,
        cursor: encodeCursor(user.id),
      }));

      return {
        edges,
        pageInfo: {
          hasNextPage,
          endCursor: edges[edges.length - 1]?.cursor ?? null,
        },
      };
    },
  },

  Mutation: {
    createUser: async (_, { input }, ctx) => {
      const existing = await ctx.db.user.findUnique({ where: { email: input.email } });
      if (existing) {
        return { user: null, errors: [{ field: "email", message: "Already taken" }] };
      }
      const user = await ctx.db.user.create({ data: input });
      return { user, errors: [] };
    },
  },

  User: {
    orders: async (parent, { first, after }, ctx) => {
      return ctx.dataloaders.userOrders.load({ userId: parent.id, first, after });
    },
  },
};
```

## Batching with DataLoader to kill N+1

```typescript
import DataLoader from "dataloader";

function createLoaders(db: Database) {
  return {
    user: new DataLoader<string, User>(async (ids) => {
      const users = await db.user.findMany({ where: { id: { in: [...ids] } } });
      const userMap = new Map(users.map(u => [u.id, u]));
      return ids.map(id => userMap.get(id) ?? new Error(`User ${id} not found`));
    }),

    userOrders: new DataLoader<{ userId: string }, Order[]>(async (keys) => {
      const userIds = keys.map(k => k.userId);
      const orders = await db.order.findMany({
        where: { userId: { in: userIds } },
        orderBy: { createdAt: "desc" },
      });
      const grouped = new Map<string, Order[]>();
      orders.forEach(o => {
        const list = grouped.get(o.userId) ?? [];
        list.push(o);
        grouped.set(o.userId, list);
      });
      return keys.map(k => grouped.get(k.userId) ?? []);
    }),
  };
}
```

Build a fresh set of loaders for each request, or one user's cached data leaks into the next.

## Subscriptions

```typescript
const pubsub = new PubSub();

const resolvers = {
  Subscription: {
    orderStatusChanged: {
      subscribe: (_, { orderId }) => {
        return pubsub.asyncIterableIterator(`ORDER_STATUS_${orderId}`);
      },
    },
  },
  Mutation: {
    updateOrderStatus: async (_, { id, status }, ctx) => {
      const order = await ctx.db.order.update({ where: { id }, data: { status } });
      await pubsub.publish(`ORDER_STATUS_${id}`, { orderStatusChanged: order });
      return { order, errors: [] };
    },
  },
};
```

## What to avoid

- Mirroring your database tables straight into the GraphQL schema
- Resolving nested fields without DataLoader and re-introducing N+1
- Offset pagination on large datasets where a cursor belongs
- Throwing raw errors out of resolvers instead of returning typed error payloads
- One giant schema file rather than modular, per-domain type definitions
- Unbounded queries with no depth or complexity ceiling

## Before you ship

- [ ] Every list field uses Relay-style cursor pagination
- [ ] Batched entity lookups go through DataLoader
- [ ] Mutations return a payload type with both result and error fields
- [ ] Mutation arguments are wrapped in input types
- [ ] Query depth and complexity limits are configured
- [ ] Loaders are constructed per request in the context
- [ ] The schema is split into domain-specific modules
- [ ] Subscriptions publish to filtered topics instead of broadcasting to everyone
