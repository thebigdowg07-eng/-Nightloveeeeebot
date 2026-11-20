
import os
import logging
from collections import deque

from telegram import Update, BotCommand
from telegram.ext import (
    Application,
    CommandHandler,
    MessageHandler,
    ContextTypes,
    filters,
)
from telegram.error import TelegramError

logging.basicConfig(
    format="%(asctime)s - %(levelname)s - %(message)s",
    level=logging.INFO
)

TOKEN = os.getenv("BOT_TOKEN")
ADMIN_ID = os.getenv("ADMIN_ID")

if not TOKEN:
    raise RuntimeError("BOT_TOKEN environment variable is missing")

ADMIN_ID = int(ADMIN_ID) if ADMIN_ID else None

waiting_queue = deque()
waiting_users = set()
active_chats = {}


async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await update.message.reply_text(
        "👋 Welcome to Beautiful Liars!\n\n"
        "Chat anonymously with strangers.\n\n"
        "Use /find to find someone.\n"
        "Use /help to see all commands."
    )


async def help_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await update.message.reply_text(
        "📌 AVAILABLE COMMANDS\n\n"
        "/find - Find a stranger\n"
        "/next - Find another stranger\n"
        "/stop - End your chat\n"
        "/report - Report your partner\n"
        "/rules - View chat rules\n"
        "/about - About this bot\n"
        "/help - Show help"
    )


async def about(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await update.message.reply_text(
        "💬 Beautiful Liars\n\n"
        "Meet new people and enjoy anonymous conversations."
    )


async def rules(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await update.message.reply_text(
        "📜 CHAT RULES\n\n"
        "1. Respect everyone.\n"
        "2. No harassment or threats.\n"
        "3. Do not share personal information.\n"
        "4. Do not send illegal content.\n"
        "5. Report abusive users."
    )


async def find_partner(user_id, context):
    if user_id in active_chats:
        await context.bot.send_message(
            user_id, "You are already chatting. Use /next or /stop."
        )
        return

    if user_id in waiting_users:
        await context.bot.send_message(
            user_id, "⏳ You are already waiting for a stranger."
        )
        return

    partner = None

    while waiting_queue:
        candidate = waiting_queue.popleft()

        if (
            candidate != user_id
            and candidate in waiting_users
            and candidate not in active_chats
        ):
            waiting_users.discard(candidate)
            partner = candidate
            break

    if partner:
        active_chats[user_id] = partner
        active_chats[partner] = user_id

        await context.bot.send_message(
            user_id,
            "🎉 Stranger found! Say hello.\nUse /next or /stop anytime."
        )

        await context.bot.send_message(
            partner,
            "🎉 Stranger found! Say hello.\nUse /next or /stop anytime."
        )

    else:
        waiting_users.add(user_id)
        waiting_queue.append(user_id)

        await context.bot.send_message(
            user_id,
            "🔎 Searching for a stranger...\n"
            "Please wait until someone connects."
        )


async def find(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await find_partner(update.effective_user.id, context)


async def end_chat(user_id, context):
    waiting_users.discard(user_id)

    partner = active_chats.pop(user_id, None)

    if partner is not None:
        active_chats.pop(partner, None)

        await context.bot.send_message(
            partner,
            "❌ Your stranger has left the chat.\nUse /find to meet someone new."
        )

        return partner

    return None


async def stop(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_id = update.effective_user.id

    was_waiting = user_id in waiting_users
    partner = await end_chat(user_id, context)

    if partner:
        await update.message.reply_text("Chat ended.")
    elif was_waiting:
        await update.message.reply_text("Search cancelled.")
    else:
        await update.message.reply_text("You are not in an active chat.")


async def next_chat(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_id = update.effective_user.id

    was_waiting = user_id in waiting_users
    partner = await end_chat(user_id, context)

    if partner or was_waiting:
        await update.message.reply_text("Finding a new stranger...")
    else:
        await update.message.reply_text("🔎 Finding a stranger...")

    await find_partner(user_id, context)


async def report(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_id = update.effective_user.id
    partner = active_chats.get(user_id)

    if not partner:
        await update.message.reply_text(
            "You need to be chatting with someone to report them."
        )
        return

    if not ADMIN_ID:
        await update.message.reply_text(
            "Reporting is not configured yet. Please contact the bot administrator."
        )
        return

    reason = " ".join(context.args) or "No reason provided"

    try:
        await context.bot.send_message(
            ADMIN_ID,
            f"⚠️ USER REPORT\n\n"
            f"Reporter ID: {user_id}\n"
            f"Reported user ID: {partner}\n"
            f"Reason: {reason}"
        )

        await end_chat(user_id, context)

        await update.message.reply_text(
            "Your report has been sent. The chat has ended."
        )

    except TelegramError:
        await update.message.reply_text(
            "Could not send your report. Please try again later."
        )


async def relay_message(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_id = update.effective_user.id
    partner = active_chats.get(user_id)

    if not partner:
        if user_id in waiting_users:
            await update.message.reply_text(
                "⏳ Please wait for a stranger to connect."
            )
        else:
            await update.message.reply_text(
                "Use /find to start chatting with a stranger."
            )
        return

    try:
        await context.bot.copy_message(
            chat_id=partner,
            from_chat_id=user_id,
            message_id=update.message.message_id
        )
    except TelegramError:
        await update.message.reply_text(
            "Message could not be delivered. Try again."
        )


async def post_init(application: Application):
    await application.bot.set_my_commands([
        BotCommand("start", "Start the bot"),
        BotCommand("find", "Find a stranger"),
        BotCommand("next", "Find another stranger"),
        BotCommand("stop", "End your chat"),
        BotCommand("report", "Report a user"),
        BotCommand("help", "Show help"),
        BotCommand("about", "About this bot"),
        BotCommand("rules", "View chat rules"),
    ])


def main():
    app = (
        Application.builder()
        .token(TOKEN)
        .post_init(post_init)
        .build()
    )

    app.add_handler(CommandHandler("start", start))
    app.add_handler(CommandHandler("help", help_command))
    app.add_handler(CommandHandler("about", about))
    app.add_handler(CommandHandler("rules", rules))
    app.add_handler(CommandHandler("find", find))
    app.add_handler(CommandHandler("next", next_chat))
    app.add_handler(CommandHandler("stop", stop))
    app.add_handler(CommandHandler("report", report))

    app.add_handler(
        MessageHandler(filters.ALL & ~filters.COMMAND, relay_message)
    )

    print("Beautiful Liars bot is running...")
    app.run_polling()


if __name__ == "__main__":
    main()
