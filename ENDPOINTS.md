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
* **GET /chats/{chatId}/members** — загружает список участников выбранной комнаты.
* **POST /chats/{chatId}/members** — приглашает пользователя в комнату по его ID.
* **DELETE /chats/{chatId}/members/{userId}** — удаляет участника из комнаты (модерация).
* **GET /chats/{chatId}/messages** — загружает историю сообщений при открытии чата.
* **POST /chats/{chatId}/messages** — отправляет новое сообщение (текст, эмодзи, файлы).
* **DELETE /chats/{chatId}/messages/{messageId}** — удаляет сообщение по просьбе модератора.
* **POST /chats/{chatId}/mute/{userId}** — временно запрещает пользователю отправлять сообщения.
* **POST /chats/{chatId}/unmute/{userId}** — снимает запрет на отправку сообщений.

## CompetitionService

* **GET /competitions** — список всех соревнований для каталога.
* **POST /competitions** — создаёт новое соревнование в админ-интерфейсе или приложении.
* **GET /competitions/{competitionId}** — детали соревнования: описание, даты, правила.
* **PUT /competitions/{competitionId}** — обновляет настройки соревнования (название, даты, правила).
* **DELETE /competitions/{competitionId}** — удаляет соревнование (например, при отмене).
* **GET /competitions/{competitionId}/registrations** — показывает зарегистрированных участников и команды.
* **POST /competitions/{competitionId}/registrations** — регистрирует участника или команду.
* **DELETE /competitions/{competitionId}/registrations/{registrationId}** — отменяет регистрацию.
* **POST /competitions/{competitionId}/finish** — отмечает соревнование завершённым, инициирует расчёт результатов.

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
