# Combine — Вопросы, ответы и практические задачи
> iOS-собеседование · Combine Framework · Reactive Programming
> Уровни: Beginner · Middle · Senior + Практические задачи

---

## Publisher & Subscriber

### Beginner

**Что такое Publisher в Combine?**
> Протокол описывающий тип который может публиковать последовательность значений во времени. `protocol Publisher { associatedtype Output; associatedtype Failure: Error; func receive<S: Subscriber>(subscriber: S) }`. Примеры: `Just`, `Future`, `PassthroughSubject`, `URLSession.dataTaskPublisher`. Не производит значения без subscriber.

**Что такое Subscriber?**
> Протокол получающий значения от Publisher. `protocol Subscriber { associatedtype Input; associatedtype Failure: Error; func receive(subscription: Subscription); func receive(_ input: Input) -> Subscribers.Demand; func receive(completion: Subscribers.Completion<Failure>) }`. Встроенные: `sink`, `assign`. Subscriber контролирует backpressure через `Demand`.

**Что такое Subscription?**
> Протокол представляющий связь между Publisher и Subscriber. Создаётся Publisher при подписке. `protocol Subscription: Cancellable { func request(_ demand: Subscribers.Demand) }`. Subscriber вызывает `request` чтобы запросить значения. `cancel()` — завершить подписку. Хранится в `AnyCancellable`.

**Что такое `sink`?**
> Встроенный subscriber принимающий значения через замыкания. Две формы:
```swift
// Только значения (Failure == Never):
publisher.sink { value in print(value) }

// Значения + completion:
publisher.sink(
    receiveCompletion: { completion in
        switch completion {
        case .finished: print("Done")
        case .failure(let error): print("Error: \(error)")
        }
    },
    receiveValue: { value in print(value) }
)
```
> Возвращает `AnyCancellable`. Запрашивает `.unlimited` demand сразу.

**Что такое `assign`?**
> Subscriber автоматически присваивающий значения свойству объекта через KeyPath:
```swift
class ViewModel {
    @Published var title = ""
}
let vm = ViewModel()

cancellable = publisher
    .assign(to: \.title, on: vm)

// iOS 14+: assign(to:) для @Published без retain cycle:
publisher
    .assign(to: &vm.$title)  // не нужен AnyCancellable!
```
> `assign(to:on:)` — требует хранить `AnyCancellable`. `assign(to:)` — автоматически управляет lifecycle через `@Published`.

**Что такое `Just`?**
> Publisher публикующий одно значение и сразу завершающийся. `Just(42).sink { print($0) }` → `42, finished`. `Failure == Never` — никогда не бросает ошибку. Полезен для тестов и дефолтных значений.

**Что такое `Future`?**
> Publisher доставляющий одно async значение. Closure вызывается немедленно при создании (не при подписке!). `Future<Int, Error> { promise in DispatchQueue.global().async { promise(.success(42)) } }`. Кэширует результат — все последующие subscribers получают то же значение.

---

### Middle

**Что такое `PassthroughSubject`?**
> Subject (Publisher + Subscriber) без хранения текущего значения. Передаёт значения только активным subscribers в момент отправки. Новый subscriber не получит прошлые значения.
```swift
let subject = PassthroughSubject<Int, Never>()

let sub1 = subject.sink { print("Sub1:", $0) }
subject.send(1)  // Sub1: 1

let sub2 = subject.sink { print("Sub2:", $0) }
subject.send(2)  // Sub1: 2, Sub2: 2
// sub2 не получил 1!

subject.send(completion: .finished)
```

**Что такое `CurrentValueSubject`?**
> Subject хранящий и публикующий текущее значение. Новый subscriber сразу получает текущее значение.
```swift
let subject = CurrentValueSubject<Int, Never>(0)

print(subject.value)  // 0 — доступно синхронно

subject.send(1)

let sub = subject.sink { print($0) }  // сразу напечатает: 1
subject.send(2)  // 2
```

**Чем PassthroughSubject отличается от CurrentValueSubject?**
| | PassthroughSubject | CurrentValueSubject |
|--|--|--|
| Хранит значение | ❌ | ✅ (`.value`) |
| Новый subscriber | ничего не получает | получает текущее значение |
| Использование | события (tap, notification) | состояние (isLoading, count) |
| Аналог | EventEmitter | BehaviorSubject (RxSwift) |

**Что такое `AnyPublisher`?**
> Type-erased wrapper скрывающий конкретный тип Publisher. `AnyPublisher<Output, Failure>`. Используется в API: не раскрывать внутреннюю реализацию цепочки операторов. `URLSession.shared.dataTaskPublisher(for: url)` возвращает конкретный тип — в API возвращай `AnyPublisher<Data, Error>`.

**Что такое `eraseToAnyPublisher`?**
> Метод превращающий любой Publisher в `AnyPublisher`:
```swift
func fetchUser(id: Int) -> AnyPublisher<User, Error> {
    URLSession.shared.dataTaskPublisher(for: url)
        .map(\.data)
        .decode(type: User.self, decoder: JSONDecoder())
        .eraseToAnyPublisher()  // скрываем длинный generic тип
}
// Без eraseToAnyPublisher тип был бы:
// Publishers.Decode<Publishers.Map<URLSession.DataTaskPublisher, Data>, User, JSONDecoder>
```

**Что такое `AnyCancellable`?**
> Type-erased `Cancellable`. Хранит подписку и отменяет её при deinit или `.cancel()`.
```swift
// Хранить одну:
var cancellable: AnyCancellable?
cancellable = publisher.sink { ... }

// Хранить несколько:
var cancellables = Set<AnyCancellable>()
publisher.sink { ... }.store(in: &cancellables)

// Отмена всех:
cancellables.removeAll()
```
> Если не хранить — подписка отменяется немедленно.

**Как управлять lifecycle подписки?**
```swift
class ViewModel {
    private var cancellables = Set<AnyCancellable>()
    
    init() {
        // Живёт пока ViewModel
        publisher
            .sink { value in self.handle(value) }
            .store(in: &cancellables)
    }
    
    deinit {
        cancellables.removeAll()  // опционально — произойдёт автоматически
    }
    
    func cancelSpecific() {
        // Для отмены конкретной:
        let specific = publisher.sink { ... }
        // specific.cancel() — немедленная отмена
    }
}
```

**Что такое backpressure в Combine?**
> Механизм контроля скорости производства данных со стороны Consumer. Subscriber запрашивает ровно столько значений сколько может обработать через `Demand`. Предотвращает переполнение когда Publisher производит быстрее чем Subscriber потребляет.

**Что такое `Demand`?**
> Запрос количества значений от Subscriber к Publisher. `Subscribers.Demand`:
```swift
.unlimited  // принять все
.none       // ничего не присылать
.max(N)     // не более N значений

// В receive(_ input:) -> Demand:
func receive(_ input: Int) -> Subscribers.Demand {
    return .max(1)  // запросить ещё 1 после получения текущего
}
```
> Demand аддитивен: `.max(3)` + `.max(2)` = `.max(5)`. Уменьшить нельзя.

**Как реализовать кастомный Publisher?**
```swift
struct TimerPublisher: Publisher {
    typealias Output = Date
    typealias Failure = Never
    
    let interval: TimeInterval
    
    func receive<S: Subscriber>(subscriber: S) where S.Input == Date, S.Failure == Never {
        let subscription = TimerSubscription(subscriber: subscriber, interval: interval)
        subscriber.receive(subscription: subscription)
    }
}

final class TimerSubscription<S: Subscriber>: Subscription where S.Input == Date, S.Failure == Never {
    private var subscriber: S?
    private var timer: Timer?
    
    init(subscriber: S, interval: TimeInterval) {
        self.subscriber = subscriber
        timer = Timer.scheduledTimer(withTimeInterval: interval, repeats: true) { [weak self] _ in
            _ = self?.subscriber?.receive(Date())
        }
    }
    
    func request(_ demand: Subscribers.Demand) { }  // ignore backpressure
    
    func cancel() {
        timer?.invalidate()
        timer = nil
        subscriber = nil
    }
}
```

---

### Senior

**Как Combine реализует backpressure?**
> Через протокол `Subscription.request(_:)`. Subscriber вызывает `request(.max(N))` — Publisher отправляет не более N значений. При `receive(_:) -> Demand` возвращает дополнительный спрос. `sink` запрашивает `.unlimited` сразу. Кастомные subscribers могут точно контролировать поток.

**Что такое `Subscription` протокол?**
```swift
public protocol Subscription: Cancellable, AnyObject {
    func request(_ demand: Subscribers.Demand)
}
```
> Создаётся Publisher при получении subscriber. Lifetime подписки. `request` — уведомить Publisher о готовности принять значения. `cancel()` — завершить подписку и освободить ресурсы. Subscriber хранит subscription и вызывает `request` в `receive(subscription:)`.

**Как работает граф Publishers?**
> Каждый оператор создаёт новый Publisher обёртку. При подписке: subscriber подписывается на последний оператор → тот на предыдущий → до источника. Subscription chain создаётся снизу вверх. Значения идут сверху вниз. Cancellation propagates вверх через subscription chain.

```
Source → map → filter → sink
  ↑        ↑       ↑       ↓
  ←subscription chain←
```

**Что такое hot vs cold publisher?**
| | Cold Publisher | Hot Publisher |
|--|--|--|
| Начало работы | При подписке каждого subscriber | Независимо от subscribers |
| Данные | Уникальные для каждого subscriber | Разделяются между всеми |
| Примеры | `URLSession.dataTaskPublisher`, `Just` | `PassthroughSubject`, Timer |
| Пропуск значений | Нет | Да (если нет subscriber) |
> `share()` превращает cold в hot.

**Как реализовать собственный оператор в Combine?**
```swift
// Простой способ — extension на Publisher:
extension Publisher {
    func filterNil<T>() -> Publishers.CompactMap<Self, T> where Output == T? {
        compactMap { $0 }
    }
    
    func withPrevious() -> AnyPublisher<(Output?, Output), Failure> {
        scan((nil, nil) as (Output?, Output?)) { ($0.1, $1) }
            .compactMap { prev, curr -> (Output?, Output)? in
                guard let curr = curr else { return nil }
                return (prev, curr)
            }
            .eraseToAnyPublisher()
    }
}

// Использование:
[1, nil, 3, nil, 5].publisher.filterNil().sink { print($0) }  // 1, 3, 5

[1, 2, 3].publisher
    .withPrevious()
    .sink { prev, curr in print("prev: \(String(describing: prev)), curr: \(curr)") }
// prev: nil, curr: 1
// prev: 1, curr: 2
// prev: 2, curr: 3
```

**Как Combine взаимодействует с async/await?**
```swift
// Publisher → async:
let value = try await publisher.values.first(where: { $0 > 0 })

// Итерация через AsyncSequence:
for await value in publisher.values {
    print(value)
}

// Async → Publisher (через Future):
func asyncToPublisher() -> AnyPublisher<Data, Error> {
    Future { promise in
        Task {
            do {
                let data = try await URLSession.shared.data(from: url).0
                promise(.success(data))
            } catch {
                promise(.failure(error))
            }
        }
    }.eraseToAnyPublisher()
}

// @Published → async:
class ViewModel: ObservableObject {
    @Published var value = 0
}
let vm = ViewModel()
Task { for await v in vm.$value.values { print(v) } }
```

**Как тестировать Combine Publishers?**
```swift
import XCTest
import Combine

class CombineTests: XCTestCase {
    var cancellables = Set<AnyCancellable>()
    
    // Синхронный publisher:
    func test_justPublisher() {
        var received: [Int] = []
        Just(42).sink { received.append($0) }.store(in: &cancellables)
        XCTAssertEqual(received, [42])
    }
    
    // Async publisher:
    func test_asyncPublisher() {
        let expectation = expectation(description: "received value")
        var result: Int?
        
        someAsyncPublisher
            .sink(receiveCompletion: { _ in },
                  receiveValue: { value in
                      result = value
                      expectation.fulfill()
                  })
            .store(in: &cancellables)
        
        waitForExpectations(timeout: 1)
        XCTAssertEqual(result, 42)
    }
    
    // Тест с mock subject:
    func test_searchViewModel() {
        let subject = PassthroughSubject<String, Never>()
        let vm = SearchViewModel(searchPublisher: subject.eraseToAnyPublisher())
        
        subject.send("swift")
        XCTAssertEqual(vm.searchQuery, "swift")
    }
}
```

**Что такое `share()` и `multicast()`?**
```swift
// share() — горячий publisher из холодного, shared между subscribers:
let shared = URLSession.shared.dataTaskPublisher(for: url)
    .share()  // один HTTP запрос для всех subscribers

shared.sink { _ in }.store(in: &cancellables)  // subscribes
shared.sink { _ in }.store(in: &cancellables)  // reuses same request

// multicast() — ручной контроль когда начать:
let subject = PassthroughSubject<Data, URLError>()
let multicast = URLSession.shared.dataTaskPublisher(for: url)
    .multicast(subject: subject)

multicast.sink { _ in }.store(in: &cancellables)
multicast.sink { _ in }.store(in: &cancellables)

let connection = multicast.connect()  // теперь запускаем
// connection.cancel() — остановить
```

**Как реализовать retry в Combine?**
```swift
// Базовый retry:
URLSession.shared.dataTaskPublisher(for: url)
    .retry(3)  // 3 попытки при ошибке
    .sink(receiveCompletion: { print($0) }, receiveValue: { _ in })

// Retry с задержкой:
func retryWithDelay<T, E: Error>(
    _ publisher: AnyPublisher<T, E>,
    retries: Int,
    delay: TimeInterval
) -> AnyPublisher<T, E> {
    publisher
        .catch { error -> AnyPublisher<T, E> in
            guard retries > 0 else { return Fail(error: error).eraseToAnyPublisher() }
            return retryWithDelay(
                publisher.delay(for: .seconds(delay), scheduler: DispatchQueue.main).eraseToAnyPublisher(),
                retries: retries - 1,
                delay: delay * 2  // exponential backoff
            )
        }
        .eraseToAnyPublisher()
}
```

**Что такое `switchToLatest`?**
```swift
// Отменяет предыдущий inner publisher при получении нового:
let searchSubject = PassthroughSubject<String, Never>()

searchSubject
    .map { query in
        URLSession.shared.dataTaskPublisher(for: searchURL(query))
            .map(\.data)
            .replaceError(with: Data())
    }
    .switchToLatest()  // отменяет предыдущий запрос при новом query
    .sink { data in updateUI(data) }

// Без switchToLatest — все запросы выполняются, результаты могут прийти в неправильном порядке
// С switchToLatest — только последний запрос активен
```

---

## Combine Operators

### Beginner

**Что такое `map` в Combine?**
> Трансформирует каждое значение через замыкание:
```swift
[1, 2, 3].publisher
    .map { $0 * 2 }
    .sink { print($0) }
// 2, 4, 6

// KeyPath map:
struct User { var name: String }
userPublisher
    .map(\.name)
    .sink { print($0) }

// Можно трансформировать тип:
["1", "2", "3"].publisher
    .map { Int($0)! }
    .sink { print($0) }  // Int: 1, 2, 3
```

**Что такое `filter`?**
> Пропускает только значения удовлетворяющие предикату:
```swift
(1...10).publisher
    .filter { $0.isMultiple(of: 2) }
    .sink { print($0) }
// 2, 4, 6, 8, 10

// Не завершается при filter — completion propagates как обычно
// Нет значений → тишина, потом completion
```

**Что такое `debounce`?**
> Публикует значение только если после него прошла заданная пауза без новых значений:
```swift
let searchField = PassthroughSubject<String, Never>()

searchField
    .debounce(for: .milliseconds(300), scheduler: DispatchQueue.main)
    .sink { query in performSearch(query) }

searchField.send("s")      // отменено — пришло следующее
searchField.send("sw")     // отменено
searchField.send("swift")  // через 300ms после → performSearch("swift")
```
> Идеален для: поиск при вводе, автосохранение, resize events.

**Что такое `throttle`?**
> Публикует значение не чаще заданного интервала:
```swift
button.tap
    .throttle(for: .seconds(1), scheduler: DispatchQueue.main, latest: false)
    .sink { handleTap() }

// latest: true  → публикует последнее значение за интервал
// latest: false → публикует первое значение за интервал
// В отличие от debounce — не ждёт паузы, гарантирует регулярные события
```

---

### Middle

**Чем `debounce` отличается от `throttle`?**
| | debounce | throttle |
|--|--|--|
| Когда публикует | После паузы N сек без событий | Первое/последнее за N сек |
| Гарантирует регулярность | ❌ | ✅ |
| При непрерывном потоке | Ничего не публикует | Публикует раз в N сек |
| Сценарий | Поиск при вводе | Rate limiting, кнопка |

```swift
// debounce: ждёт тишины
// Ввод: a...b...c...пауза → "abc"

// throttle: публикует с интервалом
// Ввод: a b c d → a ... d (каждую секунду)
```

**Что такое `flatMap`?**
> Трансформирует каждое значение в Publisher и подписывается на него. Мержит все inner publishers:
```swift
// Поиск: каждый запрос создаёт новый Publisher
searchSubject
    .flatMap { query in
        URLSession.shared.dataTaskPublisher(for: searchURL(query))
            .map(\.data)
            .catch { _ in Empty() }
    }
    .sink { data in updateUI(data) }

// Параметр maxPublishers — ограничить concurrent publishers:
.flatMap(maxPublishers: .max(2)) { ... }

// Проблема: если приходит новый query — предыдущий запрос НЕ отменяется
// Решение: switchToLatest или flatMap + сам отменяй
```

**Что такое `merge`?**
> Объединяет несколько Publishers одного типа в один:
```swift
let p1 = PassthroughSubject<Int, Never>()
let p2 = PassthroughSubject<Int, Never>()
let p3 = PassthroughSubject<Int, Never>()

Publishers.Merge3(p1, p2, p3)
    .sink { print($0) }

p1.send(1)  // 1
p3.send(3)  // 3
p2.send(2)  // 2
// порядок: в порядке отправки

// Или через метод:
p1.merge(with: p2, p3)
// завершается когда ВСЕ источники завершились
```

**Что такое `zip`?**
> Ждёт по одному значению от каждого Publisher и отправляет их как пару:
```swift
let p1 = PassthroughSubject<Int, Never>()
let p2 = PassthroughSubject<String, Never>()

p1.zip(p2)
    .sink { int, string in print("\(int): \(string)") }

p1.send(1)
p1.send(2)
p2.send("A")  // (1, "A") — пара с первым p1
p2.send("B")  // (2, "B") — пара со вторым p1
// p1.send(3) — будет ждать от p2
```

**Что такое `combineLatest`?**
> Публикует комбинацию последних значений от каждого Publisher при любом изменении:
```swift
let username = CurrentValueSubject<String, Never>("")
let password = CurrentValueSubject<String, Never>("")

username.combineLatest(password)
    .map { user, pass in !user.isEmpty && pass.count >= 6 }
    .assign(to: &$isLoginEnabled)

username.send("john")    // (john, "") → false
password.send("secret")  // (john, secret) → true
username.send("")        // ("", secret) → false
```
> Требует минимум одно значение от каждого. Идеален для form validation.

**Что такое `catch`?**
> Обрабатывает ошибку заменяя publisher на другой:
```swift
URLSession.shared.dataTaskPublisher(for: url)
    .catch { error in
        // возвращаем fallback publisher
        Just(Data()).setFailureType(to: URLError.self)
    }
    .sink { data in handle(data) }

// После catch — pipeline продолжается без ошибки
// catch вызывается один раз при первой ошибке
// После catch оригинальный publisher завершён

// tryMap + catch:
publisher
    .tryMap { try parse($0) }
    .catch { _ in Just(defaultValue) }
```

**Что такое `retry`?**
> Повторяет подписку на upstream при ошибке:
```swift
// Повторить 3 раза:
URLSession.shared.dataTaskPublisher(for: url)
    .retry(3)

// Важно: retry пересоздаёт подписку с нуля
// Для URLSession — выполняет новый HTTP запрос
// Без retry(0) — не повторяет (0 = нет retry)

// После исчерпания retry — ошибка propagates дальше
// Комбо: retry + catch:
publisher
    .retry(3)
    .catch { _ in fallbackPublisher }
```

**Что такое `receive(on:)`?**
> Переключает доставку значений на указанный Scheduler (поток):
```swift
URLSession.shared.dataTaskPublisher(for: url)
    .map(\.data)
    .decode(type: User.self, decoder: JSONDecoder())
    .receive(on: DispatchQueue.main)  // переключаемся на main thread
    .sink { user in
        self.label.text = user.name  // UI update — всегда на main
    }

// receive(on:) влияет только на downstream от этой точки
// subscribe(on:) влияет на upstream (где выполняется работа)
```

**Что такое `subscribe(on:)`?**
> Определяет Scheduler на котором выполняется подписка и работа upstream:
```swift
loadDataPublisher()
    .subscribe(on: DispatchQueue.global(qos: .background))  // тяжёлая работа в background
    .receive(on: DispatchQueue.main)  // результат — на main thread
    .sink { data in updateUI(data) }

// subscribe(on:) — где СОЗДАЁТСЯ и работает publisher
// receive(on:)   — где ДОСТАВЛЯЮТСЯ значения subscriber'у
```

**Что такое `removeDuplicates`?**
> Фильтрует последовательные одинаковые значения:
```swift
[1, 1, 2, 2, 3, 1, 1].publisher
    .removeDuplicates()
    .sink { print($0) }
// 1, 2, 3, 1 — убирает только ПОСЛЕДОВАТЕЛЬНЫЕ дубли

// Кастомное сравнение:
publisher
    .removeDuplicates { prev, curr in
        prev.id == curr.id  // сравниваем по id
    }

// Полезно: предотвратить лишние UI перерисовки
viewModel.$state
    .removeDuplicates()
    .sink { updateUI($0) }
```

---

### Senior

**Как `flatMap` и `switchToLatest` решают проблему overlapping requests?**
```swift
// Проблема: пользователь быстро вводит → множество запросов
// Результаты могут прийти в неправильном порядке

// flatMap — выполняет ВСЕ запросы одновременно:
searchSubject
    .flatMap { search(for: $0) }  // 3 запроса параллельно, неопределённый порядок

// switchToLatest — отменяет предыдущий при новом:
searchSubject
    .map { search(for: $0) }     // создаёт publisher
    .switchToLatest()             // подписывается только на последний

// Или через flatMap с maxPublishers:
searchSubject
    .flatMap(maxPublishers: .max(1)) { search(for: $0) }  // serial, не параллельно
```

**Что такое `withLatestFrom` и как реализовать его вручную?**
> Оператор не из стандартного Combine (есть в CombineExt). При каждом значении от source берёт последнее значение из другого publisher:
```swift
// Реализация вручную:
extension Publisher {
    func withLatestFrom<P: Publisher>(_ other: P) -> AnyPublisher<(Output, P.Output), Failure>
    where P.Failure == Failure {
        self.flatMap { value in
            other.first().map { (value, $0) }
        }.eraseToAnyPublisher()
    }
}

// Или через combineLatest:
button.tap
    .flatMap { [weak self] _ in
        (self?.viewModel.$data ?? Just(nil).eraseToAnyPublisher())
            .first()
    }
    .compactMap { $0 }
    .sink { data in handle(data) }
```

**Что такое `scan` оператор?**
> Аккумулятор: применяет функцию к предыдущему результату и текущему значению:
```swift
// Текущая сумма:
(1...5).publisher
    .scan(0) { accumulator, value in accumulator + value }
    .sink { print($0) }
// 1, 3, 6, 10, 15

// Running count:
events.publisher
    .scan(0) { count, _ in count + 1 }
    .sink { print("Total events: \($0)") }

// Аккумуляция в массив:
values.publisher
    .scan([]) { array, value in array + [value] }
    .sink { print($0) }
// [1], [1,2], [1,2,3]...
```

**Что такое `collect`?**
> Собирает значения в массив:
```swift
// Собрать все до completion:
(1...5).publisher
    .collect()
    .sink { print($0) }  // [1, 2, 3, 4, 5]

// Группировать по N:
(1...10).publisher
    .collect(3)
    .sink { print($0) }
// [1,2,3], [4,5,6], [7,8,9], [10]

// По времени:
publisher
    .collect(.byTime(DispatchQueue.main, .seconds(1)))
    .sink { batch in process(batch) }  // каждую секунду

// По времени ИЛИ количеству:
publisher
    .collect(.byTimeOrCount(DispatchQueue.main, .seconds(1), 10))
```

**Как реализовать поиск с debounce и отменой предыдущего?**
```swift
class SearchViewModel: ObservableObject {
    @Published var searchText = ""
    @Published var results: [String] = []
    @Published var isLoading = false
    
    private var cancellables = Set<AnyCancellable>()
    
    init() {
        $searchText
            .debounce(for: .milliseconds(300), scheduler: DispatchQueue.main)
            .removeDuplicates()
            .filter { !$0.isEmpty }
            .handleEvents(receiveOutput: { [weak self] _ in
                self?.isLoading = true
            })
            .flatMap { [weak self] query -> AnyPublisher<[String], Never> in
                guard let self else { return Just([]).eraseToAnyPublisher() }
                return self.search(query: query)
                    .catch { _ in Just([]) }
                    .eraseToAnyPublisher()
            }
            // switchToLatest вместо flatMap для автоотмены:
            // .map { [weak self] query in self?.search(query: query) ?? Just([]).eraseToAnyPublisher() }
            // .switchToLatest()
            .receive(on: DispatchQueue.main)
            .handleEvents(receiveOutput: { [weak self] _ in
                self?.isLoading = false
            })
            .assign(to: &$results)
    }
    
    private func search(query: String) -> AnyPublisher<[String], Error> {
        URLSession.shared.dataTaskPublisher(for: searchURL(query))
            .map(\.data)
            .decode(type: [String].self, decoder: JSONDecoder())
            .eraseToAnyPublisher()
    }
}
```

**Что такое `breakpoint` для отладки?**
```swift
publisher
    .breakpoint(
        receiveOutput: { value in
            value == 42  // остановить debugger если value == 42
        },
        receiveCompletion: { completion in
            if case .failure = completion { return true }  // stop on error
            return false
        }
    )
    .sink { ... }

// Простой breakpoint на любое значение:
publisher
    .breakpointOnError()  // только при ошибке

// Принудительный break всегда:
publisher
    .breakpoint(receiveOutput: { _ in true })
```

**Как `handleEvents` помогает отлаживать Combine?**
```swift
publisher
    .handleEvents(
        receiveSubscription: { subscription in
            print("📌 Subscribed: \(subscription)")
        },
        receiveOutput: { value in
            print("📦 Value: \(value)")
        },
        receiveCompletion: { completion in
            print("✅ Completed: \(completion)")
        },
        receiveCancel: {
            print("❌ Cancelled")
        },
        receiveRequest: { demand in
            print("📊 Demand: \(demand)")
        }
    )
    .sink { ... }

// Логирование pipeline:
extension Publisher {
    func log(_ label: String) -> AnyPublisher<Output, Failure> {
        handleEvents(
            receiveOutput: { print("[\(label)] output: \($0)") },
            receiveCompletion: { print("[\(label)] completion: \($0)") },
            receiveCancel: { print("[\(label)] cancelled") }
        ).eraseToAnyPublisher()
    }
}

// Использование:
searchPublisher
    .log("search")
    .map { ... }
    .log("after-map")
    .sink { ... }
```

---

## Дополнительные операторы

**Что такое `compactMap`?**
> `map` + фильтрация nil значений:
```swift
["1", "foo", "3", "bar"].publisher
    .compactMap { Int($0) }
    .sink { print($0) }
// 1, 3

// Очень полезно для Optional-returning transformations:
publisher
    .compactMap { $0 as? String }  // cast + filter nil
```

**Что такое `replaceError`?**
> Заменяет ошибку значением и завершает успешно:
```swift
URLSession.shared.dataTaskPublisher(for: url)
    .map(\.data)
    .replaceError(with: Data())  // при ошибке → пустой Data
    .sink { data in handle(data) }
// После replaceError — Failure == Never
```

**Что такое `replaceEmpty`?**
> Публикует значение если upstream завершился без значений:
```swift
Empty<Int, Never>()
    .replaceEmpty(with: 0)
    .sink { print($0) }  // 0

// Полезно для empty results:
searchResults
    .replaceEmpty(with: [defaultResult])
```

**Что такое `prefix` и `first`?**
```swift
// prefix — взять первые N:
(1...100).publisher
    .prefix(3)
    .sink { print($0) }  // 1, 2, 3

// first — взять первое удовлетворяющее условию:
(1...100).publisher
    .first { $0 > 50 }
    .sink { print($0) }  // 51

// prefix(while:):
(1...10).publisher
    .prefix(while: { $0 < 5 })
    .sink { print($0) }  // 1, 2, 3, 4
```

**Что такое `drop` и `dropFirst`?**
```swift
// dropFirst — пропустить первые N:
[1, 2, 3, 4, 5].publisher
    .dropFirst(2)
    .sink { print($0) }  // 3, 4, 5

// drop(while:):
[1, 2, 5, 6, 7].publisher
    .drop(while: { $0 < 5 })
    .sink { print($0) }  // 5, 6, 7 (пропускает пока условие true)

// drop(untilOutputFrom:) — ждёт сигнала:
let start = PassthroughSubject<Void, Never>()
dataPublisher
    .drop(untilOutputFrom: start)
    .sink { print($0) }
start.send()  // теперь начинаем получать данные
```

**Что такое `delay`?**
```swift
publisher
    .delay(for: .seconds(2), scheduler: DispatchQueue.main)
    .sink { print($0) }  // каждое значение задержано на 2 секунды

// Не задерживает completion — только значения
// Полезно для: retry с задержкой, UI feedback задержки
```

**Что такое `timeout`?**
```swift
publisher
    .timeout(.seconds(5), scheduler: DispatchQueue.main)
    .sink(
        receiveCompletion: { completion in
            // Если timeout — .finished (не error по умолчанию)
        },
        receiveValue: { ... }
    )

// С кастомной ошибкой:
enum FetchError: Error { case timeout }
publisher
    .timeout(.seconds(5), scheduler: DispatchQueue.main, customError: { FetchError.timeout })
```

**Что такое `measureInterval`?**
```swift
publisher
    .measureInterval(using: DispatchQueue.main)
    .sink { stride in
        print("Interval: \(stride.magnitude) seconds")
    }
// Измеряет время между последовательными значениями
```

---

## Практические задачи

### Задача 1: Форма логина с валидацией
```swift
// Задание: реализовать валидацию формы
// - username: минимум 3 символа
// - password: минимум 6 символов
// - кнопка Login активна только когда оба поля валидны
// - показывать ошибку с debounce 500ms

class LoginViewModel: ObservableObject {
    @Published var username = ""
    @Published var password = ""
    @Published var isLoginEnabled = false
    @Published var usernameError: String?
    @Published var passwordError: String?
    
    private var cancellables = Set<AnyCancellable>()
    
    init() {
        // Реализуй: isLoginEnabled через combineLatest
        // Реализуй: ошибки с debounce
        
        // Решение:
        let isUsernameValid = $username.map { $0.count >= 3 }
        let isPasswordValid = $password.map { $0.count >= 6 }
        
        isUsernameValid.combineLatest(isPasswordValid)
            .map { $0 && $1 }
            .assign(to: &$isLoginEnabled)
        
        $username
            .debounce(for: .milliseconds(500), scheduler: RunLoop.main)
            .map { $0.count >= 3 ? nil : "Минимум 3 символа" }
            .assign(to: &$usernameError)
        
        $password
            .debounce(for: .milliseconds(500), scheduler: RunLoop.main)
            .map { $0.count >= 6 ? nil : "Минимум 6 символов" }
            .assign(to: &$passwordError)
    }
}
```

### Задача 2: Поиск с кэшированием
```swift
// Задание: реализовать поиск
// - debounce 300ms
// - отменять предыдущий запрос
// - кэшировать результаты
// - не делать запрос если строка < 2 символов

class SearchViewModel: ObservableObject {
    @Published var query = ""
    @Published var results: [String] = []
    
    private var cache: [String: [String]] = [:]
    private var cancellables = Set<AnyCancellable>()
    
    init() {
        // Решение:
        $query
            .debounce(for: .milliseconds(300), scheduler: DispatchQueue.main)
            .removeDuplicates()
            .filter { $0.count >= 2 }
            .flatMap { [weak self] query -> AnyPublisher<[String], Never> in
                guard let self else { return Just([]).eraseToAnyPublisher() }
                
                // Проверяем кэш:
                if let cached = self.cache[query] {
                    return Just(cached).eraseToAnyPublisher()
                }
                
                return self.search(query)
                    .handleEvents(receiveOutput: { [weak self] results in
                        self?.cache[query] = results  // кэшируем
                    })
                    .catch { _ in Just([]) }
                    .eraseToAnyPublisher()
            }
            .receive(on: DispatchQueue.main)
            .assign(to: &$results)
    }
    
    private func search(_ query: String) -> AnyPublisher<[String], Error> {
        // HTTP запрос...
        Just([]).setFailureType(to: Error.self).eraseToAnyPublisher()
    }
}
```

### Задача 3: Polling с интервалом
```swift
// Задание: запрашивать статус заказа каждые 5 секунд
// пока статус не станет "delivered" или не пройдёт 60 секунд

func pollOrderStatus(orderId: String) -> AnyPublisher<OrderStatus, Error> {
    // Решение:
    Timer.publish(every: 5, on: .main, in: .common)
        .autoconnect()
        .flatMap { _ in fetchOrderStatus(orderId: orderId) }
        .prefix(while: { $0 != .delivered })
        .timeout(.seconds(60), scheduler: DispatchQueue.main, customError: { PollError.timeout })
        .eraseToAnyPublisher()
}
```

### Задача 4: Последовательная загрузка с retry
```swift
// Задание: загрузить список пользователей
// Для каждого пользователя загрузить аватар
// При ошибке аватара — retry 2 раза, потом placeholder
// Ограничить: не более 3 параллельных загрузок аватаров

func loadUsersWithAvatars() -> AnyPublisher<[UserWithAvatar], Error> {
    fetchUsers()
        .flatMap { users in
            users.publisher
                .flatMap(maxPublishers: .max(3)) { user in  // max 3 параллельных
                    self.loadAvatar(for: user)
                        .retry(2)
                        .catch { _ in Just(UIImage(named: "placeholder")!) }
                        .map { avatar in UserWithAvatar(user: user, avatar: avatar) }
                }
                .collect()
        }
        .eraseToAnyPublisher()
}
```

### Задача 5: Реализовать оператор `withLatestFrom`
```swift
// Задание: реализовать аналог RxSwift withLatestFrom
// При каждом событии source — брать последнее значение other

extension Publisher {
    func withLatestFrom<P: Publisher, R>(
        _ other: P,
        resultSelector: @escaping (Output, P.Output) -> R
    ) -> AnyPublisher<R, Failure> where P.Failure == Failure {
        
        // Решение через zip + share:
        let sharedOther = other.share()
        
        return self
            .flatMap { value in
                sharedOther
                    .first()
                    .map { otherValue in resultSelector(value, otherValue) }
            }
            .eraseToAnyPublisher()
    }
}

// Или через scan для хранения последнего значения:
extension Publisher {
    func withLatestFrom<P: Publisher>(_ other: P) -> AnyPublisher<P.Output, Failure>
    where P.Failure == Failure {
        combineLatest(other)
            .map(\.1)
            .eraseToAnyPublisher()
    }
}
```

### Задача 6: NotificationCenter → Publisher
```swift
// Задание: подписаться на keyboard notifications
// вычислять высоту клавиатуры
// обновлять constraint

struct KeyboardHeight {
    let height: CGFloat
    let animationDuration: TimeInterval
}

func keyboardHeightPublisher() -> AnyPublisher<KeyboardHeight, Never> {
    let willShow = NotificationCenter.default
        .publisher(for: UIResponder.keyboardWillShowNotification)
        .map { notification -> KeyboardHeight in
            let frame = notification.userInfo?[UIResponder.keyboardFrameEndUserInfoKey] as? CGRect
            let duration = notification.userInfo?[UIResponder.keyboardAnimationDurationUserInfoKey] as? TimeInterval
            return KeyboardHeight(height: frame?.height ?? 0, duration: duration ?? 0.25)
        }
    
    let willHide = NotificationCenter.default
        .publisher(for: UIResponder.keyboardWillHideNotification)
        .map { notification -> KeyboardHeight in
            let duration = notification.userInfo?[UIResponder.keyboardAnimationDurationUserInfoKey] as? TimeInterval
            return KeyboardHeight(height: 0, duration: duration ?? 0.25)
        }
    
    return willShow.merge(with: willHide).eraseToAnyPublisher()
}

// Использование в ViewController:
keyboardHeightPublisher()
    .receive(on: DispatchQueue.main)
    .sink { [weak self] keyboard in
        UIView.animate(withDuration: keyboard.animationDuration) {
            self?.bottomConstraint.constant = keyboard.height
            self?.view.layoutIfNeeded()
        }
    }
    .store(in: &cancellables)
```

### Задача 7: Combine Pipeline тест
```swift
// Задание: написать тесты для pipeline поиска
class SearchPipelineTests: XCTestCase {
    var cancellables = Set<AnyCancellable>()
    
    func test_debounce_ignoresRapidInput() {
        let subject = PassthroughSubject<String, Never>()
        var received: [String] = []
        let expectation = self.expectation(description: "debounced")
        
        subject
            .debounce(for: .milliseconds(100), scheduler: DispatchQueue.main)
            .sink {
                received.append($0)
                expectation.fulfill()
            }
            .store(in: &cancellables)
        
        subject.send("a")
        subject.send("ab")
        subject.send("abc")  // только это должно пройти
        
        waitForExpectations(timeout: 0.5)
        XCTAssertEqual(received, ["abc"])
    }
    
    func test_combineLatest_emitsOnEitherChange() {
        let name = CurrentValueSubject<String, Never>("Alice")
        let age = CurrentValueSubject<Int, Never>(25)
        var received: [(String, Int)] = []
        
        name.combineLatest(age)
            .sink { received.append($0) }
            .store(in: &cancellables)
        
        name.send("Bob")
        age.send(30)
        
        XCTAssertEqual(received.count, 3)  // initial + 2 changes
        XCTAssertEqual(received.last?.0, "Bob")
        XCTAssertEqual(received.last?.1, 30)
    }
    
    func test_retry_onFailure() {
        var attempts = 0
        let publisher = Deferred {
            Future<Int, Error> { promise in
                attempts += 1
                if attempts < 3 {
                    promise(.failure(NSError(domain: "", code: 0)))
                } else {
                    promise(.success(42))
                }
            }
        }
        
        let expectation = self.expectation(description: "succeeded")
        var result: Int?
        
        publisher
            .retry(3)
            .sink(
                receiveCompletion: { _ in },
                receiveValue: { value in
                    result = value
                    expectation.fulfill()
                }
            )
            .store(in: &cancellables)
        
        waitForExpectations(timeout: 1)
        XCTAssertEqual(result, 42)
        XCTAssertEqual(attempts, 3)
    }
}
```

### Задача 8: Custom Subject с историей
```swift
// Задание: реализовать ReplaySubject — хранит N последних значений
// и отдаёт их новым subscribers

final class ReplaySubject<Output, Failure: Error>: Subject {
    private var buffer: [Output] = []
    private let bufferSize: Int
    private var subscribers: [AnySubscriber<Output, Failure>] = []
    private var completion: Subscribers.Completion<Failure>?
    private let lock = NSLock()
    
    init(bufferSize: Int) {
        self.bufferSize = bufferSize
    }
    
    func send(_ value: Output) {
        lock.withLock {
            buffer.append(value)
            if buffer.count > bufferSize { buffer.removeFirst() }
            subscribers.forEach { _ = $0.receive(value) }
        }
    }
    
    func send(completion: Subscribers.Completion<Failure>) {
        lock.withLock {
            self.completion = completion
            subscribers.forEach { $0.receive(completion: completion) }
        }
    }
    
    func send(subscription: Subscription) { subscription.request(.unlimited) }
    
    func receive<S: Subscriber>(subscriber: S) where S.Input == Output, S.Failure == Failure {
        lock.withLock {
            let subscriber = AnySubscriber(subscriber)
            // Отправить буфер новому subscriber:
            buffer.forEach { _ = subscriber.receive($0) }
            if let completion = completion {
                subscriber.receive(completion: completion)
            } else {
                subscribers.append(subscriber)
            }
        }
    }
}

// Использование:
let replay = ReplaySubject<Int, Never>(bufferSize: 3)
replay.send(1); replay.send(2); replay.send(3); replay.send(4)

replay.sink { print($0) }  // 2, 3, 4 (последние 3)
```

---

## Шпаргалка: когда что использовать

```
Источник одного значения              → Just, Future
Источник без значений (только finish) → Empty
Никогда не завершается               → PassthroughSubject, Timer
Хранить текущее значение             → CurrentValueSubject, @Published

Трансформация:
  map          → изменить тип/значение
  flatMap      → создать Publisher для каждого значения (параллельно)
  switchToLatest → отменять предыдущий при новом
  compactMap   → map + filter nil
  scan         → аккумулятор (running value)

Фильтрация:
  filter         → пропустить по условию
  removeDuplicates → убрать последовательные дубли
  debounce       → после паузы
  throttle       → не чаще чем
  first/prefix   → взять первые N

Комбинирование:
  merge          → объединить (любой порядок)
  zip            → по парам (ждёт оба)
  combineLatest  → при любом изменении (последние значения)

Ошибки:
  catch          → заменить publisher при ошибке
  retry          → повторить N раз
  replaceError   → заменить ошибку значением
  tryMap         → map с возможностью throw

Потоки:
  receive(on:)   → где доставляются значения
  subscribe(on:) → где выполняется работа

Shared:
  share()        → один upstream для всех subscribers
  multicast()    → ручной контроль запуска shared publisher
```

---

*Конец документа · Combine Framework*
