# Ревью Simulation
Проект: https://github.com/ETOOOOOOCHAAAAAAAAAAI/Simulation

Терминальная симуляция экосистемы с ходами, BFS для поиска цели, разделением кода по пакетам (`cmd`, `models`, `actions`, `render`) и `Action`-интерфейсом для шагов хода. По структуре близко к ТЗ: есть `Occupier`, `Movable`, `Vulnerable`, фабрики через `SpawnAction`, отдельные `InitActions` и `TurnActions`. Симуляция запускается и идёт бесконечным циклом. Есть баги и слабые места, но архитектура заметно лучше, чем в Hangman.

<img src="https://i.pinimg.com/736x/92/24/82/922482c5f041b9dea6cffa73a360b535.jpg" width="400"/>

---

## СООТВЕТСТВИЕ ТЗ

- ✅ `Occupier` интерфейс с `Type()` и `Pos()`.
- ✅ `Action` интерфейс, `InitActions` и `TurnActions` массивы.
- ✅ BFS реализован руками (`BreadthFirstSearch` в [move_action.go:9](Simulation/internal/actions/move_action.go#L9)).
- ✅ Поведение в `Action`, данные в моделях ("smart services, dumb data").
- ✅ Композиция через встраивание `Position`.
- ✅ Сетка N×M, одна сущность на клетку.
- ✅ Травоядное ест траву, хищник атакует травоядное, рендер после каждого хода.
- ✅ `ReplenishGrassAction`: допфича из ТЗ (поддержание ресурса).
- ✅ Hunger-система: тоже допфича из ТЗ.
- ❌ Препятствие Tree не реализовано.
- ❌ Метода `Pause()` нет, у `Simulation` только бесконечный `Start()`. `NextTurn()` тоже как отдельный метод не выделен, весь цикл закопан в `Start`.
- ❌ Скорость существ (`speed`, клеток за ход) из ТЗ не реализована, все двигаются ровно на одну клетку. Интерфейса `Creature` (Occupier + HP + speed) тоже нет, вместо него `Movable`/`Vulnerable`. Само по себе нормально, но скорость потеряна.

---

## НЕДОСТАТКИ РЕАЛИЗАЦИИ

- **Off-by-one в проверке границ в `WorldMap.Set`** ([map.go:11](Simulation/internal/models/map.go#L11)):

```go
if p.X < 0 || p.X > m.Width || p.Y < 0 || p.Y > m.Height {
    return errors.New("за границу")
}
```

Сетка занимает координаты `0..Width-1` и `0..Height-1`. Условие должно быть `>=`, а не `>`. Сейчас `Set` пропускает позиции `X=Width` и `Y=Height`, клетки, которых на поле не существует. В рендерере (где сравнение правильное, `x < m.Width`) такие сущности просто никогда не отрисуются, останутся призраками в `Entities`.

- **Ошибка от `m.Set` игнорируется везде**. `Set` возвращает `error`, но во всех вызовах ошибку молча выкидывают:

```go
// attack_action.go
m.Set(newPos, models.NewSkull(newPos))   // err игнор

// lifecycle_action.go
m.Set(currentPos, models.NewSkull(currentPos))  // err игнор

// replenish_grass_action.go
m.Set(pos, models.NewGrass(pos))  // err игнор

// spawn_action.go
m.Set(currentPos, entity)  // err игнор
```

Либо обрабатывай ошибку, либо не возвращай её, сейчас она бесполезна. И сообщение `"за границу"` это не имя ошибки, а обрывок фразы. Ошибки в Go принято писать строчными буквами и без многоточий: `errors.New("position out of bounds")`.

- **`WorldMap.Move` молча перезаписывает занятую клетку** ([map.go:18](Simulation/internal/models/map.go#L18)):

```go
func (m *WorldMap) Move(from, to Position) bool {
    // ...
    delete(m.Entities, from)
    movable.SetNewPos(to)
    m.Entities[to] = entity  // если в to кто-то был, он стирается без следа
    return true
}
```

Сейчас вызывающий код проверяет `IsEmpty` сам, но это легко забыть. `Move` должен сам проверять, что `to` свободна (или содержит то, что хищник/травоядное может съесть), и возвращать `false` иначе. К тому же `Move` уже возвращает `bool`, но **ни один вызывающий код этот результат не проверяет**: `m.Move(currentPos, newPos)` везде вызывается как процедура. Либо проверяй успех перемещения, либо убирай возврат.

- **Бесконечный цикл в `ReplenishGrassAction`** ([replenish_grass_action.go:25](Simulation/internal/actions/replenish_grass_action.go#L25)):

```go
for i := 0; i < r.SpawnCount; i++ {
    for {
        x := rand.IntN(m.Width)
        y := rand.IntN(m.Height)
        pos := models.Position{X: x, Y: y}
        if m.IsEmpty(pos) {
            m.Set(pos, models.NewGrass(pos))
            break
        }
    }
}
```

Если поле полностью заполнено (трава + камни + черепа), внутренний `for {}` крутится вечно. Та же проблема в [spawn_action.go:16](Simulation/internal/actions/spawn_action.go#L16). Нужен либо предварительный список пустых клеток (тогда выбираешь случайный из них), либо счётчик попыток с выходом.

---

## ХОРОШО

- Разделение по пакетам: `models`, `actions`, `render`, `cmd`. Явный шаг вперёд по сравнению с Hangman.
- `Action` интерфейс с одним методом `Perform(m *WorldMap)`, все шаги хода единообразны.
- Фабрика через `SpawnAction.Factory FactoryFunc`: `main` декларативно описывает мир без знания внутренностей сущностей.
- `Occupier`, `Movable`, `Vulnerable` разнесены по обязанностям, маленькие, ровно по делу. Хорошая композиция интерфейсов: `Movable` встраивает `Occupier`.
- Встраивание `Position` в `Herbivore`, `Predator`, `Grass`, `Rock`, `Skull` плюс метод `Pos()` на самом `Position`: каждая сущность автоматически удовлетворяет `Occupier` по `Pos()` без копипасты.
- BFS реализован руками с восстановлением пути через `cameFrom`, реконструкция корректна, без off-by-one.
- `math/rand/v2` (новый пакет), без устаревшего `rand.Seed`. Многие до сих пор пишут по старому.
- В `BreadthFirstSearch` есть аккуратный кейс `currentPos != startPos`: если стартовая клетка уже совпадает с целью, не возвращаем её.
- Render через ANSI-escape `\033[H\033[2J` и `time.Sleep`, простая, но рабочая анимация.

---

## ЗАМЕЧАНИЯ

### 1. `whoIsThis` это не имя переменной

В `MoveAction` и `AttackAction` слайс позиций называется `whoIsThis`:

```go
var whoIsThis []models.Position
// ...
whoIsThis = append(whoIsThis, currentPos)
```

Имя ничего не говорит читателю. Это список позиций хищников (в `AttackAction`) или травоядных (в `MoveAction`). Назови соответственно: `predators`, `herbivores` или `positions`.

### 2. Дублирование сбора сущностей по карте

В `AttackAction`, `MoveAction`, `LifeCycleAction`, `ReplenishGrassAction` повторяется один и тот же цикл `for y { for x { ... } }`. Каждый раз с небольшой вариацией: где-то собирают позиции хищников, где-то травоядных, где-то считают траву.

Вынеси в метод карты:

```go
func (m *WorldMap) PositionsOf(t EntityType) []Position { ... }
func (m *WorldMap) CountOf(t EntityType) int { ... }
```

Тогда `AttackAction.Perform` начинается с `predators := m.PositionsOf(models.PredatorType)`, короче и понятнее.

### 3. Type assertion и type switch вместо строки `Type()`

Сейчас в моделях есть и `EntityType` (строковая константа), и интерфейсы (`Movable`, `Vulnerable`). В `LifeCycleAction` идёт двойной type assertion:

```go
if h, ok := entity.(*models.Herbivore); ok { ... }
if p, ok := entity.(*models.Predator); ok { ... }
```

Это можно переписать через интерфейс. У хищника и травоядного одинаковая логика голода и смерти, так что выдели интерфейс `Living` с `Hunger`, `TakeDamage`, `IsDead` и обрабатывай через него:

```go
type Living interface {
    Vulnerable
    Starve()  // -20 голода или -30 HP, если голод 0
}

if living, ok := entity.(Living); ok {
    living.Starve()
    if living.IsDead() { m.Set(currentPos, models.NewSkull(currentPos)) }
}
```

Сейчас `*Predator` и `*Herbivore` это две почти идентичные структуры с одинаковыми методами, разница только в `Type()`. Явный кандидат на общую базу через встраивание (например, `Creature` с полями `HP`, `Hunger`).

### 4. Магические числа повсюду

```go
victim.TakeDamage(50)              // attack_action.go
p.Hunger += 60                     // attack_action.go
h.Hunger += 60                     // move_action.go
h.Hunger -= 20                     // lifecycle_action.go
h.TakeDamage(30)                   // lifecycle_action.go
HP: 100, Hunger: 100               // herbivore.go, predator.go
time.Sleep(1 * time.Second)        // simulation.go
Width: 10, Height: 10              // main.go
```

Балансные параметры из разных файлов размазаны магическими литералами. По ТЗ "балансировка" это отдельный пункт. Вынеси константы в один файл (например, `internal/config/balance.go`), будет видно всю физику мира на одной странице и проще крутить.

### 5. Комментарии-"хлебные крошки"

```go
// сразу же подмечаем чтоб в будущем не приходилось сюда заходить
cameFrom[neighbor] = currentPos // ЗАПИСЫВАЕМ ХЛЕБНУЮ КРОШКУ

if victim, ok := target.(models.Vulnerable); ok { //Type Assertation типо проверка имеет ли тип методы интерфейса
```

Комментарии в стиле "рассказываю сам себе, что тут происходит" это шум. Если код требует пояснения, назови переменные правильно (`cameFrom` уже хорошее имя), если нет, просто убери комментарий. Особенно стрёмно `Type Assertation типо проверка`, это уровень конспекта, такому не место в коде. Оставляй комментарий в одну строку только там, где важно **почему**, а не **что**.

### 6. `String()` методы у моделей не используются

```go
func (g Grass) String() string { return string(g.Type()) }
func (r Rock) String() string  { return string(r.Type()) }
func (h Herbivore) String() string { return string(h.Type()) }
// и т.д.
```

Эти методы нигде не вызываются (рендер делает `entity.Type()` напрямую). Если хочешь, чтобы `fmt.Println(entity)` печатал эмодзи, пусть `String()` возвращает эмодзи. А пока это мёртвый код, удали.

### 7. BFS считает траву проходимой даже для хищника

Фильтр соседей в `BreadthFirstSearch`:

```go
if entity.Type() != targetType && entity.Type() != models.GrassType {
    continue
}
```

Когда BFS ищет травоядное (targetType = `HerbivoreType`), он проходит **сквозь траву**. Само по себе ок, хищник может пробежать по траве. Но `AttackAction` обрабатывает случай "первый шаг это трава":

```go
} else if target.Type() == models.GrassType {
    m.Remove(newPos)
    m.Move(currentPos, newPos)
}
```

То есть если первая клетка пути это трава, хищник удаляет её при движении. Побочный эффект странный: хищник не должен уничтожать траву на пути. Проще не считать траву проходимой для хищника:

```go
// внутри BFS добавить параметр passableTypes []EntityType
```

или сделать BFS принимающим `canEnter(Position) bool`.

### 8. `entity, _ := m.Entities[currentPos]` после уже сделанной проверки

В [attack_action.go:35](Simulation/internal/actions/attack_action.go#L35):

```go
_, exists := m.Entities[currentPos]
if !exists {
    continue
}
// ...
entity, _ := m.Entities[currentPos]  // повторное чтение того же самого
if p, ok := entity.(*models.Predator); ok { ... }
```

В первой проверке `_` отбрасывается, потом снова делается то же чтение. Сохрани значение с первого раза:

```go
entity, exists := m.Entities[currentPos]
if !exists { continue }
if p, ok := entity.(*models.Predator); ok { ... }
```

### 9. Бесконечный `for {}` в `Start` без условия выхода

```go
func Start(s *Simulation) {
    // ...
    for {
        // ...
        time.Sleep(1 * time.Second)
    }
}
```

Никакого способа остановить: ни условия победы (например, "все хищники мертвы"), ни кнопки выхода, ни отслеживания сигнала (`os.Signal`/`SIGINT`). В ТЗ `NextTurn()` и `Pause()` упомянуты как отдельные методы. Минимум, обработка `Ctrl+C`:

```go
ctx, cancel := signal.NotifyContext(context.Background(), os.Interrupt)
defer cancel()
for {
    select {
    case <-ctx.Done(): return
    default:
    }
    // ход
}
```

### 10. `Renderer` это структура без полей

```go
type Renderer struct{}
func (r Renderer) RenderMap(m *models.WorldMap) { ... }
```

Метод не использует ресивер. Раз у `Renderer` нет состояния, это просто функция:

```go
func RenderMap(m *models.WorldMap) { ... }
```

Появятся настройки (цвета, размеры, формат), тогда и сделаешь структуру. Сейчас это пустая обёртка.

### 11. `TurnCount: 1` в `main`

```go
sim := render.Simulation{
    // ...
    TurnCount: 1,
}
```

Странно инициализировать счётчик ходов конкретным значением. Естественнее `0`, а ходы нумеровать с 1 при выводе (`s.TurnCount + 1`), либо инкрементировать **в начале** хода, а не в конце. Это вкусовщина, но конструктор `NewSimulation(...)` убрал бы все эти ручные инициализации.

### 12. Нет конструктора `WorldMap`, риск паники на nil-мапе

`Entities` инициализируется руками в `main`:

```go
world := &models.WorldMap{
    Width:    10,
    Height:   10,
    Entities: make(map[models.Position]models.Occupier),
}
```

Создашь `WorldMap{Width: 10, Height: 10}` и забудешь `make`, первый же `Set` (`m.Entities[p] = e`) упадёт с паникой, запись в nil-мапу. Конструктор `NewWorldMap(w, h)` инкапсулировал бы инициализацию и сделал невалидное состояние непредставимым.

### 13. Выравнивание рендера держится на ширине эмодзи

```go
fmt.Print(getSymbol(entity) + " ")  // эмодзи + 1 пробел
// ...
fmt.Print(".  ")                     // точка + 2 пробела
```

Сетка ровная только потому, что эмодзи в большинстве терминалов занимает две колонки, а точка одну, и пробелы это компенсируют. В терминале, который рисует эмодзи в одну колонку (или с другой шириной), всё поедет.

---

## АРХИТЕКТУРА

Сравнительно с Hangman большой шаг вперёд. Декомпозиция через интерфейсы (`Occupier`, `Movable`, `Vulnerable`) и `Action` сделана правильно. Симуляция и действия разделены, `main` это декларативное описание мира.

Что можно улучшить:

**1. Сущности.** `Herbivore` и `Predator` это близнецы. Одинаковые поля (`HP`, `Hunger`), одинаковые методы (`SetNewPos`, `TakeDamage`, `IsDead`). Просится встроенная структура:

```go
type Creature struct {
    Position
    HP, Hunger int
}

type Herbivore struct { Creature }
type Predator  struct { Creature }
```

Тогда `TakeDamage`/`IsDead`/`SetNewPos` определяются один раз на `*Creature` и наследуются через встраивание. `LifeCycleAction` обрабатывает любых `*Creature` через type assertion или интерфейс `Living`.

**2. `WorldMap` слишком тонкий.** Сейчас это структура с тремя полями и пятью маленькими методами. Полезные методы (`PositionsOf`, `CountOf`, `Neighbors`) разнесены по экшенам в виде циклов. Перенесёшь их в `WorldMap`, экшены станут декларативными.

**3. BFS лежит в `actions/move_action.go`.** А ТЗ прямо говорит "выделить алгоритм поиска пути в отдельный класс". Создай `internal/pathfinding/bfs.go`, BFS не зависит от MoveAction и используется ещё в AttackAction.

**4. `Simulation.Start` делает всё.** По ТЗ нужны `NextTurn()`, `Start()`, `Pause()`. Разнеси:

```go
func (s *Simulation) NextTurn() { /* выполнить TurnActions, инкремент */ }
func (s *Simulation) Start()    { for { s.NextTurn(); ... } }
func (s *Simulation) Pause()    { /* флаг / канал */ }
```

Тогда можно прогонять ходы в тестах поштучно.

---

## Итог

Архитектурно решение приличное: интерфейсы, экшены, фабрики, BFS реализован вручную. ТЗ покрыто почти полностью (нет `Tree`, `Pause()`, `NextTurn()` отдельно, скорости существ). Из багов: off-by-one в `WorldMap.Set`, потенциально вечный цикл в `SpawnAction`/`ReplenishGrassAction` при заполненной карте, молчаливая перезапись клетки в `Move`. Из мелочи: игнорируемые ошибки `Set`, дублирование цикла обхода карты, магические числа, мёртвый код (`String()`-методы, пустой `Renderer`), комментарии-конспекты, поедание травы хищником на пути BFS. После починки багов и небольшого подравнивания архитектуры будет крепкий учебный проект.
