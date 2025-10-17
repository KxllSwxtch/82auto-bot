# CLAUDE.md - 82 Auto Telegram Bot Developer Guide

## 🎯 Your Role & Expertise

You're a Senior Telegram Bot developer with 40+ years of experience, you can develop any type of telegram bot for any type of business. You're also an expert in Python language and you know everything about it, you're following all PEP standards when coding in Python, breaking down each task into smaller chunks for the best solution possible, following all security principles and always checking the code for any vulnerabilities and fix them if they appear.

### Core Principles

- ✅ **PEP Compliance**: Strictly follow PEP 8 (style), PEP 257 (docstrings), PEP 484 (type hints)
- 🔒 **Security First**: Always validate inputs, use parameterized queries, protect sensitive data
- 🧪 **Testing**: Write tests before or alongside code (test-driven development)
- 🔄 **Task Breakdown**: Decompose complex problems into manageable, testable units
- 🛡️ **Vulnerability Scanning**: Review every change for security implications
- 🎯 **Best Practices**: Use async/await, proper error handling, logging, and monitoring

---

## 📊 Project Overview

### Business Purpose

**82 Auto Telegram Bot** is a Korean car import calculator and order management system. It helps users calculate the total cost of importing vehicles from South Korea (Encar, KBChaCha, KCar marketplaces) to Russia, including all fees, customs, shipping, and logistics.

### Key Features

1. **Car Cost Calculator** - Parse Korean marketplace URLs and calculate total import costs
2. **Order Management System** - Track orders through 6-stage delivery pipeline
3. **Subscription Management** - Free calculation limits and premium subscriptions
4. **FAQ System** - Categorized questions about the import process
5. **Manager Dashboard** - View all users and orders, update order statuses
6. **Multi-Currency Support** - KRW, USD, RUB conversions with real-time rates

### Target Users

- **Individual Buyers (физ)** - Private customers importing personal vehicles
- **Legal Entities (юр)** - Businesses importing commercial vehicle fleets
- **Managers** - Admin users managing orders and customer interactions

### Technology Stack

| Component           | Technology                           | Version                 |
| ------------------- | ------------------------------------ | ----------------------- |
| **Language**        | Python                               | 3.10.12                 |
| **Bot Framework**   | pyTelegramBotAPI                     | 4.26.0                  |
| **Database**        | PostgreSQL                           | -                       |
| **DB Driver**       | psycopg2                             | 2.9.10                  |
| **Web Scraping**    | BeautifulSoup4, Selenium, Playwright | 4.12.3, 4.25.0, 1.48.0  |
| **HTTP Clients**    | requests, httpx, aiohttp             | 2.32.3, 0.25.1, 3.10.10 |
| **Task Scheduling** | APScheduler                          | 3.11.0                  |
| **Web Frameworks**  | FastAPI, Flask                       | 0.115.12, 3.1.0         |
| **Configuration**   | python-dotenv                        | 1.0.1                   |

---

## 🏗️ Architecture & Design Patterns

### Current Architecture: Monolithic Handler-Based

```
┌─────────────────────────────────────────────────────────┐
│                    Telegram API                         │
│                  (Bot Polling Mode)                     │
└──────────────────┬──────────────────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────────────────────┐
│                   main.py (2,870 lines)                 │
│  ┌────────────────────────────────────────────────┐    │
│  │  Message/Callback Handlers (23+ handlers)      │    │
│  │  - /start, /orders, /my_cars, /users          │    │
│  │  - order_car_*, update_status_*, FAQ_*        │    │
│  └────────────┬───────────────────────────────────┘    │
│               │                                          │
│  ┌────────────▼───────────────────────────────────┐    │
│  │  Business Logic (46 functions)                 │    │
│  │  - get_car_info(), calculate_cost()           │    │
│  │  - process_order(), notify_managers()         │    │
│  └────────────┬───────────────────────────────────┘    │
│               │                                          │
│  ┌────────────▼───────────────────────────────────┐    │
│  │  Global State Management                       │    │
│  │  - car_data, user_data, user_type_map         │    │
│  │  - krw_rub_rate, usd_rate, usdt_to_krw_rate   │    │
│  └────────────┬───────────────────────────────────┘    │
└───────────────┼──────────────────────────────────────────┘
                │
       ┌────────┴────────┐
       ▼                 ▼
┌──────────────┐  ┌──────────────────┐
│ database.py  │  │    utils.py      │
│ (323 lines)  │  │  (142 lines)     │
│              │  │                  │
│ CRUD Ops:    │  │ Helpers:         │
│ - orders     │  │ - calculate_age()│
│ - users      │  │ - customs_fees() │
│ - subscr.    │  │ - format_number()│
│ - calcs      │  │ - clean_number() │
└──────┬───────┘  └──────────────────┘
       │
       ▼
┌──────────────────────────┐
│   PostgreSQL Database    │
│  ┌────────────────────┐  │
│  │ orders             │  │
│  │ users              │  │
│  │ subscriptions      │  │
│  │ calculations       │  │
│  │ logs               │  │
│  └────────────────────┘  │
└──────────────────────────┘

External Services:
- Encar API (fem.encar.com)
- KBChaCha (kbchachacha.com)
- KCar (kcar.com)
- CBR Currency API (cbr-xml-daily.ru)
- Customs Calculator (calcus.ru)
```

### Design Patterns Used

#### 1. **Handler-Based Pattern**

```python
# Command handlers
@bot.message_handler(commands=["start"])
def send_welcome(message):
    # Initialize user session
    pass

# Callback query handlers
@bot.callback_query_handler(func=lambda call: call.data.startswith("order_car_"))
def order_car(call):
    # Handle car ordering flow
    pass

# Text message handlers (catch-all)
@bot.message_handler(func=lambda message: True)
def handle_text_message(message):
    # Route based on text content
    pass
```

#### 2. **Global State Management** (⚠️ Needs Refactoring)

```python
# Current implementation uses global dictionaries
car_data = {}           # {user_id: {car_info}}
user_data = {}          # {user_id: {session_data}}
user_type_map = {}      # {user_id: "физ" | "юр"}

# Exchange rate caching
krw_rub_rate = None
usd_rate = None
usdt_to_krw_rate = None
```

**⚠️ Problem**: Global state doesn't scale, causes issues in multi-threaded environments.

**✅ Recommended**: Use class-based state management or Redis for session storage.

#### 3. **Database Abstraction Layer**

```python
# database.py provides clean interface
def add_order(user_id, car_id, title, price, ...):
    """Insert order with proper error handling"""
    pass

def get_orders(user_id):
    """Fetch user orders with RealDictCursor"""
    pass
```

#### 4. **Multi-Step Conversation Flow**

```python
# State machine pattern for complex interactions
def order_car(call):
    # Step 1: Capture user intent
    bot.send_message(chat_id, "Выберите тип плательщика:")
    # Store intermediate state in user_data[user_id]

def process_user_type(message):
    # Step 2: Process previous input, request next
    # Validate and store

def finalize_order(message):
    # Step 3: Complete transaction
    pass
```

---

## 📁 Codebase Structure & Navigation

### File Breakdown

```
/Users/shindmitriy/Desktop/Coding/82Auto/82auto-bot/
├── main.py                      # 2,870 lines - Core bot logic
│   ├── Initialization           # Lines 1-150: Imports, logging, global vars
│   ├── Utility Functions        # Lines 151-500: Helpers, formatters
│   ├── Currency Services        # Lines 501-700: CBR, USDT, USD rates
│   ├── Car Information          # Lines 701-1400: Parsers for 3 marketplaces
│   ├── Cost Calculation         # Lines 1401-1900: Main calc engine
│   ├── Order Management         # Lines 1901-2300: CRUD for orders
│   ├── FAQ System              # Lines 2301-2500: Q&A navigation
│   ├── Handlers                 # Lines 2501-2850: 23+ handler functions
│   └── Main Loop               # Lines 2851-2870: bot.polling()
│
├── database.py                  # 323 lines - Database operations
│   ├── Connection Management    # Lines 1-50
│   ├── Table Creation          # Lines 51-150
│   ├── Order Operations        # Lines 151-220
│   ├── User Management         # Lines 221-260
│   ├── Subscription CRUD       # Lines 261-290
│   └── Calculation Counter     # Lines 291-323
│
├── utils.py                     # 142 lines - Helper functions
│   ├── Age Calculation         # Lines 1-30
│   ├── Customs Fee Calculator  # Lines 31-90
│   ├── Number Formatting       # Lines 91-120
│   └── URL Generators          # Lines 121-142
│
├── get_currency_rates.py        # Currency rate fetching service
├── requirements.txt             # 150+ dependencies
├── runtime.txt                  # Python 3.10.12
├── Procfile                     # Heroku: worker: python3 main.py
├── .env                         # Environment variables (not in git)
└── assets/
    └── logo.jpeg               # Welcome message logo
```

### Key Functions Reference

| Function                | Location        | Purpose                            |
| ----------------------- | --------------- | ---------------------------------- |
| `send_welcome()`        | main.py:2501    | `/start` command handler           |
| `get_car_info(url)`     | main.py:701     | Parse Encar/KBChaCha/KCar URLs     |
| `calculate_cost()`      | main.py:1401    | Main cost calculation engine       |
| `add_order()`           | database.py:151 | Insert order into DB               |
| `calculate_age()`       | utils.py:1      | Determine age category for customs |
| `get_customs_fees()`    | utils.py:31     | POST to calcus.ru for tariff       |
| `notify_managers()`     | main.py:2200    | Alert managers of new orders       |
| `update_order_status()` | main.py:2100    | Manager status update handler      |
| `show_orders()`         | main.py:1950    | Display user's order list          |
| `handle_faq()`          | main.py:2301    | FAQ system entry point             |

### Handler Mapping (23 Handlers)

| Handler Type | Pattern                  | Function                | Purpose               |
| ------------ | ------------------------ | ----------------------- | --------------------- |
| Command      | `/start`                 | `send_welcome()`        | Initialize bot        |
| Command      | `/orders`                | `show_orders()`         | View user orders      |
| Command      | `/my_cars`               | `show_favorite_cars()`  | View saved cars       |
| Command      | `/users`                 | `list_all_users()`      | Manager: view users   |
| Command      | `/exchange_rates`        | `send_exchange_rates()` | Show currency rates   |
| Callback     | `order_car_*`            | `order_car()`           | Start order flow      |
| Callback     | `place_order_*`          | `place_order()`         | Finalize order        |
| Callback     | `add_favorite_*`         | `add_favorite_car()`    | Save car to DB        |
| Callback     | `delete_order_*`         | `delete_order()`        | Remove order          |
| Callback     | `update_status_*`        | `update_order_status()` | Manager status update |
| Callback     | `faq_*`                  | `handle_faq_*()`        | FAQ navigation        |
| Text         | Contains URL             | `calculate_cost()`      | Parse car link        |
| Text         | "Рассчитать Автомобиль"  | `calculate_cost()`      | Start calculation     |
| Text         | "Мои автомобили"         | `show_favorite_cars()`  | View saved cars       |
| Text         | "Мои заказы"             | `show_orders()`         | View orders           |
| Text         | "FAQ"                    | `handle_faq()`          | Open FAQ              |
| Text         | "Связаться с менеджером" | Send contact info       | Manager contact       |

---

## 🗄️ Database Schema & Data Management

### PostgreSQL Schema (5 Tables)

#### 1. **orders** - Main Business Table

```sql
CREATE TABLE IF NOT EXISTS orders (
    id SERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL,                 -- Telegram user ID
    car_id TEXT,                             -- External marketplace ID
    title TEXT,                              -- Car model name
    price FLOAT,                             -- Original price (KRW)
    link TEXT,                               -- Marketplace URL
    year INTEGER,                            -- Manufacturing year
    month INTEGER,                           -- Manufacturing month
    mileage INTEGER,                         -- Odometer reading (km)
    engine_volume INTEGER,                   -- Engine size (cc)
    transmission TEXT,                       -- "Автомат" | "Механика"
    user_name TEXT,                          -- Telegram username
    full_name TEXT,                          -- Customer full name
    phone_number TEXT,                       -- Contact phone
    images TEXT[],                           -- Array of photo URLs
    status TEXT DEFAULT '🔄 Не заказано',   -- Order status
    total_cost_usd FLOAT,                    -- Calculated total (USD)
    total_cost_krw FLOAT,                    -- Calculated total (KRW)
    total_cost_rub FLOAT,                    -- Calculated total (RUB)
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Indexes for performance
CREATE INDEX idx_orders_user_id ON orders(user_id);
CREATE INDEX idx_orders_status ON orders(status);
```

**Order Statuses** (6 stages):

1. `🚗 Авто выкуплен (на базе)` - Car purchased at base
2. `🚢 Отправлен в порт г. Пусан на погрузку` - Sent to Busan port
3. `🌊 В пути до Владивостока` - In transit to Vladivostok
4. `🛃 Растаможка` - Customs clearance
5. `📦 Загрузка на грузовик` - Loading to truck
6. `🚛 Доставка к клиенту` - Delivery to customer

#### 2. **users** - User Registry

```sql
CREATE TABLE IF NOT EXISTS users (
    id SERIAL PRIMARY KEY,
    telegram_id BIGINT UNIQUE NOT NULL,      -- Telegram user ID
    username TEXT,                           -- @username
    first_name TEXT,                         -- First name
    last_name TEXT,                          -- Last name
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Unique constraint on telegram_id
CREATE UNIQUE INDEX idx_users_telegram_id ON users(telegram_id);
```

#### 3. **subscriptions** - Premium Status

```sql
CREATE TABLE IF NOT EXISTS subscriptions (
    user_id BIGINT PRIMARY KEY,              -- References telegram_id
    status BOOLEAN DEFAULT FALSE,            -- TRUE = subscribed
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

#### 4. **calculations** - Free Usage Tracking

```sql
CREATE TABLE IF NOT EXISTS calculations (
    user_id BIGINT PRIMARY KEY,              -- References telegram_id
    count INTEGER DEFAULT 0,                 -- Number of free calculations used
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

#### 5. **logs** - Audit Trail

```sql
CREATE TABLE IF NOT EXISTS logs (
    id SERIAL PRIMARY KEY,
    telegram_id BIGINT,                      -- User who performed action
    action TEXT,                             -- Action description
    data JSONB,                              -- Flexible JSON data
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_logs_telegram_id ON logs(telegram_id);
CREATE INDEX idx_logs_created_at ON logs(created_at);
```

### CRUD Operation Patterns

#### Best Practices

```python
# ✅ GOOD: Parameterized queries prevent SQL injection
def add_order(user_id, car_id, title, price):
    conn = connect_db()
    cursor = conn.cursor()
    cursor.execute("""
        INSERT INTO orders (user_id, car_id, title, price)
        VALUES (%s, %s, %s, %s)
        RETURNING id
    """, (user_id, car_id, title, price))
    order_id = cursor.fetchone()[0]
    conn.commit()
    conn.close()
    return order_id

# ❌ BAD: String interpolation vulnerable to SQL injection
def bad_add_order(user_id, title):
    query = f"INSERT INTO orders (user_id, title) VALUES ({user_id}, '{title}')"
    # NEVER DO THIS!
```

#### Upsert Pattern

```python
# INSERT with ON CONFLICT for idempotent operations
def increment_calculation_count(user_id):
    conn = connect_db()
    cursor = conn.cursor()
    cursor.execute("""
        INSERT INTO calculations (user_id, count)
        VALUES (%s, 1)
        ON CONFLICT (user_id)
        DO UPDATE SET count = calculations.count + 1
    """, (user_id,))
    conn.commit()
    conn.close()
```

#### Using RealDictCursor

```python
from psycopg2.extras import RealDictCursor

def get_orders(user_id):
    """Returns list of dicts instead of tuples"""
    conn = connect_db()
    cursor = conn.cursor(cursor_factory=RealDictCursor)
    cursor.execute(
        "SELECT * FROM orders WHERE user_id = %s ORDER BY created_at DESC",
        (user_id,)
    )
    orders = cursor.fetchall()
    conn.close()
    return orders  # [{'id': 1, 'title': 'Hyundai', ...}, ...]
```

### Connection Management

```python
import os
import psycopg2

def connect_db():
    """
    Create database connection from environment variable.
    Always use context managers or close connections explicitly.
    """
    DATABASE_URL = os.getenv("DATABASE_URL")
    return psycopg2.connect(DATABASE_URL, sslmode='require')

# ✅ GOOD: Context manager (recommended)
def safe_query():
    with connect_db() as conn:
        with conn.cursor() as cursor:
            cursor.execute("SELECT * FROM users")
            return cursor.fetchall()
    # Connection automatically closed

# ⚠️ ACCEPTABLE: Explicit close
def manual_query():
    conn = connect_db()
    try:
        cursor = conn.cursor()
        cursor.execute("SELECT * FROM users")
        return cursor.fetchall()
    finally:
        conn.close()  # Always close!
```

---

## 🔌 API Integration Details

### 1. Encar API Integration

**Base URL**: `https://fem.encar.com/`
**API Endpoint**: `https://api.encar.com/v1/`

#### Car Information Extraction

```python
def get_car_info(url):
    """
    Parse Encar car listing page.

    Args:
        url: Encar car URL (fem.encar.com/cars/detail/...)

    Returns:
        dict: {
            'car_id': str,
            'price': int,
            'year': int,
            'month': int,
            'mileage': int,
            'engine_volume': int,
            'transmission': str,
            'images': list[str]
        }
    """
    response = requests.get(url, headers={'User-Agent': '...'})
    soup = BeautifulSoup(response.text, 'html.parser')

    # Extract car ID from URL
    car_id = re.search(r'carid=(\d+)', url).group(1)

    # Price extraction
    price_elem = soup.find('strong', {'class': 'price'})
    price = int(price_elem.text.replace('만원', '').replace(',', '')) * 10000

    # Technical specs from detail table
    details = soup.find('table', {'class': 'detail-info'})
    year = int(details.find(text='연식').parent.next_sibling.text[:4])
    mileage = int(details.find(text='주행거리').parent.next_sibling.text.replace('km', '').replace(',', ''))

    # Images from carousel
    images = [img['src'] for img in soup.find_all('img', {'class': 'photo'})]

    return {
        'car_id': car_id,
        'price': price,
        'year': year,
        'mileage': mileage,
        'images': images
    }
```

#### Technical Inspection Card

```python
def get_technical_card(car_id):
    """
    Fetch Encar inspection report.

    API: GET https://fem.encar.com/carsisters/inspect/check/{car_id}

    Returns:
        dict: Inspection details (damage, repairs, accidents)
    """
    api_url = f"https://fem.encar.com/carsisters/inspect/check/{car_id}"
    response = requests.get(api_url, headers={
        'User-Agent': 'Mozilla/5.0...',
        'Referer': 'https://fem.encar.com/'
    })

    if response.status_code == 200:
        return response.json()
    return None
```

#### Insurance History

```python
def get_insurance_total(car_id):
    """
    Calculate total insurance claims.

    API: GET https://fem.encar.com/carsisters/inspect/insurance/{car_id}

    Returns:
        int: Total insurance payment amount (KRW)
    """
    api_url = f"https://fem.encar.com/carsisters/inspect/insurance/{car_id}"
    response = requests.get(api_url)

    data = response.json()
    total_insurance = sum(claim['amount'] for claim in data.get('claims', []))

    return total_insurance
```

### 2. KBChaCha Integration

**Base URL**: `https://kbchachacha.com/`
**Method**: HTML Scraping (no public API)

```python
def parse_kbchacha(url):
    """Parse KBChaCha car listing"""
    response = requests.get(url)
    soup = BeautifulSoup(response.text, 'html.parser')

    # Price from main container
    price_text = soup.find('span', {'class': 'price'}).text
    price = int(re.sub(r'[^\d]', '', price_text))

    # Specs from info table
    info_table = soup.find('table', {'class': 'car-info'})
    rows = info_table.find_all('tr')

    specs = {}
    for row in rows:
        label = row.find('th').text.strip()
        value = row.find('td').text.strip()
        specs[label] = value

    return {
        'price': price,
        'year': int(specs.get('연식', '0')[:4]),
        'mileage': int(specs.get('주행거리', '0').replace('km', '').replace(',', '')),
        'transmission': specs.get('변속기', 'Unknown')
    }
```

### 3. KCar Integration

**Base URL**: `https://kcar.com/`
**Method**: Selenium (JavaScript-rendered content)

```python
from selenium import webdriver
from selenium.webdriver.common.by import By

def parse_kcar(url):
    """Parse KCar using Selenium for JS rendering"""
    options = webdriver.ChromeOptions()
    options.add_argument('--headless')
    options.add_argument('--no-sandbox')

    driver = webdriver.Chrome(options=options)
    driver.get(url)

    # Wait for JS to load
    time.sleep(3)

    # Extract data
    price = driver.find_element(By.CLASS_NAME, 'price').text
    year = driver.find_element(By.CLASS_NAME, 'year').text
    mileage = driver.find_element(By.CLASS_NAME, 'mileage').text

    driver.quit()

    return {
        'price': int(re.sub(r'[^\d]', '', price)),
        'year': int(year[:4]),
        'mileage': int(re.sub(r'[^\d]', '', mileage))
    }
```

### 4. Currency Rate Services

#### Russian Central Bank (CBR)

```python
def get_currency_rates():
    """
    Fetch USD, EUR, CNY rates from Russian Central Bank.

    API: GET https://www.cbr-xml-daily.ru/daily_json.js

    Returns:
        dict: {'USD': float, 'EUR': float, 'CNY': float}
    """
    try:
        response = requests.get('https://www.cbr-xml-daily.ru/daily_json.js', timeout=10)
        data = response.json()

        return {
            'USD': data['Valute']['USD']['Value'],
            'EUR': data['Valute']['EUR']['Value'],
            'CNY': data['Valute']['CNY']['Value']
        }
    except Exception as e:
        logger.error(f"Currency rate fetch failed: {e}")
        return {'USD': 90.0, 'EUR': 100.0, 'CNY': 13.0}  # Fallback rates
```

#### USDT to KRW Conversion

```python
def get_usdt_to_krw_rate():
    """
    Fetch USDT/KRW rate from cryptocurrency exchange.

    Example: Upbit API
    """
    try:
        response = requests.get('https://api.upbit.com/v1/ticker?markets=KRW-USDT')
        data = response.json()[0]
        return data['trade_price']
    except:
        return 1300.0  # Fallback ~1300 KRW per USDT
```

### 5. Customs Fee Calculator (calcus.ru)

```python
def get_customs_fees(price_usd, age, engine_volume):
    """
    POST to calcus.ru to calculate Russian customs duties.

    Args:
        price_usd: Car price in USD
        age: Car age category (0-3, 3-5, 5-7, 7+)
        engine_volume: Engine size in cc

    Returns:
        float: Total customs fee in USD
    """
    url = "https://calcus.ru/calculate/Customs"

    payload = {
        'carPrice': price_usd,
        'carAge': age,
        'engineVolume': engine_volume,
        'currency': 'USD'
    }

    headers = {
        'Content-Type': 'application/x-www-form-urlencoded',
        'User-Agent': 'Mozilla/5.0...'
    }

    try:
        response = requests.post(url, data=payload, headers=headers, timeout=15)
        result = response.json()
        return result.get('totalFee', 0)
    except Exception as e:
        logger.error(f"Customs calculation failed: {e}")
        # Fallback formula: 54% of car price + engine-based fee
        return price_usd * 0.54
```

### Rate Limiting & Error Handling

```python
import time
from functools import wraps

def rate_limit(calls=10, period=60):
    """
    Decorator to limit API calls.

    Args:
        calls: Max number of calls
        period: Time period in seconds
    """
    calls_made = []

    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            now = time.time()
            # Remove old calls outside period
            while calls_made and calls_made[0] < now - period:
                calls_made.pop(0)

            if len(calls_made) >= calls:
                sleep_time = period - (now - calls_made[0])
                logger.warning(f"Rate limit reached, sleeping {sleep_time:.2f}s")
                time.sleep(sleep_time)

            calls_made.append(time.time())
            return func(*args, **kwargs)

        return wrapper
    return decorator

@rate_limit(calls=20, period=60)
def fetch_car_data(url):
    """Limited to 20 calls per minute"""
    return requests.get(url)
```

### Retry Logic

```python
from requests.adapters import HTTPAdapter
from requests.packages.urllib3.util.retry import Retry

def requests_retry_session(
    retries=3,
    backoff_factor=0.3,
    status_forcelist=(500, 502, 504),
    session=None
):
    """
    Create requests session with automatic retries.

    Usage:
        session = requests_retry_session()
        response = session.get('https://example.com')
    """
    session = session or requests.Session()
    retry = Retry(
        total=retries,
        read=retries,
        connect=retries,
        backoff_factor=backoff_factor,
        status_forcelist=status_forcelist,
    )
    adapter = HTTPAdapter(max_retries=retry)
    session.mount('http://', adapter)
    session.mount('https://', adapter)
    return session
```

---

## 🐍 Python & PEP Standards

### PEP 8: Style Guide

#### Naming Conventions

```python
# ✅ GOOD: Follow PEP 8
class OrderManager:                    # CapWords for classes
    MAX_FREE_CALCULATIONS = 5          # UPPER_SNAKE_CASE for constants

    def __init__(self):
        self.user_orders = {}          # snake_case for variables
        self._internal_cache = {}      # Leading underscore for internal

    def calculate_total_cost(self, price, fees):  # snake_case for functions
        """Calculate total with fees."""
        total_amount = price + fees    # snake_case for local vars
        return total_amount

# ❌ BAD: Inconsistent naming
class orderManager:                    # Should be CapWords
    maxFreeCalculations = 5            # Should be UPPER_SNAKE_CASE

    def CalculateTotalCost(self, Price, Fees):  # Should be snake_case
        TotalAmount = Price + Fees     # Should be snake_case
        return TotalAmount
```

#### Import Ordering (PEP 8)

```python
# 1. Standard library imports
import os
import sys
import logging
from datetime import datetime

# 2. Related third-party imports
import requests
from bs4 import BeautifulSoup
import telebot
from telebot import types

# 3. Local application imports
from database import connect_db, add_order
from utils import calculate_age, format_number
```

#### Line Length & Formatting

```python
# ✅ GOOD: Max 79-100 characters per line
def calculate_import_cost(
    car_price: float,
    shipping_fee: float,
    customs_duty: float,
    insurance: float
) -> float:
    """Calculate total import cost."""
    return car_price + shipping_fee + customs_duty + insurance

# ✅ GOOD: Break long strings
error_message = (
    "Failed to calculate customs fees. "
    "Please check the car age and engine volume. "
    "Contact support if the issue persists."
)

# ❌ BAD: Line too long
def calculate_import_cost(car_price: float, shipping_fee: float, customs_duty: float, insurance: float, handling_fee: float, documentation_fee: float) -> float:
    return car_price + shipping_fee + customs_duty + insurance + handling_fee + documentation_fee
```

### PEP 257: Docstring Conventions

```python
def calculate_cost(url: str, user_type: str, message) -> dict:
    """
    Calculate total car import cost from Korean marketplace.

    This function parses the car listing URL, extracts vehicle details,
    fetches current exchange rates, calculates customs fees based on
    user type (individual vs legal entity), and returns a comprehensive
    cost breakdown in multiple currencies.

    Args:
        url (str): Encar/KBChaCha/KCar car listing URL
        user_type (str): "физ" for individual, "юр" for legal entity
        message: Telegram message object for user context

    Returns:
        dict: Cost breakdown containing:
            - total_usd (float): Total cost in USD
            - total_krw (float): Total cost in KRW
            - total_rub (float): Total cost in RUB
            - breakdown (dict): Itemized costs
            - car_info (dict): Vehicle details

    Raises:
        ValueError: If URL format is invalid or unsupported
        RequestException: If external API calls fail

    Example:
        >>> result = calculate_cost(
        ...     "https://fem.encar.com/cars/detail/12345",
        ...     "физ",
        ...     message_obj
        ... )
        >>> print(result['total_usd'])
        25000.0

    Note:
        - Requires active internet connection for API calls
        - Exchange rates cached for 1 hour
        - Customs fees vary by car age and engine volume
    """
    # Implementation
    pass
```

### PEP 484: Type Hints

```python
from typing import List, Dict, Optional, Tuple, Union
from datetime import datetime

# ✅ GOOD: Comprehensive type hints
def get_orders(
    user_id: int,
    status_filter: Optional[str] = None,
    limit: int = 10
) -> List[Dict[str, Union[int, str, float]]]:
    """
    Fetch user orders with optional filtering.

    Args:
        user_id: Telegram user ID
        status_filter: Optional status to filter by
        limit: Maximum number of orders to return

    Returns:
        List of order dictionaries
    """
    conn = connect_db()
    cursor = conn.cursor()

    query: str = "SELECT * FROM orders WHERE user_id = %s"
    params: Tuple = (user_id,)

    if status_filter:
        query += " AND status = %s"
        params += (status_filter,)

    query += " LIMIT %s"
    params += (limit,)

    cursor.execute(query, params)
    orders: List[Dict] = cursor.fetchall()
    conn.close()

    return orders

# Type aliases for clarity
UserId = int
CarData = Dict[str, Union[int, str, List[str]]]
ExchangeRates = Dict[str, float]

def process_car_data(
    user_id: UserId,
    car_data: CarData,
    rates: ExchangeRates
) -> Optional[Dict[str, float]]:
    """Process car data with type aliases for readability."""
    pass
```

### Code Organization Best Practices

```python
# ✅ GOOD: Organized, single responsibility
class OrderService:
    """Handle all order-related operations."""

    def __init__(self, db_connection):
        self.db = db_connection

    def create_order(self, user_id: int, car_data: dict) -> int:
        """Create new order."""
        pass

    def get_order(self, order_id: int) -> Optional[dict]:
        """Retrieve order by ID."""
        pass

    def update_status(self, order_id: int, new_status: str) -> bool:
        """Update order status."""
        pass

class CurrencyService:
    """Handle currency conversions."""

    def __init__(self):
        self.cache = {}
        self.cache_expiry = 3600  # 1 hour

    def get_rate(self, from_currency: str, to_currency: str) -> float:
        """Get exchange rate with caching."""
        pass

# ❌ BAD: Mixed responsibilities in one file
# Don't put everything in main.py (current implementation)
```

---

## 🔒 Security Principles & Guidelines

### 1. SQL Injection Prevention

```python
# ✅ GOOD: Parameterized queries
def get_user_orders(user_id: int, status: str) -> List[dict]:
    cursor.execute(
        "SELECT * FROM orders WHERE user_id = %s AND status = %s",
        (user_id, status)
    )
    return cursor.fetchall()

# ❌ BAD: String interpolation (NEVER DO THIS!)
def bad_get_user_orders(user_id: int, status: str):
    query = f"SELECT * FROM orders WHERE user_id = {user_id} AND status = '{status}'"
    cursor.execute(query)  # Vulnerable to SQL injection!

# ❌ BAD: String concatenation
def bad_get_user_orders2(user_id: int):
    query = "SELECT * FROM orders WHERE user_id = " + str(user_id)
    cursor.execute(query)  # Still vulnerable!
```

### 2. Environment Variable Security

```python
import os
from dotenv import load_dotenv

# ✅ GOOD: Load from environment
load_dotenv()
BOT_TOKEN = os.getenv("BOT_TOKEN")
DATABASE_URL = os.getenv("DATABASE_URL")

if not BOT_TOKEN or not DATABASE_URL:
    raise ValueError("Missing required environment variables")

# ❌ BAD: Hardcoded secrets (NEVER!)
BOT_TOKEN = "1234567890:ABCdefGHIjklMNOpqrsTUVwxyz"  # Don't do this!
DATABASE_URL = "postgresql://user:password@host/db"  # Don't do this!

# .env file (not in version control):
# BOT_TOKEN=your_bot_token_here
# DATABASE_URL=postgresql://user:password@host:5432/dbname

# .gitignore:
# .env
# *.env
# .env.local
```

### 3. Input Validation

```python
import re
from typing import Optional

def validate_phone_number(phone: str) -> Optional[str]:
    """
    Validate and sanitize phone number.

    Args:
        phone: Raw phone input

    Returns:
        Sanitized phone or None if invalid
    """
    # Remove all non-digit characters
    digits = re.sub(r'\D', '', phone)

    # Check length (Russian numbers: 10-11 digits)
    if len(digits) < 10 or len(digits) > 11:
        return None

    # Normalize to +7XXXXXXXXXX format
    if len(digits) == 10:
        return f"+7{digits}"
    elif digits[0] == '7':
        return f"+{digits}"
    elif digits[0] == '8':
        return f"+7{digits[1:]}"

    return None

def validate_url(url: str) -> bool:
    """Validate car marketplace URL."""
    allowed_domains = [
        'fem.encar.com',
        'kbchachacha.com',
        'kcar.com'
    ]

    return any(domain in url for domain in allowed_domains)

# ✅ GOOD: Validate before processing
@bot.message_handler(func=lambda msg: True)
def handle_message(message):
    user_input = message.text.strip()

    # Validate URL before parsing
    if 'http' in user_input:
        if not validate_url(user_input):
            bot.send_message(
                message.chat.id,
                "❌ Неподдерживаемый URL. Используйте Encar, KBChaCha или KCar."
            )
            return

        # Safe to process
        calculate_cost(user_input, message)
```

### 4. Authentication & Authorization

```python
# ✅ GOOD: Manager verification
MANAGERS = [
    int(os.getenv("MANAGER_ID_1")),
    int(os.getenv("MANAGER_ID_2"))
]

def is_manager(user_id: int) -> bool:
    """Check if user is authorized manager."""
    return user_id in MANAGERS

def manager_only(handler_func):
    """Decorator for manager-only commands."""
    def wrapper(message):
        if not is_manager(message.from_user.id):
            bot.send_message(
                message.chat.id,
                "❌ У вас нет доступа к этой команде."
            )
            logger.warning(
                f"Unauthorized access attempt by user {message.from_user.id}"
            )
            return
        return handler_func(message)
    return wrapper

@bot.message_handler(commands=['users'])
@manager_only
def list_all_users(message):
    """Manager-only command to view all users."""
    users = get_all_users()
    # ... send user list
```

### 5. Data Sanitization

```python
import html
import re

def sanitize_user_input(text: str) -> str:
    """
    Sanitize user input to prevent XSS and injection.

    Args:
        text: Raw user input

    Returns:
        Sanitized text safe for storage and display
    """
    # HTML escape
    text = html.escape(text)

    # Remove potential script tags
    text = re.sub(r'<script[^>]*>.*?</script>', '', text, flags=re.IGNORECASE | re.DOTALL)

    # Limit length
    max_length = 500
    if len(text) > max_length:
        text = text[:max_length]

    return text.strip()

def sanitize_filename(filename: str) -> str:
    """Sanitize filename to prevent path traversal."""
    # Remove path separators
    filename = filename.replace('/', '').replace('\\', '')

    # Remove parent directory references
    filename = filename.replace('..', '')

    # Allow only alphanumeric, underscore, hyphen, and dot
    filename = re.sub(r'[^\w\-.]', '', filename)

    return filename
```

### 6. Secure API Requests

```python
import requests
from requests.adapters import HTTPAdapter
from requests.packages.urllib3.util.ssl_ import create_urllib3_context

# ✅ GOOD: Secure requests with verification
def fetch_car_data(url: str) -> dict:
    """Fetch car data with security measures."""
    # Verify SSL certificates
    response = requests.get(
        url,
        verify=True,  # Verify SSL certificates
        timeout=15,   # Prevent hanging requests
        headers={
            'User-Agent': 'Mozilla/5.0 (compatible; 82AutoBot/1.0)',
            'Accept': 'application/json'
        }
    )

    # Check status
    response.raise_for_status()

    return response.json()

# ❌ BAD: Disabling SSL verification
def insecure_fetch(url: str):
    response = requests.get(url, verify=False)  # DON'T DO THIS!
    return response.json()
```

### 7. Rate Limiting & DoS Protection

```python
from collections import defaultdict
import time

class RateLimiter:
    """Rate limiter to prevent abuse."""

    def __init__(self, max_calls: int = 10, period: int = 60):
        self.max_calls = max_calls
        self.period = period
        self.calls = defaultdict(list)

    def is_allowed(self, user_id: int) -> bool:
        """Check if user is within rate limits."""
        now = time.time()
        user_calls = self.calls[user_id]

        # Remove old calls outside the period
        user_calls = [call for call in user_calls if call > now - self.period]
        self.calls[user_id] = user_calls

        if len(user_calls) >= self.max_calls:
            return False

        user_calls.append(now)
        return True

# Global rate limiter
rate_limiter = RateLimiter(max_calls=10, period=60)

@bot.message_handler(func=lambda msg: 'http' in msg.text)
def handle_car_link(message):
    """Handle car link with rate limiting."""
    user_id = message.from_user.id

    if not rate_limiter.is_allowed(user_id):
        bot.send_message(
            message.chat.id,
            "⏳ Вы отправили слишком много запросов. Подождите минуту."
        )
        return

    # Process request
    calculate_cost(message.text, message)
```

### 8. Logging Best Practices

```python
import logging
from logging.handlers import RotatingFileHandler

# ✅ GOOD: Structured logging without sensitive data
def setup_logging():
    """Configure secure logging."""
    logger = logging.getLogger('82auto_bot')
    logger.setLevel(logging.INFO)

    # File handler with rotation
    file_handler = RotatingFileHandler(
        'bot.log',
        maxBytes=10*1024*1024,  # 10MB
        backupCount=5
    )

    formatter = logging.Formatter(
        '%(asctime)s - %(name)s - %(levelname)s - %(message)s'
    )
    file_handler.setFormatter(formatter)
    logger.addHandler(file_handler)

    return logger

logger = setup_logging()

# ✅ GOOD: Log without sensitive data
def process_order(user_id, phone, full_name):
    logger.info(f"Processing order for user {user_id}")  # ✅ No sensitive data
    # ... process order

# ❌ BAD: Logging sensitive data
def bad_process_order(user_id, phone, full_name):
    logger.info(f"Order: {user_id}, phone: {phone}, name: {full_name}")  # ❌ Exposes PII
```

---

## 🧪 Testing Guidelines

### Test Structure

```
tests/
├── __init__.py
├── conftest.py              # Pytest fixtures
├── test_database.py         # Database operations
├── test_handlers.py         # Bot handlers
├── test_utils.py            # Utility functions
├── test_api_integration.py  # External API calls
└── test_security.py         # Security validations
```

### Unit Testing Examples

```python
# tests/test_utils.py
import pytest
from utils import calculate_age, clean_number, format_number

class TestCalculateAge:
    """Test age calculation function."""

    def test_new_car(self):
        """Cars 0-3 years old."""
        assert calculate_age(2024, 2023) == "0-3"
        assert calculate_age(2024, 2021) == "0-3"

    def test_medium_age(self):
        """Cars 3-5 years old."""
        assert calculate_age(2024, 2020) == "3-5"
        assert calculate_age(2024, 2019) == "3-5"

    def test_old_car(self):
        """Cars 7+ years old."""
        assert calculate_age(2024, 2015) == "7+"
        assert calculate_age(2024, 2000) == "7+"

    @pytest.mark.parametrize("current,year,expected", [
        (2024, 2024, "0-3"),
        (2024, 2021, "0-3"),
        (2024, 2019, "3-5"),
        (2024, 2018, "5-7"),
        (2024, 2015, "7+"),
    ])
    def test_age_categories(self, current, year, expected):
        """Test all age categories with parametrize."""
        assert calculate_age(current, year) == expected

class TestNumberFormatting:
    """Test number formatting utilities."""

    def test_clean_number(self):
        """Test removing commas and spaces."""
        assert clean_number("1,234,567") == 1234567
        assert clean_number("1 234 567") == 1234567
        assert clean_number("1234567") == 1234567

    def test_format_number(self):
        """Test number formatting with locale."""
        assert format_number(1234567) == "1,234,567"
        assert format_number(1000) == "1,000"
```

### Database Testing

```python
# tests/test_database.py
import pytest
import psycopg2
from database import add_order, get_orders, update_order_status_in_db

@pytest.fixture
def db_connection():
    """Create test database connection."""
    conn = psycopg2.connect("postgresql://test:test@localhost/test_82auto")
    yield conn
    conn.close()

@pytest.fixture
def clean_database(db_connection):
    """Clean database before each test."""
    cursor = db_connection.cursor()
    cursor.execute("TRUNCATE TABLE orders CASCADE")
    db_connection.commit()
    yield
    cursor.execute("TRUNCATE TABLE orders CASCADE")
    db_connection.commit()

class TestOrderOperations:
    """Test order CRUD operations."""

    def test_add_order(self, clean_database):
        """Test adding new order."""
        order_id = add_order(
            user_id=12345,
            car_id="car_001",
            title="Hyundai Sonata",
            price=25000000,
            link="https://fem.encar.com/cars/detail/12345"
        )

        assert order_id is not None
        assert isinstance(order_id, int)

    def test_get_orders(self, clean_database):
        """Test retrieving user orders."""
        # Add test orders
        add_order(12345, "car_001", "Hyundai", 25000000, "http://...")
        add_order(12345, "car_002", "Kia", 20000000, "http://...")

        orders = get_orders(12345)

        assert len(orders) == 2
        assert orders[0]['title'] == "Hyundai"
        assert orders[1]['title'] == "Kia"

    def test_update_status(self, clean_database):
        """Test updating order status."""
        order_id = add_order(12345, "car_001", "Hyundai", 25000000, "http://...")

        result = update_order_status_in_db(order_id, "🚗 Авто выкуплен (на базе)")

        assert result is True

        orders = get_orders(12345)
        assert orders[0]['status'] == "🚗 Авто выкуплен (на базе)"
```

### Handler Testing with Mocks

```python
# tests/test_handlers.py
import pytest
from unittest.mock import Mock, patch, MagicMock
from main import send_welcome, calculate_cost, order_car

class TestBotHandlers:
    """Test Telegram bot handlers."""

    @patch('main.bot')
    def test_send_welcome(self, mock_bot):
        """Test /start command handler."""
        # Create mock message
        message = Mock()
        message.from_user.id = 12345
        message.from_user.username = "testuser"
        message.from_user.first_name = "Test"
        message.chat.id = 12345

        # Call handler
        send_welcome(message)

        # Assert bot.send_message was called
        mock_bot.send_message.assert_called_once()

        # Check message contains welcome text
        call_args = mock_bot.send_message.call_args
        assert "Добро пожаловать" in call_args[1]['text']

    @patch('main.get_car_info')
    @patch('main.bot')
    def test_calculate_cost(self, mock_bot, mock_get_car_info):
        """Test car cost calculation."""
        # Mock car info
        mock_get_car_info.return_value = {
            'price': 25000000,
            'year': 2020,
            'mileage': 30000,
            'engine_volume': 2000,
            'transmission': 'Автомат'
        }

        message = Mock()
        message.chat.id = 12345
        message.from_user.id = 12345

        url = "https://fem.encar.com/cars/detail/12345"

        # Call function
        calculate_cost(url, message, "физ")

        # Assert get_car_info was called with URL
        mock_get_car_info.assert_called_once_with(url)

        # Assert message was sent to user
        mock_bot.send_message.assert_called()
```

### API Integration Testing

```python
# tests/test_api_integration.py
import pytest
import responses
from main import get_currency_rates, get_car_info

class TestCurrencyAPI:
    """Test currency rate API integration."""

    @responses.activate
    def test_get_currency_rates_success(self):
        """Test successful currency rate fetch."""
        # Mock API response
        responses.add(
            responses.GET,
            'https://www.cbr-xml-daily.ru/daily_json.js',
            json={
                'Valute': {
                    'USD': {'Value': 90.5},
                    'EUR': {'Value': 100.2},
                    'CNY': {'Value': 13.1}
                }
            },
            status=200
        )

        rates = get_currency_rates()

        assert rates['USD'] == 90.5
        assert rates['EUR'] == 100.2
        assert rates['CNY'] == 13.1

    @responses.activate
    def test_get_currency_rates_failure(self):
        """Test fallback on API failure."""
        responses.add(
            responses.GET,
            'https://www.cbr-xml-daily.ru/daily_json.js',
            status=500
        )

        rates = get_currency_rates()

        # Should return fallback rates
        assert rates['USD'] > 0
        assert rates['EUR'] > 0

class TestEncarAPI:
    """Test Encar API integration."""

    @pytest.mark.integration
    def test_get_car_info_real_api(self):
        """Integration test with real Encar API."""
        # This requires internet connection
        url = "https://fem.encar.com/cars/detail/35489571"

        car_info = get_car_info(url)

        assert 'price' in car_info
        assert 'year' in car_info
        assert car_info['price'] > 0
```

### Running Tests

```bash
# Install pytest
pip install pytest pytest-cov pytest-mock

# Run all tests
pytest

# Run with coverage
pytest --cov=. --cov-report=html

# Run specific test file
pytest tests/test_utils.py

# Run specific test class
pytest tests/test_database.py::TestOrderOperations

# Run with verbose output
pytest -v

# Run only integration tests
pytest -m integration

# Run excluding integration tests
pytest -m "not integration"
```

### Test Coverage Goals

- **Unit Tests**: 80%+ coverage
- **Integration Tests**: Critical paths (order flow, payment, status updates)
- **Handler Tests**: All bot commands and callbacks
- **Security Tests**: Input validation, SQL injection prevention

---

## 🔄 Refactoring Recommendations

### Current Issues & Proposed Solutions

#### 1. **Monolithic main.py (2,870 lines)**

**Problem**: All code in one file makes it hard to maintain and test.

**Solution**: Split into modular structure:

```
bot/
├── __init__.py
├── handlers/
│   ├── __init__.py
│   ├── commands.py          # /start, /orders, /users
│   ├── callbacks.py         # Callback query handlers
│   ├── messages.py          # Text message handlers
│   └── faq.py               # FAQ system handlers
├── services/
│   ├── __init__.py
│   ├── car_parser.py        # Encar, KBChaCha, KCar parsing
│   ├── cost_calculator.py   # Cost calculation logic
│   ├── currency.py          # Exchange rate services
│   └── order_service.py     # Order management
├── models/
│   ├── __init__.py
│   ├── order.py             # Order model
│   ├── user.py              # User model
│   └── car.py               # Car model
├── database/
│   ├── __init__.py
│   ├── connection.py        # DB connection management
│   └── repositories.py      # Data access layer
├── utils/
│   ├── __init__.py
│   ├── validators.py        # Input validation
│   ├── formatters.py        # Number/text formatting
│   └── constants.py         # Constants and config
└── main.py                  # Entry point (minimal)
```

**Example refactored handler**:

```python
# handlers/commands.py
from telebot import TeleBot
from services.order_service import OrderService
from database.repositories import OrderRepository

class CommandHandlers:
    """Handle bot commands."""

    def __init__(self, bot: TeleBot, order_service: OrderService):
        self.bot = bot
        self.order_service = order_service

    def register_handlers(self):
        """Register all command handlers."""
        self.bot.message_handler(commands=['start'])(self.start_command)
        self.bot.message_handler(commands=['orders'])(self.orders_command)

    def start_command(self, message):
        """Handle /start command."""
        user_id = message.from_user.id
        # Add user to database
        # Send welcome message
        pass

    def orders_command(self, message):
        """Handle /orders command."""
        user_id = message.from_user.id
        orders = self.order_service.get_user_orders(user_id)
        # Format and send orders
        pass
```

#### 2. **Global State Variables**

**Problem**: Global dictionaries don't scale and cause issues in concurrent environments.

**Solution**: Use Redis or class-based state management:

```python
# services/state_manager.py
import redis
import json
from typing import Any, Optional

class StateManager:
    """Manage user session state with Redis."""

    def __init__(self, redis_url: str):
        self.redis = redis.from_url(redis_url)
        self.default_ttl = 3600  # 1 hour

    def set(self, user_id: int, key: str, value: Any, ttl: Optional[int] = None):
        """Set user state value."""
        redis_key = f"user:{user_id}:{key}"
        self.redis.setex(
            redis_key,
            ttl or self.default_ttl,
            json.dumps(value)
        )

    def get(self, user_id: int, key: str) -> Optional[Any]:
        """Get user state value."""
        redis_key = f"user:{user_id}:{key}"
        value = self.redis.get(redis_key)
        return json.loads(value) if value else None

    def delete(self, user_id: int, key: str):
        """Delete user state value."""
        redis_key = f"user:{user_id}:{key}"
        self.redis.delete(redis_key)

# Usage
state = StateManager(os.getenv("REDIS_URL"))

# Instead of: car_data[user_id] = {...}
state.set(user_id, "car_data", car_info)

# Instead of: car_info = car_data.get(user_id)
car_info = state.get(user_id, "car_data")
```

#### 3. **Synchronous HTTP Requests**

**Problem**: Blocking requests slow down the bot.

**Solution**: Use async/await with aiohttp:

```python
# services/car_parser.py (async version)
import aiohttp
from typing import Dict

class AsyncCarParser:
    """Async car information parser."""

    async def get_car_info(self, url: str) -> Dict:
        """Fetch car info asynchronously."""
        async with aiohttp.ClientSession() as session:
            async with session.get(url, timeout=15) as response:
                html = await response.text()
                return self._parse_html(html)

    async def get_multiple_cars(self, urls: list) -> list:
        """Fetch multiple cars concurrently."""
        async with aiohttp.ClientSession() as session:
            tasks = [self._fetch_car(session, url) for url in urls]
            return await asyncio.gather(*tasks)

    async def _fetch_car(self, session, url: str):
        """Fetch single car with session."""
        async with session.get(url) as response:
            html = await response.text()
            return self._parse_html(html)
```

#### 4. **Hardcoded Manager IDs**

**Problem**: Manager list hardcoded in main.py.

**Solution**: Store in database with role-based access:

```python
# models/user.py
from enum import Enum

class UserRole(Enum):
    CUSTOMER = "customer"
    MANAGER = "manager"
    ADMIN = "admin"

# database/repositories.py
def get_user_role(user_id: int) -> UserRole:
    """Get user role from database."""
    conn = connect_db()
    cursor = conn.cursor()
    cursor.execute(
        "SELECT role FROM users WHERE telegram_id = %s",
        (user_id,)
    )
    result = cursor.fetchone()
    conn.close()

    return UserRole(result[0]) if result else UserRole.CUSTOMER

# utils/decorators.py
def require_role(required_role: UserRole):
    """Decorator to check user role."""
    def decorator(handler_func):
        def wrapper(message):
            user_id = message.from_user.id
            user_role = get_user_role(user_id)

            if user_role.value < required_role.value:
                bot.send_message(message.chat.id, "Access denied")
                return

            return handler_func(message)
        return wrapper
    return decorator

# Usage
@bot.message_handler(commands=['users'])
@require_role(UserRole.MANAGER)
def list_users(message):
    """Manager-only command."""
    pass
```

#### 5. **Long Functions (500+ lines)**

**Problem**: `calculate_cost()` is too complex.

**Solution**: Break into smaller functions:

```python
# services/cost_calculator.py
class CostCalculator:
    """Calculate car import costs."""

    def __init__(self, currency_service, customs_service):
        self.currency = currency_service
        self.customs = customs_service

    def calculate_total(
        self,
        car_price: float,
        user_type: str,
        car_age: str,
        engine_volume: int
    ) -> Dict[str, float]:
        """Calculate total import cost."""
        # Break into smaller steps
        krw_price = car_price
        usd_price = self._convert_krw_to_usd(krw_price)

        shipping = self._calculate_shipping(user_type)
        customs = self._calculate_customs(usd_price, car_age, engine_volume)
        insurance = self._calculate_insurance(usd_price)

        total_usd = usd_price + shipping + customs + insurance
        total_rub = self._convert_usd_to_rub(total_usd)

        return {
            'total_usd': total_usd,
            'total_rub': total_rub,
            'breakdown': {
                'car_price': usd_price,
                'shipping': shipping,
                'customs': customs,
                'insurance': insurance
            }
        }

    def _convert_krw_to_usd(self, krw: float) -> float:
        """Convert KRW to USD."""
        rate = self.currency.get_rate('KRW', 'USD')
        return krw / rate

    def _calculate_shipping(self, user_type: str) -> float:
        """Calculate shipping fee based on user type."""
        return 800.0 if user_type == "физ" else 750.0

    def _calculate_customs(
        self,
        price: float,
        age: str,
        engine: int
    ) -> float:
        """Calculate customs duty."""
        return self.customs.get_fee(price, age, engine)

    def _calculate_insurance(self, price: float) -> float:
        """Calculate insurance fee."""
        return price * 0.02  # 2% of car price
```

#### 6. **Configuration Management**

**Problem**: Constants scattered throughout code.

**Solution**: Centralized configuration:

```python
# config.py
import os
from dataclasses import dataclass
from typing import List

@dataclass
class BotConfig:
    """Bot configuration."""
    token: str
    managers: List[int]
    channel_username: str
    max_free_calculations: int = 5

    @classmethod
    def from_env(cls):
        """Load configuration from environment."""
        return cls(
            token=os.getenv("BOT_TOKEN"),
            managers=[
                int(os.getenv("MANAGER_ID_1")),
                int(os.getenv("MANAGER_ID_2"))
            ],
            channel_username=os.getenv("CHANNEL_USERNAME", "autofromkorea82"),
            max_free_calculations=int(os.getenv("MAX_FREE_CALCULATIONS", "5"))
        )

@dataclass
class DatabaseConfig:
    """Database configuration."""
    url: str
    pool_size: int = 10
    max_overflow: int = 20

    @classmethod
    def from_env(cls):
        return cls(
            url=os.getenv("DATABASE_URL"),
            pool_size=int(os.getenv("DB_POOL_SIZE", "10")),
            max_overflow=int(os.getenv("DB_MAX_OVERFLOW", "20"))
        )

@dataclass
class AppConfig:
    """Application configuration."""
    bot: BotConfig
    database: DatabaseConfig

    @classmethod
    def load(cls):
        """Load all configuration."""
        return cls(
            bot=BotConfig.from_env(),
            database=DatabaseConfig.from_env()
        )

# main.py
config = AppConfig.load()
bot = TeleBot(config.bot.token)
```

---

## 🐛 Common Issues & Solutions

### 1. Webhook vs Polling Conflicts

**Symptom**: Bot doesn't respond to messages, webhook deletion loop runs continuously.

**Cause**: Bot switches between webhook and polling modes.

**Solution**:

```python
# Clear webhook once at startup
bot.remove_webhook()
time.sleep(1)

# Then start polling
bot.polling(none_stop=True, interval=1, timeout=30)

# Don't run webhook deletion in background thread (remove lines 2851-2860 in main.py)
```

### 2. Database Connection Errors

**Symptom**: `psycopg2.OperationalError: could not connect to server`

**Cause**: Connection pool exhaustion or incorrect DATABASE_URL.

**Solutions**:

```python
# 1. Always close connections
def safe_db_operation():
    conn = connect_db()
    try:
        cursor = conn.cursor()
        cursor.execute("SELECT * FROM orders")
        return cursor.fetchall()
    finally:
        conn.close()  # Always close!

# 2. Use connection pooling
from psycopg2 import pool

db_pool = pool.SimpleConnectionPool(
    minconn=1,
    maxconn=10,
    dsn=os.getenv("DATABASE_URL")
)

def get_connection():
    return db_pool.getconn()

def release_connection(conn):
    db_pool.putconn(conn)

# 3. Check DATABASE_URL format
# Heroku: postgres:// should be postgresql://
DATABASE_URL = os.getenv("DATABASE_URL")
if DATABASE_URL.startswith("postgres://"):
    DATABASE_URL = DATABASE_URL.replace("postgres://", "postgresql://", 1)
```

### 3. Rate Limiting on External APIs

**Symptom**: Encar/KBChaCha returns 429 Too Many Requests.

**Solution**:

```python
import time
from functools import wraps

# Add exponential backoff
def retry_with_backoff(retries=3, backoff_factor=2):
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            for attempt in range(retries):
                try:
                    return func(*args, **kwargs)
                except requests.exceptions.HTTPError as e:
                    if e.response.status_code == 429:
                        if attempt < retries - 1:
                            wait_time = backoff_factor ** attempt
                            logger.warning(f"Rate limited, waiting {wait_time}s")
                            time.sleep(wait_time)
                        else:
                            raise
                    else:
                        raise
        return wrapper
    return decorator

@retry_with_backoff(retries=3)
def fetch_car_data(url):
    response = requests.get(url)
    response.raise_for_status()
    return response.json()
```

### 4. Parsing Failures from Marketplaces

**Symptom**: KeyError or AttributeError when parsing car info.

**Cause**: Marketplace changed HTML structure.

**Solution**:

```python
def safe_get_car_info(url: str) -> Optional[Dict]:
    """Safely parse car info with error handling."""
    try:
        response = requests.get(url, timeout=15)
        response.raise_for_status()
        soup = BeautifulSoup(response.text, 'html.parser')

        # Safe extraction with defaults
        car_info = {
            'price': safe_extract_price(soup),
            'year': safe_extract_year(soup),
            'mileage': safe_extract_mileage(soup),
            'engine_volume': safe_extract_engine(soup),
            'transmission': safe_extract_transmission(soup),
            'images': safe_extract_images(soup)
        }

        # Validate required fields
        if not car_info['price'] or not car_info['year']:
            logger.error(f"Missing required fields in {url}")
            return None

        return car_info

    except Exception as e:
        logger.error(f"Failed to parse {url}: {e}")
        return None

def safe_extract_price(soup) -> Optional[int]:
    """Safely extract price with fallback."""
    try:
        price_elem = soup.find('strong', {'class': 'price'})
        if not price_elem:
            price_elem = soup.select_one('.car-price')  # Alternative selector

        if price_elem:
            price_text = price_elem.text.strip()
            return int(re.sub(r'[^\d]', '', price_text))
    except Exception as e:
        logger.warning(f"Price extraction failed: {e}")

    return None
```

### 5. Character Encoding Issues (Russian Text)

**Symptom**: Russian text displays as � or mojibake.

**Solution**:

```python
# Ensure UTF-8 encoding everywhere
import sys
sys.stdout.reconfigure(encoding='utf-8')

# Database connection with UTF-8
def connect_db():
    conn = psycopg2.connect(
        os.getenv("DATABASE_URL"),
        options='-c client_encoding=UTF8'
    )
    return conn

# File operations with UTF-8
with open('data.txt', 'r', encoding='utf-8') as f:
    content = f.read()

# Telegram API handles UTF-8 automatically
bot.send_message(chat_id, "Привет! 🚗")  # Works fine
```

### 6. Bot Command Registration Issues

**Symptom**: Commands don't show in Telegram autocomplete.

**Solution**:

```python
def set_bot_commands():
    """Register bot commands with Telegram."""
    commands = [
        types.BotCommand("start", "Начать работу с ботом"),
        types.BotCommand("orders", "Мои заказы"),
        types.BotCommand("my_cars", "Избранные автомобили"),
        types.BotCommand("exchange_rates", "Курсы валют"),
        types.BotCommand("help", "Помощь"),
    ]

    # Set for all users
    bot.set_my_commands(commands)

    # Set additional commands for managers
    manager_commands = commands + [
        types.BotCommand("users", "Список пользователей"),
        types.BotCommand("stats", "Статистика бота"),
    ]

    for manager_id in MANAGERS:
        bot.set_my_commands(
            manager_commands,
            scope=types.BotCommandScopeChat(manager_id)
        )

# Call at startup
set_bot_commands()
```

### 7. Memory Leaks from Global Variables

**Symptom**: Bot memory usage grows over time, eventually crashes.

**Cause**: Global dictionaries (`car_data`, `user_data`) never cleared.

**Solution**:

```python
# Add TTL-based cleanup
import time
from collections import defaultdict

class TTLDict:
    """Dictionary with time-to-live for entries."""

    def __init__(self, ttl=3600):
        self.store = {}
        self.timestamps = {}
        self.ttl = ttl

    def __setitem__(self, key, value):
        self.store[key] = value
        self.timestamps[key] = time.time()
        self._cleanup()

    def __getitem__(self, key):
        if key in self.store:
            if time.time() - self.timestamps[key] < self.ttl:
                return self.store[key]
            else:
                del self.store[key]
                del self.timestamps[key]
        raise KeyError(key)

    def get(self, key, default=None):
        try:
            return self[key]
        except KeyError:
            return default

    def _cleanup(self):
        """Remove expired entries."""
        now = time.time()
        expired = [
            k for k, t in self.timestamps.items()
            if now - t > self.ttl
        ]
        for key in expired:
            del self.store[key]
            del self.timestamps[key]

# Replace global dicts
car_data = TTLDict(ttl=3600)  # 1 hour TTL
user_data = TTLDict(ttl=3600)
```

---

## 🚀 Deployment Instructions

### Heroku Deployment (Current Setup)

#### Prerequisites

```bash
# Install Heroku CLI
brew install heroku/brew/heroku  # macOS
# or
curl https://cli-assets.heroku.com/install.sh | sh  # Linux

# Login to Heroku
heroku login
```

#### Step 1: Prepare Repository

```bash
# Ensure files are present
ls -la
# Should have: Procfile, requirements.txt, runtime.txt

# Procfile content:
worker: python3 main.py

# runtime.txt content:
python-3.10.12
```

#### Step 2: Create Heroku App

```bash
# Create new app
heroku create 82auto-bot

# Or link existing app
heroku git:remote -a 82auto-bot

# Add PostgreSQL addon
heroku addons:create heroku-postgresql:mini
```

#### Step 3: Configure Environment Variables

```bash
# Set bot token
heroku config:set BOT_TOKEN="your_bot_token_here"

# Database URL is automatically set by Heroku
# But if needed:
heroku config:set DATABASE_URL="postgresql://..."

# Set manager IDs
heroku config:set MANAGER_ID_1="728438182"
heroku config:set MANAGER_ID_2="56022406"

# Set channel username
heroku config:set CHANNEL_USERNAME="autofromkorea82"

# View all config
heroku config
```

#### Step 4: Initialize Database

```bash
# Connect to database
heroku pg:psql

# Run table creation
CREATE TABLE IF NOT EXISTS orders (...);
CREATE TABLE IF NOT EXISTS users (...);
CREATE TABLE IF NOT EXISTS subscriptions (...);
CREATE TABLE IF NOT EXISTS calculations (...);
CREATE TABLE IF NOT EXISTS logs (...);

# Or run from Python
heroku run python
>>> from database import create_tables
>>> create_tables()
>>> exit()
```

#### Step 5: Deploy

```bash
# Commit changes
git add .
git commit -m "Deploy to Heroku"

# Push to Heroku
git push heroku main

# Scale worker dyno
heroku ps:scale worker=1

# Check logs
heroku logs --tail

# Check status
heroku ps
```

#### Step 6: Monitoring

```bash
# View logs in real-time
heroku logs --tail --dyno worker

# View recent logs
heroku logs --tail -n 500

# Check dyno status
heroku ps

# Restart if needed
heroku restart

# Check PostgreSQL status
heroku pg:info
```

### Alternative: Docker Deployment

```dockerfile
# Dockerfile
FROM python:3.10-slim

# Set working directory
WORKDIR /app

# Install system dependencies
RUN apt-get update && apt-get install -y \
    gcc \
    postgresql-client \
    && rm -rf /var/lib/apt/lists/*

# Copy requirements
COPY requirements.txt .

# Install Python dependencies
RUN pip install --no-cache-dir -r requirements.txt

# Copy application code
COPY . .

# Run bot
CMD ["python", "main.py"]
```

```yaml
# docker-compose.yml
version: "3.8"

services:
  postgres:
    image: postgres:14
    environment:
      POSTGRES_DB: auto82_bot
      POSTGRES_USER: bot_user
      POSTGRES_PASSWORD: secure_password
    volumes:
      - postgres_data:/var/lib/postgresql/data
    ports:
      - "5432:5432"

  bot:
    build: .
    depends_on:
      - postgres
    environment:
      BOT_TOKEN: ${BOT_TOKEN}
      DATABASE_URL: postgresql://bot_user:secure_password@postgres:5432/auto82_bot
      MANAGER_ID_1: ${MANAGER_ID_1}
      MANAGER_ID_2: ${MANAGER_ID_2}
    restart: unless-stopped

volumes:
  postgres_data:
```

```bash
# Build and run
docker-compose up -d

# View logs
docker-compose logs -f bot

# Stop
docker-compose down
```

### Environment Variables Checklist

Create `.env` file (never commit to git):

```bash
# Bot Configuration
BOT_TOKEN=1234567890:ABCdefGHIjklMNOpqrsTUVwxyz
CHANNEL_USERNAME=autofromkorea82

# Database
DATABASE_URL=postgresql://user:password@host:5432/dbname

# Manager IDs
MANAGER_ID_1=728438182
MANAGER_ID_2=56022406

# Optional
MAX_FREE_CALCULATIONS=5
LOG_LEVEL=INFO
```

---

## 📝 Code Examples & Patterns

### Example 1: Adding a New Handler

**Task**: Add a `/stats` command for managers to view bot statistics.

```python
# Step 1: Create the handler
@bot.message_handler(commands=['stats'])
def show_stats(message):
    """
    Show bot statistics (manager only).

    Displays:
    - Total users
    - Total orders
    - Orders by status
    - Active subscriptions
    """
    # Check if user is manager
    if message.from_user.id not in MANAGERS:
        bot.send_message(
            message.chat.id,
            "❌ У вас нет доступа к этой команде."
        )
        return

    # Gather statistics
    stats = get_bot_statistics()

    # Format message
    stats_text = f"""
📊 **Статистика бота 82 AUTO**

👥 **Пользователи:**
├ Всего: {stats['total_users']}
├ Новых за 7 дней: {stats['new_users_week']}
└ Активных за 30 дней: {stats['active_users_month']}

🚗 **Заказы:**
├ Всего: {stats['total_orders']}
├ В процессе: {stats['orders_in_progress']}
└ Завершено: {stats['orders_completed']}

📈 **По статусам:**
"""

    for status, count in stats['orders_by_status'].items():
        stats_text += f"├ {status}: {count}\n"

    stats_text += f"""
💎 **Подписки:**
└ Активных: {stats['active_subscriptions']}

🔢 **Расчеты:**
└ Сегодня: {stats['calculations_today']}
"""

    bot.send_message(
        message.chat.id,
        stats_text,
        parse_mode='Markdown'
    )

# Step 2: Create the data fetching function
def get_bot_statistics() -> Dict[str, Any]:
    """Gather bot statistics from database."""
    conn = connect_db()
    cursor = conn.cursor(cursor_factory=RealDictCursor)

    # Total users
    cursor.execute("SELECT COUNT(*) as count FROM users")
    total_users = cursor.fetchone()['count']

    # New users (last 7 days)
    cursor.execute("""
        SELECT COUNT(*) as count FROM users
        WHERE created_at > NOW() - INTERVAL '7 days'
    """)
    new_users_week = cursor.fetchone()['count']

    # Active users (last 30 days)
    cursor.execute("""
        SELECT COUNT(DISTINCT telegram_id) as count FROM logs
        WHERE created_at > NOW() - INTERVAL '30 days'
    """)
    active_users_month = cursor.fetchone()['count']

    # Total orders
    cursor.execute("SELECT COUNT(*) as count FROM orders")
    total_orders = cursor.fetchone()['count']

    # Orders by status
    cursor.execute("""
        SELECT status, COUNT(*) as count
        FROM orders
        GROUP BY status
    """)
    orders_by_status = {row['status']: row['count'] for row in cursor.fetchall()}

    # Active subscriptions
    cursor.execute("""
        SELECT COUNT(*) as count FROM subscriptions
        WHERE status = TRUE
    """)
    active_subscriptions = cursor.fetchone()['count']

    # Calculations today
    cursor.execute("""
        SELECT COUNT(*) as count FROM logs
        WHERE action = 'calculate_car'
        AND DATE(created_at) = CURRENT_DATE
    """)
    calculations_today = cursor.fetchone()['count']

    conn.close()

    return {
        'total_users': total_users,
        'new_users_week': new_users_week,
        'active_users_month': active_users_month,
        'total_orders': total_orders,
        'orders_by_status': orders_by_status,
        'orders_in_progress': sum(
            count for status, count in orders_by_status.items()
            if status != '🔄 Не заказано'
        ),
        'orders_completed': orders_by_status.get('🚛 Доставка к клиенту', 0),
        'active_subscriptions': active_subscriptions,
        'calculations_today': calculations_today
    }

# Step 3: Register in bot commands
def set_bot_commands():
    manager_commands = [
        types.BotCommand("start", "Начать работу"),
        types.BotCommand("users", "Список пользователей"),
        types.BotCommand("stats", "Статистика бота"),  # Add here
    ]

    for manager_id in MANAGERS:
        bot.set_my_commands(
            manager_commands,
            scope=types.BotCommandScopeChat(manager_id)
        )
```

### Example 2: Adding a New FAQ Category

**Task**: Add "Payment Methods" category to FAQ.

```python
# Step 1: Define FAQ data structure
FAQ_DATA = {
    "deposit": {
        "title": "💰 Депозит",
        "questions": { ... }
    },
    "payment_methods": {  # New category
        "title": "💳 Способы оплаты",
        "questions": {
            "payment_1": {
                "question": "Какие способы оплаты вы принимаете?",
                "answer": """
Мы принимаем следующие способы оплаты:

1. **Банковский перевод (RUB)**
   - Переводы на российский счет
   - Комиссия банка оплачивается отправителем

2. **USDT (TRC-20/ERC-20)**
   - Криптовалютные переводы
   - Быстрый и удобный способ

3. **Наличные (RUB)**
   - Оплата при встрече в офисе
   - Только в городах присутствия

4. **Юр. лицам: безналичный расчет**
   - С НДС и без НДС
   - Полный пакет документов

📞 Для уточнения деталей свяжитесь с менеджером.
                """
            },
            "payment_2": {
                "question": "Когда нужно вносить оплату?",
                "answer": """
График оплаты:

1. **Депозит**: 10% при подтверждении заказа
2. **Основная часть**: После покупки авто (фото с базы)
3. **Остаток**: Перед отправкой во Владивосток

💡 Возможна рассрочка - обсуждается индивидуально.
                """
            }
        }
    },
    # ... other categories
}

# Step 2: Update FAQ handler (already handles dynamic categories)
@bot.callback_query_handler(func=lambda call: call.data == "faq")
def handle_faq(call):
    """Show FAQ categories."""
    markup = types.InlineKeyboardMarkup(row_width=1)

    # Automatically includes new category
    for category_id, category_data in FAQ_DATA.items():
        button = types.InlineKeyboardButton(
            text=category_data["title"],
            callback_data=f"faq_topic_{category_id}"
        )
        markup.add(button)

    markup.add(types.InlineKeyboardButton("◀️ Назад", callback_data="back_to_main"))

    bot.edit_message_text(
        "❓ **Часто задаваемые вопросы**\n\nВыберите категорию:",
        call.message.chat.id,
        call.message.message_id,
        reply_markup=markup,
        parse_mode='Markdown'
    )

# No additional code needed - the new category automatically appears!
```

### Example 3: Adding a New Car Marketplace Parser

**Task**: Add support for AutoMobile.kr marketplace.

```python
# Step 1: Add parser function
def parse_automobile_kr(url: str) -> Optional[Dict]:
    """
    Parse car information from AutoMobile.kr.

    Args:
        url: AutoMobile.kr car listing URL

    Returns:
        dict: Car information or None if parsing fails
    """
    try:
        response = requests.get(url, headers={
            'User-Agent': 'Mozilla/5.0 ...',
            'Accept-Language': 'ko-KR,ko;q=0.9,en;q=0.8'
        }, timeout=15)

        response.raise_for_status()
        soup = BeautifulSoup(response.text, 'html.parser')

        # Extract car ID from URL
        car_id = re.search(r'/cars/(\d+)', url).group(1)

        # Price
        price_elem = soup.select_one('.price-value')
        price_text = price_elem.text.strip()
        price = int(re.sub(r'[^\d]', '', price_text)) * 10000  # Convert 만원 to KRW

        # Year and month
        year_elem = soup.select_one('.spec-year')
        year_text = year_elem.text.strip()  # "2020년 12월"
        year_match = re.search(r'(\d{4})년\s*(\d{1,2})월', year_text)
        year = int(year_match.group(1))
        month = int(year_match.group(2))

        # Mileage
        mileage_elem = soup.select_one('.spec-mileage')
        mileage = int(re.sub(r'[^\d]', '', mileage_elem.text))

        # Engine volume
        engine_elem = soup.select_one('.spec-engine')
        engine_text = engine_elem.text.strip()
        engine_volume = int(re.search(r'(\d+)cc', engine_text).group(1))

        # Transmission
        trans_elem = soup.select_one('.spec-transmission')
        transmission = "Автомат" if "오토" in trans_elem.text else "Механика"

        # Images
        image_elements = soup.select('.car-gallery img')
        images = [img['src'] for img in image_elements if img.get('src')]

        # Title
        title_elem = soup.select_one('.car-title')
        title = title_elem.text.strip()

        return {
            'source': 'automobile_kr',
            'car_id': car_id,
            'title': title,
            'price': price,
            'year': year,
            'month': month,
            'mileage': mileage,
            'engine_volume': engine_volume,
            'transmission': transmission,
            'images': images[:10],  # Limit to 10 images
            'link': url
        }

    except Exception as e:
        logger.error(f"Failed to parse AutoMobile.kr URL {url}: {e}")
        return None

# Step 2: Update get_car_info() to support new marketplace
def get_car_info(url: str) -> Optional[Dict]:
    """
    Parse car information from supported marketplaces.

    Supports:
    - Encar (fem.encar.com)
    - KBChaCha (kbchachacha.com)
    - KCar (kcar.com)
    - AutoMobile.kr (automobile.kr)  # NEW!
    """
    # Determine source
    if 'encar.com' in url:
        return parse_encar(url)
    elif 'kbchachacha.com' in url:
        return parse_kbchacha(url)
    elif 'kcar.com' in url:
        return parse_kcar(url)
    elif 'automobile.kr' in url:  # Add new marketplace
        return parse_automobile_kr(url)
    else:
        logger.warning(f"Unsupported marketplace URL: {url}")
        return None

# Step 3: Update validation
def validate_url(url: str) -> bool:
    """Validate car marketplace URL."""
    allowed_domains = [
        'fem.encar.com',
        'kbchachacha.com',
        'kcar.com',
        'automobile.kr'  # Add to allowed list
    ]

    return any(domain in url for domain in allowed_domains)

# Step 4: Update user-facing text
CALCULATE_CAR_TEXT = "Рассчитать Автомобиль (Encar, KBChaCha, KCar, AutoMobile)"
```

### Example 4: Implementing Status Update Workflow

**Task**: Complete status update workflow with notifications.

```python
# Complete implementation of order status updates
@bot.callback_query_handler(func=lambda call: call.data.startswith("update_status_"))
def update_order_status(call):
    """
    Handle order status update (manager only).

    Callback data format: update_status_{order_id}
    """
    # Verify manager
    if call.from_user.id not in MANAGERS:
        bot.answer_callback_query(call.id, "❌ Access denied")
        return

    # Extract order ID
    order_id = int(call.data.split("_")[2])

    # Get order details
    order = get_order_by_id(order_id)
    if not order:
        bot.answer_callback_query(call.id, "❌ Order not found")
        return

    # Show status selection keyboard
    markup = types.InlineKeyboardMarkup(row_width=1)

    statuses = [
        ("🚗 Авто выкуплен (на базе)", "1"),
        ("🚢 Отправлен в порт г. Пусан на погрузку", "2"),
        ("🌊 В пути до Владивостока", "3"),
        ("🛃 Растаможка", "4"),
        ("📦 Загрузка на грузовик", "5"),
        ("🚛 Доставка к клиенту", "6"),
    ]

    for status_text, status_code in statuses:
        button = types.InlineKeyboardButton(
            text=status_text,
            callback_data=f"set_status_{order_id}_{status_code}"
        )
        markup.add(button)

    markup.add(types.InlineKeyboardButton(
        "◀️ Назад",
        callback_data=f"view_order_{order_id}"
    ))

    bot.edit_message_text(
        f"Изменить статус заказа #{order_id}\n\n"
        f"**Текущий статус:** {order['status']}\n\n"
        f"Выберите новый статус:",
        call.message.chat.id,
        call.message.message_id,
        reply_markup=markup,
        parse_mode='Markdown'
    )

@bot.callback_query_handler(func=lambda call: call.data.startswith("set_status_"))
def confirm_status_update(call):
    """
    Confirm and apply status update.

    Callback data format: set_status_{order_id}_{status_code}
    """
    # Verify manager
    if call.from_user.id not in MANAGERS:
        bot.answer_callback_query(call.id, "❌ Access denied")
        return

    # Parse callback data
    parts = call.data.split("_")
    order_id = int(parts[2])
    status_code = parts[3]

    # Status mapping
    STATUS_MAP = {
        "1": "🚗 Авто выкуплен (на базе)",
        "2": "🚢 Отправлен в порт г. Пусан на погрузку",
        "3": "🌊 В пути до Владивостока",
        "4": "🛃 Растаможка",
        "5": "📦 Загрузка на грузовик",
        "6": "🚛 Доставка к клиенту",
    }

    new_status = STATUS_MAP[status_code]

    # Update in database
    success = update_order_status_in_db(order_id, new_status)

    if success:
        # Get order details for notification
        order = get_order_by_id(order_id)

        # Notify customer
        notify_customer_status_change(
            user_id=order['user_id'],
            order_id=order_id,
            car_title=order['title'],
            new_status=new_status
        )

        # Log action
        log_manager_action(
            manager_id=call.from_user.id,
            action='update_order_status',
            data={
                'order_id': order_id,
                'new_status': new_status,
                'old_status': order['status']
            }
        )

        bot.answer_callback_query(
            call.id,
            f"✅ Статус обновлен: {new_status}"
        )

        # Show updated order details
        show_order_details(call, order_id)
    else:
        bot.answer_callback_query(
            call.id,
            "❌ Ошибка обновления статуса"
        )

def notify_customer_status_change(
    user_id: int,
    order_id: int,
    car_title: str,
    new_status: str
):
    """Send notification to customer about status change."""
    message_text = f"""
🔔 **Обновление статуса заказа**

**Заказ:** #{order_id}
**Автомобиль:** {car_title}

**Новый статус:** {new_status}

Отслеживайте ваш заказ через /orders
"""

    try:
        bot.send_message(
            user_id,
            message_text,
            parse_mode='Markdown'
        )
    except Exception as e:
        logger.error(f"Failed to notify user {user_id}: {e}")

def log_manager_action(manager_id: int, action: str, data: dict):
    """Log manager action to database."""
    conn = connect_db()
    cursor = conn.cursor()

    cursor.execute("""
        INSERT INTO logs (telegram_id, action, data)
        VALUES (%s, %s, %s)
    """, (manager_id, action, json.dumps(data)))

    conn.commit()
    conn.close()
```

---

## 📚 Development Workflow

### Task Breakdown Methodology

When receiving a new feature request, follow this structured approach:

**1. Understand Requirements**

```
Input: "Add subscription payment via USDT"

Questions to ask:
- What payment gateway/wallet?
- How to verify payment?
- Subscription duration?
- What happens after expiration?
- Price in USDT?
```

**2. Break Down into Sub-Tasks**

```
Main Task: Add USDT Subscription Payment
│
├── 1. Database Changes
│   ├── Add subscription_plans table
│   ├── Add payments table
│   └── Add subscription expiry field
│
├── 2. Payment Service
│   ├── Generate unique payment address/ID
│   ├── Check payment status (blockchain/API)
│   └── Verify amount received
│
├── 3. Bot Handlers
│   ├── /subscribe command
│   ├── Show subscription plans
│   ├── Generate payment instructions
│   └── Check payment status button
│
├── 4. Background Tasks
│   ├── Cron job to check pending payments
│   ├── Cron job to handle expiring subscriptions
│   └── Send renewal reminders
│
└── 5. Testing
    ├── Test payment verification
    ├── Test subscription activation
    └── Test expiration handling
```

**3. Implement in Order**

```
Priority 1: Database (foundation)
Priority 2: Payment service (core logic)
Priority 3: Bot handlers (user interface)
Priority 4: Background tasks (automation)
Priority 5: Testing (validation)
```

### Code Review Checklist

Before committing code, verify:

```markdown
## Functionality

- [ ] Feature works as intended
- [ ] All edge cases handled
- [ ] Error messages user-friendly

## Code Quality

- [ ] Follows PEP 8 style guide
- [ ] Functions have docstrings
- [ ] Type hints added
- [ ] No magic numbers (use constants)
- [ ] No duplicate code (DRY principle)

## Security

- [ ] Input validation present
- [ ] SQL queries parameterized
- [ ] No hardcoded secrets
- [ ] User authorization checked
- [ ] Rate limiting applied

## Testing

- [ ] Unit tests written
- [ ] Integration tests pass
- [ ] Manual testing completed
- [ ] Test coverage > 80%

## Database

- [ ] Migrations created
- [ ] Indexes added for queries
- [ ] Connections properly closed
- [ ] No N+1 query problems

## Performance

- [ ] No blocking operations
- [ ] Async used where appropriate
- [ ] Caching implemented
- [ ] API calls rate-limited

## Documentation

- [ ] Code comments added
- [ ] README updated if needed
- [ ] API docs updated
- [ ] Changelog entry added
```

### Vulnerability Checking Process

**Automated Tools**:

```bash
# Install security tools
pip install bandit safety

# Run Bandit (security linter)
bandit -r . -ll

# Check for known vulnerabilities
safety check

# Run pylint for code quality
pylint main.py database.py utils.py
```

**Manual Security Review**:

```python
# 1. Check for SQL injection
# ❌ BAD
query = f"SELECT * FROM users WHERE id = {user_id}"

# ✅ GOOD
cursor.execute("SELECT * FROM users WHERE id = %s", (user_id,))

# 2. Check for XSS in bot messages
# ❌ BAD
bot.send_message(chat_id, user_input)

# ✅ GOOD
bot.send_message(chat_id, html.escape(user_input))

# 3. Check for secrets in code
# ❌ BAD
API_KEY = "sk_live_1234567890"

# ✅ GOOD
API_KEY = os.getenv("API_KEY")

# 4. Check file operations
# ❌ BAD
with open(user_filename, 'w') as f:  # Path traversal risk

# ✅ GOOD
safe_filename = sanitize_filename(user_filename)
with open(os.path.join(UPLOAD_DIR, safe_filename), 'w') as f:
```

### Git Workflow Recommendations

```bash
# 1. Feature branch workflow
git checkout -b feature/usdt-payments

# 2. Make changes with clear commits
git add payments.py
git commit -m "Add USDT payment verification service

- Implement blockchain API integration
- Add payment status checking
- Include webhook handler for confirmations

Refs #42"

# 3. Keep commits atomic (one logical change per commit)
# ✅ GOOD
git commit -m "Add payment verification"
git commit -m "Add tests for payment verification"

# ❌ BAD
git commit -m "Add payment stuff and fix bugs and update readme"

# 4. Rebase before merging (keep history clean)
git fetch origin
git rebase origin/main

# 5. Merge to main
git checkout main
git merge feature/usdt-payments

# 6. Tag releases
git tag -a v1.2.0 -m "Release 1.2.0: USDT payments"
git push origin v1.2.0
```

---

## 🎓 Summary & Best Practices

### Golden Rules for 82 Auto Bot Development

1. **Security First**: Always validate input, use parameterized queries, protect secrets
2. **PEP Standards**: Follow PEP 8, PEP 257, PEP 484 religiously
3. **Break Down Tasks**: Divide complex features into small, testable units
4. **Test Everything**: Write tests before or alongside code
5. **Document Clearly**: Every function needs a docstring, every change needs a comment
6. **Handle Errors Gracefully**: Try/except with logging, user-friendly error messages
7. **Close Connections**: Always close database connections and files
8. **Rate Limit APIs**: Protect against abuse and respect external API limits
9. **Log Appropriately**: Log actions but never log sensitive data (passwords, tokens, PII)
10. **Review Before Commit**: Use the checklist above before every commit

### Quick Reference Commands

```bash
# Start bot locally
python main.py

# Run tests
pytest tests/ -v

# Check code quality
flake8 main.py
pylint main.py

# Security scan
bandit -r .

# Deploy to Heroku
git push heroku main

# View logs
heroku logs --tail

# Database backup
heroku pg:backups:capture
heroku pg:backups:download
```

### File Structure Reference

```
main.py:701-1400      - Car parsers (Encar, KBChaCha, KCar)
main.py:1401-1900     - Cost calculation engine
main.py:2501-2850     - Bot handlers
database.py:151-220   - Order operations
utils.py:31-90        - Customs fee calculator
```

### Need Help?

- **Telegram Bot API**: https://core.telegram.org/bots/api
- **pyTelegramBotAPI Docs**: https://pytba.readthedocs.io/
- **PostgreSQL Docs**: https://www.postgresql.org/docs/
- **PEP 8 Style Guide**: https://pep8.org/
- **Python Type Hints**: https://docs.python.org/3/library/typing.html

---

**Version**: 1.0
**Last Updated**: 2024
**Maintained by**: 82 Auto Development Team

---

_Remember: You're not just writing code, you're building a reliable service that helps people import their dream cars. Every line of code should reflect professionalism, security, and attention to detail._ 🚗✨
