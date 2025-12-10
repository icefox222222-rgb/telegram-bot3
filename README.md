import telebot
import requests
from datetime import datetime

TOKEN = "8437969121:AAHmQofGYzIJlnk6ZdAwAElpuYz9kkH9oyw"
WEATHER_API_KEY = "277d8442cd2cdd6ad76dd4d094082157"

bot = telebot.TeleBot(TOKEN)

# Состояние диалога
user_state = {}


# ===================== START ======================
@bot.message_handler(commands=['start'])
def start(message):
    bot.send_message(
        message.chat.id,
        f"Привет, {message.from_user.first_name}! ☀\n"
        f"Я могу показать погоду утром, днём и вечером.\n"
        f"Напиши /forecast"
    )


# ===================== FORECAST ======================
@bot.message_handler(commands=['forecast'])
def ask_city(message):
    bot.send_message(message.chat.id, "В каком городе ты живёшь?")
    user_state[message.chat.id] = "ask_city"


@bot.message_handler(content_types=['text'])
def handle_city(message):
    chat_id = message.chat.id

    # шаг: ждём город
    if user_state.get(chat_id) == "ask_city":
        city = message.text.strip()
        forecast = get_day_forecast(city)

        bot.send_message(chat_id, forecast, parse_mode="Markdown")
        user_state.pop(chat_id, None)
        return

    # обычные ответы
    if message.text.lower() == "привет":
        bot.send_message(chat_id, f"Привет, {message.from_user.first_name}!")
    elif message.text.lower() == "id":
        bot.send_message(chat_id, f"Твой ID: {message.from_user.id}")


# ===================== ЛОГИКА ПОГОДЫ ======================

def get_day_forecast(city):
    """Погода утром, днём и вечером + советы по одежде"""

    url = f"http://api.openweathermap.org/data/2.5/forecast?q={city}&appid={WEATHER_API_KEY}&units=metric&lang=ru"

    try:
        res = requests.get(url).json()
        print("Ответ:", res)

        # Ошибки
        if res.get("cod") == "401":
            return "❌ Неверный API ключ."
        if res.get("cod") == "404":
            return "❌ Город не найден. Напиши на английском, например Tashkent."
        if res.get("cod") != "200":
            return f"❌ Ошибка: {res.get('message')}"

        morning = day = evening = None

        for entry in res["list"]:
            time = entry["dt_txt"]

            if "09:00:00" in time:
                morning = entry
            if "15:00:00" in time:
                day = entry
            if "21:00:00" in time:
                evening = entry

        if not (morning and day and evening):
            return "⚠ У этого города нет точного прогноза по часам."

        # БЕЗ отступа между погодой и рекомендацией
        # ОТСТУП только внизу блока
        def format_block(name, data):
            temp = data["main"]["temp"]
            feels = data["main"]["feels_like"]
            condition = data["weather"][0]["description"].capitalize()

            return (
                f"*{name}:*\n"
                f"🌡 Температура: {temp}°C\n"
                f"🤔 Ощущается как: {feels}°C\n"
                f"☁ Погодa: {condition}\n"
                f"👕 {clothes_recommendation(temp)}\n\n"   # ← Отступ ТОЛЬКО здесь
            )

        result = (
            f"*Погода в {city.title()} на сегодня*\n\n"
            f"{format_block('Утром', morning)}"
            f"{format_block('Днём', day)}"
            f"{format_block('Вечером', evening)}"
        )

        return result

    except Exception as e:
        return f"❌ Ошибка: {e}"


# ===================== СОВЕТЫ ПО ОДЕЖДЕ ======================

def clothes_recommendation(temp):
    """Рекомендации по одежде в зависимости от температуры"""
    if temp < -10:
        return "Очень холодно! Тёплую куртку, перчатки, шапку."
    elif -10 <= temp < 0:
        return "Холодно. Куртку, шарф, шапку."
    elif 0 <= temp < 10:
        return "Прохладно. Лёгкая куртка или толстовка."
    elif 10 <= temp < 18:
        return "Немного прохладно. Кофта/толстовка."
    elif 18 <= temp < 25:
        return "Комфортно. Можно одеться легко."
    elif 25 <= temp < 32:
        return "Жарко. Футболка и легкая одежда."
    else:
        return "Очень жарко! Пить воду, панама, лёгкая одежда."


# ===================== RUN ======================
bot.polling(none_stop=True)
