import asyncio
import sqlite3
from datetime import datetime, timedelta
from aiogram import Bot, Dispatcher, types, F
from aiogram.fsm.storage.memory import MemoryStorage
from aiogram.filters import Command
from aiogram.types import ReplyKeyboardMarkup, KeyboardButton, InlineKeyboardMarkup, InlineKeyboardButton
from aiohttp import web

# Настройки бота
TOKEN = "7735755561:AAGC7KVBotDxj9hA_crYWXZHZfAipauDcek"
CHANNEL_ID = "@ygarstuker"
ADMIN_ID = 1608230607
WEBHOOK_PATH = "/webhook"
WEBHOOK_URL = "https://your-app-name.onrender.com" + WEBHOOK_PATH  # Замените your-app-name на имя вашего приложения
WEBAPP_HOST = "0.0.0.0"
WEBAPP_PORT = 3000  # Render использует порт 3000 по умолчанию
DB_PATH = "referrals.db"  # Путь к базе данных на Render

# Инициализация бота и диспетчера
bot = Bot(token=TOKEN)
dp = Dispatcher(storage=MemoryStorage())

# Создание клавиатур
MAIN_MENU = ReplyKeyboardMarkup(
    keyboard=[
        [KeyboardButton(text="📊 Моя статистика"), KeyboardButton(text="🔗 Получить реф. ссылку")],
        [KeyboardButton(text="🏆 Топ 10 рефералов"), KeyboardButton(text="📝 Инструкция")]
    ],
    resize_keyboard=True
)

SUBSCRIBE_KB = InlineKeyboardMarkup(
    inline_keyboard=[
        [InlineKeyboardButton(text="Подписаться", url=f"https://t.me/{CHANNEL_ID[1:]}")],
        [InlineKeyboardButton(text="✅ Проверить подписку", callback_data="check_sub")]
    ]
)

# Инициализация базы данных
def init_db():
    with sqlite3.connect(DB_PATH) as conn:
        conn.execute('''CREATE TABLE IF NOT EXISTS users 
                        (user_id INTEGER PRIMARY KEY, 
                         ref_count INTEGER DEFAULT 0, 
                         ref_link TEXT, 
                         join_date TEXT)''')
        conn.execute('''CREATE TABLE IF NOT EXISTS referrals 
                        (user_id INTEGER, 
                         inviter_id INTEGER, 
                         timestamp TEXT, 
                         PRIMARY KEY (user_id, inviter_id))''')

# Генерация реферальной ссылки
async def generate_ref_link(user_id):
    bot_info = await bot.get_me()
    return f"https://t.me/{bot_info.username}?start={user_id}"

# Проверка подписки
async def is_subscribed(user_id):
    try:
        member = await bot.get_chat_member(CHANNEL_ID, user_id)
        return member.status != 'left'
    except Exception:
        return False

# Обработка /start с анти-накруткой
@dp.message(Command("start"))
async def start_command(message: types.Message):
    user_id = message.from_user.id
    args = message.text.split(maxsplit=1)[1] if len(message.text.split()) > 1 else ""

    if not await is_subscribed(user_id):
        await message.answer("Для использования бота подпишитесь на канал!", reply_markup=SUBSCRIBE_KB)
        return

    init_db()
    with sqlite3.connect(DB_PATH) as conn:
        cursor = conn.cursor()
        cursor.execute("SELECT ref_link FROM users WHERE user_id = ?", (user_id,))
        user = cursor.fetchone()

        if not user:
            ref_link = await generate_ref_link(user_id)
            cursor.execute("INSERT INTO users (user_id, ref_count, ref_link, join_date) VALUES (?, 0, ?, ?)",
                           (user_id, ref_link, datetime.now().strftime("%Y-%m-%d %H:%M:%S")))
            conn.commit()

        if args and args.isdigit() and int(args) != user_id:
            inviter_id = int(args)
            cursor.execute("SELECT 1 FROM users WHERE user_id = ?", (inviter_id,))
            if cursor.fetchone():
                cursor.execute("SELECT timestamp FROM referrals WHERE user_id = ? AND inviter_id = ?",
                               (user_id, inviter_id))
                existing = cursor.fetchone()
                if existing:
                    await message.answer("Вы уже использовали эту реферальную ссылку!")
                    return

                cursor.execute("SELECT timestamp FROM referrals WHERE user_id = ? ORDER BY timestamp DESC LIMIT 1",
                               (user_id,))
                last_ref = cursor.fetchone()
                if last_ref:
                    last_time = datetime.strptime(last_ref[0], "%Y-%m-%d %H:%M:%S")
                    if datetime.now() - last_time < timedelta(hours=24):
                        await message.answer("Слишком частые попытки! Подождите 24 часа.")
                        return

                cursor.execute("UPDATE users SET ref_count = ref_count + 1 WHERE user_id = ?", (inviter_id,))
                cursor.execute("INSERT INTO referrals (user_id, inviter_id, timestamp) VALUES (?, ?, ?)",
                               (user_id, inviter_id, datetime.now().strftime("%Y-%m-%d %H:%M:%S")))
                conn.commit()
                await bot.send_message(inviter_id, "🎉 У вас новый реферал!")

    await message.answer("👋 Добро пожаловать в реферальный бот!\nВыберите действие:", reply_markup=MAIN_MENU)

# Проверка подписки через кнопку
@dp.callback_query(F.data == "check_sub")
async def check_subscription(callback: types.CallbackQuery):
    if await is_subscribed(callback.from_user.id):
        await callback.message.delete()
        await callback.message.answer("👋 Добро пожаловать в реферальный бот!\nВыберите действие:", reply_markup=MAIN_MENU)
    else:
        await callback.answer("Вы ещё не подписаны!", show_alert=True)

# Статистика пользователя
@dp.message(F.text == "📊 Моя статистика")
async def my_stats(message: types.Message):
    with sqlite3.connect(DB_PATH) as conn:
        cursor = conn.cursor()
        cursor.execute("SELECT ref_count FROM users WHERE user_id = ?", (message.from_user.id,))
        result = cursor.fetchone()
        ref_count = result[0] if result else 0
    await message.answer(f"👤 Ваши рефералы: {ref_count}")

# Получение реферальной ссылки
@dp.message(F.text == "🔗 Получить реф. ссылку")
async def get_ref_link(message: types.Message):
    with sqlite3.connect(DB_PATH) as conn:
        cursor = conn.cursor()
        cursor.execute("SELECT ref_link FROM users WHERE user_id = ?", (message.from_user.id,))
        result = cursor.fetchone()
        ref_link = result[0] if result else await generate_ref_link(message.from_user.id)
    await message.answer(f"🔗 Ваша реферальная ссылка:\n`{ref_link}`", parse_mode="Markdown")

# Топ-10 рефералов
@dp.message(F.text == "🏆 Топ 10 рефералов")
async def top_referrals(message: types.Message):
    with sqlite3.connect(DB_PATH) as conn:
        cursor = conn.cursor()
        cursor.execute("SELECT user_id, ref_count FROM users ORDER BY ref_count DESC LIMIT 10")
        top = cursor.fetchall()

    if not top:
        await message.answer("🏆 Пока нет рефералов в топе.")
        return

    text = "🏆 Топ 10 рефералов:\n\n"
    for i, (user_id, ref_count) in enumerate(top, 1):
        try:
            user = await bot.get_chat(user_id)
            username = user.username or f"User {user_id}"
            text += f"{i}. @{username} - {ref_count} реф.\n"
        except Exception:
            text += f"{i}. User {user_id} - {ref_count} реф.\n"
    await message.answer(text)

# Инструкция
@dp.message(F.text == "📝 Инструкция")
async def instructions(message: types.Message):
    await message.answer("📌 Приглашайте друзей по своей реферальной ссылке и участвуйте в розыгрышах!")

# Админ-панель
@dp.message(Command("admin"), lambda message: message.from_user.id == ADMIN_ID)
async def admin_panel(message: types.Message):
    with sqlite3.connect(DB_PATH) as conn:
        cursor = conn.cursor()
        cursor.execute("SELECT user_id, ref_count, join_date FROM users")
        users = cursor.fetchall()

    if not users:
        await message.answer("👨‍💼 Пока нет участников.")
        return

    text = "👨‍💼 Список участников:\n\n"
    for user_id, ref_count, join_date in users:
        try:
            user = await bot.get_chat(user_id)
            username = user.username or f"User {user_id}"
            text += f"@{username} | Реф: {ref_count} | Дата: {join_date}\n"
        except Exception:
            text += f"User {user_id} | Реф: {ref_count} | Дата: {join_date}\n"
    await message.answer(text)

# Анти-накрутка для добавления в чат
@dp.message(F.content_type == types.ContentType.NEW_CHAT_MEMBERS)
async def anti_cheat(message: types.Message):
    for member in message.new_chat_members:
        if member.id == bot.id:
            await message.answer("Бот добавлен в чат. Используйте /start для начала!")

# Обработка webhook
async def handle_webhook(request):
    update = await request.json()
    await dp.feed_raw_update(bot, update)
    return web.Response(text="OK")

# Настройка и запуск
async def on_startup():
    init_db()
    await bot.set_webhook(WEBHOOK_URL)
    print(f"Webhook установлен: {WEBHOOK_URL}")

async def on_shutdown():
    await bot.delete_webhook()
    print("Webhook удалён")

async def main():
    app = web.Application()
    app.router.add_post(WEBHOOK_PATH, handle_webhook)
    runner = web.AppRunner(app)
    await runner.setup()
    site = web.TCPSite(runner, WEBAPP_HOST, WEBAPP_PORT)
    await site.start()
    await on_startup()
    try:
        await asyncio.Event().wait()  # Бесконечный цикл
    finally:
        await on_shutdown()

if __name__ == "__main__":
    asyncio.run(main())
