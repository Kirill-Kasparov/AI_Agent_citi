# Телеграм-бот AI-ассистент для менеджеров по продажам

Этот проект представляет собой Телеграм-бота, который помогает менеджерам по продажам генерировать ответы для клиентов

Предпосылки:
---------
Рост ставки рефинансирования и демпинг рыночных цен со стороны маркетплейса привет к непопулярному решению - выполнять больше задач тем же ресурсом.
Оценка большинства трудозатратных процессов приводила к единственному выводу, нужно автоматизировать. Первые попытки автоматизации задач обходились даже дороже человеческого ресурса - 50 руб. на 1 ответ, но пересмотр алгоритмов и оптимизации позволили снизить затраты до 70 копеек (в 70 раз!)

Решение:
---------
Решение было найдено, благодаря двойной обработки сообщения через GPT:
1. GPT определяет категорию, в коготой относится запрос. Ответ в 1-2 слова обходился 5 копеек.
2. Запрос повторно направляется в GPT, но инструкции (промпт), передаются уже из узкой категории, что в 10 раз сэкономило трафик.

Оптимизация:

3. Большинство категорий отрабатывалось GPT 3.5, что снижало необходимость переплачивать.
4. Оптимизация промпта сократила входящий трафик почти вдвое.

Комбинация 4 пунктов дала экономию в 70 раз!

## Основные возможности
- Быстрое получение шаблонов ответов для различных запросов клиентов.
- Управление доступом пользователей по индивидуальным ключам.
- Логирование всех сообщений для анализа работы бота.
- Поддержка оценок ответов с обратной связью для улучшения ПРОМПТ.
- Подсчёт использованных токенов и контроль лимитов.

## Установка и запуск

### Требования
- Python 3.8+
- Библиотеки: `telebot`, `requests`, `pandas`, `tiktoken`

### Установка зависимостей
Установите необходимые зависимости с помощью команды:
```bash
pip install pyTelegramBotAPI pandas requests tiktoken
```

### Настройка
1. Создайте файл `bot_token_cl_assist.txt` и поместите в него токен вашего Телеграм-бота.
2. Создайте файл `gpt_api.txt` и добавьте ваш API-ключ для доступа к OpenAI.
3. Создайте файл `keys.txt` для хранения ключей доступа пользователей (по одному ключу на строку).
4. Создайте файлы с промптами: `gpt_prompt.txt`, `gpt_prompt_delivery.txt`, `gpt_prompt_refund.txt`, `gpt_prompt_docs.txt`, `gpt_prompt_order.txt`, `gpt_prompt_selection.txt` с подходящими шаблонами сообщений для каждой категории.

### Запуск бота
Запустите бота с помощью команды:
```bash
python bot.py
```

## Использование
### Команды для пользователей
- `/start` — регистрация и начало работы.

### Команды для администраторов
- `/add` — сгенерировать новый ключ для пользователя.
- `/admin` — список доступных админ-команд.
- `/limit` — проверить текущий лимит токенов.
- `/users_list` — получить список зарегистрированных пользователей.
- `/logs_plz` — выгрузить логи.

## Структура логов
Логи хранятся в файле `logs.csv` и содержат следующие столбцы:
- `date` — дата запроса.
- `user_id`, `user_name` — идентификаторы пользователя.
- `message_id` — идентификатор сообщения.
- `mess_input`, `mess_output` — запрос и ответ.
- `category` — категория запроса.
- `prompt_tokens`, `user_message_tokens`, `api_response_tokens` — количество токенов.
- `like_or_not` — оценка ответа.

## Оценка и обратная связь
Пользователи могут оценивать ответы с помощью кнопок:
- 👍 — позитивная оценка.
- 👎 — негативная оценка.

Оценки автоматически сохраняются в логах для последующего анализа.

## Контроль лимитов токенов
Бот отслеживает общее количество токенов, чтобы предотвратить превышение лимита. При достижении лимита отправляется уведомление администратору с возможностью его увеличения.

```python
import telebot    # by Kirill Kasparov, 2024
import requests
import json
from telebot.types import InlineKeyboardMarkup, InlineKeyboardButton
import pandas as pd
import random
import string
import os
from datetime import date
import time

# Инициализация Telegram-бота
with open("bot_token_cl_assist.txt", "r") as bot_file:
    api_telegram = bot_file.read().strip()
bot = telebot.TeleBot(api_telegram)

# Инициализация DataFrame для логов
log_columns = [
    "date", "user_id", "user_name", "message_id", "mess_input", "mess_output", "category",
    "prompt_tokens", "user_message_tokens", "api_response_tokens", "like_or_not"
]

try:
    logs = pd.read_csv("logs.csv", sep=';', encoding='windows-1251')
except FileNotFoundError:
    logs = pd.DataFrame(columns=log_columns)
# Функция для обновления логов
def save_logs():
    logs.to_csv("logs.csv", sep=";", index=False, encoding='windows-1251', errors='replace')

# Инициализация пользователей
users_list_df = pd.read_csv('users_list.csv', sep=';', encoding='windows-1251')
admin = ['1294387514', '256234785']


# Инициализация GPT и загрузка ПРОМПТА
gpt_model = 'gpt-3.5-turbo'
with open("gpt_api.txt", "r") as api_file:
    api_gpt = api_file.read().strip()
with open("gpt_prompt.txt", "r", encoding="utf-8") as prompt_file:
    prompt_category = prompt_file.read()
with open("gpt_prompt_delivery.txt", "r", encoding="utf-8") as prompt_file:
    prompt_delivery = prompt_file.read()
with open("gpt_prompt_refund.txt", "r", encoding="utf-8") as prompt_file:
    prompt_refund = prompt_file.read()
with open("gpt_prompt_docs.txt", "r", encoding="utf-8") as prompt_file:
    prompt_docs = prompt_file.read()
with open("gpt_prompt_order.txt", "r", encoding="utf-8") as prompt_file:
    prompt_order = prompt_file.read()
with open("gpt_prompt_selection.txt", "r", encoding="utf-8") as prompt_file:
    prompt_selection = prompt_file.read()
prompt_other = 'Отвечаешь: у меня нет ответа на вопрос'


# Счетчик лимита токенов
token_limit = 100000
token_sum_now = 0

# Кустарный подсчет токенов
def count_tokens(text):
    words = text.split()
    token_count = len(words) * 2
    return token_count

# Функция для отправки запроса к API
def send_to_gpt(user_message, prompt, gpt_model = 'gpt-3.5-turbo'):
    API_URL = "https://api.proxyapi.ru/openai/v1/chat/completions"
    headers = {
        "Content-Type": "application/json",
        "Authorization": f"Bearer {api_gpt}"
    }
    payload = {
        "model": gpt_model,
        "messages": [
            {"role": "system", "content": prompt},  # Системный промпт
            {"role": "user", "content": user_message + ". Use no more than 250 tokens"}     # Сообщение пользователя
        ],
        "temperature": 0.3,
        "max_tokens": 1000
    }

    try:
        response = requests.post(API_URL, headers=headers, data=json.dumps(payload))
        response.raise_for_status()  # Проверка статуса HTTP
        data = response.json()
        return data["choices"][0]["message"]["content"].strip()
    except requests.exceptions.RequestException as e:
        return f"Ошибка при обращении к API: {e}"
    except KeyError:
        return "Ошибка: некорректный ответ от API."


# Обработчик команды /start
@bot.message_handler(commands=['start'])
def handle_start_command(message):
    chat_id = message.chat.id
    user_id = str(message.from_user.id)
    user_name = message.from_user.username

    # Проверка, если пользователь уже зарегистрирован
    if user_id in users_list_df['user_id'].astype(str).tolist():
        bot.send_message(chat_id, f"Привет! Мы уже знакомы, {user_name}.")
    else:
        bot.send_message(chat_id, "Привет! Для доступа к ассистенту, введите индивидуальный ключ:")
        bot.register_next_step_handler(message, handle_key_input, user_id, user_name)

def handle_key_input(message, user_id, user_name):
    chat_id = message.chat.id
    key = message.text.strip()
    file_path = "keys.txt"

    if os.path.exists(file_path):
        with open(file_path, "r") as file:
            keys = file.read().splitlines()

        if key in keys:
            keys.remove(key)
            with open(file_path, "w") as file:
                file.writelines('\n'.join(keys))

            bot.send_message(chat_id, "Ключ активирован. Пожалуйста, введите рабочую почту. (например: assistant@citilink.ru)")
            bot.register_next_step_handler(message, handle_email_input, user_id, user_name, key)
        else:
            bot.send_message(chat_id, "Неверный ключ. Пожалуйста, попробуйте снова.")
            bot.register_next_step_handler(message, handle_key_input, user_id, user_name)
    else:
        bot.send_message(chat_id, "Файл с ключами отсутствует. Свяжитесь с администратором.")


def handle_email_input(message, user_id, user_name, key):
    chat_id = message.chat.id
    user_email = message.text.strip()

    bot.send_message(chat_id, "Введите Ваш город для локализации ответов (например: Москва)")
    bot.register_next_step_handler(message, handle_city_input, user_id, user_name, key, user_email)


def handle_city_input(message, user_id, user_name, key, user_email):
    chat_id = message.chat.id
    user_city = message.text.strip()

    user_data = {
        "user_key": key,
        "user_id": user_id,
        "user_name": user_name,
        "user_email": user_email,
        "user_city": user_city
    }
    df_columns = ["user_key", "user_id", "user_name", "user_email", "user_city"]
    try:
        users_df = pd.read_csv('users_list.csv', sep=';', encoding='windows-1251')
    except FileNotFoundError:
        users_df = pd.DataFrame(columns=df_columns)

    users_df = pd.concat([users_df, pd.DataFrame([user_data])], ignore_index=True)
    users_df.to_csv('users_list.csv', sep=';', encoding='windows-1251', index=False)

    global users_list_df
    users_list_df = pd.read_csv('users_list.csv', sep=';', encoding='windows-1251')

    bot.send_message(chat_id, "Регистрация завершена! Добро пожаловать в систему.")


# Выдает админу ключ для нового пользователя
@bot.message_handler(commands=['add'])
def handle_add_command(message):
    if str(message.from_user.id) in admin:
        # Генерация случайного ключа из 11 букв и цифр
        key = ''.join(random.choices(string.ascii_letters + string.digits, k=11))
        # Делаем "моноширенный" формат для быстрого копирования из чата
        escaped_key = key.replace('_', '\\_').replace('*', '\\*').replace('[', '\\[').replace('`', '\\`')

        # Добавление ключа в файл
        file_path = "keys.txt"
        mode = 'a' if os.path.exists(file_path) else 'w'  # Проверка существования файла
        with open(file_path, mode) as file:
            file.write(key + '\n')

        # Отправка ключа пользователю
        bot.reply_to(message, f"Ваш ключ: `{escaped_key}`", parse_mode="MarkdownV2")

#Админские команды
@bot.message_handler(commands=['admin'])
def admin_commands(message):
    if str(message.from_user.id) in admin:
        bot.send_message(message.from_user.id, '/add - выдать ключ для нового пользователя')
        bot.send_message(message.from_user.id, '/limit - проверить лимит токенов')
        bot.send_message(message.from_user.id, '/users_list - список пользователей')
        bot.send_message(message.from_user.id, '/logs_plz - логи запросов')


# Выдает админу ключ для нового пользователя
@bot.message_handler(commands=['limit'])
def handle_add_command(message):
    if str(message.from_user.id) in admin:
        global token_limit
        if token_sum_now < token_limit:
            bot.send_message(message.chat.id, f"Израсходовано токенов: `{token_sum_now}` из '{token_limit}'")
        else:
            bot.send_message(message.chat.id, f"Израсходовано токенов: `{token_sum_now}` из '{token_limit}'")
            token_limit += 50000
            bot.send_message(message.chat.id, f"Увеличил лимит до: '{token_limit}'")

# Присылает логи
@bot.message_handler(commands=['logs_plz'])
def admin_logs(message):
    if str(message.from_user.id) in admin:
        bot.send_message(message.from_user.id, 'После прочтения сжечь:')
        f = open('logs.csv', "rb")
        bot.send_document(message.chat.id, f)

# Присылает список пользователей
@bot.message_handler(commands=['users_list'])
def admin_users_list(message):
    if str(message.from_user.id) in admin:
        bot.send_message(message.from_user.id, 'Список строевых:')
        f = open('users_list.csv', "rb")
        bot.send_document(message.chat.id, f)

# Обработчик текстовых сообщений
@bot.message_handler(func=lambda message: True)
def handle_message(message):
    user_input = message.text
    user_id = str(message.from_user.id)
    global token_sum_now
    # Проверка наличия пользователя в списке и лимитов токенов
    if user_id in users_list_df['user_id'].astype(str).tolist() and token_sum_now < token_limit:
        # Подсчет токенов
        prompt_tokens = count_tokens(prompt_category)
        user_message_tokens = count_tokens(user_input)
        api_response_tokens = 0

        # Отправка сообщения в API
        api_response = send_to_gpt(user_input, prompt_category)
        api_response_tokens += count_tokens(api_response)
        category = api_response

        # Определяем соответствие ключевых слов промптам и моделям
        prompt_mapping = {
            'доставк': (prompt_delivery, 'gpt-4o'),
            'гарант': (prompt_refund, 'gpt-3.5-turbo'),
            'документ': (prompt_docs, 'gpt-3.5-turbo'),
            'заказ': (prompt_order, 'gpt-3.5-turbo'),
            'подбор': (prompt_selection, 'gpt-4o')
        }

        # Обработка ответа
        if 'Ошибка при обращении к API' in api_response:
            api_response = "... какая-то ошибка"
        elif 'другое' in api_response.lower():
            api_response = "Извините, у меня нет ответа на ваш вопрос."
        else:
            for key, (prompt, model) in prompt_mapping.items():
                if key in api_response.lower():
                    api_response = send_to_gpt(user_input, prompt, gpt_model=model)
                    prompt_tokens += count_tokens(prompt)
                    user_message_tokens = count_tokens(user_input)
                    api_response_tokens += count_tokens(api_response)
                    break
            else:
                # Если ни одно ключевое слово не найдено
                api_response = send_to_gpt(user_input, prompt_other)
                prompt_tokens += count_tokens(prompt_other)
                user_message_tokens = count_tokens(user_input)
                api_response_tokens += count_tokens(api_response)

        # Сохранение логов
        log_entry = {
            "date": date.today().strftime("%Y-%m-%d"),
            "user_id": int(message.from_user.id),
            "user_name": str(message.from_user.username),
            "message_id": int(message.message_id),
            "mess_input": str(user_input),
            "mess_output": str(api_response),
            "category": str(category),
            "prompt_tokens": int(prompt_tokens),
            "user_message_tokens": int(user_message_tokens),
            "api_response_tokens": int(api_response_tokens),
            "like_or_not": None
        }

        global logs
        logs = pd.concat([logs, pd.DataFrame([log_entry])], ignore_index=True)
        save_logs()

        try:
            # Отправка ответа пользователю
            bot.send_message(message.chat.id, api_response)
            # Отправка клавиатуры с оценкой
            markup = InlineKeyboardMarkup()
            like_button = InlineKeyboardButton("👍", callback_data=str(message.message_id) + "_1_like")
            dislike_button = InlineKeyboardButton("👎", callback_data=str(message.message_id) + "_0_dislike")
            markup.add(like_button, dislike_button)
            bot.send_message(message.chat.id, "Оцените ответ:", reply_markup=markup)
        except Exception as e:
            log_entry = {
                "date": date.today().strftime("%Y-%m-%d"),
                "user_id": int(message.from_user.id),
                "user_name": str(message.from_user.username),
                "message_id": int(message.message_id),
                "mess_input": str(user_input),
                "mess_output": str(e),
                "category": str("Ошибка"),
                "prompt_tokens": int(prompt_tokens),
                "user_message_tokens": int(user_message_tokens),
                "api_response_tokens": int(api_response_tokens),
                "like_or_not": None
            }
            logs = pd.concat([logs, pd.DataFrame([log_entry])], ignore_index=True)
            save_logs()
            bot.send_message(message.chat.id, "Найдена ошибка, отправили админу. Повторите запрос еще раз.")



        # Подсчет стоимости токенов
        if gpt_model == 'gpt-3.5-turbo':
            price_out = 0.144 * 1.7
            price_in = 0.432 * 1.7
        elif gpt_model == 'gpt-4o':
            price_out = 0.72 * 1.1
            price_in = 2.88 * 1.1
        else:
            price_out = 0.72
            price_in = 2.88
        prompt_tokens_cost = round(prompt_tokens * price_out / 1000, 4)
        user_message_tokens_cost = round(user_message_tokens * price_out / 1000, 4)
        api_response_tokens_cost = round(api_response_tokens * price_in / 1000, 4)
        total_cost = prompt_tokens_cost + user_message_tokens_cost + api_response_tokens_cost

        token_sum_now += prompt_tokens + user_message_tokens + api_response_tokens

        # Логирование токенов
        # print(f"Токены для prompt: {prompt_tokens} price: ", prompt_tokens_cost)
        # print(f"Токены для user_message: {user_message_tokens} price: ", user_message_tokens_cost)
        # print(f"Токены для response: {api_response_tokens} price: ", api_response_tokens_cost)
        # print("total price: ", total_cost)
    elif user_id in users_list_df['user_id'].astype(str).tolist() and token_sum_now > token_limit:
        bot.send_message(message.chat.id, "Превышен лимит запросов на сегодня. Для подтверждения доп. лимитов, обратитесь к @KirillKasparov")




# Обработчик коллбеков для оценки
@bot.callback_query_handler(func=lambda call: "like" in call.data or "dislike" in call.data)
def handle_feedback(call):
    global logs  # Глобальная переменная для логов

    # Разбор callback_data
    data_parts = call.data.split("_")
    if len(data_parts) >= 2:
        message_id = int(data_parts[0])  # Идентификатор сообщения
        like_or_not = int(data_parts[1])  # Оценка
        if message_id in logs["message_id"].values:
            logs.loc[logs["message_id"] == message_id, "like_or_not"] = like_or_not
            save_logs()

        # Ответ пользователю
        if like_or_not == 1:
            bot.answer_callback_query(call.id, "Спасибо за лайк! 👍")
        else:
            bot.answer_callback_query(call.id, "Нужно доработать! 👎")

        # Удаление клавиатуры
        bot.edit_message_reply_markup(call.message.chat.id, call.message.message_id, reply_markup=None)
    else:
        bot.answer_callback_query(call.id, "Ошибка: сообщение не найдено в логах.")



try:  # перезапуск при достижении лимита и дисконекте
    if __name__ == "__main__":
        print("Бот запущен...")
        bot.infinity_polling()
except:  # перезапуск при дисконекте
    time.sleep(10)
    os.startfile("AI_agent_citi.exe")
```
