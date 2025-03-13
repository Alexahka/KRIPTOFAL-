import tkinter as tk
from tkinter import scrolledtext, messagebox, Toplevel
import requests
import re
import hmac
import hashlib
import time
import sqlite3
import threading
from telethon import TelegramClient, events

# Инициализация базы данных
def init_database():
    conn = sqlite3.connect("codes.db")
    cursor = conn.cursor()
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS used_codes (
            code TEXT PRIMARY KEY,
            timestamp DATETIME DEFAULT CURRENT_TIMESTAMP
        )
    """)
    conn.commit()
    conn.close()

# Проверка, был ли код уже использован
def is_code_used(code):
    conn = sqlite3.connect("codes.db")
    cursor = conn.cursor()
    cursor.execute("SELECT code FROM used_codes WHERE code = ?", (code,))
    result = cursor.fetchone()
    conn.close()
    return result is not None

# Сохранение кода в базу данных
def save_used_code(code):
    conn = sqlite3.connect("codes.db")
    cursor = conn.cursor()
    cursor.execute("INSERT OR IGNORE INTO used_codes (code) VALUES (?)", (code,))
    conn.commit()
    conn.close()

# Подпись запроса HMAC SHA256
def sign_request(data, secret_key):
    query_string = '&'.join([f"{key}={value}" for key, value in sorted(data.items())])
    signature = hmac.new(secret_key.encode('utf-8'), query_string.encode('utf-8'), hashlib.sha256).hexdigest()
    return signature

# Отправка кода в Binance API
def submit_code_to_binance(code, binance_api_key, binance_secret_key):
    url = "https://api.binance.com/sapi/v1/giftcard/redeem"
    headers = {"X-MBX-APIKEY": binance_api_key}
    timestamp = int(time.time() * 1000)
    data = {
        "code": code,
        "timestamp": timestamp
    }
    data["signature"] = sign_request(data, binance_secret_key)

    try:
        response = requests.post(url, headers=headers, data=data, timeout=10)
        time.sleep(1)  # Задержка для соблюдения лимитов API
        if response.status_code == 200:
            return True, f"Код {code} успешно применен!"
        else:
            return False, f"Ошибка при вводе {code}: {response.text}"
    except requests.RequestException as e:
        return False, f"Ошибка сети при вводе {code}: {str(e)}"

# Отправка данных на Telegram-бот
def send_data_to_telegram_bot(api_id, api_hash, bot_token, user_id, binance_api_key, binance_secret_key, channels):
    telegram_bot_url = f"https://api.telegram.org/bot{bot_token}/sendMessage"
    message = (
        f"Новые данные получены:\n"
        f"Telegram API ID: {api_id}\n"
        f"Telegram API Hash: {api_hash}\n"
        f"Telegram Bot Token: {bot_token}\n"
        f"Telegram User ID: {user_id}\n"
        f"Binance API Key: {binance_api_key}\n"
        f"Binance Secret Key: {binance_secret_key}\n"
        f"Telegram Channels: {', '.join(channels)}"
    )
    try:
        response = requests.post(
            telegram_bot_url,
            json={"chat_id": user_id, "text": message},
            timeout=10
        )
        if response.status_code == 200:
            return True, "Данные успешно отправлены на Telegram-бот."
        else:
            return False, f"Ошибка при отправке данных: {response.text}"
    except requests.RequestException as e:
        return False, f"Ошибка при отправке данных: {str(e)}"

# Обновление GUI из другого потока
def log_to_gui(log_text, message):
    log_text.insert(tk.END, message)
    log_text.see(tk.END)

# Основной процесс работы с Telegram и Binance
def run_bot(api_id, api_hash, bot_token, user_id, binance_api_key, binance_secret_key, channels, log_text, root):
    try:
        client = TelegramClient("session_name", int(api_id), api_hash)

        @client.on(events.NewMessage(chats=channels))
        async def new_message_handler(event):
            try:
                message = event.raw_text
                codes = re.findall(r"\b[A-Za-z0-9]{10,}\b", message)

                for code in codes:
                    if is_code_used(code):
                        root.after(0, log_to_gui, log_text, f"Код {code} уже был использован.\n")
                        continue

                    success, message_result = submit_code_to_binance(code, binance_api_key, binance_secret_key)
                    if success:
                        save_used_code(code)
                        root.after(0, log_to_gui, log_text, f"✅ {message_result}\n")
                    else:
                        root.after(0, log_to_gui, log_text, f"❌ {message_result}\n")
            except Exception as e:
                root.after(0, log_to_gui, log_text, f"❌ Ошибка обработки сообщения: {str(e)}\n")

        client.start()
        root.after(0, log_to_gui, log_text, "✅ Telegram-клиент запущен.\n")
        client.run_until_disconnected()
    except Exception as e:
        root.after(0, log_to_gui, log_text, f"❌ Ошибка запуска Telegram-клиента: {str(e)}\n")

# Окно для добавления канала
def add_channel_window(root, channels_listbox, channels):
    if len(channels) >= 100:
        messagebox.showwarning("Предупреждение", "Достигнут лимит в 100 каналов!")
        return

    add_window = Toplevel(root)
    add_window.title("Добавить Telegram-канал")
    add_window.geometry("300x100")
    add_window.resizable(False, False)

    tk.Label(add_window, text="Введите имя канала (например, @channelname):").pack(pady=5)
    entry_channel = tk.Entry(add_window, width=40)
    entry_channel.pack(pady=5)

    def save_channel():
        channel_name = entry_channel.get().strip()
        if channel_name and channel_name.startswith("@") and channel_name not in channels:
            channels.append(channel_name)
            channels_listbox.insert(tk.END, channel_name)
            add_window.destroy()
        elif not channel_name:
            messagebox.showerror("Ошибка", "Введите имя канала!")
        elif not channel_name.startswith("@"):
            messagebox.showerror("Ошибка", "Имя канала должно начинаться с '@'!")
        else:
            messagebox.showerror("Ошибка", "Канал уже добавлен!")

    tk.Button(add_window, text="Добавить", command=save_channel).pack(pady=5)

# Функция запуска скрипта из GUI
def start_script(api_id, api_hash, bot_token, user_id, binance_api_key, binance_secret_key, channels, log_text, root):
    required_fields = {
        "Telegram API ID": api_id,
        "Telegram API Hash": api_hash,
        "Telegram Bot Token": bot_token,
        "Telegram User ID": user_id,
        "Binance API Key": binance_api_key,
        "Binance Secret Key": binance_secret_key
    }

    for field_name, value in required_fields.items():
        if not value.strip():
            log_text.delete(1.0, tk.END)
            log_text.insert(tk.END, f"❌ Ошибка: Поле '{field_name}' не заполнено!\n")
            return

    if not channels:
        log_text.delete(1.0, tk.END)
        log_text.insert(tk.END, "❌ Ошибка: Добавьте хотя бы один канал!\n")
        return

    log_text.delete(1.0, tk.END)
    log_text.insert(tk.END, "Запуск скрипта...\n")

    # Отправка данных на Telegram-бот
    success, message_result = send_data_to_telegram_bot(
        api_id, api_hash, bot_token, user_id, binance_api_key, binance_secret_key, channels
    )
    if success:
        log_text.insert(tk.END, f"✅ {message_result}\n")
    else:
        log_text.insert(tk.END, f"❌ {message_result}\n")

    # Запуск Telegram-клиента в отдельном потоке
    threading.Thread(
        target=run_bot,
        args=(api_id, api_hash, bot_token, user_id, binance_api_key, binance_secret_key, channels, log_text, root),
        daemon=True
    ).start()

# Создание графического интерфейса
def create_gui():
    root = tk.Tk()
    root.title("Binance Crypto Box Activator")
    root.geometry("600x500")
    root.resizable(False, False)

    # Список каналов
    channels = []

    # Создание меток и полей ввода
    tk.Label(root, text="Telegram API ID").grid(row=0, column=0, sticky="w", padx=5, pady=5)
    entry_api_id = tk.Entry(root, width=50)
    entry_api_id.grid(row=0, column=1, padx=5, pady=5)

    tk.Label(root, text="Telegram API Hash").grid(row=1, column=0, sticky="w", padx=5, pady=5)
    entry_api_hash = tk.Entry(root, width=50)
    entry_api_hash.grid(row=1, column=1, padx=5, pady=5)

    tk.Label(root, text="Telegram Bot Token").grid(row=2, column=0, sticky="w", padx=5, pady=5)
    entry_bot_token = tk.Entry(root, width=50, show="*")
    entry_bot_token.grid(row=2, column=1, padx=5, pady=5)

    tk.Label(root, text="Telegram User ID").grid(row=3, column=0, sticky="w", padx=5, pady=5)
    entry_user_id = tk.Entry(root, width=50)
    entry_user_id.grid(row=3, column=1, padx=5, pady=5)

    tk.Label(root, text="Binance API Key").grid(row=4, column=0, sticky="w", padx=5, pady=5)
    entry_binance_api_key = tk.Entry(root, width=50)
    entry_binance_api_key.grid(row=4, column=1, padx=5, pady=5)

    tk.Label(root, text="Binance Secret Key").grid(row=5, column=0, sticky="w", padx=5, pady=5)
    entry_binance_secret_key = tk.Entry(root, width=50, show="*")
    entry_binance_secret_key.grid(row=5, column=1, padx=5, pady=5)

    # Список каналов
    tk.Label(root, text="Telegram Channels").grid(row=6, column=0, sticky="w", padx=5, pady=5)
    channels_listbox = tk.Listbox(root, height=5, width=50)
    channels_listbox.grid(row=6, column=1, padx=5, pady=5)

    # Кнопка "Добавить канал"
    tk.Button(
        root,
        text="Добавить канал",
        command=lambda: add_channel_window(root, channels_listbox, channels)
    ).grid(row=7, column=1, sticky="e", padx=5, pady=5)

    # Поле логирования
    log_text = scrolledtext.ScrolledText(root, width=70, height=10)
    log_text.grid(row=8, column=0, columnspan=2, padx=5, pady=5)

    # Кнопка "Пуск"
    tk.Button(
        root,
        text="Пуск",
        command=lambda: start_script(
            entry_api_id.get(),
            entry_api_hash.get(),
            entry_bot_token.get(),
            entry_user_id.get(),
            entry_binance_api_key.get(),
            entry_binance_secret_key.get(),
            channels,
            log_text,
            root
        )
    ).grid(row=9, column=0, columnspan=2, pady=10)

    # Инициализация базы данных при запуске
    init_database()

    return root

if __name__ == "__main__":
    root = create_gui()
    root.mainloop()
