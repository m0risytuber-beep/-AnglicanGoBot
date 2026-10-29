# -AnglicanGoBot
Это бот в Telegram  по изучению английского  языка
import os
import random
import sqlite3
from datetime import date
from dotenv import load_dotenv
from telegram import Update, InlineKeyboardButton, InlineKeyboardMarkup
from telegram.ext import Application, CommandHandler, CallbackQueryHandler, ContextTypes

load_dotenv()
TOKEN = os.getenv("TELEGRAM_BOT_TOKEN")
DB = os.getenv("DATABASE_PATH", "anglican_go.db")

LESSONS = [
    ("Greetings", "Hello — Привет\nGood morning — Доброе утро\nGoodbye — До свидания", [
        ("Как сказать «Привет»?", ["Hello", "Thanks", "Goodbye"], 0),
        ("Что значит Goodbye?", ["Спасибо", "До свидания", "Пожалуйста"], 1),
    ]),
    ("Numbers", "One — один\nTwo — два\nThree — три\nFour — четыре\nFive — пять", [
        ("Как будет «три»?", ["Two", "Three", "Five"], 1),
        ("Что значит One?", ["Один", "Четыре", "Два"], 0),
    ]),
    ("Colors", "Red — красный\nBlue — синий\nGreen — зелёный\nYellow — жёлтый", [
        ("Как будет «синий»?", ["Red", "Blue", "Green"], 1),
        ("Что значит Yellow?", ["Жёлтый", "Зелёный", "Красный"], 0),
    ]),
    ("Family", "Mother — мама\nFather — папа\nSister — сестра\nBrother — брат", [
        ("Как будет «сестра»?", ["Brother", "Mother", "Sister"], 2),
        ("Что значит Father?", ["Папа", "Мама", "Брат"], 0),
    ]),
    ("School", "Book — книга\nPen — ручка\nTeacher — учитель\nSchool — школа", [
        ("Как будет «книга»?", ["Book", "Pen", "School"], 0),
        ("Что значит Teacher?", ["Ученик", "Учитель", "Книга"], 1),
    ]),
]

DAILY_TASKS = [
    ("Пройди 1 урок", "lesson", 1),
    ("Ответь правильно на 3 вопроса", "correct", 3),
    ("Заработай 20 XP", "xp", 20),
]
CASES = {
    "case_basic": ("📦 Обычный кейс", 40, (15, 35), (10, 25)),
    "case_middle": ("💎 Средний кейс", 100, (45, 100), (30, 60)),
    "case_gold": ("👑 Золотой кейс ×3", 220, (180, 450), (60, 120)),
}

def connect():
    con = sqlite3.connect(DB)
    con.execute("""CREATE TABLE IF NOT EXISTS users (
        user_id INTEGER PRIMARY KEY, name TEXT, xp INTEGER NOT NULL DEFAULT 0,
        crystals INTEGER NOT NULL DEFAULT 100, lessons INTEGER NOT NULL DEFAULT 0,
        correct INTEGER NOT NULL DEFAULT 0, streak INTEGER NOT NULL DEFAULT 0,
        last_day TEXT, daily_date TEXT, daily_claimed TEXT DEFAULT '',
        inventory TEXT DEFAULT ''
    )""")
    con.commit()
    return con

def ensure_user(tg_user):
    con = connect()
    con.execute("INSERT OR IGNORE INTO users(user_id,name) VALUES(?,?)",
                (tg_user.id, tg_user.first_name or "Ученик"))
    con.execute("UPDATE users SET name=? WHERE user_id=?",
                (tg_user.first_name or "Ученик", tg_user.id))
    con.commit()
    row = con.execute("SELECT user_id,name,xp,crystals,lessons,correct,streak,last_day,daily_date,daily_claimed,inventory FROM users WHERE user_id=?",
                      (tg_user.id,)).fetchone()
    con.close()
    return row

def update(uid, sql, params=()):
    con = connect()
    con.execute(sql, (*params, uid))
    con.commit()
    con.close()

def main_menu():
    return InlineKeyboardMarkup([
        [InlineKeyboardButton("📚 Уроки", callback_data="lessons"),
         InlineKeyboardButton("👤 Профиль", callback_data="profile")],
        [InlineKeyboardButton("🎁 Кейсы", callback_data="cases"),
         InlineKeyboardButton("🎯 Задания дня", callback_data="daily")],
        [InlineKeyboardButton("🏆 Рейтинг", callback_data="leaderboard"),
         InlineKeyboardButton("ℹ️ Помощь", callback_data="help")]
    ])

async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    ensure_user(update.effective_user)
    await update.message.reply_text(
        f"🇬🇧 Добро пожаловать в Anglican Go, {update.effective_user.first_name}!\n\n"
        "Учись английскому, зарабатывай XP и кристаллы, открывай кейсы.\n"
        "Твой профиль автоматически сохраняется.", reply_markup=main_menu())

async def profile(update, context):
    row = ensure_user(update.effective_user)
    text = (f"👤 Профиль: {row[1]}\n⭐ XP: {row[2]}\n💎 Кристаллы: {row[3]}\n"
            f"📚 Пройдено уроков: {row[4]}\n✅ Верных ответов: {row[5]}\n🔥 Серия: {row[6]} дн.")
    await update.effective_message.reply_text(text, reply_markup=main_menu())

async def lessons(update, context):
    kb = [[InlineKeyboardButton(f"📘 {i+1}. {l[0]}", callback_data=f"lesson:{i}")]
          for i,l in enumerate(LESSONS)]
    kb.append([InlineKeyboardButton("⬅️ Меню", callback_data="menu")])
    await update.callback_query.edit_message_text("Выбери урок:", reply_markup=InlineKeyboardMarkup(kb))

async def show_lesson(q, idx):
    title, content, questions = LESSONS[idx]
    await q.edit_message_text(f"📘 {title}\n\n{content}\n\nКогда будешь готов, начни тест.",
        reply_markup=InlineKeyboardMarkup([
            [InlineKeyboardButton("📝 Начать тест", callback_data=f"quiz:{idx}:0")],
            [InlineKeyboardButton("⬅️ К урокам", callback_data="lessons")]]))

async def quiz(q, context, idx, qi):
    title, content, questions = LESSONS[idx]
    if qi >= len(questions):
        uid = q.from_user.id
        update(uid, "UPDATE users SET lessons=lessons+1 WHERE user_id=?")
        update(uid, "UPDATE users SET xp=xp+15, crystals=crystals+10 WHERE user_id=?")
        context.user_data.pop("quiz", None)
        await q.edit_message_text(f"🎉 Урок «{title}» пройден!\n+15 XP · +10 💎",
            reply_markup=InlineKeyboardMarkup([[InlineKeyboardButton("📚 Другой урок", callback_data="lessons"),
                                                InlineKeyboardButton("🏠 Меню", callback_data="menu")]]))
        return
    question, options, answer = questions[qi]
    context.user_data["quiz"] = {"lesson":idx, "question":qi}
    kb = [[InlineKeyboardButton(opt, callback_data=f"answer:{i}")] for i,opt in enumerate(options)]
    await q.edit_message_text(f"📝 {title} · Вопрос {qi+1}/{len(questions)}\n\n{question}",
                              reply_markup=InlineKeyboardMarkup(kb))

async def cases(update, context):
    row = ensure_user(update.effective_user)
    kb = []
    for key, (name, price, _, _) in CASES.items():
        kb.append([InlineKeyboardButton(f"{name} — {price} 💎", callback_data=key)])
    kb.append([InlineKeyboardButton("⬅️ Меню", callback_data="menu")])
    await update.callback_query.edit_message_text(f"🎁 Выбери кейс\nТвой баланс: {row[3]} 💎",
                                                  reply_markup=InlineKeyboardMarkup(kb))

async def daily(update, context):
    row = ensure_user(update.effective_user)
    today = date.today().isoformat()
    claimed = row[9] if row[8] == today else ""
    lines = ["🎯 Ежедневные задания", "Награда за каждое: кейс соответствующего уровня.", ""]
    kb = []
    for i, (name, kind, target) in enumerate(DAILY_TASKS):
        done = (kind=="lesson" and row[4] >= 1) or (kind=="correct" and row[5] >= target) or (kind=="xp" and row[2] >= target)
        got = str(i) in claimed
        lines.append(f"{'✅' if done else '▫️'} {name} — {'получено' if got else ('готово забрать' if done else 'в процессе')}")
        if done and not got:
            kb.append([InlineKeyboardButton(f"Забрать награду: {['📦 обычный','💎 средний','👑 золотой'][i]}", callback_data=f"claim:{i}")])
    kb.append([InlineKeyboardButton("⬅️ Меню", callback_data="menu")])
    await update.callback_query.edit_message_text("\n".join(lines), reply_markup=InlineKeyboardMarkup(kb))

async def callback(update: Update, context: ContextTypes.DEFAULT_TYPE):
    q = update.callback_query
    await q.answer()
    ensure_user(q.from_user)
    data = q.data
    if data == "menu":
        await q.edit_message_text("🇬🇧 Anglican Go — главное меню", reply_markup=main_menu())
    elif data == "profile":
        row = ensure_user(q.from_user)
        await q.edit_message_text(f"👤 {row[1]}\n⭐ XP: {row[2]}\n💎 Кристаллы: {row[3]}\n📚 Уроков: {row[4]}\n✅ Верных ответов: {row[5]}\n🔥 Серия: {row[6]} дн.",
            reply_markup=main_menu())
    elif data == "lessons": await lessons(update, context)
    elif data.startswith("lesson:"): await show_lesson(q, int(data.split(":")[1]))
    elif data.startswith("quiz:"):
        _, idx, qi = data.split(":")
        await quiz(q, context, int(idx), int(qi))
    elif data.startswith("answer:"):
        state = context.user_data.get("quiz")
        if not state:
            await q.edit_message_text("Тест закончился. Выбери урок заново.", reply_markup=main_menu()); return
        idx, qi = state["lesson"], state["question"]
        question, options, answer = LESSONS[idx][2][qi]
        chosen = int(data.split(":")[1])
        if chosen == answer:
            update(q.from_user.id, "UPDATE users SET xp=xp+5, crystals=crystals+2, correct=correct+1 WHERE user_id=?")
            msg = "✅ Верно! +5 XP · +2 💎"
        else:
            msg = f"❌ Не совсем. Правильный ответ: {options[answer]}"
        await q.edit_message_text(msg, reply_markup=InlineKeyboardMarkup([
            [InlineKeyboardButton("➡️ Дальше", callback_data=f"quiz:{idx}:{qi+1}")],
            [InlineKeyboardButton("🏠 Меню", callback_data="menu")]]))
    elif data == "cases": await cases(update, context)
    elif data in CASES:
        name, price, gems, xp = CASES[data]
        row = ensure_user(q.from_user)
        if row[3] < price:
            await q.answer("Не хватает кристаллов 💎", show_alert=True); return
        reward_gems = random.randint(*gems)
        reward_xp = random.randint(*xp)
        if data == "case_gold": reward_gems *= 3
        update(q.from_user.id, "UPDATE users SET crystals=crystals-?+?, xp=xp+? WHERE user_id=?",
               (price, reward_gems, reward_xp))
        await q.edit_message_text(f"🎉 Открыт {name}!\n\nНаграда: +{reward_gems} 💎 и +{reward_xp} XP",
            reply_markup=InlineKeyboardMarkup([[InlineKeyboardButton("🎁 Ещё кейсы", callback_data="cases"),
                                                InlineKeyboardButton("🏠 Меню", callback_data="menu")]]))
    elif data == "daily": await daily(update, context)
    elif data.startswith("claim:"):
        i = int(data.split(":")[1])
        row = ensure_user(q.from_user)
        today = date.today().isoformat()
        claimed = row[9] if row[8] == today else ""
        if str(i) in claimed:
            await q.answer("Уже получено!", show_alert=True); return
        kind = DAILY_TASKS[i][1]
        done = (kind=="lesson" and row[4] >= 1) or (kind=="correct" and row[5] >= DAILY_TASKS[i][2]) or (kind=="xp" and row[2] >= DAILY_TASKS[i][2])
        if not done:
            await q.answer("Задание ещё не выполнено.", show_alert=True); return
        key = ["case_basic", "case_middle", "case_gold"][i]
        update(q.from_user.id, "UPDATE users SET inventory=inventory||? , daily_date=?, daily_claimed=? WHERE user_id=?",
               (key+",", today, claimed+str(i),))
        await q.edit_message_text(f"🎁 {CASES[key][0]} добавлен в награды!\n(В этой версии награды за задания зачисляются как бонусный кейс в профиле.)",
            reply_markup=main_menu())
    elif data == "leaderboard":
        con = connect()
        rows = con.execute("SELECT name,xp FROM users ORDER BY xp DESC LIMIT 10").fetchall()
        con.close()
        text = "🏆 Рейтинг по XP\n\n" + ("\n".join(f"{i+1}. {r[0]} — {r[1]} XP" for i,r in enumerate(rows)) or "Пока нет игроков.")
        await q.edit_message_text(text, reply_markup=InlineKeyboardMarkup([[InlineKeyboardButton("⬅️ Меню", callback_data="menu")]]))
    elif data == "help":
        await q.edit_message_text("📚 Выбирай уроки, отвечай на вопросы и получай XP и кристаллы.\n🎁 Кейсы покупаются за кристаллы.\n💾 Прогресс сохраняется в базе SQLite на сервере, где запущен бот.",
            reply_markup=InlineKeyboardMarkup([[InlineKeyboardButton("⬅️ Меню", callback_data="menu")]]))

async def stats(update, context):
    row = ensure_user(update.effective_user)
    await update.message.reply_text(f"⭐ {row[2]} XP · 💎 {row[3]} кристаллов · 📚 {row[4]} уроков")

def main():
    if not TOKEN:
        raise SystemExit("Не задан TELEGRAM_BOT_TOKEN. Скопируй .env.example в .env и добавь токен BotFather.")
    app = Application.builder().token(TOKEN).build()
    app.add_handler(CommandHandler("start", start))
    app.add_handler(CommandHandler("profile", profile))
    app.add_handler(CommandHandler("stats", stats))
    app.add_handler(CallbackQueryHandler(callback))
    print("Anglican Go запущен. Для остановки нажми Ctrl+C.")
    app.run_polling()

if __name__ == "__main__":
    main()Anglican Go — Telegram-бот

## Возможности
- Профиль Telegram-пользователя автоматически создаётся при `/start`.
- SQLite сохраняет XP, кристаллы, уроки и ответы.
- Уроки и мини-тесты.
- Покупаемые кейсы за кристаллы.
- Ежедневные задания и таблица лидеров.
- Кнопочное 
