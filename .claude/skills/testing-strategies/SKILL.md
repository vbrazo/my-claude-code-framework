---
name: testing-strategies
description: Beyond unit tests — contract, snapshot, property-based, and container-backed integration testing, plus how to organize it all
---

# Testing strategies

## Structure: arrange, act, assert

```typescript
describe("OrderService", () => {
  describe("createOrder", () => {
    it("creates an order with valid items and returns order ID", async () => {
      const repo = new InMemoryOrderRepository();
      const service = new OrderService(repo);
      const input = { customerId: "c1", items: [{ productId: "p1", quantity: 2 }] };

      const result = await service.createOrder(input);

      expect(result.id).toBeDefined();
      expect(result.status).toBe("pending");
      expect(result.items).toHaveLength(1);
      const saved = await repo.findById(result.id);
      expect(saved).toEqual(result);
    });

    it("rejects order with empty items", async () => {
      const service = new OrderService(new InMemoryOrderRepository());

      await expect(
        service.createOrder({ customerId: "c1", items: [] })
      ).rejects.toThrow("Order must have at least one item");
    });
  });
});
```

Name a test after the behavior it pins down, not the method it calls, and keep each one independent and self-contained.

## Contract testing with Pact

```typescript
import { PactV4 } from "@pact-foundation/pact";

const provider = new PactV4({
  consumer: "OrderService",
  provider: "UserService",
});

describe("UserService contract", () => {
  it("returns user by ID", async () => {
    await provider
      .addInteraction()
      .given("user with id user-1 exists")
      .uponReceiving("a request for user user-1")
      .withRequest("GET", "/api/users/user-1")
      .willRespondWith(200, (builder) => {
        builder.jsonBody({
          id: "user-1",
          name: "Alice",
          email: "alice@example.com",
        });
      })
      .executeTest(async (mockServer) => {
        const client = new UserClient(mockServer.url);
        const user = await client.getUser("user-1");
        expect(user.name).toBe("Alice");
      });
  });
});
```

Contract tests confirm that what a consumer expects lines up with what a provider actually delivers — without standing up both services at once.

## Snapshot testing

```typescript
import { render } from "@testing-library/react";

it("renders the user profile card", () => {
  const { container } = render(
    <UserCard user={{ name: "Alice", email: "alice@example.com", role: "admin" }} />
  );

  expect(container).toMatchSnapshot();
});

it("renders the order summary with inline snapshot", () => {
  const summary = formatOrderSummary(mockOrder);

  expect(summary).toMatchInlineSnapshot(`
    "Order #123
    Items: 3
    Total: $45.99
    Status: Pending"
  `);
});
```

Keep inline snapshots for small outputs, and actually read the snapshot diffs in review instead of rubber-stamping them.

## Property-based testing

```typescript
import fc from "fast-check";

describe("sortUsers", () => {
  it("always returns the same number of elements", () => {
    fc.assert(
      fc.property(
        fc.array(fc.record({ name: fc.string(), age: fc.nat(120) })),
        (users) => {
          const sorted = sortUsers(users, "name");
          return sorted.length === users.length;
        }
      )
    );
  });

  it("produces a sorted result for any input", () => {
    fc.assert(
      fc.property(
        fc.array(fc.record({ name: fc.string(), age: fc.nat(120) })),
        (users) => {
          const sorted = sortUsers(users, "age");
          for (let i = 1; i < sorted.length; i++) {
            if (sorted[i].age < sorted[i - 1].age) return false;
          }
          return true;
        }
      )
    );
  });
});
```

## Integration tests on real containers

```typescript
import { PostgreSqlContainer } from "@testcontainers/postgresql";

let container: any;
let db: Database;

beforeAll(async () => {
  container = await new PostgreSqlContainer("postgres:16").start();
  db = await createDatabase(container.getConnectionUri());
  await db.migrate();
}, 60000);

afterAll(async () => {
  await db.close();
  await container.stop();
});

it("creates and retrieves a user", async () => {
  const user = await db.user.create({ name: "Alice", email: "alice@test.com" });
  const found = await db.user.findById(user.id);
  expect(found).toEqual(user);
});
```

## Test doubles

```typescript
function createMockEmailService(): EmailService {
  const sent: Array<{ to: string; subject: string }> = [];
  return {
    send: async (to, subject, body) => { sent.push({ to, subject }); },
    getSent: () => sent,
  };
}

const emailService = createMockEmailService();
const service = new NotificationService(emailService);
await service.notifyUser("user-1", "Welcome");
expect(emailService.getSent()).toHaveLength(1);
expect(emailService.getSent()[0].subject).toBe("Welcome");
```

## What to avoid

- Asserting on implementation details rather than behavior
- Carrying mutable state between tests with no `beforeEach` reset
- Tests that only pass in a particular execution order
- Mocking everything when an integration test wants real dependencies
- Tolerating flaky tests instead of chasing the root cause
- Covering trivial getters and setters while the edge cases go untested

## Before you ship

- [ ] Tests are grouped by behavior, not by method or file
- [ ] Every test follows arrange-act-assert
- [ ] Contract tests guard cross-service API compatibility
- [ ] Snapshots are reviewed, not blindly regenerated
- [ ] Property-based tests cover the invariants in algorithmic code
- [ ] Integration tests run against real dependencies in containers
- [ ] Test doubles stay minimal and behavior-focused
- [ ] CI flags flaky tests instead of passing them
