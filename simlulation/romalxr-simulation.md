# Ревью Simulation
Проект: https://github.com/romalxr/simulation-go

Терминальная симуляция экосистемы на ходах. Пакеты разнесены чисто (`action`, `entity`, `world`, `pathFinding`, `position`, корневой `main`), BFS лежит отдельно, есть полноценный `Creature`-интерфейс, кулдаун-система задаёт скорость, у `Simulation` есть `Start`/`NextTurn`/`Pause`/`Stop`. ТЗ выполнено почти буквально. Симуляция запускается, читает команды `p`/`q` со стдин в отдельной горутине, рисует поле эмодзи.

<img src="https://i.pinimg.com/736x/b2/b8/58/b2b8581334fc853e0416ba14e704f8e7.jpg" width="400"/>

---

## СООТВЕТСТВИЕ ТЗ

- ✅ Все статичные объекты из ТЗ на месте: `Grass`, `Rock`, `Tree`.
- ✅ `Occupier` интерфейс (`Type`, `Symbol`, `Position`, `SetPosition`).
- ✅ `Creature` интерфейс из ТЗ: `Occupier` + `Speed` + `Hp`/`SetHp` + `Cooldown`/`SetCooldown`.
- ✅ Скорость существ реализована через кулдаун: после действия выставляется `Cooldown = Speed`, каждый ход `UpdateCooldownAction` уменьшает его. Чем меньше Speed, тем чаще существо ходит.
- ✅ `Action` интерфейс, `initActions` и `turnActions` массивы, поведение в экшенах.
- ✅ BFS в отдельном пакете `pathFinding/bfs.go` (ТЗ это прямо требует).
- ✅ У `Simulation` отдельные методы `Start`, `NextTurn`, `Pause`, `Stop`.
- ✅ Сетка N×M, одна сущность на клетку, проверка обоих концов в `MoveTile`.
- ✅ Конструкторы для всех сущностей и для мира (`NewWorldMap`, `NewSimulation`).
- ⚠ ТЗ требует композицию через встраивание, но `Position` лежит обычным полем. Поэтому `Position()` и `SetPosition()` повторяются пятью одинаковыми реализациями по сущностям. При embedding эти методы пришли бы автоматически через продвижение.

---

## НЕДОСТАТКИ РЕАЛИЗАЦИИ

- **Рендер транспонирован: X идёт по строкам, Y по столбцам** ([renderer.go:30](simulation-go/renderer.go#L30)):

```go
for x := 0; x < width; x++ {
    for y := 0; y < height; y++ {
        occupier := r.worldMap.GetTile(position.Position{X: x, Y: y})
        // ...
    }
    fmt.Println("")  // конец строки после прохода по Y
}
```

Мир создаётся как `NewWorldMap(10, 20)`: width=10, height=20. По смыслу X это столбцы (горизонталь), Y это строки (вертикаль). В рендере наоборот: внешний цикл по X печатает строки, внутренний по Y печатает колонки. В коде BFS направления подписаны как `{0,-1} up`, то есть "вверх" означает Y--. На экране же это будет "влево", потому что Y идёт по горизонтали. Сетка не падает, всё рисуется, но координатная система перевёрнута. Правильно:

```go
for y := 0; y < height; y++ {
    for x := 0; x < width; x++ {
        // ...
    }
    fmt.Println("")
}
```

- **Потенциальный вечный цикл в `SpawnAction`** ([spawnAction.go:27](simulation-go/action/spawnAction.go#L27)):

```go
placed := 0
for placed < a.count {
    if pos := wm.GetRandomEmptyPos(); pos != nil {
        wm.SetTile(*pos, a.factory(*pos))
        placed++
    }
}
```

`GetRandomEmptyPos` делает 100 случайных попыток найти пустую клетку, иначе возвращает `nil`. Если карта почти заполнена, `nil` приходит часто, `placed` не увеличивается, внешний `for placed < a.count` крутится бесконечно. В текущем `main` это латентно: на старте мир пустой, 33 сущности на 200 клеток ставятся легко. Контракт всё равно хрупкий. Минимум добавь счётчик попыток и `break` после фейлов, либо при `nil` выходи из цикла и логируй, сколько успел поставить.

- **`SpawnAction.Execute` мутирует `a.count`** ([spawnAction.go:22-24](simulation-go/action/spawnAction.go#L22)):

```go
if a.count > totalCells {
    a.count = totalCells
}
```

Если действие вызвать второй раз (например, переиспользовать как turn action), оно увидит изменённое значение. Сейчас не стреляет, потому что `SpawnAction` используется только в `initActions`. Лучше через локальную переменную: `count := min(a.count, totalCells)`.

- **`MoveTile` возвращает `error`, но во всех вызовах он игнорируется** ([seekMoveAction.go:43](simulation-go/action/seekMoveAction.go#L43), [randomMoveAction.go:50](simulation-go/action/randomMoveAction.go#L50)):

```go
wm.MoveTile(oldPos, *nextStep)   // err игнор
```

В `RandomMoveAction` это безопасно, `available` только что был отфильтрован по `IsEmpty`. В `SeekMoveAction` хуже: BFS может вернуть клетку самой цели, если она в одной клетке от существа (см. ниже), тогда `MoveTile` отдаёт ошибку, существо стоит на месте, и никто об этом не знает. Хотя бы логируй ошибку, как остальные экшены.

---

## ХОРОШО

- Полная реализация интерфейса `Creature` из ТЗ.
- BFS вынесен в отдельный пакет `pathFinding`, не висит внутри `MoveAction`. ТЗ это прямо просит.
- Реализованы `Pause()`/`Stop()` и команды `p`/`q` через горутину со `stdin`. Через `sync/atomic.Bool` для конкурентного доступа из горутины и основного цикла.
- `WorldMapView` интерфейс, рендер зависит только от того, что ему нужно (`GetWidth`, `GetHeight`, `GetTile`). Чистая interface segregation.
- `MoveTile` проверяет оба конца: и что источник занят, и что цель свободна, и возвращает осмысленную ошибку.
- `IsValid` корректный: `pos.X >= 0 && pos.X < wm.width` (без off-by-one).
- `NewWorldMap` обязательный конструктор, `tiles` гарантированно `make()`-нут, состояние сразу валидное.
- Кулдаун-модель скорости. Вместо явных циклов "шагнул N раз за ход" используется один универсальный кулдаун, чисто и однообразно по всем экшенам.
- `RespawnAction` через вероятность, а не порог `< minGrass`. Естественнее: каждый ход с вероятностью X появляется трава, с вероятностью Y травоядное. Не нужно высчитывать, "когда уже мало".
- 4 направления вынесены в локальный слайс `directions := []Position{...}`, а не четыре отдельных `if`.
- `debugMode` гейтит логи: при `false` они уходят в `io.Discard` и не мусорят в выводе. При включении всё попадает в файл.
- Каждая сущность реализована своим конструктором (`NewHerbivore`, `NewPredator`, ...), все возвращают `*Type`, везде одинаково.
- `pathfinding.NextStepToTarget` хранит `firstStep` не как `cameFrom`-цепочку, а сразу как "первый шаг для текущей клетки". Реконструкция пути сводится к одному `firstStep[current]`. Быстрее и проще, чем стандартный backtrack.

---

## ЗАМЕЧАНИЯ

### 1. Dot imports в `world/worldMap.go`

```go
import (
    "errors"
    "math/rand"
    . "simulation/entity"
    . "simulation/position"
)
```

Точечные импорты считаются плохим стилем в Go: читателю непонятно, откуда взялись `Position`, `Occupier`. Когда пакетов станет больше, конфликты имён будут вылезать. Пиши обычные импорты:

```go
import (
    "simulation/entity"
    "simulation/position"
)
// ...
type WorldMap struct {
    tiles map[position.Position]entity.Occupier
    // ...
}
```

### 2. `entityType` хранится в каждой сущности как поле

```go
type Grass struct {
    entityType EntityType
    position   Position
}

func NewGrass(pos Position) *Grass {
    return &Grass{entityType: TypeGrass, position: pos}
}

func (g *Grass) Type() EntityType {
    return g.entityType
}
```

Тип уже статически известен по Go-типу. Хранить его в каждом экземпляре бесполезно (плюс память на каждую траву). Достаточно:

```go
type Grass struct {
    position Position
}

func (g *Grass) Type() EntityType { return TypeGrass }
```

То же самое в `Rock`, `Tree`, `Herbivore`, `Predator`.

### 3. Дублирование `Position()` и `SetPosition()` в каждой сущности

Все пять сущностей имеют одинаковый код:

```go
func (g *Grass) Position() Position           { return g.position }
func (g *Grass) SetPosition(pos Position)     { g.position = pos }
```

Через embedding пишется один раз. Например, базовая структура с позицией:

```go
type entityBase struct {
    position Position
}

func (e *entityBase) Position() Position          { return e.position }
func (e *entityBase) SetPosition(pos Position)    { e.position = pos }

type Grass struct {
    entityBase
}
```

Тогда у `Grass` сразу есть `Position()`/`SetPosition()` через продвижение методов. И ТЗ говорит ровно про это: "Embedding (not class hierarchy)".

### 4. Магические числа в коде и в `main`

```go
speed := 2 + rand.Intn(4)               // simulation.go
return entity.NewHerbivore(pos, speed, 50)
return entity.NewPredator(pos, speed, 70, 20)
// ...
action.NewRespawnAction(10, 5)
// ...
pathfinding.NextStepToTarget(wm, oldPos, targetType, 5)
// ...
time.Sleep(500 * time.Millisecond)
maxAttempts := 100  // worldMap.go GetRandomEmptyPos
```

Стартовые HP, скорости, дальность BFS, частота тика, лимит попыток рандомного поиска размазаны магическими литералами. ТЗ отдельным пунктом отмечает "параметры для балансировки". Вынеси в один файл констант (`internal/balance.go` или просто `balance.go` в корне), будет видно всю физику мира на одной странице.

### 5. `Predator` не может умереть

В `Predator` есть `Hp()` и `SetHp()`, но никто никогда не вызывает `SetHp` для уменьшения. Только `Herbivore` получает урон в `AttackAction`. Если это сознательно (хищники бессмертны, балансируем через голод), то поле `hitPoints` у Predator избыточно. Если нет, то жизненный цикл хищника просто не реализован. Голода у хищника тоже нет: атаковал и ладно.

### 6. SeekMove + BFS: контракт первого шага сломан, когда цель соседняя

`NextStepToTarget` возвращает первый шаг к цели. Если цель в одной клетке от существа, "первый шаг" это сама клетка цели. В `SeekMoveAction` тогда вызывается `wm.MoveTile(oldPos, *nextStep)`, где `*nextStep` занят целью, `MoveTile` отдаёт ошибку "target position is occupied", существо стоит на месте.

В текущей сборке это маскируется порядком экшенов: `EatGrassAction` и `AttackAction` идут до `SeekMoveAction`. Если травоядное рядом с травой, оно его съест в EatGrass, кулдаун встаёт, SeekMove пропускает. Если хищник рядом с травоядным, атакует, кулдаун встаёт, SeekMove пропускает. На практике в плохую ветку почти не попадаем. Но контракт BFS всё равно сломан: метод обещает "следующий шаг к цели", а возвращает клетку цели.

Два варианта починки:

- BFS возвращает клетку, в которую можно зайти (то есть до цели, а не саму цель).
- Или вызывающий код проверяет: если `*nextStep` это и есть цель, не двигаться (раз уже соседи), пусть Attack/EatGrass сработает в следующем такте.

### 7. `EatGrassAction` молчит, остальные экшены логируют

```go
// attackAction.go:
log.Printf("AttackAction: Существо %s %p атаковало", ...)
log.Printf("AttackAction: Существо %s %p не атаковало", ...)

// randomMoveAction.go, seekMoveAction.go: тоже логируют

// eatGrassAction.go:
// никаких логов вообще
```

Когда `debugMode = true`, в логе видно действия всех существ кроме поедания травы. Непоследовательно. Либо везде логируй, либо нигде.

### 8. Опечатка "след шаг" вместо "следующий шаг"

```go
log.Printf("RandomMoveAction: Существо %s %p нашли след шаг", ...)
log.Printf("SeekMoveAction:   Существо %s %p не нашли след шаг", ...)
```

Везде в логах одно и то же сокращение. Если это сознательно, ок, но выглядит как опечатка. Плюс "нашли" во множественном числе при единственном существе. Лучше: "нашёл следующий шаг" / "не нашёл следующий шаг".

### 9. `handleCommands` без выхода

```go
func (s *Simulation) handleCommands() {
    var cmd string
    for {
        fmt.Scanln(&cmd)
        switch cmd { ... }
    }
}
```

После `Stop()` основной цикл выходит из `Start`, `main` возвращается, процесс заканчивается. Горутина умирает с процессом. На практике ок, но если `stdin` закрыли (например, пайпом), `fmt.Scanln` сразу возвращается с ошибкой, и цикл становится busy-loop, который выжигает ядро до выхода из `Start`. Добавь проверку ошибки и завершайся при EOF:

```go
for {
    if _, err := fmt.Scanln(&cmd); err != nil {
        return
    }
    // ...
}
```

### 10. `Pause` гонка между `Load` и `Store`

```go
func (s *Simulation) Pause() {
    s.isPaused.Store(!s.isPaused.Load())
}
```

Между `Load` и `Store` другая горутина могла бы изменить значение. Сейчас `Pause` вызывает только `handleCommands`, и он один, так что не стреляет. Но если когда-то появится второй источник команд (например, HTTP-эндпоинт), будет неприятный гейзенбаг. Аккуратнее через CAS-цикл:

```go
for {
    old := s.isPaused.Load()
    if s.isPaused.CompareAndSwap(old, !old) { break }
}
```

Или проще через `sync.Mutex` и обычный `bool`:

```go
type Simulation struct {
    // ...
    mu       sync.Mutex
    isPaused bool
}

func (s *Simulation) Pause() {
    s.mu.Lock()
    s.isPaused = !s.isPaused
    s.mu.Unlock()
}

func (s *Simulation) IsPaused() bool {
    s.mu.Lock()
    defer s.mu.Unlock()
    return s.isPaused
}
```

Мьютекс читается проще, чем CAS-цикл. `atomic.Bool` оправдан, когда операция действительно атомарная (один `Load` или один `Store`), но "прочитать, инвертировать, записать" это уже три операции, и тут мьютекс уместнее.

### 11. Pre-allocate ёмкости там, где это бесплатно

```go
nextLevel := []position.Position{}   // bfs.go
var available []position.Position    // attack/eat/move actions
```

Слайсы растут через `append`. Для соседей размер заранее известен (≤ 4 направлений), можно `make([]Position, 0, 4)`. Для `nextLevel` ёмкость пропорциональна предыдущему уровню. Минимально, но если решишь профайлить, стартовая точка.

### 12. `debugMode` это переменная пакета, не флаг

```go
var debugMode = false
```

Чтобы включить логирование, нужно пересобрать проект. Прокинь через `flag.BoolVar(&debugMode, "debug", false, ...)`, тогда `go run . --debug` или собранный бинарь с флагом включал бы логи.

### 13. `GetAll` возвращает в случайном порядке

`map`-итерация в Go не детерминирована. Каждый ход существа обходятся в разном порядке. С точки зрения честности симуляции это даже плюс. Но если когда-то понадобятся юнит-тесты с воспроизводимым поведением, это сразу проблема. Имей в виду, для тестов потом отсортируй по `Position` или вынеси итерацию в метод, который сортирует.

---

## АРХИТЕКТУРА

ТЗ выполнено почти буквально: `Creature` интерфейс, BFS в отдельном пакете, `Pause/Start/NextTurn` разделены, фабрики через замыкания в `SpawnAction`, конструкторы у всего, что должно их иметь.

Что можно подтянуть:

**1. Embedding вместо повторяющегося `Position` поля.** Сейчас каждый из пяти типов сущностей содержит `position Position` и пару `Position()`/`SetPosition()`. Базовая структура с embedding (см. замечание 3) уберёт около 30 строк дубля.

**2. Хранить тип в `entityType` поле не нужно.** Type определяется Go-типом, метод `Type()` может вернуть константу напрямую. См. замечание 2.

**3. Балансные параметры в одном месте.** См. замечание 4. Сейчас они в `main`, в actions, в worldMap, в pathfinding.

**4. Жизненный цикл хищника.** HP/SetHp у Predator есть, но ничто не наносит ему урон, нет голода. Если хочется живой экосистемы, либо убирай поля, либо добавляй `StarveAction` или урон в `AttackAction` (например, травоядное может лягнуть).

**5. Юнит-тесты.** `pathfinding.NextStepToTarget`, `WorldMap.MoveTile`, `UpdateCooldownAction` это чистая логика без I/O, удобные кандидаты на покрытие.

---

## Итог

ТЗ выполнено почти буквально: интерфейс `Creature` из ТЗ есть, BFS в отдельном пакете, `Pause/Start/NextTurn` разделены, `Tree` реализован, скорость через кулдаун, `WorldMapView` для рендера, валидация в `MoveTile`, конструкторы у всего. Из реальных багов: транспонированный рендер (X и Y перепутаны местами в циклах), потенциально вечный цикл в `SpawnAction` при заполненной карте, сломанный контракт BFS для случая "цель в соседней клетке" (маскируется порядком экшенов). Из мелочи: dot-imports, `entityType` хранится зря, дубль `Position()`/`SetPosition()` в каждой сущности (просится embedding), магические числа, `Predator` не умирает, `Pause` без CAS. После починки рендера и спавна, плюс лёгкой шлифовки сущностей, будет крепкая учебная реализация.
