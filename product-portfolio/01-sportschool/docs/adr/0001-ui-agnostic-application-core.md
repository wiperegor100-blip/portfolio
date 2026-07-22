# Application core остаётся UI-agnostic — Telegram это один adapter

`core/` не импортирует `telegram`. Все каналы доставки Notification (Telegram сегодня, web-push и email завтра) реализуются как адаптеры интерфейса `Notifier`. Это решение принято потому что у нас уже два запланированных адаптера (Telegram + будущий мобайл/веб), что делает seam реальным, а не гипотетическим.

## Considered Options

- **Прямые вызовы `bot.send_message` из `core/`** — удобно сейчас, но когда FastAPI-роут или крон-задача захочет уведомить родителя, у них не будет `telegram.Bot`. Пришлось бы дублировать логику или тащить PTB в API-слой.
- **Абстрактный `Notifier` с адаптерами** — выбрано. `core/` принимает `Notifier`, не знает про канал. `TelegramNotifier` живёт в `bot/`. Будущий `PushNotifier` живёт в `api/`.

## Consequences

- Тесты на `core/notifications` не требуют мока `telegram.Bot` — достаточно `FakeNotifier`.
- Добавление нового канала (email, push) = новый адаптер, `core/` не трогается.
