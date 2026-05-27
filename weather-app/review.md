# Ревью Weather Viewer
Проект: https://github.com/mom4uk/weather-viewer

[Ревью сделано в рамках учебной подписки](https://zhukovsd.it/services/student-subscription/)

Веб-приложение для просмотра погоды по коллекции локаций. Слоистая архитектура (controllers -> services -> repositories), PostgreSQL для пользователей и локаций, Redis для сессий, шаблоны через `embed`, Docker, CI с линтером и тестами. Проект собирается и запускается, покрытие интеграционными тестами хорошее.

<img src="https://i.pinimg.com/736x/eb/35/fb/eb35fb307451dc9c72f2397359fbfefe.jpg" width="400"/>

---

## НЕДОСТАТКИ РЕАЛИЗАЦИИ

- **IDOR: любой залогиненный пользователь может удалить чужую локацию.** `RemoveLocation` удаляет по `id` без проверки владельца:

```go
// repositories/location_repository.go
func (r *LocationRepository) RemoveLocation(id int) error {
    query := `DELETE FROM locations WHERE id = $1`   // нет user_id
    ...
}

// controllers/location_controller.go - id берётся из path, владелец не проверяется
locationID, err := strconv.Atoi(locationIDStr)
...
c.locationService.RemoveLocation(locationID)
```

Авторизованный пользователь A шлёт `DELETE /removeLocation/{любой id}` и удаляет локацию пользователя B. ТЗ прямо называет это в списке типичных ошибок ("проверки доступа, не дающие лезть к чужим данным по чужому id"). Нужно фильтровать по `user_id` из контекста:

```go
func (r *LocationRepository) RemoveLocation(id, userID int) (int64, error) {
    res, err := r.db.Exec(`DELETE FROM locations WHERE id = $1 AND user_id = $2`, id, userID)
    if err != nil {
        return 0, err
    }
    return res.RowsAffected()   // 0 строк -> 404/403, чужое не трогаем
}
```

- **Тот же IDOR в чтении.** `GetLocation`/`searchLocation/{id}` тоже игнорируют владельца - `SELECT ... FROM locations WHERE id = $1`. Любой может получить координаты и погоду чужой локации по перебору id. Добавить `AND user_id = $2`.

- **Поиск локаций не реализован - `search-results.html` это статический макет.** В шаблоне захардкожен "San Francisco" с пятью фейковыми результатами, и ни один handler его не рендерит. Геокодинга нет вообще. По ТЗ ключевой сценарий такой: ввод названия -> запрос к geocoding API -> список городов-кандидатов с координатами -> пользователь выбирает нужный -> добавление. Здесь форма на главной постит имя сразу в `/addLocation` со скрытыми `latitude=0&longitude=0`, выбора кандидата нет:

```html
<!-- templates/index.html -->
<form action="/addLocation" method="post">
    <input type="text" name="name" placeholder="Enter location">
    <input type="hidden" name="latitude" value="0">
    <input type="hidden" name="longitude" value="0">
```

Следствие - города-тёзки (San Francisco в US/PH/AR/...) различить невозможно, а `UNIQUE (user_id, name)` в миграции вообще запрещает добавить два "San Francisco" одному пользователю. Это прямо противоречит требованию ТЗ "поддержка локаций с одинаковыми названиями, различаемых по координатам".

- **Weather-клиент дергает current-weather по имени, а не geocoding + weather по координатам.** Из-за этого сохранённые в БД `latitude/longitude` фактически не используются - при чтении они перезаписываются координатами из ответа:

```go
// clients/weather_client.go
url := fmt.Sprintf("%s/data/2.5/weather?q=%s&appid=%s", c.URL, location.Name, c.APIKey)

// services/location_service.go - координаты из БД затираются
location.Weather = weather
location.Latitude = weather.Coord.Lat
location.Longitude = weather.Coord.Lon
```

То есть столбцы `latitude/longitude` - мёртвый груз, а погода всегда берётся по строке-имени (всегда первый матч OpenWeather). Правильный путь по ТЗ: `/geo/1.0/direct?q=` для поиска кандидатов, затем `/data/2.5/weather?lat=&lon=` для погоды уже выбранной точки.

- **N+1 синхронных запросов к API на каждой загрузке главной, и одна ошибка роняет всю страницу.** `GetLocations` в цикле делает по HTTP-запросу на каждую локацию, последовательно:

```go
// services/location_service.go
for i := range locations {
    weather, err := s.GetWeather(locations[i])
    if err != nil {
        return []domain.Location{}, err   // упал один город -> упала вся страница
    }
    ...
}
```

При ошибке `PageController.Home` рендерит `error.html`. При этом в шаблоне есть ветка для частичной деградации (`{{if .WeatherError}}`), но поле `Location.WeatherError` нигде не выставляется - мёртвый код. Плюс бесплатный тариф OpenWeather - 60 запросов/мин: 10 локаций = 10 запросов на каждый рефреш. Нужно: продолжать при ошибке отдельного города (заполнять `WeatherError`), а лучше - кэшировать погоду (Redis уже есть) и/или запрашивать конкурентно с ограничением.

- **В лог пишется строка подключения с паролем.** 

```go
// db/db.go
log.Println("DATABASE_URL =", os.Getenv("DATABASE_URL"))  // postgres://user:PASSWORD@...
log.Println("REDIS_URL =", os.Getenv("REDIS_URL"))
```

Креды утекают в логи. Убрать целиком.

- **`InitPostgres` не падает, если БД так и не поднялась.** После 10 неудачных пингов функция всё равно возвращает `db` без ошибки, и приложение стартует с мёртвым соединением:

```go
for i := 0; i < 10; i++ {
    if err := db.Ping(); err == nil {
        break
    }
    time.Sleep(2 * time.Second)
}
return db   // ошибку последнего пинга никто не проверил
```

Нужно сохранить ошибку из цикла и `log.Fatal`, если соединение не установилось.

- **Запись тела после `204 No Content`.** В `RemoveLocation` JSON-путь:

```go
w.WriteHeader(http.StatusNoContent)
if err := json.NewEncoder(w).Encode(nil); err != nil { ... }  // пишет "null\n" в тело 204
```

204 по определению без тела. `Encode(nil)` лишний - просто `WriteHeader` и `return`.

- **Нет middleware логирования запросов.** ТЗ отдельным пунктом требует middleware, логирующий `time / req / resp / code / duration_ms`. В проекте есть только `JSON()` (ставит `Content-Type`) и `Auth()`. Логирования запросов нет.

---

## ХОРОШО

- Слои разделены, зависимости через конструкторы. Бизнес-логика не смешана с SQL.
- `LocationService` зависит от интерфейсов (`interfaces.LocationRepository`, `interfaces.Weather`), а не от конкретных типов - поэтому есть честные юнит-тесты сервиса с фейками без БД и без сети.
- Типизированные доменные ошибки + централизованный маппинг на HTTP-статусы в `apierrors.Response`. Статусы осмысленные: 404 / 409 / 422 / 401 / 502. Это сильно лучше, чем "всё в 400/500".
- Параметризованные SQL-запросы везде, инъекций нет.
- bcrypt для паролей, сравнение через `CompareHashAndPassword`.
- Сессии в Redis с TTL - естественное хранилище для сессий, истечение бесплатно.
- Хорошее покрытие интеграционными тестами (регистрация, логин, логаут, локации, истечение сессии), отдельная тестовая БД, стаб OpenWeather через `httptest`, фикстуры погоды.
- Закрытие `rows`/`resp.Body` с проверкой и логированием ошибки. Проверяется `rows.Err()` после цикла.
- FK с `ON DELETE CASCADE`, `UNIQUE(login)`.
- Инфраструктура: разнесённые compose (base/dev/prod/ci), `healthcheck` + `depends_on: condition: service_healthy`, CI с линтером и тестами, `embed` для шаблонов, multi-stage Dockerfile.

---

## ЗАМЕЧАНИЯ

### 1. Мёртвая миграция `sessions`

Сессии лежат в Redis, но в `db/migrations` есть таблица `sessions` (Postgres), которую приложение не использует - к ней обращается только тестовый `SeedSession`. Вдобавок несоответствие модели: в миграции `expires_at TIMESTAMP`, а в `domain.Session` - `Duration time.Duration`. Читаешь код и не понимаешь, где правда. Удалить миграцию (и переписать тест на Redis), либо явно решить, где хранятся сессии.

### 2. No-op строка в `main.go`

```go
middlewares.Auth(sessionService)   // результат отбрасывается, ничего не делает
```

`Auth` возвращает `Middleware`, который тут же выкидывается. Реальный `Auth` навешивается в `RegisterLocationRoutes`. Эту строку удалить.

### 3. Мёртвый код

- `internal/httputil/httputil.go` - пустой пакет (только `package httputil`).
- `dto.AddLocationRequest` - структура нигде не используется.
- Шаблоны `sign-in-with-errors.html` и `sign-up-with-errors.html` не рендерятся (ошибки показываются прямо в `sign-in.html`/`sign-up.html`).

Всё удалить, чтобы не вводить в заблуждение.

### 4. URL к погоде собирается без экранирования

```go
url := fmt.Sprintf("%s/data/2.5/weather?q=%s&appid=%s", c.URL, location.Name, c.APIKey)
```

`location.Name` подставляется как есть. Для "Санкт-Петербург", "Rio de Janeiro" (пробел), названий с `&` запрос сломается или исказится. Собирай через `url.Values`:

```go
q := url.Values{}
q.Set("q", location.Name)
q.Set("appid", c.APIKey)
reqURL := c.URL + "/data/2.5/weather?" + q.Encode()
```

### 5. Пустой список локаций сериализуется как `null`

```go
// controllers/location_controller.go
var response []dto.LocationResponse   // nil -> "null" в JSON
```

Клиент, ожидающий массив, получит `null`. Инициализируй пустым срезом: `response := make([]dto.LocationResponse, 0)`.

### 6. Cookie сессии без `SameSite`

```go
http.SetCookie(w, &http.Cookie{Name: "session_token", Value: session.ID, Path: "/", HttpOnly: true, Secure: false})
```

`HttpOnly` хорошо, но нет `SameSite`. Поставь хотя бы `SameSite: http.SameSiteLaxMode`, а `Secure` сделай зависящим от окружения (в проде за HTTPS - `true`).

### 7. Время жизни сессии захардкожено

```go
session := domain.Session{ID: uuid.New().String(), UserID: userId, Duration: time.Hour}
```

ТЗ говорит про настраиваемые "N часов". Вынеси в конфиг/env.

### 8. `EXPOSE 8000`, а слушает `8080`

В `Dockerfile` - `EXPOSE 8000`, приложение слушает `:8080` (`server.go`). Несоответствие. Плюс в `docker-compose.prod.yml` у `application` вообще нет проброса портов - непонятно, как до него достучаться снаружи (ТЗ требует доступности на `:8080`). Если предполагается реверс-прокси - это стоит зафиксировать.

### 9. Нет префикса `/api`, имена роутов не RESTful

ТЗ требует общий префикс: "все эндпоинты находятся под общим путём `/api`" (пример - `/api/auth/sign-up`). Конкретные пути локаций ТЗ не задаёт, оставляет на усмотрение разработчика. Здесь же роуты висят прямо в корне без `/api` (`/auth/register`, `/addLocation`, `/getLocations`, `/removeLocation/{id}`) - это нарушение требования про префикс.

Отдельно, уже как стилистический совет (не требование ТЗ): глаголы в путях и `GET /searchLocation/{id}`, который ничего не ищет, а отдаёт погоду сохранённой локации по DB-id. Ресурсные пути читались бы чище: `GET /api/locations`, `POST /api/locations`, `DELETE /api/locations/{id}`.

### 10. Нет защиты от CSRF

Все изменяющие действия - это `POST`-формы на cookie-аутентификации (`/addLocation`, `/removeLocation/{id}`, `/auth/logout`). Без CSRF-токена сторонний сайт может инициировать их от имени залогиненного пользователя. `SameSite=Lax` (см. п.6) закроет основную часть, для полноты - токен в форме.

### 11. `ValidateName` слишком строгий

```go
var nameRegex = regexp.MustCompile(`^[\p{L}\s-]+$`)
```

Режет цифры, апострофы, точки. Реальные названия `Washington D.C.`, `Coeur d'Alene`, `Add-on-23` не пройдут. Раз имя и так приходит из ответа геокодера (а должно - см. недостатки реализации), валидация по белому списку символов скорее вредит.

### 12. `SignIn` (JSON-путь) возвращает пустое тело

`SignUp` отдаёт `dto.UserResponse`, а `SignIn` - просто `200 OK` без тела. Несогласованно; стоит тоже вернуть логин/что-то осмысленное.

### 13. Пакет `internal/utilities` - свалка

```go
// internal/utilities/utilities.go
func ComparePasswords(hashedPassword, password string) error { ... }
func HashPassword(password string) (string, error) { ... }
```

`utilities`/`utils`/`helpers`/`common` - это пакеты-свалки: имя ничего не говорит о содержимом, и со временем туда сваливается всё подряд, образуя зависимости во все стороны. В Go пакет именуют по тому, что он предоставляет. Здесь внутри - работа с паролями, значит пакет `password` (или `hash`/`crypto`):

```go
// internal/password/password.go
package password

func Hash(p string) (string, error) { ... }
func Compare(hash, p string) error { ... }
```

Вызов читается как `password.Hash(...)` - самодокументируемо. То же относится к пустому `internal/httputil` (см. п.3).

### 14. Один эндпоинт отдаёт и JSON, и HTML - контракт размыт

`AddLocation`, `SignUp`, `SignIn`, `RemoveLocation` смотрят на заголовок `Accept` и в зависимости от него ведут себя по-разному: либо REST-ответ (JSON + код), либо HTML-редирект/рендер шаблона.

```go
func wantsHTML(r *http.Request) bool {
    return strings.Contains(r.Header.Get("Accept"), "text/html")
}
// ...
if wantsHTML(r) {
    http.Redirect(w, r, "/", http.StatusSeeOther)   // браузер
    return
}
w.WriteHeader(http.StatusCreated)                    // API
json.NewEncoder(w).Encode(res)
```

У одного URL фактически два разных контракта, выбираемых по `Accept`. Это не REST: тот же `POST /addLocation` для браузера возвращает `303` с `Location: /`, а для API - `201` с JSON-телом. Тестировать и документировать такое тяжело, поведение зависит от заголовка, который легко не прислать. Нужно разнести: серверный рендеринг страниц (формы -> редиректы) живёт под `PageController` на "человеческих" путях, а REST-API под `/api/...` всегда отдаёт JSON. Тогда и `wantsHTML` не нужен.

### 15. Про логаут (отвечаю на комментарий в коде)

```go
// я хз, как мне правильно обрабатывать logout...
```

Контракт логаута - "после вызова сессии у клиента нет". Поэтому cookie надо чистить **всегда**, а не только на успешном пути `DeleteSession`. Сейчас при ошибке удаления из Redis ты редиректишь/возвращаешь 204, не сбросив cookie - пользователь "разлогинен" в UI, но cookie живёт. Порядок: сначала затираем cookie (`MaxAge: -1`), потом пытаемся удалить из хранилища, ошибку удаления - в лог, клиенту всё равно успех. Битый/несуществующий session_id - тоже успех (идемпотентность).

---

## АРХИТЕКТУРА

Основа здоровая: слои разнесены (`controllers -> services -> repositories`, рядом `domain`/`dto`/`clients`/`render`), композиция в `main.go` как composition root, DI через конструкторы. Это уже выше среднего. Ниже - разбор по чистой архитектуре, SOLID и чистому коду, что мешает назвать её "чистой" полностью.

### Чистая архитектура (правило зависимостей)

Правило: зависимости направлены внутрь, в центре - доменные сущности, которые ничего не знают про внешний мир (БД, HTTP, сторонние API). Здесь оно нарушается в нескольких местах.

**`domain.Weather` - это DTO OpenWeatherMap, а не доменная сущность.** В нём `json`-теги, поля `Coord`, `Sys`, `Base`, `Cod`, `Dt` - буквально форма ответа стороннего API:

```go
// internal/domain/weather.go
type Weather struct {
    Coord struct{ Lon, Lat float64 } `json:"coord"`
    Sys   struct{ Country string ... } `json:"sys"`
    Cod   int `json:"cod"`
    ...
}
```

Ядро домена теперь зависит от формата конкретного провайдера. Сменишь OpenWeather на другой сервис - придётся менять доменную сущность и всё, что её использует. Правильно: клиент парсит ответ в *свой* приватный DTO (`clients.openWeatherResponse`) и мапит его в чистый `domain.Weather` с осмысленными полями (`TemperatureK`, `City`, `Country`, `Lat`, `Lon`). Тогда домен не знает, откуда взялась погода.

**Презентация тоже завязана на формат API.** `render/functions.go` принимает `domain.Weather` и оперирует кодами иконок OpenWeather и Кельвинами:

```go
func weatherIcon(w domain.Weather) string {
    return fmt.Sprintf("https://openweathermap.org/img/wn/%s@4x.png", w.Weather[0].Icon)
}
func temperatureC(kelvin float64) int { return int(math.Round(kelvin - 273.15)) }
```

То есть связь с OpenWeather протекла и в слой шаблонов. После чистого `domain.Weather` это станет `weatherIcon(domain.Weather)` без знания про URL-схему провайдера.

**Поле представления в доменной сущности.** `domain.Location.WeatherError string` - это view-model для шаблона, ему не место в сущности. Заведи отдельную view-структуру в `controllers`/`render`.

**Инварианты проверяются на краю, а не в ядре.** Валидация (`dto.ValidateCredentials`, `ValidateName`, `ValidateId`) лежит в `dto` и зовётся из контроллеров. Сервис принимает уже "как есть" и ничего не проверяет - `AuthService.SignUp` не валидирует логин/пароль, `LocationService.AddLocation` не валидирует имя. Если кто-то вызовет сервис мимо контроллера, инварианты не сработают. Доменные правила ("логин 6–20, латиница") логичнее держать в домене/сервисе, а `dto` оставить транспортной структурой.

### SOLID

**DIP соблюдён только наполовину.** Инвертирована единственная зависимость - `LocationService` -> интерфейсы репозитория/клиента (хорошо, потому и тестируется юнитами). Всё остальное завязано на конкретику:

- `UserService`/`SessionService` зависят от конкретных `*repositories.UserRepository`/`*repositories.SessionRepository`.
- `AuthService` зависит от конкретных `*UserService`/`*SessionService`.
- Контроллеры держат конкретные `*services.X`.

Высокоуровневые модули зависят от низкоуровневых напрямую - это и есть нарушение DIP. Из-за этого auth-логику нельзя протестировать без реальных Postgres/Redis (спасают интеграционные тесты).

При этом `UserService`/`SessionService` - **тонкие прослойки 1-в-1** над репозиторием без логики:

```go
func (s *UserService) CreateUser(login, hash string) (domain.User, error) {
    return s.repo.CreateUser(login, hash)
}
```

И вот тут две проблемы сходятся: прослойка пустая ровно потому, что логика, которой положено быть здесь, уехала на край. Доменные проверки логина/пароля (`ValidateCredentials`) сейчас живут в `dto` и зовутся из контроллера (см. "Инварианты проверяются на краю" выше) - а их естественный дом именно `UserService`/`AuthService`. Перенеси проверку сюда, и прослойка перестанет быть пустой, и инвариант будет соблюдаться при любом вызове, не только из HTTP-хендлера:

```go
func (s *UserService) CreateUser(login, hash string) (domain.User, error) {
    if err := domain.ValidateLogin(login); err != nil { // правило домена - в домене/сервисе
        return domain.User{}, err
    }
    return s.repo.CreateUser(login, hash)
}
```

Тогда выбор такой: либо сервис несёт доменную валидацию (и тогда ему нужен интерфейс репозитория, как у `LocationService`, чтобы юнит-тестить эти правила без БД), либо, если логики действительно не предвидится, прослойку убрать и дать `AuthService` зависеть от интерфейсов репозиториев напрямую. Но раз доменные проверки есть - первый вариант правильнее.

**Интерфейсы лучше держать у потребителя, а не в отдельном пакете `interfaces`.** `LocationRepository` и `Weather` лежат в `internal/interfaces`, хотя используются в `services`. В Go интерфейс объявляют там, где он *потребляется* ("consumer-side interfaces"): пакет описывает ровно тот минимальный контракт, который ему нужен. Отдельный пакет `interfaces` - паттерн из Java/C#; в Go он создаёт обратную зависимость (и реализации, и потребители смотрят в общий пакет) и провоцирует "толстые" интерфейсы. Перенеси в `services`:

```go
// internal/services/location_service.go
package services

type LocationRepository interface {
    GetLocation(id int) (domain.Location, error)
    AddLocation(location domain.Location) (domain.Location, error)
    GetLocations(userID int) ([]domain.Location, error)
    RemoveLocation(id int) error
}

type WeatherClient interface {
    GetWeather(location domain.Location) (domain.Weather, error)
}
```

Репозиторий и клиент про интерфейс не знают - просто удовлетворяют его. Пакет `internal/interfaces` исчезает.

**ISP / "зависим только от того, что используем": у `AuthController` две мёртвые зависимости.** В конструктор инжектятся `userService`, `sessionService`, `authService`, `renderer`, но хендлеры (`SignUp`/`SignIn`/`SignOut`) используют только `authService` и `renderer`:

```go
type AuthController struct {
    userService    *services.UserService    // нигде не используется
    sessionService *services.SessionService // нигде не используется
    authService    *services.AuthService
    renderer       *render.TemplateRenderer
}
```

`userService`/`sessionService` можно выкинуть из контроллера и из вызова `NewAuthController` в `main.go`/`test_app.go`.

**SRP: контроллеры делают слишком много.** `AddLocation` парсит форму, валидирует, в HTML-режиме ходит за геокодом, сохраняет, и отвечает в двух форматах. Хендлер совмещает несколько обязанностей. Связано с двумя пунктами: получение координат размазано между контроллером и сервисом (логика "имя -> координаты" должна жить в сервисе одним методом), и один URL отдаёт и JSON, и HTML (см. замечание #14). Чистое решение: HTML-сценарии - в `PageController` на "человеческих" путях, REST под `/api/...` всегда JSON, content-negotiation (`wantsHTML`) убирается.

**OCP (для контекста, не баг).** `apierrors.Response` - один большой `switch` по всем доменным ошибкам. Каждый новый тип ошибки требует правки этого `switch`. Для текущего размера это нормально и даже удобно (вся карта статусов в одном месте); просто держи в уме, что это точка роста.

### Чистый код

- **Магическая строка `"session_token"` повторяется 6 раз** (оба контроллера, middleware). Опечатаешься в одном месте - отвалится логин, и компилятор не поймает. Вынеси в `const SessionCookieName = "session_token"`.
- **Хрупкое определение "дубликата" по тексту ошибки.** В двух репозиториях:

  ```go
  if strings.Contains(err.Error(), "duplicate key value violates unique constraint") { ... }
  ```

  Сломается при смене локали/версии драйвера. `lib/pq` отдаёт типизированную ошибку - проверяй код:

  ```go
  var pqErr *pq.Error
  if errors.As(err, &pqErr) && pqErr.Code == "23505" { // unique_violation
      return domain.ErrUserAlreadyExists
  }
  ```

- **Анонимные вложенные структуры в `domain.Weather`.** Поля `Weather`, `Coord`, `Main`, `Sys`, `Wind` объявлены как безымянные структуры прямо внутри типа:

  ```go
  Weather []struct {
      ID          int    `json:"id"`
      Main        string `json:"main"`
      Description string `json:"description"`
      Icon        string `json:"icon"`
  } `json:"weather"`
  ```

  На такой тип нельзя сослаться по имени, его нельзя переиспользовать, передать в функцию или замокать отдельно. Цена видна прямо в тестах: чтобы собрать фикстуру в `tests/fixtures/weather.go`, пришлось дословно переписать все эти анонимные структуры целиком - десятки строк копипасты, которые сломаются от любого изменения в домене. Выноси вложенные части в именованные типы (`WeatherCondition`, `Coord`, `Main`, `Wind`), тогда фикстура станет `WeatherCondition{ID: 804, Main: "Clouds", ...}`, а не повтором всего определения. То же относится к остальным безымянным структурам в этом файле.
- **Публичные поля у `LocationService`** (`LocationRepository`, `WeatherClient` - экспортируемые), тогда как у остальных сервисов поля приватные (`repo`). Несогласованно и ломает инкапсуляцию; экспортированы они, судя по всему, чтобы тесты подставляли клиент напрямую - но для этого уже есть `NewTestAppForWeather`/конструктор. Сделай приватными.
- **Именование.** Файл `auth_contoller.go` (опечатка, "contoller"). В интерфейсе `GetLocations(sessionToken int)` параметр назван `sessionToken`, хотя это `userID` - сбивает с толку. `dsl` вместо `dsn` в `db.go`. `nameTrimed` -> `nameTrimmed`.
- **Отладочный вывод в проде.** В `middlewares.Auth`: `fmt.Print("error", err)` - без переноса строки, мимо логгера. Убрать или заменить на `slog`.
- **DRY в тестах.** `NewTestApp` и `NewTestAppForWeather` - почти полная копия проводки приложения. Вынеси общую сборку, отличие (weather-клиент) передавай параметром.
- Мёртвый код (`httputil`, `AddLocationRequest`, неиспользуемые шаблоны и миграция, no-op `middlewares.Auth(...)` в `main`) - см. замечания #2–#4.

---

## КОНКУРЕНТНОСТЬ (где напрашивается concurrency)

Проект почти весь - последовательный I/O, и это нормально для CRUD. Но есть одно место, где параллелизм реально нужен по делу, и оно же - узкое место производительности (см. N+1 в недостатках реализации).

### `GetLocations`: N последовательных запросов к OpenWeather

```go
// services/location_service.go - сейчас
for i := range locations {
    weather, err := s.GetWeather(locations[i])   // блокирующий HTTP, по одному
    if err != nil {
        return []domain.Location{}, err
    }
    locations[i].Weather = weather
    locations[i].Latitude = weather.Coord.Lat
    locations[i].Longitude = weather.Coord.Lon
}
```

10 локаций = 10 HTTP-запросов один за другим. Если каждый ~200 мс, главная грузится ~2 сек на ровном месте. Запросы независимы - их можно пускать параллельно. Идиоматичный инструмент в Go для "много независимых задач, верни первую ошибку" - `errgroup`:

```go
import "golang.org/x/sync/errgroup"

func (s *LocationService) GetLocations(userID int) ([]domain.Location, error) {
    locations, err := s.LocationRepository.GetLocations(userID)
    if err != nil {
        return nil, err
    }

    g, ctx := errgroup.WithContext(context.Background())
    g.SetLimit(8) // бесплатный тариф OpenWeather - 60 req/min, не выстреливаем все разом

    for i := range locations {
        i := i // до Go 1.22; на 1.26 не нужно, но привычка безопасная
        g.Go(func() error {
            weather, err := s.GetWeatherCtx(ctx, locations[i])
            if err != nil {
                return err
            }
            locations[i].Weather = weather          // каждая горутина пишет в СВОЙ индекс -
            locations[i].Latitude = weather.Coord.Lat  // это разные элементы, гонки нет
            locations[i].Longitude = weather.Coord.Lon
            return nil
        })
    }

    if err := g.Wait(); err != nil {
        return nil, err
    }
    return locations, nil
}
```

Ключевые моменты:

- **Гонки нет, хотя пишем в один срез.** Каждая горутина трогает только `locations[i]` со своим `i` - это разные ячейки памяти. Длину/`cap` среза никто не меняет. А вот если бы вы делали `result = append(result, ...)` из горутин - была бы гонка, и понадобился бы мьютекс или сбор через канал.
- **`g.SetLimit(n)` - обязателен здесь.** Без ограничения 100 локаций = 100 одновременных запросов, и OpenWeather упрётся в rate limit (429). `SetLimit` держит не больше n горутин в полёте.
- **`errgroup.WithContext` даёт отмену.** Как только одна задача вернула ошибку, `ctx` отменяется, и остальные запросы (если `GetWeather` принимает `ctx` и зовёт `http.NewRequestWithContext`) прервутся, не доделывая лишнюю работу. Для этого клиенту нужен метод с контекстом - сейчас `GetWeather` контекст не принимает, это стоит добавить заодно.
- `g.Wait()` возвращает **первую** ошибку. Это согласуется с текущим поведением "упал один - вернули ошибку". Если же хочется деградировать частично (показать остальные города, а упавшему проставить `WeatherError` - см. недостатки), то ошибку отдельной локации не возвращай из `g.Go`, а клади в `locations[i].WeatherError` и всегда `return nil`.

### Если хочется именно каналами (для понимания)

`errgroup` под капотом - это `sync.WaitGroup` + `sync.Once` для первой ошибки. То же руками:

```go
type result struct {
    idx     int
    weather domain.Weather
    err     error
}

ch := make(chan result, len(locations))
sem := make(chan struct{}, 8) // семафор-ограничитель, аналог SetLimit

var wg sync.WaitGroup
for i := range locations {
    wg.Add(1)
    go func(i int) {
        defer wg.Done()
        sem <- struct{}{}        // занять слот
        defer func() { <-sem }() // освободить
        w, err := s.GetWeather(locations[i])
        ch <- result{idx: i, weather: w, err: err}
    }(i)
}

go func() { wg.Wait(); close(ch) }() // закрыть канал, когда все отписались

for r := range ch {
    if r.err != nil {
        return nil, r.err
    }
    locations[r.idx].Weather = r.weather
    locations[r.idx].Latitude = r.weather.Coord.Lat
    locations[r.idx].Longitude = r.weather.Coord.Lon
}
```

Это ровно то, что делает `errgroup`, но многословнее и легче ошибиться (забыть `close(ch)` -> дедлок, забыть буфер -> утечка горутин при раннем `return`). В проде бери `errgroup`; каналы тут полезны только чтобы понять, что внутри.

### Где concurrency НЕ нужна

- `InitPostgres` + `InitRedis` в `main` идут последовательно - можно распараллелить, но это разовый старт на пару сотен миллисекунд, выигрыша ноль, читаемость хуже. Не трогать.
- Одиночные `GetLocation`, `AddLocation`, репозиторные методы - один запрос, параллелить нечего.

Вывод: единственное оправданное место - фан-аут запросов погоды в `GetLocations`. Туда - `errgroup` с `SetLimit` и контекстом.

---

## Итог

Заметно сильнее, чем "учебный минимум": чистые слои, интерфейсы и юнит-тесты у локаций, типизированные ошибки с осмысленными HTTP-статусами, Redis-сессии, нормальная инфра и CI. Но есть критичное: **IDOR на удаление/чтение чужих локаций**, и **ключевая фича поиска локаций фактически не сделана** - `search-results.html` это статический макет, геокодинга нет, города-тёзки не различаются, а сохранённые координаты не используются. Плюс утечка пароля БД в логи, N+1 запросов к API с падением всей страницы из-за одной ошибки и отсутствие middleware логирования из ТЗ. Сначала чинить IDOR и поиск/геокодинг, потом остальное.
