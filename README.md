import asyncio
import os
from datetime import datetime, timedelta
import re
from aiogram import Bot, Dispatcher, F
from aiogram.filters import Command
from aiogram.types import (
    Message, InlineKeyboardMarkup, InlineKeyboardButton,
    LabeledPrice, PreCheckoutQuery, CallbackQuery
)

BOT_TOKEN = os.getenv('BOT_TOKEN', '')

TRIAL_DAYS = 3

PRICES = {'week': 10, 'twoweeks': 20, 'threeweeks': 30, 'month': 40}
DAYS_MAP = {'week': 7, 'twoweeks': 14, 'threeweeks': 21, 'month': 28}

# Хранилище в памяти (будет сбрасываться при перезапуске)
users_data = {}

bot = Bot(token=BOT_TOKEN)
dp = Dispatcher()

# ====================== МЕНЮ ======================

def main_menu():
    return InlineKeyboardMarkup(
        inline_keyboard=[
            [InlineKeyboardButton(text="❓ Как это работает", callback_data="how_it_works")],
            [InlineKeyboardButton(text="⚙️ Настроить копирование", callback_data="setup_copy")]
        ]
    )

def main_menu_expired():
    return InlineKeyboardMarkup(
        inline_keyboard=[
            [InlineKeyboardButton(text="⭐ Продолжить использование", callback_data="subscription")],
            [InlineKeyboardButton(text="👥 Партнёрская программа", callback_data="referral_program")]
        ]
    )

def setup_menu():
    return InlineKeyboardMarkup(
        inline_keyboard=[
            [InlineKeyboardButton(text="📢 Настроить канал", callback_data="set_channel")],
            [InlineKeyboardButton(text="🔼 Верхний блок", callback_data="set_block_top")],
            [InlineKeyboardButton(text="🔽 Нижний блок", callback_data="set_block_bottom")],
            [InlineKeyboardButton(text="🔙 Назад", callback_data="back_main")]
        ]
    )

def back_only_menu():
    return InlineKeyboardMarkup(
        inline_keyboard=[
            [InlineKeyboardButton(text="🔙 Назад", callback_data="back_setup")]
        ]
    )

def block_input_menu(block_type: str):
    if block_type == "top":
        return InlineKeyboardMarkup(
            inline_keyboard=[
                [InlineKeyboardButton(text="🔗 Вставить кликабельную ссылку", callback_data="add_link_top")],
                [InlineKeyboardButton(text="⏭️ Пропустить", callback_data="skip_block")],
                [InlineKeyboardButton(text="🔙 Назад", callback_data="back_setup")]
            ]
        )
    else:
        return InlineKeyboardMarkup(
            inline_keyboard=[
                [InlineKeyboardButton(text="🔗 Вставить кликабельную ссылку", callback_data="add_link_bottom")],
                [InlineKeyboardButton(text="⏭️ Пропустить", callback_data="skip_block")],
                [InlineKeyboardButton(text="🔙 Назад", callback_data="back_setup")]
            ]
        )

def link_input_menu():
    return InlineKeyboardMarkup(
        inline_keyboard=[
            [InlineKeyboardButton(text="🔙 Назад", callback_data="back_to_block_input")]
        ]
    )

# ====================== ВСПОМОГАТЕЛЬНЫЕ ФУНКЦИИ ======================

def get_user(user_id: int):
    if user_id not in users_data:
        users_data[user_id] = {}
    return users_data[user_id]

async def is_active(user_id: int) -> bool:
    user = get_user(user_id)
    now = datetime.utcnow()
    
    sub_end = user.get('sub_end')
    if sub_end and datetime.fromisoformat(sub_end) > now:
        return True
    
    trial_end = user.get('trial_end')
    if trial_end and datetime.fromisoformat(trial_end) > now:
        return True
    
    extra = user.get('extra_days', 0)
    if extra > 0:
        return True
    
    if 'trial_started' not in user:
        return True
    
    return False

async def is_expired(user_id: int) -> bool:
    user = get_user(user_id)
    now = datetime.utcnow()
    
    sub_end = user.get('sub_end')
    if sub_end and datetime.fromisoformat(sub_end) > now:
        return False
    
    trial_end = user.get('trial_end')
    if trial_end and datetime.fromisoformat(trial_end) > now:
        return False
    
    extra = user.get('extra_days', 0)
    if extra > 0:
        return False
    
    if 'trial_started' not in user:
        return False
    
    return True

async def start_trial(user_id: int):
    user = get_user(user_id)
    if 'trial_started' in user:
        return False
    
    now = datetime.utcnow()
    end = now + timedelta(days=TRIAL_DAYS)
    
    user['trial_started'] = now.isoformat()
    user['trial_end'] = end.isoformat()
    
    return True

async def get_referral_link(user_id: int) -> str:
    user = get_user(user_id)
    
    if 'ref_code' not in user:
        ref_code = f'ref{user_id % 1000000:06d}'
        user['ref_code'] = ref_code
        # Сохраняем обратную связь код -> user_id
        if 'ref_codes' not in users_data:
            users_data['ref_codes'] = {}
        users_data['ref_codes'][ref_code] = user_id
    
    ref_code = user['ref_code']
    
    try:
        bot_info = await bot.get_me()
        return f"https://t.me/{bot_info.username}?start={ref_code}"
    except:
        return f"https://t.me/bot?start={ref_code}"

async def get_user_settings(user_id: int):
    user = get_user(user_id)
    return (
        user.get('channel'),
        user.get('block_top', ''),
        user.get('block_bottom', '')
    )

def clean_text(text: str) -> str:
    if not text:
        return ""
    text = re.sub(r'#\w+', '', text)
    text = re.sub(r'https?://\S+|t\.me/\S+', '', text)
    text = re.sub(r'\n\s*\n', '\n', text.strip())
    return text.strip()

# ====================== КОМАНДЫ ======================

@dp.message(Command('start'))
async def cmd_start(message: Message):
    user_id = message.from_user.id
    user = get_user(user_id)
    
    is_new_user = 'first_seen' not in user
    
    if is_new_user:
        user['first_seen'] = datetime.utcnow().isoformat()
        user['username'] = message.from_user.username or 'нет'
    
    # Обработка реферальной ссылки
    ref_param = message.text.split()[-1] if len(message.text.split()) > 1 else None
    if ref_param and ref_param.startswith('ref'):
        referrer_code = ref_param[3:]
        ref_codes = users_data.get('ref_codes', {})
        
        if referrer_code in ref_codes:
            referrer_id = ref_codes[referrer_code]
            if referrer_id != user_id:
                if 'referred_by' not in user:
                    referrer = get_user(referrer_id)
                    referrer['extra_days'] = referrer.get('extra_days', 0) + 1
                    user['referred_by'] = referrer_id
                    
                    try:
                        await bot.send_message(
                            referrer_id,
                            f"🎉 По вашей ссылке зарегистрировался пользователь!\n"
                            f"➕ +1 день бесплатно!"
                        )
                    except:
                        pass
                    
                    await message.answer("✅ Вы приглашены по партнёрской ссылке!\n🔥 +1 день начислен пригласившему!")

    if await is_expired(user_id):
        ref_link = await get_referral_link(user_id)
        extra_days = user.get('extra_days', 0)
        
        await message.answer(
            f"⏰ <b>Время использования истекло</b>\n\n"
            f"Чтобы продолжить:\n"
            f"1️⃣ Продолжить использование звёздами\n"
            f"2️⃣ Пригласите друга → +1 день бесплатно\n\n"
            f"👥 <b>Ваша ссылка:</b>\n<code>{ref_link}</code>\n\n"
            f"📊 Бонусных дней: {extra_days}",
            parse_mode='HTML',
            reply_markup=main_menu_expired()
        )
        return

    trial_just_started = await start_trial(user_id)
    
    welcome_text = (
        "👋 <b>Привет!</b>\n\n"
        "Я помогу копировать посты из других каналов.\n"
        "Пересылай мне посты — я очищу их и добавлю твои блоки.\n\n"
    )
    
    if trial_just_started:
        welcome_text += f"🎁 <b>Вам доступен пробный период {TRIAL_DAYS} дня!</b>\n\n"
    
    welcome_text += "Нажми <b>«Настроить копирование»</b> чтобы начать!"
    
    await message.answer(welcome_text, parse_mode='HTML', reply_markup=main_menu())

@dp.callback_query(F.data == "how_it_works")
async def how_it_works(callback: CallbackQuery):
    text = (
        "❓ <b>Как это работает</b>\n\n"
        
        "🔹 <b>Шаг 1: Настройка</b>\n"
        "• Нажмите «Настроить копирование»\n"
        "• Укажите ваш канал (куда публиковать)\n"
        "• Настройте верхний и нижний блоки\n\n"
        
        "🔹 <b>Шаг 2: Копирование</b>\n"
        "• Перешлите мне любой пост из другого канала\n"
        "• Я автоматически обработаю и опубликую в ваш канал\n\n"
        
        "🧹 <b>Что убирает бот:</b>\n"
        "• Хэштеги (#новости, #важное)\n"
        "• Все ссылки (https://..., t.me/...)\n"
        "• Лишние пустые строки\n\n"
        
        "➕ <b>Что добавляет бот:</b>\n"
        "• Верхний блок — в начало поста\n"
        "• Нижний блок — в конец поста\n"
        "• Можно: текст, хэштеги, ссылки, эмодзи\n"
        "• <b>Кликабельные ссылки через кнопку 🔗</b>\n\n"
        
        "⚠️ <b>ВАЖНО:</b>\n"
        "• Бот должен быть администратором вашего канала\n"
        "• Добавьте бота в канал и дайте права на публикацию сообщений\n"
        "• Укажите @username канала или ID канала (начинается с -100)\n\n"
        
        "⚡ <b>Всё работает автоматически!</b>"
    )
    
    kb = InlineKeyboardMarkup(
        inline_keyboard=[
            [InlineKeyboardButton(text="⚙️ Настроить копирование", callback_data="setup_copy")],
            [InlineKeyboardButton(text="🔙 Назад", callback_data="back_main")]
        ]
    )
    
    await callback.message.edit_text(text, parse_mode='HTML', reply_markup=kb)
    await callback.answer()

@dp.callback_query(F.data == "back_main")
async def back_main(callback: CallbackQuery):
    user_id = callback.from_user.id
    
    if await is_expired(user_id):
        await callback.message.edit_text(
            "⏰ Время использования истекло.\n\nВыберите действие:",
            reply_markup=main_menu_expired()
        )
    else:
        await callback.message.edit_text(
            "Главное меню:\n\nНажмите «Настроить копирование» для настройки.",
            reply_markup=main_menu()
        )
    await callback.answer()

@dp.callback_query(F.data == "setup_copy")
async def setup_copy(callback: CallbackQuery):
    user_id = callback.from_user.id
    
    if await is_expired(user_id):
        await callback.answer("❌ Время использования истекло!", show_alert=True)
        ref_link = await get_referral_link(user_id)
        await callback.message.edit_text(
            f"⏰ <b>Время использования истекло</b>\n\n"
            f"👥 <b>Ваша ссылка:</b>\n<code>{ref_link}</code>",
            parse_mode='HTML',
            reply_markup=main_menu_expired()
        )
        return

    channel, block_top, block_bottom = await get_user_settings(user_id)
    
    status_text = "⚙️ <b>Настройка копирования</b>\n\n"
    status_text += f"📢 Канал: {channel if channel else 'не указан'}\n"
    status_text += f"🔼 Верхний блок: {'настроен' if block_top else 'пусто'}\n"
    status_text += f"🔽 Нижний блок: {'настроен' if block_bottom else 'пусто'}\n\n"
    status_text += "Выберите, что настроить:"
    
    await callback.message.edit_text(status_text, parse_mode='HTML', reply_markup=setup_menu())
    await callback.answer()

@dp.callback_query(F.data == "back_setup")
async def back_setup(callback: CallbackQuery):
    await setup_copy(callback)

@dp.callback_query(F.data == "skip_block")
async def skip_block(callback: CallbackQuery):
    user_id = callback.from_user.id
    user = get_user(user_id)
    state = user.get('state')
    
    if state in ['set_block_top', 'waiting_link_text_top', 'waiting_link_url_top']:
        user.pop('block_top', None)
        await callback.message.edit_text(
            "✅ Верхний блок пропущен.\n\nВыберите действие:",
            reply_markup=setup_menu()
        )
    elif state in ['set_block_bottom', 'waiting_link_text_bottom', 'waiting_link_url_bottom']:
        user.pop('block_bottom', None)
        await callback.message.edit_text(
            "✅ Нижний блок пропущен.\n\nВыберите действие:",
            reply_markup=setup_menu()
        )
    
    user.pop('state', None)
    user.pop('temp_link_text', None)
    await callback.answer()

@dp.callback_query(F.data == "back_to_block_input")
async def back_to_block_input(callback: CallbackQuery):
    user_id = callback.from_user.id
    user = get_user(user_id)
    current_block = user.get('current_block')
    
    if current_block == 'top':
        await set_block_top(callback)
    else:
        await set_block_bottom(callback)
    
    user.pop('temp_link_text', None)
    user.pop('current_block', None)

@dp.callback_query(F.data == "set_channel")
async def set_channel(callback: CallbackQuery):
    user_id = callback.from_user.id
    
    if await is_expired(user_id):
        await callback.answer("❌ Время использования истекло!", show_alert=True)
        return
    
    user = get_user(user_id)
    
    await callback.message.edit_text(
        "📢 <b>Настройка канала</b>\n\n"
        "⚠️ <b>ВАЖНО:</b> Бот должен быть администратором канала!\n\n"
        "1. Добавьте бота в ваш канал как администратора\n"
        "2. Дайте права: публикация сообщений\n"
        "3. Отправьте @username канала или ID\n\n"
        "Пример: <code>@mychannel</code>\n"
        "Или ID: <code>-1001234567890</code>",
        parse_mode='HTML',
        reply_markup=back_only_menu()
    )
    user['state'] = 'set_channel'
    await callback.answer()

@dp.callback_query(F.data == "set_block_top")
async def set_block_top(callback: CallbackQuery):
    user_id = callback.from_user.id
    
    if await is_expired(user_id):
        await callback.answer("❌ Время использования истекло!", show_alert=True)
        return
    
    user = get_user(user_id)
    current = user.get('block_top', '')
    
    text = (
        "🔼 <b>Верхний блок</b>\n\n"
        "Текст в начало каждого поста.\n\n"
        "Можно: обычный текст, хэштеги, эмодзи\n"
        "Или нажмите 🔗 для кликабельной ссылки\n\n"
    )
    
    if current:
        text += f"<b>Текущий:</b>\n<code>{current}</code>\n\n"
    
    text += "Отправьте текст:"
    
    await callback.message.edit_text(text, parse_mode='HTML', reply_markup=block_input_menu("top"))
    user['state'] = 'set_block_top'
    await callback.answer()

@dp.callback_query(F.data == "set_block_bottom")
async def set_block_bottom(callback: CallbackQuery):
    user_id = callback.from_user.id
    
    if await is_expired(user_id):
        await callback.answer("❌ Время использования истекло!", show_alert=True)
        return
    
    user = get_user(user_id)
    current = user.get('block_bottom', '')
    
    text = (
        "🔽 <b>Нижний блок</b>\n\n"
        "Текст в конец каждого поста.\n\n"
        "Можно: обычный текст, хэштеги, эмодзи\n"
        "Или нажмите 🔗 для кликабельной ссылки\n\n"
    )
    
    if current:
        text += f"<b>Текущий:</b>\n<code>{current}</code>\n\n"
    
    text += "Отправьте текст:"
    
    await callback.message.edit_text(text, parse_mode='HTML', reply_markup=block_input_menu("bottom"))
    user['state'] = 'set_block_bottom'
    await callback.answer()

@dp.callback_query(F.data == "add_link_top")
async def add_link_top(callback: CallbackQuery):
    user_id = callback.from_user.id
    user = get_user(user_id)
    
    await callback.message.edit_text(
        "🔗 <b>Кликабельная ссылка</b>\n\n"
        "Шаг 1/2: Отправьте текст ссылки\n"
        "(то, что будет видно в посте)\n\n"
        "Пример: <code>Подписаться на канал</code>",
        parse_mode='HTML',
        reply_markup=link_input_menu()
    )
    user['state'] = 'waiting_link_text_top'
    user['current_block'] = 'top'
    await callback.answer()

@dp.callback_query(F.data == "add_link_bottom")
async def add_link_bottom(callback: CallbackQuery):
    user_id = callback.from_user.id
    user = get_user(user_id)
    
    await callback.message.edit_text(
        "🔗 <b>Кликабельная ссылка</b>\n\n"
        "Шаг 1/2: Отправьте текст ссылки\n"
        "(то, что будет видно в посте)\n\n"
        "Пример: <code>Наш сайт</code> или <code>👉 Жми сюда</code>",
        parse_mode='HTML',
        reply_markup=link_input_menu()
    )
    user['state'] = 'waiting_link_text_bottom'
    user['current_block'] = 'bottom'
    await callback.answer()

@dp.message(F.text)
async def handle_input(message: Message):
    user_id = message.from_user.id
    user = get_user(user_id)
    state = user.get('state')
    
    if not state:
        return

    text = message.text.strip()

    if state == "set_channel":
        if text.startswith('@'):
            channel_id = text
        elif text.startswith('-100'):
            channel_id = text
        else:
            await message.answer(
                "❌ Неверный формат. Пример: @mychannel или -1001234567890",
                reply_markup=back_only_menu()
            )
            return
        
        try:
            bot_member = await bot.get_chat_member(chat_id=channel_id, user_id=(await bot.get_me()).id)
            
            if bot_member.status not in ['administrator', 'creator']:
                await message.answer(
                    "❌ Я не администратор в этом канале!\n\n"
                    "Добавьте меня в канал и дайте права администратора.",
                    parse_mode='HTML',
                    reply_markup=back_only_menu()
                )
                return
            
            chat = await bot.get_chat(channel_id)
            user['channel'] = str(chat.id)
            
            await message.answer(
                f"✅ Канал настроен: {chat.title}\n\n"
                f"Пересылайте посты для публикации!",
                reply_markup=main_menu()
            )
            
        except Exception as e:
            await message.answer(
                f"❌ Ошибка: {str(e)[:100]}\n\n"
                "Убедитесь, что бот добавлен в канал как администратор.",
                reply_markup=back_only_menu()
            )
            return

    elif state == "set_block_top":
        user['block_top'] = text
        preview = text[:200] + ('...' if len(text) > 200 else '')
        await message.answer(
            f"✅ Верхний блок сохранён!\n\n<code>{preview}</code>",
            parse_mode='HTML',
            reply_markup=setup_menu()
        )
        user.pop('state', None)

    elif state == "set_block_bottom":
        user['block_bottom'] = text
        preview = text[:200] + ('...' if len(text) > 200 else '')
        await message.answer(
            f"✅ Нижний блок сохранён!\n\n<code>{preview}</code>",
            parse_mode='HTML',
            reply_markup=setup_menu()
        )
        user.pop('state', None)

    elif state == "waiting_link_text_top":
        user['temp_link_text'] = text
        await message.answer(
            "🔗 <b>Шаг 2/2</b>\n\n"
            "Отправьте URL ссылки\n\n"
            "Примеры:\n"
            "<code>https://t.me/channel</code>\n"
            "<code>https://google.com</code>",
            parse_mode='HTML',
            reply_markup=link_input_menu()
        )
        user['state'] = 'waiting_link_url_top'

    elif state == "waiting_link_url_top":
        link_text = user.get('temp_link_text', '')
        link_url = text
        
        clickable_link = f'<a href="{link_url}">{link_text}</a>'
        user['block_top'] = clickable_link
        
        await message.answer(
            f"✅ Верхний блок сохранён!\n\n"
            f"🔗 <a href='{link_url}'>{link_text}</a>\n\n"
            f"Ссылка будет кликабельной в постах!",
            parse_mode='HTML',
            reply_markup=setup_menu()
        )
        user.pop('state', None)
        user.pop('temp_link_text', None)
        user.pop('current_block', None)

    elif state == "waiting_link_text_bottom":
        user['temp_link_text'] = text
        await message.answer(
            "🔗 <b>Шаг 2/2</b>\n\n"
            "Отправьте URL ссылки\n\n"
            "Примеры:\n"
            "<code>https://t.me/channel</code>\n"
            "<code>https://site.com</code>",
            parse_mode='HTML',
            reply_markup=link_input_menu()
        )
        user['state'] = 'waiting_link_url_bottom'

    elif state == "waiting_link_url_bottom":
        link_text = user.get('temp_link_text', '')
        link_url = text
        
        clickable_link = f'<a href="{link_url}">{link_text}</a>'
        user['block_bottom'] = clickable_link
        
        await message.answer(
            f"✅ Нижний блок сохранён!\n\n"
            f"🔗 <a href='{link_url}'>{link_text}</a>\n\n"
            f"Ссылка будет кликабельной в постах!",
            parse_mode='HTML',
            reply_markup=setup_menu()
        )
        user.pop('state', None)
        user.pop('temp_link_text', None)
        user.pop('current_block', None)

@dp.message(F.forward_origin)
async def handle_forward(message: Message):
    user_id = message.from_user.id
    
    if await is_expired(user_id):
        ref_link = await get_referral_link(user_id)
        await message.answer(
            f"⏰ <b>Время использования истекло</b>\n\n"
            f"Чтобы продолжить:\n"
            f"👥 <code>{ref_link}</code>\n\n"
            f"/start — для продолжения",
            parse_mode='HTML',
            reply_markup=main_menu_expired()
        )
        return
    
    if not await is_active(user_id):
        ref_link = await get_referral_link(user_id)
        await message.answer(
            f"⏰ <b>Время использования истекло</b>\n\n"
            f"👥 <code>{ref_link}</code>",
            parse_mode='HTML',
            reply_markup=main_menu_expired()
        )
        return

    channel, block_top, block_bottom = await get_user_settings(user_id)
    
    if not channel:
        await message.answer(
            "⚠️ Сначала настройте канал!\n\n"
            "Нажмите «Настроить копирование» → «Настроить канал»",
            reply_markup=main_menu()
        )
        return

    trial_just_started = await start_trial(user_id)
    if trial_just_started:
        await message.answer(
            f"🎁 <b>Пробный период активирован!</b>\n"
            f"У вас {TRIAL_DAYS} дня бесплатного использования.",
            parse_mode='HTML'
        )

    original_text = message.caption or message.text or ""
    clean = clean_text(original_text)

    new_text = ""
    if block_top:
        new_text += block_top + "\n\n"
    if clean:
        new_text += clean + "\n\n"
    elif not block_top and not block_bottom:
        new_text += "📎 Медиа\n\n"
    if block_bottom:
        new_text += block_bottom

    new_text = new_text.strip()

    try:
        await bot.copy_message(
            chat_id=channel,
            from_chat_id=message.chat.id,
            message_id=message.message_id,
            caption=new_text if new_text else None,
            parse_mode='HTML',
            protect_content=False
        )
        
        success_text = "✅ Пост опубликован!\n\n"
        if block_top:
            success_text += "🔼 Верхний блок добавлен\n"
        if block_bottom:
            success_text += "🔽 Нижний блок добавлен\n"
        success_text += "🧹 Очищено"
        
        await message.answer(success_text, parse_mode='HTML', reply_markup=main_menu())
        
    except Exception as e:
        await message.answer(
            f"❌ Ошибка: {str(e)[:100]}\n\n"
            "Убедитесь, что бот всё ещё администратор канала.",
            reply_markup=main_menu()
        )

@dp.callback_query(F.data == "referral_program")
async def show_referral(callback: CallbackQuery):
    user_id = callback.from_user.id
    ref_link = await get_referral_link(user_id)
    user = get_user(user_id)
    extra_days = user.get('extra_days', 0)
    
    kb = InlineKeyboardMarkup(
        inline_keyboard=[
            [InlineKeyboardButton(text="📤 Поделиться", url=f"https://t.me/share/url?url={ref_link}&text=Клонируй посты!")],
            [InlineKeyboardButton(text="🔙 Назад", callback_data="back_main")]
        ]
    )
    
    await callback.message.edit_text(
        f"👥 <b>Партнёрская программа</b>\n\n"
        f"Пригласи друга и получи <b>+1 день бесплатно</b> за каждого!\n\n"
        f"🔗 <code>{ref_link}</code>\n\n"
        f"📊 Бонусных дней: {extra_days}",
        parse_mode='HTML',
        reply_markup=kb
    )
    await callback.answer()

@dp.callback_query(F.data == "subscription")
async def subscription(callback: CallbackQuery):
    kb = InlineKeyboardMarkup(
        inline_keyboard=[
            [InlineKeyboardButton(text="⭐ 10 Stars — 1 неделя", callback_data="pay_week")],
            [InlineKeyboardButton(text="⭐ 20 Stars — 2 недели", callback_data="pay_twoweeks")],
            [InlineKeyboardButton(text="⭐ 30 Stars — 3 недели", callback_data="pay_threeweeks")],
            [InlineKeyboardButton(text="⭐ 40 Stars — 4 недели", callback_data="pay_month")],
            [InlineKeyboardButton(text="🔙 Назад", callback_data="back_main")]
        ]
    )
    
    await callback.message.edit_text(
        "💳 <b>Выберите тариф:</b>",
        parse_mode='HTML',
        reply_markup=kb
    )
    await callback.answer()

@dp.callback_query(F.data.startswith("pay_"))
async def process_payment(callback: CallbackQuery):
    period = callback.data[4:]
    
    if period not in PRICES:
        await callback.answer("Ошибка")
        return
    
    price = PRICES[period]
    days = DAYS_MAP[period]
    
    try:
        await callback.message.delete()
    except Exception as e:
        print(f"Не удалось удалить сообщение: {e}")
    
    try:
        await bot.send_invoice(
            chat_id=callback.from_user.id,
            title=f"Продолжение — {days} дней",
            description=f"Доступ к боту на {days} дней",
            payload=f"sub_{callback.from_user.id}_{period}",
            provider_token="",
            currency="XTR",
            prices=[LabeledPrice(label=f"{days} дней", amount=price)],
            need_name=False,
            need_phone_number=False,
            need_email=False,
            need_shipping_address=False,
            is_flexible=False,
            start_parameter=f"pay_{period}"
        )
    except Exception as e:
        print(f"Ошибка отправки инвойса: {e}")
        await callback.message.answer(
            "❌ Ошибка создания оплаты. Попробуйте снова.",
            reply_markup=main_menu_expired()
        )
    
    await callback.answer()

@dp.pre_checkout_query()
async def pre_checkout(pre: PreCheckoutQuery):
    await bot.answer_pre_checkout_query(pre.id, ok=True)

@dp.message(F.successful_payment)
async def got_payment(message: Message):
    payload = message.successful_payment.invoice_payload
    parts = payload.split('_')
    
    if len(parts) < 3 or parts[0] != 'sub':
        return
    
    user_id = int(parts[1])
    period = parts[2]
    
    if period not in DAYS_MAP:
        return
    
    days = DAYS_MAP[period]
    user = get_user(user_id)
    extra = user.get('extra_days', 0)
    total_days = days + extra
    
    if extra > 0:
        user['extra_days'] = 0
    
    now = datetime.utcnow()
    
    # Проверяем, есть ли активная подписка — продлеваем от неё
    current_sub_end = user.get('sub_end')
    if current_sub_end:
        current_end = datetime.fromisoformat(current_sub_end)
        if current_end > now:
            end_date = current_end + timedelta(days=total_days)
            is_extended = True
        else:
            end_date = now + timedelta(days=total_days)
            is_extended = False
    else:
        end_date = now + timedelta(days=total_days)
        is_extended = False
    
    user['sub_end'] = end_date.isoformat()
    user.pop('trial_end', None)
    user.pop('trial_started', None)
    user.pop('expiry_notified', None)
    
    # Уведомление о продлении/покупке
    if is_extended:
        await message.answer(
            f"✅ <b>Подписка продлена!</b>\n\n"
            f"📅 Новая дата окончания: {end_date.strftime('%d.%m.%Y')}\n"
            f"➕ Добавлено дней: {total_days}\n"
            f"⏳ Всего дней доступа: {(end_date - now).days}",
            reply_markup=main_menu()
        )
    else:
        await message.answer(
            f"✅ <b>Подписка активирована!</b>\n\n"
            f"📅 Доступ до: {end_date.strftime('%d.%m.%Y')}\n"
            f"⏳ Дней: {total_days}",
            reply_markup=main_menu()
        )

async def subscription_checker():
    """Фоновая задача для проверки подписок и отправки уведомлений"""
    while True:
        await asyncio.sleep(3600)  # Проверка каждый час
        
        now = datetime.utcnow()
        
        for user_id, user in list(users_data.items()):
            if not isinstance(user_id, int):
                continue
                
            try:
                # Проверка пробного периода
                trial_end = user.get('trial_end')
                if trial_end:
                    trial_end_dt = datetime.fromisoformat(trial_end)
                    
                    # Уведомление за 1 день до окончания
                    day_before = trial_end_dt - timedelta(days=1)
                    if 0 <= (now - day_before).total_seconds() <= 3600:
                        if not user.get('trial_reminder_sent'):
                            try:
                                await bot.send_message(
                                    user_id,
                                    f"⏰ <b>Пробный период закончится завтра!</b>\n\n"
                                    f"Чтобы продолжить использовать бота:\n"
                                    f"• Купите подписку звёздами\n"
                                    f"• Или пригласите друга\n\n"
                                    f"Нажмите /start для выбора тарифа",
                                    parse_mode='HTML'
                                )
                                user['trial_reminder_sent'] = True
                            except Exception as e:
                                print(f"Не удалось отправить напоминание {user_id}: {e}")
                    
                    # Уведомление об окончании
                    time_diff = (now - trial_end_dt).total_seconds()
                    if 0 <= time_diff <= 3600:
                        if user.get('expiry_notified'):
                            continue
                        
                        sub_end = user.get('sub_end')
                        if sub_end and datetime.fromisoformat(sub_end) > now:
                            continue
                        
                        extra = user.get('extra_days', 0)
                        if extra > 0:
                            continue
                        
                        user['expiry_notified'] = True
                        ref_link = await get_referral_link(user_id)
                        
                        try:
                            await bot.send_message(
                                user_id,
                                f"⏰ <b>Пробный период истёк</b>\n\n"
                                f"Чтобы продолжить копировать посты:\n"
                                f"1️⃣ Купите подписку звёздами\n"
                                f"2️⃣ Пригласите друга → +1 день бесплатно\n\n"
                                f"👥 <b>Ваша ссылка:</b>\n<code>{ref_link}</code>\n\n"
                                f"Нажмите /start для продолжения",
                                parse_mode='HTML',
                                reply_markup=main_menu_expired()
                            )
                            print(f"Уведомление об истечении отправлено user {user_id}")
                        except Exception as e:
                            print(f"Не удалось уведомить {user_id}: {e}")
                
                # Проверка платных подписок
                sub_end = user.get('sub_end')
                if sub_end:
                    sub_end_dt = datetime.fromisoformat(sub_end)
                    
                    # Уведомление за 1 день до окончания
                    day_before = sub_end_dt - timedelta(days=1)
                    if 0 <= (now - day_before).total_seconds() <= 3600:
                        if not user.get('sub_reminder_sent'):
                            ref_link = await get_referral_link(user_id)
                            try:
                                await bot.send_message(
                                    user_id,
                                    f"⏰ <b>Ваша подписка заканчивается завтра!</b>\n\n"
                                    f"📅 Дата окончания: {sub_end_dt.strftime('%d.%m.%Y')}\n\n"
                                    f"Чтобы продолжить:\n"
                                    f"1️⃣ Продлите подписку звёздами\n"
                                    f"2️⃣ Пригласите друга → +1 день бесплатно\n\n"
                                    f"👥 <b>Ваша ссылка:</b>\n<code>{ref_link}</code>\n\n"
                                    f"Нажмите /start для продления",
                                    parse_mode='HTML'
                                )
                                user['sub_reminder_sent'] = True
                                print(f"Напоминание о продлении отправлено user {user_id}")
                            except Exception as e:
                                print(f"Не удалось отправить напоминание {user_id}: {e}")
                    
                    # Уведомление об окончании подписки
                    time_diff = (now - sub_end_dt).total_seconds()
                    if 0 <= time_diff <= 3600:
                        if user.get('sub_expired_notified'):
                            continue
                        
                        extra = user.get('extra_days', 0)
                        if extra > 0:
                            continue
                        
                        user['sub_expired_notified'] = True
                        ref_link = await get_referral_link(user_id)
                        
                        try:
                            await bot.send_message(
                                user_id,
                                f"⏰ <b>Ваша подписка истекла</b>\n\n"
                                f"Чтобы продолжить копировать посты:\n"
                                f"1️⃣ Продлите подписку звёздами\n"
                                f"2️⃣ Пригласите друга → +1 день бесплатно\n\n"
                                f"👥 <b>Ваша ссылка:</b>\n<code>{ref_link}</code>\n\n"
                                f"Нажмите /start для продолжения",
                                parse_mode='HTML',
                                reply_markup=main_menu_expired()
                            )
                            print(f"Уведомление об окончании подписки отправлено user {user_id}")
                        except Exception as e:
                            print(f"Не удалось уведомить {user_id}: {e}")
                            
            except Exception as e:
                print(f"Ошибка проверки user {user_id}: {e}")

async def main():
    if not BOT_TOKEN:
        print("ERROR: BOT_TOKEN not set!")
        return
        
    await bot.delete_webhook(drop_pending_updates=True)
    
    # Запускаем фоновую задачу
    asyncio.create_task(subscription_checker())
    
    print("Бот запущен!")
    await dp.start_polling(bot)

if __name__ == '__main__':
    asyncio.run(main())
