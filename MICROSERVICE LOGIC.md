# CompetitionEngine

## Описание работы

1. **Получение входных данных**

   * Принимает от CompetitionService параметры турнира:

     * Тип системы (олимпийская, швейцарская, круговая)
     * Даты и время турнира
     * Список участников/команд
     * Конфигурация площадок (кол-во, доступность по времени)
     * Настройки тайм-слотов, перерывов

2. **Выбор алгоритма**

   * **Олимпийская система (single-elimination)**
     Строит дерево плей-офф: пары по сеяным, по раундам.
   * **Швейцарская система**
     Пары по очкам на каждом раунде, избегая повторных встреч.
   * **Круговая система (round-robin)**
     Каждая пара встречается ровно раз, циклический сдвиг.

3. **Генерация расписания**

   * Формирует все возможные тайм-слоты для площадок
   * Распределяет матчи по слотам и раундам, учитывая перерывы и ограничения
   * Не допускает игр подряд для одного участника, даёт паузы, позволяет задавать приоритетные матчи

4. **Выдача результата**

   * По завершении генерации или расчёта отдает результат по taskId

## Эндпоинты

### 1. POST /engine/schedule

* **Назначение:** Запустить процесс генерации расписания для турнира
* **Когда:** После создания/редактирования турнира, изменения участников или дат
* **Тело запроса:**

  * competitionId
  * тип системы, даты, участники, площадки, настройки тайминга
* **Ответ:**

  ```json
  { "taskId": "schedule-task-xyz" }
  ```

### 2. GET /engine/schedule/{taskId}

* **Назначение:** Проверить статус и получить расписание матчей
* **Ответ:**

  ```json
  {
    "status": "READY",
    "matches": [
      {
        "id": "m1",
        "round": 1,
        "playerA": "team_1",
        "playerB": "team_2",
        "start": "2025-07-15T10:00:00",
        "end": "2025-07-15T11:00:00",
        "venue": "Court A"
      }
    ]
  }
  ```
* **Если в процессе:** `status: "IN_PROGRESS"`

### 3. POST /engine/calculate

* **Назначение:** Запустить процесс расчёта итогов по завершению матча/раунда/турнира
* **Тело запроса:** competitionId, список матчей с результатами, флаг финального раунда
* **Ответ:**

  ```json
  { "taskId": "calc-task-abc" }
  ```

### 4. GET /engine/calculate/{taskId}

* **Назначение:** Получить результаты расчёта (выигравшие, пары на следующий этап и т.д.)
* **Ответ:**

  ```json
  {
    "status": "READY",
    "results": [
      {
        "matchId": "m1",
        "playerA": "team_1",
        "playerB": "team_2",
        "scoreA": 2,
        "scoreB": 1,
        "winner": "team_1"
      }
    ],
    "nextRoundPairs": [
      {
        "playerA": "team_3",
        "playerB": "team_7"
      }
    ]
  }
  ```

### 5. GET /engine/health (опционально)

* **Назначение:** Проверка работоспособности (healthcheck)
* **Ответ:**

  ```json
  { "status": "ok", "uptime": 142000 }
  ```

## Реально-временная логика

* Все процессы асинхронны: сначала POST-запрос с данными — получаем taskId, затем GET-получение статуса и результата по этому taskId.
* CompetitionEngine работает только с CompetitionService (не с клиентом напрямую).
* Для устойчивой работы можно поднимать сервис как отдельный контейнер, с поддержкой healthcheck для Docker/Kubernetes.

---

# StatisticsService

## Описание работы

1. **Приём и обработка результатов**

   * Получает результаты матчей по POST /stats/update после завершения очередного матча или раунда
   * Парсит, сохраняет в базу и кэширует

2. **Подсчёт статистики**

   * Актуализирует лидерборды, счета, рейтинги, индивидуальную и командную статистику
   * Для крупных турниров заранее рассчитывает precomputed-views

3. **Актуализация данных**

   * Чаще всего Redis используется как быстрый кэш для популярных данных (развернуть отдельным контейнером)
   * Если данных нет — достаёт из базы (PostgreSQL/MongoDB)

4. **Интеграция событий (Kafka)**

   * Подписан на Kafka-топик ResultsCalculated (Kafka можно поднять как отдельный контейнер)
   * При новых событиях синхронизирует данные

5. **Live-уведомления**

   * Поддержка SSE/WebSocket для мгновенного обновления интерфейса клиентов

## Эндпоинты

* **POST /stats/update** — приём результатов матчей
* **GET /stats/competition/{competitionId}** — агрегированные данные турнира
* **GET /stats/player/{playerId}** — статистика игрока
* **GET /stats/team/{teamId}** — статистика команды
* **GET /stats/leaderboard** — глобальный рейтинг

## Реально-временная логика

1. Подписка на Kafka-топик ResultsCalculated, обновление кэша и базы
2. Счётчики в Redis для быстрого ответа на частые запросы
3. Precomputed views для популярных турниров
4. Live-рассылка обновлений через SSE/WebSocket всем подписанным клиентам

## Пример ответа

```json
{
  "competitionId": "c123",
  "leaderboard": [
    { "teamId": "t1", "points": 18, "wins": 6, "draws": 0, "losses": 0 }
  ],
  "roundStats": [
    { "round": 1, "topScorer": "player_9", "goals": 4 }
  ]
}
```

---

# FeedbackService

## Описание работы

1. **Приём отзывов**

   * POST /feedback/competitions/{competitionId} — сохраняет отзыв в базу и кэширует в Redis

2. **Модерация и редактирование**

   * Статус pending — модератор может одобрить/отклонить/отредактировать через PUT/DELETE
   * Для автоматизации модерации можно подключить модуль анализа текста

3. **Уведомления администраторам**

   * При критичных отзывах публикует событие NewFeedback в Kafka или уведомляет через NotificationService

4. **Live-обновления**

   * SSE/WebSocket для обновления UI в режиме реального времени (например, в админке)

## Эндпоинты

* **POST /feedback/competitions/{competitionId}** — отправка нового отзыва
* **GET /feedback/competitions/{competitionId}** — список отзывов по турниру
* **GET /feedback/{feedbackId}** — детали отзыва
* **PUT /feedback/{feedbackId}** — редактирование
* **DELETE /feedback/{feedbackId}** — удаление

## Реально-временная логика

1. Одновременная запись в базу и Redis
2. События NewFeedback через Kafka/NotificationService для модераторов
3. Workflow модерации — статусы pending/approved/rejected
4. Анализ отзывов на токсичность
5. Live-обновления для UI через SSE/WebSocket

---

# CompetitionService

## Описание работы

1. **Создание турнира**

   * POST /competitions — приём и сохранение параметров турнира, создание записи в CompetitionDB

2. **Управление регистрацией**

   * Открывает/закрывает регистрацию, проверяет лимиты, дедлайны, уникальность
   * POST/DELETE на /competitions/{competitionId}/registrations

3. **Дедлайны и автоматизация**

   * Планировщик (Quartz/cron) автоматически закрывает регистрацию и запускает расчёты, когда наступает нужное время

4. **Взаимодействие с другими сервисами**

   * Запускает генерацию расписания через CompetitionEngine (POST /engine/schedule)
   * После генерации/расчёта публикует события через Kafka

5. **Статус турнира и live-обновления**

   * Рассылает статусные обновления через SSE/WebSocket

6. **Технологии**

   * Отдельный сервис, планировщик задач, Kafka (docker-compose), NotificationService

## Эндпоинты

* **GET /competitions** — каталог турниров
* **POST /competitions** — создание
* **GET /competitions/{competitionId}** — детали
* **PUT /competitions/{competitionId}** — редактирование
* **DELETE /competitions/{competitionId}** — удаление
* **GET /competitions/{competitionId}/registrations** — список регистраций
* **POST /competitions/{competitionId}/registrations** — регистрация участника
* **DELETE /competitions/{competitionId}/registrations/{registrationId}** — отмена регистрации
* **POST /competitions/{competitionId}/finish** — закрытие регистрации, запуск расчётов

## Реально-временная логика

1. Планировщик по времени вызывает закрытие регистрации и старт расчёта расписания
2. События через Kafka для синхронизации движка и статистики
3. Очередь расчётов (RabbitMQ/Kafka)
4. Webhook от CompetitionEngine меняет статус «расписание готово»
5. Push/SSE/WebSocket уведомления клиентам и админам

---

# ChatService

## Описание работы

1. **Создание и настройка чатов**

   * POST /chats — создание комнат для турниров и команд
   * Каждый чат имеет ID, список участников, права (модератор/пользователь/гость)

2. **Управление участниками**

   * Приглашение (POST /chats/{chatId}/members), исключение (DELETE), мут/размут

3. **Передача сообщений**

   * История хранится в базе и кэшируется в Redis
   * POST /chats/{chatId}/messages — рассылка по WebSocket всем в комнате

4. **Live-логика**

   * WebSocket-сервер (Spring, ws, socket.io), рассылка новых сообщений
   * Горячая модерация — сообщения и муты применяются мгновенно

5. **Пуш-уведомления**

   * Kafka или NotificationService для отправки push, если пользователь не онлайн

6. **Поиск и история**

   * Elasticsearch для поиска по тексту сообщений, пагинация

## Эндпоинты

* **GET /chats** — список чатов
* **POST /chats** — создать чат
* **GET /chats/{chatId}/members** — участники чата
* **POST /chats/{chatId}/members** — пригласить участника
* **DELETE /chats/{chatId}/members/{userId}** — исключить участника
* **GET /chats/{chatId}/messages** — история сообщений
* **POST /chats/{chatId}/messages** — отправить сообщение
* **DELETE /chats/{chatId}/messages/{messageId}** — удалить сообщение
* **POST /chats/{chatId}/mute/{userId}** — замутить участника
* **POST /chats/{chatId}/unmute/{userId}** — размутить участника

## Реально-временная логика

1. WebSocket-сервер внутри сервиса
2. Мгновенная модерация и рассылка сообщений
3. Кэширование истории сообщений в Redis
4. Push-уведомления через Kafka/NotificationService
5. Поиск и пагинация (Elasticsearch)

---

# UserService

## Описание работы

1. **Регистрация и аутентификация**

   * POST /register — создание пользователя, хеширование пароля, сохранение в UserDB
   * POST /login — вход, выдача HS256 JWT (access/refresh)

2. **Управление токенами**

   * Access JWT (15–30 мин), refresh JWT (7–30 дней), refresh хранится с уникальным jti в Redis
   * Все сервисы используют общий секрет (можно хранить в Vault/env)

3. **Валидация токенов**

   * Проверка подписи и срока действия в каждом микросервисе
   * Проверка ролей/scopes

4. **Изменение профиля и пароля**

   * PUT /users/{userId} — изменение профиля
   * PATCH /users/{userId}/password — смена пароля, отзыв сессий

5. **Сброс пароля**

   * POST /users/{userId}/password-reset-request — генерация токена сброса, отправка через NotificationService
   * PUT /users/{userId}/password-reset — сброс по токену

6. **Безопасность**

   * Лимит неудачных попыток входа, временная блокировка
   * Логирование событий

7. **Где разворачивать**

   * UserService и Redis как отдельные контейнеры, NotificationService — отдельный сервис

## Эндпоинты

* **POST /register** — регистрация
* **POST /login** — вход, выдача JWT
* **POST /refresh-token** — обновление access-токена
* **GET /users/{userId}** — профиль
* **PUT /users/{userId}** — изменить профиль
* **PATCH /users/{userId}/password** — сменить пароль
* **POST /users/{userId}/password-reset-request** — запрос сброса пароля
* **PUT /users/{userId}/password-reset** — сброс пароля

## Реально-временная логика

1. Управление и отзыв токенов через Redis
2. Проверка подписи/exp/scopes в каждом микросервисе
3. Лимиты на неудачные попытки, временная блокировка
4. Сброс пароля — токены и NotificationService
5. Аудит и логирование

---

# iOS App

## Описание работы

1. **Регистрация и вход**

   * POST /register, /login, /refresh-token, хранение JWT в Keychain
   * При истечении access-token — автоматическое обновление

2. **Работа с профилем**

   * GET/PUT/PATCH /users — загрузка/изменение данных, смена пароля, сброс через email/SMS

3. **Соревнования**

   * GET/POST/DELETE /competitions — просмотр турниров, регистрация, отмена

4. **Чаты**

   * GET/POST/DELETE /chats — подключение к комнатам, отправка сообщений, live-обновления через WebSocket

5. **Статистика**

   * GET /stats — загрузка таблицы лидеров, персональных данных, глобального рейтинга

6. **Обратная связь**

   * GET/POST/PUT/DELETE /feedback — работа с отзывами

7. **Live-логика, push, offline**

   * WebSocket/SSE подписки на обновления, push через APNs, кэширование данных через CoreData/Realm, оптимистичный UI (результат сразу отображается, потом подтверждается сервером)

## Эндпоинты

### Аутентификация и профиль

* POST /register
* POST /login
* POST /refresh-token
* GET /users/{userId}
* PUT /users/{userId}
* PATCH /users/{userId}/password
* POST /users/{userId}/password-reset-request
* PUT /users/{userId}/password-reset

### Соревнования

* GET /competitions
* GET /competitions/{competitionId}
* POST /competitions/{competitionId}/registrations
* GET /competitions/{competitionId}/registrations
* DELETE /competitions/{competitionId}/registrations/{registrationId}

### Чат

* GET /chats
* GET /chats/{chatId}/members
* GET /chats/{chatId}/messages
* POST /chats/{chatId}/messages
* DELETE /chats/{chatId}/messages/{messageId}

### Статистика

* GET /stats/competition/{competitionId}
* GET /stats/player/{playerId}
* GET /stats/leaderboard

### Обратная связь

* GET /feedback/competitions/{competitionId}
* POST /feedback/competitions/{competitionId}
* PUT /feedback/{feedbackId}
* DELETE /feedback/{feedbackId}

## Реально-временная логика

1. JWT хранится в Keychain, автоматическое обновление токена при необходимости
2. Кэш и offline через CoreData/Realm
3. Live-обновления через WebSocket/SSE
4. Push через APNs и NotificationService
5. UI — сразу отклик на действие, потом синхронизация с сервером
6. Переходы между экранами через Coordinators/Router

---