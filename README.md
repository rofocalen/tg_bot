from tenacity import retry, wait_fixed, stop_after_attempt, retry_if_exception_type
import requests
import telebot
import psycopg2
import random
from telebot import types
import certifi
import ssl
import urllib3
import requests
from requests.adapters import HTTPAdapter
from requests.packages.urllib3.util.retry import Retry

session = requests.Session()
retry = Retry(connect=5, backoff_factor=1)
adapter = HTTPAdapter(max_retries=retry)
session.mount('https://', adapter)
session.timeout = (10, 60)  

from telebot import apihelper
apihelper._get_req_session = lambda: session

ssl_context = ssl.create_default_context(cafile=certifi.where())
http = urllib3.PoolManager(ssl_context=ssl_context)

TOKEN = '8434116009:AAFMgotJ3Vl0T4kOEWV4IdZuPUk5yOGJTOE'

bot = telebot.TeleBot(TOKEN)

conn = psycopg2.connect(
    dbname='englishcard',
    user='postgres',
    host='localhost'
)
cursor = conn.cursor()


@bot.message_handler(commands=['start'])
def start_message(message):
    bot.send_message(message.chat.id, 'Добро пожаловать в EnglishCard bot!')


@bot.message_handler(commands=['quiz'])
def quiz_handler(message):
    user_id = message.from_user.id
    word_data = get_random_word(user_id)

def get_random_word():
    cursor.execute("SELECT id, english_word, translation FROM words ORDER BY RANDOM() LIMIT 1")
    return cursor.fetchone()


def get_quiz_options(correct_word_id):
    cursor.execute("SELECT english_word FROM words WHERE id != %s ORDER BY RANDOM() LIMIT 3", (correct_word_id,))
    wrong_words = [row[0] for row in cursor.fetchall()]
    cursor.execute("SELECT english_word FROM words WHERE id = %s", (correct_word_id,))
    correct_word = cursor.fetchone()[0]

    options = wrong_words + [correct_word]
    random.shuffle(options)
    return options


@bot.message_handler(commands=['quiz'])
def send_quiz(message):
    result = get_random_word()
    if not result:
        bot.send_message(message.chat.id, "No words available for quiz.")
        return
    word_id, correct_word, translation = result
    options = get_quiz_options(word_id)

    markup = types.InlineKeyboardMarkup()
    for option in options:
        callback_data = f"answer:{word_id}:{option}"
        markup.add(types.InlineKeyboardButton(option, callback_data=callback_data))

    bot.send_message(message.chat.id, f"What is the English word for \"{translation}\"?", reply_markup=markup)

@bot.callback_query_handler(func=lambda call: call.data.startswith('answer'))
def callback_answer(call):
    _, word_id, chosen_word = call.data.split(':')
    cursor.execute("SELECT english_word FROM words WHERE id = %s", (word_id,))
    correct_word = cursor.fetchone()[0]

    if chosen_word == correct_word:
        bot.answer_callback_query(call.id, "Correct!")
    else:
        bot.answer_callback_query(call.id, "Wrong, try again!")


@bot.message_handler(commands=['new'])
def add_new_word(message):
    msg = bot.send_message(message.chat.id, "Send me the English word:")
    bot.register_next_step_handler(msg, process_english_word)

def process_english_word(message):
    english_word = message.text
    msg = bot.send_message(message.chat.id, "Send me the translation:")
    bot.register_next_step_handler(msg, process_translation, english_word)

def process_translation(message, english_word):
    translation = message.text

    cursor.execute("SELECT id FROM words WHERE english_word = %s", (english_word,))
    result = cursor.fetchone()
    if result:
        word_id = result[0]
    else:
        cursor.execute("INSERT INTO words (english_word, translation) VALUES (%s, %s) RETURNING id", (english_word, translation))
        word_id = cursor.fetchone()[0]

    telegram_id = message.from_user.id
    cursor.execute("SELECT id FROM users WHERE telegram_id = %s", (telegram_id,))
    user_id = cursor.fetchone()[0]

    bot.send_message(message.chat.id, f"Word '{english_word}' with translation '{translation}' added.")

    cursor.execute("SELECT COUNT(*) FROM user_words WHERE user_id = %s", (user_id,))
    count = cursor.fetchone()[0]
    bot.send_message(message.chat.id, f"You are learning {count} words now.")


@bot.message_handler(commands=['delete'])
def delete_word(message):
    telegram_id = message.from_user.id
    cursor.execute("SELECT id FROM users WHERE telegram_id = %s", (telegram_id,))
    user_id = cursor.fetchone()
    if not user_id:
        bot.send_message(message.chat.id, "You are not registered. Use /start first.")
        return
    user_id = user_id[0]
    

@bot.callback_query_handler(func=lambda call: call.data.startswith('delete'))
def callback_delete(call):
    word_id = call.data.split(':')[1]
    telegram_id = call.from_user.id
    cursor.execute("SELECT id FROM users WHERE telegram_id = %s", (telegram_id,))
    user_id = cursor.fetchone()[0]


@retry(wait=wait_fixed(5), stop=stop_after_attempt(5),
       retry=retry_if_exception_type(requests.exceptions.RequestException))
  

if __name__ == '__main__':
    print("Bot started")
    bot.polling(none_stop=True, timeout=60)
