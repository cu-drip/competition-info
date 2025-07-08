# Описание использования запросов микросервисов

## User Service

-- user 
* **GET /api/v1/users** — получить всех пользователей.  
* **POST /api/v1/users** — создать пользователя.  
* **GET /api/v1/users/{id}** — получить пользователя по ID.  
* **PATCH /api/v1/users/{id}** — обновить пользователя.  
* **DELETE /api/v1/users/{id}** — удалить пользователя.  

-- auth
* **POST /api/v1/auth/register** — регистрация пользователя.  
* **POST /api/v1/auth/login** — вход пользователя.  
* **GET /api/v1/auth/me** — получить свой профиль.  
* **PATCH /api/v1/auth/me** — обновить свой профиль.  
* **PATCH /api/v1/auth/change-password** — сменить пароль (авторизованный пользователь).  
* **POST /api/v1/auth/password-reset-request** — запрос на сброс пароля (забыли пароль).  
* **POST /api/v1/auth/password-reset** — сброс пароля по токену (забыли пароль).  


## ChatService

* **GET /chats** — показывает список комнат чата на экране «Чаты».
* **POST /chats** — создаёт новую комнату (например, чат для команды или соревнования).
* **GET /chats/{chatId}** — информация о чате.
* **GET /chats/{chatId}/users** — загружает список участников выбранной комнаты.
* **POST /chats/{chatId}/users** — приглашает пользователей в комнату по их ID.
* **DELETE /chats/{chatId}/users/{userId}** — удаляет участника из комнаты.
* **PATCH /chats/{chatId}/users/{userId}** — mute/unmute пользователя.
* **GET /chats/{chatId}/messages?limit=100&after=123** — загружает историю сообщений при открытии чата.
* **WS /ws/chats/{chatId}** — подключится к сокету чата

## CompetitionService

* **GET /api/v1/tournaments** — получить список турниров.
* **POST /api/v1/tournaments** — создать новый турнир (только для админов).
* **GET /api/v1/tournaments/{id}** — получить турнир по ID.
* **PUT /api/v1/tournaments/{id}** — полностью обновить турнир (только для админов).
* **PATCH /api/v1/tournaments/{id}** — частично обновить турнир (только для админов).
* **DELETE /api/v1/tournaments/{id}** — удалить турнир (только для админов).
* **GET /api/v1/tournaments/{id}/participants** — получить всех участников турнира.
* **POST /api/v1/tournaments/{id}/register** — зарегистрировать участника или команду.
* **DELETE /api/v1/tournaments/{id}/register** — отменить регистрацию участника или команды.
* **POST /api/v1/tournaments/{id}/notify** — отправить уведомления участникам или командам (только для админов).

### TeamService (Реализован в паре CompetitionService)

* **GET /api/v1/teams** — получить список команд.
* **POST /api/v1/teams** — создать команду (только для админов).
* **GET /api/v1/teams/{id}** — получить команду по ID.
* **PUT /api/v1/teams/{id}** — Обновить команду целиком (только для админов)
* **PATCH /api/v1/teams/{id}** — Обновить команду частично (только для админов)
* **DELETE /api/v1/teams/{id}** — Удалить команду (только для админов)
* **GET /api/v1/teams/{id}/participants** — Получить всех участников команды
* **POST /api/v1/teams/{teamId}/participants/{userId}** — Добавить нового участника в команду
* **DELETE /api/v1/teams/{teamId}/participants/{userId}** — Удалить участника из команды

## CompetitionEngine

* **POST /engine/schedule** — запускает задачу генерации расписания игр после создания или изменения соревнования.
* **GET /engine/schedule/{taskId}** — проверяет статус и результат генерации расписания.
* **POST /engine/calculate** — запускает расчёт результатов матчей по итогам проведения игр.
* **GET /engine/calculate/{taskId}** — проверяет статус вычислений и получает итоговые данные.

## StatisticsService

* **POST /stats/update** — загрузка результатов матчей по турниру.
* **GET /stats/player/{userId}** — возвращает индивидуальную статистику игрока.
* **GET /stats/team/{teamId}** — возвращает статистику команды.
* **GET /stats/top/players** — возвращает топ-5 игроков по очкам.
* **GET /stats/matches/history** — возвращает историю матчей.

## FeedbackService

* **POST /feedback/competitions/{competitionId}** — создаёт отзыв о соревновании из формы обратной связи.
* **GET /feedback/competitions/{competitionId}** — показывает список отзывов для соревнования.
* **GET /feedback/{feedbackId}** — получает данные конкретного отзыва.
* **PUT /feedback/{feedbackId}** — редактирует текст отзыва (по инициативе автора или модератора).
* **DELETE /feedback/{feedbackId}** — удаляет отзыв (автор или админ).
