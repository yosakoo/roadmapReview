# Ревью Hangman
Проект: https://github.com/mom4uk/hangman

[Ревью сделано в рамках учебной подписки](https://zhukovsd.it/services/student-subscription/)

Реализация написана в процедурном стиле, код разбит по пакетам. Есть баги. Игра запускается.

<img src="https://i.pinimg.com/736x/5a/ea/09/5aea0924f2e97cfe7e63b626441e9d07.jpg" width="300"/>

---

## НЕДОСТАТКИ РЕАЛИЗАЦИИ

- Неправильно введённую букву можно вводить сколько угодно раз - каждый раз программа принимает её и увеличивает счётчик ошибок:

```
Число ошибок: 1
Введите букву:
а
Этой буквы нет
Число ошибок: 2
Введите букву:
а
Этой буквы нет
Число ошибок: 3
```

- `Run()` не зацикливается. Если ввести что угодно кроме `start`, программа печатает "Введите 'start'" и завершается. Пользователь вынужден запускать программу заново.

- Слова захардкожены в массиве - только три штуки: `"тест"`, `"солнце"`, `"лес"`. При проигрыше очевидно, какое слово следующим - играть неинтересно. Почему слова не читаются из файла?

- Опечатка в сообщении о победе: `"поздарвляю"` вместо `"поздравляю"`.

- Сообщение о поражении противоречит само себе:

```go
fmt.Print("Вы проиграли.\nЧтобы закончить или начать новую игру введите n / y\nЧтобы начать новую игру введите 'yes' или введите любой символ для выхода и нажмите 'Enter'.\n")
```

Сначала написано "введите n / y", потом "введите 'yes'". `promptRestart` принимает только `"yes"`.

---

## ХОРОШО

- Код разбит по пакетам: `cmd/`, `internal/`, `utilities/` - правильный подход.
- Валидация ввода: проверяется кириллица, один символ, нижний регистр.
- Рунная обработка строк (`[]rune`) - правильно для работы с кириллицей.
- Вспомогательные функции вынесены в отдельный пакет `utilities`.
- Есть возможность перезапустить игру.
- `rand.Intn(len(words))` - нет off-by-one, в отличие от частой ошибки с `len-1`.
- Смешной дизайн висельника со сменой эмодзи по ходу игры: 🙂 → 😐 → 😟 → 😨 → 😰 → 😱 → 💀. Приятная деталь.

---

## ЗАМЕЧАНИЯ

### 1. Неправильный порядок проверок в `validateInput`

Первой строкой в `validateInput` обращение к `[]rune(*input)[0]`, хотя длина ещё не проверена:

```go
func validateInput(input *string, userAnswer []rune) (bool, string) {
    doesAnswerHasInputChar := strings.ContainsRune(string(userAnswer), []rune(*input)[0])  // берёт первый символ до проверки длины
    // ...
    hasInputOneChar := len([]rune(*input)) != 1  // проверка длины - слишком поздно
```

Если ввести `"аб"` - функция сначала проверит, есть ли `'а'` в маске. Если `'а'` уже открыт, вернётся `"Данный символ уже отгадан"` вместо `"Ввод может содержать только одну букву"`. Проверка длины должна идти первой:

```go
func validateInput(input *string, userAnswer []rune) (bool, string) {
    if len([]rune(*input)) != 1 {
        return false, "Ввод может содержать только одну букву\n"
    }

    letter := []rune(*input)[0]

    if strings.ContainsRune(string(userAnswer), letter) {
        return false, "Данный символ уже отгадан\n"
    }
    // ...
```

### 2. Имя переменной `hasInputOneChar` логически инвертировано

```go
hasInputOneChar := len([]rune(*input)) != 1
if hasInputOneChar {
    // ...
}
```

Имя говорит "ввод содержит одну букву", но значение `true` означает обратное - символов больше одного. Читать такой код неудобно.

**Правильно:**
```go
hasNotOneChar := len([]rune(*input)) != 1
if hasNotOneChar {
    // ...
}
```

### 3. Ошибка выводится дважды

При вводе нескольких символов сообщение об ошибке печатается два раза: один раз внутри `validateInput`, второй раз в `makeMove`:

```go
// validateInput:
if hasInputOneChar {
    fmt.Print("Ввод может содержать только одну букву\n")  // первый вывод
    return false, "Ввод может содержать только одну букву\n"
}

// makeMove:
if !isInputValid {
    fmt.Print(errorText)  // второй вывод того же текста
    return
}
```

`validateInput` не должна печатать - только возвращать результат. Печатает вызывающий код.

### 4. Повторный ввод неправильной буквы

Проверка на повтор ищет символ в маске (`userAnswer`):

```go
doesAnswerHasInputChar := strings.ContainsRune(string(userAnswer), []rune(*input)[0])
```

Но ошибочные буквы в маску не попадают. Если буква не входит в слово, она нигде не сохраняется - вводи снова, счётчик растёт. Нужно хранить все введённые буквы отдельно:

```go
guessedLetters := map[rune]bool{}

// при каждом вводе:
if guessedLetters[letter] {
    return false, "Вы уже вводили эту букву\n"
}
guessedLetters[letter] = true
```

### 5. Пакет `utilities` - плохое имя для пакета в Go

Официальный блог Go прямо говорит об этом (https://go.dev/blog/package-names):

> Packages named `util`, `common`, or `misc` provide clients with no sense of what the package contains. This makes it harder for clients to use the package and makes it harder for maintainers to keep the package focused.

Пакет `utilities` - это "мусорный ящик": туда можно положить что угодно, и по имени непонятно, что внутри. Сейчас там три функции, которые можно разнести по смысловым пакетам:

```
utilities.IsCyrillicChar          → internal/input или прямо в internal
utilities.FindAllIndexes          → internal/word
utilities.ReplaceUnderscoreByChar → internal/word
```

Или, если функции слишком маленькие для отдельных пакетов - держать их в `internal` напрямую, без прослойки.

### 6. Вся статика перечитывается с диска на каждом ходу

```go
func printCurrentGameState(movesCounter *int, errorsCounter *int) {
    var pictures = []string{          // пересоздаётся на каждом вызове
        "assets/first_move.txt",
        // ...
    }
    path := pictures[*movesCounter]
    text, _ := os.ReadFile(path)      // читает файл с диска на каждом ходу
    fmt.Print(string(text))
```

Файлы со стадиями висельника не меняются. Их стоит загрузить один раз при старте и держать в памяти:

```go
func loadStages() []string {
    paths := []string{
        "assets/first_move.txt",
        // ...
    }
    stages := make([]string, len(paths))
    for i, path := range paths {
        text, err := os.ReadFile(path)
        if err != nil {
            log.Fatalf("не удалось загрузить стадию %s: %v", path, err)
        }
        stages[i] = string(text)
    }
    return stages
}
```

Тогда в игровом цикле нет никакого I/O - только обращение к слайсу.

### 7. `doesUserLose` использует `==` вместо `>=`

```go
func doesUserLose(movesCounter *int, numOfAvaliableMoves int) bool {
    return *movesCounter == numOfAvaliableMoves
}
```

Если счётчик ходов обгонит `numOfAvaliableMoves`, условие `==` станет `false` и проигрыш не зафиксируется. Безопаснее:

```go
func doesUserLose(movesCounter *int, numOfAvaliableMoves int) bool {
    return *movesCounter >= numOfAvaliableMoves
}
```

### 8. Рекурсивный перезапуск через `StartGame()`

При нажатии `"yes"` в `promptRestart` вызывается `StartGame()` изнутри уже запущенного `StartGame()`:

```go
case "yes":
    StartGame()  // бесконечный рост стека
```

Каждый перезапуск добавляет фрейм в стек. После многих игр - переполнение стека. Нужно использовать цикл на уровне `Run()`:

```go
func Run() {
    // ...
    for {
        StartGame()
        // спросить: играть снова?
    }
}
```

### 9. Исполнение продолжается после `promptRestart`

После победы или поражения поток выполнения не прерывается:

```go
for usersMoves <= numberOfAvaliableMoves {
    printCurrentGameState(&usersMoves, &userErrors)

    if isGameFinished(...) {
        printResult(...)
        promptRestart()   // если вернётся - выполнение продолжится
    }
    makeMove(...)  // вызывается даже после окончания игры
}
```

Если `promptRestart` вернётся при ошибке ввода, сразу после него вызывается `makeMove`. Нужен `return`:

```go
if isGameFinished(...) {
    printResult(...)
    promptRestart()
    return
}
```

---

## АРХИТЕКТУРА

Пакеты разделены - это хорошо. Но внутри `internal` всё состояние игры живёт в отдельных переменных, которые передаются в каждую функцию через указатели:

```go
makeMove(&usersMoves, &userAnswer, actualAnswer, &userErrors)
printCurrentGameState(&usersMoves, &userErrors)
printResult(&usersMoves, &userAnswer, actualAnswer, numberOfAvaliableMoves)
```

По сути - те же глобальные переменные, только их ещё надо передавать вручную. Напрашивается тип `Game`:

**`Game`** - состояние игры:
- секретное слово и маска
- счётчик ошибок
- множество введённых букв
- методы: `GuessLetter`, `IsWon`, `IsLost`, `MaskString`, `WrongLetters`

**`Gallows`** - картинки по стадиям:
- метод `Stage(n int) string`

С таким делением игровой цикл становится читаемым.

---

## Итог

Игра работает. Ввод валидируется, перезапуск есть. Но неправильную букву можно вводить сколько угодно раз, слова захардкожены, рестарт через рекурсию рано или поздно сломает стек. Над архитектурой надо думать. Проект нужно дорабатывать.
