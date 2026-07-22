# VK Architecture Plan

Технический план запуска VK-направления для SportSchool Bot.

Документ отвечает на вопросы:

- как встроить VK без поломки Telegram-пилота
- какие модули добавить
- что нужно поменять в хранении auth и parent channels
- в каком порядке безопасно реализовывать VK


## Технический Вывод

VK нужно добавлять как отдельный adapter layer поверх существующего `core/`.

Правильная стратегия:

1. не трогать рабочий Telegram transport
2. не копировать бизнес-логику в `vk/`
3. вынести channel-specific вещи в новые адаптеры
4. осторожно расширить auth и parent delivery model


## Что Уже Подходит Для VK

Можно переиспользовать почти без изменения:

- `core/pending_training.py`
- `core/attendance.py`
- `core/notifications.py`
- `core/parent_link.py`
- `core/auth_access.py` как базовую идею ролей и кодов
- `db/repository.py`
- `recognition/*`
- `integrations/google_sheets.py`

Почему это хорошо:

- основная логика уже живёт вне Telegram SDK
- VK можно добавлять как ещё один transport adapter


## Что Сейчас Блокирует Прямой Запуск VK

### 1. Авторизация привязана к одному user id

Сейчас `core/auth_access.py` хранит пользователей в `storage/auth_users.json` по ключу:

- `str(user_id)`

Это работает для Telegram, но плохо для multi-channel, потому что:

- `telegram_user_id` и `vk_user_id` живут в разных пространствах идентификаторов
- один и тот же Parent или Admin может прийти из двух каналов
- нельзя надёжно понять, что это один и тот же человек


### 2. Уведомления родителям привязаны к Telegram

Сейчас `core/notifications.py` использует:

- `get_parent_telegram_ids(student_id)`

Это сразу ограничивает доставку Telegram-каналом.


### 3. Parent flow и menu flow транспортно смешаны с Telegram UI

В `bot/handlers/parent.py`, `bot/handlers/auth.py`, `bot/handlers/link_child.py` и `bot/handlers/menu.py` логика сценария уже полезная, но рендер и navigation полностью Telegram-specific.


### 4. Session handling завязано на Telegram persistence

Сейчас состояние Telegram-диалогов живёт в:

- `storage/bot_persistence.pkl`

VK нужно отдельное состояние диалогов и экранов.


## Целевая Архитектура

Рекомендуемая структура:

```text
bot/                    # текущий Telegram adapter
vk/                     # новый VK adapter
core/                   # общая бизнес-логика
db/                     # общие данные
integrations/           # внешние сервисы
storage/                # persistence
```

Новые модули:

```text
vk/
  __init__.py
  app.py
  notifier.py
  router.py
  session_store.py
  handlers/
    start.py
    parent.py
    lead.py
```


## Минимальные Архитектурные Изменения

### 1. Ввести понятие channel identity

Нужна новая модель идентичности пользователя.

Вместо логики:

- один пользователь = один `user_id`

Нужна логика:

- один пользовательский доступ = роль + org + channel + channel_user_id

Минимальный безопасный формат для JSON-хранилища:

```json
{
  "telegram:123456": {
    "role": "parent",
    "org_id": "default",
    "display_name": "..."
  },
  "vk:987654321": {
    "role": "parent",
    "org_id": "default",
    "display_name": "..."
  }
}
```

Это позволит:

- не ломать Telegram
- аккуратно добавить VK
- избежать коллизий id


### 2. Ввести parent delivery channels

Сейчас нужно хранить не только `parent_telegram_id`, но и дополнительные parent channels.

Самый безопасный путь:

- не ломать старое поле `parent_telegram_id`
- добавить новую сущность связи каналов родителя

Вариант хранения:

- новая таблица `parent_channels`

Поля:

- `id`
- `student_id`
- `channel`
- `channel_user_id`
- `created_at`
- `is_active`

Минимально достаточные значения:

- `channel = telegram | vk`
- `channel_user_id = string/int`

Почему это лучше, чем добавлять `parent_vk_id` в `students`:

- один родитель может быть привязан к нескольким детям
- один ребёнок может быть связан с несколькими родителями
- может быть несколько каналов доставки
- это лучше масштабируется под future web/app


### 3. Сделать channel-aware notifier routing

Сейчас `Notifier` хороший, но вызывающий код должен понимать, куда именно отправлять.

Нужен слой:

- получить все родительские каналы
- сгруппировать по transport
- вызвать нужный notifier

Вариант:

- оставить `core.notifier.Notifier`
- добавить orchestration в adapter layer

Например:

- `TelegramNotifier`
- `VKNotifier`

И вызывать их по каналам родителя.


### 4. Вынести parent use-cases из Telegram handlers в reusable core layer

Нужно не просто копировать `bot/handlers/parent.py`, а выделить reusable use-cases.

Новые модули в `core/`:

```text
core/parent_portal.py
core/lead_capture.py
```

В `core/parent_portal.py`:

- получить список детей родителя
- получить карточку ребёнка
- получить историю посещений
- получить расписание
- получить summary для home screen

Тогда:

- Telegram handler рендерит это в Telegram
- VK handler рендерит это в VK


### 5. Отдельный VK session store

Для VK MVP не нужен тяжёлый state machine как в Telegram.

Хватит простого session store:

- `vk:123 -> current_screen`
- `vk:123 -> pending_role`
- `vk:123 -> waiting_for_code`

Хранилище можно сделать сначала в JSON:

- `storage/vk_sessions.json`

Позже можно перенести в SQLite.


## Рекомендуемая Реализация По Этапам

### Этап 1. Подготовка Data Model

Сделать:

- channel-aware auth keys
- parent channel storage
- repository functions для получения parent channels

Не делать пока:

- trainer VK flow

Критерий готовности:

- Telegram продолжает работать без изменения сценариев
- можно привязать Parent через VK id


### Этап 2. VK notifier

Сделать:

- `vk/notifier.py`
- единый интерфейс отправки текстовых сообщений
- безопасное логирование ошибок доставки

Критерий готовности:

- можно отправить родителю тестовое сообщение в VK


### Этап 3. VK auth + child linking

Сделать:

- VK start screen
- ввод кода ребёнка
- привязка к `vk_user_id`
- выдача роли `parent`

Критерий готовности:

- родитель может открыть VK и сам получить доступ


### Этап 4. VK parent portal

Сделать:

- home screen родителя
- мои дети
- карточка ребёнка
- история посещений
- расписание
- помощь

Критерий готовности:

- родительский сценарий полностью живёт в VK без Telegram


### Этап 5. Lead flow

Сделать:

- роль лида или pre-auth visitor
- экран “что умеет продукт”
- заявка на демо
- запись события в event log или отдельное хранилище лидов

Критерий готовности:

- VK начинает работать как канал роста


## Что Не Нужно Делать На Первом Этапе

- не переписывать `core/pending_training.py`
- не переносить trainer recognition flow
- не пытаться сразу объединить Telegram и VK UI в один общий renderer
- не делать полноценный web backend ради VK MVP


## Нужные Новые Repository Functions

Минимальный набор:

- `link_parent_channel(student_id, channel, channel_user_id)`
- `get_parent_channels(student_id)`
- `is_parent_channel_linked(student_id, channel, channel_user_id)`
- `get_students_by_parent_channel(channel, channel_user_id)`

Желательно:

- `unlink_parent_channel(...)`
- `list_parent_channels(student_id)`


## Нужные Новые Core Functions

Минимально:

- `get_parent_dashboard(channel, channel_user_id)`
- `get_parent_children(channel, channel_user_id)`
- `get_parent_child_card(channel, channel_user_id, student_id)`
- `get_parent_attendance(channel, channel_user_id, student_id)`
- `get_parent_schedule(channel, channel_user_id)`

Для lead flow:

- `create_lead_request(...)`


## Риски Реализации

### Риск 1. Поломать Telegram auth

Защита:

- не заменять старый формат мгновенно
- сделать backward-compatible чтение auth


### Риск 2. Поломать parent notifications

Защита:

- сначала оставить Telegram route как есть
- VK delivery добавлять параллельно


### Риск 3. Дублировать сценарную логику

Защита:

- переносить use-cases в `core/`
- не копировать Telegram handlers целиком


### Риск 4. Перегрузить MVP

Защита:

- только parent + lead
- без trainer flow


## Самый Правильный Первый Coding Slice

Если начинать реализацию прямо после этого документа, лучший первый технический slice такой:

1. добавить channel-aware parent linking
2. добавить `get_students_by_parent_channel(...)`
3. выделить reusable parent portal use-cases в `core/`
4. затем уже писать `vk/` adapter

Это даст максимальную пользу с минимальным риском.
