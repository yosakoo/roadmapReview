# Ревью Simulation
Проект: https://github.com/themidnightdev404/simulation

Терминальная симуляция экосистемы на ходах. Разбита на пакеты `entity`, `action`, `pathfinding`, `sim`, корневой `main`. `Creature`-интерфейс из ТЗ на месте, BFS в отдельном пакете, `BaseCreature`/`StaticEntity` через embedding, у `Simulation` есть `NextTurn`/`Start`/`Pause`/`Stop`, конкурентный доступ к паузе через `sync.Mutex`, стдин читается через `bufio.Reader` с обработкой EOF. Есть grass regrowth, голод, размножение. Запускается, работает.

<img src="https://i.pinimg.com/736x/6d/32/6a/6d326a1f9ed3d3c7eac5df5f9965412f.jpg" width="400"/>

---

## СООТВЕТСТВИЕ ТЗ

- ✅ Все статичные объекты: `Grass`, `Rock`, `Tree`.
- ✅ `Occupier` интерфейс (`Type`, `Pos`).
- ✅ `Creature` интерфейс со всеми методами из ТЗ: `Speed`, `HP`, `TakeDamage`, `IsAlive`, `MoveTo`, `Hunger`, `Eat`, `Tick`.
- ✅ Композиция через embedding: `Herbivore`/`Predator` встраивают `BaseCreature`, `Grass`/`Rock`/`Tree` встраивают `StaticEntity`.
- ✅ `Action` интерфейс, `initActions` и `turnActions` массивы.
- ✅ BFS в отдельном пакете `pathfinding/bfs.go`.
- ✅ У `Simulation` отдельные методы `NextTurn`, `Start`, `Pause`, `Stop`.
- ✅ `MapView` и `Renderer` интерфейсы. Рендер видит только узкий `MapView`, а не весь `Map`.
- ✅ Сетка N×M, одна сущность на клетку.
- ✅ Конструкторы: `NewMap`, `NewHerbivore`, `NewPredator`, `NewBaseCreature`, `NewSimulation`.
- ✅ Голод и размножение (допфичи из ТЗ).
- ✅ Grass regrowth (допфича).

---

## НЕДОСТАТКИ РЕАЛИЗАЦИИ

- `Map.Move` не проверяет, свободна ли целевая клетка ([map.go:43](simulation/entity/map.go#L43)):

```go
func (m *Map) Move(c Creature, newPos Position) {
    oldPos := c.Pos()
    c.MoveTo(newPos)
    delete(m.Cells, oldPos)
    m.Cells[newPos] = c
}
```

Если `newPos` уже занят другой сущностью, она молча стирается. Сейчас не стреляет, потому что `moveToward` берёт `path[steps]` из BFS, а BFS помечает проходимыми только свободные клетки. Но контракт слабый: любой вызывающий может передать занятую позицию, и данные пропадут без звука. Проверяй `IsFree(newPos)` и отдавай ошибку (или bool).

- `spawnCount` тихо ставит меньше объектов, чем запросили ([spawn.go:25](simulation/action/spawn.go#L25)):

```go
func spawnCount(m *entity.Map, count int, create func(pos entity.Position) entity.Occupier) {
    for i := 0; i < count; i++ {
        pos, ok := randomPosition(m)
        if !ok {
            continue
        }
        m.Place(create(pos))
    }
}
```

Счётчик `i` инкрементится каждую итерацию, даже когда `randomPosition` вернула false. На плотной карте можно попросить 10 травы, а получить 3, и никто про это не узнает. Либо крути внутренний цикл до успеха (с общим лимитом попыток), либо хотя бы возвращай счётчик реально размещённых.

- Поиск "ближайшего" по манхэттену, а пути может не быть ([herbivore_action.go:23](simulation/action/herbivore_action.go#L23), [predator_action.go:25](simulation/action/predator_action.go#L25)):

```go
d := manhattan(h.Pos(), pos)
if !found || d < bestDist {
    nearestGrass = pos
    // ...
}
// ...
moveToward(m, h, nearestGrass)   // BFS может вернуть nil, шаг пропущен
```

Ближайшая по манхэттену трава может стоять за грядой камней, пути нет. `moveToward` тогда возвращается ни с чем, травоядное стоит на месте, а до достижимой травы чуть дальше даже не пытается пройти. В открытом мире почти не видно, с препятствиями существа зависают. Вариант: сортировать кандидатов по манхэттену и брать первого, для кого BFS вернул путь. Или сделать BFS с multi-target, чтобы за один проход находить ближайшую достижимую цель.

- Хищник восстанавливает голод при любом ударе, а не только при убийстве ([predator_action.go:36-41](simulation/action/predator_action.go#L36)):

```go
if bestDist <= 1 {
    nearestVictim.TakeDamage(h.AttackPower())
    h.Eat()                       // hunger = 0 при каждом ударе
    if !nearestVictim.IsAlive() {
        m.Remove(nearestHerb)
    }
}
```

Травоядное `Eat()` вызывает, только когда грасс реально снят с карты. Хищник же сбрасывает голод в 0 на каждом ударе, независимо от того, добил он жертву или нет. Пока рядом кто-то есть, чтобы шлёпнуть, хищник вечно сыт. Перенеси `h.Eat()` внутрь `if !nearestVictim.IsAlive()`, либо давай частичное восстановление пропорционально урону.

---

## ХОРОШО

- `Creature` интерфейс со всем набором методов из ТЗ.
- `BaseCreature` встроен в `Herbivore` и `Predator`, поля и методы наследуются без копипасты. То же с `StaticEntity` для `Grass`/`Rock`/`Tree`.
- `MapView` интерфейс, рендер видит только `Width`/`Height`/`At`. Interface segregation по назначению.
- `Renderer` тоже интерфейс, можно подставить веб или GUI, не трогая `Simulation`.
- Конкурентный доступ к paused/stopped через `sync.Mutex`. Для операции "инвертировать флаг" мьютекс уместнее, чем `atomic.Bool` с гонкой между `Load` и `Store`.
- Чтение стдин через `bufio.NewReader(os.Stdin).ReadString('\n')` с проверкой ошибки. Если пайп закрыли или пришёл Ctrl+D, горутина выходит по EOF, а не крутится вхолостую.
- BFS живёт в своём пакете, `moveToward` его использует. Логика поиска пути изолирована.
- `NextTurn`/`Start`/`Pause`/`Stop` разнесены, ходы можно прогонять поштучно.
- `Tree`, `Rock`, `Grass` все на месте.
- После убийства хищник удаляет труп через `m.Remove`, никаких черепов-блокираторов.
- `HungerAction` работает через интерфейс `Creature`, не разбирая конкретный тип. `Tick` инкрементит голод и снимает HP при переполнении, `HungerAction` убирает мёртвых.
- Grass regrowth через шанс, а не порог. Естественнее для экосистемы.
- Пустая клетка рендерится как `⬜`, сетка не рассыпается.
- Все конструкторы на месте (`NewMap`, `NewHerbivore`, `NewPredator`, `NewBaseCreature`, `NewSimulation`), состояние после создания сразу валидное.
- Turn order осмысленный: сперва прибавляем ресурс (grass regrowth), потом голод, потом действия, в конце размножение.

---

## ЗАМЕЧАНИЯ

### 1. `Map.Cells` публичное поле

```go
type Map struct {
    width  int
    height int
    Cells  map[Position]Occupier   // публичное
}
```

`width`/`height` приватные, `Cells` открытая. Actions пользуются этим и ходят по мапе напрямую:

```go
for _, occ := range m.Cells { ... }
```

Инкапсуляция ломается неровно: рендер работает через `MapView`, actions лезут в потроха. Спрячь `Cells`, заведи метод типа `ForEach(func(Occupier))` или `AllOfType(EntityType) []Occupier`.

### 2. Простыни параметров в `RandomSpawn` и `ReproduceAction`

```go
type RandomSpawn struct {
    HerbivoreCount    int
    HerbivoreMinHP    int
    HerbivoreMaxHP    int
    HerbivoreMinSpeed int
    HerbivoreMaxSpeed int
    // и то же самое для Predator + Grass/Rock/Tree
}
```

`ReproduceAction` содержит те же 8 полей для существ, кроме count. В `NewSimulation` эти числа заданы дважды: для спавна и для размножения. Вынеси в отдельные типы:

```go
type HerbivoreConfig struct {
    MinHP, MaxHP, MinSpeed, MaxSpeed int
}

type RandomSpawn struct {
    HerbivoreCount int
    Herbivore      HerbivoreConfig
    // ...
}
```

Один `HerbivoreConfig` пойдёт и в спавн, и в размножение.

### 3. Линейный поиск ближайшей цели

```go
for pos, occ := range m.Cells {
    if occ.Type() != entity.GrassType { continue }
    // ...
}
```

`HerbivoreAction` и `PredatorAction` для каждого существа обходят всю карту в поисках ближайшей цели по манхэттену. При N существ и K клеток получается O(N × K) на ход. Как минимум можно один раз пройти по карте, собрать позиции `Grass` в срез, и потом для каждого травоядного искать по срезу.

### 4. `moveToward` строит весь путь, а нужен один шаг

```go
path := pathfinding.FindPath(m, c.Pos(), target)
// ...
m.Move(c, path[steps])
```

BFS восстанавливает полный путь `[start, ..., target]`, выделяет два слайса (один при обратном обходе, второй при реверсе), и всё это ради одного индекса `path[steps]`. Для длинных путей расточительно. Можно возвращать сразу шаг на нужной глубине, например BFS с ограничением по глубине и запоминанием "первого шага" для каждой посещённой клетки.

### 5. `Simulation` поля публичные

```go
type Simulation struct {
    Map         *entity.Map
    CountStep   int
    InitActions []action.Action
    TurnActions []action.Action
    Render      Renderer
    // ...
}
```

`main.go` использует `s.CountStep` напрямую. Наружу лучше отдавать через геттер `Turn()`, а поля закрыть. То же самое с `Map`, `InitActions`, `TurnActions`, `Render`: снаружи после `NewSimulation` они не нужны.

### 6. Строка карты собирается через конкатенацию

```go
line := ""
for x := 0; x < m.Width(); x++ {
    // ...
    line = line + symbol
}
```

Каждый `line + symbol` выделяет новую строку. `strings.Builder` идиоматичнее и не тратит аллокации:

```go
var b strings.Builder
for x := 0; x < m.Width(); x++ {
    // ...
    b.WriteString(symbol)
}
fmt.Println(b.String())
```

### 7. Цепочка `if occ.Type() == ...` в рендере

```go
if occ.Type() == entity.GrassType {
    symbol = "🍃"
} else if occ.Type() == entity.RockType {
    symbol = "🗿"
} // ...
```

Пять веток `else if` подряд. `switch occ.Type() { case ... }` короче. Ещё лучше вынести `Symbol()` в саму сущность, тогда рендер сведётся к `symbol = occ.Symbol()`.

### 8. Размножение возможно только сразу после еды

```go
if c.Hunger() != 0 {
    continue
}
```

`Tick()` инкрементит голод каждый ход, `Eat()` сбрасывает в 0. Значит, `Hunger == 0` бывает только в тот ход, когда существо съело. Идея понятная (сытое размножается), но окно однотактовое: `Tick` идёт до `Herbivore`/`Predator`Action, а `Reproduce` после, так что фаза сытости длится ровно один такт. Мягкий порог (например, `< 5`) сделал бы размножение чаще без потери смысла.

### 9. `BaseCreature.Position` публичное поле

```go
type BaseCreature struct {
    Position Position
    speed    int
    hp       int
    hunger   int
}
```

`speed`/`hp`/`hunger` приватные, `Position` публичное. Скорее всего, чтобы embedding работал за пределами пакета `entity`. Но раз есть `Pos()` и `MoveTo()`, публичное поле только вредит: кто-то из пакета `entity` может изменить `Position` в обход `MoveTo`, и позиция в `Map.Cells` разъедется с внутренней позицией существа.


## АРХИТЕКТУРА

По декомпозиции разложено чисто. Разделение `entity` / `action` / `pathfinding` / `sim` естественное, `MapView` и `Renderer` вынесены в интерфейсы, embedding используется по назначению. Мьютексы для конкурентного стейта корректные.

Что можно подтянуть:

1. Инкапсулировать `Map.Cells`. Сейчас поле открыто и actions лазят в него напрямую. Через методы итерации/фильтрации в `Map` actions станут короче.

2. Общие параметры баланса вынести в отдельные типы. Диапазоны HP/Speed/Attack дублируются в `RandomSpawn` и `ReproduceAction`. Общий `HerbivoreConfig`/`PredatorConfig` уберёт дубль.

3. Закрыть публичные поля `Simulation`. Наружу нужен только `Turn()` (или хук в рендер).

4. Пересмотреть предаторский `Eat`. Сейчас хищник вечно сыт, если рядом хоть кто-то. Либо только на убийстве, либо частично на удар.

5. Тесты. `FindPath`, `Move`, `Tick`, `HungerAction` это удобные кандидаты на покрытие.

---

## Итог

ТЗ покрыто целиком: полный `Creature` интерфейс, BFS в отдельном пакете, embedding для `BaseCreature`/`StaticEntity`, `MapView`/`Renderer` интерфейсы, `Pause`/`Stop`/`NextTurn`/`Start` разделены, mutex для конкурентного стейта, корректная обработка EOF на стдин. Из багов: `Map.Move` не проверяет свободу целевой клетки (тихая перезапись), `spawnCount` тихо ставит меньше запрошенного, поиск целей по манхэттену без проверки достижимости, хищник вечно сыт из-за `Eat` на любом ударе, экран не чистится между ходами. Из мелочи: публичное `Map.Cells` и прямой доступ из actions, `Attacker` интерфейс без использования, дублирование параметров баланса, конкатенация строк в рендере, магические числа.