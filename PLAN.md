# PLAN.md — motosan-chat 開發計畫

## v0.1 目標（第一版，Telegram only）

讓一個 Telegram bot 能：
1. 收到訊息
2. 立刻回「思考中...」（Phase 1）
3. 背景跑任意 async 任務（Phase 2）
4. 結果 push 回去

---

## Crate 清單

### `motosan-core`（必須先做）

```
src/
├── lib.rs
├── adapter.rs      # trait Adapter { reply, push, loading }
├── source.rs       # trait Source { next() -> IncomingEvent }
├── event.rs        # struct IncomingEvent { user_id, reply_token, text, ... }
├── thread.rs       # struct Thread { post(), push(), defer() }
└── bot.rs          # struct Bot + BotBuilder + run()
```

依賴：`tokio`, `tokio-util`, `async-trait`, `thiserror`

---

### `motosan-telegram`（第一個 adapter）

```
src/
├── lib.rs
├── adapter.rs      # TelegramAdapter: impl Adapter
│   ├── reply()     # sendMessage（用 chat_id）
│   ├── push()      # sendMessage（同 reply，Telegram 沒差）
│   └── loading()   # sendChatAction { action: "typing" }
│
└── polling.rs      # TelegramPolling: impl Source
    └── next()      # getUpdates long-polling（offset 管理）
```

依賴：`motosan-core`, `reqwest`, `serde_json`

---

### `examples/hello-bot`

```rust
// 最簡單的 demo：
// 1. 收到訊息
// 2. 立刻回「收到，處理中...」
// 3. sleep 3 秒（模擬 LLM）
// 4. push 完整回覆
```

---

## 實作順序

```
Step 1: motosan-core
  ├─ IncomingEvent struct
  ├─ trait Adapter
  ├─ trait Source
  ├─ Thread struct（含 deferred Vec）
  └─ Bot::run()（loop + TaskTracker + drain deferred）

Step 2: motosan-telegram
  ├─ TelegramPolling::next()（getUpdates）
  └─ TelegramAdapter::reply/push/loading（sendMessage）

Step 3: examples/hello-bot
  └─ 接起來，本機跑通
```

---

## 關鍵設計決策

### Thread.defer() 的執行時機

```
handler() 執行
    │
    ├─ thread.post("思考中...")   ← 同步，立刻打 API
    └─ thread.defer(future)       ← 只是 push 進 Vec

handler 返回
    │
    ├─ Bot drain deferred Vec
    └─ tracker.spawn(future)      ← 這時才真正執行

→ 使用者看不到這個細節
```

### Telegram 的 reply 等於 push

```
Telegram 沒有「reply_token」概念，
sendMessage 直接用 chat_id，任何時間都可以送。
所以 Telegram 的 reply() 和 push() 實作相同。
→ 不扣任何額度
→ Phase 1 / Phase 2 都免費
```

---

## 不在第一版的東西

| 功能 | 版本 |
|------|------|
| LINE adapter | v0.2 |
| deduplication lock | v0.3 |
| 對話歷史 | v0.4 |
| Discord | v0.5 |
| 多平台同時跑 | v0.5 |
| Redis StateStore | v0.3+ |

---

## 本機測試流程

```bash
# 1. 從 @BotFather 拿 token
export TELEGRAM_TOKEN="123456:ABC..."

# 2. 跑 example
cargo run -p hello-bot

# 3. 在 Telegram 傳訊息給 bot
# 預期：立刻看到「收到，處理中...」，3 秒後看到完整回覆
```

---

## 成功標準（v0.1 done）

- [ ] `cargo build -p motosan-core` 成功
- [ ] `cargo build -p motosan-telegram` 成功
- [ ] `cargo run -p hello-bot` 跑起來
- [ ] Telegram 傳訊息，兩則回覆依序出現（Phase 1 → Phase 2）
- [ ] Ctrl+C 優雅關機（TaskTracker 等背景任務完成）
