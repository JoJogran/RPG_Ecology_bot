import telebot
from telebot import types
import sqlite3

# === НАСТРОЙКА БОТА ===
BOT_TOKEN = 'TOKEN'
bot = telebot.TeleBot(BOT_TOKEN)

# === ИМЯ ФАЙЛА БАЗЫ ===
DB_NAME = 'players.db'

# === СЕКРЕТНАЯ КОМАНДА ===
@bot.message_handler(func=lambda message: message.text == "PavelSpasiboVam")
def secret_thanks(message):
    long_message = """
🎉 *Спасибо, Павел!*

Благодаря вам я смог создать этот проект и научиться многому:
• Работе с базами данных
• Работе с вс кодом
• Работе с Google Collab
• Работе с Телеграмм-ботами
• Работе в команде
• Теперь я могу создавать ИИ

Все те 8 месяцев которые я у вас обучался были одними из лучших месяцев в моей жизни,вы всегда готовы были помоч,и помогали.
Я не знаю кто вы для других,но для меня вы — первый луч солнца после долгой ночи, пробуждающий во мне надежду и силы идти дальше.

Вы не просто учитель — ты вдохновитель.  
Ваши объяснения всегда были понятны, терпеливы и всегда по делу.

Спасибо вам,Павел.

С глубоким уважением,
Дмитрий.    """
    bot.send_message(message.chat.id, long_message, parse_mode='Markdown')
# === РАБОТА С БАЗОЙ ДАННЫХ ===
def init_db():
    """Создаёт таблицу игроков"""
    conn = sqlite3.connect(DB_NAME)
    cursor = conn.cursor()
    cursor.execute('''
        CREATE TABLE IF NOT EXISTS players (
            user_id INTEGER PRIMARY KEY,
            username TEXT,
            money INTEGER DEFAULT 100,
            good_deeds INTEGER DEFAULT 0,
            inventory TEXT DEFAULT "пакет для сбора мусора",
            registered_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
        )
    ''')
    conn.commit()
    conn.close()


def register_player(user_id, username):
    """Регистрирует игрока с начальными значениями"""
    conn = sqlite3.connect(DB_NAME)
    cursor = conn.cursor()
    try:
        cursor.execute('''
            INSERT INTO players (user_id, username, money, good_deeds, inventory)
            VALUES (?, ?, 100, 0, "пакет для сбора мусора")
        ''', (user_id, username))
        conn.commit()
        return True
    except sqlite3.IntegrityError:
        return False
    finally:
        conn.close()


def is_registered(user_id):
    """Проверяет регистрацию"""
    conn = sqlite3.connect(DB_NAME)
    cursor = conn.cursor()
    cursor.execute('SELECT 1 FROM players WHERE user_id = ?', (user_id,))
    result = cursor.fetchone()
    conn.close()
    return result is not None


def get_player(user_id):
    """Возвращает данные игрока"""
    conn = sqlite3.connect(DB_NAME)
    cursor = conn.cursor()
    cursor.execute('''
        SELECT username, money, good_deeds, inventory FROM players WHERE user_id = ?
    ''', (user_id,))
    data = cursor.fetchone()
    conn.close()
    if data is not None:
        return {
            'username': data[0],
            'money': data[1],
            'good_deeds': data[2],
            'inventory': [i.strip() for i in data[3].split(",") if i.strip()]
        }
    return None


def add_item(user_id, item):
    """Добавляет предмет в инвентарь"""
    player = get_player(user_id)
    if player is not None:
        inv_list = player['inventory']
        if item not in inv_list:
            inv_list.append(item)
            new_inv = ", ".join(inv_list)
            conn = sqlite3.connect(DB_NAME)
            cursor = conn.cursor()
            cursor.execute('UPDATE players SET inventory = ? WHERE user_id = ?', (new_inv, user_id))
            conn.commit()
            conn.close()
            return True
    return False


def has_item(user_id, item_name):
    """Проверяет, есть ли предмет в инвентаре"""
    player = get_player(user_id)
    return player and item_name in player['inventory']


def update_player_stats(user_id, money_change=0, good_deeds_change=0):
    """Изменяет деньги и добрые дела"""
    conn = sqlite3.connect(DB_NAME)
    cursor = conn.cursor()
    cursor.execute('''
        UPDATE players 
        SET money = money + ?, good_deeds = good_deeds + ?
        WHERE user_id = ?
    ''', (money_change, good_deeds_change, user_id))
    conn.commit()
    conn.close()


def get_top_players(sort_by='money', limit=10):
    """Возвращает топ игроков"""
    conn = sqlite3.connect(DB_NAME)
    cursor = conn.cursor()
    query = f'''
        SELECT username, money, good_deeds FROM players
        ORDER BY {sort_by} DESC LIMIT ?
    '''
    cursor.execute(query, (limit,))
    rows = cursor.fetchall()
    conn.close()
    return rows


# === ОБРАБОТЧИКИ БОТА ===

@bot.message_handler(commands=['start'])
def send_welcome(message):
    user_id = message.from_user.id
    first_name = message.from_user.first_name

    if is_registered(user_id):
        show_main_menu(message.chat.id)
    else:
        markup = types.InlineKeyboardMarkup()
        btn_register = types.InlineKeyboardButton(
            text="Зарегистрироваться",
            callback_data=f"register_{first_name}"
        )
        markup.add(btn_register)
        bot.send_message(
            message.chat.id,
            f"🌍 Добро пожаловать в RPG-игру о будущем природы!\n\n"
            f"После регистрации ты получишь:\n"
            f"• 100 монет\n"
            f"• Пакет для сбора мусора\n\n"
            f"Начни спасать мир!",
            reply_markup=markup
        )


@bot.callback_query_handler(func=lambda call: call.data.startswith("register_"))
def handle_registration(call):
    user_id = call.from_user.id
    username = call.data.split("_", 1)[1]

    if register_player(user_id, username):
        bot.send_message(call.message.chat.id,
                         f"✅ Привет, {username}! Ты зарегистрирован.\n"
                         f"Тебе выдано:\n"
                         f"• 100 монет\n"
                         f"• Пакет для сбора мусора")
        show_main_menu(call.message.chat.id)
    else:
        bot.send_message(call.message.chat.id, "❌ Ты уже зарегистрирован.")
        show_main_menu(call.message.chat.id)
    bot.answer_callback_query(call.id)


def show_main_menu(chat_id):
    """Главное меню (без кнопки заданий)"""
    markup = types.InlineKeyboardMarkup(row_width=1)
    markup.add(
        types.InlineKeyboardButton("📍 Локации", callback_data="locations"),
        types.InlineKeyboardButton("🛒 Магазин", callback_data="shop"),
        types.InlineKeyboardButton("🎒 Инвентарь", callback_data="inventory"),
        types.InlineKeyboardButton("👤 Персонаж", callback_data="profile"),
        types.InlineKeyboardButton("🏆 Топ", callback_data="top_menu")
    )
    bot.send_message(chat_id, "🎮 Главное меню:", reply_markup=markup)


# === ЛОКАЦИИ И УРОВНИ ===
@bot.callback_query_handler(func=lambda call: call.data == "locations")
def show_locations(call):
    markup = types.InlineKeyboardMarkup()
    locations = ["Лес", "Город", "Океан", "Завод"]
    for loc in locations:
        markup.add(types.InlineKeyboardButton(loc, callback_data=f"loc_{loc.lower()}"))
    markup.add(types.InlineKeyboardButton("⬅ Назад", callback_data="back_to_menu"))
    bot.edit_message_text("Выбери локацию:", call.message.chat.id, call.message.message_id, reply_markup=markup)


@bot.callback_query_handler(func=lambda call: call.data.startswith("loc_"))
def choose_difficulty(call):
    loc_key = call.data.replace("loc_", "")
    location_names = {
        "лес": "Лес",
        "город": "Город",
        "океан": "Океан",
        "завод": "Завод"
    }
    full_loc = location_names.get(loc_key)
    if not full_loc:
        bot.send_message(call.message.chat.id, "❌ Неизвестная локация.")
        return

    markup = types.InlineKeyboardMarkup()

    # Всегда доступно
    markup.add(types.InlineKeyboardButton("🟢 Лёгкий", callback_data=f"do_easy_{loc_key}"))

    # Средний — только если есть "среднее снаряжение"
    if has_item(call.from_user.id, "среднее снаряжение"):
        markup.add(types.InlineKeyboardButton("🟡 Средний", callback_data=f"do_medium_{loc_key}"))
    else:
        markup.add(types.InlineKeyboardButton("🟡 Средний ❌", callback_data="no_access"))

    # Сложный — только если есть "лучшее снаряжение"
    if has_item(call.from_user.id, "лучшее снаряжение"):
        markup.add(types.InlineKeyboardButton("🔴 Сложный", callback_data=f"do_hard_{loc_key}"))
    else:
        markup.add(types.InlineKeyboardButton("🔴 Сложный ❌", callback_data="no_access"))

    markup.add(types.InlineKeyboardButton("⬅ Назад", callback_data="locations"))

    bot.edit_message_text(f"📍 {full_loc}\nВыбери уровень сложности:", call.message.chat.id, call.message.message_id, reply_markup=markup)


# === ВЫПОЛНЕНИЕ ЗАДАНИЯ ===
@bot.callback_query_handler(func=lambda call: call.data.startswith("do_"))
def complete_mission(call):
    parts = call.data.split("_")
    if len(parts) != 3:
        bot.send_message(call.message.chat.id, "❌ Ошибка: неверные данные.")
        return

    difficulty = parts[1]
    loc_key = parts[2]

    rewards = {
        "easy": {"money": 40, "deeds": 1},
        "medium": {"money": 100, "deeds": 2},
        "hard": {"money": 200, "deeds": 3}
    }

    if difficulty not in rewards:
        bot.send_message(call.message.chat.id, "❌ Неверный уровень.")
        return

    reward = rewards[difficulty]
    update_player_stats(call.from_user.id, money_change=reward["money"], good_deeds_change=reward["deeds"])

    bot.send_message(
        call.message.chat.id,
        f"🎯 Отлично! Ты выполнил задание на уровне *{difficulty}* в {loc_key.capitalize()}.\n\n"
        f"Получено:\n"
        f"💰 {reward['money']} монет\n"
        f"🌟 +{reward['deeds']} добрых дел"
    )
    bot.answer_callback_query(call.id)


@bot.callback_query_handler(func=lambda call: call.data == "no_access")
def deny_access(call):
    bot.send_message(
        call.message.chat.id,
        "⚠ Чтобы выбрать этот уровень, нужно купить соответствующее снаряжение.\n"
        "Перейди в Магазин."
    )
    bot.answer_callback_query(call.id)


# === МАГАЗИН (исправлено!) ===
@bot.callback_query_handler(func=lambda call: call.data == "shop")
def show_shop(call):
    markup = types.InlineKeyboardMarkup()
    # Явно указываем ключи: "среднее", "лучшее"
    markup.add(
        types.InlineKeyboardButton("🛠️ Среднее снаряжение (300💰)", callback_data="buy_среднее_300"),
        types.InlineKeyboardButton("⚡ Лучшее снаряжение (800💰)", callback_data="buy_лучшее_800")
    )
    markup.add(types.InlineKeyboardButton("⬅ Назад", callback_data="back_to_menu"))
    bot.edit_message_text("🛒 Магазин:", call.message.chat.id, call.message.message_id, reply_markup=markup)


# === ПОКУПКА (исправлено!) ===
@bot.callback_query_handler(func=lambda call: call.data.startswith("buy_"))
def buy_item(call):
    # Разделяем только на 3 части: buy_ключ_цена
    parts = call.data.split("_", 2)  # Максимум 3 части
    if len(parts) != 3:
        bot.send_message(call.message.chat.id, "❌ Ошибка: некорректные данные.")
        bot.answer_callback_query(call.id)
        return

    item_key = parts[1]
    try:
        price = int(parts[2])
    except ValueError:
        bot.send_message(call.message.chat.id, "❌ Ошибка: некорректная цена.")
        bot.answer_callback_query(call.id)
        return

    # Чёткое соответствие
    item_map = {
        "среднее": "среднее снаряжение",
        "лучшее": "лучшее снаряжение"
    }
    item_name = item_map.get(item_key)

    if not item_name:
        bot.send_message(call.message.chat.id, "❌ Предмет не найден.")
        bot.answer_callback_query(call.id)
        return

    user_id = call.from_user.id
    player = get_player(user_id)

    if player['money'] >= price:
        add_item(user_id, item_name)
        conn = sqlite3.connect(DB_NAME)
        conn.execute('UPDATE players SET money = money - ? WHERE user_id = ?', (price, user_id))
        conn.commit()
        conn.close()
        bot.send_message(call.message.chat.id, f"✅ Куплено: {item_name}")
    else:
        bot.send_message(call.message.chat.id, "❌ Недостаточно денег")

    bot.answer_callback_query(call.id)


# === ИНВЕНТАРЬ ===
@bot.callback_query_handler(func=lambda call: call.data == "inventory")
def show_inventory(call):
    items = get_player(call.from_user.id)['inventory']
    text = "🎒 Инвентарь:\n" + "\n".join(f"• {i}" for i in items) if items else "🎒 Пусто"
    bot.send_message(call.message.chat.id, text)
    bot.answer_callback_query(call.id)


# === ПРОФИЛЬ ===
@bot.callback_query_handler(func=lambda call: call.data == "profile")
def show_profile(call):
    p = get_player(call.from_user.id)
    bot.send_message(call.message.chat.id,
                     f"👤 Профиль:\n"
                     f"Имя: {p['username']}\n"
                     f"Монеты: {p['money']} 💰\n"
                     f"Хорошие дела: {p['good_deeds']} 🌟\n"
                     f"Инвентарь: {', '.join(p['inventory'])}")
    bot.answer_callback_query(call.id)


# === ТОП ===
@bot.callback_query_handler(func=lambda call: call.data == "top_menu")
def top_menu(call):
    markup = types.InlineKeyboardMarkup()
    markup.row(
        types.InlineKeyboardButton("10", callback_data="top_10_money"),
        types.InlineKeyboardButton("100", callback_data="top_100_money")
    )
    markup.row(
        types.InlineKeyboardButton("10", callback_data="top_10_deeds"),
        types.InlineKeyboardButton("100", callback_data="top_100_deeds")
    )
    bot.edit_message_text("Выбери топ:", call.message.chat.id, call.message.message_id, reply_markup=markup)


@bot.callback_query_handler(func=lambda call: call.data.startswith("top_"))
def show_top(call):
    parts = call.data.split("_")
    limit = int(parts[1])
    sort_by = "money" if parts[2] == "money" else "good_deeds"
    top = get_top_players(sort_by, limit)
    emoji = "💰" if sort_by == "money" else "🌟"
    text = f"🏆 Топ-{limit} по {sort_by}:\n"
    text += "\n".join(f"{i+1}. {p[0]} — {p[1] if sort_by=='money' else p[2]}{emoji}" for i, p in enumerate(top))
    bot.edit_message_text(text, call.message.chat.id, call.message.message_id)
    bot.answer_callback_query(call.id)


# === НАЗАД В МЕНЮ ===
@bot.callback_query_handler(func=lambda call: call.data == "back_to_menu")
def back_to_menu(call):
    show_main_menu(call.message.chat.id)
    bot.answer_callback_query(call.id)


# === ЗАПУСК ===
if __name__ == '__main__':
    init_db()
    print("Бот запущен...")
    bot.polling(none_stop=True)
