# SportSchool Bot

Telegram-бот, который ведёт посещаемость детских студий по групповому фото после тренировки. Тренер фотографирует группу — бот распознаёт лиц, тренер подтверждает список, бот списывает занятия с абонемента и уведомляет родителей. Параллельно работает родительский кабинет: остаток занятий, история посещений, напоминания о тренировках.

## Language

### Roles

**Trainer**:
Сотрудник студии, который ведёт занятия и фотографирует группу для отметки посещаемости. Авторизуется через `ALLOWED_TRAINERS` в `.env`.
_Avoid_: coach, instructor, sensei, "user".

**Parent**:
Родитель ученика. Идентифицируется по `parent_telegram_id` в таблице `students`. Получает уведомления о списании занятий и напоминания.
_Avoid_: client, customer, account.

**Admin**:
Руководитель студии. Может редактировать шаблоны уведомлений, расписание, импортировать учеников. Подмножество **Trainer** с расширенными правами (`ADMIN_IDS`).
_Avoid_: owner, manager, supervisor.

**Student**:
Ребёнок, который ходит на занятия. У него есть **Subscription** и (опционально) **Parent**.
_Avoid_: child, kid, pupil, ребёнок, ученик.

### Studio organization

**Studio**:
Юридическое лицо — клиент продукта. Сейчас система single-tenant (одна студия на инсталляцию); название хранится в `STUDIO_NAME`.
_Avoid_: school, gym, club, организация.

**Group**:
Набор учеников, которых тренирует один **Trainer** в одно время по расписанию. Не путать с *group photo* — это просто фото с несколькими лицами.
_Avoid_: class, team, секция.

**Schedule**:
Регулярное расписание тренировок одной **Group**: дни недели + время + за сколько минут напоминать.
_Avoid_: timetable, calendar.

### Subscription & money

**Subscription**:
Привязанный к **Student** счёт занятий: `total_lessons` (куплено) и `used_lessons` (списано). Один Student → одна Subscription.
_Avoid_: pass, package, абонемент в смысле «бумажка».

**Lesson**:
Единица учёта в Subscription. Списывается, когда **Trainer** подтверждает посещение в **Confirmed Training**.
_Avoid_: class, session, занятие как событие в календаре (это **Training**).

**Topup**:
Операция увеличения `total_lessons` в Subscription. Делается тренером вручную (`/topup`); онлайн-оплата — будущая работа.
_Avoid_: payment, refill, recharge, пополнение баланса.

**Deduction**:
Атомарное списание одного **Lesson** (`used_lessons += 1`). Идемпотентно по студенту в рамках одного дня (см. правило ниже).
_Avoid_: charge, withdraw.

### Attendance flow

**Training**:
Один сеанс отметки посещаемости: тренер выбрал **Group**, прислал групповое фото, бот распознал лиц. Запись в таблице `trainings` со статусом `pending → confirmed | cancelled`. Это событие, **не** запись в расписании (расписание — **Schedule**).
_Avoid_: session, workout, AI training, тренировка модели.

**Recognition**:
Процесс «фото → список имён с confidence» через гибридный движок. Не имеет персистентного состояния — это чистая функция (вход: bytes + список Students; выход: список матчей).
_Avoid_: detection, identification, разпознавание лиц как отдельный модуль.

**Pending Training**:
**Training** в статусе `pending` — записи `attendance_records` уже созданы с `confirmed=False`, но ни один **Lesson** ещё не списан. Висит в `context.user_data` тренера до подтверждения или отмены.
_Avoid_: draft, candidate.

**Confirmed Training**:
**Training** в статусе `confirmed` — тренер нажал «✔️ Подтвердить», `attendance_records` помечены `confirmed=True`, **Lessons** списаны, родители уведомлены.
_Avoid_: completed, finished.

**Attendance Record**:
Одна строка в `attendance_records` — связка (Training, Student, confidence, confirmed). Запись существует и для pending, и для confirmed.
_Avoid_: visit, check-in, отметка.

**Undo**:
Откат последнего **Confirmed Training** того же тренера: возвращает **Lessons** в Subscriptions и переводит статус в `cancelled`. Не удаляет запись — оставляет аудит-след.
_Avoid_: cancel (это про отмену *pending*), revert, rollback.

### Face recognition stack

**Face Encoding**:
Числовой вектор лица (512 чисел для DeepFace, переменная длина для Eigenfaces). У Student хранится в трёх полях: `face_encoding` (основной), `face_encodings_extra` (доп. фото, до 5), `face_encoding_eigen` (для теневой модели).
_Avoid_: embedding (нормально, но в коде канон — encoding), descriptor.

**Primary Engine**:
DeepFace + Facenet512 — основной движок Recognition. Его результат идёт в UI и записывается в Attendance Records.
_Avoid_: main model.

**Shadow Engine**:
Eigenfaces — учится в фоне на каждом распознавании. Когда совпадает с Primary в `≥ 90%` случаев из `≥ 50` сессий — становится готов к замене Primary. Используется для будущей независимости от внешних весов.
_Avoid_: secondary, fallback.

**Learning Log**:
JSON-журнал сравнений Primary vs Shadow в `storage/learning_log.json`. Хранит последние 500 сессий и агрегированную точность.
_Avoid_: training log, AI log.

### Communication

**Notification**:
Telegram-сообщение **Parent**'у. Четыре типа: `lesson_deducted`, `low_balance`, `reminder`, `welcome`. Текст рендерится из **Notification Template**.
_Avoid_: message, alert, push.

**Notification Template**:
Текстовый шаблон одного типа Notification с переменными `{student_name}`, `{remaining}`, `{studio_name}`, `{group_name}`, `{time}`. Хранится в БД, редактируется руководителем.
_Avoid_: message template, шаблон сообщения (это в коде, но в домене — Notification Template).

**Reminder**:
Notification типа `reminder` — отправляется родителям группы за `remind_before_minutes` минут до тренировки по **Schedule**. Запускается job-ом в JobQueue каждую минуту.
_Avoid_: ping, alert.

## Relationships

- A **Studio** has many **Trainers**, **Groups** and **Students** _(пока неявно — single-tenant)_.
- A **Group** has many **Students**, one **Trainer** и один или несколько **Schedules**.
- A **Student** has exactly one **Subscription** и опционально одного **Parent**.
- A **Training** belongs to one **Group** и одному **Trainer**, имеет много **Attendance Records**.
- A **Confirmed Training** triggers exactly one **Deduction** per confirmed Student per day (idempotent).
- An **Undo** reverses exactly one **Confirmed Training** — последний этого тренера.
- A **Notification** is rendered from one **Notification Template** и адресована одному **Parent**.
- **Recognition** is stateless: `(photo, [Student]) → [Match{name, confidence}]`. Никаких побочных эффектов кроме записи в **Learning Log**.

## Example dialogue

> **Dev:** «Когда **Trainer** делает `/training` и присылает фото — это уже **Training** или ещё нет?»
> **Domain expert:** «Это **Pending Training**. **Attendance Records** созданы с `confirmed=False`, но **Deduction** не произошла. **Training** становится **Confirmed Training** только когда тренер нажал ✔️.»
> **Dev:** «А если он нажмёт ❌ — это тоже **Confirmed Training** просто пустой?»
> **Domain expert:** «Нет, статус — `cancelled`. **Confirmed Training** означает «зачёт», а `cancelled` означает «фото было ошибочное, забудь». **Undo** работает только над `confirmed`.»
> **Dev:** «А если у двух **Students** одинаковое имя?»
> **Domain expert:** «Сейчас сломается — **Recognition** возвращает имя, а не student_id. Нужно фиксить: матч должен возвращать **Student**, а не строку.»

## Flagged ambiguities

- **«Training»** в коде используется и для расписания (`Schedule`), и для события (`Training` table), и для AI («model training»). Резолюция: расписание — **Schedule**, событие — **Training**, обучение модели — **Learning Log session** (никогда не «training»).
- **«Confirm»** перегружено: тренер подтверждает **Pending Training** (это _confirmation_), родитель подтверждает биометрию (это _consent_). Резолюция: «consent» только для биометрии.
- **«Group»** — в коде «group» = и **Group** (класс учеников), и «group photo» (фото). Резолюция: **Group** — только класс. Для фото — «training photo» или «attendance photo».
- **«Cancel»** — у **Pending Training** есть кнопка ❌ Cancel (статус → `cancelled`), у **Confirmed Training** есть `/undo` (тоже статус → `cancelled`). Это две разные операции с одинаковым итоговым статусом. Резолюция: Cancel = до **Deduction**; Undo = после **Deduction**.
- **«Лесн / занятие / training»** в RU-копи смешано. Резолюция: в копи везде «занятие» = **Lesson** (то что списывается); «тренировка» = **Training** (событие). Никогда не «занятие» в смысле события.
