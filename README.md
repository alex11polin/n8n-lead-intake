# Lead intake — n8n workflow

Сценарий автоматизации на n8n: принимает заявку по webhook, проверяет и нормализует данные,
уведомляет администратора в Telegram и сохраняет заявку в Google Sheets.

```
POST /webhook/lead-intake
        │
        ▼
 Validate & normalize ──► Valid? ──yes──► Telegram (уведомление) ──► Google Sheets (строка) ──► 200 {status, id}
  (Code node)                 │
                              └─no────► 400 {status: "error", errors: [...]}
```

## Что делает

- **Webhook** принимает `POST` с JSON `{ name, phone, message, source }` — например, из формы на сайте или из бота.
- **Валидация** (Code node): имя ≥ 2 символов, телефон 10–15 цифр (приводится к виду `+79991234567`),
  сообщение ≥ 3 символов. Каждой заявке присваивается `id` и время создания.
- **Невалидная заявка** получает ответ `400` со списком ошибок, ничего не сохраняется.
- **Валидная заявка**: сообщение админу в Telegram → строка в Google Sheets → ответ `200` с `id` заявки.
- Узлы Telegram и Sheets настроены так, что сбой одного из них (нет доступа, лимит) не роняет весь сценарий
  и заявитель всё равно получает ответ.

## Запуск

Нужен Node.js 20+.

```bash
npm install          # ставит n8n локально
npx n8n              # UI на http://localhost:5678
```

Импорт сценария: в UI → **Workflows → Import from file** → `lead-intake.workflow.json`.

Настроить в UI (значения в файле — заглушки):

1. **Telegram**: создайте credential с токеном бота от @BotFather; в узле замените `YOUR_TELEGRAM_CHAT_ID`
   на id чата администратора.
2. **Google Sheets**: подключите Google-аккаунт (OAuth); замените `YOUR_GOOGLE_SHEET_ID`, в таблице создайте
   лист `Leads` с колонками `id, createdAt, name, phone, message, source`.
3. Нажмите **Publish/Activate**.

## Проверка

Валидная заявка:

```bash
curl -X POST http://localhost:5678/webhook/lead-intake \
  -H 'Content-Type: application/json' \
  -d '{"name":"Иван","phone":"+7 (999) 123-45-67","message":"Хочу бота для салона","source":"site"}'
# → {"status":"ok","id":"L..."}
```

Невалидная:

```bash
curl -i -X POST http://localhost:5678/webhook/lead-intake \
  -H 'Content-Type: application/json' -d '{"name":"И","phone":"123","message":""}'
# → 400 {"status":"error","errors":["name: ...","phone: ...","message: ..."]}
```

## Ограничения

- Нет защиты webhook от спама (ни токена в заголовке, ни лимита запросов).
- Нет защиты от дублей: повторная отправка создаёт вторую заявку.
- Хранилище — Google Sheets: подходит для десятков заявок в день, для большего лучше CRM или база данных.
- Сценарий рассчитан на одного администратора; маршрутизации по менеджерам нет.

## Как сделано

Сценарий собран с помощью ИИ-ассистента (Claude): я формулировал требования и проверял поведение
запросами `curl`, разбирался в узлах n8n и обработке ошибок.
