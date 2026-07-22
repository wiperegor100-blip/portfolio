# SportSchool Bot

Telegram-бот для спортивной студии: посещаемость по групповому фото, абонементы, уведомления родителей, расписание, импорт и Excel-отчеты.

## Быстрый запуск на сервере

1. Установить Docker и Docker Compose.
2. Скопировать `.env.example` в `.env`.
3. Заполнить `TOKEN`, `SUPERADMIN_ID`, `STUDIO_NAME`.
4. Запустить:

```bash
docker compose up -d --build
```

5. Проверить логи:

```bash
docker compose logs -f bot
```

## Первый пилот

1. Суперадмин открывает бота и нажимает `/start`.
2. Суперадмин выдает `admin`-код руководителю.
3. Руководитель входит по коду и выдает `trainer`-код тренеру.
4. Руководитель или тренер создает группу.
5. Добавляются 3-5 учеников с фото.
6. Тренер отмечает посещаемость по групповому фото.
7. Проверяются списание, `undo`, Excel-выгрузка и уведомление родителя.

## Что обязательно сохранять

Все рабочие данные лежат в `./storage`, а контейнер монтирует эту папку целиком:

- `storage/face_db/app.db` - основная SQLite-база.
- `storage/auth_users.json` - авторизованные пользователи.
- `storage/invites.json` - коды приглашений.
- `storage/bot_persistence.pkl` - состояние диалогов Telegram.
- `storage/photos/` - временные фото тренировок.
- `storage/learning_log.json` - статистика распознавания.

Перед обновлением сервера делайте backup всей папки:

```bash
tar -czf sportschool-storage-backup.tgz storage
```

## Google Sheets

Интеграция необязательна. Если она нужна:

1. Положите `credentials.json` в `storage/credentials.json`.
2. Заполните `GOOGLE_SPREADSHEET_ID`.
3. Дайте service account доступ к таблице.

Если `GOOGLE_SPREADSHEET_ID` пустой, бот работает без Sheets.

## Проверка перед запуском

```bash
docker compose config
docker compose up -d --build
docker compose logs --tail=100 bot
```

В логах не должно быть ошибок импорта, токена Telegram или доступа к базе.

Дополнительная проверка внутри контейнера:

```bash
docker compose run --rm bot python scripts/preflight.py
```
