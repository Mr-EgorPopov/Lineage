# ==================== СКРЫТИЕ КОНСОЛИ В EXE ====================
import sys
import os
import ctypes

def hide_console():
    """Скрывает консольное окно при запуске из EXE"""
    try:
        if getattr(sys, 'frozen', False):  # Если это собранный EXE
            kernel32 = ctypes.WinDLL('kernel32')
            user32 = ctypes.WinDLL('user32')
            hwnd = kernel32.GetConsoleWindow()
            if hwnd:
                user32.ShowWindow(hwnd, 0)  # 0 = SW_HIDE
    except:
        pass  # Игнорируем ошибки, если что-то пошло не так

# Скрываем консоль сразу при запуске
hide_console()

# ==================== ОСНОВНЫЕ ИМПОРТЫ ====================
import pyautogui
import time
import random
import pytesseract
from PIL import Image
import keyboard
import re
import threading
import requests
import tkinter as tk
from tkinter import ttk
import numpy as np

# ==================== КОНФИГУРАЦИЯ OCR ====================
pytesseract.pytesseract.tesseract_cmd = r'C:\Program Files\Tesseract-OCR\tesseract.exe'

# ==================== ГЛОБАЛЬНЫЕ ПЕРЕМЕННЫЕ ====================
Limit_summ = 1
stop_event = threading.Event()
current_price = 0
stop_sum = 0
mode = "sell"  # sell или buy
script_running = False
answer = "1"

# ==================== ТЕЛЕГРАМ НАСТРОЙКИ ====================
TELEGRAM_SELL_TOKEN = '7648784424:AAEX1UE4V8azmzTFfiSaoOXKWqQn6KbDTOA'
TELEGRAM_BUY_TOKEN = '8137914112:AAEYHuuu_rbL5fhHLQ41DtW2GxZZUJp-aLM'
TELEGRAM_CHAT_ID = '5268693450'

# ==================== КООРДИНАТЫ (продажа) ====================
SCREEN_MY_NICKNAME = (1361, 296, 1485, 331)
SCREEN_ENEMY = (1173, 356, 1327, 386)
SCREEN_PRICE = (1159, 302, 1326, 333)
SCREEN_BALANCE = (606, 922, 741, 946)
CLICK_UPDATE = (1639, 246)
CLICK_COLUMN = (1282, 277)
CLICK_REDACT = (226, 458)
DCLICK_COIN = (277, 187)
CLICK_ALL = (443, 366)
DCLICK_MYCOIN = (32, 187)
CLICK_START_SELL = (303, 496)

# ==================== КООРДИНАТЫ (покупка) ====================
screen_my_nickname = (1361, 296, 1485, 331)
screen_enemy = (1173, 356, 1327, 386)
screen_nickname = (1373, 358, 1502, 392)
click_update = (1639, 246)
click_column = (1282, 277)
click_redact = (226, 458)
dclick_coin = (277, 187)
click_all = (443, 366)
dclick_mycoin = (32, 187)
screen_price = (1159, 302, 1326, 333)
screen_balance = (606, 922, 741, 946)
click_start_sell = (303, 496)
screen_wait = (286, 279, 383, 290)

# ==================== ОБЩИЕ НАСТРОЙКИ ====================
DELAY_BEFORE_CLICK = 0.5
DELAY_AFTER_CLICK = 0.5
DELAY_MOUSE = 0.1
TOLERANCE = 10

# ==================== БАЗОВЫЕ ФУНКЦИИ ====================

def take_screenshot(x1, y1, x2, y2):
    """Возвращает PIL.Image.Image — без сохранения на диск"""
    return pyautogui.screenshot(region=(x1, y1, x2 - x1, y2 - y1))


def recognize_price(image: Image.Image):
    try:
        image = image.resize((328, 62))
        text = pytesseract.image_to_string(image, config='digit', lang='rus')
        text = text.replace('мпн', 'млн').replace('мл', 'млн')
        total = 0
        if 'млн' in text:
            part = text.split(' млн')[0]
            if ' ' in part:
                part = part.split(' ')[-1]
            total += int(part) * 1000000 if part.isdigit() else 0
        if 'тыс' in text:
            part = text.split(' тыс')[0]
            if ' ' in part:
                part = part.split(' ')[-1]
            total += int(part) * 1000 if part.isdigit() else 0
        if 'аден' in text:
            part = text.split(' аден')[0]
            if ' ' in part:
                part = part.split(' ')[-1]
            if part.isdigit() and part not in ['тыс', 'млн']:
                total += int(part)
        return total
    except Exception as e:
        print(f"Ошибка распознавания цены: {e}")
        return 0


def recognize_sum(image: Image.Image):
    try:
        image = image.resize((328, 62))
        text = pytesseract.image_to_string(image, config='digit', lang='rus')
        text = text.replace('мпн', 'млн').replace('мл', 'млн')
        mln = 0
        if 'млн' in text:
            mln = text.split(' млн')[0]
            if ' ' in mln:
                mln = text.split(' ')[1]
        tsh = 0
        if 'тыс' in text:
            tsh = text.split(' тыс')[0]
            if ' ' in tsh:
                tsh = tsh.split(' ')
                n = tsh.__len__()
                tsh = tsh[n - 1]
        eden = 0
        if 'аден' in text:
            eden = text.split(' аден')[0]
            if ' ' in eden:
                eden = eden.split(' ')
                n = eden.__len__()
                eden = eden[n - 1]
        mln = int(1) * 2000000
        tsh = int(tsh) * 1000 if tsh else 0
        if eden == 'тыс' or eden == 'млн':
            eden = 0
        else:
            eden = int(eden) if eden else 0
        rez = str(mln + tsh + eden)
        return int(rez)
    except Exception as e:
        print(f"Ошибка распознавания текста: {e}")
        return 0


def recognize_balance():
    try:
        image = take_screenshot(*SCREEN_BALANCE)
        image = image.resize((328, 62))
        text = pytesseract.image_to_string(image, config='--psm 6 --oem 3', lang='rus')
        text = ''.join(re.findall(r'\d', text))
        return int(text) if text else 0
    except Exception as e:
        print(f"Ошибка распознавания баланса: {e}")
        return 0


def click(x, y, double=False):
    pyautogui.moveTo(x, y, duration=0.1)
    time.sleep(DELAY_BEFORE_CLICK)
    if double:
        for _ in range(2):
            pyautogui.mouseDown()
            time.sleep(DELAY_MOUSE)
            pyautogui.mouseUp()
            time.sleep(DELAY_MOUSE)
    else:
        pyautogui.mouseDown()
        time.sleep(DELAY_MOUSE)
        pyautogui.mouseUp()
    time.sleep(DELAY_AFTER_CLICK)


def double_click(x, y):
    pyautogui.moveTo(x, y, duration=0.1)
    time.sleep(DELAY_BEFORE_CLICK)
    pyautogui.mouseDown(button='left')
    time.sleep(DELAY_MOUSE)
    pyautogui.mouseUp(button='left')
    time.sleep(DELAY_MOUSE)
    pyautogui.mouseDown(button='left')
    time.sleep(DELAY_MOUSE)
    pyautogui.mouseUp(button='left')
    time.sleep(DELAY_AFTER_CLICK)


def type_and_enter(text):
    pyautogui.write(str(text))
    time.sleep(DELAY_BEFORE_CLICK)
    pyautogui.press('enter')


def compare_images(img1: Image.Image, img2: Image.Image):
    try:
        arr1 = np.array(img1.convert('L'))
        arr2 = np.array(img2.convert('L'))
        if arr1.shape != arr2.shape:
            return False
        diff = np.max(np.abs(arr1.astype(int) - arr2.astype(int)))
        return diff <= TOLERANCE
    except Exception as e:
        print(f"Ошибка сравнения: {e}")
        return False


def send_telegram_message(message, is_buy=False):
    try:
        token = TELEGRAM_BUY_TOKEN if is_buy else TELEGRAM_SELL_TOKEN
        url = f"https://api.telegram.org/bot{token}/sendMessage"
        params = {"chat_id": TELEGRAM_CHAT_ID, "text": message}
        requests.get(url, params=params, timeout=5)
    except Exception as e:
        print(f"Ошибка Telegram: {e}")


def get_wait_time(mode):
    if mode == "1":
        return 70
    else:
        return random.randint(70, 180)


# ==================== ФУНКЦИИ ДЛЯ ПРОДАЖИ ====================

def check_balance_and_notify_sell():
    balance = recognize_balance()
    print(f"Баланс: {balance}")
    if balance > (Limit_summ*1000000000):
        send_telegram_message("🔥🔥🔥ВСЕ ПРОДАЛОСЬ🔥🔥🔥", is_buy=False)
        time.sleep(3)
        send_telegram_message(f"🔥🔥🔥ПОСЛЕДНЯЯ ЦЕНА: {current_price}🔥🔥🔥", is_buy=False)
        return True
    return False


def change_price(new_price):
    print(f"Меняем цену на {new_price}")
    click(CLICK_REDACT[0], CLICK_REDACT[1])
    time.sleep(2)
    click(DCLICK_COIN[0], DCLICK_COIN[1], double=True)
    time.sleep(1.5)
    click(CLICK_ALL[0], CLICK_ALL[1])
    time.sleep(0.8)
    pyautogui.press('enter')
    time.sleep(0.8)
    click(DCLICK_MYCOIN[0], DCLICK_MYCOIN[1], double=True)
    time.sleep(1)
    type_and_enter(new_price)
    time.sleep(0.5)
    click(CLICK_ALL[0], CLICK_ALL[1])
    time.sleep(0.8)
    pyautogui.press('enter')
    time.sleep(1.5)
    click(CLICK_START_SELL[0], CLICK_START_SELL[1])
    return True


def create_my_nickname_screenshot():
    print("Создаю эталон своего ника...")
    try:
        img = take_screenshot(*SCREEN_MY_NICKNAME)
        print("Эталон создан")
        return img
    except Exception as e:
        print(f"Ошибка: {e}")
        return None


def process_selling():
    global current_price, stop_sum
    my_nick = create_my_nickname_screenshot()
    if not my_nick:
        return

    price_img = take_screenshot(*SCREEN_PRICE)
    current_price = recognize_price(price_img)
    print(f"Начальная цена: {current_price}")

    while not stop_event.is_set():
        try:
            if check_balance_and_notify_sell():
                break

            click(CLICK_UPDATE[0], CLICK_UPDATE[1])
            time.sleep(1)
            click(CLICK_COLUMN[0], CLICK_COLUMN[1])
            time.sleep(0.4)
            click(CLICK_COLUMN[0], CLICK_COLUMN[1])
            time.sleep(0.4)

            curr_nick_img = take_screenshot(*SCREEN_MY_NICKNAME)
            if compare_images(my_nick, curr_nick_img):
                print("Это мой лот, проверяю конкурента...")
                comp_img = take_screenshot(*SCREEN_ENEMY)
                comp_price = recognize_price(comp_img)
                print(f"Моя цена: {current_price}, цена конкурента: {comp_price}")
                if comp_price > 0 and (comp_price - current_price) > 40:
                    new_price = comp_price - random.choice([1, 2, 5, 10, 20])
                    if new_price >= stop_sum:
                        change_price(new_price)
                        current_price = new_price
                    else:
                        print(f"Новая цена {new_price} ниже stop_sum {stop_sum}")
                        send_telegram_message("❌❌НИЖЕ ГРАНИЦЫ❌❌", is_buy=False)
                else:
                    print("Разница небольшая, жду...")
            else:
                print("Чужой лот, подрезаю...")
                target_img = take_screenshot(*SCREEN_PRICE)
                target_price = recognize_price(target_img)
                print(f"Цена цели: {target_price}")
                if target_price > 0:
                    my_price = target_price - random.choice([1, 2, 5, 10, 20])
                    if my_price >= stop_sum:
                        change_price(my_price)
                        current_price = my_price
                    else:
                        print(f"Цена {my_price} ниже лимита {stop_sum}")
                        send_telegram_message("❌❌НИЖЕ ГРАНИЦЫ❌❌", is_buy=False)

            wait_sec = get_wait_time(answer)
            print(f"Жду {wait_sec} сек...")
            for _ in range(wait_sec):
                if stop_event.is_set():
                    return
                time.sleep(1)
        except Exception as e:
            print(f"Ошибка в цикле: {e}")
            time.sleep(5)


# ==================== ФУНКЦИИ ДЛЯ ПОКУПКИ ====================

def check_balance_and_notify_buy(added_sum):
    try:
        image = take_screenshot(*screen_balance)
        image = image.resize((328, 62))
        text = pytesseract.image_to_string(image, config='digit', lang='rus')
        text = ''.join(re.findall(r'\d', text))
        sum2 = int(text) if text else 0
        print(f"Остаток адены: {sum2}")
        if sum2 < 1500000:
            try:
                send_telegram_message(f"🔥🔥🔥ВСЕ СКУПИЛИ🔥🔥🔥", is_buy=True)
                time.sleep(3)
                send_telegram_message(f"🔥🔥🔥ПОСЛЕДНЯЯ ЦЕНА: {added_sum}🔥🔥🔥", is_buy=True)
                return True
            except Exception as e:
                print(f"Ошибка при отправке уведомления о sum2: {e}")
        return False
    except Exception as e:
        print(f"Ошибка при выполнении проверки sum2: {e}")
        return False


def calculate_quantity(price_per_item):
    try:
        image = take_screenshot(*screen_balance)
        image = image.resize((328, 62))
        text = pytesseract.image_to_string(image, config='--psm 6 --oem 3', lang='rus')
        text = ''.join(re.findall(r'\d', text))
        current_balance = int(text) if text else 0
        print(f"Текущий баланс: {current_balance}")
        if price_per_item <= 0:
            print("Цена должна быть больше 0")
            return 1
        quantity = current_balance // price_per_item
        quantity = max(1, quantity)
        print(f"Можем купить {quantity} предметов по цене {price_per_item} каждый")
        return quantity
    except Exception as e:
        print(f"Ошибка при вычислении количества: {e}")
        return 1


def process_buying():
    global stop_sum, stop_event
    step = 0
    added_sum = 0
    my_nickname_screenshot = create_my_nickname_screenshot()
    if my_nickname_screenshot is None:
        print("Не удалось создать скриншот ника. Остановка скрипта.")
        return

    while not stop_event.is_set():
        if step == 0:
            if check_balance_and_notify_buy(added_sum):
                break
            print("Шаг 0: Обновляем список товаров...")
            click(click_update[0], click_update[1])
            time.sleep(1)
            click(click_column[0], click_column[1])
            time.sleep(0.4)
            click(click_column[0], click_column[1])
            time.sleep(0.4)
            step = 1

        elif step == 1:
            print("Шаг 1: Проверяем продавца...")
            current_nickname_img = take_screenshot(*screen_my_nickname)
            if compare_images(my_nickname_screenshot, current_nickname_img):
                print("Я продаю этот товар! Пропускаем...")
                step = 7
            else:
                print("Это не я продаю, продолжаем покупку...")
                step = 2

        elif step == 2:
            print("Шаг 2: Определяем цену покупки...")
            price_img = take_screenshot(*screen_price)
            current_price = recognize_sum(price_img)
            print(f"Цена конкурента: {current_price}")
            added_sum = current_price + random.choice([1, 2, 5, 10, 20])
            print(f"Наша цена покупки: {added_sum}")
            if added_sum >= stop_sum:
                print(f"Цена {added_sum} превысила лимит {stop_sum}!")
                try:
                    send_telegram_message(f"❌❌ВЫШЕ ГРАНИЦЫ❌❌", is_buy=True)
                    time.sleep(2)
                except Exception as e:
                    print(f"Ошибка при отправке сообщения Telegram: {e}")
                wait_time = 0
                pizdec = get_wait_time(answer)
                while wait_time < pizdec and not stop_event.is_set():
                    time.sleep(1)
                    wait_time += 1
                if stop_event.is_set():
                    break
                step = 0
                continue

            print("Начинаем процесс покупки...")
            click(click_redact[0], click_redact[1])
            time.sleep(2)
            double_click(dclick_coin[0], dclick_coin[1])
            time.sleep(1.5)
            click(click_all[0], click_all[1])
            time.sleep(1)
            pyautogui.press('enter')
            time.sleep(0.8)
            double_click(dclick_mycoin[0], dclick_mycoin[1])
            time.sleep(1)
            step = 3

        elif step == 3:
            print("Шаг 3: Вводим цену...")
            pyautogui.write(str(added_sum))
            time.sleep(0.5)
            pyautogui.press('enter')
            time.sleep(0.8)
            step = 4

        elif step == 4:
            print("Шаг 4: Вычисляем количество для покупки...")
            quantity = calculate_quantity(added_sum)
            pyautogui.write(str(quantity))
            time.sleep(0.5)
            pyautogui.press('enter')
            time.sleep(1.5)
            step = 5

        elif step == 5:
            print("Шаг 5: Подтверждаем покупку...")
            click(click_start_sell[0], click_start_sell[1])
            wait_time = 0
            pizdec = get_wait_time(answer)
            print(f"Ожидание {pizdec} секунд...")
            while wait_time < pizdec and not stop_event.is_set():
                time.sleep(1)
                wait_time += 1
            if stop_event.is_set():
                break
            step = 0
            continue

        elif step == 6:
            print("Я продаю этот товар, ждем...")
            wait_time = 0
            pizdec = get_wait_time(answer)
            while wait_time < pizdec and not stop_event.is_set():
                time.sleep(1)
                wait_time += 1
            if stop_event.is_set():
                break
            step = 0
            continue

        elif step == 7:
            print("Шаг 7: Проверяем мою цену и конкурента...")
            my_price_img = take_screenshot(*screen_price)
            my_price = recognize_sum(my_price_img)
            enemy_price_img = take_screenshot(*screen_enemy)
            enemy_price = recognize_sum(enemy_price_img)
            raznica = my_price - enemy_price
            added_sum = enemy_price + random.choice([1, 2, 5, 10, 20])
            if raznica > 40:
                print("Начинаем процесс покупки...")
                click(click_redact[0], click_redact[1])
                time.sleep(2)
                double_click(dclick_coin[0], dclick_coin[1])
                time.sleep(1.5)
                click(click_all[0], click_all[1])
                time.sleep(1)
                pyautogui.press('enter')
                time.sleep(0.8)
                double_click(dclick_mycoin[0], dclick_mycoin[1])
                time.sleep(1)
                step = 3
            else:
                step = 6


# ==================== СОВРЕМЕННЫЙ ГРАФИЧЕСКИЙ ИНТЕРФЕЙС ====================

class ModernTradeApp:
    def __init__(self, root):
        self.root = root
        self.root.title("Auto Trade Pro")
        self.root.geometry("500x550")
        self.root.resizable(False, False)
        # Современная цветовая схема
        self.bg_color = "#0f0f23"
        self.card_bg = "#1a1a2e"
        self.accent_color = "#00d4ff"
        self.success_color = "#00ff9d"
        self.warning_color = "#ffaa00"
        self.danger_color = "#ff4757"
        self.text_color = "#ffffff"
        self.text_muted = "#b0b0b0"
        self.border_color = "#2d3047"
        self.switch_bg = "#2d3047"
        self.switch_knob = "#00d4ff"
        self.switch_knob_shadow = "#0088aa"
        self.switch_text_active = "#ffffff"
        self.switch_text_inactive = "#b0b0b0"
        # Настройка фона главного окна
        self.root.configure(bg=self.bg_color)
        # Настройка стилей для ttk
        self.setup_styles()
        # Создание интерфейса
        self.create_ui()
        # Горячие клавиши
        keyboard.add_hotkey('f10', self.start_script)
        keyboard.add_hotkey('f9', self.pause_script)
        keyboard.add_hotkey('esc', self.stop_script)

    def setup_styles(self):
        """Настройка современных стилей для виджетов"""
        style = ttk.Style()
        style.theme_use('clam')
        # Кастомные стили
        style.configure('Title.TLabel',
                       background=self.bg_color,
                       foreground=self.accent_color,
                       font=('Segoe UI', 20, 'bold'))
        style.configure('Subtitle.TLabel',
                       background=self.bg_color,
                       foreground=self.text_color,
                       font=('Segoe UI', 11, 'bold'))
        style.configure('Card.TFrame',
                       background=self.card_bg,
                       relief='flat',
                       borderwidth=0)
        style.configure('Primary.TButton',
                       background=self.accent_color,
                       foreground=self.bg_color,
                       font=('Segoe UI', 10, 'bold'),
                       borderwidth=0,
                       focuscolor='none',
                       padding=10)
        style.map('Primary.TButton',
                 background=[('active', '#00a8cc')],
                 foreground=[('active', self.bg_color)])
        style.configure('Secondary.TButton',
                       background=self.card_bg,
                       foreground=self.text_color,
                       font=('Segoe UI', 10),
                       borderwidth=0,
                       padding=8)
        style.map('Secondary.TButton',
                 background=[('active', '#2d3047')])
        style.configure('Modern.TEntry',
                       fieldbackground=self.card_bg,
                       foreground=self.text_color,
                       borderwidth=2,
                       relief='solid',
                       insertcolor=self.text_color)
        style.configure('Status.TLabel',
                       background=self.card_bg,
                       foreground=self.success_color,
                       font=('Segoe UI', 10, 'bold'),
                       padding=10)

    def create_card_frame(self, parent, title):
        """Создает карточку с закругленными углами"""
        container = ttk.Frame(parent, style='Card.TFrame', padding="15")
        # Заголовок карточки
        if title:
            title_label = ttk.Label(container,
                                  text=title,
                                  style='Subtitle.TLabel',
                                  background=self.card_bg)
            title_label.pack(anchor='w', pady=(0, 10))
        return container

    def create_modern_switch(self, parent):
        """Создает современный переключатель-ползунок"""
        switch_frame = tk.Frame(parent, bg=self.card_bg, height=60)
        switch_frame.pack_propagate(False)
        # Переменная для хранения состояния
        self.mode_var = tk.StringVar(value="buy")  # buy или sell
        # Основной контейнер переключателя
        switch_container = tk.Frame(switch_frame, bg=self.card_bg)
        switch_container.pack(expand=True, fill='both')
        # Создаем Canvas для сглаженной графики
        self.switch_canvas = tk.Canvas(switch_container,
                                      bg=self.card_bg,
                                      height=50,
                                      width=260,
                                      highlightthickness=0,
                                      relief='flat')
        self.switch_canvas.pack(pady=5)
        # Параметры переключателя
        self.switch_width = 240
        self.switch_height = 40
        self.knob_radius = 18
        self.padding = 10
        # Текущая позиция ползунка
        self.knob_position = "left"  # или "right"
        # Рисуем начальное состояние
        self.draw_switch()

        def toggle_switch(event=None):
            """Переключает состояние"""
            current_mode = self.mode_var.get()
            new_mode = "sell" if current_mode == "buy" else "buy"
            self.mode_var.set(new_mode)
            # Обновляем глобальную переменную
            global mode
            mode = new_mode
            # Анимация перемещения ползунка
            self.animate_knob(new_mode)

        # Привязываем клик
        self.switch_canvas.bind("<Button-1>", toggle_switch)
        return switch_frame

    def draw_switch(self):
        """Рисует переключатель на canvas"""
        canvas = self.switch_canvas
        canvas.delete("all")
        # Фон переключателя (прямоугольник с закругленными краями)
        bg_color = self.switch_bg
        canvas.create_rectangle(self.padding, 5,
                              self.padding + self.switch_width, self.switch_height - 5,
                              fill=bg_color, outline="", width=0)
        # Закругленные края
        corner_radius = 8
        canvas.create_oval(self.padding, 5,
                          self.padding + corner_radius * 2, 5 + corner_radius * 2,
                          fill=bg_color, outline="", width=0)
        canvas.create_oval(self.padding, self.switch_height - 5 - corner_radius * 2,
                          self.padding + corner_radius * 2, self.switch_height - 5,
                          fill=bg_color, outline="", width=0)
        canvas.create_oval(self.padding + self.switch_width - corner_radius * 2, 5,
                          self.padding + self.switch_width, 5 + corner_radius * 2,
                          fill=bg_color, outline="", width=0)
        canvas.create_oval(self.padding + self.switch_width - corner_radius * 2,
                          self.switch_height - 5 - corner_radius * 2,
                          self.padding + self.switch_width, self.switch_height - 5,
                          fill=bg_color, outline="", width=0)

        # Тексты на переключателе
        left_text = "💰 ПОКУПКА"
        right_text = "🛒 ПРОДАЖА"
        left_color = self.switch_text_active if self.mode_var.get() == "buy" else self.switch_text_inactive
        right_color = self.switch_text_inactive if self.mode_var.get() == "buy" else self.switch_text_active
        canvas.create_text(self.padding + 60, self.switch_height // 2,
                          text=left_text,
                          font=('Segoe UI', 10, 'bold'),
                          fill=left_color)
        canvas.create_text(self.padding + self.switch_width - 60, self.switch_height // 2,
                          text=right_text,
                          font=('Segoe UI', 10, 'bold'),
                          fill=right_color)

        # Ползунок
        if self.mode_var.get() == "buy":
            knob_x = self.padding + 30
        else:
            knob_x = self.padding + self.switch_width - 30
        knob_y = self.switch_height // 2

        shadow_offset = 2
        canvas.create_oval(knob_x - self.knob_radius + shadow_offset,
                          knob_y - self.knob_radius + shadow_offset,
                          knob_x + self.knob_radius + shadow_offset,
                          knob_y + self.knob_radius + shadow_offset,
                          fill=self.switch_knob_shadow, outline="", width=0)
        canvas.create_oval(knob_x - self.knob_radius,
                          knob_y - self.knob_radius,
                          knob_x + self.knob_radius,
                          knob_y + self.knob_radius,
                          fill=self.switch_knob, outline="", width=0)
        highlight_radius = self.knob_radius - 4
        canvas.create_oval(knob_x - highlight_radius,
                          knob_y - highlight_radius,
                          knob_x + highlight_radius,
                          knob_y + highlight_radius,
                          fill="#88eeff", outline="", width=0)

    def animate_knob(self, new_mode):
        """Анимирует перемещение ползунка"""
        if new_mode == "buy":
            target_x = self.padding + 30
        else:
            target_x = self.padding + self.switch_width - 30
        current_x = self.padding + 30 if self.mode_var.get() == "sell" else self.padding + self.switch_width - 30
        steps = 15
        dx = (target_x - current_x) / steps
        for i in range(steps):
            current_x += dx
            self.draw_switch()
            self.root.update()
            time.sleep(0.01)
        self.draw_switch()

    def create_ui(self):
        """Создание современного интерфейса"""
        main_container = ttk.Frame(self.root, style='Card.TFrame')
        main_container.pack(fill='both', expand=True, padx=20, pady=20)

        title_label = ttk.Label(main_container,
                               text="AUTO TRADE PRO",
                               style='Title.TLabel')
        title_label.pack(pady=(0, 20))

        mode_card = self.create_card_frame(main_container, "📱 ВЫБЕРИТЕ РЕЖИМ")
        mode_card.pack(fill='x', pady=(0, 15), ipady=10)
        self.switch = self.create_modern_switch(mode_card)
        self.switch.pack(fill='x', pady=10)

        settings_card = self.create_card_frame(main_container, "⚙️ НАСТРОЙКИ")
        settings_card.pack(fill='x', pady=(0, 15), ipady=10)

        stop_frame = ttk.Frame(settings_card, style='Card.TFrame')
        stop_frame.pack(fill='x', pady=(0, 10))
        stop_label = ttk.Label(stop_frame,
                              text="СТОП СУММА:",
                              style='Subtitle.TLabel',
                              background=self.card_bg)
        stop_label.pack(side='left', padx=(0, 10))
        self.stop_sum_entry = ttk.Entry(stop_frame,
                                       style='Modern.TEntry',
                                       font=('Segoe UI', 10),
                                       width=20)
        self.stop_sum_entry.pack(side='left')

        delay_frame = ttk.Frame(settings_card, style='Card.TFrame')
        delay_frame.pack(fill='x')
        delay_label = ttk.Label(delay_frame,
                               text="ЗАДЕРЖКА:",
                               style='Subtitle.TLabel',
                               background=self.card_bg)
        delay_label.pack(side='left', padx=(0, 10))
        self.delay_var = tk.StringVar(value="1")
        delay_btn_frame = tk.Frame(delay_frame, bg=self.card_bg)
        delay_btn_frame.pack(side='left')
        delay_70 = tk.Radiobutton(delay_btn_frame,
                                 text="70 сек",
                                 variable=self.delay_var,
                                 value="1",
                                 font=('Segoe UI', 10),
                                 bg=self.card_bg,
                                 fg=self.text_color,
                                 activebackground=self.card_bg,
                                 activeforeground=self.text_color,
                                 selectcolor=self.card_bg,
                                 cursor="hand2",
                                 highlightthickness=0,
                                 bd=0)
        delay_70.pack(side='left', padx=(0, 15))
        delay_rand = tk.Radiobutton(delay_btn_frame,
                                   text="70-180 сек",
                                   variable=self.delay_var,
                                   value="2",
                                   font=('Segoe UI', 10),
                                   bg=self.card_bg,
                                 fg=self.text_color,
                                 activebackground=self.card_bg,
                                 activeforeground=self.text_color,
                                 selectcolor=self.card_bg,
                                 cursor="hand2",
                                 highlightthickness=0,
                                 bd=0)
        delay_rand.pack(side='left')

        self.status_var = tk.StringVar(value="✅ ГОТОВ К ЗАПУСКУ")
        status_card = ttk.Frame(main_container, style='Card.TFrame', padding="10")
        status_card.pack(fill='x', pady=(0, 15))
        status_label = ttk.Label(status_card,
                                textvariable=self.status_var,
                                style='Status.TLabel')
        status_label.pack(fill='x')

        control_card = self.create_card_frame(main_container, "🎮 УПРАВЛЕНИЕ")
        control_card.pack(fill='x', pady=(0, 15), ipady=10)
        btn_frame = ttk.Frame(control_card, style='Card.TFrame')
        btn_frame.pack(fill='x', pady=10)
        self.start_button = ttk.Button(btn_frame,
                                      text="🚀 СТАРТ (F10)",
                                      command=self.start_script,
                                      style='Primary.TButton',
                                      width=15)
        self.start_button.pack(side='left', expand=True, padx=5)
        self.pause_button = ttk.Button(btn_frame,
                                       text="⏸️ ПАУЗА (F9)",
                                       command=self.pause_script,
                                       style='Secondary.TButton',
                                       width=15,
                                       state='disabled')
        self.pause_button.pack(side='left', expand=True, padx=5)
        self.stop_button = ttk.Button(btn_frame,
                                      text="🛑 СТОП (ESC)",
                                      command=self.stop_script,
                                      style='Secondary.TButton',
                                      width=15)
        self.stop_button.pack(side='left', expand=True, padx=5)

        info_card = self.create_card_frame(main_container, "ℹ️ ИНФОРМАЦИЯ")
        info_card.pack(fill='x', ipady=10)
        info_text = """• F10 - Запуск скрипта
• F9 - Пауза скрипта
• ESC - Аварийная остановка
📊 Скрипт автоматически анализирует рынок
🔔 Отправляет уведомления в Telegram
⚡ Работает в фоновом режиме"""
        info_label = tk.Label(info_card,
                             text=info_text,
                             font=('Segoe UI', 9),
                             bg=self.card_bg,
                             fg=self.text_muted,
                             justify='left',
                             anchor='w')
        info_label.pack(fill='x', pady=5)

        footer = ttk.Label(main_container,
                          text="Auto Trade Pro v2.1 • Python 3.14",
                          style='Subtitle.TLabel',
                          foreground=self.text_muted)
        footer.pack(pady=(15, 0))

    def start_script(self):
        global stop_sum, mode, answer, script_running
        if script_running:
            return
        try:
            stop_sum = int(self.stop_sum_entry.get())
        except ValueError:
            self.status_var.set("❌ ОШИБКА: введите число в стоп сумму")
            return
        mode = self.mode_var.get()
        answer = self.delay_var.get()
        status_text = f"🚀 ЗАПУЩЕН РЕЖИМ: {'ПРОДАЖИ' if mode == 'sell' else 'ПОКУПКИ'}"
        self.status_var.set(status_text)
        self.start_button.config(state='disabled')
        self.pause_button.config(state='normal')
        stop_event.clear()
        script_running = True
        thread = threading.Thread(target=self.run_script)
        thread.daemon = True
        thread.start()

    def run_script(self):
        global script_running
        try:
            if mode == "sell":
                process_selling()
            else:
                process_buying()
        except Exception as e:
            self.root.after(0, lambda: self.status_var.set(f"❌ ОШИБКА: {str(e)[:30]}..."))
        script_running = False
        self.root.after(0, self.reset_buttons)

    def pause_script(self):
        global script_running
        if script_running:
            stop_event.set()
            self.status_var.set("⏸️ ПАУЗА")
            self.pause_button.config(state='disabled')
            self.start_button.config(state='normal')
            script_running = False

    def stop_script(self):
        global script_running
        stop_event.set()
        self.status_var.set("🛑 ОСТАНОВЛЕНО")
        self.reset_buttons()
        script_running = False

    def reset_buttons(self):
        self.start_button.config(state='normal')
        self.pause_button.config(state='disabled')
        self.status_var.set("✅ ГОТОВ К ЗАПУСКУ")


# ==================== ЗАПУСК ПРИЛОЖЕНИЯ ====================
if __name__ == "__main__":
    root = tk.Tk()
    try:
        root.iconbitmap('icon.ico')
    except:
        pass
    app = ModernTradeApp(root)

    def on_closing():
        stop_event.set()
        keyboard.unhook_all_hotkeys()
        root.destroy()
        sys.exit()

    root.protocol("WM_DELETE_WINDOW", on_closing)
    root.update_idletasks()
    width = root.winfo_width()
    height = root.winfo_height()
    x = (root.winfo_screenwidth() // 2) - (width // 2)
    y = (root.winfo_screenheight() // 2) - (height // 2)
    root.geometry(f'{width}x{height}+{x}+{y}')
    root.mainloop()