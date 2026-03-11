# motosan-chat

A Rust framework for building multi-platform AI chatbots with **two-phase response** —
reply instantly, generate fully in the background.

```
User sends message
      │
      ├─ Phase 1: Instant reply  (< 1s)    "Got it, thinking..."
      │
      └─ Phase 2: Background generation    Full AI response
```

## Design Goals

- **Zero-assumption core** — traits only, no forced dependencies
- **Two-phase response** — quick reply + background full generation, built-in
- **Multi-platform** — LINE, Telegram, Discord via feature flags
- **Webhook + Polling** — same handler code, swap the source
- **Graceful shutdown** — background tasks complete before exit

## Workspace Structure

```
motosan-chat/
├── motosan-core/        # Traits + types, zero external deps
│   ├── Adapter trait    # How to SEND  (reply / push / loading)
│   ├── Source  trait    # How to RECEIVE events
│   ├── Thread  struct   # Handler interface (post / push / defer)
│   └── Bot     struct   # run() loop + TaskTracker
│
├── motosan-telegram/    # Telegram adapter + polling source
├── motosan-line/        # LINE adapter + webhook source       (v0.2)
├── motosan-discord/     # Discord adapter + gateway source    (v0.5)
│
└── examples/
    └── hello-bot/       # Telegram polling — two-phase demo
```

## Quick Start (Telegram)

```rust
use motosan_core::{Bot, Thread};
use motosan_telegram::{TelegramAdapter, TelegramPolling};

#[tokio::main]
async fn main() {
    let token = std::env::var("TELEGRAM_TOKEN").unwrap();

    Bot::builder()
        .adapter(TelegramAdapter::new(&token))
        .source(TelegramPolling::new(&token))
        .on_message(|thread| async move {
            // Phase 1: instant reply
            thread.post("Got it, thinking...").await?;

            // Phase 2: runs in the background after handler returns
            thread.defer(async move {
                let response = generate_response().await;
                thread.push(&response).await?;
            });

            Ok(())
        })
        .build()
        .run()
        .await;
}
```

## Core Traits

```rust
/// How to send messages to a platform
pub trait Adapter: Send + Sync {
    async fn reply(&self, token: &str, text: &str)   -> Result<()>;
    async fn push(&self, user_id: &str, text: &str)  -> Result<()>;
    async fn loading(&self, user_id: &str)           -> Result<()>;
}

/// How to receive events from a platform
pub trait Source: Send + Sync {
    async fn next(&self) -> Result<IncomingEvent>;
}
```

## Thread API

```rust
impl Thread {
    /// Send an immediate reply (before handler returns)
    pub async fn post(&self, text: &str) -> Result<()>;

    /// Push a message any time (uses Push API, not reply token)
    pub async fn push(&self, text: &str) -> Result<()>;

    /// Register a background task — runs after handler returns + 200 OK sent
    pub fn defer<F: Future<Output = ()> + Send + 'static>(&self, f: F);
}
```

## Two-Phase Response Explained

| Phase | Timing | API Used | Cost |
|-------|--------|----------|------|
| Phase 1 | Immediately (< 1s) | Reply API (reply token) | Free |
| Phase 2 | Background | Push API (user_id) | LINE: uses quota |

**LINE optimization**: Use Loading Animation (free) + hold reply token →
reply for free if LLM responds within 30 seconds.

## Handler Patterns

```rust
// Simple reply — no background task
.on_message(|thread| async move {
    thread.post("pong").await?;
    Ok(())
})

// Two-phase — instant ack + background work
.on_message(|thread| async move {
    thread.post("Processing...").await?;
    let t = thread.clone();
    thread.defer(async move {
        let result = heavy_work().await;
        t.push(&result).await?;
    });
    Ok(())
})

// Conditional — decide at runtime
.on_message(|thread| async move {
    if is_simple(&thread.event.text) {
        thread.post(quick_answer()).await?;
    } else {
        thread.post("Analyzing...").await?;
        let t = thread.clone();
        thread.defer(async move {
            t.push(&llm_response().await).await?;
        });
    }
    Ok(())
})
```

## Optional Features

```toml
[dependencies]
motosan-core     = "0.1"
motosan-telegram = { version = "0.1", optional = true }
motosan-line     = { version = "0.1", optional = true }
motosan-discord  = { version = "0.1", optional = true }
```

| Feature   | Adds |
|-----------|------|
| `state`   | Deduplication + distributed lock (prevents webhook retry duplicates) |
| `history` | Conversation history per thread (multi-turn interactions) |

## Roadmap

- [x] Architecture design
- [ ] **v0.1** — `motosan-core` + `motosan-telegram` + `hello-bot` example
- [ ] **v0.2** — `motosan-line` (webhook + loading animation + reply token optimization)
- [ ] **v0.3** — `state` feature (dedup lock, prevents duplicate processing)
- [ ] **v0.4** — `history` feature (multi-turn conversation context)
- [ ] **v0.5** — `motosan-discord` (Gateway WebSocket)

## License

MIT
