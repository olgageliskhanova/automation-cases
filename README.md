# Automation Cases

Публичное портфолио кейсов по автоматизации и AI-инжинирингу. Семь production-проектов: AI-боты с RAG, многоязычный куратор сообщества с модерацией, генерация контента, конкурентная разведка, обработка лидов, гибридная транскрипция речи.

Имена клиентов, внутренних сервисов, customer-данные удалены. Фокус — на архитектурных решениях, выборе технологий и trade-off'ах.

---

## Кейсы

| # | Проект | Стек | Что показывает |
|---|---|---|---|
| 01 | [AI Support Bot для онлайн-школы](./01-support-bot/) | n8n · Vertex AI · Gemini · Supabase · multi-channel | RAG с двухуровневым fallback и operator takeover |
| 02 | [Community Curator — Scheduled Greetings](./02-community-greetings/) | n8n · Telegram · Google Sheets · cron | Декаплинг scheduled от realtime для надёжности |
| 03 | [AI Instagram Carousel Generator](./03-carousel-generator/) | n8n · AI image gen · async polling · chunked upload | Async batch-пайплайн с rate-limit handling |
| 04 | [Instagram Competitor Reels Tracker](./04-competitor-tracker/) | n8n · Apify · ScrapeCreators · Airtable · Supabase | Outlier detection по personal baseline |
| 05 | [Lead Intake Automation](./05-lead-intake/) | Albato / Make · amoCRM · Telegram · Gmail | Валидация, дедуп, multi-channel роутинг |
| 06 | [Transcribe Bot для команды](./06-transcribe-bot/) | Python · python-telegram-bot · Groq + faster-whisper · SQLite · yt-dlp | Очередь с атомарным claim, subs-first, hybrid STT с fallback |
| 07 | [AI-куратор турецкоязычного сообщества](./07-turkish-curator/) | TypeScript · grammY · Gemini (Vertex AI) · sqlite-vec · Groq Whisper | Debounce + мьютекс, человекоподобные паузы, модерация через LLM-судью, память per-user |

---

## Tech Stack

**Automation:** n8n · Make · Albato · Webhooks · SQLite-очереди
**AI:** Vertex AI · Gemini · OpenAI · Qwen (ModelScope) · pgvector · Whisper · Groq Whisper-v3-turbo
**Data:** Supabase · Airtable · Google Sheets API
**Scraping:** Apify · ScrapeCreators · yt-dlp
**Messaging:** Telegram Bot API · grammY · WhatsApp (через Wazzup)
**CRM:** amoCRM · Bitrix24

---

## Принципы

- **Декаплинг scheduled и realtime** — поздравления и алерты живут в отдельных workflow'ах от основного бота, чтобы scheduled-задачи не падали вместе с AI-стеком
- **Двухуровневая защита AI** — при сбое embedding или generation диалог автоматически передаётся живому оператору без потери сообщения
- **Async polling вместо блокирующих ожиданий** — для image generation, video processing, любых долгих внешних API
- **Sheet как content store** — чтобы не-технические сотрудники могли править тексты без касания n8n
- **Два источника данных для критичных пайплайнов** — Apify + ScrapeCreators для парсинга, чтобы при падении одного работала вторая ветка
- **Резервный движок вместо отказа** — критичные операции (распознавание речи) имеют запасной локальный движок: при сбое или лимите основного облачного задача доделывается, а не теряется
- **Пропускать дорогую работу, когда можно** — если у видео уже есть субтитры, расшифровка обходит распознавание целиком: экономия CPU, трафика и диска
- **Склейка очередей сообщений вместо ответа на каждое** — дебаунс-буфер и мьютекс на пользователя коалесцируют поток коротких сообщений в один осмысленный ответ и не плодят параллельные AI-вызовы на одного человека
- **Модерация AI до публикации** — ответ бота в живом сообществе проходит проверку LLM-судьёй на соответствие правилам, прежде чем уйти в группу

---

## Контакты

- Telegram: [@jinny85](https://t.me/jinny85)
- Email: ola_85@mail.ru
