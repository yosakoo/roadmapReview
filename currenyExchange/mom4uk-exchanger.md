# Ревью Currency Exchange
Проект:  https://github.com/mom4uk/currency-exchange

REST API для работы с валютами и обменными курсами. SQLite, миграции, интеграционные тесты, трёхслойная архитектура (domain -> repositories -> services -> controllers). Проект запускается, тесты есть.

<img src="https://i.pinimg.com/736x/12/2e/0e/122e0eeab0434911b09935ef4bbcddcc.jpg" width="300"/>

---

## НЕДОСТАТКИ РЕАЛИЗАЦИИ

- **`GetExchangeRates` не закрывает `rows` и не проверяет `rows.Err()`.** В `currency_repository.go` есть аккуратный `defer` с логированием и проверка `rows.Err()` после цикла, а в `exchange_rate_repository.go` обоих нет. Соединение из пула утекает, а ошибки итератора (например, таймаут соединения во время чтения) игнорируются:

```go
// exchange_rate_repository.go
rows, err := r.db.Query(query)
result := []domain.ExchangeRate{}

if err != nil { ... }

for rows.Next() {
    // defer rows.Close() отсутствует
}
// rows.Err() не проверяется
return result, nil
```

- **`found` проверяется до `err` в двух местах.** Если `GetExchangeRate` вернёт ошибку БД, `found` будет `false` и функция вернёт `ErrExchangeRateNotFound` вместо реальной ошибки. Паттерн встречается и в репозитории, и в сервисе:

```go
// exchange_rate_repository.go — UpdateExchangeRate
exchangeRate, found, err := r.GetExchangeRate(baseCurrency.ID, targetCurrency.ID)
if !found {
    return domain.ExchangeRate{}, domain.ErrExchangeRateNotFound  // маскирует ошибку БД
}
if err != nil {
    return domain.ExchangeRate{}, err
}

// exchange_rate_service.go — GetExchangeRateByCodes
rate, found, err := s.exchangeRateRepository.GetExchangeRate(baseCurrency.ID, targetCurrency.ID)
if !found {
    return domain.ExchangeRate{}, domain.ErrExchangeRateNotFound  // та же ошибка
}
if err != nil {
    return domain.ExchangeRate{}, err
}
```

В обоих случаях сначала проверяй `err`:

```go
if err != nil {
    return domain.ExchangeRate{}, err
}
if !found {
    return domain.ExchangeRate{}, domain.ErrExchangeRateNotFound
}
```

- **`GetExchange` не валидирует query-параметры.** `ValidateExchangeFields` определена в `dto/exchange.go`, но в контроллере не вызывается. Если передать пустой `from` или `to`, запрос уйдёт в `GetCurrencyByCode("")` и вернётся `ErrCurrencyNotFound` вместо "параметр не передан". Плюс невалидный `amount` и невалидный `rate` в PATCH-обработчике оба возвращают 500 вместо 400:

```go
// exchange_controller.go
_, ok := amountValue.SetString(amountStr)
if !ok {
    http.Error(w, "Ошибка в amount", http.StatusInternalServerError)  // должен быть 400
    return
}

// exchange_rate_controller.go
_, ok = rateValue.SetString(rateStr)
if !ok {
    http.Error(w, "Ошибка в rate", http.StatusInternalServerError)  // должен быть 400
    return
}
```

- **`sql.ErrNoRows` после `db.Query` - мёртвый код.** `db.Query` никогда не возвращает `sql.ErrNoRows`, эту ошибку возвращает только `QueryRow().Scan()`. При пустой таблице `Query` вернёт пустой итератор без ошибки, и ветка никогда не выполнится:

```go
rows, err := r.db.Query(query)
if err != nil {
    if err == sql.ErrNoRows {  // это никогда не true
        return nil, sql.ErrNoRows
    }
    return nil, err
}
```

- **Ошибки отдаются в plain text в нескольких местах.** Спецификация требует формат `{"message": "..."}` для всех ошибок, но `http.Error` пишет `Content-Type: text/plain`. Затронуто несколько контроллеров:

```go
// currency_controller.go, exchange_rate_controller.go
http.Error(w, "This method is not allowed", http.StatusMethodNotAllowed)  // plain text
http.Error(w, err.Error(), http.StatusBadRequest)  // plain text при ошибке ParseForm
```

Замени на `utilities.WriteError()`, которая уже используется в этом же проекте для всего остального.

- **Тест `TestPatchExchangeRate_error_currencyNotFound` сломан.** Тест отправляет PATCH без поля `rate`, получает валидационную ошибку "missing rate". Но при этом:

```go
if rr.Code != http.StatusBadRequest {
    t.Fatalf("expected 404, got %d\n...")  // проверяет 400, но пишет "expected 404"
}
```

Название теста, проверяемый статус и ожидаемое сообщение все не совпадают друг с другом. Скорее всего это скопированный тест, который не доделали.

---

## ХОРОШО

- Интеграционные тесты с in-memory SQLite. Тесты запускаются без реального сервера, покрывают happy path и ошибки.
- `domain/errors.go` со sentinel-ошибками. `HandleError` в utilities централизует маппинг ошибок в HTTP-статусы - это правильный подход.
- `math/big.Rat` для вычислений обменных курсов. Точность не теряется при делении.
- Миграции с timestamp-именами и отдельными `.down.sql` файлами.
- `rows.Err()` проверяется после цикла в `CurrencyRepository.GetCurrencies`. Редко кто об этом помнит.
- Маршруты вынесены в отдельный `routes.go`, `main.go` чистый.
- `db.Ping()` после открытия соединения.
- Три стратегии конвертации в `ExchangeService`: прямой курс, обратный, и через USD.

---

## ЗАМЕЧАНИЯ

### 1. `CurrencyService` содержит неиспользуемое поле

```go
type CurrencyService struct {
    exchangeRateRepository *repositories.ExchangeRateRepository  // нигде не используется
    currencyRepository     *repositories.CurrencyRepository
}
```

`exchangeRateRepository` ни в одном методе не вызывается. Убери или используй.

### 2. Два одинаковых типа `ErrorResponse`

```go
// domain/errors.go
type ErrorResponse struct {
    Message string  // нет json-тега
}

// utilities/utilities.go
type ErrorResponse struct {
    Message string `json:"message"`  // есть json-тег
}
```

`WriteError` кодирует `utilities.ErrorResponse` с тегом `json:"message"`, JSON получается `{"message": "..."}`. Тесты декодируют в `domain.ErrorResponse` без тега - работает только из-за case-insensitive матчинга в Go json-декодере. Один тип путает, второй лишний. Оставь один с тегом.

### 3. `FormatRat` теряет точность через `float64`

```go
func FormatRat(r *big.Rat) string {
    f, _ := r.Float64()           // float64 теряет точность при больших числах
    return fmt.Sprintf("%.2f", f)
}
```

Смысл `big.Rat` пропадает на последнем шаге. Используй `r.FloatString(2)` - он форматирует без промежуточного float64.

### 4. `CurrencyResponse.ID` - строка вместо числа

```go
type CurrencyResponse struct {
    ID string `json:"id"`  // "id": "1" вместо "id": 1
    ...
}
```

ID в JSON-API принято отдавать числом. Из-за этого в сервисе стоит `strconv.Itoa(currency.ID)` при каждом создании DTO. Поменяй тип на `int` и убери конвертации.

### 5. `getCurrencies` не использует `HandleError`

```go
func (c *CurrencyController) getCurrencies(w http.ResponseWriter) {
    currencies, err := c.service.GetCurrencies()
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)  // обходит HandleError
        return
    }
```

Все остальные методы используют `utilities.HandleError`. Этот один обходит централизованную обработку и всегда возвращает 500.

### 6. `GetCurrencyById` возвращает `sql.ErrNoRows` вместо доменной ошибки

```go
// GetCurrencyByCode - правильно:
if errors.Is(err, sql.ErrNoRows) {
    return domain.Currency{}, domain.ErrCurrencyNotFound
}

// GetCurrencyById - неправильно:
if err == sql.ErrNoRows {
    return domain.Currency{}, sql.ErrNoRows  // сырая ошибка из драйвера
}
```

Если `GetCurrencyById` вызовут через контроллер, `sql.ErrNoRows` попадёт в `HandleError` в `default` ветку и вернёт 500 вместо 404.

### 7. Роуты используют switch вместо Go 1.22 pattern routing

```go
func (c *CurrencyController) HandleCurrencies(w http.ResponseWriter, r *http.Request) {
    switch r.Method {
    case "GET":
        c.getCurrencies(w)
    case "POST":
        c.addCurrency(w, r)
    }
}
```

Начиная с Go 1.22, `http.ServeMux` поддерживает метод прямо в паттерне: `"GET /currencies"`, `"POST /currencies"`. Switch внутри обработчика при этом не нужен, код становится проще.

### 8. Порт сервера захардкожен

```go
// internal/server/server.go
return http.ListenAndServe(":8080", s.mux)
```

Порт нельзя изменить без перекомпиляции. Читай его из переменной окружения через конфиг, как уже сделано с DSN базы данных:

```go
addr := cfg.Addr  // например, ":8080" из SERVER_ADDR
return http.ListenAndServe(addr, s.mux)
```

---

## АРХИТЕКТУРА

Разделение на слои хорошее: domain, repositories, services, controllers, dto, utilities. `test_utilities` для тестов - отдельно, не мешает основному коду.

Те же две проблемы, что и в других проектах на Go:

**Сервисы зависят от конкретных типов репозиториев.** Для текущего размера проекта это ок, но когда дойдёшь до юнит-тестов отдельно от интеграционных - придётся вводить интерфейсы.

**`utilities` делает слишком много.** Middleware для JSON, парсинг URL, централизованная обработка ошибок, форматирование `big.Rat` - всё в одном пакете. При росте проекта это превратится в свалку. Уже сейчас `HandleError` логичнее лежал бы в пакете `controllers` или отдельном `apierrors`.

---

## Итог

Уровень заметно выше предыдущих проектов в этом треке: есть тесты, миграции, domain errors, централизованная обработка ошибок, `big.Rat`. Все 8 эндпоинтов по спецификации присутствуют, POST возвращает 201. Несколько настоящих багов: утечка соединения и пропущенный `rows.Err()` в `GetExchangeRates`, неправильный порядок проверок в `UpdateExchangeRate`, игнорируемая валидация в `GetExchange`. Часть ошибок отдаётся в plain text вместо JSON, нарушая спецификацию. Тест с неверным названием и перепутанными ожиданиями нужно исправить. В целом хорошая база, доработать детали.
