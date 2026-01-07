# CLAUDE.md - 82 Auto Telegram Bot

## Principles

- Follow PEP 8 (style), PEP 257 (docstrings), PEP 484 (type hints)
- Security first: parameterized SQL queries, environment variables for secrets, input validation
- Always close database connections
- Use async/await where possible

---

## Project Overview

**82 Auto Telegram Bot** - Korean car import calculator and order management system.

**Features:**
1. Car Cost Calculator - Parse Encar/KBChaCha/KCar URLs, calculate total import costs
2. Order Management - Track orders through 6 delivery stages
3. Subscription Management - Free calculation limits + premium subscriptions
4. FAQ System - Categorized Q&A about import process
5. Manager Dashboard - View users/orders, update statuses
6. Multi-Currency - KRW, USD, RUB with real-time rates

**User Types:**
- `физ` - Individual buyers (private import)
- `юр` - Legal entities (commercial import)
- Managers - Admin users (MANAGER_ID_1, MANAGER_ID_2 env vars)

**Tech Stack:**
| Component | Technology |
|-----------|------------|
| Language | Python 3.10.12 |
| Bot Framework | pyTelegramBotAPI 4.26.0 |
| Database | PostgreSQL (psycopg2) |
| Web Scraping | BeautifulSoup4, Selenium, Playwright |
| HTTP | requests, httpx, aiohttp |
| Scheduling | APScheduler |

---

## File Structure

```
82auto-bot/
├── main.py              # Core bot logic (~2,870 lines)
├── database.py          # DB operations (~323 lines)
├── utils.py             # Helpers (~142 lines)
├── get_currency_rates.py
├── requirements.txt
├── Procfile             # worker: python3 main.py
└── assets/logo.jpeg
```

**Key Functions:**

| Function | File | Purpose |
|----------|------|---------|
| `send_welcome()` | main.py | `/start` handler |
| `get_car_info(url)` | main.py | Parse car listing URL |
| `calculate_cost()` | main.py | Main cost calculation |
| `add_order()` | database.py | Insert order |
| `get_orders()` | database.py | Fetch user orders |
| `calculate_age()` | utils.py | Car age category for customs |
| `get_customs_fees()` | utils.py | Calculate customs via calcus.ru |

**main.py Structure:**
- Lines 1-150: Imports, logging, globals
- Lines 151-500: Utility functions
- Lines 501-700: Currency services
- Lines 701-1400: Car parsers (Encar, KBChaCha, KCar)
- Lines 1401-1900: Cost calculation
- Lines 1901-2300: Order management
- Lines 2301-2500: FAQ system
- Lines 2501-2850: Handlers
- Lines 2851+: bot.polling()

---

## Database Tables

**orders** - Main business table
- id, user_id, car_id, title, price, link, year, month, mileage, engine_volume
- transmission, user_name, full_name, phone_number, images[]
- status, total_cost_usd/krw/rub, created_at

**Order Statuses:**
1. `🚗 Авто выкуплен (на базе)`
2. `🚢 Отправлен в порт г. Пусан`
3. `🌊 В пути до Владивостока`
4. `🛃 Растаможка`
5. `📦 Загрузка на грузовик`
6. `🚛 Доставка к клиенту`

**users** - telegram_id (unique), username, first_name, last_name, created_at

**subscriptions** - user_id (PK), status (bool), created_at, updated_at

**calculations** - user_id (PK), count (int), created_at

**logs** - id, telegram_id, action, data (JSONB), created_at

---

## External APIs

| Service | URL | Purpose |
|---------|-----|---------|
| Encar | fem.encar.com | Car listings, inspection reports |
| KBChaCha | kbchachacha.com | Car listings (HTML scraping) |
| KCar | kcar.com | Car listings (Selenium required) |
| CBR | cbr-xml-daily.ru/daily_json.js | USD/EUR/CNY rates |
| Upbit | api.upbit.com | USDT/KRW rate |
| Calcus | calcus.ru/calculate/Customs | Russian customs duties |

---

## Security Reminders

- Always use parameterized queries: `cursor.execute("... WHERE id = %s", (id,))`
- Load secrets from env: `os.getenv("BOT_TOKEN")`
- Validate marketplace URLs before parsing
- Check `is_manager(user_id)` before admin commands
- Never log PII (phone, full_name)
- Always verify SSL (`verify=True` in requests)
- Add rate limiting to prevent abuse
