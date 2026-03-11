# PLAN.md — motosan-chat Development Plan

## v0.1 Goal (Telegram only)

A working Telegram bot that:
1. Receives a message
2. Instantly replies "Got it, processing..." (Phase 1)
3. Runs any async task in the background (Phase 2)
4. Pushes the full result back to the user

---

## Crates

### `motosan-core` (must be done first)

```
src/
├── lib.rs
├── adapter.rs      # trait Adapter { reply, push, loading }
├── source.rs       # trait Source { next() -> IncomingEvent }
├── event.rs        # struct IncomingEvent { user_id, reply_token, text, platform, ... }
├── thread.rs       # struct Thread { post(), push(), defer() }
└── bot.rs          # struct Bot + BotBuilder + run()
```

Dependencies: `tokio`, `tokio-util`, `async-trait`, `thiserror`

---

### `motosan-telegram` (first adapter)

```
src/
├── lib.rs
├── adapter.rs      # TelegramAdapter: impl Adapter
│   ├── reply()     # sendMessage (chat_id — Telegram has no reply token)
│   ├── push()      # sendMessage (same as reply on Telegram)
│   └── loading()   # sendChatAction { action: "typing" }
│
└── polling.rs      # TelegramPolling: impl Source
    └── next()      # getUpdates long-polling (manages offset)
```

Dependencies: `motosan-core`, `reqwest`, `serde_json`

---

### `examples/hello-bot`

```rust
// Minimal demo:
// 1. Receive a message
// 2. Instantly reply "Got it, processing..."
// 3. Sleep 3 seconds (simulate LLM)
// 4. Push the full response
```

---

## Implementation Order

```
Step 1: motosan-core
  ├─ IncomingEvent struct
  ├─ trait Adapter
  ├─ trait Source
  ├─ Thread struct (deferred Vec inside)
  └─ Bot::run() (loop + TaskTracker + drain deferred after handler)

Step 2: motosan-telegram
  ├─ TelegramPolling::next() (getUpdates)
  └─ TelegramAdapter::reply / push / loading (sendMessage)

Step 3: examples/hello-bot
  └─ Wire everything together and run locally
```

---

## Key Design Decisions

### How `defer()` works internally

```
handler() executes
    │
    ├─ thread.post("Got it...")   ← sync, calls API immediately
    └─ thread.defer(future)       ← just pushes into internal Vec, returns instantly

handler returns
    │
    ├─ Bot drains the deferred Vec
    └─ tracker.spawn(future)      ← background task starts here

→ Users never see this detail
```

### Telegram: reply == push

```
Telegram has no "reply_token" concept.
sendMessage uses chat_id and works at any time.
So TelegramAdapter::reply() and push() are identical.
→ No quota consumed
→ Phase 1 and Phase 2 are both free
```

---

## Out of Scope for v0.1

| Feature | Version |
|---------|---------|
| LINE adapter | v0.2 |
| Deduplication lock | v0.3 |
| Conversation history | v0.4 |
| Discord | v0.5 |
| Multiple platforms simultaneously | v0.5 |
| Redis StateStore | v0.3+ |

---

## Local Test Flow

```bash
# 1. Get a token from @BotFather on Telegram
export TELEGRAM_TOKEN="123456:ABC..."

# 2. Run the example
cargo run -p hello-bot

# 3. Send a message to your bot in Telegram
# Expected: "Got it, processing..." appears instantly,
#           full response appears ~3 seconds later
```

---

## v0.1 Done Criteria

- [ ] `cargo build -p motosan-core` succeeds
- [ ] `cargo build -p motosan-telegram` succeeds
- [ ] `cargo run -p hello-bot` starts without errors
- [ ] Sending a Telegram message triggers two sequential replies (Phase 1 → Phase 2)
- [ ] Ctrl+C triggers graceful shutdown (TaskTracker waits for background tasks)
