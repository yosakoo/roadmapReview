# Ревью Currency Exchange
Проект: https://gitlab.com/fanatioon/Currency-Exchange

REST API для работы с валютами и обменными курсами. Трёхслойная архитектура (controllers -> services -> repo), Docker, PostgreSQL. Проект запускается.

<img src="https://i.pinimg.com/736x/c7/46/85/c74685ff9087a80de9cddb74ffb975a0.jpg" width="300"/>

---

## НЕДОСТАТКИ РЕАЛИЗАЦИИ

- **`defer rows.Close()` вызывается до проверки ошибки.** Если `Query` вернёт ошибку, `rows` будет `nil` и `rows.Close()` упадёт с паникой:

```go
// repo/currency.go и repo/exchange_rate.go
rows, err := r.db.Query(query)
defer rows.Close()   // паника, если err != nil

if err != nil {
    return nil, err
}
```

`defer` ставится только после успешной проверки:

```go
rows, err := r.db.Query(query)
if err != nil {
    return nil, err
}
defer rows.Close()
```

- **Ошибки сканирования строк молча игнорируются.** В обоих `GetAll` объявляется `scanErr`, но в `return` возвращается внешний `err`, который на тот момент `nil`:

```go
// repo/currency.go
scanErr := rows.Scan(&currency.ID, &currency.Code, &currency.Name, &currency.Sign)

if scanErr != nil {
    return nil, err   // должно быть scanErr, здесь err всегда nil!
}
```

Любая ошибка при чтении строки из базы вернёт `nil, nil` вместо ошибки.

- **`Create` не сохраняет ID.** Метод принимает значение, а не указатель. `RETURNING id` сканируется в копию, которая сразу уничтожается:

```go
// repo/currency.go
func (r *CurrencyRepository) Create(currency models.Currency) error {
    query := `INSERT INTO currencies (code, name, sign) VALUES ($1, $2, $3) RETURNING id`
    return r.db.QueryRow(query, ...).Scan(&currency.ID)  // currency копия, ID теряется
}
```

Контроллер возвращает клиенту ответ с `id: 0`.

- **Порт сервера захардкожен, `APP_PORT` не работает.** В `docker-compose.yml` прокидывается `${APP_PORT}:8080`, но в `main.go` сервер всегда слушает `:8080`. Если изменить `APP_PORT`, маппинг портов сломается:

```go
// cmd/main.go
http.ListenAndServe(":8080", nil)  // APP_PORT из env не читается
```

- **Ошибка `http.ListenAndServe` игнорируется.** Если порт занят, программа молча завершится:

```go
http.ListenAndServe(":8080", nil)
```

```go
log.Fatal(http.ListenAndServe(":8080", nil))
```

- **`writeJSON` пишет заголовки до кодирования, потом вызывает `http.Error`.** `w.WriteHeader(http.StatusOK)` отправляет заголовки клиенту ещё до `Encode`. Если `Encode` падает, `http.Error` пытается изменить уже отправленные заголовки. Клиент получает 200 и обрезанный JSON:

```go
func writeJSON(w http.ResponseWriter, data any) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(http.StatusOK)

    if err := json.NewEncoder(w).Encode(data); err != nil {
        http.Error(w, err.Error(), http.StatusBadRequest)  // слишком поздно
        return
    }
}
```

---

## ХОРОШО

- Трёхслойная архитектура: controllers -> services -> repo. Слои разделены, бизнес-логика не смешана с SQL.
- Dependency injection через конструкторы. Зависимости явные, не глобальные.
- Параметризованные SQL-запросы везде, SQL-инъекции невозможны.
- `errors.Is(err, sql.ErrNoRows)` для проверки "не найдено". Идиоматично.
- `r.PathValue("code")` из стандартного `net/http` (Go 1.22+) вместо сторонних роутеров.
- Docker Compose для локального запуска, `.env.example` в репозитории.

---

## ЗАМЕЧАНИЯ

### 1. `repo/db.go`, мёртвый код

Файл объявляет глобальную переменную `DB` и функции `InitDB`/`CloseDB`, которые нигде не используются. В `main.go` соединение открывается напрямую. Читаешь код и думаешь, что есть два способа инициализации БД.

Удали `repo/db.go` целиком.

### 2. Неправильный дефолтный порт в `config.go`

```go
Port: getEnv("POSTGRES_PORT", "80"),  // порт PostgreSQL по умолчанию 5432, не 80
```

Если `POSTGRES_PORT` не задана, будет попытка подключиться к порту 80.

### 3. Нет `db.Ping()` после `sql.Open`

`sql.Open` не устанавливает соединение, только проверяет строку DSN. Реальное подключение происходит при первом запросе. Если БД недоступна, сервер стартует без ошибок и проблема всплывёт только на первом запросе:

```go
db, err := sql.Open("postgres", postgresInfo)
if err != nil {
    log.Fatal(err)
}

if err := db.Ping(); err != nil {
    log.Fatal("не удалось подключиться к БД:", err)
}
```

### 4. Все ошибки возвращают `400 Bad Request`

```go
if err != nil {
    http.Error(w, err.Error(), http.StatusBadRequest)  // и для 404, и для 500
    return
}
```

HTTP-статусы несут смысл, клиент должен понимать что произошло:

| Ситуация | Статус |
|---|---|
| Валюта не найдена | `404 Not Found` |
| Валюта уже существует | `409 Conflict` |
| Ошибка базы данных | `500 Internal Server Error` |
| Неверные данные в теле | `400 Bad Request` |

Для этого нужны sentinel-ошибки в сервисах:

```go
var ErrNotFound = errors.New("not found")
var ErrAlreadyExists = errors.New("already exists")
```

И в контроллере:

```go
switch {
case errors.Is(err, services.ErrNotFound):
    http.Error(w, err.Error(), http.StatusNotFound)
case errors.Is(err, services.ErrAlreadyExists):
    http.Error(w, err.Error(), http.StatusConflict)
default:
    http.Error(w, "internal error", http.StatusInternalServerError)
}
```

### 5. Валидация в `Create` происходит после запроса к БД

```go
func (s *CurrencyService) Create(currency models.Currency) error {
    code := currency.Code
    existingCurrency, err := s.GetByCode(code)  // запрос к БД до валидации!

    if len(currency.Code) != 3 || len(currency.Name) == 0 || len(currency.Sign) == 0 {
        return errors.New("invalid value")
    }
    ...
```

Если передать `code = ""`, сначала уйдёт запрос в базу, потом вернётся ошибка валидации. Валидация должна идти первой:

```go
func (s *CurrencyService) Create(currency models.Currency) error {
    if len(currency.Code) != 3 || len(currency.Name) == 0 || len(currency.Sign) == 0 {
        return errors.New("invalid value")
    }
    existingCurrency, err := s.GetByCode(currency.Code)
    ...
```

### 6. `GetByCodes` проверяет длину в байтах, а не символах

```go
if len(codes) != 6 {  // len() считает байты, не символы
```

Для `USDRUB` работает, но `len` в Go считает байты. Для кодов не из ASCII сломается. Используй `utf8.RuneCountInString(codes) != 6`.

### 7. `GetByID` использует `SELECT *`

```go
query := `SELECT * FROM exchange_rates WHERE base_currency_id = $1 AND target_currency_id = $2`
err := r.db.QueryRow(query, baseID, targetID).Scan(&exchangeRate.ID, &exchangeRate.BaseCurrency.ID, ...)
```

`SELECT *` возвращает столбцы в порядке из схемы. Добавишь новый столбец и `Scan` молча запишет данные не в те поля. Перечисляй столбцы явно.

### 8. В `docker-compose.yml` нет ожидания готовности БД

Приложение стартует одновременно с PostgreSQL. База ещё не готова принимать подключения, а приложение уже пытается подключиться и падает. Добавь healthcheck для `db` и `depends_on` в `app`:

```yaml
db:
    image: postgres:15
    healthcheck:
        test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER}"]
        interval: 5s
        timeout: 5s
        retries: 5

app:
    depends_on:
        db:
            condition: service_healthy
```

### 9. Можно добавить загрузку `.env` через `godotenv`

`godotenv` это стандарт для загрузки `.env`-файлов в Go, зависимость уже есть в `go.mod`. Сейчас без явного вызова `godotenv.Load()` файл `.env` читается только в Docker через `environment:` в compose — при локальном запуске без Docker переменные надо выставлять вручную.

Вызов лучше класть в пакет `config`, туда, где переменные окружения и так уже читаются:

```go
// internal/config/config.go
func NewConfig() *Config {
    _ = godotenv.Load()  // игнорируем ошибку: файл может отсутствовать
    return &Config{...}
}
```

### 10. Ошибки сервиса не логируются

```go
exchange, err := c.service.Exchange(req.From, req.To, req.Amount)
if err != nil {
    writeServiceError(w, err)  // ошибка ушла клиенту, в логах тишина
    return
}
```

В продакшене получишь 500 и не узнаешь что произошло. Логируй перед отправкой ответа:

```go
if err != nil {
    log.Printf("Exchange error: %v", err)
    writeServiceError(w, err)
    return
}
```

Для чего-то серьёзнее подключи `slog` (стандартная библиотека) или `zap`, чтобы к каждой ошибке прикладывался контекст запроса.

### 11. Доп задача: добавить `/health` и `/ready`

Без этих эндпоинтов оркестратор (Docker, Kubernetes) не знает, живо ли приложение и можно ли слать на него трафик. Разница между ними ([подробнее](https://habr.com/ru/companies/slurm/articles/692450/)):

- `/health` (liveness): процесс жив, зависимости не проверяет. Если не отвечает, контейнер перезапускается.
- `/ready` (readiness): все зависимости доступны. Пока не готов, трафик не идёт.

```go
http.HandleFunc("GET /health", func(w http.ResponseWriter, r *http.Request) {
    w.WriteHeader(http.StatusOK)
})

http.HandleFunc("GET /ready", func(w http.ResponseWriter, r *http.Request) {
    if err := db.Ping(); err != nil {
        http.Error(w, "db unavailable", http.StatusServiceUnavailable)
        return
    }
    w.WriteHeader(http.StatusOK)
})
```

---

## АРХИТЕКТУРА

Слои разделены, зависимости через конструкторы. Хорошая основа.

**Сервисы зависят от конкретных типов репозиториев, а не от интерфейсов:**

```go
type ExchangeRateService struct {
    repo         *repo.ExchangeRateRepository   // конкретный тип
    currencyRepo *repo.CurrencyRepository        // конкретный тип
}
```

Для текущего размера проекта это ок, но когда дойдёшь до тестов — придётся переделывать. Сервисы нельзя тестировать без реальной базы. Объяви интерфейсы в пакете `services`:

```go
type CurrencyRepo interface {
    GetAll() ([]models.Currency, error)
    GetByCode(code string) (*models.Currency, error)
    Create(currency models.Currency) error
}

type ExchangeRateService struct {
    repo         ExchangeRateRepo
    currencyRepo CurrencyRepo
}
```

Тогда в тестах подставишь заглушку без PostgreSQL.

**Нет типизированных ошибок.** Сервис возвращает `errors.New("this currency exists")`, просто строку. Контроллер не может понять, это 404, 409 или 500, и бьёт всё под одну гребёнку с 400.

**Маршруты регистрируются в `main.go`.** `main.go` это точка старта, создаёт зависимости и поднимает сервер. При росте числа эндпоинтов он распухнет. Вынеси маршруты в `internal/server`:

```go
// internal/server/router.go
func NewRouter(currency *controllers.CurrencyHandler, rate *controllers.ExchangeRateHandler) http.Handler {
    mux := http.NewServeMux()
    mux.HandleFunc("GET /currencies", currency.GetAll)
    // ...
    return mux
}

// cmd/main.go
handler := server.NewRouter(currencyHandler, exchangeRateHandler)
log.Fatal(http.ListenAndServe(addr, handler))
```

**Инициализация БД тоже в `main.go`.** Логика создания зависимостей как раз там и должна жить. Но когда появятся реплики или второй пул соединений, сам факторинг подключений стоит вынести в `pkg` - туда, где будет функция для создания конкретного `*sql.DB`. `main.go` вызывает её нужное количество раз и решает что куда передать:

```go
// pkg/db/db.go
func NewConnection(dsn string) (*sql.DB, error) { ... }

// cmd/main.go
mainDB, err := db.NewConnection(cfg.MainDSN)
replicaDB, err := db.NewConnection(cfg.ReplicaDSN)
```

---

## Итог

Структура нормальная: слои разделены, SQL-инъекций нет, DI есть. Но несколько серьёзных багов: паника при ошибке запроса к БД, ошибки сканирования строк не возвращаются, ID при создании записи теряется. HTTP-статусы везде 400 без разбора. Сервисы привязаны к конкретным типам репозиториев, тесты без базы не напишешь. Проект нужно дорабатывать.
