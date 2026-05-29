# 01 — AI Support Bot для онлайн-школы

RAG-бот техподдержки с обработкой 3 параллельных каналов (Telegram, WhatsApp, сайт),
двухуровневой защитой от сбоев AI и автоматической передачей оператору.

**Стек:** n8n · Vertex AI Embeddings · Gemini · Supabase · Telegram Bot API · WhatsApp (Wazzup) · Site Chat

---

## Задача

Закрыть типовые вопросы студентов курса 24/7 без участия живых кураторов, при этом:
- не путаться в каналах (студент пишет туда, где ему удобно);
- не терять сообщение при сбое AI-сервисов;
- мгновенно передавать диалог оператору, как только он включился в переписку;
- держать память диалога per-user, чтобы бот не задавал заново уже отвеченные вопросы.

---

## Архитектура

Система декомпозирована на **два независимых workflow'а**, чтобы синхронный ответ webhook'у не зависел от скорости AI:

### Ingest (синхронный, отвечает на webhook за <500мс)

```mermaid
flowchart LR
  TG[Webhook · Telegram] --> N1[Normalize]
  WA[Webhook · WhatsApp/Wazzup] --> N2[Normalize]
  WB[Webhook · Site Chat] --> N3[Normalize]
  N1 --> S[Switch sender_type]
  N2 --> S
  N3 --> S
  S -->|user| R1[RPC handle_user_message]
  S -->|operator| R2[RPC handle_operator_takeover]
  S -->|bot| R3[NoOp echo]
  R1 --> OK[HTTP 200]
  R2 --> OK
  R3 --> OK
```

Каждый канал нормализуется к единому формату payload'а, потом Switch роутит по типу отправителя в нужный RPC.
Webhook возвращает HTTP 200 практически мгновенно.

### Processor (асинхронный, cron каждую минуту)

```mermaid
flowchart LR
  CR[Cron 1 min] --> P[Pick ready conversations]
  P --> VE[Vertex Embedding]
  VE -->|OK| CTX[Build context]
  VE -->|fail| F1[Pause bot + Send Operator]
  CTX --> G[Gemini generate]
  G -->|OK| PR[Parse Gemini]
  G -->|fail| F2[Pause bot + Send Operator]
  PR --> GT[Gate · need operator?]
  GT -->|no| SUP[Suppressed?]
  GT -->|yes| F3[Send Operator]
  SUP -->|no| SW{Switch by source}
  SW --> SO[Send Operator]
  SW --> ST[Send Telegram]
  SW --> SS[Send Site]
  SO --> M[Update AI Memory]
  ST --> M
  SS --> M
  M --> D[Mark done]
```

Cron-trigger каждую минуту забирает диалоги, готовые к ответу. Каждая AI-нода имеет
**fallback-ветку**: при сбое Vertex Embedding или Gemini диалог автоматически
передаётся живому оператору, бот ставится на паузу для этого диалога.

---

## Архитектурные решения

| Решение | Почему |
|---|---|
| Ingest и Processor разнесены | Webhook отвечает мгновенно, не дожидаясь AI. Каналы не отваливаются по timeout даже если Gemini тормозит. |
| Cron 1 мин вместо webhook-driven AI | Можно батчить обработку, проще наблюдать в Executions, легко добавлять retry. |
| Fallback на оператора при сбое AI | Лучше передать живому человеку чем потерять сообщение или дать галлюцинацию. |
| Operator takeover gate | Как только оператор написал — бот замолкает в этом диалоге автоматически, без ручной паузы. |
| AI Memory per conversation | Бот помнит контекст диалога между сессиями, не задаёт уже отвеченные вопросы. |
| Suppressed-флаг | Возможность точечно отключить бота в конкретном диалоге, не трогая основной workflow. |

---

## Скриншоты

### Ingest workflow

![Support Bot Ingest](./screenshots/ingest.png)

Единая точка входа для 3 каналов: webhook → нормализация payload'а к общему формату →
Switch по типу отправителя (user / operator / bot) → роутинг в нужный RPC → HTTP 200.

### Processor workflow

![Support Bot Processor](./screenshots/processor.png)

AI-пайплайн с fallback-ветками: cron 1 мин → Vertex Embedding (при сбое →
пауза бота + алерт оператору) → сборка контекста → Gemini generate (тот же fallback) →
gate «нужен оператор?» → Switch by source → отправка в правильный канал →
обновление AI-памяти.
