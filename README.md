import telebot
from telebot import types
import json
import os

TOKEN = "8205228989:AAH9DJuWYXg8HNdTRJpBZsTFC2dfg-w2vRY"
bot = telebot.TeleBot(TOKEN)
DATA_FILE = "raspisanie.json"
# Замените на ваш реальный ID, если он отличается
admin_id = 8404436066

# Исходное расписание (используется, если файл еще не создан)
default_raspisanie = {
    "Понедельник": {
        '13:00': None,
        '14:00': None,
        '15:00': None,
        '16:00': None,
        '17:00': 'Sergey',
        '18:00': None,
        '19:00': 'Cloudy',
        '20:00': 'Cloudy',
    },
    'Вторник': {
        '13:00': None,
        '14:00': None,
        '15:00': None,
        '16:00': 'Sunny',
        '17:00': 'Cloudy',
        '18:00': None,
        '19:00': 'Cloudy',
        '20:00': 'Cloudy',
    },
    "Среда": {
        '13:00': None,
        '14:00': None,
        '15:00': None,
        '16:00': None,
        '17:00': 'Sergey',
        '18:00': None,
        '19:00': 'Cloudy',
        '20:00': 'Cloudy',
    },
    'Четверг': {
        '13:00': None,
        '14:00': None,
        '15:00': None,
        '16:00': 'Sunny',
        '17:00': 'Cloudy',
        '18:00': 'Cloudy',
        '19:00': 'Cloudy',
        '20:00': None,
    },
    "Пятница": {
        '13:00': None,
        '14:00': None,
        '15:00': None,
        '16:00': None,
        '17:00': 'Sergey',
        '18:00': None,
        '19:00': 'Cloudy',
        '20:00': 'Cloudy',
    },
    'Суббота': {
        '13:00': None,
        '14:00': None,
        '15:00': None,
        '16:00': 'Sunny',
        '17:00': 'Cloudy',
        '18:00': 'Cloudy',
        '19:00': None,
        '20:00': 'Cloudy',
    }
}

def load_data():
    if os.path.exists(DATA_FILE):
        with open(DATA_FILE, 'r', encoding='utf-8') as f:
            return json.load(f)
    return default_raspisanie

raspisanie = load_data()

def save_data():
    with open(DATA_FILE, 'w', encoding='utf-8') as f:
        json.dump(raspisanie, f, ensure_ascii=False, indent=4)

def get_user_id_string(message):
    user_name = message.from_user.first_name
    username = f" (@{message.from_user.username})" if message.from_user.username else ""
    return f"{user_name}{username}"

@bot.message_handler(commands=['start', 'help'])
def start(message):
    markup = types.ReplyKeyboardMarkup(resize_keyboard=True)
    btn1 = types.KeyboardButton("Запись на урок")
    btn2 = types.KeyboardButton("Показать расписание работы")
    btn3 = types.KeyboardButton("Мои записи / Отмена")
    markup.add(btn1, btn2, btn3)
    bot.send_message(message.chat.id, "Привет! Я бот-помощник. Выберите действие:", reply_markup=markup)

@bot.message_handler(func=lambda message: message.text == "Показать расписание работы")
def show_schedule(message):
    global raspisanie
    raspisanie = load_data()
    text = " *Свободное время для записи:*\n\n"

    for day, times in raspisanie.items():
        free_slots = [time for time, status in times.items() if status is None]
        if free_slots:
            text += f"*{day}*:\n`{', '.join(free_slots)}` \n\n"
        else:
            text += f"*{day}*:\n_Нет свободных мест_ \n\n"

    bot.send_message(message.chat.id, text, parse_mode="Markdown")

@bot.message_handler(func=lambda message: message.text == "Мои записи / Отмена")
def show_my_records(message):
    user_id_str = get_user_id_string(message)
    user_records = []

    # Ищем все совпадения имени пользователя в словаре
    for day, times in raspisanie.items():
        for time, student in times.items():
            if student == user_id_str:
                user_records.append((day, time))

    if not user_records:
        bot.send_message(message.chat.id, "У вас пока нет активных записей.")
        return

    markup = types.ReplyKeyboardMarkup(resize_keyboard=True, row_width=1)
    text = "🗓 *Ваши записи:*\n\n"
    for day, time in user_records:
        text += f"🔸 {day} в {time}\n"
        # Создаем кнопку отмены для каждой записи
        markup.add(types.KeyboardButton(f"❌ Отменить: {day} {time}"))

    markup.add(types.KeyboardButton("В главное меню"))
    bot.send_message(message.chat.id, text + "\nВыберите запись, чтобы отменить её, или вернитесь назад:",
                     reply_markup=markup, parse_mode="Markdown")

@bot.message_handler(func=lambda message: message.text.startswith("❌ Отменить:"))
def cancel_record(message):
    try:
        # Извлекаем день и время из текста кнопки
        parts = message.text.replace("❌ Отменить: ", "").split(" ")
        day = parts[0]
        time = parts[1]
        user_id_str = get_user_id_string(message)

        # Проверяем, действительно ли это запись этого пользователя (защита от хитрецов)
        if raspisanie[day][time] == user_id_str:
            raspisanie[day][time] = None
            save_data()
            bot.send_message(message.chat.id, f"✅ Ваша запись на *{day}* в *{time}* успешно отменена.",
                             parse_mode="Markdown")
            bot.send_message(admin_id, f"⚠️ *Отмена записи!*\nКлиент {user_id_str} отменил урок на {day} в {time}.",
                             parse_mode="Markdown")
        else:
            bot.send_message(message.chat.id, "Ошибка: это не ваша запись.")
    except Exception as e:
        bot.send_message(message.chat.id, "Произошла ошибка при отмене.")

    start(message)

@bot.message_handler(func=lambda message: message.text == "Запись на урок")
def choose_day(message):
    markup = types.ReplyKeyboardMarkup(resize_keyboard=True, row_width=3)
    for day in raspisanie.keys():
        markup.add(types.KeyboardButton(day))
    markup.add(types.KeyboardButton("В главное меню"))
    msg = bot.send_message(message.chat.id, "Выберите день недели:", reply_markup=markup)
    bot.register_next_step_handler(msg, step_2_ask_time)

def step_2_ask_time(message):
    day = message.text
    if day == "В главное меню":
        start(message)
        return

    if day not in raspisanie:
        bot.send_message(message.chat.id, "Такого дня нет. Попробуйте еще раз.")
        choose_day(message)
        return

    markup = types.ReplyKeyboardMarkup(resize_keyboard=True, row_width=4)
    found_free_time = False

    for time, student in raspisanie[day].items():
        if student is None:
            markup.add(types.KeyboardButton(time))
            found_free_time = True

    if not found_free_time:
        bot.send_message(message.chat.id, "К сожалению, на этот день нет свободных мест.")
        start(message)
        return

    markup.add(types.KeyboardButton("В главное меню"))
    msg = bot.send_message(message.chat.id, f"Свободное время на {day}:", reply_markup=markup)
    bot.register_next_step_handler(msg, step_3_confirm, day)  # Идем на шаг подтверждения

def step_3_confirm(message, day):
    time = message.text
    if time == "В главное меню":
        start(message)
        return

    if day in raspisanie and time in raspisanie[day] and raspisanie[day][time] is None:
        markup = types.ReplyKeyboardMarkup(resize_keyboard=True, row_width=2)
        markup.add(types.KeyboardButton("✅ Подтверждаю"), types.KeyboardButton("❌ Отмена"))
        msg = bot.send_message(message.chat.id, f"Вы хотите записаться на *{day}* в *{time}*?", reply_markup=markup,
                               parse_mode="Markdown")
        bot.register_next_step_handler(msg, step_4_finish, day, time)
    else:
        bot.send_message(message.chat.id, "Ошибка: время уже занято или указано неверно.")
        start(message)


def step_4_finish(message, day, time):
    if message.text == "✅ Подтверждаю":
        user_id_str = get_user_id_string(message)

        if raspisanie[day][time] is None:
            raspisanie[day][time] = user_id_str
            save_data()

            bot.send_message(message.chat.id, f"🎉 Вы успешно записаны на {day} в {time}.")
            bot.send_message(admin_id, f"*Новая запись!*\nКлиент: {user_id_str}\nДень: {day}\nВремя: {time}",
                             parse_mode="Markdown")

            try:
                with open(DATA_FILE, "rb") as f:
                    bot.send_document(admin_id, f, caption="Обновленный файл расписания после записи")
            except Exception as e:
                print(f"Ошибка отправки файла: {e}")
        else:
            bot.send_message(message.chat.id, "Упс! Пока вы подтверждали, это время кто-то занял.")
    else:
        bot.send_message(message.chat.id, "Запись отменена.")

    start(message)

@bot.message_handler(func=lambda message: message.text)
def send_json_file(message):
    global raspisanie
    raspisanie = load_data()
    text = " *Свободное время для записи:*\n\n"

    for day, times in raspisanie.items():
        free_slots = [time for time, status in times.items() if status is None]
        if free_slots:
            text += f"*{day}*:\n`{', '.join(free_slots)}` \n\n"
        else:
            text += f"*{day}*:\n_Нет свободных мест_ \n\n"

    bot.send_message(admin_id, text, parse_mode="Markdown")

if __name__ == '__main__':
    print("Бот запущен...")
    bot.polling(none_stop=True)
