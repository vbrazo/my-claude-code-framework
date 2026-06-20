---
name: rust-systems
description: Systems Rust in practice — ownership and borrowing, traits and generics, async on Tokio, error handling, and unsafe discipline
---

# Systems programming in Rust

## Ownership and borrowing

```rust
fn process_data(data: &[u8]) -> Vec<u8> {
    data.iter().map(|b| b.wrapping_add(1)).collect()
}

fn modify_in_place(data: &mut Vec<u8>) {
    data.retain(|b| *b != 0);
    data.sort_unstable();
}

fn take_ownership(data: Vec<u8>) -> Vec<u8> {
    let mut result = data;
    result.push(0xFF);
    result
}

fn main() {
    let data = vec![1, 2, 3, 0, 4];
    let processed = process_data(&data);     // borrow: data still usable
    let mut owned = take_ownership(data);     // move: data no longer usable
    modify_in_place(&mut owned);              // mutable borrow
}
```

Borrow (`&T`, `&mut T`) before you move, and reach for `Clone` only when there's no borrow that works.

## Handling errors

```rust
use thiserror::Error;

#[derive(Error, Debug)]
pub enum AppError {
    #[error("database error: {0}")]
    Database(#[from] sqlx::Error),

    #[error("not found: {resource} with id {id}")]
    NotFound { resource: &'static str, id: String },

    #[error("validation failed: {0}")]
    Validation(String),
}

type Result<T> = std::result::Result<T, AppError>;

async fn get_user(pool: &PgPool, id: &str) -> Result<User> {
    sqlx::query_as::<_, User>("SELECT * FROM users WHERE id = $1")
        .bind(id)
        .fetch_optional(pool)
        .await?
        .ok_or_else(|| AppError::NotFound {
            resource: "User",
            id: id.to_string(),
        })
}
```

Reach for `thiserror` in libraries and `anyhow` at the application edge, and keep `.unwrap()` out of production paths.

## Traits and generics

```rust
trait Repository {
    type Item;
    type Error;

    async fn find_by_id(&self, id: &str) -> std::result::Result<Option<Self::Item>, Self::Error>;
    async fn save(&self, item: &Self::Item) -> std::result::Result<(), Self::Error>;
}

struct PgUserRepo {
    pool: PgPool,
}

impl Repository for PgUserRepo {
    type Item = User;
    type Error = AppError;

    async fn find_by_id(&self, id: &str) -> Result<Option<User>> {
        let user = sqlx::query_as::<_, User>("SELECT * FROM users WHERE id = $1")
            .bind(id)
            .fetch_optional(&self.pool)
            .await?;
        Ok(user)
    }

    async fn save(&self, user: &User) -> Result<()> {
        sqlx::query("INSERT INTO users (id, name, email) VALUES ($1, $2, $3)")
            .bind(&user.id)
            .bind(&user.name)
            .bind(&user.email)
            .execute(&self.pool)
            .await?;
        Ok(())
    }
}
```

## Async on Tokio

```rust
use tokio::sync::Semaphore;
use futures::stream::{self, StreamExt};

async fn fetch_all(urls: Vec<String>, max_concurrent: usize) -> Vec<Result<String>> {
    let semaphore = Arc::new(Semaphore::new(max_concurrent));

    stream::iter(urls)
        .map(|url| {
            let sem = semaphore.clone();
            async move {
                let _permit = sem.acquire().await.unwrap();
                reqwest::get(&url).await?.text().await.map_err(Into::into)
            }
        })
        .buffer_unordered(max_concurrent)
        .collect()
        .await
}

async fn graceful_shutdown(handle: tokio::runtime::Handle) {
    let ctrl_c = tokio::signal::ctrl_c();
    ctrl_c.await.expect("Failed to listen for Ctrl+C");
    handle.shutdown_timeout(std::time::Duration::from_secs(30));
}
```

## The builder pattern

```rust
pub struct ServerConfig {
    host: String,
    port: u16,
    workers: usize,
    tls: bool,
}

pub struct ServerConfigBuilder {
    host: String,
    port: u16,
    workers: usize,
    tls: bool,
}

impl ServerConfigBuilder {
    pub fn new() -> Self {
        Self { host: "0.0.0.0".into(), port: 8080, workers: 4, tls: false }
    }

    pub fn host(mut self, host: impl Into<String>) -> Self { self.host = host.into(); self }
    pub fn port(mut self, port: u16) -> Self { self.port = port; self }
    pub fn workers(mut self, n: usize) -> Self { self.workers = n; self }
    pub fn tls(mut self, enabled: bool) -> Self { self.tls = enabled; self }

    pub fn build(self) -> ServerConfig {
        ServerConfig { host: self.host, port: self.port, workers: self.workers, tls: self.tls }
    }
}
```

## What to avoid

- `.unwrap()` or `.expect()` inside library code
- Cloning where a borrow would do
- Holding a `MutexGuard` across an `.await` — that's how you deadlock
- `Arc<Mutex<Vec<T>>>` where a channel would model the flow better
- `unsafe` blocks with no documented invariants
- Result-returning functions that forget `#[must_use]`

## Before you ship

- [ ] Error types use `thiserror`, with `?` carrying them up the stack
- [ ] No `.unwrap()` on production paths
- [ ] The ownership model keeps cloning to a minimum
- [ ] Async work is bounded (semaphores or `buffer_unordered`)
- [ ] Traits provide the abstraction seams for testing
- [ ] Every `unsafe` block spells out its safety invariants
- [ ] Complex config goes through a builder
- [ ] Clippy is on and its warnings are addressed
