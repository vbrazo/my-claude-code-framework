---
name: redis-patterns
description: Practical Redis recipes — cache-aside, sliding-window rate limits, pub/sub, durable streams, atomic Lua scripts, and the data structures behind them
---

# Working with Redis

## Caching: read-through and cache-aside

```typescript
async function getUser(userId: string): Promise<User> {
  const cacheKey = `user:${userId}`;
  const cached = await redis.get(cacheKey);

  if (cached) {
    return JSON.parse(cached);
  }

  const user = await db.user.findUnique({ where: { id: userId } });
  if (user) {
    await redis.set(cacheKey, JSON.stringify(user), "EX", 3600);
  }

  return user;
}

async function invalidateUser(userId: string): Promise<void> {
  await redis.del(`user:${userId}`);
  await redis.del(`user:${userId}:orders`);
}

async function cacheAside<T>(
  key: string,
  ttlSeconds: number,
  fetcher: () => Promise<T>
): Promise<T> {
  const cached = await redis.get(key);
  if (cached) return JSON.parse(cached);

  const value = await fetcher();
  await redis.set(key, JSON.stringify(value), "EX", ttlSeconds);
  return value;
}
```

## Rate Limiting with Sliding Window

```typescript
async function isRateLimited(
  clientId: string,
  limit: number,
  windowSeconds: number
): Promise<boolean> {
  const key = `ratelimit:${clientId}`;
  const now = Date.now();
  const windowStart = now - windowSeconds * 1000;

  const pipe = redis.multi();
  pipe.zremrangebyscore(key, 0, windowStart);
  pipe.zadd(key, now, `${now}:${crypto.randomUUID()}`);
  pipe.zcard(key);
  pipe.expire(key, windowSeconds);

  const results = await pipe.exec();
  const count = results[2][1] as number;
  return count > limit;
}
```

## Pub/Sub

```typescript
const subscriber = redis.duplicate();
await subscriber.subscribe("notifications", "orders");

subscriber.on("message", (channel, message) => {
  const event = JSON.parse(message);
  switch (channel) {
    case "notifications":
      handleNotification(event);
      break;
    case "orders":
      handleOrderEvent(event);
      break;
  }
});

async function publishEvent(channel: string, event: object): Promise<void> {
  await redis.publish(channel, JSON.stringify(event));
}
```

## Streams for Event Processing

```typescript
async function produceEvent(stream: string, event: Record<string, string>) {
  await redis.xadd(stream, "*", ...Object.entries(event).flat());
}

async function consumeEvents(
  stream: string,
  group: string,
  consumer: string
) {
  try {
    await redis.xgroup("CREATE", stream, group, "0", "MKSTREAM");
  } catch {
    // group already exists
  }

  while (true) {
    const results = await redis.xreadgroup(
      "GROUP", group, consumer,
      "COUNT", 10,
      "BLOCK", 5000,
      "STREAMS", stream, ">"
    );

    if (!results) continue;

    for (const [, messages] of results) {
      for (const [id, fields] of messages) {
        await processMessage(fields);
        await redis.xack(stream, group, id);
      }
    }
  }
}
```

Unlike pub/sub, streams persist their entries — consumer groups let you acknowledge, retry, and replay messages, so nothing is lost when a consumer is offline.

## Lua Script for Atomic Operations

```typescript
const acquireLock = `
  local key = KEYS[1]
  local token = ARGV[1]
  local ttl = ARGV[2]
  if redis.call("SET", key, token, "NX", "EX", ttl) then
    return 1
  end
  return 0
`;

const releaseLock = `
  local key = KEYS[1]
  local token = ARGV[1]
  if redis.call("GET", key) == token then
    return redis.call("DEL", key)
  end
  return 0
`;

async function withLock<T>(
  resource: string,
  ttl: number,
  fn: () => Promise<T>
): Promise<T> {
  const token = crypto.randomUUID();
  const acquired = await redis.eval(acquireLock, 1, `lock:${resource}`, token, ttl);
  if (!acquired) throw new Error("Failed to acquire lock");
  try {
    return await fn();
  } finally {
    await redis.eval(releaseLock, 1, `lock:${resource}`, token);
  }
}
```

## What to avoid

- Dropping large payloads (>100KB) into a key without compressing them first
- Reaching for `KEYS *` on a live server — it blocks everything; scan incrementally with `SCAN`
- Leaving cache entries without a TTL, so memory creeps up forever
- Treating pub/sub as a message queue — anything published while no one is listening is gone
- Making Redis your only system of record with no persistence plan behind it
- Firing sequential commands one at a time instead of batching them in a pipeline

## Before you ship

- [ ] Keys follow one predictable shape (`entity:id:field`)
- [ ] Every cached value carries a TTL so memory can't leak
- [ ] Production pattern lookups use `SCAN`, never `KEYS`
- [ ] Anything that must be atomic runs inside a Lua script
- [ ] Durability requirements are served by streams rather than pub/sub
- [ ] High-throughput paths sit behind a connection pool
- [ ] Rate limits are sliding-window, backed by sorted sets
- [ ] Locks carry a fencing token and a TTL
