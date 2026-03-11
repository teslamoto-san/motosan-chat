# motosan-chat

A Rust framework for building multi-platform AI chatbots with **two-phase response** —
reply instantly, generate fully in the background.

```
User sends message
      │
      ├─ Phase 1: Instant reply (< 1s)   "思考中..."
      │
      └─ Phase 2: Background generation  完整 AI 回覆
```

## Design Goals

- **Zero-assumption core** — traits only, no forced dependencies
- **Two-phase response** — quick reply + background full generation, built-in
- **Multi-platform** — LINE, Telegram, Discord via feature flags
- **Webhook + Polling** — same handler code, swap the source

## Workspace Structure

```
motosan-chat/
├── motosan-core/        # Traits + types, zero deps
│   ├── Adapter trait    # How to SEND (post / push / loading)
│   ├── Source trait     # How to RECEIVE events
│   ├── Thread struct    # Handler interface (post / push / defer)
│   └── Bot struct       # run() loop + TaskTracker
│
├── motosan-telegram/    # Telegram adapter + polling source
├── motosan-line/        # LINE adapter + webhook source (Phase 2)
├── motosan-discord/     # Discord adapter + gateway source (Phase 3)
│
└── examples/
    └── hello-bot/       # Telegram polling, two-phase demo
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
            // Phase 1: instant
            thread.post("思考中...").await?;

            // Phase 2: background
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
// How to send messages
pub trait Adapter: Send + Sync {
    async fn reply(&self, token: &str, text: &str) -> Result<()>;
    async fn push(&self, user_id: &str, text: &str) -> Result<()>;
    async fn loading(&self, user_id: &str) -> Result<()>;
}

// How to receive events
pub trait Source: Send + Sync {
    async fn next(&self) -> Result<IncomingEvent>;
}
```

## Two-Phase Response Explained

| Phase | When | API Used | Cost |
|-------|------|----------|------|
| Phase 1 | Immediately (< 1s) | Reply API (reply_token) | Free |
| Phase 2 | Background | Push API (user_id) | LINE: uses quota |

**LINE optimization**: Use Loading Animation (free) + hold reply_token → reply for free if LLM < 30s.

## Optional Features

```toml
[dependencies]
motosan-core = "0.1"
motosan-telegram = { version = "0.1", optional = true }
motosan-line     = { version = "0.1", optional = true }
motosan-discord  = { version = "0.1", optional = true }
```

| Feature | Adds |
|---------|------|
| `state` | Deduplication + distributed lock |
| `history` | Conversation history (multi-turn) |

## Roadmap

- [x] Architecture design
- [ ] **v0.1** — `motosan-core` traits + `motosan-telegram` + `hello-bot` example
- [ ] **v0.2** — `motosan-line` (webhook + loading animation + reply token optimization)
- [ ] **v0.3** — `state` feature (dedup + lock)
- [ ] **v0.4** — `history` feature (multi-turn conversation)
- [ ] **v0.5** — `motosan-discord` (Gateway WebSocket)

## License

MIT
