# MCD import os
import sqlite3
import logging
from telegram import Update, InlineKeyboardButton, InlineKeyboardMarkup
from telegram.ext import Application, CommandHandler, CallbackQueryHandler, MessageHandler, Filters, ContextTypes
from flask import Flask, request
import threading
import uuid
import requests

# Logging sozlamalari
logging.basicConfig(format='%(asctime)s - %(name)s - %(levelname)s - %(message)s', level=logging.INFO)
logger = logging.getLogger(__name__)

# Flask ilovasini yaratish
flask_app = Flask(__name__)

# Ma'lumotlar bazasini sozlash
def init_db():
    conn = sqlite3.connect('transport_bot.db')
    c = conn.cursor()
    c.execute('''CREATE TABLE IF NOT EXISTS users (
        user_id INTEGER PRIMARY KEY,
        role TEXT,
        name TEXT,
        phone TEXT,
        location TEXT
    )''')
    c.execute('''CREATE TABLE IF NOT EXISTS loads (
        load_id INTEGER PRIMARY KEY AUTOINCREMENT,
        user_id INTEGER,
        description TEXT,
        start_location TEXT,
        end_location TEXT,
        status TEXT
    )''')
    conn.commit()
    conn.close()

# Telegram bot tokeni va webhook URL
TOKEN = os.getenv("BOT_TOKEN", "YOUR_BOT_TOKEN")
WEBHOOK_URL = os.getenv("WEBHOOK_URL", "https://your-render-app.onrender.com/webhook")

# Boshlang'ich buyruq
async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user = update.effective_user
    keyboard = [
        [InlineKeyboardButton("Haydovchi", callback_data='driver'),
         InlineKeyboardButton("Yuk beruvchi", callback_data='client')]
    ]
    reply_markup = InlineKeyboardMarkup(keyboard)
    await update.message.reply_text(
        f"Assalomu alaykum, {user.first_name}! Siz haydovchimisiz yoki yuk beruvchimisiz?",
        reply_markup=reply_markup
    )

# Foydalanuvchi rolini tanlash
async def button(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    await query.answer()
    role = query.data
    context.user_data['role'] = role
    await query.message.reply_text(
        "Iltimos, ismingizni kiriting:"
    )
    return "GET_NAME"

# Ismni olish
async def get_name(update: Update, context: ContextTypes.DEFAULT_TYPE):
    context.user_data['name'] = update.message.text
    await update.message.reply_text("Telefon raqamingizni kiriting (masalan, +998901234567):")
    return "GET_PHONE"

# Telefon raqamini olish
async def get_phone(update: Update, context: ContextTypes.DEFAULT_TYPE):
    context.user_data['phone'] = update.message.text
    await update.message.reply_text("Joylashuvingizni kiriting (masalan, Toshkent):")
    return "GET_LOCATION"

# Joylashuvni olish va ro'yxatdan o'tkazish
async def get_location(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user = update.effective_user
    location = update.message.text
    role = context.user_data.get('role')
    name = context.user_data.get('name')
    phone = context.user_data.get('phone')

    conn = sqlite3.connect('transport_bot.db')
    c = conn.cursor()
    c.execute("INSERT OR REPLACE INTO users (user_id, role, name, phone, location) VALUES (?, ?, ?, ?, ?)",
              (user.id, role, name, phone, location))
    conn.commit()
    conn.close()

    if role == 'driver':
        await update.message.reply_text(
            "Ro'yxatdan o'tdingiz! /newload buyrug'i bilan yangi yuklarni ko'rishingiz mumkin."
        )
    else:
        await update.message.reply_text(
            "Ro'yxatdan o'tdingiz! /postload buyrug'i bilan yuk e'lon qilishingiz mumkin."
        )

# Yuk e'lon qilish
async def post_load(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await update.message.reply_text(
        "Yuk haqida ma'lumot kiriting (masalan, 5 tonna paxta):"
    )
    return "GET_LOAD_DESC"

async def get_load_desc(update: Update, context: ContextTypes.DEFAULT_TYPE):
    context.user_data['load_desc'] = update.message.text
    await update.message.reply_text("Yukning boshlang'ich manzilini kiriting:")
    return "GET_START_LOC"

async def get_start_loc(update: Update, context: ContextTypes.DEFAULT_TYPE):
    context.user_data['start_loc'] = update.message.text
    await update.message.reply_text("Yukning yakuniy manzilini kiriting:")
    return "GET_END_LOC"

async def get_end_loc(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user = update.effective_user
    load_desc = context.user_data.get('load_desc')
    start_loc = context.user_data.get('start_loc')
    end_loc = update.message.text

    conn = sqlite3.connect('transport_bot.db')
    c = conn.cursor()
    c.execute("INSERT INTO loads (user_id, description, start_location, end_location, status) VALUES (?, ?, ?, ?, ?)",
              (user.id, load_desc, start_loc, end_loc, 'open'))
    conn.commit()
    conn.close()

    await update.message.reply_text("Yuk e'lon qilindi! Haydovchilar bilan bog'lanish uchun /finddriver buyrug'ini ishlatishingiz mumkin.")

# Haydovchilarni topish
async def find_driver(update: Update, context: ContextTypes.DEFAULT_TYPE):
    conn = sqlite3.connect('transport_bot.db')
    c = conn.cursor()
    c.execute("SELECT user_id, name, phone, location FROM users WHERE role = 'driver'")
    drivers = c.fetchall()
    conn.close()

    if drivers:
        message = "Mavjud haydovchilar:\n"
        for driver in drivers:
            message += f"Ism: {driver[1]}, Telefon: {driver[2]}, Manzil: {driver[3]}\n"
        await update.message.reply_text(message)
    else:
        await update.message.reply_text("Hozirda mavjud haydovchilar yo'q.")

# Yangi yuklarni ko'rish
async def new_load(update: Update, context: ContextTypes.DEFAULT_TYPE):
    conn = sqlite3.connect('transport_bot.db')
    c = conn.cursor()
    c.execute("SELECT load_id, description, start_location, end_location FROM loads WHERE status = 'open'")
    loads = c.fetchall()
    conn.close()

    if loads:
        message = "Mavjud yuklar:\n"
        for load in loads:
            message += f"Yuk ID: {load[0]}, Tavsif: {load[1]}, Boshlanish: {load[2]}, Yakun: {load[3]}\n"
        await update.message.reply_text(message)
    else:
        await update.message.reply_text("Hozirda mavjud yuklar yo'q.")

# Xato ishlov berish
async def error_handler(update: Update, context: ContextTypes.DEFAULT_TYPE):
    logger.error(f"Update {update} caused error {context.error}")
    if update.message:
        await update.message.reply_text("Xatolik yuz berdi. Iltimos, qaytadan urinib ko'ring.")

# Webhook so'rovlarini qabul qilish
@flask_app.route('/webhook', methods=['POST'])
def webhook():
    update = Update.de_json(request.get_json(force=True), application.bot)
    application.process_update(update)
    return 'OK', 200

# Flask serverini ishga tushirish
def run_flask():
    flask_app.run(host='0.0.0.0', port=int(os.getenv('PORT', 8443)))

# Botni ishga tushirish
def main():
    global application
    init_db()
    application = Application.builder().token(TOKEN).build()

    # Buyruqlarni qo'shish
    application.add_handler(CommandHandler("start", start))
    application.add_handler(CallbackQueryHandler(button))
    application.add_handler(CommandHandler("postload", post_load))
    application.add_handler(CommandHandler("finddriver", find_driver))
    application.add_handler(CommandHandler("newload", new_load))

    # Ma'lumot olish jarayonlari
    application.add_handler(MessageHandler(Filters.text & ~Filters.command, get_name), group=1)
    application.add_handler(MessageHandler(Filters.text & ~Filters.command, get_phone), group=2)
    application.add_handler(MessageHandler(Filters.text & ~Filters.command, get_location), group=3)
    application.add_handler(MessageHandler(Filters.text & ~Filters.command, get_load_desc), group=4)
    application.add_handler(MessageHandler(Filters.text & ~Filters.command, get_start_loc), group=5)
    application.add_handler(MessageHandler(Filters.text & ~Filters.command, get_end_loc), group=6)

    application.add_error_handler(error_handler)

    # Webhookni o'rnatish
    application.bot.set_webhook(url=WEBHOOK_URL)

    # Flask serverini alohida threadda ishga tushirish
    flask_thread = threading.Thread(target=run_flask)
    flask_thread.start()

if __name__ == '__main__':
    main()
