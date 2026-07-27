# Ревью Task Tracker
Проект: https://gitlab.com/mmeow/task-tracker

Многосервисный тасктрекер на Go: backend REST API, статический фронтенд на Nginx, планировщик на gRPC с cron, email-sender с воркер-пулом. PostgreSQL, миграции, Kafka как шина событий, bcrypt, JWT. Разбит по слоям (controllers, services, repositories, domain, dto). Есть e2e на Playwright, интеграционные тесты, GitLab CI с публикацией образов. Проект собирается, тесты запускаются через `make compose-tests`.

<img src="" width="400"/>

---

## СООТВЕТСТВИЕ ТЗ

- ✅ Четыре сервиса: `task-tracker-backend`, `task-tracker-frontend`, `task-tracker-scheduler`, `task-tracker-email-sender`. Плюс Postgres и Kafka в композе.
- ✅ Регистрация `POST /api/user`, вход `POST /api/auth/login`, текущий пользователь `GET /api/user`, задачи `GET /api/tasks`. JWT в заголовке `Authorization: Bearer ...`.
- ✅ JWT валидируется middleware, `user_id` кладётся в контекст запроса, контроллеры достают его через `middlewares.UserIDFromContext`.
- ✅ Bcrypt для паролей, уникальный email в БД, ошибка 409 `ErrUserAlreadyExists` при коллизии.
- ✅ Формат ошибок `{"message": "..."}` через централизованный `httputils.HandleError`, маппинг доменных ошибок в HTTP-статусы.
- ✅ Миграции в `db/migrations`, таблица `users` с уникальным индексом на email, таблица `tasks` с FK на users и CHECK на статус.
- ✅ Kafka-топик `EMAIL_SENDING_TASKS`, продюсер в backend/scheduler, консьюмер в email-sender.
- ✅ Welcome-email отправляется через Kafka в `AuthService.SignUp`.
- ✅ Email-sender с воркер-пулом (буферизованный канал + 3 горутины).
- ✅ Scheduler со своим gRPC API (`RegisterJob`/`RunJob`/`GetJobStatus`) и cron (`robfig/cron/v3`), публикует событие `notification.started` в топик `NOTIFICATION_EVENTS`.
- ✅ Backend подписан на `NOTIFICATION_EVENTS`, `NotificationReportService` собирает отчёт по каждому пользователю (незавершённые + завершённые за 24 часа) и публикует email-задания.
- ✅ Ограничение 5 задач в превью письма, три ветки текста (обе категории, только незавершённые, только завершённые).
- ✅ Docker Compose со всеми сервисами, healthcheck на Postgres и Kafka.
- ✅ CI: GitLab pipeline (build, test, publish), coverage-бэйдж, публикация образов в GitLab Container Registry.
- ✅ Тесты: unit-контроллеры, интеграционные, Kafka publisher, notification report, e2e через Playwright.
- ⚠ Схема БД в ТЗ описана как `status boolean` + `completed_timestamp`. У тебя `status varchar('new'|'in_progress'|'done')` + `completed_at timestamptz`. Богаче ТЗ, но `boolean` из ТЗ ушёл. По духу не хуже.
- ⚠ Route `POST /api/createTask` выбивается из общего стиля. Остальные пути REST-овые (`GET /api/tasks`, `PATCH /api/task/{id}`, `DELETE /api/task/{id}`), тут action-verb. Ожидается `POST /api/tasks`.
- ⚠ ТЗ говорит "Migrations applied on application startup". По факту миграции запускаются вручную через `make compose-migrate`, backend их сам не накатывает.
- ⚠ CI на GitLab вместо GitHub Actions из ТЗ. Формально сути не меняет, но пункт стоит явно упомянуть.

---

## АРХИТЕКТУРА В ЦЕЛОМ

Четыре Go-сервиса плюс два внешних (Postgres, Kafka).

`frontend/nginx.conf` проксирует `/api/*` на `backend:8080`, статику отдаёт напрямую из `/usr/share/nginx/html`. Никакого CORS: браузер видит один origin, Nginx делает пробрасывание.

Backend это единственный сервис, который смотрит в Postgres. Он же владелец схемы (миграции в `db/migrations`). Scheduler и email-sender к БД не подключаются.

Kafka используется как асинхронная шина событий, два топика:

- `EMAIL_SENDING_TASKS`, pub: backend (welcome и отчёты), scheduler косвенно через backend. sub: email-sender.
- `NOTIFICATION_EVENTS`, pub: scheduler по cron. sub: backend.

Сообщения key-based: partition key равен `userID` для email tasks и `jobID` для notification events. Это гарантирует порядок для одного пользователя/джоба в рамках партиции.

Scheduler отдельно ещё выставляет gRPC (`:50051`), но снаружи его сейчас никто не дёргает, регистрация дефолтной джобы происходит в `main` самого scheduler'а на старте. Ключевой архитектурный промах: gRPC используется только для регистрации джоб, а не для их фактического выполнения. Триггер уходит в Kafka fire-and-forget, из-за чего scheduler не видит результата работы и статусы джоб не значат ничего (подробнее в разделе SCHEDULER ниже).

### Микросервисные паттерны

- Event-driven через Kafka. Backend не ждёт email-sender: публикует в топик и идёт дальше. Тяжёлая SMTP-отправка не блокирует HTTP-запрос пользователя.
- Choreography, не orchestration. Нет отдельного оркестратора: scheduler публикует событие, backend его подхватывает, публикует новое событие, email-sender подхватывает. Каждый сервис знает только свой топик.
- Sync RPC для управления. gRPC у scheduler'а для команд регистрации/запуска джоб. Не для критичных путей.
- API Gateway через Nginx. Nginx стоит перед backend и служит и раздатчиком статики, и прокси. Из-за этого не нужен CORS: браузер видит один origin.
- Config через env-переменные, следуя 12-factor. С оговорками: часть настроек всё ещё захардкожена.
- Worker pool. Email-sender использует классический паттерн с buffered channel и фиксированным числом воркеров. Простой backpressure.
- Consumer groups. Отдельные group ID для разных подписчиков (`email-sender` и `backend-notification-reports`), что позволяет масштабировать реплики каждого сервиса горизонтально.
- Partition key. `userID` как partition key для email tasks. Все сообщения одного юзера идут в одну партицию по порядку.

Чего нет из типового микросервисного набора:

- Idempotency. При повторной доставке (Kafka at-least-once) welcome-email отправится дважды. Нет ключа идемпотентности в EmailTask, нет дедупа на стороне email-sender. Отчёты по расписанию ещё хуже: если consumer поднялся с offset ago, старые события переиграются и пользователь получит вчерашний отчёт заново.
- Distributed tracing. Ни OpenTelemetry, ни trace-id в заголовках/сообщениях. Отследить путь запроса через все сервисы будет тяжело.
- Metrics/observability. Только `log/slog`, ни Prometheus `/metrics`, ни healthcheck-эндпоинтов у самих сервисов (только у Postgres/Kafka в композе).
- Circuit breaker. При падении SMTP email-sender будет бесконечно ретраить (кроме poison-messages). Хорошо бы `sony/gobreaker` или явные бэкоффы.
- Dead letter queue. Poison-messages в email-sender просто помечаются и теряются. Стоит писать их в отдельный DLQ-топик для разбора.
- Retry with backoff в consumer'ах. У продюсеров есть `Retry.Max = 5`, у консьюмеров нет ничего. См. отдельно проблему с `NotificationReportService` ниже.

### Зависимости между сервисами

Compose:

- `backend` зависит от `postgres: service_healthy` и `kafka: service_healthy`.
- `email-sender` зависит только от `kafka: service_healthy`.
- `scheduler` зависит только от `kafka: service_healthy`.
- `frontend` зависит от `backend`. Nginx ждёт backend, но `depends_on` без `condition` не проверяет readiness, nginx может стартовать до того, как backend начнёт принимать запросы. Не критично, но лучше добавить healthcheck на backend и `service_healthy`.

Runtime:

- Все три Go-сервиса имеют жёсткий хардкод `kafka:9092` внутри кода. Один env-варьей `KAFKA_BROKERS` было бы кратно проще.

### Общие замечания по инфраструктуре кода

- `docker-compose.yml` без прод-портов, override-файлы `docker-compose.dev.yml` и `docker-compose.prod.yml` для окружений. Слоями конфиг разложен аккуратно.
- `Dockerfile.dev` ставит `air`, `dlv`, `gotestsum`, `migrate`. Прод-Dockerfile (`Dockerfile`) собирает только бинарь. Значит `make compose-migrate` работает только в dev-режиме, прод-образ backend'a не содержит `/go/bin/migrate`. В проде миграции придётся катить каким-то другим способом (init-container, отдельный джоб).
- Топики Kafka создаются не автоматически, а через `make compose-kafka-init`. Логика та же, нужен ручной шаг перед первым запуском. Стоит либо включить auto-create в Kafka, либо сделать одноразовый init-контейнер.
- CI: `.gitlab-ci.yml` со стадиями build, test, publish, тесты используют реальный Postgres из service, миграции применяются перед тестами.

---

## BACKEND

REST API + бизнес-логика + БД + два Kafka-канала (продюсер `EMAIL_SENDING_TASKS`, консьюмер `NOTIFICATION_EVENTS`). Layered arch: `controllers`, `services`, `repositories`, `db`, плюс `domain` (модели, ошибки, валидации), `dto` (request/response), `middlewares`, `httputils`, `validator`, `messaging`, `password`.

### Хорошо

- Чистая слоистая архитектура. Контроллеры не знают про БД, репозитории не знают про HTTP, сервисы посередине.
- Централизованный маппинг доменных ошибок в HTTP через `httputils.HandleError`. Формат ответа `{"message": "..."}` везде.
- JWT валидация вынесена в middleware, `user_id` в контексте, контроллеры получают его через типизированный хелпер `UserIDFromContext(ctx) (int, bool)`.
- Bcrypt для паролей через `golang.org/x/crypto/bcrypt`. Стандартная стойкость.
- Валидация email через `net/mail.ParseAddress` + строгое сравнение `addr.Address == email`, отсекает форматы вида `Name <email>`.
- Все SQL-запросы параметризованы. Инъекции невозможны в принципе.
- В `GetUserByID`/`GetUserByEmail` возвращается `domain.ErrUserNotFound`, а не сырой `sql.ErrNoRows`. Драйверные ошибки не текут наружу.
- В `SignIn` при отсутствии пользователя возвращается `ErrInvalidCredentials`, а не сообщение про несуществующего юзера. Так атакующий не различает случай незарегистрированного email и случай неверного пароля.
- `http.ServeMux` из Go 1.22 с pattern-routing (`POST /api/user`, `PATCH /api/task/{id}`), без внешних роутеров.
- Structured logging через `log/slog` в middleware с method/status/duration_ms/path.
- `defer rows.Close()` + `rows.Err()` после цикла в репозиториях. Про эти два шага работы с драйвером многие забывают.
- В `TaskController.GetTasks` слайс инициализируется с capacity: `make([]dto.TaskResponse, 0, len(tasks))`. Мелочь, но чувствуется.
- Consumer группа `backend-notification-reports` использует `OffsetOldest`, при первом старте прочитает все накопленные события.
- Publisher key для Kafka это `userID`, обеспечивает порядок сообщений одного пользователя.

### Недостатки

- Копипаст-ошибка в `TaskRepository.CreateTask` ([task_repository.go:33](task-tracker/internal/repositories/task_repository.go#L33)):

```go
if errors.As(err, &pqErr) && pqErr.Code == "23505" {
    return domain.Task{}, domain.ErrUserAlreadyExists
}
```

Обрабатывается unique_violation, возвращается `ErrUserAlreadyExists`. У таблицы `tasks` вообще нет UNIQUE-констрейнтов, ветка мёртвая. Плюс имя ошибки не про то.

- JWT-ключ молча пустой при отсутствии `JWT_KEY` ([cmd/main.go:38](task-tracker/cmd/main.go#L38)):

```go
jwtKey := []byte(os.Getenv("JWT_KEY"))
```

`os.Getenv` вернёт пустую строку. `jwt.SignedString([]byte(""))` подпишет токен пустым ключом. Крайне слабая подпись. Сравни с `db.NewPostgres`, где для `DATABASE_URL` явно паникуют. Добавь такую же проверку:

```go
jwtKey := []byte(os.Getenv("JWT_KEY"))
if len(jwtKey) == 0 {
    panic("JWT_KEY environment variable not set")
}
```

- `SignOut` требует непустое JSON-тело, иначе 400:

```go
func (c *AuthController) SignOut(w http.ResponseWriter, r *http.Request) {
    var req dto.SignOutRequest
    err := json.NewDecoder(r.Body).Decode(&req)
    if err != nil {
        httputils.HandleError(w, domain.ErrInvalidJSON)
        return
    }
    w.WriteHeader(http.StatusNoContent)
}
```

`SignOutRequest` пустая, но декодер на пустом теле возвращает `io.EOF`. Клиент вынужден слать `{}`. Убери чтение тела совсем, либо игнорируй `io.EOF`.

Плюс: JWT stateless, сервер не может действительно разлогинить токен без blacklist-хранилища. Endpoint сейчас чисто декоративный. Либо добавляй blacklist (Redis), либо переключайся на refresh-токены.

- Нет graceful shutdown:

```go
func (s *Server) Start() {
    err := http.ListenAndServe(":8080", s.mux)
    if err != nil {
        log.Fatal(err)
    }
}
```

`log.Fatal` вызывает `os.Exit(1)`, defer'ы в `main` (`producer.Close`, `notificationConsumerGroup.Close`) не выполнятся. При SIGTERM тот же результат. Как надо:

```go
srv := &http.Server{Addr: ":8080", Handler: mux, ReadTimeout: ..., WriteTimeout: ...}
ctx, cancel := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
defer cancel()

go func() {
    if err := srv.ListenAndServe(); err != nil && !errors.Is(err, http.ErrServerClosed) {
        log.Printf("server error: %v", err)
    }
}()

<-ctx.Done()
shutdownCtx, sc := context.WithTimeout(context.Background(), 10*time.Second)
defer sc()
srv.Shutdown(shutdownCtx)
```

Заодно заведи таймауты у `http.Server` (`ReadHeaderTimeout`, `ReadTimeout`, `WriteTimeout`, `IdleTimeout`).

- `EditTask` в репозитории собран из 7 SQL-запросов, написанных руками ([task_repository.go:134-223](task-tracker/internal/repositories/task_repository.go#L134)):

```go
switch {
case input.Title != nil && input.Description != nil && input.Status != nil: ...
case input.Title != nil && input.Description != nil: ...
case input.Title != nil && input.Status != nil: ...
case input.Description != nil && input.Status != nil: ...
case input.Title != nil: ...
case input.Description != nil: ...
case input.Status != nil: ...
}
```

Каждый вариант это отдельный SQL с почти идентичной логикой completed_at. Добавь ещё одно опциональное поле, веток станет 15. ТЗ прямо упоминает Squirrel, но подойдёт либо он, либо самостоятельно собранный динамический запрос со списком `SET`-выражений и параметров:

```go
q := sq.Update("tasks").Where(sq.Eq{"id": input.ID, "user_id": userID}).Set("updated_at", sq.Expr("NOW()"))
if input.Title != nil { q = q.Set("title", *input.Title) }
if input.Description != nil { q = q.Set("description", *input.Description) }
if input.Status != nil {
    q = q.Set("status", *input.Status)
    q = q.Set("completed_at", sq.Expr("CASE WHEN ?::varchar = 'done' THEN COALESCE(completed_at, NOW()) ELSE NULL END", *input.Status))
}
```

- `NotificationReportService` даёт бесконечный ретрай poison-события:

```go
// subscriber.go
if err := s.handler.HandleNotificationStarted(session.Context(), event); err != nil {
    log.Printf("notification event handling failed: %v", err)
    continue    // не марк
}
session.MarkMessage(msg, "")
```

Если БД упала во время обработки одного пользователя, `HandleNotificationStarted` вернёт ошибку, сообщение не пометится. Consumer group его переиграет, и снова упадёт на том же пользователе. Retry без лимита, без backoff. Сравни с email-sender'ом: там `ErrInvalidEmailTask` явно распознан как poison и всё равно помечается.

Fix: либо retry-limit + DLQ, либо частичный успех внутри обработчика, ловить ошибки на пользователя и логировать, чтобы одно падение не блокировало весь batch.

- `CreateTask` принимает произвольный `status` от клиента. Можно создать задачу сразу помеченную как `done`, тогда SQL проставит `completed_at = NOW()`. Свежая задача не должна быть выполненной. Форсь `status = new`:

```go
task, err := c.service.CreateTask(
    domain.Task{
        Title:       req.Title,
        Description: req.Description,
        Status:      domain.StatusNew,
    },
    userID,
)
```

- `POST /api/createTask` это action-verb в REST API. Остальные endpoint'ы выдержаны в REST-стиле. Ожидается `POST /api/tasks`.

### Замечания

- Сервисы копируют репозитории по значению:

```go
type UserService struct {
    repo repositories.UserRepository   // не указатель
}

func NewUserService(repo *repositories.UserRepository) *UserService {
    return &UserService{repo: *repo}   // разыменование и копирование
}
```

Работает, потому что `UserRepository` внутри содержит `*sql.DB`. Но неконсистентно: `AuthService` держит репозиторий как `*UserRepository`, `UserService`/`TaskService`, как значение. Выбери одно.

- `ValidateTaskRequest` считает длину title в байтах через `len`, а `ValidateCredentials`, в рунах через `utf8.RuneCountInString`. Кириллический title получает вдвое меньше свободного места. Приведи к общему знаменателю.

- `EditTask` в сервисе триммит title, но не description. Либо оба, либо ни одного.

- В сервисе для `EditTask` валидация полей написана по месту, в `CreateTask`, через `ValidateTaskRequest`. Общий валидатор `ValidateTaskFields(title, description *string, status *TaskStatus)` сжал бы обе точки.

- `defer rows.Close()` присваивает во внешний `err`:

```go
defer func() {
    if err = rows.Close(); err != nil {
        slog.Warn(...)
    }
}()
```

Технически ничего не ломает (безымянные returns уже захватили значение). Но стилистически лучше локальная переменная `closeErr` внутри defer'а.

- В `SignUpRequest` странный порядок полей: сначала пароли, потом email. По ТЗ порядок: email, password, repeat_password. Плюс `password_confirm` vs ТЗ-шного `repeat_password`.

- Middleware chain повторяется дословно на каждом роуте: `middlewares.Logger(middlewares.JSON(middlewares.Auth(handler, s)))`. Мелкий хелпер `chain(handler, mws...)` или пакет `alice` уберут визуальный шум.

- TTL токена 15 минут захардкожен. Вынеси в env, чтобы можно было менять без пересборки.

- `Auth` middleware не различает отсутствующий токен и невалидный. Оба случая, 401 с одинаковым сообщением. Клиент не поймёт, редиректить на форму логина или пытаться refresh.

- Сообщения ошибок разноязычные: `HandleError` пишет по-русски (например, `Задача не найдена`), сами `error.Error()` по-английски (например, `task not found`). В API русские, в логах английские. Определись с одним языком.

- `NotificationReportService` тянет всех пользователей одним запросом `GetUsers()`. Даже на MVP при паре тысяч записей это уже начнёт сажать память. Нужна пагинация или курсорный обход.

---

## SCHEDULER

Отдельный процесс с двумя точками входа: gRPC-сервер на `:50051` для управления джобами и cron-планировщик (`robfig/cron/v3`), публикующий события в Kafka. Джобы хранятся в памяти в мапе `map[string]domain.Job` под `sync.RWMutex`.

### Хорошо

- Полностью изолированный сервис. Не знает про Postgres, не знает про SMTP, не знает про пользователей. Задача одна: запустить X по крону и кинуть событие Y в Kafka. Доменная граница прочерчена без утечек.
- `robfig/cron/v3`, стандартная библиотека для крона в Go.
- gRPC-контракт (`cron.proto`) описан отдельно, чистый, три RPC: `RegisterJob`, `RunJob`, `GetJobStatus`.
- `sync.RWMutex` вокруг мапы джоб. Верно: чтения (`GetJobStatus`) идут через RLock, записи, через Lock.
- Статусы `scheduled / running / failed` в жизненном цикле джобы.
- Каждый вызов cron задаёт `context.WithTimeout(30*time.Second)` внутри лямбды. Джоба не зависнет насмерть.
- В `RunJob` перед публикацией в Kafka явно проверяется `ctx.Err()`. Если контекст успел отмениться, не публикуем.
- Timestamp `TriggeredAt` присылается в UTC. Ни у кого не поедут таймзоны при сериализации.
- `cron_service_kafka_test.go`, есть тест на публикацию.
- Дефолтная джоба регистрируется с 10-секундным таймаутом контекста, потом контекст отменяется через defer.

### Недостатки

- Все ошибки в gRPC-хендлере маппятся в `InvalidArgument`:

```go
if err != nil {
    if errors.Is(err, sservices.ErrInvalidJob) {
        return nil, status.Error(codes.InvalidArgument, "...")
    }
    return nil, status.Error(codes.InvalidArgument, err.Error())
}
```

`ErrInvalidJob` действительно `InvalidArgument`. Но если `scheduler.AddFunc` вернул ошибку от cron-парсера (`invalid cron expression`), это тоже InvalidArgument, а если cron упал по внутренним причинам, это уже Internal. Сейчас всё сваливается в 400-й gRPC-статус. Как надо:

```go
if errors.Is(err, sservices.ErrInvalidJob) {
    return nil, status.Error(codes.InvalidArgument, ...)
}
if isCronParseError(err) {
    return nil, status.Error(codes.InvalidArgument, err.Error())
}
return nil, status.Error(codes.Internal, err.Error())
```

- Джобы теряются при рестарте scheduler'а. Мапа `jobs` живёт в памяти. При перезапуске регистрируется только `defaultNotificationJob` из `main`, кастомные gRPC-регистрации пропадают. Для этого сервиса джобы обязаны храниться в Postgres: `RegisterJob` пишет запись, при старте scheduler читает таблицу и восстанавливает расписание, `RunJob` обновляет `last_run_at` там же. Сейчас же весь стейт живёт в in-memory `map[string]domain.Job`, что делает gRPC-API бесполезным между перезапусками. Это откровенный косяк, а не мелочь на будущее.

- Нет endpoint'ов для удаления и списка джоб. Есть только `RegisterJob`/`RunJob`/`GetJobStatus`. Управлять жизненным циклом (дерегистрировать, посмотреть все) через gRPC нельзя.

- Дефолтная cron-строка `"0 0 * * *"` захардкожена в `main.go`. Хочется её поменять, только через пересборку. Стандартный env-переменной здесь очень не хватает.

- `kafka:9092` захардкожена в `messaging.NewSyncProducer([]string{"kafka:9092"})`. Ровно та же проблема, что в backend и email-sender.

- `:50051` порт gRPC-сервера захардкожен. Env хотел бы.

### Замечания

- Асинхронность через Kafka ломает саму идею статусов джобы. Сейчас `RunJob` делает так: поставил `Status = running`, отправил сообщение в Kafka, поставил `Status = scheduled` обратно. Реальная работа (сбор отчёта, публикация email-задач, отправка писем) выполняется где-то в backend'e и email-sender'e, а scheduler про это не знает ничего. `GetJobStatus` вернёт `scheduled` и `LastRunAt` последнего *публикейта в Kafka*, а не последнего фактического выполнения. Статусы `running`/`failed` бесполезны, потому что живут секунду-две между Lock/Unlock и не отражают исход работы. `failed` реально фиксирует только ошибку публикации в Kafka, а не падение самого джоба.

Чтобы это чинилось нормально, надо развернуть контракт: каждый исполнитель джобы должен реализовывать gRPC-эндпоинт (например, `RunCron(RunCronRequest) returns (RunCronResponse)`) и регистрировать себя в scheduler'e. Scheduler по крону вызывает этот gRPC синхронно, ждёт ответа, и уже по возврату (или таймауту) выставляет `succeeded`/`failed`, пишет `LastRunAt` и `LastError`. Тогда gRPC-контракт scheduler'а перестаёт быть декоративным и `GetJobStatus` начинает возвращать что-то полезное.

Kafka в этой схеме остаётся, но не как способ триггерить работу, а либо как транспорт для broadcast-событий (`notification.started` для реплик backend'а), либо просто убирается. Сейчас же гибрид: scheduler и владелец расписания, и просто publisher в Kafka, а бизнес-логика висит на подписчике, до которого scheduler не дотягивается.

- `markFailed` меняет статус на `failed`, но джоба остаётся зарегистрированной в cron. Следующий тик снова её выполнит. Ок как поведение по расписанию, но без отдельного `LastError` из этого статуса ничего не выжать.

- `LastRunAt` пишется только при успехе. При падении между началом `RunJob` и публикацией `LastRunAt` не обновляется, статус `failed`, но неясно, когда именно произошёл этот failed. Пиши `LastRunAt` в любом случае.

- Дефолтная джоба публикует событие с `EventType = "notification.started"`. Это же значение проверяется в backend (`if event.EventType != NotificationStartedEventType { return nil }`). Между сервисами захардкожена строка. При смене конвенции нужно править в двух местах. Общий пакет с константами уменьшил бы риск.

---

## EMAIL-SENDER

Kafka consumer подписан на `EMAIL_SENDING_TASKS`, для каждого сообщения парсит JSON `EmailTask{To, Subject, Body}` и шлёт SMTP. Классический worker pool: буферизованный канал на 10, три горутины-воркера.

### Хорошо

- Явное различение poison-messages. `ErrInvalidEmailTask` (невалидный JSON или пустые `to`/`subject`) помечается, чтобы не переигрывать бесконечно. Ошибки SMTP не помечаются, следующий poll попробует ещё раз.
- Buffered channel + 3 воркера. Простой backpressure.
- `SMTPConfig.LoadSMTPConfig` валидирует required-поля, возвращает ошибку с явным сообщением, если чего-то нет.
- Поддержка `StartTLS` через ручной SMTP-flow (`smtp.Dial`, `StartTLS`, `Auth`, `Mail`, `Rcpt`, `Data`).
- Subject кодируется через `mime.QEncoding`, UTF-8 не сломается.
- MIME-заголовки собираются вручную, но корректно (`MIME-Version`, `Content-Type`, `From`, `To`, `Subject`).
- Валидация email-адреса получателя через `net/mail.ParseAddress`.
- Отдельный `domain.EmailTask` в `cmd/email-sender/domain`. Consumer не тянет доменную модель бэкенда, только контракт события. Сервисы связаны через контракт, а не через shared code.

### Недостатки

- Race при завершении `ConsumeClaim`:

```go
func (s *Subscriber) ConsumeClaim(session, claim) error {
    jobs := make(chan *sarama.ConsumerMessage, 10)
    defer close(jobs)
    for i := 0; i < 3; i++ {
        go worker(session, jobs, s.sender)
    }
    for msg := range claim.Messages() {
        jobs <- msg
    }
    return nil
}
```

Когда `claim.Messages()` закрывается (rebalance или shutdown), for-range выходит, defer закрывает `jobs`, воркеры видят закрытие и выходят. Но `ConsumeClaim` возвращается до того, как воркеры доделали текущие сообщения. Sarama может закрыть session, а воркер в этот момент вызывает `session.MarkMessage`. Race: MarkMessage может не пройти. Через `sync.WaitGroup`:

```go
var wg sync.WaitGroup
jobs := make(chan *sarama.ConsumerMessage, 10)
for i := 0; i < 3; i++ {
    wg.Add(1)
    go func() { defer wg.Done(); worker(session, jobs, s.sender) }()
}
for msg := range claim.Messages() {
    jobs <- msg
}
close(jobs)
wg.Wait()
return nil
```

- Нет retry с backoff при SMTP-ошибке. Если Mailjet вернул 5xx, worker логирует и не помечает. Consumer group переиграет сообщение сразу же, снова уйдёт на SMTP. Если SMTP лежит минуту, все это время цикл будет молотить впустую. Хорошо бы exponential backoff или временный circuit breaker.

- Нет DLQ для сообщений, которые почти-корректны, но не отправляются. Poison messages помечаются и теряются в логах. Лучше выписывать их в DLQ-топик.

- Не учтён rate limit Mailjet (200 писем/день на free tier). При флаге спама scheduler'ом эти лимиты будут пробиваться быстро.

- Хардкод `kafka:9092`, `"email-sender"` (consumer group), `3` воркера, `10` в буфере. Всё в переменные окружения.

- `worker` не отменяется по context. Если consumer group переходит в rebalance, sarama закрывает `claim.Messages()`, но воркер продолжает висеть, если SMTP медленный. Использовать `session.Context()` внутри воркера с cancellation.

### Замечания

- `SendEmail` в `handler.go` дублирует проверку `to`/`subject` empty, она уже есть в `Sender.Send`. Одна из двух, лишняя.

- `SMTP_STARTTLS` env-переменная парсится через `env("SMTP_STARTTLS", "true") != "false"`. Значение `"FALSE"` (uppercase) даст `true`. Хрупко. `strconv.ParseBool` устойчивее.

- `Sender.Send` строит MIME-письмо каждый вызов. Хедеры простые, но для миллиона писем, миллион аллокаций. Мелочь для scale проекта.

- Нет метрики по числу отправленных и упавших писем за интервал. Только логи.

- Consumer group ID `"email-sender"` захардкожен. Одно значение на всё, при масштабировании реплики email-sender'а автоматически балансятся, и это то, что нужно. Но для локальных тестов иногда полезно менять группу.

---

## ИНФРАСТРУКТУРА

### Docker Compose

Layered override files:

- `docker-compose.yml`, базовая конфигурация, без публичных портов (Postgres открыт на `54330:5432` для доступа с хоста, всё остальное внутри docker DNS).
- `docker-compose.dev.yml`, dev-порты (`8081`, `8082`, `50051`), `Dockerfile.dev` с `air` для hot-reload и `dlv` для дебага, volume `.:/backend` для проброса кода.
- `docker-compose.prod.yml`, публичные порты (`80`, `8080`, `50051`).

Слоями конфигурация разложена аккуратно. `make dev` собирает всё разом и накатывает миграции.

### Healthchecks

- Postgres: `pg_isready` каждые 5 сек. Хорошо.
- Kafka: `kafka-topics.sh --list` каждые 10 сек с 30-сек start_period. Тоже нормально.
- Backend, scheduler, email-sender: healthcheck'ов нет. Compose не может проверить, что backend поднялся. Если стартует frontend раньше готовности backend, nginx проксирует в 502.

### Миграции

`db/migrations/000001_create_users.up.sql` и `000002_create_tasks.up.sql`. Формат отдельных `up`/`down` файлов, стандартный для `golang-migrate`.

Проблемы:

1. Миграции не применяются на старте. ТЗ явно требует "Migrations applied on application startup". По факту нужен ручной `make compose-migrate`.
2. Прод-образ backend'а не содержит `/go/bin/migrate`. `Dockerfile` (production) собирает только бинарь, `migrate` устанавливается только в `Dockerfile.dev`. Значит `make compose-migrate` в проде не сработает без дополнительной подготовки образа.

Fix: либо запускать миграции автоматически в `main.go` перед `srv.Start()`, либо добавить отдельный init-контейнер с `migrate`, зависящий от `postgres: service_healthy` и предшествующий `backend`.

### Kafka топики

Создаются вручную через `make compose-kafka-init` (два топика: `EMAIL_SENDING_TASKS`, `NOTIFICATION_EVENTS`). Ещё один шаг после первого запуска. Либо включить `KAFKA_AUTO_CREATE_TOPICS_ENABLE=true`, либо init-контейнер.

---

## Итог

Проект соответствует ТЗ на редкость полно: все четыре сервиса, JWT, bcrypt, Postgres с миграциями, Kafka с двумя топиками, воркер-пул в email-sender, gRPC scheduler с cron, docker-compose со всем стеком, GitLab CI, тесты трёх типов. Архитектурно чисто: слои в backend'e разделены, сервисы не имеют лишних зависимостей друг от друга, event-driven через Kafka сделан по книге.

Из реальных багов и рисков:

- JWT-ключ молча пустой при отсутствии env-переменной (безопасность).
- Копипаст `ErrUserAlreadyExists` в `TaskRepository.CreateTask` (мёртвая ветка + не тот error).
- `SignOut` требует непустое JSON-тело, иначе 400.
- Нет graceful shutdown ни в одном сервисе, при SIGTERM defer'ы не выполняются, продюсеры и consumer group'ы не закрываются.
- Poison-message без retry-limit в notification-consumer у backend'а.
- Race при завершении `ConsumeClaim` в email-sender (нет `sync.WaitGroup`).
- Миграции не применяются на старте (ТЗ требует), плюс `migrate` в прод-образе нет.
- `EditTask` в репозитории из 7 хардкоженных SQL-веток.
- `CreateTask` принимает произвольный `status` от клиента (можно создать сразу выполненную задачу).
- Все ошибки в gRPC-хендлере scheduler'а маппятся в `InvalidArgument` вне зависимости от природы.

Из мелочи: хардкод `kafka:9092` и `":8080"` в трёх местах, нет метрик, нет tracing'а, нет idempotency ключей в событиях Kafka, action-verb `POST /api/createTask`, копия репозиториев по значению вместо указателя, `len(title)` вместо `utf8.RuneCountInString`, разноязычные сообщения ошибок, отсутствие таймаутов на HTTP-сервере.
