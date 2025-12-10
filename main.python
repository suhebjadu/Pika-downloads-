#!/usr/bin/env python3
import os
import logging
import sqlite3
import secrets
import string

from telegram import Update, BotCommand
from telegram.ext import (
    ApplicationBuilder, CommandHandler, MessageHandler,
    filters, ContextTypes, ConversationHandler
)

# ---- ENV VARIABLES ----
BOT_TOKEN = os.environ.get("BOT_TOKEN")
ADMIN_CHAT_ID = int(os.environ.get("ADMIN_CHAT_ID", "0"))
ADMIN_PASSWORD = os.environ.get("ADMIN_PASSWORD")

DB_PATH = "database.db"

# ---- STATES ----
STATE_WAIT_PASSWORD = 1
STATE_WAIT_FILE = 2
STATE_WAIT_AD_TEXT = 3
STATE_WAIT_IMG = 4
STATE_WAIT_VIDEO = 5


# ---- DATABASE SETUP ----
def init_db():
    conn = sqlite3.connect(DB_PATH)
    cur = conn.cursor()

    cur.execute("""
    CREATE TABLE IF NOT EXISTS uploads (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        token TEXT UNIQUE,
        file_id TEXT,
        file_type TEXT,
        filename TEXT
    );
    """)

    cur.execute("""
    CREATE TABLE IF NOT EXISTS users (
        chat_id INTEGER PRIMARY KEY
    );
    """)

    conn.commit()
    conn.close()


def save_user(chat_id):
    conn = sqlite3.connect(DB_PATH)
    conn.execute("INSERT OR IGNORE INTO users(chat_id) VALUES(?)", (chat_id,))
    conn.commit()
    conn.close()


def all_users():
    conn = sqlite3.connect(DB_PATH)
    rows = conn.execute("SELECT chat_id FROM users").fetchall()
    conn.close()
    return [r[0] for r in rows]


def save_upload(token, file_id, file_type, filename):
    conn = sqlite3.connect(DB_PATH)
    conn.execute(
        "INSERT INTO uploads(token, file_id, file_type, filename) VALUES (?,?,?,?)",
        (token, file_id, file_type, filename)
    )
    conn.commit()
    conn.close()


def get_upload(token):
    conn = sqlite3.connect(DB_PATH)
    row = conn.execute(
        "SELECT file_id, file_type, filename FROM uploads WHERE token=?",
        (token,)
    ).fetchone()
    conn.close()
    return row


admin_sessions = {}


def random_token(n=12):
    return "".join(secrets.choice(string.ascii_letters + string.digits) for _ in range(n))


# ---- BOT HANDLERS ----
async def start_handler(update: Update, context: ContextTypes.DEFAULT_TYPE):
    args = context.args

    if args:
        token = args[0]
        data = get_upload(token)

        if not data:
            return

        file_id, file_type, filename = data
        user_id = update.effective_chat.id

        save_user(user_id)

        if file_type == "photo":
            await update.message.reply_photo(photo=file_id)
        elif file_type == "video":
            await update.message.reply_video(video=file_id)
        else:
            await update.message.reply_document(document=file_id, filename=filename)


async def admin_cmd(update, context):
    if update.effective_chat.id != ADMIN_CHAT_ID:
        return

    await update.message.reply_text("Enter password:")
    return STATE_WAIT_PASSWORD


async def password_entered(update, context):
    if update.message.text == ADMIN_PASSWORD:
        admin_sessions[update.effective_chat.id] = True
        await update.message.reply_text("Admin login successful.")
    else:
        await update.message.reply_text("Incorrect password.")

    return ConversationHandler.END


async def upload_cmd(update, context):
    chat_id = update.effective_chat.id
    if chat_id != ADMIN_CHAT_ID or not admin_sessions.get(chat_id):
        return

    await update.message.reply_text("Send file now:")
    return STATE_WAIT_FILE


async def file_received(update, context):
    chat_id = update.effective_chat.id
    if chat_id != ADMIN_CHAT_ID:
        return ConversationHandler.END

    msg = update.message

    file_id = None
    file_type = None
    filename = None

    if msg.document:
        file_id = msg.document.file_id
        filename = msg.document.file_name
        file_type = "document"

    elif msg.photo:
        file_id = msg.photo[-1].file_id
        file_type = "photo"

    elif msg.video:
        file_id = msg.video.file_id
        file_type = "video"

    else:
        await update.message.reply_text("Invalid file.")
        return STATE_WAIT_FILE

    token = random_token()
    save_upload(token, file_id, file_type, filename)

    bot_username = (await context.bot.get_me()).username
    link = f"https://t.me/{bot_username}?start={token}"

    await update.message.reply_text(f"Uploaded successfully.\nURL:\n{link}")

    return ConversationHandler.END


# ---- BROADCAST: TEXT ----
async def sendads_cmd(update, context):
    if update.effective_chat.id != ADMIN_CHAT_ID:
        return

    await update.message.reply_text("Send broadcast text:")
    return STATE_WAIT_AD_TEXT


async def receive_ads_text(update, context):
    text = update.message.text
    users = all_users()

    sent = 0
    for u in users:
        try:
            await context.bot.send_message(u, text)
            sent += 1
        except:
            pass

    await update.message.reply_text(f"Sent to {sent} users.")
    return ConversationHandler.END


# ---- BROADCAST: IMAGE ----
async def sendimgads_cmd(update, context):
    if update.effective_chat.id != ADMIN_CHAT_ID:
        return

    await update.message.reply_text("Send image:")
    return STATE_WAIT_IMG


async def img_received(update, context):
    if not update.message.photo:
        await update.message.reply_text("Send an image.")
        return STATE_WAIT_IMG

    img = update.message.photo[-1].file_id
    users = all_users()
    sent = 0

    for u in users:
        try:
            await context.bot.send_photo(u, img)
            sent += 1
        except:
            pass

    await update.message.reply_text(f"Image sent to {sent} users.")
    return ConversationHandler.END


# ---- BROADCAST: VIDEO ----
async def sendvidads_cmd(update, context):
    if update.effective_chat.id != ADMIN_CHAT_ID:
        return

    await update.message.reply_text("Send video:")
    return STATE_WAIT_VIDEO


async def video_received(update, context):
    if not update.message.video:
        await update.message.reply_text("Send a video.")
        return STATE_WAIT_VIDEO

    vid = update.message.video.file_id
    users = all_users()
    sent = 0

    for u in users:
        try:
            await context.bot.send_video(u, vid)
            sent += 1
        except:
            pass

    await update.message.reply_text(f"Video sent to {sent} users.")
    return ConversationHandler.END


async def unknown(update, context):
    return


# ---- MAIN ----
def main():
    init_db()

    app = ApplicationBuilder().token(BOT_TOKEN).build()

    admin_conv = ConversationHandler(
        entry_points=[CommandHandler("admin", admin_cmd)],
        states={STATE_WAIT_PASSWORD: [MessageHandler(filters.TEXT, password_entered)]},
        fallbacks=[]
    )

    upload_conv = ConversationHandler(
        entry_points=[CommandHandler("upload", upload_cmd)],
        states={STATE_WAIT_FILE: [MessageHandler(filters.ALL, file_received)]},
        fallbacks=[]
    )

    text_conv = ConversationHandler(
        entry_points=[CommandHandler("sendads", sendads_cmd)],
        states={STATE_WAIT_AD_TEXT: [MessageHandler(filters.TEXT, receive_ads_text)]},
        fallbacks=[]
    )

    img_conv = ConversationHandler(
        entry_points=[CommandHandler("sendimgads", sendimgads_cmd)],
        states={STATE_WAIT_IMG: [MessageHandler(filters.PHOTO, img_received)]},
        fallbacks=[]
    )

    vid_conv = ConversationHandler(
        entry_points=[CommandHandler("sendvidads", sendvidads_cmd)],
        states={STATE_WAIT_VIDEO: [MessageHandler(filters.VIDEO, video_received)]},
        fallbacks=[]
    )

    app.add_handler(CommandHandler("start", start_handler))
    app.add_handler(admin_conv)
    app.add_handler(upload_conv)
    app.add_handler(text_conv)
    app.add_handler(img_conv)
    app.add_handler(vid_conv)

    app.add_handler(MessageHandler(filters.ALL, unknown))

    app.run_polling()


if __name__ == "__main__":
    main()
