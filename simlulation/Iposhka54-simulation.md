# Ревью Simulation

Проект: https://github.com/Iposhka54/simulation-go
Автор: Iposhka54

[Ревью сделано в рамках учебной подписки](https://zhukovsd.it/services/student-subscription/)

<img src="https://i.pinimg.com/1200x/58/ed/17/58ed17ba4599c2522a74a8ec57df2b9c.jpg" width="400"/>

Пошаговая симуляция экосистемы в терминале. Код разложен по пакетам (`cmd`, `internal/entity`, `internal/game`), сущности на встраивании, за интерфейсами `Entity`, `Creature`, `Action`, `Renderer`, `GlyphSet`. BFS написан руками, скорость существ работает, есть пауза и нормальное завершение через `context`. Собирается, `go vet` чистый. Структура взрослая. Основные претензии не к структуре, а к самой логике симуляции, плюс гонка на флагах, пара ООП-привычек из Java и немного мёртвого кода. Ниже подробно, с примерами исправлений.

---

## СООТВЕТСТВИЕ ТЗ

- ✅ Сетка N×M, одна сущность на клетку (`World` с проверкой `IsEmpty` в `PlaceEntity`).
- ✅ Композиция через встраивание: `BaseEntity` → `BaseCreature` → `Herbivore` → `Rabbit`, `StaticEntity` → `Grass`/`Rock`/`Tree`.
- ✅ Интерфейсы вместо наследования: `Entity`, `Creature`, `Action`, `Renderer`, `GlyphSet`.
- ✅ `Action` интерфейс, отдельные `initActions` (`SpawnAction`) и `turnActions` (`MoveAction`).
- ✅ BFS реализован руками ([path_finder.go:17](../simulation-go/internal/game/path/path_finder.go#L17)) с восстановлением пути по `parents`.
- ✅ Скорость существ работает: `moveAlongPath` шагает на `speed` клеток вдоль пути (этого часто не делают).
- ✅ У `Simulation` выделены `Start`, `Pause`, `Resume`, `Stop`, `nextTurn`. Пауза есть.
- ✅ Травоядное ест траву, хищник атакует травоядное с уроном и убирает труп, рендер после каждого хода.
- ✅ Все статичные объекты на месте: трава, камень, дерево.
- ⚠️ ТЗ просит "умные сервисы + глупые данные", а вышло наоборот: всю работу делает существо (`PerformMove` сам ищет еду, строит путь, ходит), а `MoveAction` просто дёргает `cr.MakeMove`. Подробнее в АРХИТЕКТУРЕ.
- ❌ Симуляция никуда не идёт: голода нет, трава не отрастает, размножения нет. Мир только беднеет (см. НЕДОСТАТКИ).

---

## НЕДОСТАТКИ РЕАЛИЗАЦИИ

### 1. Хищники бессмертны, а HP и поедание травы ни на что не влияют

HP падает только в `Predator.EatAdjacentFood` через `prey.TakeDamage`. Самому волку урон не наносит никто, голода у него нет. Кролик ест траву, но `EatAdjacentFood` её просто удаляет ([herbivore.go:33](../simulation-go/internal/entity/herbivore/herbivore.go#L33)): HP от еды не растёт, без еды не падает. Получается, HP кролика двигается, только когда его кусает волк. Трава не отрастает, никто не плодится, и мир едет в одну сторону: трава кончилась, кролики кончились, дальше волки до скончания веков ходят по пустому полю. Ни равновесия, ни конца.

Минимально это лечится голодом и отрастанием травы. Голод: каждый ход существо теряет HP, еда восстанавливает.

```go
// creature.go
const hungerPerTurn = 1

func (bc *BaseCreature) PerformMove(c Creature, w *world.World) error {
    bc.TakeDamage(hungerPerTurn) // голод каждый ход
    if !bc.IsAlive() {
        return bc.Die(w)
    }
    // ... как было
}

// herbivore.go: при поедании травы восстанавливаем HP
func (h *Herbivore) EatAdjacentFood(w *world.World) (bool, error) {
    foodPosition, exists, err := h.findAdjacentFood(w)
    if err != nil || !exists {
        return false, err
    }
    if err = w.RemoveEntity(foodPosition); err != nil {
        return false, err
    }
    h.Heal(grassNutrition) // <- HP теперь имеет смысл
    return true, nil
}
```

Отрастание травы это просто ещё один `Action` в `turnActions`, по аналогии со `SpawnAction`:

```go
type ReplenishGrassAction struct{ Chance float64 }

func (a *ReplenishGrassAction) Execute(w *world.World) {
    // с некоторой вероятностью посадить траву на свободную клетку
}
```

И в `main.go`:

```go
turnActions := []action.Action{
    &action.MoveAction{},
    &action.ReplenishGrassAction{Chance: 0.1},
}
```

### 2. Гонка данных на флагах `running` и `paused`

`Start` крутится в своей горутине и пишет `s.paused`/`s.running` ([simulation.go:41](../simulation-go/internal/game/simulation/simulation.go#L41)), а `Pause`/`Resume`/`Stop` читают эти же поля из горутины ввода. Синхронизации нет:

```go
func (s *Simulation) Pause() {
    if s.running && !s.paused {   // читается из другой горутины
        s.pauseChan <- struct{}{}
    }
}
```

`go vet` это пропускает, `go test -race` ругнулся бы. Самый простой фикс это мьютекс на доступ к флагам:

```go
type Simulation struct {
    // ...
    mu      sync.Mutex
    running bool
    paused  bool
}

func (s *Simulation) Pause() {
    s.mu.Lock()
    canPause := s.running && !s.paused
    s.mu.Unlock()
    if canPause {
        s.pauseChan <- struct{}{}
    }
}
```

Но чище вообще не выносить состояние наружу, а держать его внутри цикла `Start` и управлять только через каналы (см. п.3, там обе беды лечатся разом).

### 3. `Stop` может уронить программу повторным закрытием канала

`close(s.stopChan)` прикрыт только проверкой `if !s.running`, но `running` меняется в другой горутине, так что на проверку полагаться нельзя. Два `Stop` подряд дадут панику `close of closed channel`. Точечно лечится `sync.Once`:

```go
type Simulation struct {
    // ...
    stopOnce sync.Once
}

func (s *Simulation) Stop() {
    s.stopOnce.Do(func() {
        close(s.stopChan)
    })
}
```

А по-хорошему весь жизненный цикл стоит свести к одному источнику истины: пусть `Start` принимает `context.Context`, тогда не нужны ни `stopChan`, ни флаги `running`/`paused`, ни их синхронизация:

```go
func (s *Simulation) Start(ctx context.Context) {
    ticker := time.NewTicker(s.interval)
    defer ticker.Stop()

    paused := false // локальная переменная, гонки нет в принципе
    for {
        select {
        case <-ctx.Done():
            return
        case <-s.pauseChan:
            paused = !paused
        case <-ticker.C:
            if !paused {
                s.nextTurn()
            }
        }
    }
}
```

Здесь `paused` это локальная переменная горутины, её никто снаружи не читает, поэтому гонки нет конструктивно. Пауза и продолжение это один и тот же сигнал по `pauseChan` (флип флага), а остановка приходит через `ctx`.

### 4. Хищник видит только конкретного `*herbivore.Rabbit`

Проверка добычи завязана на конкретный тип ([predator.go:100](../simulation-go/internal/entity/predator/predator.go#L100)):

```go
func isHerbivore(e entity.Entity) bool {
    _, ok := e.(*herbivore.Rabbit)
    return ok
}
```

А ТЗ как раз про интерфейсы вместо конкретных типов. Добавишь второго травоядного, и волк его не увидит. Заводим интерфейс-маркер и проверяем через него:

```go
// herbivore/herbivore.go
type Herbivore interface {
    creature.Creature
    isHerbivore() // приватный метод-маркер, чтобы хищники не путались
}

func (h *Herbivore) isHerbivore() {}
```

```go
// predator.go
func isPrey(e entity.Entity) bool {
    _, ok := e.(herbivore.Herbivore)
    return ok
}
```

Теперь любой новый травоядный, встроивший `*Herbivore`, автоматически становится добычей.

### 5. Горутина ввода не отменяется

```go
for {
    select {
    case <-a.ctx.Done():
        return
    default:
        if scanner.Scan() { ... } // блокируется здесь
    }
}
```

`select` с `default` проверяет `ctx.Done()` ровно один раз и тут же уходит в блокирующий `scanner.Scan()`. Если в это время пришёл сигнал или нажали `q` в другом месте, горутина останется висеть на `Scan` до следующего нажатия Enter. Сейчас спасает только то, что процесс всё равно завершится и утащит горутину с собой, но как приём это неверно. Чтение из stdin честно отменяемым не делается одним select, но как минимум не надо притворяться, что оно отменяемо: проще убрать `default` и читать в цикле, а выход из программы пусть закрывает stdin или просто завершает процесс.

### 6. Ошибка `PlaceEntity` при спавне выбрасывается

[spawn_action.go:68](../simulation-go/internal/game/action/spawn_action.go#L68):

```go
if world.IsEmpty(spawnPosition) {
    world.PlaceEntity(spawnPosition, spawner()) // err проглочен
    placed++
}
```

Позиция перед этим проверена на `IsEmpty`, так что сейчас не стреляет, но глотать ошибку всё равно не стоит, иначе при будущих правках баг пройдёт незаметно:

```go
if world.IsEmpty(spawnPosition) {
    if err := world.PlaceEntity(spawnPosition, spawner()); err != nil {
        log.Printf("spawn failed at %s: %v", spawnPosition.String(), err)
        continue
    }
    placed++
}
```

---

## ЗАМЕЧАНИЯ

### 1. Мёртвые "абстрактные" методы в `BaseCreature` (привычка из Java)

```go
func (bc *BaseCreature) HasAdjacentFood(w *world.World) bool {
    _ = w
    panic("implement in subclasses")
}
```

`BaseCreature` объявляет `HasAdjacentFood`, `EatAdjacentFood`, `IsFoodAdjacent` с паникой внутри, причём сигнатуры даже не совпадают с интерфейсом `Creature` (`bool` против `(bool, error)`). Настоящие методы `Herbivore` и `Predator` их перекрывают, до заглушек выполнение не доходит никогда. Это абстрактные методы в стиле Java, а в Go так не работает: на этапе компиляции они ничего не проверяют и просто висят мёртвым кодом. Их надо удалить целиком. Что метод реализован, и так гарантирует интерфейс в `PerformMove(c Creature, ...)`: если конкретный тип не реализует `Creature`, код не скомпилируется. А если хочется явной проверки на этапе сборки:

```go
var _ creature.Creature = (*Herbivore)(nil)
var _ creature.Creature = (*Predator)(nil)
```

### 2. `context.Context` лежит полем структуры (тоже привычка из ООП)

```go
type App struct {
    simulation *simulation.Simulation
    ctx        context.Context
    cancel     context.CancelFunc
}
```

Тут затащили всё в поля объекта вместо того, чтобы передавать. Документация `context` прямо просит так не делать: контекст не хранят в структуре, а передают первым аргументом в функции, которым он нужен. Поле `cancel` хранить нормально, а `ctx` нет:

```go
func (a *App) Run(ctx context.Context) {
    ctx, cancel := context.WithCancel(ctx)
    defer cancel()

    go a.simulation.Start(ctx)        // см. п.3 НЕДОСТАТКОВ
    go a.handleConsoleInput(ctx, cancel)
    // ...
}
```

Это та же история, что и в п.3 НЕДОСТАТКОВ: контекст в `App` и каналы `stopChan`/`pauseChan` в `Simulation` дублируют один сигнал "стоп". Если `Start(ctx)` начнёт слушать контекст, отдельный `stopChan` исчезнет, и всё управление жизненным циклом сойдётся в одном месте.

### 3. `println` для пользовательского вывода

В `app.go`, `simulation.go` и `console_renderer.go` вывод идёт через встроенный `println` (`println("Turn: ", s.turn)`, `println(output.String())`). Это отладочный билтин: пишет в stderr и без форматирования, для вывода игроку он не предназначен. Меняем на `fmt`:

```go
fmt.Printf("Turn: %d\n", s.turn)
fmt.Print(output.String())
```

### 4. Двойное-тройное сканирование соседей за один ход

`PerformMove` сначала зовёт `HasAdjacentFood`, потом `EatAdjacentFood`, и каждый из них внутри вызывает `findAdjacentFood`, который заново обходит соседние клетки:

```go
exists, err := c.HasAdjacentFood(w) // первый обход соседей
// ...
if exists {
    _, err = c.EatAdjacentFood(w)   // второй обход тех же соседей
    return err
}
```

Достаточно одного обхода. Например, объединить в один метод, который возвращает и факт наличия, и позицию:

```go
foodPos, ok, err := c.FindAdjacentFood(w)
if err != nil {
    return err
}
if ok {
    return c.EatFoodAt(w, foodPos) // позиция уже найдена, второй раз не ищем
}
```

### 5. `Point.String()` на указателе

```go
func (c *Point) String() string { ... }
```

`Point` везде ходит по значению, а `String()` висит на указателе. Из-за этого `Point` не реализует `fmt.Stringer`, и `fmt.Sprintf("%s", pointValue)` выведет структуру, а не аккуратную строку. Сейчас метод срабатывает только потому, что его зовут на адресуемых переменных. Для маленького value-типа ресивер должен быть по значению:

```go
func (c Point) String() string {
    return fmt.Sprintf("Point{X=%d, Y=%d}", c.X, c.Y)
}
```

### 6. `reconstructPath` собирает путь за квадрат

```go
path := []coordinate.Point{goalCord}
for currentCord != startCord {
    currentCord = parents[currentCord]
    path = append([]coordinate.Point{currentCord}, path...) // вставка в начало
}
```

Вставка в начало слайса копирует его целиком на каждом шаге. Проще собрать с конца и развернуть один раз:

```go
var path []coordinate.Point
for cur := goalCord; ; cur = parents[cur] {
    path = append(path, cur)
    if cur == startCord {
        break
    }
}
slices.Reverse(path)
return path
```

### 7. Несогласованный API `World`

`PlaceEntity` и `RemoveEntity` принимают `coordinate.Point`, а `Get` принимает `x, y int`:

```go
func (w *World) Get(x, y int) entity.Entity
func (w *World) PlaceEntity(p coordinate.Point, e entity.Entity) error
```

Из-за этого по коду рассыпаны `w.Get(position.X, position.Y)`. Логичнее единообразно работать с `Point`:

```go
func (w *World) Get(p coordinate.Point) entity.Entity
```

### 8. Текст подсказки не совпадает с командами

Меню показывает `p`, `r`, `q`, а подсказка в `default` пишет "Доступно: pause, resume, quit" ([app.go:74](../simulation-go/internal/app/app.go#L74)). В одном месте буквы, в другом слова. И `Pause`, и `Resume` печатают один эмодзи ▶️, паузе просится ⏸️.

```go
default:
    fmt.Println("Неизвестная команда. Доступно: p (пауза), r (продолжить), q (выход)")
```

### 9. Мёртвые `String()` у статичных объектов и `//todo`

У `Grass`, `Rock`, `Tree` есть `String()` с "grass"/"rock"/"tree", но рендер берёт глиф через тип-свитч в `EmojiGlyphSet`, а не через `String()`. То есть эти методы не зовёт никто. Рядом висит `//todo: move constant in utility class`, ещё один `//todo` в `moveRandomly`. Либо задействовать, либо убрать вместе с todo. Перед сдачей по коду стоит пройтись и вычистить все `//todo` и заглушки.

### 10. `neighbors[:0]` в `FindReachableNeighbors`

```go
filtered := neighbors[:0]
```

Фильтрация на месте поверх свежего слайса из `GetNeighbors`. Работает и без лишних аллокаций, но приём неочевидный: со стороны легко заподозрить алиасинг. Здесь безопасно, но стоило бы хоть комментарий, либо явный новый слайс, раз производительность тут некритична:

```go
filtered := make([]coordinate.Point, 0, len(neighbors))
```

---

## ХОРОШО

- Раскладка `cmd/` + `internal/` по слоям, пакеты названы по смыслу (`entity`, `world`, `path`, `action`, `renderer`).
- Встраивание используется последовательно по всей иерархии сущностей. Ровно то, что просит ТЗ.
- `World` хранит две карты: `entitiesByPoints` и `pointsByEntitiesID`. Поиск и по координате, и по сущности за O(1).
- ID сущностей выдаются атомарно (`atomic.AddUint64`), без гонок.
- BFS корректный: очередь, `visited`, `parents`, восстановление пути. Скорость учитывается при шаге по пути.
- Ошибки оформлены как sentinel-значения с `%w` (`ErrOutOfBounds`, `ErrAlreadyOccupied` и т.д.), есть `validateWorldSize`. По истории коммитов видно, что паники осознанно заменены на ошибки.
- Рендер отвязан от данных через `GlyphSet`, набор глифов можно подменить (эмодзи, ASCII).
- Грейсфул-шатдаун: `context` + перехват `SIGINT`/`SIGTERM`, отдельные горутины на ввод и сигналы.
- В `MoveAction` есть защита: `isStillAtPosition` и `IsAlive` отсеивают существ, которых уже съели или сдвинули в этом ходу ([move_action.go:21](../simulation-go/internal/game/action/move_action.go#L21)).
- Спавн по плотности с потолком попыток (`maxAttempts`), бесконечного цикла на заполненном поле не будет.
- Ошибки ходов логируются через `log.Printf` с ID существа, а не глотаются.

---

## АРХИТЕКТУРА

Каркас сделан правильно: слои разделены, встраивание и интерфейсы на месте, рендер и поиск пути вынесены. Фундамент хороший. Что бы я пересмотрел:

- **Где живёт поведение.** ТЗ просит "умные сервисы + глупые данные", а вышло ровно наоборот. Весь ход (`PerformMove`: проверить еду, поесть, построить путь, шагнуть) сидит в `BaseCreature`, а `MoveAction.Execute` только зовёт `cr.MakeMove`. То есть сущности умные, а сервис пустой. Логику хода стоит перенести в `Action`, а существам оставить состояние и мелкие геттеры (`Hp`, `Speed`, `IsAlive`).

- **Шаблонный метод через интерфейс.** `PerformMove(c Creature, w)` принимает сам себя как интерфейс, чтобы дёрнуть переопределённые методы. Приём в Go рабочий, но именно из-за него и завелись мёртвые заглушки из п.1 ЗАМЕЧАНИЙ. Уедет логика в `Action`, и трюк станет не нужен.

- **Java** Три места выдают перенос ООП-привычек в Go: абстрактные методы с `panic` (заглушки), хранение `context` в структуре `App`, и `Die()`/проверка `!IsAlive` в начале `PerformMove`, которые фактически мертвы (мёртвых существ `MoveAction` уже отфильтровал, а добычу хищник удаляет сразу через `RemoveEntity`). В Go дешевле думать про данные и функции над ними, чем про объекты с состоянием и виртуальными методами.

- **Привязка к конкретным типам.** Волк знает про `*herbivore.Rabbit`, глиф-сет тоже разбирает конкретные типы. У хищника это лечится интерфейсом `Prey`/`Herbivore` (п.4 НЕДОСТАТКОВ). У рендера тип-свитч ок, отрисовка по природе про конкретику.

- **Единый источник жизненного цикла.** Сейчас стоп размазан между `context` в `App` и каналами в `Simulation`. Свести к `Start(ctx)` (п.3 НЕДОСТАТКОВ, п.2 ЗАМЕЧАНИЙ): один сигнал, ноль гонок, меньше полей.

Это не переделка, а доводка.

---

## Итог

Один из самых зрелых проектов: собирается и проходит `vet` чисто, аккуратная раскладка по пакетам, композиция и интерфейсы как в ТЗ, BFS со скоростью, пауза и нормальное завершение. Главное по делу: симуляция никуда не движется (волки бессмертны, трава не отрастает, голода нет), есть гонка на флагах паузы, волк завязан на конкретный тип добычи, и местами проступают ООП-привычки из Java (абстрактные заглушки, контекст в структуре). Всё это правится точечно, фундамент трогать не надо. Сделано сильно.
