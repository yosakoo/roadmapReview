# Ревью Hangman
Проект: https://github.com/DarkRubin/hangman-go

Код разбит по пакетам `dictionary`, `game`, `parser`, `scanner`, состояние игры в типе `GameState`. Слова английские, читаются из файла. Игра запускается и в целом работает, но с вводом есть баги, и в паре мест код написан не так, как принято в Go.

<img src="" width="400"/>

---

## НЕДОСТАТКИ РЕАЛИЗАЦИИ

- При проигрыше слово так и не показывается:

```
Guess the word:  * e r * e
...
You lost! Congrats!
Do you want to play again? 'Y' / 'n'
```

```go
// main.go
fmt.Println(state.GallowsArt())
if state.IsWin() {
    fmt.Println("You won! Congrats!")
} else {
    fmt.Println("You lost! Congrats!")
}
```

- Заглавные буквы не принимаются:

```
Incorrect letter 'A'. Enter enter one letter from English alphabet
```

```go
// internal/scanner/scanner.go
matched, err := regexp.MatchString("^[a-z]$", line)
```

Слово к нижнему регистру приводится, а ввод нет. Достаточно `strings.ToLower(line)` перед проверкой.

- На русскую букву пишет, что ввёл слишком много:

```
You can choose only one English letter in one try
```

```go
if len(line) > 1 {
    return ' ', ErrInputTooLong
}
```

`б` в UTF-8 занимает два байта, а `len` считает байты. Подробнее в разделе про типовые ошибки.

- На ввод `a b` два раза пишет "Empty or missing input", а `b` теряется:

```go
// internal/scanner/scanner.go
func ScanLine() (line string, err error) {
    n, err := fmt.Scanln(&line)
```

`fmt.Scanln` читает до пробела. Он забрал `a`, потом съел `b`, вернул `expected newline`, а оставшийся перевод строки следующий вызов посчитал пустым вводом. Пользователь ввёл одну строку, а получил два сообщения про пустой ввод. Строки лучше читать целиком через `bufio.Scanner`:

```go
var in = bufio.NewScanner(os.Stdin)

func ScanLine() (string, error) {
    if !in.Scan() {
        if err := in.Err(); err != nil {
            return "", err
        }
        return "", io.EOF
    }
    return strings.TrimSpace(in.Text()), nil
}
```

- `Ctrl+D` / `Ctrl+Z` посреди игры:

```
Error. Could not start/continue game
```

`play` возвращает EOF, а `main` любую ошибку из `play` считает сбоем. EOF это нормальный выход, его надо проверить отдельно и просто выйти.

- Причина ошибки теряется:

```go
if err = play(); err != nil {
    fmt.Println("Error. Could not start/continue game")
    return
}
```

---

## ХОРОШО

- Код разбит по пакетам, у каждого своя задача.
- Есть тип `GameState` с приватными полями.
- Повторно ввести букву нельзя, ни угаданную, ни неугаданную. Неправильный ввод не тратит попытку.
- Новая игра запускается в цикле, а не рекурсией.
- Ошибки ввода разные (`ErrInvalidInput`, `ErrInputTooLong`, `ErrInvalidLetter`) и разбираются через `errors.Is`, на каждую своё сообщение.
- `math/rand/v2`, без ручного сида.
- Слова режутся через `strings.Fields`. В файле переводы строк CRLF, и `\r` вместе с пустыми строками отсекается сам. Пустое слово не выпадет.
- `MaxMistakesCount = 6` совпадает с количеством картинок, за границы слайса не вылезаем.
- Маска собирается через `strings.Builder`.
- Есть README с командой запуска.

---

## ТИПОВЫЕ ОШИБКИ В GO

### 1. Глобальная переменная и ленивая инициализация

```go
// main.go
const WordStorageFilename = "embed/words.txt"

var dict *dictionary.Dictionary

func play() (err error) {
    if dict == nil {
        dict, err = dictionary.NewDictionary(WordStorageFilename)
        ...
```

`dict` это глобальное состояние пакета, которое `play` сама проверяет и сама заполняет. Зависимость не видна по сигнатуре: глядя на `play()`, не поймёшь, что она лезет в файл. В Go так обычно не делают. Зависимости создаются в `main` и передаются явно:

```go
func main() {
    dict, err := dictionary.NewDictionary(wordsFile)
    if err != nil {
        fmt.Println("could not load words:", err)
        os.Exit(1)
    }
    ...
    play(dict)
}

func play(dict *dictionary.Dictionary) error { ... }
```

Ещё один плюс: если файла нет, программа скажет об этом сразу при старте, а не после того, как игрок согласился играть. В ТЗ лишние глобальные переменные есть в списке типовых ошибок.

### 2. Путь к словарю захардкожен, да ещё и экспортирован

```go
const WordStorageFilename = "embed/words.txt"
```

С большой буквы пишут то, что должно быть видно из других пакетов. Пакет `main` импортировать нельзя, так что экспортировать оттуда нечего.

Но главное, что путь к файлу вообще не стоит зашивать в код. Это настройка, а не логика: захотел другой словарь, например русский или только длинные слова, и надо лезть в код и пересобирать. Такие вещи выносят в конфиг: файл `config.yaml` / `config.json` рядом с кодом, переменные окружения или флаги запуска. Туда же просится `MaxMistakesCount`.

```yaml
# config.yaml
words_file: words.txt
max_mistakes: 6
```

```go
type Config struct {
    WordsFile   string `yaml:"words_file"`
    MaxMistakes int    `yaml:"max_mistakes"`
}
```

Или для такой маленькой игры хватит и флага:

```go
wordsFile := flag.String("words", "", "path to words file (default: built-in dictionary)")
flag.Parse()
```

Конфиг читается один раз в `main`, и значения передаются дальше. С `go:embed` (п. 3) это сочетается так: встроенный словарь по умолчанию, а путь из конфига или флага его заменяет.

### 3. Папка `embed` без `go:embed`

Файл лежит в папке `embed`, но читается с диска по относительному пути. Поэтому `go run main.go` из корня работает, а бинарь, запущенный из другой папки, нет. В Go есть `go:embed`, он зашивает файл прямо в бинарь:

```go
//go:embed embed/words.txt
var words string
```

И `os.ReadFile` больше не нужен.

### 4. `len` считает байты, а не буквы

```go
// internal/game/gamestate.go
mask := strings.Repeat("*", len(word))
...
return state.correctGuesses == len(state.word)

// internal/scanner/scanner.go
if len(line) > 1 {
```

Это самая частая ошибка со строками в Go. `len(s)` возвращает число байт. Для латиницы байт и буква совпадают, поэтому пока всё работает. Но в UTF-8 русская буква занимает два байта. Отсюда сообщение "слишком много" на `б`. А если поменять словарь на русский, сломается и сама игра: `len("кот")` равно 6, в маске будет шесть звёздочек, откроются только три, и выиграть будет невозможно. Буквы считают через `utf8.RuneCountInString(s)` или `len([]rune(s))`.

### 5. Свой `ErrEOF` вместо `io.EOF`

```go
var (
    ErrEOF           = errors.New("EOF")
    ErrInvalidInput  = errors.New("invalid input")
    ErrInputTooLong  = errors.New("input too long")
    ErrInvalidLetter = errors.New("invalid letter")
)
```

`ErrEOF` повторяет стандартный `io.EOF`. Все в Go проверяют конец ввода через `errors.Is(err, io.EOF)`, своя копия только путает. Ещё и сравнение внутри сделано через `err == io.EOF`: если ошибку где-то обернут, оно перестанет срабатывать. Возвращай `io.EOF` как есть, а проверяй через `errors.Is`.

### 6. Геттеры с `Get` и повтор имени пакета

```go
func (state *GameState) GetMask() string
func (state *GameState) GetMistakes() int
```

В Go геттеры называют без `Get`: `Mask()`, `Mistakes()`. А тип `game.GameState` при вызове читается как "game game state". Имя пакета уже часть имени, так что `game.State` достаточно.

### 7. Регулярка на одну букву

```go
matched, err := regexp.MatchString("^[a-z]$", line)
```

`regexp.MatchString` каждый раз заново компилирует выражение. Проверить одну букву можно проще: `r >= 'a' && r <= 'z'`. Если регулярка всё же нужна, её компилируют один раз в переменную пакета через `regexp.MustCompile`.

### 8. Слайс там, где нужно множество

```go
guessedLetters []rune
...
return slices.Contains(state.guessedLetters, letter)
```

Вопрос "вводили ли эту букву" это вопрос к множеству. В Go множество делают через `map[rune]struct{}`. На 26 буквах по скорости разницы нет, но так понятнее, что хранится. В ТЗ это тоже есть в типовых ошибках.

### 9. Слайс картинок создаётся на каждый вызов

```go
func getGallowsAsciiArts() []string {
    return []string{ ... }
}
```

На каждый ход заново собирается слайс из семи строк. Картинки не меняются, их место в переменной пакета:

```go
var gallowsStages = []string{ ... }
```

### 10. Мелочи

- В `ScanEngLetter` есть `for`, но каждая ветка делает `return`, второй итерации не бывает. Цикл лишний.
- При ошибке возвращается то `0`, то `' '`. Лучше везде одно и то же, обычно `0`.
- `RandomWord` на значении (`dict Dictionary`), а `NewDictionary` возвращает указатель. Обычно выбирают что-то одно.
- Поле `count` повторяет `len(words)`.
- `state.correctGuesses = state.correctGuesses + 1` пишут как `state.correctGuesses++`.
- `panic("unreachable")` в `default`. Если когда-нибудь добавишь новую ошибку и забудешь про `switch`, игра упадёт посреди партии. Лучше напечатать общее сообщение.

---

## АРХИТЕКТУРА

Деление в целом правильное: логика игры в `game`, чтение ввода в `scanner`, разбор "да/нет" в `parser`, словарь отдельно. `main` крутит циклы и печатает. Но правила игры частично утекли в `main`, а отрисовка, наоборот, залезла в `GameState`.

### `Guess` надеется, что его вызовут правильно

```go
func (state *GameState) Guess(letter rune) (success bool) {
    state.guessedLetters = append(state.guessedLetters, letter)
    if strings.ContainsRune(state.word, letter) {
        for i, symbol := range []rune(state.word) {
            if symbol == letter {
                state.mask[i] = symbol
                state.correctGuesses = state.correctGuesses + 1
            }
        }
        return true
    }
    ...
}

func (state *GameState) IsWin() bool {
    return state.correctGuesses == len(state.word)
}
```

Сам `Guess` повтор не проверяет. Это делает `main` через `IsAlreadyGuessed` перед каждым ходом. Если `Guess` вызвать дважды с одной буквой, счётчик посчитает её два раза:

```go
s := New("ab")
s.Guess('a')
s.Guess('a')
// mask="a * " win=true
```

Победа, хотя `b` так и не открыта. А на слове `"aab"` наоборот: после `a`, `a`, `b` всё открыто, но `correctGuesses = 5`, а `len = 3`, и игра не заканчивается.

Сейчас до этого не доходит только благодаря `main`. Но правила игры должны жить в типе игры, иначе любой новый код (тест, другой интерфейс) их сломает. Пусть `Guess` сам говорит, чем кончился ход, а победа проверяется по маске:

```go
type GuessResult int

const (
    Hit GuessResult = iota
    Miss
    Repeated
)

func (s *State) Guess(letter rune) GuessResult {
    if _, ok := s.used[letter]; ok {
        return Repeated
    }
    s.used[letter] = struct{}{}
    ...
}

func (s *State) IsWin() bool {
    return !slices.Contains(s.mask, '*')
}
```

Тогда `correctGuesses` и `IsAlreadyGuessed` не нужны, а `main` по результату только выбирает, что напечатать.

### Отрисовка внутри `GameState`

```go
func (state *GameState) GallowsArt() string {
    return getGallowsAsciiArts()[state.mistakes]
}
```

Плюс `GetMask` ставит пробелы между буквами. Это уже про то, как игру показать, а не про саму игру.

### `scanner` знает про английский алфавит

`ScanEngLetter` одновременно читает ввод и проверяет, что это латинская буква. Если словарь станет русским, придётся менять пакет, который просто читает строки. Чтение строки и проверку буквы лучше разделить: `scanner` отдаёт строку, а что считается правильной буквой, решает игра.

### `ParseDecision`

```go
switch len(input) {
case 1:
    switch strings.ToLower(input)[0] {
    case 'y':
        return true, nil
    ...
case 2:
    if strings.EqualFold(input, "no") {
    ...
```

Ради одной функции заведён отдельный пакет, а сама функция запутанная: `switch` по длине, внутри по первому байту, потом `EqualFold`. То же самое пишется короче, и пакет для этого не нужен:

```go
switch strings.ToLower(input) {
case "y", "yes":
    return true, nil
case "n", "no":
    return false, nil
}
return false, errors.New("invalid input")
```

---

## Итог

Игра работает, код разбит по пакетам, повтор букв и неправильный ввод не тратят попытки, новая игра запускается циклом. В первую очередь стоит починить ввод и вывод: показывать слово после проигрыша, принимать заглавные буквы, читать строку целиком вместо `fmt.Scanln`, не выдавать EOF за ошибку и печатать причину, когда что-то сломалось. Дальше разобраться с Go-шными вещами: убрать глобальный `dict`, вынести путь к словарю в конфиг и зашить словарь по умолчанию через `go:embed`, считать длину строк в рунах, переименовать геттеры. А в архитектуре главное перенести проверку повтора внутрь `Guess` и вынести отрисовку из `GameState`.
