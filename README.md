import asyncio
import logging
import random
import sqlite3
import secrets
import aiohttp

from decimal import Decimal, ROUND_HALF_UP

from aiogram import Bot, Dispatcher, F
from aiogram.filters import CommandStart
from aiogram.types import Message, CallbackQuery
from aiogram.types import InlineKeyboardMarkup, InlineKeyboardButton
from aiogram.enums import ParseMode
from aiogram.client.default import DefaultBotProperties


BOT_TOKEN = "8868297876:AAHR6MeuTwsPNmGhJKWC-2HmIvyRJtVsrtk"
CRYPTO_PAY_TOKEN = "640458:AAqOUKYIGCXta5OhMWZbDjZu6EGvUF7htbM"

SUPPORT_USERNAME = "@fhcnncns"

DB_NAME = "proxy_shop.db"

NORMAL_PRICE = Decimal("0.30")
PARSING_PRICE = Decimal("0.50")

CRYPTO_API = "https://pay.crypt.bot/api"

logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s | %(levelname)s | %(message)s"
)


db = sqlite3.connect(
    DB_NAME,
    check_same_thread=False
)

cursor = db.cursor()


cursor.execute("""
CREATE TABLE IF NOT EXISTS users (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    telegram_id INTEGER UNIQUE NOT NULL,
    username TEXT,
    first_name TEXT,
    created_at TEXT DEFAULT CURRENT_TIMESTAMP,
    last_activity TEXT DEFAULT CURRENT_TIMESTAMP
)
""")


cursor.execute("""
CREATE TABLE IF NOT EXISTS orders (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    user_id INTEGER NOT NULL,
    product TEXT,
    country TEXT,
    category TEXT,
    quantity INTEGER NOT NULL,
    amount TEXT NOT NULL,
    invoice_id TEXT,
    status TEXT DEFAULT 'pending',
    content TEXT,
    created_at TEXT DEFAULT CURRENT_TIMESTAMP
)
""")

db.commit()


def migrate_orders_table():
    cursor.execute("PRAGMA table_info(orders)")
    columns = {
        row[1]
        for row in cursor.fetchall()
    }

    migrations = {
        "product": "ALTER TABLE orders ADD COLUMN product TEXT",
        "country": "ALTER TABLE orders ADD COLUMN country TEXT",
        "category": "ALTER TABLE orders ADD COLUMN category TEXT",
        "invoice_id": "ALTER TABLE orders ADD COLUMN invoice_id TEXT",
        "status": "ALTER TABLE orders ADD COLUMN status TEXT DEFAULT 'pending'",
        "content": "ALTER TABLE orders ADD COLUMN content TEXT",
        "created_at": "ALTER TABLE orders ADD COLUMN created_at TEXT"
    }

    for column, query in migrations.items():
        if column not in columns:
            cursor.execute(query)

    db.commit()


migrate_orders_table()


bot = Bot(
    token=BOT_TOKEN,
    default=DefaultBotProperties(
        parse_mode=ParseMode.HTML
    )
)

dp = Dispatcher()

user_state = {}


COUNTRIES = {
    "russia": "🇷🇺 Россия",
    "belarus": "🇧🇾 Беларусь",
    "kazakhstan": "🇰🇿 Казахстан",
    "uzbekistan": "🇺🇿 Узбекистан",
    "kyrgyzstan": "🇰🇬 Кыргызстан",
    "tajikistan": "🇹🇯 Таджикистан",
    "turkmenistan": "🇹🇲 Туркменистан",
    "armenia": "🇦🇲 Армения",
    "azerbaijan": "🇦🇿 Азербайджан",
    "georgia": "🇬🇪 Грузия",
    "moldova": "🇲🇩 Молдова",
    "albania": "🇦🇱 Албания",
    "andorra": "🇦🇩 Андорра",
    "austria": "🇦🇹 Австрия",
    "belgium": "🇧🇪 Бельгия",
    "bosnia": "🇧🇦 Босния и Герцеговина",
    "bulgaria": "🇧🇬 Болгария",
    "croatia": "🇭🇷 Хорватия",
    "cyprus": "🇨🇾 Кипр",
    "czechia": "🇨🇿 Чехия",
    "denmark": "🇩🇰 Дания",
    "estonia": "🇪🇪 Эстония",
    "finland": "🇫🇮 Финляндия",
    "france": "🇫🇷 Франция",
    "germany": "🇩🇪 Германия",
    "greece": "🇬🇷 Греция",
    "hungary": "🇭🇺 Венгрия",
    "iceland": "🇮🇸 Исландия",
    "ireland": "🇮🇪 Ирландия",
    "italy": "🇮🇹 Италия",
    "latvia": "🇱🇻 Латвия",
    "liechtenstein": "🇱🇮 Лихтенштейн",
    "lithuania": "🇱🇹 Литва",
    "luxembourg": "🇱🇺 Люксембург",
    "malta": "🇲🇹 Мальта",
    "monaco": "🇲🇨 Монако",
    "montenegro": "🇲🇪 Черногория",
    "netherlands": "🇳🇱 Нидерланды",
    "north_macedonia": "🇲🇰 Северная Македония",
    "norway": "🇳🇴 Норвегия",
    "poland": "🇵🇱 Польша",
    "portugal": "🇵🇹 Португалия",
    "romania": "🇷🇴 Румыния",
    "serbia": "🇷🇸 Сербия",
    "slovakia": "🇸🇰 Словакия",
    "slovenia": "🇸🇮 Словения",
    "spain": "🇪🇸 Испания",
    "sweden": "🇸🇪 Швеция",
    "switzerland": "🇨🇭 Швейцария",
    "turkey": "🇹🇷 Турция",
    "uk": "🇬🇧 Великобритания",
    "usa": "🇺🇸 США",
    "canada": "🇨🇦 Канада",
    "mexico": "🇲🇽 Мексика",
    "brazil": "🇧🇷 Бразилия",
    "argentina": "🇦🇷 Аргентина",
    "chile": "🇨🇱 Чили",
    "colombia": "🇨🇴 Колумбия",
    "peru": "🇵🇪 Перу",
    "china": "🇨🇳 Китай",
    "japan": "🇯🇵 Япония",
    "south_korea": "🇰🇷 Южная Корея",
    "india": "🇮🇳 Индия",
    "indonesia": "🇮🇩 Индонезия",
    "malaysia": "🇲🇾 Малайзия",
    "singapore": "🇸🇬 Сингапур",
    "thailand": "🇹🇭 Таиланд",
    "vietnam": "🇻🇳 Вьетнам",
    "philippines": "🇵🇭 Филиппины",
    "israel": "🇮🇱 Израиль",
    "uae": "🇦🇪 ОАЭ",
    "egypt": "🇪🇬 Египет",
    "morocco": "🇲🇦 Марокко",
    "south_africa": "🇿🇦 ЮАР",
    "nigeria": "🇳🇬 Нигерия",
    "kenya": "🇰🇪 Кения",
    "australia": "🇦🇺 Австралия",
    "new_zealand": "🇳🇿 Новая Зеландия",
    "cuba": "🇨🇺 Куба"
}

COUNTRY_KEYS = list(COUNTRIES.keys())


def register_user(user):
    cursor.execute(
        """
        INSERT OR IGNORE INTO users
        (telegram_id, username, first_name)
        VALUES (?, ?, ?)
        """,
        (
            user.id,
            user.username or "",
            user.first_name or ""
        )
    )

    cursor.execute(
        """
        UPDATE users
        SET username = ?,
            first_name = ?,
            last_activity = CURRENT_TIMESTAMP
        WHERE telegram_id = ?
        """,
        (
            user.username or "",
            user.first_name or "",
            user.id
        )
    )

    db.commit()


def get_user_id(telegram_id):
    cursor.execute(
        """
        SELECT id
        FROM users
        WHERE telegram_id = ?
        """,
        (telegram_id,)
    )

    row = cursor.fetchone()

    return row[0] if row else None


def main_menu():
    return InlineKeyboardMarkup(
        inline_keyboard=[
            [
                InlineKeyboardButton(
                    text="🛒 Купить прокси",
                    callback_data="buy_proxy"
                )
            ],
            [
                InlineKeyboardButton(
                    text="👤 Профиль",
                    callback_data="profile"
                )
            ],
            [
                InlineKeyboardButton(
                    text="💬 Поддержка",
                    callback_data="support"
                )
            ]
        ]
    )


def countries_menu(page=0):
    per_page = 12

    start = page * per_page
    end = start + per_page

    items = COUNTRY_KEYS[start:end]

    rows = []

    for i in range(0, len(items), 2):
        row = []

        for country in items[i:i + 2]:
            row.append(
                InlineKeyboardButton(
                    text=COUNTRIES[country],
                    callback_data=f"country:{country}"
                )
            )

        rows.append(row)

    navigation = []

    if page > 0:
        navigation.append(
            InlineKeyboardButton(
                text="⬅️",
                callback_data=f"countries:{page - 1}"
            )
        )

    if end < len(COUNTRY_KEYS):
        navigation.append(
            InlineKeyboardButton(
                text="➡️",
                callback_data=f"countries:{page + 1}"
            )
        )

    if navigation:
        rows.append(navigation)

    rows.append([
        InlineKeyboardButton(
            text="🏠 Главное меню",
            callback_data="home"
        )
    ])

    return InlineKeyboardMarkup(
        inline_keyboard=rows
    )


def category_menu(country):
    return InlineKeyboardMarkup(
        inline_keyboard=[
            [
                InlineKeyboardButton(
                    text="🔒 Обычные — $0.30",
                    callback_data=f"category:normal:{country}"
                )
            ],
            [
                InlineKeyboardButton(
                    text="📊 Для парсинга — $0.50",
                    callback_data=f"category:parsing:{country}"
                )
            ],
            [
                InlineKeyboardButton(
                    text="◀️ Назад",
                    callback_data="buy_proxy"
                )
            ]
        ]
    )


def quantity_menu():
    return InlineKeyboardMarkup(
        inline_keyboard=[
            [
                InlineKeyboardButton(
                    text="10",
                    callback_data="qty:10"
                ),
                InlineKeyboardButton(
                    text="50",
                    callback_data="qty:50"
                ),
                InlineKeyboardButton(
                    text="100",
                    callback_data="qty:100"
                )
            ],
            [
                InlineKeyboardButton(
                    text="120",
                    callback_data="qty:120"
                ),
                InlineKeyboardButton(
                    text="250",
                    callback_data="qty:250"
                ),
                InlineKeyboardButton(
                    text="500",
                    callback_data="qty:500"
                )
            ],
            [
                InlineKeyboardButton(
                    text="1000",
                    callback_data="qty:1000"
                ),
                InlineKeyboardButton(
                    text="5000",
                    callback_data="qty:5000"
                )
            ],
            [
                InlineKeyboardButton(
                    text="✏️ Другое количество",
                    callback_data="qty:custom"
                )
            ],
            [
                InlineKeyboardButton(
                    text="◀️ Назад",
                    callback_data="back_category"
                )
            ]
        ]
    )


def payment_menu():
    return InlineKeyboardMarkup(
        inline_keyboard=[
            [
                InlineKeyboardButton(
                    text="💳 Оплатить через Crypto Bot",
                    callback_data="pay"
                )
            ],
            [
                InlineKeyboardButton(
                    text="◀️ Назад",
                    callback_data="back_quantity"
                )
            ]
        ]
    )


def generate_proxy():
    ip = (
        f"{random.randint(1, 223)}."
        f"{random.randint(0, 255)}."
        f"{random.randint(0, 255)}."
        f"{random.randint(1, 254)}"
    )

    port = random.choice([
        80,
        8080,
        3128,
        8000,
        8888
    ])

    return f"{ip}:{port}"


def generate_proxies(quantity):
    result = []
    used = set()

    while len(result) < quantity:
        proxy = generate_proxy()

        if proxy not in used:
            used.add(proxy)
            result.append(proxy)

    return result


async def crypto_api(method, data=None):
    headers = {
        "Crypto-Pay-API-Token": CRYPTO_PAY_TOKEN,
        "Content-Type": "application/json"
    }

    timeout = aiohttp.ClientTimeout(total=30)

    try:
        async with aiohttp.ClientSession(
            timeout=timeout
        ) as session:

            async with session.post(
                f"{CRYPTO_API}/{method}",
                headers=headers,
                json=data or {}
            ) as response:

                text = await response.text()

                if response.status != 200:
                    logging.error(
                        "Crypto Pay HTTP %s: %s",
                        response.status,
                        text
                    )
                    return None

                try:
                    result = await response.json()
                except Exception:
                    logging.error(
                        "Crypto Pay returned invalid JSON: %s",
                        text
                    )
                    return None

                if not result.get("ok"):
                    logging.error(
                        "Crypto Pay API error: %s",
                        result
                    )

                return result

    except asyncio.TimeoutError:
        logging.error(
            "Crypto Pay request timeout"
        )
        return None

    except aiohttp.ClientError as error:
        logging.error(
            "Crypto Pay connection error: %s",
            error
        )
        return None

    except Exception:
        logging.exception(
            "Unexpected Crypto Pay error"
        )
        return None


async def create_invoice(amount, order_id):
    return await crypto_api(
        "createInvoice",
        {
            "currency_type": "fiat",
            "fiat": "USD",
            "amount": str(amount),
            "description": f"Proxy Shop order #{order_id}",
            "payload": str(order_id)
        }
    )


async def get_invoice(invoice_id):
    result = await crypto_api(
        "getInvoices",
        {
            "invoice_ids": str(invoice_id)
        }
    )

    if not result:
        return None

    if not result.get("ok"):
        return None

    items = result.get(
        "result",
        {}
    ).get(
        "items",
        []
    )

    if not items:
        return None

    return items[0]


@dp.message(CommandStart())
async def start(message: Message):
    register_user(
        message.from_user
    )

    user_state.pop(
        message.from_user.id,
        None
    )

    await message.answer(
        "👋 <b>Добро пожаловать в Proxy Shop!</b>\n\n"
        "🔐 Прокси для различных задач.\n\n"
        "Выберите нужный раздел:",
        reply_markup=main_menu()
    )


@dp.callback_query(F.data == "home")
async def home(callback: CallbackQuery):
    register_user(
        callback.from_user
    )

    await callback.answer()

    user_state.pop(
        callback.from_user.id,
        None
    )

    await callback.message.edit_text(
        "🏠 <b>Главное меню</b>\n\n"
        "Выберите нужный раздел:",
        reply_markup=main_menu()
    )


@dp.callback_query(F.data == "buy_proxy")
async def buy_proxy(callback: CallbackQuery):
    register_user(
        callback.from_user
    )

    await callback.answer()

    await callback.message.edit_text(
        "🌍 <b>Выберите страну</b>\n\n"
        "Доступные локации:",
        reply_markup=countries_menu()
    )


@dp.callback_query(F.data.startswith("countries:"))
async def countries_page(callback: CallbackQuery):
    await callback.answer()

    page = int(
        callback.data.split(":")[1]
    )

    await callback.message.edit_text(
        "🌍 <b>Выберите страну</b>",
        reply_markup=countries_menu(page)
    )


@dp.callback_query(F.data.startswith("country:"))
async def select_country(callback: CallbackQuery):
    register_user(
        callback.from_user
    )

    await callback.answer()

    country = callback.data.split(":")[1]

    user_state[
        callback.from_user.id
    ] = {
        "product": "proxy",
        "country": country
    }

    await callback.message.edit_text(
        f"🌍 <b>{COUNTRIES[country]}</b>\n\n"
        "Выберите категорию:",
        reply_markup=category_menu(country)
    )


@dp.callback_query(F.data.startswith("category:"))
async def select_category(callback: CallbackQuery):
    register_user(
        callback.from_user
    )

    await callback.answer()

    _, category, country = callback.data.split(":")

    user_state[
        callback.from_user.id
    ] = {
        "product": "proxy",
        "country": country,
        "category": category
    }

    if category == "normal":
        name = "🔒 Обычные прокси"
        price = NORMAL_PRICE
    else:
        name = "📊 Прокси для парсинга"
        price = PARSING_PRICE

    await callback.message.edit_text(
        f"{name}\n\n"
        f"🌍 Страна: <b>{COUNTRIES[country]}</b>\n"
        f"💵 Цена: <b>${price:.2f}</b> за 1 шт.\n\n"
        "🔢 <b>Выберите количество:</b>",
        reply_markup=quantity_menu()
    )


@dp.callback_query(F.data.startswith("qty:"))
async def select_quantity(callback: CallbackQuery):
    register_user(
        callback.from_user
    )

    await callback.answer()

    value = callback.data.split(":")[1]

    state = user_state.get(
        callback.from_user.id
    )

    if not state:
        await callback.message.edit_text(
            "❌ Сессия заказа истекла.",
            reply_markup=main_menu()
        )
        return

    if value == "custom":
        state["waiting_quantity"] = True

        await callback.message.edit_text(
            "✏️ <b>Введите количество прокси</b>\n\n"
            "Минимум: <b>1</b>\n"
            "Максимум: <b>5000</b>"
        )

        return

    quantity = int(value)

    state["quantity"] = quantity

    price = (
        NORMAL_PRICE
        if state["category"] == "normal"
        else PARSING_PRICE
    )

    state["amount"] = (
        price * quantity
    ).quantize(
        Decimal("0.01"),
        rounding=ROUND_HALF_UP
    )

    await show_order(
        callback.message,
        state
    )


@dp.message(F.text)
async def custom_quantity(message: Message):
    register_user(
        message.from_user
    )

    state = user_state.get(
        message.from_user.id
    )

    if not state:
        return

    if not state.get(
        "waiting_quantity"
    ):
        return

    try:
        quantity = int(
            message.text.strip()
        )
    except ValueError:
        await message.answer(
            "❌ Введите целое число."
        )
        return

    if quantity < 1 or quantity > 5000:
        await message.answer(
            "❌ Количество должно быть "
            "от 1 до 5000."
        )
        return

    state["waiting_quantity"] = False
    state["quantity"] = quantity

    price = (
        NORMAL_PRICE
        if state["category"] == "normal"
        else PARSING_PRICE
    )

    state["amount"] = (
        price * quantity
    ).quantize(
        Decimal("0.01"),
        rounding=ROUND_HALF_UP
    )

    await show_order(
        message,
        state
    )


async def show_order(message, state):
    if state["category"] == "normal":
        category_name = "🔒 Обычные прокси"
        price = NORMAL_PRICE
    else:
        category_name = "📊 Прокси для парсинга"
        price = PARSING_PRICE

    country = state["country"]
    quantity = state["quantity"]
    amount = state["amount"]

    await message.answer(
        "🧾 <b>Ваш заказ</b>\n\n"
        f"📦 Категория: <b>{category_name}</b>\n"
        f"🌍 Страна: <b>{COUNTRIES[country]}</b>\n"
        f"🔢 Количество: <b>{quantity}</b>\n"
        f"💵 Цена: <b>${price:.2f}</b> / шт.\n\n"
        f"💰 <b>Итого: ${amount:.2f}</b>\n\n"
        "Выберите способ оплаты:",
        reply_markup=payment_menu()
    )


@dp.callback_query(F.data == "pay")
async def pay(callback: CallbackQuery):
    register_user(
        callback.from_user
    )

    await callback.answer()

    state = user_state.get(
        callback.from_user.id
    )

    if not state:
        await callback.message.edit_text(
            "❌ Сессия заказа истекла.\n\n"
            "Начните оформление заново.",
            reply_markup=main_menu()
        )
        return

    if (
        "country" not in state
        or "category" not in state
        or "quantity" not in state
        or "amount" not in state
    ):
        await callback.message.edit_text(
            "❌ Данные заказа неполные.\n\n"
            "Создайте заказ заново.",
            reply_markup=main_menu()
        )
        return

    user_id = get_user_id(
        callback.from_user.id
    )

    country = state["country"]
    category = state["category"]
    quantity = state["quantity"]
    amount = state["amount"]

    cursor.execute(
        """
        INSERT INTO orders
        (
            user_id,
            product,
            country,
            category,
            quantity,
            amount,
            status
        )
        VALUES (?, ?, ?, ?, ?, ?, ?)
        """,
        (
            user_id,
            "proxy",
            country,
            category,
            quantity,
            str(amount),
            "pending"
        )
    )

    order_id = cursor.lastrowid

    db.commit()

    invoice_result = await create_invoice(
        amount,
        order_id
    )

    if not invoice_result:
        cursor.execute(
            """
            UPDATE orders
            SET status = ?
            WHERE id = ?
            """,
            (
                "error",
                order_id
            )
        )

        db.commit()

        await callback.message.edit_text(
            "❌ <b>Не удалось создать счёт.</b>\n\n"
            "Проверьте Crypto Pay API Token "
            "и попробуйте снова.",
            reply_markup=main_menu()
        )
        return

    if not invoice_result.get("ok"):
        await callback.message.edit_text(
            "❌ <b>Crypto Bot отклонил создание счёта.</b>\n\n"
            "Проверьте API Token и настройки Crypto Bot.",
            reply_markup=main_menu()
        )
        return

    invoice = invoice_result.get(
        "result"
    )

    if not invoice:
        await callback.message.edit_text(
            "❌ Crypto Bot не вернул данные счёта.",
            reply_markup=main_menu()
        )
        return

    invoice_id = invoice.get(
        "invoice_id"
    )

    pay_url = (
        invoice.get("bot_invoice_url")
        or invoice.get("mini_app_invoice_url")
        or invoice.get("web_app_invoice_url")
    )

    if not invoice_id or not pay_url:
        logging.error(
            "Invalid invoice response: %s",
            invoice
        )

        await callback.message.edit_text(
            "❌ Crypto Bot не вернул "
            "корректную ссылку на оплату.",
            reply_markup=main_menu()
        )
        return

    cursor.execute(
        """
        UPDATE orders
        SET invoice_id = ?
        WHERE id = ?
        """,
        (
            str(invoice_id),
            order_id
        )
    )

    db.commit()

    keyboard = InlineKeyboardMarkup(
        inline_keyboard=[
            [
                InlineKeyboardButton(
                    text="💳 Оплатить",
                    url=pay_url
                )
            ],
            [
                InlineKeyboardButton(
                    text="🔄 Проверить оплату",
                    callback_data=f"check:{order_id}"
                )
            ],
            [
                InlineKeyboardButton(
                    text="🏠 Главное меню",
                    callback_data="home"
                )
            ]
        ]
    )

    await callback.message.edit_text(
        "💳 <b>Счёт создан</b>\n\n"
        f"🧾 Заказ: <code>#{order_id}</code>\n"
        f"📦 Количество: <b>{quantity}</b>\n"
        f"💰 Сумма: <b>${amount:.2f}</b>\n\n"
        "1. Нажмите «Оплатить».\n"
        "2. Завершите оплату.\n"
        "3. Вернитесь сюда.\n"
        "4. Нажмите «Проверить оплату».",
        reply_markup=keyboard
    )


@dp.callback_query(F.data.startswith("check:"))
async def check_payment(callback: CallbackQuery):
    register_user(
        callback.from_user
    )

    order_id = int(
        callback.data.split(":")[1]
    )

    cursor.execute(
        """
        SELECT
            id,
            user_id,
            product,
            country,
            category,
            quantity,
            amount,
            invoice_id,
            status,
            content
        FROM orders
        WHERE id = ?
        """,
        (order_id,)
    )

    order = cursor.fetchone()

    if not order:
        await callback.answer(
            "Заказ не найден.",
            show_alert=True
        )
        return

    (
        db_order_id,
        order_user_id,
        product,
        country,
        category,
        quantity,
        amount,
        invoice_id,
        status,
        content
    ) = order

    current_user_id = get_user_id(
        callback.from_user.id
    )

    if order_user_id != current_user_id:
        await callback.answer(
            "Этот заказ принадлежит другому пользователю.",
            show_alert=True
        )
        return

    if status == "paid":
        await callback.answer(
            "Заказ уже был выдан.",
            show_alert=True
        )
        return

    if not invoice_id:
        await callback.answer(
            "У заказа отсутствует invoice.",
            show_alert=True
        )
        return

    await callback.answer(
        "Проверяю оплату..."
    )

    invoice = await get_invoice(
        invoice_id
    )

    if not invoice:
        await callback.message.answer(
            "❌ Не удалось получить информацию "
            "о счёте Crypto Bot."
        )
        return

    invoice_status = invoice.get(
        "status"
    )

    if invoice_status != "paid":
        await callback.answer(
            "⏳ Оплата пока не подтверждена.",
            show_alert=True
        )
        return

    cursor.execute(
        """
        SELECT status
        FROM orders
        WHERE id = ?
        """,
        (order_id,)
    )

    latest = cursor.fetchone()

    if not latest:
        return

    if latest[0] == "paid":
        await callback.message.edit_text(
            "✅ Заказ уже был выдан.",
            reply_markup=main_menu()
        )
        return

    proxies = generate_proxies(
        quantity
    )

    proxy_text = "\n".join(
        proxies
    )

    cursor.execute(
        """
        UPDATE orders
        SET status = ?,
            content = ?
        WHERE id = ?
        """,
        (
            "paid",
            proxy_text,
            order_id
        )
    )

    db.commit()

    await send_proxies(
        callback.message,
        country,
        category,
        quantity,
        amount,
        proxy_text
    )


async def send_proxies(
    message,
    country,
    category,
    quantity,
    amount,
    proxy_text
):
    if category == "normal":
        category_name = "🔒 Обычные прокси"
    else:
        category_name = "📊 Прокси для парсинга"

    await message.edit_text(
        "🎉 <b>Оплата подтверждена!</b>\n\n"
        f"📦 Категория: <b>{category_name}</b>\n"
        f"🌍 Страна: <b>{COUNTRIES[country]}</b>\n"
        f"🔢 Количество: <b>{quantity}</b>\n"
        f"💰 Оплачено: <b>${amount}</b>\n\n"
        "📥 <b>Ваш товар:</b>"
    )

    lines = proxy_text.split("\n")

    chunk = ""

    for line in lines:
        if len(chunk) + len(line) + 1 > 3500:
            await message.answer(
                f"<code>{chunk}</code>"
            )
            chunk = ""

        chunk += line + "\n"

    if chunk:
        await message.answer(
            f"<code>{chunk}</code>"
        )

    await message.answer(
        "✅ <b>Заказ успешно выдан.</b>\n\n"
        "Спасибо за покупку!",
        reply_markup=main_menu()
    )


@dp.callback_query(F.data == "profile")
async def profile(callback: CallbackQuery):
    register_user(
        callback.from_user
    )

    await callback.answer()

    telegram_id = callback.from_user.id

    user_id = get_user_id(
        telegram_id
    )

    cursor.execute(
        """
        SELECT
            first_name,
            username,
            created_at,
            last_activity
        FROM users
        WHERE id = ?
        """,
        (user_id,)
    )

    user = cursor.fetchone()

    if not user:
        await callback.message.edit_text(
            "❌ Профиль не найден.",
            reply_markup=main_menu()
        )
        return

    cursor.execute(
        """
        SELECT
            COUNT(*),
            COALESCE(
                SUM(CAST(amount AS REAL)),
                0
            ),
            COALESCE(
                SUM(quantity),
                0
            )
        FROM orders
        WHERE user_id = ?
        AND status = 'paid'
        """,
        (user_id,)
    )

    stats = cursor.fetchone()

    orders_count = stats[0]
    spent = stats[1]
    items_count = stats[2]

    first_name = user[0] or "—"

    if user[1]:
        username = f"@{user[1]}"
    else:
        username = "—"

    await callback.message.edit_text(
        "👤 <b>Мой профиль</b>\n\n"
        f"🆔 ID: <code>{user_id}</code>\n"
        f"👤 Имя: <b>{first_name}</b>\n"
        f"🔗 Username: <b>{username}</b>\n"
        f"📅 Регистрация: <b>{user[2]}</b>\n"
        f"🟢 Последняя активность: <b>{user[3]}</b>\n\n"
        "📊 <b>Статистика</b>\n\n"
        f"🧾 Заказов: <b>{orders_count}</b>\n"
        f"📦 Куплено единиц: <b>{items_count}</b>\n"
        f"💰 Потрачено: <b>${spent:.2f}</b>",
        reply_markup=InlineKeyboardMarkup(
            inline_keyboard=[
                [
                    InlineKeyboardButton(
                        text="🛒 Купить прокси",
                        callback_data="buy_proxy"
                    )
                ],
                [
                    InlineKeyboardButton(
                        text="◀️ Назад",
                        callback_data="home"
                    )
                ]
            ]
        )
    )


@dp.callback_query(F.data == "support")
async def support(callback: CallbackQuery):
    await callback.answer()

    await callback.message.edit_text(
        "💬 <b>Поддержка</b>\n\n"
        "По вопросам заказа и оплаты "
        "обратитесь в поддержку.",
        reply_markup=InlineKeyboardMarkup(
            inline_keyboard=[
                [
                    InlineKeyboardButton(
                        text="💬 Написать в поддержку",
                        url="https://t.me/fhcnncns"
                    )
                ],
                [
                    InlineKeyboardButton(
                        text="🏠 Главное меню",
                        callback_data="home"
                    )
                ]
            ]
        )
    )


@dp.callback_query(F.data == "back_category")
async def back_category(callback: CallbackQuery):
    await callback.answer()

    state = user_state.get(
        callback.from_user.id
    )

    if not state:
        await callback.message.edit_text(
            "🌍 <b>Выберите страну</b>",
            reply_markup=countries_menu()
        )
        return

    country = state.get(
        "country"
    )

    if not country:
        await callback.message.edit_text(
            "🌍 <b>Выберите страну</b>",
            reply_markup=countries_menu()
        )
        return

    await callback.message.edit_text(
        f"🌍 <b>{COUNTRIES[country]}</b>\n\n"
        "Выберите категорию:",
        reply_markup=category_menu(country)
    )


@dp.callback_query(F.data == "back_quantity")
async def back_quantity(callback: CallbackQuery):
    await callback.answer()

    state = user_state.get(
        callback.from_user.id
    )

    if not state:
        await callback.message.edit_text(
            "🏠 <b>Главное меню</b>",
            reply_markup=main_menu()
        )
        return

    await callback.message.edit_text(
        "🔢 <b>Выберите количество:</b>",
        reply_markup=quantity_menu()
    )


async def main():
    if BOT_TOKEN.startswith("ВСТАВЬ_"):
        print(
            "Ошибка: укажите новый Telegram Bot Token."
        )
        return

    if CRYPTO_PAY_TOKEN.startswith("ВСТАВЬ_"):
        print(
            "Ошибка: укажите новый Crypto Pay API Token."
        )
        return

    logging.info(
        "Proxy Shop запускается..."
    )

    await dp.start_polling(
        bot,
        allowed_updates=dp.resolve_used_update_types()
    )


if __name__ == "__main__":
    asyncio.run(main())

