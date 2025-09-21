# RPG_Ecology_bot
import telebot
from telebot import types
import sqlite3

# === НАСТРОЙКА БОТА ===
BOT_TOKEN = '8200841553:AAEociqXVzBVxVieEKEyw6QjOcGFya2BvDY'
bot = telebot.TeleBot(BOT_TOKEN)


# === РАБОТА С БАЗОЙ ДАННЫХ ===
DB_NAME = 'players.db'

def init_db():
    """Создаёт таблицы при запуске"""
    conn = sqlite3.connect(DB_NAME)
    cursor = conn.cursor()

    # Основная таблица игроков
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

    # Таблица миссий
    cursor.execute('''
        CREATE TABLE IF NOT EXISTS missions (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            user_id INTEGER,
            title TEXT,
            reward INTEGER,
            completed BOOLEAN DEFAULT 0,
            FOREIGN KEY (user_id) REFERENCES players (user_id)
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


def remove_item(user_id, item):
    """Удаляет предмет из инвентаря"""
    player = get_player(user_id)
    if item in player['inventory']:
        player['inventory'].remove(item)
        new_inv = ", ".join(player['inventory'])
        conn = sqlite3.connect(DB_NAME)
        cursor = conn.cursor()
        cursor.execute('UPDATE players SET inventory = ? WHERE user_id = ?', (new_inv, user_id))
        conn.commit()
        conn.close()


def add_mission(user_id, title, reward):
    """Добавляет задание"""
    conn = sqlite3.connect(DB_NAME)
    cursor = conn.cursor()
    cursor.execute('''
        INSERT INTO missions (user_id, title, reward, completed)
        VALUES (?, ?, ?, 0)
    ''', (user_id, title, reward))
    conn.commit()
    conn.close()


def get_active_missions(user_id):
    """Получает незавершённые миссии"""
    conn = sqlite3.connect(DB_NAME)
    cursor = conn.cursor()
    cursor.execute('SELECT id, title, reward FROM missions WHERE user_id = ? AND completed = 0', (user_id,))
    rows = cursor.fetchall()
    conn.close()
    return rows


def complete_mission(mission_id, user_id):
    """Завершает миссию, даёт награду"""
    conn = sqlite3.connect(DB_NAME)
    cursor = conn.cursor()
    cursor.execute('SELECT reward FROM missions WHERE id = ? AND user_id = ?', (mission_id, user_id))
    row = cursor.fetchone()
    if not row:
        conn.close()
        return False
    reward = row[0]

    # Отметить как выполненную
    cursor.execute('UPDATE missions SET completed = 1 WHERE id = ?', (mission_id,))
    
    # Добавить деньги и хорошее дело
    cursor.execute('UPDATE players SET money = money + ?, good_deeds = good_deeds + 1 WHERE user_id = ?',
                   (reward, user_id))
    conn.commit()
    conn.close()
    return reward


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
    markup = types.InlineKeyboardMarkup(row_width=1)
    markup.add(
        types.InlineKeyboardButton("📍 Локации", callback_data="locations"),
        types.InlineKeyboardButton("🛒 Магазин", callback_data="shop"),
        types.InlineKeyboardButton("📋 Доступные задания", callback_data="available_missions"),
        types.InlineKeyboardButton("✅ Выполненные задания", callback_data="completed_missions"),
        types.InlineKeyboardButton("🎒 Инвентарь", callback_data="inventory"),
        types.InlineKeyboardButton("👤 Персонаж", callback_data="profile"),
        types.InlineKeyboardButton("🏆 Топ", callback_data="top_menu")
    )
    bot.send_message(chat_id, "🎮 Главное меню:", reply_markup=markup)


# === ЛОКАЦИИ И ЗАДАНИЯ ===
@bot.callback_query_handler(func=lambda call: call.data == "locations")
def show_locations(call):
    markup = types.InlineKeyboardMarkup()
    for loc in ["Лес", "Город", "Океан", "Завод"]:
        markup.add(types.InlineKeyboardButton(loc, callback_data=f"loc_{loc.lower()}"))
    markup.add(types.InlineKeyboardButton("⬅ Назад", callback_data="back_to_menu"))
    bot.edit_message_text("Выбери локацию:", call.message.chat.id, call.message.message_id, reply_markup=markup)


@bot.callback_query_handler(func=lambda call: call.data.startswith("loc_"))
def show_location_missions(call):
    loc = call.data.replace("loc_", "").capitalize()
    user_id = call.from_user.id
    player = get_player(user_id)

    markup = types.InlineKeyboardMarkup()
    rewards = {"низкое": (20, "пластиковая бутылка"), "среднее": (50, "перчатки"), "высокое": (100, "солнечная батарея")}
    
    for diff, (money, item) in rewards.items():
        req = "любой" if diff == "низкое" else "перчатки" if diff == "среднее" else "солнечная батарея"
        can_do = diff == "низкое" or req in player['inventory']
        text = f"{loc}: {diff.title()} ({money}💰)" + (" ✅" if can_do else " ❌")
        cb = f"mission_{loc}_{diff}" if can_do else "cannot_do"
        markup.add(types.InlineKeyboardButton(text, callback_data=cb))

    markup.add(types.InlineKeyboardButton("⬅ Назад", callback_data="locations"))
    bot.edit_message_text(f"Задания в {loc}:", call.message.chat.id, call.message.message_id, reply_markup=markup)


@bot.callback_query_handler(func=lambda call: call.data.startswith("mission_"))
def accept_mission(call):
    parts = call.data.split("_")
    loc, diff = parts[1], parts[2]
    reward = {"низкое": 20, "среднее": 50, "высокое": 100}[diff]

    add_mission(call.from_user.id, f"Уборка в {loc} ({diff})", reward)
    bot.send_message(call.message.chat.id, f"✅ Задание принято! Награда: {reward}💰")
    bot.answer_callback_query(call.id)


# === ОСТАЛЬНЫЕ КНОПКИ ===
@bot.callback_query_handler(func=lambda call: call.data == "shop")
def show_shop(call):
    items = [
        ("🧍‍♂️ Эко-сумка", 30),
        ("🧤 Перчатки", 50),
        ("🔋 Солнечная батарея", 150)
    ]
    markup = types.InlineKeyboardMarkup()
    for name, price in items:
        markup.add(types.InlineKeyboardButton(f"{name} ({price}💰)", callback_data=f"buy_{name.split()[1]}_{price}"))
    markup.add(types.InlineKeyboardButton("⬅ Назад", callback_data="back_to_menu"))
    bot.edit_message_text("🛒 Магазин:", call.message.chat.id, call.message.message_id, reply_markup=markup)


@bot.callback_query_handler(func=lambda call: call.data.startswith("buy_"))
def buy_item(call):
    parts = call.data.split("_")
    item_name = parts[1]
    price = int(parts[2])
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


@bot.callback_query_handler(func=lambda call: call.data == "available_missions")
def list_missions(call):
    missions = get_active_missions(call.from_user.id)
    if missions:
        text = "📋 Твои задания:\n" + "\n".join(f"{i+1}. {m[1]} — {m[2]}💰" for i, m in enumerate(missions))
    else:
        text = "📋 Нет активных заданий."
    bot.send_message(call.message.chat.id, text)
    bot.answer_callback_query(call.id)


@bot.callback_query_handler(func=lambda call: call.data == "completed_missions")
def complete_random_mission(call):
    missions = get_active_missions(call.from_user.id)
    if missions:
        mission_id = missions[0][0]  # первое задание
        reward = complete_mission(mission_id, call.from_user.id)
        bot.send_message(call.message.chat.id, f"✅ Задание выполнено! Получено: {reward}💰 и +1 доброе дело!")
    else:
        bot.send_message(call.message.chat.id, "❌ Нет заданий для выполнения.")
    bot.answer_callback_query(call.id)


@bot.callback_query_handler(func=lambda call: call.data == "inventory")
def show_inventory(call):
    items = get_player(call.from_user.id)['inventory']
    text = "🎒 Инвентарь:\n" + "\n".join(f"• {i}" for i in items) if items else "Пусто"
    bot.send_message(call.message.chat.id, text)
    bot.answer_callback_query(call.id)


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


@bot.callback_query_handler(func=lambda call: call.data == "back_to_menu")
def back_to_menu(call):
    show_main_menu(call.message.chat.id)
    bot.answer_callback_query(call.id)


# === ЗАПУСК ===
if __name__ == '__main__':
    init_db()
    print("Бот запущен...")
    bot.polling(none_stop=True)
