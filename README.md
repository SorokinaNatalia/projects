from telebot import types, TeleBot
import schedule
import time
import threading
import os
from datetime import datetime, timedelta
import re
import string
import requests
import gspread
from gspread import Client, Spreadsheet, Worksheet

SPREADSHEET_URL = "https://docs.google.com/spreadsheets/d/.../"

bot = TeleBot("your TOKEN here")
user_chat_ids = {}
last_weight_entry_time = {}

sh = None  # Определение глобальной переменной sh

# Создаем встроенную клавиатуру
keyboard = types.ReplyKeyboardMarkup(row_width=1, resize_keyboard=True)

# Добавляем кнопки в клавиатуру
button_weight = types.KeyboardButton("Вес")
button_report = types.KeyboardButton("Отчет по питанию")
button_measurements = types.KeyboardButton("Замеры")

# Добавляем кнопки на клавиатуру
keyboard.add(button_weight, button_report, button_measurements)

@bot.message_handler(commands=['start'])
def send_welcome(message):
    if message:
        chat_id = message.chat.id
        if chat_id not in user_chat_ids:
            bot.reply_to(message, "Я твой помощник!\n\n"
                                        "Но для начала хочу узнать твои фамилию и имя, напиши их пожалуйста.", reply_markup=keyboard)
            user_chat_ids[chat_id] = {'name': None, 'surname': None}
        else:
            # Если пользователь уже отправил свои данные, сообщаем ему об этом
            bot.reply_to(message, "С возвращением!")

        # Устанавливаем обработчик для следующего сообщения с именем и фамилией
        bot.register_next_step_handler(message, get_name)

def get_name(message):
    chat_id = message.chat.id
    if chat_id in user_chat_ids:
        # Получаем имя и фамилию пользователя из сообщения
        name_surname = message.text.split()
        if len(name_surname) >= 2:
            user_surname = name_surname[0]
            user_name = ' '.join(name_surname[1:])

            # Сохраняем имя и фамилию пользователя
            user_chat_ids[chat_id] = {'name': user_name, 'surname': user_surname}

            # Отправляем сообщение об успешной регистрации
            bot.send_message(chat_id, f"Спасибо, {user_surname} {user_name}! Теперь ты можешь далее пользоваться ботом.")
        else:
            # Если сообщение не содержит как минимум имя и фамилию, запрашиваем их снова
            bot.send_message(chat_id, "Пожалуйста, введите и фамилию, и имя:")
            bot.register_next_step_handler(message, get_name)


@bot.message_handler(func=lambda message: message.text == "Отчет по питанию")
def request_diet_report(message):
    chat_id = message.chat.id
    keyboard = types.ReplyKeyboardMarkup(row_width=1, resize_keyboard=True)
    button_report = types.KeyboardButton("Отчет по питанию")
    keyboard.add(button_report)
    bot.send_message(chat_id, "Пожалуйста, отправьте мне отчет по питанию в формате PDF.")


@bot.message_handler(content_types=['document'])
def handle_document(message):
    chat_id = message.chat.id
    file_id = message.document.file_id
    file_info = bot.get_file(file_id)
    downloaded_file = bot.download_file(file_info.file_path)
    file_extension = os.path.splitext(file_info.file_path)[-1].lower()

    if file_extension == '.pdf':
        # Получаем имя и фамилию пользователя
        user_id = message.from_user.id
        user_name = user_chat_ids.get(user_id, {}).get('name', '')
        user_surname = user_chat_ids.get(user_id, {}).get('surname', '')

        # Проверяем, что имя и фамилия пользователя не пустые
        if user_name and user_surname:
            # Генерируем содержимое отчета с именем и фамилией пользователя
            report_content = f"Отчет по питанию от пользователя: {user_name} {user_surname}\n\n"

            # Сохраняем полученный файл
            file_path = f"{chat_id}_diet_report.pdf"
            with open(file_path, 'wb') as new_file:
                new_file.write(downloaded_file)

            # Отправляем сообщение с подтверждением
            bot.send_message(chat_id, "Спасибо! Ваш отчет по питанию получен и сохранен.")

            # Отправляем файл и сообщение в нужный чат
            bot.send_document(-1002138453651, open(file_path, "rb"), caption=report_content)
    else:
        # Если файл не в формате PDF, отправляем сообщение с просьбой отправить отчет в правильном формате
        bot.send_message(chat_id, "Неверный формат файла! Пожалуйста, отправьте отчет по питанию в формате PDF.")

def find_participant_row(ws, user_name, user_surname):
    values = ws.get_all_values()
    for i, row in enumerate(values):
        if row[1] == f"{user_surname} {user_name}":
            return i + 1  # Возвращаем номер строки (нумерация начинается с 1)
    return None  # Возвращаем None, если участник не найден

def add_new_participant(ws, user_name, user_surname, data):
    # Получаем текущее количество строк в таблице
    num_rows = len(ws.get_all_values())

    # Если таблица пуста (за исключением заголовков), начинаем с первой строки
    if num_rows == 0:
        last_row_number = 0  # Начинаем нумерацию с 0, первая строка - заголовки, вторая строка - первая запись
    else:
        # Находим последнюю заполненную строку во втором столбце
        for row in reversed(ws.col_values(2)):
            if row:
                last_row_number = len(ws.col_values(2)) + 1
                break

    # Обновляем ячейки в новой строке таблицы
    ws.update_cell(last_row_number, 2, f"{user_surname} {user_name}")  # Имя участника
    ws.update_cell(last_row_number, 3, data)  # Данные (вес или замеры)

# Обработчик для кнопки "Замеры"
@bot.message_handler(func=lambda message: message.text == "Замеры")
def request_measurements(message):
    user_id = message.from_user.id
    current_time = datetime.now()
    bot.send_message(message.chat.id, "Пожалуйста, введите замеры в формате ОГ/ОТ/ОБ")
    bot.register_next_step_handler(message, handle_measurements)

def handle_measurements(message):
    user_id = message.from_user.id
    user_name = user_chat_ids.get(user_id, {}).get('name', '')
    user_surname = user_chat_ids.get(user_id, {}).get('surname', '')

    # Проверяем, что имя и фамилия пользователя не пустые
    if user_name and user_surname:
        ws = sh.get_worksheet(1)  # Получаем второй лист таблицы
        measurements = message.text

        # Проверяем, есть ли уже запись о пользователе
        participant_row = find_participant_row(ws, user_name, user_surname)
        if participant_row is not None:
            # Если пользователь уже есть в таблице, записываем замеры в следующий столбец
            values = ws.row_values(participant_row)
            next_column = len(values) + 1
            ws.update_cell(participant_row, next_column, measurements)
            bot.send_message(message.chat.id, f"Ваши замеры успешно сохранены.")
        else:
            # Если записи о пользователе нет, добавляем новую строку с данными об участнике и замерами
            add_new_participant(ws, user_name, user_surname, measurements)
            bot.send_message(message.chat.id, f"Ваши замеры успешно сохранены.")



# Обработчик для кнопки "Вес"
@bot.message_handler(func=lambda message: message.text == "Вес")
def request_weight(message):
    user_id = message.from_user.id
    current_time = datetime.now()

    if user_id in last_weight_entry_time:
        # Если есть запись о времени последнего ввода веса для данного пользователя
        time_difference = current_time - last_weight_entry_time[user_id]
        if time_difference < timedelta(hours=17):
            # Если прошло менее 17 часов с момента последнего ввода
            time_to_wait = timedelta(hours=17) - time_difference
            hours, remainder = divmod(time_to_wait.seconds, 3600)
            minutes, _ = divmod(remainder, 60)
            bot.send_message(message.chat.id, f"Вы уже внесли вес сегодня. Пожалуйста, попробуйте через {hours} часов {minutes} минут.")
            return

    # Если прошло более 17 часов с момента последнего ввода или это первый ввод сегодня
    bot.send_message(message.chat.id, "Пожалуйста, введите вес:")
    last_weight_entry_time[user_id] = current_time  # Обновляем время последнего ввода веса для данного пользователя

def save_weight(message, sh):
    chat_id = message.chat.id
    weight = message.text
    user_id = message.from_user.id
    user_name = user_chat_ids.get(user_id, {}).get('name', '')
    user_surname = user_chat_ids.get(user_id, {}).get('surname', '')


    # Проверяем, что имя и фамилия пользователя не пустые
    if user_name and user_surname:
        ws = sh.get_worksheet(0)

        # Проверяем, есть ли уже запись о пользователе
        participant_row = find_participant_row(ws, user_name, user_surname)
        if participant_row is not None:
            # Если пользователь уже есть в таблице, записываем вес в следующий столбец
            values = ws.row_values(participant_row)
            next_column = len(values) + 1
            ws.update_cell(participant_row, next_column, weight)
            bot.send_message(chat_id, f"Ваш вес успешно сохранен.")
        else:
            # Если записи о пользователе нет, добавляем новую строку с данными об участнике и весом
            add_new_participant(ws, user_name, user_surname, weight)
            bot.send_message(chat_id, f"Ваш вес успешно сохранен.")


# Установка начальной даты для начала отправки напоминаний
start_date = datetime.now()

# Функция для обработки напоминания о замере веса
def remind_weight(bot, user_chat_ids):
    # Получаем список всех чатов из user_chat_ids
    chat_ids = list(user_chat_ids.keys())
    for chat_id in chat_ids:
        bot.send_message(chat_id, "Пора взвеситься! Пожалуйста, нажмите на кнопку Вес и введите свои данные:")

# Функция для обработки напоминания об отправке отчёта о съеденном рационе
def remind_report(bot, user_chat_ids):
    # Получаем список всех чатов из user_chat_ids
    chat_ids = list(user_chat_ids.keys())
    for chat_id in chat_ids:
        bot.send_message(chat_id, "Не забудь отправить отчёт когда съел весь суточный рацион")

# Планирование напоминания на определенное время ежедневно
schedule.every().day.at("07:00").do(remind_weight, bot, user_chat_ids)
schedule.every().day.at("21:00").do(remind_report, bot, user_chat_ids)


# Функция для выполнения планировщика задач
def run_schedule():
    while True:
        # Проверяем, прошло ли уже три месяца с начала отправки напоминаний
        if datetime.now() - start_date >= timedelta(days=90):
            break
        schedule.run_pending()
        time.sleep(60)  # Проверка каждую минуту

# Запуск планировщика задач в отдельном потоке
schedule_thread = threading.Thread(target=run_schedule)
schedule_thread.start()

def main():
    global sh  # Объявление переменной sh как глобальной
    gc: Client = gspread.service_account("./service_account.json")
    sh = gc.open_by_url(SPREADSHEET_URL)
    ws = sh.get_worksheet(0)
    ws = sh.get_worksheet(1)



# Обработчик для сохранения веса
@bot.message_handler(func=lambda message: message.text.isdigit())
def handle_weight(message):
    save_weight(message, sh)  # Передаем sh в функцию save_weight



if __name__ == '__main__':
    main()

# Запускаем бота
bot.polling()
