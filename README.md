import vk_api
from vk_api.longpoll import VkLongPoll, VkEventType
from vk_api.utils import get_random_id
import sqlite3
import time
import datetime
import random
import logging
import sys
import os
import re

TOKEN = os.environ.get('VK_TOKEN', 'vk1.a.hczRCAHcJ503htmFIydw0l5Vu32Zcgsca5MvCAg03PU5qWteDUi-jO3a_NWYnGrRk0gKmOji3-bEK5qfp_dlnOTvuErLO2h9ImG8MgxfH84Msr_ZQFfOv8_M9jijfCG8mUUz9dgqLmKiCHEtnPX2Slsf6zHqEoyphYeqaiV0ddaUZ7A4lIl9YfsFmVadZY645o7mk_aUvK_dYFihF3PVTQ')
CREATOR_ID = 1085788257
BOT_GROUP_ID = 241262855

MAT_WORDS = ['блять', 'сука', 'хуй', 'пизда', 'ебать', 'еблан', 'мудак', 'говно', 'залупа', 'член', 'шлюха', 'блядь', 'пидор', 'гандон', 'мразь']

MODER_ROLES = ['agent', 'moder', 'senior_moder', 'zgm', 'head_moder', 'manager', 'zrm', 'head_of_moderation', 'head_admin', 'helper', 'head_watcher', 'watcher', 'junior_watcher']
ROLE_NAMES = {
    'user': 'Участник', 'agent': 'Агент поддержки', 'moder': 'Модератор',
    'senior_moder': 'Старший модератор', 'zgm': 'ЗГМ', 'head_moder': 'Главный модератор',
    'manager': 'Менеджер', 'zrm': 'ЗРМ', 'head_of_moderation': 'Руководитель модерации',
    'admin': 'Администратор', 'tech': 'Тех. специалист', 'dev': 'Разработчик',
    'youtuber': 'Ютубер', 'owner': 'Владелец', 'deputy': 'Зам.владельца',
    'head_admin': 'Главный Админ', 'helper': 'Хелпер', 'head_watcher': 'Гл.Смотрящий',
    'watcher': 'Смотрящий', 'junior_watcher': 'Мл.Смотрящий', 'sponsor': 'Спонсор',
    'premium': 'Премиум пользователь'
}

logging.basicConfig(level=logging.INFO, format='%(asctime)s - %(levelname)s - %(message)s', handlers=[logging.FileHandler('bot.log', encoding='utf-8'), logging.StreamHandler(sys.stdout)])
logger = logging.getLogger(__name__)

vk_session = vk_api.VkApi(token=TOKEN)
vk = vk_session.get_api()
longpoll = VkLongPoll(vk_session)

conn = sqlite3.connect('sberhub.db', check_same_thread=False)
cursor = conn.cursor()

def ensure_column(table, column, col_type):
    cursor.execute(f"PRAGMA table_info({table})")
    cols = [row[1] for row in cursor.fetchall()]
    if column not in cols:
        if 'CURRENT_TIMESTAMP' in col_type:
            col_type = col_type.replace('DEFAULT CURRENT_TIMESTAMP', "DEFAULT ''")
        cursor.execute(f"ALTER TABLE {table} ADD COLUMN {column} {col_type}")
        conn.commit()

def init_db():
    cursor.execute('''CREATE TABLE IF NOT EXISTS users (
        user_id INTEGER PRIMARY KEY, balance REAL DEFAULT 500, passport INTEGER DEFAULT 0,
        card INTEGER DEFAULT 0, bank_worker INTEGER DEFAULT 0, rank INTEGER DEFAULT 0,
        bank_level INTEGER DEFAULT 0, experience INTEGER DEFAULT 0, house TEXT DEFAULT NULL,
        car TEXT DEFAULT NULL, company TEXT DEFAULT NULL, family INTEGER DEFAULT 0,
        last_daily TEXT, last_work TEXT, spam_balance REAL DEFAULT 0, mute_until TEXT,
        role TEXT DEFAULT 'user', credit_amount REAL DEFAULT 0, credit_due TEXT,
        vip INTEGER DEFAULT 0, promocodes_used TEXT DEFAULT '',
        tt_channel TEXT DEFAULT NULL, tt_subscribers INTEGER DEFAULT 0, tt_verified INTEGER DEFAULT 0,
        tt_videos INTEGER DEFAULT 0, tt_likes INTEGER DEFAULT 0, last_tt_film TEXT,
        level INTEGER DEFAULT 1, reputation INTEGER DEFAULT 0, clan TEXT DEFAULT NULL,
        registration_date TEXT DEFAULT '', hidden_from_top INTEGER DEFAULT 0,
        limited_items TEXT DEFAULT '')''')
    cursor.execute('''CREATE TABLE IF NOT EXISTS bans (user_id INTEGER PRIMARY KEY, reason TEXT, timestamp TEXT)''')
    cursor.execute('''CREATE TABLE IF NOT EXISTS mutes (user_id INTEGER PRIMARY KEY, until TEXT, reason TEXT)''')
    cursor.execute('''CREATE TABLE IF NOT EXISTS logs (id INTEGER PRIMARY KEY AUTOINCREMENT, user_id INTEGER, action TEXT, timestamp TEXT)''')
    cursor.execute('''CREATE TABLE IF NOT EXISTS user_stats (user_id INTEGER PRIMARY KEY, messages_count INTEGER DEFAULT 0, mat_count INTEGER DEFAULT 0, photo_count INTEGER DEFAULT 0, voice_count INTEGER DEFAULT 0, video_count INTEGER DEFAULT 0, last_message_time TEXT)''')
    cursor.execute('''CREATE TABLE IF NOT EXISTS promocodes (code TEXT PRIMARY KEY, reward REAL, vip INTEGER DEFAULT 0, created_by INTEGER, used_by TEXT DEFAULT '')''')
    cursor.execute('''CREATE TABLE IF NOT EXISTS offers (id INTEGER PRIMARY KEY AUTOINCREMENT, user_id INTEGER, text TEXT, timestamp TEXT)''')
    cursor.execute('''CREATE TABLE IF NOT EXISTS user_quests (user_id INTEGER, quest_id INTEGER, progress INTEGER DEFAULT 0, claimed INTEGER DEFAULT 0, PRIMARY KEY (user_id, quest_id))''')
    cursor.execute('''CREATE TABLE IF NOT EXISTS reports (id INTEGER PRIMARY KEY AUTOINCREMENT, user_id INTEGER, text TEXT, timestamp TEXT)''')
    cursor.execute('''CREATE TABLE IF NOT EXISTS wipe_info (id INTEGER PRIMARY KEY, last_wipe TEXT)''')
    cursor.execute('''CREATE TABLE IF NOT EXISTS limited_items (id INTEGER PRIMARY KEY AUTOINCREMENT, name TEXT UNIQUE, created_by INTEGER, timestamp TEXT)''')
    conn.commit()
    for col, col_type in [
        ('credit_amount','REAL DEFAULT 0'), ('credit_due','TEXT'), ('last_work','TEXT'),
        ('vip','INTEGER DEFAULT 0'), ('promocodes_used',"TEXT DEFAULT ''"),
        ('tt_channel','TEXT'), ('tt_subscribers','INTEGER DEFAULT 0'), ('tt_verified','INTEGER DEFAULT 0'),
        ('tt_videos','INTEGER DEFAULT 0'), ('tt_likes','INTEGER DEFAULT 0'), ('last_tt_film','TEXT'),
        ('level','INTEGER DEFAULT 1'), ('reputation','INTEGER DEFAULT 0'), ('clan','TEXT'),
        ('registration_date',"TEXT DEFAULT ''"), ('hidden_from_top','INTEGER DEFAULT 0'),
        ('limited_items',"TEXT DEFAULT ''")]:
        ensure_column('users', col, col_type)
    conn.commit()

init_db()

def generate_houses():
    types = ["Квартира", "Дом", "Вилла", "Пентхаус", "Таунхаус", "Коттедж", "Особняк", "Усадьба", "Апартаменты", "Шале"]
    levels = ["Эконом", "Стандарт", "Комфорт", "Бизнес", "Премиум", "Люкс", "Элит", "Делюкс", "Эксклюзив", "Королевский"]
    extras = ["Остров", "Планета", "Вселенная", "Галактика", "Мультивселенная"]
    houses = {}
    house_id = 1
    for e in extras:
        for l in levels[:5]:
            houses[f"{e} {l} [ID: {house_id}]"] = 50000 + house_id * 5000
            house_id += 1
    for t in types:
        for l in levels:
            houses[f"{t} {l} [ID: {house_id}]"] = 1000 + house_id * 500
            house_id += 1
    return houses

def generate_cars():
    brands = ["Lada", "Kia", "Toyota", "BMW", "Mercedes", "Audi", "Ford", "Hyundai", "Volkswagen", "Porsche"]
    models = ["Базовая", "Комфорт", "Бизнес", "Премиум", "Люкс", "Спорт", "Эксклюзив", "Лимед", "Гранд", "Королевская"]
    extras = ["Космолёт", "Звездолёт", "Телепорт", "Киберкар", "Флаер"]
    cars = {}
    car_id = 1
    for e in extras:
        for m in models[:5]:
            cars[f"{e} {m} [ID: {car_id}]"] = 20000 + car_id * 2000
            car_id += 1
    for b in brands:
        for m in models:
            cars[f"{b} {m} [ID: {car_id}]"] = 500 + car_id * 100
            car_id += 1
    return cars

def generate_companies():
    sectors = ["Ларёк", "Магазин", "Кафе", "Автосервис", "Салон красоты", "Фитнес-клуб", "Ресторан", "Отель", "Завод", "Корпорация"]
    scales = ["Малый", "Средний", "Крупный", "Сетевой", "Региональный", "Федеральный", "Международный", "Глобальный", "Транснациональный", "Мега"]
    extras = ["Банк", "Космопорт", "Звездная империя", "Галактический холдинг", "Мультивселенная корпорация"]
    companies = {}
    comp_id = 1
    for e in extras:
        for s in scales[:5]:
            companies[f"{e} {s} [ID: {comp_id}]"] = 50000 + comp_id * 5000
            comp_id += 1
    for s in sectors:
        for sc in scales:
            companies[f"{s} {sc} [ID: {comp_id}]"] = 2000 + comp_id * 200
            comp_id += 1
    return companies

HOUSES = generate_houses()
CARS = generate_cars()
COMPANIES = generate_companies()

QUESTS = [
    {'id': 1, 'desc': 'Выполните работу 3 раза', 'type': 'work', 'target': 3, 'reward_money': 100, 'reward_exp': 50},
    {'id': 2, 'desc': 'Выполните фриланс 5 раз', 'type': 'freelance', 'target': 5, 'reward_money': 150, 'reward_exp': 70},
    {'id': 3, 'desc': 'Сыграйте в казино 3 раза', 'type': 'casino', 'target': 3, 'reward_money': 200, 'reward_exp': 80},
    {'id': 4, 'desc': 'Откройте 2 контейнера', 'type': 'container', 'target': 2, 'reward_money': 300, 'reward_exp': 100},
    {'id': 5, 'desc': 'Получите ежедневный бонус 2 раза', 'type': 'daily', 'target': 2, 'reward_money': 250, 'reward_exp': 90},
    {'id': 6, 'desc': 'Напишите 20 сообщений', 'type': 'message', 'target': 20, 'reward_money': 100, 'reward_exp': 50},
]

def find_item_by_id(items_dict, item_id):
    for name, price in items_dict.items():
        if f"[ID: {item_id}]" in name:
            return name, price
    return None, None

def get_user(user_id):
    cursor.execute('SELECT * FROM users WHERE user_id = ?', (user_id,))
    row = cursor.fetchone()
    if row is None:
        cursor.execute('INSERT INTO users (user_id) VALUES (?)', (user_id,))
        conn.commit()
        cursor.execute('UPDATE users SET registration_date = ? WHERE user_id = ?', (datetime.datetime.now().isoformat(), user_id))
        conn.commit()
        cursor.execute('SELECT * FROM users WHERE user_id = ?', (user_id,))
        row = cursor.fetchone()
    columns = [desc[0] for desc in cursor.description]
    user_dict = dict(zip(columns, row))
    if not user_dict.get('registration_date'):
        cursor.execute('UPDATE users SET registration_date = ? WHERE user_id = ?', (datetime.datetime.now().isoformat(), user_id))
        conn.commit()
        cursor.execute('SELECT * FROM users WHERE user_id = ?', (user_id,))
        row = cursor.fetchone()
        columns = [desc[0] for desc in cursor.description]
        user_dict = dict(zip(columns, row))
    return user_dict

def update_user(user_id, **kwargs):
    fields = ', '.join([f'{key} = ?' for key in kwargs.keys()])
    values = list(kwargs.values()) + [user_id]
    cursor.execute(f'UPDATE users SET {fields} WHERE user_id = ?', values)
    conn.commit()

def get_stats(user_id):
    cursor.execute('SELECT * FROM user_stats WHERE user_id = ?', (user_id,))
    row = cursor.fetchone()
    if row is None:
        cursor.execute('INSERT INTO user_stats (user_id) VALUES (?)', (user_id,))
        conn.commit()
        cursor.execute('SELECT * FROM user_stats WHERE user_id = ?', (user_id,))
        row = cursor.fetchone()
    columns = [desc[0] for desc in cursor.description]
    return dict(zip(columns, row))

def update_stats(user_id, text='', attachments=None):
    stats = get_stats(user_id)
    messages_count = stats['messages_count'] + 1
    mat_count = stats['mat_count']
    photo_count = stats['photo_count']
    voice_count = stats['voice_count']
    video_count = stats['video_count']
    lower_text = text.lower() if text else ''
    for word in MAT_WORDS:
        if word in lower_text:
            mat_count += 1
            break
    if attachments and isinstance(attachments, list):
        for att in attachments:
            if isinstance(att, dict):
                att_type = att.get('type', '')
                if att_type == 'photo': photo_count += 1
                elif att_type == 'voice': voice_count += 1
                elif att_type == 'video': video_count += 1
    cursor.execute('''UPDATE user_stats SET messages_count = ?, mat_count = ?, photo_count = ?, voice_count = ?, video_count = ?, last_message_time = ? WHERE user_id = ?''',
                   (messages_count, mat_count, photo_count, voice_count, video_count, datetime.datetime.now().isoformat(), user_id))
    conn.commit()
    update_quest_progress(user_id, 'message')

def update_quest_progress(user_id, quest_type, amount=1):
    for quest in QUESTS:
        if quest['type'] == quest_type:
            cursor.execute('SELECT progress, claimed FROM user_quests WHERE user_id = ? AND quest_id = ?', (user_id, quest['id']))
            row = cursor.fetchone()
            if row is None:
                cursor.execute('INSERT INTO user_quests (user_id, quest_id, progress, claimed) VALUES (?, ?, ?, 0)', (user_id, quest['id'], amount))
            else:
                if row[1] == 0:
                    cursor.execute('UPDATE user_quests SET progress = progress + ? WHERE user_id = ? AND quest_id = ?', (amount, user_id, quest['id']))
    conn.commit()

def is_muted(user_id):
    user = get_user(user_id)
    if user.get('mute_until'):
        mute_until = datetime.datetime.fromisoformat(user['mute_until'])
        if datetime.datetime.now() < mute_until: return True
    return False

def check_ban(user_id):
    cursor.execute('SELECT * FROM bans WHERE user_id = ?', (user_id,))
    return cursor.fetchone() is not None

def get_user_name(user_id):
    try:
        info = vk.users.get(user_ids=user_id)
        if info: return f"{info[0].get('first_name','')} {info[0].get('last_name','')}".strip()
    except: pass
    return f"id{user_id}"

def send_message(peer_id, text):
    try:
        vk.messages.send(peer_id=peer_id, message=text, random_id=get_random_id())
    except Exception as e:
        logger.error(f"Ошибка отправки: {e}")

def parse_amount(text):
    try: return float(text.strip())
    except: return None

def extract_id_from_mention(text):
    if not text: return None
    match = re.search(r'\[id(\d+)\|', text)
    if match: return int(match.group(1))
    match = re.search(r'@id(\d+)', text)
    if match: return int(match.group(1))
    if text.isdigit(): return int(text)
    return None

def get_last_seen(user_id):
    try:
        info = vk.users.get(user_ids=user_id, fields='last_seen')
        if info and info[0].get('last_seen'):
            return datetime.datetime.fromtimestamp(info[0]['last_seen']['time']).strftime('%d.%m.%Y %H:%M:%S')
        return 'скрыт'
    except: return 'ошибка'

def is_admin(user_id):
    user = get_user(user_id)
    return user_id == CREATOR_ID or user.get('role') in ['admin', 'owner', 'deputy']

def is_owner(user_id):
    return user_id == CREATOR_ID or get_user(user_id).get('role') in ['owner', 'deputy']

def is_moder(user_id):
    return get_user(user_id).get('role') in MODER_ROLES or is_admin(user_id)

def is_youtuber(user_id):
    user = get_user(user_id)
    return user.get('role') in ['youtuber', 'owner', 'deputy', 'admin', 'dev']

def check_credit(user_id):
    user = get_user(user_id)
    if user.get('credit_amount', 0) > 0 and user.get('credit_due'):
        due = datetime.datetime.fromisoformat(user['credit_due'])
        if datetime.datetime.now() > due:
            cursor.execute('INSERT OR REPLACE INTO bans (user_id, reason, timestamp) VALUES (?, ?, ?)', (user_id, 'Долг!', datetime.datetime.now().isoformat()))
            conn.commit()
            update_user(user_id, credit_amount=0, credit_due=None)
            return False
    return True
