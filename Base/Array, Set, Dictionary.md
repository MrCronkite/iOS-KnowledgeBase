# Array, Set, Dictionary — Вопросы, ответы и задачи
> iOS-собеседование · Swift Collections · Алгоритмы на коллекциях
> Уровни: Beginner · Middle · Senior + Практические задачи

---

## Array

### Beginner

**Что такое Array в Swift?**
> Упорядоченная коллекция элементов одного типа с произвольным доступом по индексу. `let nums = [1, 2, 3]`. Элементы хранятся в непрерывном блоке памяти — O(1) доступ по индексу. Дубликаты разрешены. Value type с COW семантикой.

**Как создать массив?**
```swift
let empty: [Int] = []
let empty2 = [Int]()
let empty3 = Array<Int>()

let nums = [1, 2, 3]                       // литерал
let repeated = Array(repeating: 0, count: 5) // [0,0,0,0,0]
let range = Array(1...5)                    // [1,2,3,4,5]
let fromSet = Array(mySet)                  // из Set
```

**Как добавить элемент в массив?**
```swift
var arr = [1, 2, 3]
arr.append(4)              // в конец: [1,2,3,4]
arr.append(contentsOf: [5, 6])  // несколько: [1,2,3,4,5,6]
arr.insert(0, at: 0)       // в начало: [0,1,2,3,4,5,6]
arr.insert(10, at: 2)      // по индексу
arr += [7, 8]              // через +=
```

**Как удалить элемент из массива?**
```swift
var arr = [1, 2, 3, 4, 5]
arr.removeLast()           // убрать последний → [1,2,3,4]
arr.removeFirst()          // убрать первый → [2,3,4]
arr.remove(at: 1)          // по индексу → [2,4]
arr.removeAll()            // очистить
arr.removeAll(where: { $0.isMultiple(of: 2) })  // по условию
```

**Как получить элемент массива?**
```swift
let arr = [10, 20, 30, 40]
arr[0]          // 10 (краш если out of bounds!)
arr.first       // Optional(10)
arr.last        // Optional(40)
arr.first { $0 > 15 }  // Optional(20) — первый по условию

// Безопасный доступ:
extension Array {
    subscript(safe index: Int) -> Element? {
        indices.contains(index) ? self[index] : nil
    }
}
arr[safe: 10]   // nil, не краш
```

**Что такое `count` и `isEmpty`?**
> `count` — количество элементов: O(1). `isEmpty` — проверка что массив пуст: O(1), предпочтительнее чем `count == 0`. `arr.isEmpty` читается лучше и не вычисляет count полностью.

**Как итерировать массив?**
```swift
let arr = ["a", "b", "c"]

for item in arr { print(item) }

for (index, item) in arr.enumerated() {
    print("\(index): \(item)")
}

arr.forEach { print($0) }  // нельзя break/continue

// С индексом:
arr.indices.forEach { print(arr[$0]) }
```

---

### Middle

**Какова сложность операций Array?**
| Операция | Сложность | Примечание |
|----------|-----------|-----------|
| Доступ по индексу | O(1) | Непрерывная память |
| `append` | O(1) амортизированно | Иногда O(n) при resizing |
| `insert(at:)` | O(n) | Сдвиг элементов |
| `remove(at:)` | O(n) | Сдвиг элементов |
| `contains` | O(n) | Линейный поиск |
| `sort` | O(n log n) | Timsort |
| `first/last` | O(1) | — |

**Как работает COW для Array?**
```swift
var a = [1, 2, 3]
var b = a  // нет копирования — разделяют буфер

b.append(4)  // ЗДЕСЬ происходит копирование буфера a
// a = [1,2,3], b = [1,2,3,4]

// Проверить уникальность:
withUnsafeBytes(of: &a) { print($0.baseAddress!) }
withUnsafeBytes(of: &b) { print($0.baseAddress!) }
// После append — разные адреса
```

**Что такое `map`, `filter`, `reduce`?**
```swift
let nums = [1, 2, 3, 4, 5]

// map — трансформировать:
let doubled = nums.map { $0 * 2 }  // [2,4,6,8,10]

// filter — отфильтровать:
let evens = nums.filter { $0.isMultiple(of: 2) }  // [2,4]

// reduce — свернуть в одно значение:
let sum = nums.reduce(0) { $0 + $1 }  // 15
let sum2 = nums.reduce(0, +)          // 15

// compactMap — map + убрать nil:
let strs = ["1", "foo", "3"]
let ints = strs.compactMap { Int($0) }  // [1, 3]

// flatMap — map + flatten:
let nested = [[1,2], [3,4], [5]]
let flat = nested.flatMap { $0 }  // [1,2,3,4,5]
```

**Как сортировать массив?**
```swift
var nums = [3, 1, 4, 1, 5, 9, 2]

// Mutating:
nums.sort()                          // [1,1,2,3,4,5,9]
nums.sort(by: >)                     // [9,5,4,3,2,1,1]
nums.sort { $0 > $1 }

// Non-mutating:
let sorted = nums.sorted()
let sortedDesc = nums.sorted(by: >)

// Кастомная сортировка:
struct User { var name: String; var age: Int }
let users = [User(name: "Bob", age: 30), User(name: "Alice", age: 25)]
let sortedUsers = users.sorted { $0.name < $1.name }

// Multi-критерий:
let multiSorted = users.sorted {
    if $0.age != $1.age { return $0.age < $1.age }
    return $0.name < $1.name
}
```

**Что такое `flatMap` vs `compactMap`?**
```swift
// flatMap для Optional (deprecated → compactMap):
let opts: [Int?] = [1, nil, 3, nil, 5]
let nonNil = opts.compactMap { $0 }  // [1, 3, 5]

// flatMap для вложенных коллекций:
let words = ["Hello World", "Swift Programming"]
let chars = words.flatMap { $0.split(separator: " ") }
// ["Hello", "World", "Swift", "Programming"]

// Разница:
// compactMap: [A?] → [A]  (убирает nil)
// flatMap: [[A]] → [A]    (разворачивает вложенность)
```

**Как найти элемент в массиве?**
```swift
let arr = [1, 2, 3, 4, 5]

arr.contains(3)                    // true, O(n)
arr.contains(where: { $0 > 3 })   // true
arr.first(where: { $0 > 3 })      // Optional(4)
arr.last(where: { $0.isMultiple(of: 2) })  // Optional(4)
arr.firstIndex(of: 3)             // Optional(2), O(n)
arr.firstIndex(where: { $0 > 3 }) // Optional(3)
arr.indices(of: 1)                 // нет такого, нужно реализовать
```

**Как работает `stride` и `zip`?**
```swift
// stride — шаговая итерация:
for i in stride(from: 0, to: 10, by: 2) {
    print(i)  // 0, 2, 4, 6, 8
}
for i in stride(from: 10, through: 0, by: -2) {
    print(i)  // 10, 8, 6, 4, 2, 0
}

// zip — попарная итерация:
let names = ["Alice", "Bob", "Charlie"]
let scores = [95, 87, 92]
for (name, score) in zip(names, scores) {
    print("\(name): \(score)")
}

// zip в массив пар:
let pairs = Array(zip(names, scores))
// [("Alice", 95), ("Bob", 87), ("Charlie", 92)]
```

**Что такое `slice` и `ArraySlice`?**
```swift
let arr = [1, 2, 3, 4, 5]
let slice = arr[1...3]  // ArraySlice<Int> — view на оригинал
// slice.indices — 1, 2, 3 (не 0, 1, 2!)

// Безопасное обращение к индексам slice:
for i in slice.indices {
    print(slice[i])
}

// Создать новый Array:
let newArr = Array(slice)  // [2, 3, 4]

// Важно: slice удерживает оригинальный массив в памяти
```

**Как реализовать sliding window?**
```swift
// Пример: суммы окон по 3 элемента
let nums = [1, 2, 3, 4, 5, 6]
let windowSize = 3

// Ручная реализация:
var sums: [Int] = []
for i in 0...(nums.count - windowSize) {
    let window = nums[i..<(i + windowSize)]
    sums.append(window.reduce(0, +))
}
// [6, 9, 12, 15]

// Через zip:
extension Array {
    func windows(of size: Int) -> [[Element]] {
        guard size <= count else { return [] }
        return (0...count - size).map { Array(self[$0..<$0+size]) }
    }
}
nums.windows(of: 3)  // [[1,2,3],[2,3,4],[3,4,5],[4,5,6]]
```

**Что такое `partition(by:)`?**
```swift
var arr = [1, 2, 3, 4, 5, 6]
let pivot = arr.partition(by: { $0.isMultiple(of: 2) })
// arr = [1, 3, 5, 2, 4, 6] (нечётные | чётные, порядок не гарантирован)
// pivot = 3 (индекс начала второй группы)

let odds = arr[..<pivot]   // [1, 3, 5]
let evens = arr[pivot...]  // [2, 4, 6]
```

---

### Senior

**Как работает реаллокация буфера Array?**
> При `append` если текущий capacity исчерпан — создаётся новый буфер с удвоенным размером (capacity × 2), старые элементы копируются. Это делает append O(1) амортизированно. Можно избежать: `arr.reserveCapacity(1000)` — выделить память заранее.

```swift
var arr = [Int]()
print(arr.capacity)  // 0

arr.reserveCapacity(100)
print(arr.capacity)  // ≥ 100 — нет реаллокаций пока count ≤ 100

// ContiguousArray — быстрее для не-ObjC типов:
var fast = ContiguousArray<Int>()  // гарантирует непрерывную память
```

**Как эффективно удалять элементы из массива?**
```swift
// Медленно: удаление в цикле с index
// Правильно: removeAll(where:) — O(n) один проход

var arr = [1, 2, 3, 4, 5, 6]
arr.removeAll(where: { $0.isMultiple(of: 2) })  // [1,3,5]

// Или: filter (создаёт новый массив):
let filtered = arr.filter { !$0.isMultiple(of: 2) }

// Удаление в обратном порядке (для нескольких индексов):
let indicesToRemove = IndexSet([1, 3])
for i in indicesToRemove.reversed() {
    arr.remove(at: i)
}
```

**Что такое `withUnsafeBufferPointer` и зачем?**
```swift
let arr = [1, 2, 3, 4, 5]

// Прямой доступ к памяти для C API или SIMD:
arr.withUnsafeBufferPointer { buffer in
    // buffer.baseAddress — указатель на начало
    // buffer.count — количество элементов
    let sum = buffer.reduce(0, +)  // быстрее для больших массивов
}

// Для SIMD-операций:
arr.withUnsafeBufferPointer { buffer in
    // передать в C функцию
    cFunction(buffer.baseAddress, Int32(buffer.count))
}
```

**Как реализовать binary search для отсортированного массива?**
```swift
extension Array where Element: Comparable {
    func binarySearch(for target: Element) -> Int? {
        var low = 0, high = count - 1
        while low <= high {
            let mid = low + (high - low) / 2  // избегаем overflow
            if self[mid] == target { return mid }
            else if self[mid] < target { low = mid + 1 }
            else { high = mid - 1 }
        }
        return nil
    }
    
    // Найти место вставки (как lower_bound в C++):
    func insertionIndex(for target: Element) -> Int {
        var low = 0, high = count
        while low < high {
            let mid = low + (high - low) / 2
            if self[mid] < target { low = mid + 1 }
            else { high = mid }
        }
        return low
    }
}

let sorted = [1, 3, 5, 7, 9]
sorted.binarySearch(for: 5)  // Optional(2), O(log n)
```

**Чем Array отличается от ContiguousArray и ArraySlice?**
| Тип | Особенность |
|-----|-------------|
| `Array<T>` | Может bridging к NSArray для ObjC-совместимых T |
| `ContiguousArray<T>` | Всегда непрерывная память, нет NSArray overhead |
| `ArraySlice<T>` | View на часть Array, разделяет буфер |
> Для чисто Swift кода с reference types — `ContiguousArray` быстрее. `ArraySlice` не копирует данные но удерживает весь исходный массив.

---

## Set

### Beginner

**Что такое Set в Swift?**
> Неупорядоченная коллекция **уникальных** элементов. `var fruits: Set<String> = ["apple", "banana", "cherry"]`. Элементы должны быть `Hashable`. Нет дубликатов. Нет гарантии порядка. Основан на хеш-таблице — O(1) для большинства операций.

**Как создать Set?**
```swift
let empty: Set<Int> = []
let empty2 = Set<Int>()

let fruits: Set = ["apple", "banana", "apple"]  // {"apple", "banana"} — дубль удалён
let fromArray = Set([1, 2, 2, 3, 3, 3])         // {1, 2, 3}
```

**Как добавить и удалить элемент?**
```swift
var s: Set = [1, 2, 3]

s.insert(4)              // {1,2,3,4}
s.insert(2)              // {1,2,3,4} — дубль игнорируется

let (inserted, member) = s.insert(5)
// inserted = true если новый, false если уже был
// member = вставленный или существующий элемент

s.remove(2)              // Optional(2) если был, nil если нет
s.removeAll()
```

**Как проверить вхождение?**
```swift
let s: Set = [1, 2, 3, 4, 5]
s.contains(3)   // true, O(1) — в отличие от Array O(n)!
s.contains(10)  // false
```

**Что такое операции над множествами?**
```swift
let a: Set = [1, 2, 3, 4]
let b: Set = [3, 4, 5, 6]

// Пересечение (общие):
a.intersection(b)         // {3, 4}

// Объединение (все):
a.union(b)                // {1, 2, 3, 4, 5, 6}

// Разность (в a, но не в b):
a.subtracting(b)          // {1, 2}

// Симметрическая разность (в одном, но не в обоих):
a.symmetricDifference(b)  // {1, 2, 5, 6}

// Мутирующие версии:
var mutable = a
mutable.formIntersection(b)      // mutate in-place
mutable.formUnion(b)
mutable.subtract(b)
mutable.formSymmetricDifference(b)
```

---

### Middle

**Что такое операции сравнения Set?**
```swift
let a: Set = [1, 2, 3]
let b: Set = [1, 2, 3, 4, 5]
let c: Set = [1, 2, 3]

a.isSubset(of: b)         // true  — все элементы a есть в b
b.isSuperset(of: a)       // true  — b содержит все элементы a
a.isDisjoint(with: b)     // false — есть общие элементы

let d: Set = [6, 7, 8]
a.isDisjoint(with: d)     // true  — нет общих элементов

a == c                    // true  — одинаковые элементы
a == b                    // false

// Строгое подмножество (isSubset && не равны):
a.isStrictSubset(of: b)   // true
a.isStrictSubset(of: c)   // false (равны)
```

**Какова сложность операций Set?**
| Операция | Средняя | Худшая |
|----------|---------|--------|
| `insert` | O(1) | O(n) при коллизиях |
| `remove` | O(1) | O(n) |
| `contains` | O(1) | O(n) |
| `union` | O(n+m) | O(n×m) |
| `intersection` | O(min(n,m)) | — |

**Почему Set требует Hashable?**
> Set использует хеш-таблицу для хранения. Хеш значения → bucket index. O(1) доступ. Элементы должны быть `Hashable` (вычислить хеш) и `Equatable` (разрешить коллизии). `Hashable` наследует `Equatable`.

**Как Set ведёт себя при коллизиях хешей?**
> Коллизия: разные элементы с одинаковым хешем попадают в один bucket. В bucket — линейный поиск через `==`. Swift использует открытую адресацию (linear probing): при коллизии ищем следующую свободную позицию. Load factor ~75% → resizing.

**Как реализовать кастомный Hashable для Set?**
```swift
struct Point: Hashable {
    let x: Int
    let y: Int
    
    // Автосинтез при conformance:
    // func hash(into hasher: inout Hasher) {
    //     hasher.combine(x)
    //     hasher.combine(y)
    // }
    // static func == (lhs: Point, rhs: Point) -> Bool { lhs.x == rhs.x && lhs.y == rhs.y }
}

var points: Set<Point> = [Point(x: 1, y: 2), Point(x: 3, y: 4)]
points.contains(Point(x: 1, y: 2))  // true, O(1)
```

**Когда использовать Set вместо Array?**
```
Используй Set когда:
✅ Нужна уникальность элементов
✅ Часто проверяешь contains (O(1) vs O(n))
✅ Нужны операции пересечения/объединения
✅ Порядок не важен

Используй Array когда:
✅ Важен порядок элементов
✅ Нужен доступ по индексу
✅ Элементы не Hashable
✅ Нужны дубликаты
```

**Как удалить дубликаты из Array сохраняя порядок?**
```swift
extension Array where Element: Hashable {
    func uniqued() -> [Element] {
        var seen = Set<Element>()
        return filter { seen.insert($0).inserted }
    }
}

[1, 2, 3, 2, 1, 4].uniqued()  // [1, 2, 3, 4] — порядок сохранён
```

---

### Senior

**Как Set реализован в Swift (хеш-таблица под капотом)?**
> Swift Set = открытая адресация (open addressing) + Robin Hood hashing. Буфер = массив Optional<Element>. При вставке: hash(x) % capacity = index. При коллизии: linear probe (+1, +2...). Robin Hood: элемент с большим displacement вытесняет ближний — уменьшает variance. Resizing при load factor > 3/4.

**Как избежать hash collision degradation?**
```swift
// Плохая хеш-функция → все в одном bucket → O(n):
struct BadHash: Hashable {
    let value: Int
    func hash(into hasher: inout Hasher) {
        hasher.combine(0)  // всегда один bucket!
    }
}

// Хорошая хеш-функция — использует все поля:
struct GoodHash: Hashable {
    let x: Int
    let y: Int
    func hash(into hasher: inout Hasher) {
        hasher.combine(x)
        hasher.combine(y)
    }
}
// Swift использует SipHash-1-3 с random seed (защита от DoS)
```

**Как Set ведёт себя при мутации во время итерации?**
```swift
var s: Set = [1, 2, 3, 4, 5]

// ❌ Изменять во время for-in → runtime error:
// for x in s { s.remove(x) }

// ✅ Итерировать копию:
for x in s where x.isMultiple(of: 2) {
    s.remove(x)
}

// ✅ Или сначала собрать для удаления:
let toRemove = s.filter { $0.isMultiple(of: 2) }
s.subtract(toRemove)
```

---

## Dictionary

### Beginner

**Что такое Dictionary в Swift?**
> Неупорядоченная коллекция пар ключ-значение. `var dict: [String: Int] = ["a": 1, "b": 2]`. Ключи уникальны и `Hashable`. Значения — любой тип. O(1) доступ по ключу. Value type с COW.

**Как создать Dictionary?**
```swift
let empty: [String: Int] = [:]
let empty2 = [String: Int]()
let empty3 = Dictionary<String, Int>()

let dict = ["one": 1, "two": 2, "three": 3]  // литерал

// Из двух массивов:
let keys = ["a", "b", "c"]
let values = [1, 2, 3]
let zipped = Dictionary(uniqueKeysWithValues: zip(keys, values))
// ["a": 1, "b": 2, "c": 3]

// С разрешением дубликатов ключей:
let dupes = [("a", 1), ("b", 2), ("a", 3)]
let merged = Dictionary(dupes, uniquingKeysWith: { first, _ in first })
// ["a": 1, "b": 2] — берёт первое значение при дубликате
```

**Как получить значение?**
```swift
let dict = ["name": "Alice", "city": "Moscow"]

dict["name"]         // Optional("Alice")
dict["age"]          // nil (не Optional("") — просто nil)

// С дефолтным значением:
dict["age", default: "Unknown"]   // "Unknown"
dict["name", default: "Unknown"]  // "Alice"

// Изменить через subscript с default:
var counts = [String: Int]()
counts["apple", default: 0] += 1  // {"apple": 1}
counts["apple", default: 0] += 1  // {"apple": 2}
```

**Как добавить, обновить и удалить?**
```swift
var dict = ["a": 1, "b": 2]

// Добавить/обновить:
dict["c"] = 3          // добавить: {"a":1,"b":2,"c":3}
dict["a"] = 10         // обновить: {"a":10,"b":2,"c":3}

// updateValue — возвращает старое значение:
let old = dict.updateValue(99, forKey: "a")  // old = Optional(10)

// Удалить:
dict["b"] = nil        // удалить ключ "b"
dict.removeValue(forKey: "c")  // Optional(3)
dict.removeAll()
```

**Как итерировать Dictionary?**
```swift
let dict = ["a": 1, "b": 2, "c": 3]

// По парам:
for (key, value) in dict {
    print("\(key): \(value)")
}

// Только ключи:
for key in dict.keys { print(key) }

// Только значения:
for value in dict.values { print(value) }

// Отсортированные ключи:
for key in dict.keys.sorted() {
    print("\(key): \(dict[key]!)")
}
```

---

### Middle

**Что такое `mapValues`, `mapKeys`, `filter` для Dictionary?**
```swift
let prices = ["apple": 1.5, "banana": 0.75, "cherry": 3.0]

// mapValues — трансформировать значения:
let doubled = prices.mapValues { $0 * 2 }
// ["apple": 3.0, "banana": 1.5, "cherry": 6.0]

// filter — отфильтровать пары:
let expensive = prices.filter { $0.value > 1.0 }
// ["apple": 1.5, "cherry": 3.0]

// compactMapValues — map + убрать nil:
let strPrices = ["apple": "1.5", "banana": "invalid", "cherry": "3.0"]
let numPrices = strPrices.compactMapValues { Double($0) }
// ["apple": 1.5, "cherry": 3.0]
```

**Как группировать массив в Dictionary?**
```swift
let words = ["apple", "ant", "banana", "avocado", "blueberry", "cherry"]

// groupBy первой буквой:
let grouped = Dictionary(grouping: words, by: { $0.first! })
// ["a": ["apple","ant","avocado"], "b": ["banana","blueberry"], "c": ["cherry"]]

// groupBy длиной:
let byLength = Dictionary(grouping: words, by: \.count)

// Подсчёт вхождений:
let letters = ["a", "b", "a", "c", "b", "a"]
let counts = Dictionary(grouping: letters, by: { $0 }).mapValues(\.count)
// ["a": 3, "b": 2, "c": 1]

// Или проще:
let counts2 = letters.reduce(into: [:]) { $0[$1, default: 0] += 1 }
```

**Как объединить два Dictionary?**
```swift
let dict1 = ["a": 1, "b": 2]
let dict2 = ["b": 20, "c": 3]

// merge — изменяет reciever:
var merged = dict1
merged.merge(dict2) { old, new in new }  // при конфликте берём new
// ["a": 1, "b": 20, "c": 3]

// merging — возвращает новый:
let result = dict1.merging(dict2) { old, _ in old }  // при конфликте берём old
// ["a": 1, "b": 2, "c": 3]
```

**Какова сложность операций Dictionary?**
| Операция | Средняя | Худшая |
|----------|---------|--------|
| Доступ `dict[key]` | O(1) | O(n) |
| `updateValue` | O(1) | O(n) |
| `removeValue` | O(1) | O(n) |
| `filter` | O(n) | — |
| `mapValues` | O(n) | — |

**Как работает `merge` с closure для разрешения конфликтов?**
```swift
// Аккумуляция значений:
let records = [("alice", 10), ("bob", 5), ("alice", 7), ("bob", 3)]
let totals = records.reduce(into: [:]) { dict, pair in
    dict[pair.0, default: 0] += pair.1
}
// ["alice": 17, "bob": 8]

// Accumulate в массив:
let groups = records.reduce(into: [String: [Int]]()) { dict, pair in
    dict[pair.0, default: []].append(pair.1)
}
// ["alice": [10, 7], "bob": [5, 3]]
```

**Как использовать Dictionary для кэша?**
```swift
class ImageCache {
    private var cache: [URL: UIImage] = [:]
    private let maxSize = 50
    
    func image(for url: URL) -> UIImage? {
        cache[url]
    }
    
    func store(_ image: UIImage, for url: URL) {
        if cache.count >= maxSize {
            cache.removeValue(forKey: cache.keys.first!)  // простая FIFO
        }
        cache[url] = image
    }
}

// Лучше: NSCache для автоматического memory pressure handling
```

**Что такое `KeyValuePairs`?**
> Упорядоченная коллекция ключ-значение (в отличие от Dictionary). Ключи могут повторяться. Нет O(1) доступа. Используется для литералов с повторяющимися ключами или когда важен порядок:
```swift
let ordered: KeyValuePairs = ["a": 1, "b": 2, "a": 3]
// [("a", 1), ("b", 2), ("a", 3)] — порядок сохранён, дубли разрешены
```

---

### Senior

**Как реализовать LRU Cache через Dictionary?**
```swift
class LRUCache<Key: Hashable, Value> {
    private let capacity: Int
    private var cache: [Key: Value] = [:]
    private var order: [Key] = []  // от старых к новым
    
    init(capacity: Int) { self.capacity = capacity }
    
    func get(_ key: Key) -> Value? {
        guard let value = cache[key] else { return nil }
        // Переместить в конец (самый недавний):
        order.removeAll { $0 == key }
        order.append(key)
        return value
    }
    
    func put(_ key: Key, _ value: Value) {
        if cache[key] != nil {
            order.removeAll { $0 == key }
        } else if cache.count >= capacity {
            let oldest = order.removeFirst()
            cache.removeValue(forKey: oldest)
        }
        cache[key] = value
        order.append(key)
    }
}

// Производительная версия — через doubly linked list:
// get/put: O(1) вместо O(n) для order manipulation
```

**Как сделать Dictionary thread-safe?**
```swift
// Вариант 1: Actor
actor SafeDict<Key: Hashable, Value> {
    private var dict: [Key: Value] = [:]
    
    func get(_ key: Key) -> Value? { dict[key] }
    func set(_ key: Key, _ value: Value) { dict[key] = value }
    func remove(_ key: Key) { dict.removeValue(forKey: key) }
}

// Вариант 2: concurrent queue + barrier
class ConcurrentDict<Key: Hashable, Value> {
    private var dict: [Key: Value] = [:]
    private let queue = DispatchQueue(label: "dict", attributes: .concurrent)
    
    func get(_ key: Key) -> Value? {
        queue.sync { dict[key] }
    }
    
    func set(_ key: Key, _ value: Value) {
        queue.async(flags: .barrier) { self.dict[key] = value }
    }
}
```

**Как Dictionary хранится в памяти?**
> Открытая адресация: внутренний буфер (массив) из (Key, Value) пар + bitmap занятости. Capacity = степень 2 (для быстрого modulo через bitwise AND). При load factor > 3/4 → realloc + rehash всех элементов. Swift Dictionary хранит ключ и значение вместе (cache friendly).

---

## Практические задачи

### Задача 1: Два Sum
```swift
// Найти индексы двух чисел которые в сумме дают target
// Сложность: O(n) через Dictionary

func twoSum(_ nums: [Int], _ target: Int) -> [Int] {
    var seen: [Int: Int] = [:]  // value: index
    
    for (i, num) in nums.enumerated() {
        let complement = target - num
        if let j = seen[complement] {
            return [j, i]
        }
        seen[num] = i
    }
    return []
}

twoSum([2, 7, 11, 15], 9)  // [0, 1]
twoSum([3, 2, 4], 6)       // [1, 2]
```

### Задача 2: Анаграммы
```swift
// Проверить являются ли две строки анаграммами
// "listen" <-> "silent" → true

func isAnagram(_ s: String, _ t: String) -> Bool {
    guard s.count == t.count else { return false }
    
    // Способ 1: Dictionary
    var counts = [Character: Int]()
    for char in s { counts[char, default: 0] += 1 }
    for char in t {
        counts[char, default: 0] -= 1
        if counts[char]! < 0 { return false }
    }
    return true
    
    // Способ 2: sorted
    // return s.sorted() == t.sorted()  // O(n log n)
}

// Сгруппировать анаграммы:
func groupAnagrams(_ strs: [String]) -> [[String]] {
    var groups: [[Character]: [String]] = [:]
    for str in strs {
        let key = str.sorted()
        groups[key, default: []].append(str)
    }
    return Array(groups.values)
}

groupAnagrams(["eat","tea","tan","ate","nat","bat"])
// [["eat","tea","ate"],["tan","nat"],["bat"]]
```

### Задача 3: Наиболее частый элемент
```swift
// Top K Frequent Elements
// [1,1,1,2,2,3], k=2 → [1,2]

func topKFrequent(_ nums: [Int], _ k: Int) -> [Int] {
    // Подсчёт частот:
    let freq = nums.reduce(into: [:]) { $0[$1, default: 0] += 1 }
    
    // Сортировка по частоте:
    return freq.sorted { $0.value > $1.value }
               .prefix(k)
               .map(\.key)
}

// O(n log n) через sort
// O(n) через bucket sort:
func topKFrequentBucket(_ nums: [Int], _ k: Int) -> [Int] {
    let freq = nums.reduce(into: [:]) { $0[$1, default: 0] += 1 }
    
    var buckets = [[Int]](repeating: [], count: nums.count + 1)
    for (num, count) in freq {
        buckets[count].append(num)
    }
    
    var result: [Int] = []
    for i in stride(from: buckets.count - 1, through: 0, by: -1) {
        result.append(contentsOf: buckets[i])
        if result.count >= k { break }
    }
    return Array(result.prefix(k))
}
```

### Задача 4: Longest Consecutive Sequence
```swift
// Найти длину самой длинной последовательности подряд идущих чисел
// [100, 4, 200, 1, 3, 2] → 4 (1,2,3,4)
// O(n)

func longestConsecutive(_ nums: [Int]) -> Int {
    let numSet = Set(nums)  // O(1) contains
    var longest = 0
    
    for num in numSet {
        // Начинаем только если num-1 не в set (начало последовательности)
        guard !numSet.contains(num - 1) else { continue }
        
        var current = num
        var length = 1
        
        while numSet.contains(current + 1) {
            current += 1
            length += 1
        }
        
        longest = max(longest, length)
    }
    
    return longest
}

longestConsecutive([100, 4, 200, 1, 3, 2])  // 4
```

### Задача 5: Пересечение массивов
```swift
// Найти общие элементы двух массивов с учётом количества вхождений
// [1,2,2,1] ∩ [2,2] → [2,2]

func intersect(_ nums1: [Int], _ nums2: [Int]) -> [Int] {
    var counts = nums1.reduce(into: [:]) { $0[$1, default: 0] += 1 }
    
    var result: [Int] = []
    for num in nums2 {
        if let count = counts[num], count > 0 {
            result.append(num)
            counts[num]! -= 1
        }
    }
    return result
}

intersect([1,2,2,1], [2,2])     // [2,2]
intersect([4,9,5], [9,4,9,8,4]) // [9,4] или [4,9]
```

### Задача 6: Subarray Sum equals K
```swift
// Найти количество подмассивов с суммой равной k
// [1,1,1], k=2 → 2  (индексы [0,1] и [1,2])
// O(n) через prefix sum + Dictionary

func subarraySum(_ nums: [Int], _ k: Int) -> Int {
    var prefixSumCount: [Int: Int] = [0: 1]  // {prefix_sum: count}
    var currentSum = 0
    var result = 0
    
    for num in nums {
        currentSum += num
        // Ищем: currentSum - k = предыдущий prefix sum
        result += prefixSumCount[currentSum - k, default: 0]
        prefixSumCount[currentSum, default: 0] += 1
    }
    
    return result
}

subarraySum([1, 1, 1], 2)    // 2
subarraySum([1, 2, 3], 3)    // 2 (подмассивы [1,2] и [3])
```

### Задача 7: Valid Sudoku
```swift
// Проверить валидность судоку
// Каждая строка, столбец и 3x3 квадрат без повторений 1-9

func isValidSudoku(_ board: [[Character]]) -> Bool {
    var rows = [Set<Character>](repeating: [], count: 9)
    var cols = [Set<Character>](repeating: [], count: 9)
    var boxes = [Set<Character>](repeating: [], count: 9)
    
    for i in 0..<9 {
        for j in 0..<9 {
            let char = board[i][j]
            guard char != "." else { continue }
            
            let boxIndex = (i / 3) * 3 + j / 3
            
            if rows[i].contains(char) || cols[j].contains(char) || boxes[boxIndex].contains(char) {
                return false
            }
            
            rows[i].insert(char)
            cols[j].insert(char)
            boxes[boxIndex].insert(char)
        }
    }
    return true
}
```

### Задача 8: Sliding Window Maximum
```swift
// Максимум в каждом окне размера k
// [1,3,-1,-3,5,3,6,7], k=3 → [3,3,5,5,6,7]
// O(n) через deque (двусторонняя очередь)

func maxSlidingWindow(_ nums: [Int], _ k: Int) -> [Int] {
    var deque: [Int] = []  // индексы, убывающие по значению
    var result: [Int] = []
    
    for i in 0..<nums.count {
        // Убрать индексы вне окна:
        while !deque.isEmpty && deque.first! <= i - k {
            deque.removeFirst()
        }
        // Убрать индексы меньших элементов:
        while !deque.isEmpty && nums[deque.last!] < nums[i] {
            deque.removeLast()
        }
        deque.append(i)
        
        if i >= k - 1 {
            result.append(nums[deque.first!])
        }
    }
    return result
}
```

### Задача 9: Подсчёт слов в тексте
```swift
// Разобрать текст, посчитать частоту слов, вернуть топ-N

func topWords(in text: String, top n: Int) -> [(word: String, count: Int)] {
    let words = text.lowercased()
        .components(separatedBy: .punctuationCharacters).joined()
        .components(separatedBy: .whitespaces)
        .filter { !$0.isEmpty }
    
    let frequency = words.reduce(into: [String: Int]()) {
        $0[$1, default: 0] += 1
    }
    
    return frequency
        .sorted { $0.value > $1.value }
        .prefix(n)
        .map { (word: $0.key, count: $0.value) }
}

let text = "the quick brown fox jumps over the lazy dog the fox"
topWords(in: text, top: 3)
// [(word: "the", count: 3), (word: "fox", count: 2), ...]
```

### Задача 10: Flatten вложенного Dictionary
```swift
// Превратить nested dictionary в flat с dot-notation ключами
// {"a": {"b": {"c": 1}}, "d": 2}
// → {"a.b.c": 1, "d": 2}

func flatten(_ dict: [String: Any], prefix: String = "") -> [String: Any] {
    var result: [String: Any] = [:]
    
    for (key, value) in dict {
        let newKey = prefix.isEmpty ? key : "\(prefix).\(key)"
        
        if let nested = value as? [String: Any] {
            result.merge(flatten(nested, prefix: newKey)) { _, new in new }
        } else {
            result[newKey] = value
        }
    }
    return result
}

let nested: [String: Any] = ["a": ["b": ["c": 1]], "d": 2, "e": ["f": 3]]
flatten(nested)  // ["a.b.c": 1, "d": 2, "e.f": 3]
```

### Задача 11: Операции с Set — множества
```swift
// Дано: список друзей каждого пользователя
// Найти: общих друзей двух пользователей

let friends: [String: Set<String>] = [
    "Alice": ["Bob", "Charlie", "Dave"],
    "Bob": ["Alice", "Charlie", "Eve"],
    "Charlie": ["Alice", "Bob", "Frank"]
]

func mutualFriends(of user1: String, and user2: String) -> Set<String> {
    guard let f1 = friends[user1], let f2 = friends[user2] else { return [] }
    return f1.intersection(f2).subtracting([user1, user2])
}

mutualFriends(of: "Alice", and: "Bob")  // {"Charlie"}

// Рекомендовать друзей (друзья друзей, которых нет в твоих друзьях):
func recommendations(for user: String) -> Set<String> {
    guard let myFriends = friends[user] else { return [] }
    
    var candidates = Set<String>()
    for friend in myFriends {
        if let friendsFriends = friends[friend] {
            candidates.formUnion(friendsFriends)
        }
    }
    candidates.subtract(myFriends)
    candidates.remove(user)
    return candidates
}
```

### Задача 12: Минимальное окно с подстрокой (Minimum Window Substring)
```swift
// Найти наименьшую подстроку s содержащую все символы t
// s = "ADOBECODEBANC", t = "ABC" → "BANC"
// O(n) sliding window + Dictionary

func minWindow(_ s: String, _ t: String) -> String {
    guard !s.isEmpty && !t.isEmpty else { return "" }
    
    var need = t.reduce(into: [Character: Int]()) { $0[$1, default: 0] += 1 }
    var window = [Character: Int]()
    let chars = Array(s)
    
    var have = 0, required = need.count
    var left = 0
    var result = ""
    var resultLen = Int.max
    
    for right in 0..<chars.count {
        let c = chars[right]
        window[c, default: 0] += 1
        
        if let needed = need[c], window[c]! == needed {
            have += 1
        }
        
        while have == required {
            if right - left + 1 < resultLen {
                resultLen = right - left + 1
                result = String(chars[left...right])
            }
            
            let leftChar = chars[left]
            window[leftChar]! -= 1
            if let needed = need[leftChar], window[leftChar]! < needed {
                have -= 1
            }
            left += 1
        }
    }
    return result
}
```

---

## Шпаргалка: сравнение коллекций

```
              Array        Set           Dictionary
──────────────────────────────────────────────────
Порядок:      ✅ да        ❌ нет        ❌ нет
Дубликаты:    ✅ да        ❌ нет        ❌ (ключи)
Доступ:       O(1) индекс O(1) Hashable O(1) ключ
Contains:     O(n)         O(1)          O(1) ключ
Вставка:      O(1) append  O(1)          O(1)
      insert: O(n)
Удаление:     O(n)         O(1)          O(1)
Сортировка:   O(n log n)  нет           нет
Используй:    порядок важен уникальность ключ→значение
              index нужен  contains     lookups

Все три — Value type с COW
Все три — Generic: [T], Set<T: Hashable>, [K:V]
```

---

*Конец документа · Array, Set, Dictionary*
