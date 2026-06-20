---
name: microservices-design
description: Designing a microservice system — service boundaries, event-driven flows, the saga pattern, API gateways, and health checks
---

# Designing microservices

## Drawing service boundaries

Carve services around business capabilities rather than technical layers. Each one owns its data store outright and talks to the rest through a clear API contract.

```
order-service/       -> owns orders table, publishes OrderCreated events
inventory-service/   -> owns inventory table, subscribes to OrderCreated
payment-service/     -> owns payments table, handles payment processing
notification-service -> stateless, subscribes to events, sends emails/SMS
```

## Communicating through events

```typescript
interface DomainEvent {
  eventId: string;
  eventType: string;
  aggregateId: string;
  timestamp: string;
  version: number;
  payload: Record<string, unknown>;
}

const orderCreatedEvent: DomainEvent = {
  eventId: crypto.randomUUID(),
  eventType: "order.created",
  aggregateId: orderId,
  timestamp: new Date().toISOString(),
  version: 1,
  payload: { customerId, items, totalAmount },
};

await broker.publish("orders", orderCreatedEvent);
```

```typescript
async function handleOrderCreated(event: DomainEvent) {
  const { items } = event.payload as OrderPayload;

  for (const item of items) {
    await db.inventory.update({
      where: { productId: item.productId },
      data: { quantity: { decrement: item.quantity } },
    });
  }

  await markEventProcessed(event.eventId);
}
```

Lean on an idempotency key (`eventId`) so a redelivered event can be processed twice without harm.

## The saga pattern (orchestrated)

```typescript
class OrderSaga {
  private steps: SagaStep[] = [
    {
      name: "reserveInventory",
      execute: (ctx) => inventoryService.reserve(ctx.items),
      compensate: (ctx) => inventoryService.release(ctx.items),
    },
    {
      name: "processPayment",
      execute: (ctx) => paymentService.charge(ctx.customerId, ctx.amount),
      compensate: (ctx) => paymentService.refund(ctx.paymentId),
    },
    {
      name: "confirmOrder",
      execute: (ctx) => orderService.confirm(ctx.orderId),
      compensate: (ctx) => orderService.cancel(ctx.orderId),
    },
  ];

  async run(context: SagaContext): Promise<void> {
    const completed: SagaStep[] = [];

    for (const step of this.steps) {
      try {
        const result = await step.execute(context);
        Object.assign(context, result);
        completed.push(step);
      } catch (error) {
        for (const s of completed.reverse()) {
          await s.compensate(context);
        }
        throw new SagaFailedError(step.name, error);
      }
    }
  }
}
```

## Fronting it with an API gateway

```yaml
# Kong or similar gateway config
services:
  - name: orders
    url: http://order-service:3000
    routes:
      - paths: ["/api/v1/orders"]
        methods: [GET, POST]
    plugins:
      - name: rate-limiting
        config:
          minute: 100
      - name: jwt
      - name: correlation-id

  - name: users
    url: http://user-service:3000
    routes:
      - paths: ["/api/v1/users"]
    plugins:
      - name: rate-limiting
        config:
          minute: 200
```

## Health checks

```typescript
app.get("/health", async (req, res) => {
  const checks = {
    database: await checkDatabase(),
    cache: await checkRedis(),
    broker: await checkMessageBroker(),
  };

  const healthy = Object.values(checks).every(c => c.status === "up");

  res.status(healthy ? 200 : 503).json({
    status: healthy ? "healthy" : "degraded",
    checks,
    version: process.env.APP_VERSION,
    uptime: process.uptime(),
  });
});
```

## What to avoid

- One database shared across services, quietly coupling them together
- Long synchronous HTTP chains that turn one outage into a cascade
- A distributed monolith where nothing can deploy on its own
- Inter-service calls with no circuit breaker in front
- Event handlers that aren't idempotent
- Distributed transactions where a saga belongs

## Before you ship

- [ ] Every service owns its own data store
- [ ] Async workflows flow through events
- [ ] Multi-service transactions use sagas with compensation
- [ ] Circuit breakers guard against cascading failures
- [ ] The API gateway handles routing, rate limiting, and auth
- [ ] Health endpoints report the status of dependencies
- [ ] Event handlers are idempotent and safe to replay
- [ ] Services deploy and scale independently
