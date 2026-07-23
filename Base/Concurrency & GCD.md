# Concurrency & GCD — Вопросы и ответы
> iOS-собеседование · Swift Concurrency · GCD · Actor · Sendable
> Уровни: Beginner · Middle · Senior · Staff

---

## async/await

### Beginner

**Что такое async/await в Swift?**
> Механизм написания асинхронного кода в синхронном стиле. `async` помечает функцию как асинхронную. `await` — точка приостановки, где поток освобождается для других задач. Введено в Swift 5.5 / iOS 15.

**Как объявить async функцию?**
> `func fetchData() async -> Data { ... }`. Или с ошибкой: `func fetchUser() async throws -> User`. Вызов: `let data = await fetchData()` внутри другой async функции или Task.

**Что такое `await`?**
> Ключевое слово, помечающее точку где текущая функция может **приостановиться** (но не заблокироваться). Поток освобождается и может выполнять другие задачи. Когда результат готов — выполнение возобновляется (возможно на другом потоке).

**Что такое suspension point?**
> Точка в коде где async функция может приостановить выполнение — каждый `await`. В момент приостановки поток освобождается. Swift runtime сохраняет continuation (снимок состояния) и возобновляет позже. Нет блокировки потока.

**Чем async/await лучше completion handlers?**
> Линейный код вместо "callback hell". Ошибки через throws вместо `(Result, Error?)`. Компилятор проверяет корректность. Автоматическая отмена через structured concurrency. Нет `[weak self]`. Легче читать, тестировать и поддерживать.

**Можно ли использовать await вне async контекста?**
> Нет. `await` только внутри `async` функции, Task, или `@MainActor` контекста. Для запуска из синхронного кода — `Task { await myFunc() }`. В тестах: `async throws` методы тестов поддерживаются XCTest.

**Что такое `async` вычисляемое свойство?**
> `var currentUser: User { get async { await fetchUser() } }`. Требует `await` при доступе: `let user = await viewModel.currentUser`. Только getter может быть async, не setter.

---

### Middle

**Что происходит когда функция подвешивается на `await`?**
> 1) Runtime сохраняет continuation (состояние функции). 2) Текущий поток освобождается в cooperative thread pool. 3) Другие задачи могут выполняться на этом потоке. 4) Когда awaited операция завершается — continuation ставится в очередь на executor. 5) Выполнение возобновляется (возможно на другом потоке).

**Что такое cooperative thread pool?**
> Пул потоков Swift Concurrency. Количество потоков ≈ числу CPU ядер (не создаёт тысячи потоков как GCD). Потоки не блокируются — при `await` поток берёт следующую готовую task. Избегает thread explosion и context switching overhead.

**Как async функция возвращает значение?**
> Как обычная функция: `return value`. Caller получает через `let result = await asyncFunc()`. Под капотом — через continuation mechanism. При ошибке — `throw error`, caller обрабатывает в `do-catch`.

**Что такое `async throws`?**
> `func fetch() async throws -> Data`. Функция и асинхронна и может бросить ошибку. Вызов: `let data = try await fetch()`. Оба модификатора независимы — порядок не важен. Ошибка пропагируется через structured concurrency.

**Как вызвать async функцию из синхронного контекста?**
> `Task { await myAsyncFunc() }` — создаёт unstructured task. `Task.detached { await myAsyncFunc() }` — без inheritance context. В AppDelegate/SceneDelegate: `Task { @MainActor in await setup() }`. Нельзя напрямую await в синхронной функции.

**Что такое structured concurrency?**
> Иерархия задач: дочерние tasks не могут пережить родительскую. Отмена родителя → автоматическая отмена всех потомков. Ошибка всплывает вверх по иерархии. `async let` и `TaskGroup` — structured. `Task {}` — unstructured (не привязан к родителю).

**Как `async let` отличается от обычного `let`?**
> `async let` запускает задачу **немедленно и параллельно**: `async let user = fetchUser(); async let posts = fetchPosts()`. `let` — последовательно. Оба результата ожидаются через `let (u, p) = await (user, posts)`. Дочерние tasks структурированы.

**Что такое child task?**
> Task созданная внутри другой task через `async let` или `TaskGroup`. Child task: наследует приоритет и task-local values родителя, не может пережить родителя, ошибка передаётся родителю. При отмене родителя — отменяется.

**Как ошибки распространяются в structured concurrency?**
> В `async let`: первая ошибка отменяет другие дочерние tasks и propagates к родителю. В `TaskGroup`: первая ошибка из `group.next()` отменяет группу. Все дочерние tasks завершаются (с cancellation) перед propagation ошибки.

**Что такое `withCheckedContinuation`?**
> Мост между callback API и async/await: `let value = await withCheckedContinuation { cont in oldAPI.fetch { result in cont.resume(returning: result) } }`. "Checked" — проверяет что resume вызван ровно один раз (иначе warning/crash в debug).

**Как отличается `async let` от `Task {}`?**
> `async let` — structured: child task, отменяется вместе с родителем, ошибка всплывает. `Task {}` — unstructured: не привязан к parent scope, живёт независимо, ошибки нужно обрабатывать явно. `async let` предпочтительнее для parallel work.

**Что такое `withTaskCancellationHandler`?**
> `withTaskCancellationHandler(operation:onCancel:)` — выполняет operation, при отмене Task вызывает onCancel немедленно (даже если operation приостановлена на await). Нужно для cleanup при cancellation: отменить network request, закрыть файл.

---

### Senior

**Как Swift runtime управляет cooperative scheduling?**
> Swift Concurrency использует custom executor на основе libdispatch. Cooperative: задачи добровольно уступают поток на каждом `await`. Нет preemption. Количество потоков в пуле ≤ CPU cores. Задачи — легковесные (stackless coroutines), не потоки.

**Что такое executor в Swift Concurrency?**
> Protocol `Executor` — планировщик выполнения задач. `SerialExecutor` — serial (как serial queue). `MainActor` использует Main thread executor. Можно создать custom executor для actor. Default: global cooperative pool.

**Как async/await влияет на стек вызовов?**
> Async функции — stackless coroutines. При приостановке стек не сохраняется — только continuation (heap-allocated). Stacktrace в debugger выглядит иначе: не показывает полный call stack через awaits. Инструменты: `#function`, os_signpost для tracing.

**Что такое task-local storage?**
> `@TaskLocal static var userID: String = "anonymous"`. Значение доступно всем дочерним tasks автоматически. Установка: `UserContext.$userID.withValue("123") { await work() }`. Нет явной передачи через аргументы. Используется для logging, tracing.

**Как компилятор трансформирует async функцию?**
> В state machine: каждый `await` — transition между states. Локальные переменные — в heap-allocated continuation context. `resume` перемещает state machine к следующему state. Аналогично как async/await в C# или Rust coroutines.

**Что такое continuation leaking?**
> `withCheckedContinuation` где `resume` никогда не вызывается — task зависает навсегда. Или resume вызывается дважды — undefined behavior. "Checked" версия детектирует это в debug (warning при deinit без resume, crash при двойном resume).

**Как `withUnsafeContinuation` отличается от `withCheckedContinuation`?**
> `withUnsafeContinuation` — нет runtime проверок (быстрее, меньше overhead). Двойной resume = UB (crash или corruption). Нет warning при leaking. Использовать только когда уверен в корректности и нужна максимальная производительность.

**Как async функции взаимодействуют с ARC?**
> Continuation (heap-allocated) удерживает все захваченные объекты. При await — объекты живут пока task не завершится или не дойдёт до next await. Нет retain cycle через `self` в отличие от closures (нет strong capture). `weak` не нужен в большинстве случаев.

**Что такое priority propagation в Tasks?**
> Дочерняя task наследует priority родителя. При ожидании высокоприоритетной task на низкоприоритетной — priority инвертируется (повышается). `Task(priority: .high) { await lowPriorityTask.value }` → lowPriorityTask получает .high priority.

**Как async/await взаимодействует с ObjC runtime?**
> ObjC не понимает Swift async. Для bridging: `withCheckedContinuation` оборачивает ObjC completion handlers. `@objc` функция не может быть `async`. Swift async функции не экспортируются в ObjC напрямую. Можно создать sync ObjC wrapper.

**Что такое `@_silgen_name` и как связан с async?**
> Внутренний атрибут для именования символов в SIL. Async функции имеют специальные SIL символы. Не для публичного использования. Используется в stdlib для низкоуровневого bridging между C runtime и Swift async.

---

### Staff

**Как Swift Concurrency runtime реализован?**
> Поверх libdispatch (GCD). `swift_task_create` создаёт Task. Cooperative scheduler — DispatchQueue под капотом. `swift_task_switch` — переключение между executors. Continuation — heap-allocated структура со state machine и resume function.

**Что такое Swift Job scheduler?**
> Внутренний компонент Swift Concurrency runtime. Управляет очередью Jobs (единиц работы). Job = одна "часть" async функции между awaits. Scheduler решает какой Job запустить следующим на основе priority и доступных потоков.

**Как custom executor влияет на поведение Actor?**
> Actor с `nonisolated(unsafe) var unownedExecutor: UnownedSerialExecutor` — переопределяет где выполняются его методы. `MainActor` использует Main thread executor. Custom executor: `actor DatabaseActor { nonisolated var unownedExecutor: UnownedSerialExecutor { dbQueue.asUnownedSerialExecutor() } }`.

**Что такое SerialExecutor protocol?**
> `protocol SerialExecutor: Executor`. Гарантирует serial выполнение Jobs — не параллельно. Actor executor должен быть SerialExecutor. `enqueue(_ job: consuming ExecutorJob)` — добавить Job в очередь. `asUnownedSerialExecutor()` — для использования в Actor.

**Как `Task` связан с POSIX threads?**
> Task — не POSIX thread. Task — lightweight coroutine на heap. Многие Tasks выполняются на одном POSIX thread (cooperative). Swift Concurrency runtime поддерживает пул POSIX threads (через libdispatch). Tasks переключаются между threads без POSIX thread context switch.

---

## Task & TaskGroup

### Beginner

**Что такое `Task` в Swift Concurrency?**
> Единица асинхронной работы. Аналог thread, но легковесный. `Task { await doWork() }`. Имеет: priority, cancellation, task-local values. Structured (child) или unstructured (независимый). Можно ожидать результат через `.value`.

**Как создать Task?**
> `Task { await doWork() }` — unstructured, наследует actor context и priority. `Task(priority: .background) { await heavyWork() }` — с приоритетом. `Task.detached { await doWork() }` — без inheritance контекста. `async let` — structured child task.

**Что такое `Task.sleep`?**
> `try await Task.sleep(for: .seconds(2))` (Swift 5.7+) или `Task.sleep(nanoseconds: 2_000_000_000)`. Не блокирует поток — приостанавливает task. При cancellation бросает `CancellationError`. Идеально для delays, retries, polling.

**Как Task возвращает результат?**
> `let task = Task { return 42 }; let result = await task.value`. `.value` ожидает завершения и возвращает результат. `.result` — возвращает `Result<T, Error>`. Для throwing task: `try await task.value`.

**Чем `Task {}` отличается от Thread?**
> Task — легковесная coroutine, не OS thread. Тысячи Tasks на немногих потоках. Task не блокирует поток при `await`. Thread — OS ресурс, дорогой (512KB stack). Task создаётся за микросекунды vs Thread за миллисекунды.

---

### Middle

**Что такое structured vs unstructured task?**
> Structured: `async let`, `TaskGroup` — child task, не переживает родителя, наследует context. Unstructured: `Task {}`, `Task.detached` — независимый, переживает scope создания, ручное управление cancellation. Structured предпочтительнее.

**Чем `Task.detached` отличается от `Task {}`?**
> `Task {}` — наследует actor isolation (`@MainActor` если создан в нём), priority, task-locals. `Task.detached` — нет inheritance, запускается без actor context, нет task-local inheritance. `detached` нужен когда требуется полная независимость.

**Что такое `TaskGroup`?**
> Динамическая structured concurrency: `withTaskGroup(of: Int.self) { group in group.addTask { await work1() }; group.addTask { await work2() } }`. Все child tasks завершаются до выхода из замыкания. Итерировать результаты: `for await result in group`.

**Как добавить задачи в `TaskGroup`?**
> `group.addTask { await doWork() }` — добавляет child task. `group.addTaskUnlessCancelled { await doWork() }` — только если группа не отменена. Задачи выполняются немедленно параллельно. Результаты получаем через `group.next()` или итерацию.

**Что такое `withTaskGroup` vs `withThrowingTaskGroup`?**
> `withTaskGroup` — child tasks не бросают ошибки (или ошибки обрабатываются внутри). `withThrowingTaskGroup` — child tasks могут throws, первая ошибка отменяет группу и propagates. Выбор зависит от нужды в error propagation.

**Как отменить задачу в группе?**
> `group.cancelAll()` — отменяет все child tasks в группе. Отмена cooperative: tasks должны проверять `Task.isCancelled` или `try Task.checkCancellation()`. При выходе из `withTaskGroup` с ошибкой — группа отменяется автоматически.

**Что такое task cancellation?**
> Cooperative механизм: не принудительный kill. Task получает сигнал отмены. Task сам должен проверить и отреагировать. `Task.isCancelled` — проверить. `Task.checkCancellation()` — проверить и throw `CancellationError`. `withTaskCancellationHandler` — handler при отмене.

**Как проверить отмену задачи (`Task.isCancelled`)?**
> `Task.isCancelled` — `Bool`, static property, проверяет текущую task. Для periodic check в loops: `while !Task.isCancelled { await process() }`. `Task.checkCancellation()` — то же но бросает `CancellationError` при отмене.

**Что такое `Task.checkCancellation()`?**
> `try Task.checkCancellation()` — если task отменена, бросает `CancellationError`. Удобно для точек в коде где нужно немедленно остановиться: `for item in items { try Task.checkCancellation(); await process(item) }`.

**Как Task наследует priority и task-local values?**
> `Task {}` (не detached): наследует priority текущей task, все `@TaskLocal` значения. `Task.detached {}`: priority `.medium` по умолчанию, пустые task-locals. `async let` child: всегда наследует оба от родителя.

---

### Senior

**Как TaskGroup обрабатывает ошибки из дочерних задач?**
> В `withThrowingTaskGroup`: если child task бросает — ошибка попадает при `group.next()` или итерации. Первая необработанная ошибка отменяет группу (`group.cancelAll()`), потом propagates из `withThrowingTaskGroup`. Можно перехватить через try-catch внутри.

**Что такое structured concurrency дерево задач?**
> Иерархия: root task → child tasks (async let, TaskGroup) → их дочерние tasks. Свойства: child не переживает parent, отмена propagates вниз, ошибки propagates вверх, parent ждёт всех детей. Похоже на иерархию процессов в OS.

**Как реализовать concurrent map через TaskGroup?**
```swift
func concurrentMap<T, U>(_ array: [T], transform: @escaping (T) async throws -> U) async throws -> [U] {
    try await withThrowingTaskGroup(of: (Int, U).self) { group in
        for (i, element) in array.enumerated() {
            group.addTask { (i, try await transform(element)) }
        }
        var results = [(Int, U)]()
        for try await result in group { results.append(result) }
        return results.sorted { $0.0 < $1.0 }.map(\.1)
    }
}
```

**Как ограничить параллелизм в TaskGroup?**
> `group.addTask` запускает все задачи сразу. Для лимита: semaphore или batching. Паттерн: `let limit = 4; var active = 0; for item in items { if active >= limit { _ = try await group.next(); active -= 1 }; group.addTask { await process(item) }; active += 1 }`.

**Что такое `DiscardingTaskGroup`?**
> Swift 5.9+: `withDiscardingTaskGroup { group in ... }`. Как TaskGroup но результаты child tasks игнорируются (Void). Более эффективен — не накапливает результаты в памяти. Идеален для fire-and-forget parallel work.

**Как Task lifecycle влияет на ARC?**
> Task удерживает (strong) все захваченные объекты пока не завершится. Долгоживущая task = удерживает view controller, view model. `Task.detached` без `[weak self]` — классическая проблема. Cancellation через `task.cancel()` + check = корректный cleanup.

**Что происходит когда parent task отменяется?**
> Сигнал отмены propagates ко всем child tasks (async let, TaskGroup). Child tasks получают `isCancelled = true`. При следующем `await` или `checkCancellation()` — бросают `CancellationError`. Parent ждёт пока все child tasks не завершатся (cancelled или normally).

**Как реализовать timeout для Task?**
```swift
func withTimeout<T>(seconds: Double, operation: @escaping () async throws -> T) async throws -> T {
    try await withThrowingTaskGroup(of: T.self) { group in
        group.addTask { try await operation() }
        group.addTask {
            try await Task.sleep(for: .seconds(seconds))
            throw TimeoutError()
        }
        let result = try await group.next()!
        group.cancelAll()
        return result
    }
}
```

**Как Task.priority влияет на scheduling?**
> `Task(priority: .high)` — scheduler предпочитает высокоприоритетные task. `.userInitiated` > `.medium` > `.low` > `.background`. Priority не абсолютна — cooperative scheduling. High priority task может ждать если все потоки заняты.

**Что такое priority inversion в Task?**
> High priority task ждёт результата low priority task. Swift Concurrency автоматически повышает priority low-priority task (priority donation). Предотвращает классический priority inversion. GCD требовал ручного управления через QoS.

**Как правильно хранить Task для отмены?**
```swift
class ViewModel {
    private var loadTask: Task<Void, Error>?
    
    func load() {
        loadTask?.cancel()  // отменяем предыдущую
        loadTask = Task {
            try await fetchData()
        }
    }
    
    func cancel() { loadTask?.cancel() }
    deinit { loadTask?.cancel() }
}
```

---

## Actor

### Beginner

**Что такое Actor в Swift?**
> Reference type защищающий mutable state от concurrent access. Все обращения к actor-isolated state — сериализованы (один за раз). Нет data races на состояние актора. Синтаксис как class но с isolation гарантиями.

**Как объявить Actor?**
> `actor BankAccount { var balance: Double = 0; func deposit(_ amount: Double) { balance += amount } }`. Вызов снаружи: `await account.deposit(100)`. Методы внутри актора — синхронные (нет await нужен).

**Что такое actor isolation?**
> Гарантия что только одна задача одновременно выполняет код на actor. Все свойства и методы actor-isolated — доступны без await изнутри актора, с `await` снаружи. Компилятор проверяет isolation статически.

**Как вызвать метод актора?**
> Снаружи: `await actor.method()` — обязателен `await`. Внутри актора: `self.method()` — синхронно, без await. Из `nonisolated` метода актора: нужен `await` (выход из isolation).

**Чем actor отличается от класса с lock?**
> Actor: compile-time проверка, нет deadlock (нет явных lock), cooperative (не блокирует поток при await). Class + lock: runtime проверки, risk of deadlock, блокирует поток. Actor значительно безопаснее и удобнее.

---

### Middle

**Что такое `@MainActor`?**
> Global actor привязанный к main thread. Аннотация: `@MainActor class ViewModel { }` — все методы выполняются на главном потоке. `@MainActor func updateUI() { }`. Заменяет `DispatchQueue.main.async { }`. Компилятор проверяет.

**Как `@MainActor` связан с main thread?**
> `@MainActor` использует `MainActor.shared` — global actor чей executor запускает Jobs на main thread (через `RunLoop.main` / `DispatchQueue.main`). Гарантирует UI обновления на правильном потоке без явного `DispatchQueue.main.async`.

**Что такое global actor?**
> Singleton actor доступный глобально: `@globalActor actor MyActor { static let shared = MyActor() }`. Используется как аннотация: `@MyActor class Service { }`. Все `@MyActor` code — serial на одном executor. `MainActor` — самый известный.

**Как объявить custom global actor?**
```swift
@globalActor
actor DatabaseActor {
    static let shared = DatabaseActor()
    
    // Опционально: custom executor
    nonisolated var unownedExecutor: UnownedSerialExecutor {
        dbQueue.asUnownedSerialExecutor()
    }
}

@DatabaseActor class DatabaseService { }
```

**Что такое reentrancy в акторах?**
> Actor reentrancy: пока actor ждёт на `await`, другой вызов может войти в actor. State может измениться между `await` точками! `let count = self.count; await slowOp(); // count могло измениться!`. Нужно проверять state после каждого await.

**Как Actor предотвращает data races?**
> Все доступы к actor-isolated state сериализованы через actor's executor (serial). Нет двух concurrent доступов одновременно. Компилятор запрещает доступ к actor state без `await` снаружи. `Sendable` проверяет безопасность передаваемых данных.

**Что такое `nonisolated`?**
> `nonisolated func description() -> String { return "..." }` — метод не требует actor isolation, может вызываться синхронно снаружи. Используется для: computed properties (Identifiable.id), методов без доступа к state, protocol conformances (CustomStringConvertible).

**Когда использовать `nonisolated`?**
> Когда метод не обращается к actor-isolated state: `nonisolated var id: UUID { _id }` где `_id` — let. Для protocol conformances требующих синхронный доступ. Для expensive operations не требующих state: pure computations, logging.

**Как Actor взаимодействует с Sendable?**
> Actor passes через concurrency boundaries должны быть `Sendable`. Actor reference type — автоматически `Sendable` (isolation гарантирует thread safety). Параметры и возвращаемые значения actor методов — должны быть `Sendable`.

**Как тестировать код с акторами?**
```swift
actor Counter {
    private(set) var value = 0
    func increment() { value += 1 }
}

// XCTest поддерживает async throws
func test_counterIncrement() async {
    let counter = Counter()
    await counter.increment()
    let value = await counter.value
    XCTAssertEqual(value, 1)
}
```

---

### Senior

**Как Actor реализован под капотом?**
> Actor — class с `SerialExecutor`. Все вызовы изолированных методов — через executor queue. Внутри: `__executor_impl` хранит serial executor. При вызове снаружи: dispatch job на executor. При вызове изнутри — проверка что уже на executor, прямой вызов.

**Что такое actor executor?**
> `SerialExecutor` — компонент определяющий где и когда выполняются actor methods. Default: глобальный cooperative pool с serial guarantee. `MainActor.shared.executor` — main thread. Custom executor позволяет привязать actor к конкретной queue.

**Как custom executor для Actor работает?**
```swift
actor FileActor {
    private let queue = DispatchSerialQueue(label: "file-actor")
    
    nonisolated var unownedExecutor: UnownedSerialExecutor {
        queue.asUnownedSerialExecutor()
    }
    
    func readFile() -> Data { /* выполнится на queue */ }
}
```

**Что такое actor hopping и как его избежать?**
> Actor hopping: переключение между акторами через `await`. `await actorA.method()` внутри `actorB` → hop A→B→A. Каждый hop = suspend/resume overhead. Минимизировать: batch операции, передавать данные а не вызывать методы, `nonisolated` для чистых функций.

**Как Actor взаимодействует с существующим GCD кодом?**
> Нельзя использовать `DispatchQueue.sync` внутри actor (deadlock риск). `DispatchQueue.async` — ок но теряешь await. Паттерн: `withCheckedContinuation` для GCD callbacks. Постепенная миграция: оберни GCD в async функцию.

**Что такое actor interleaving и почему это важно?**
> Между `await` точками внутри actor другой caller может войти и изменить state. Пример: `let x = state; await asyncOp(); use(x) // x устарело!`. Важно: не делай assumptions о state после await. Используй локальные копии или атомарные операции.

**Как реализовать actor с кастомным SerialExecutor?**
```swift
final class MyExecutor: SerialExecutor {
    private let queue = DispatchQueue(label: "my-executor")
    
    func enqueue(_ job: consuming ExecutorJob) {
        let unownedJob = UnownedJob(job)
        queue.async { unownedJob.runSynchronously(on: self.asUnownedSerialExecutor()) }
    }
    
    func asUnownedSerialExecutor() -> UnownedSerialExecutor {
        UnownedSerialExecutor(ordinary: self)
    }
}
```

**Что такое isolated параметр?**
> Swift 5.7+: `func process(isolation: isolated any Actor = #isolation)`. Параметр указывает на какой actor выполняется функция. Компилятор проверяет isolation. `#isolation` — захватывает текущий actor context автоматически.

**Как `@globalActor` влияет на static members?**
> `@MyActor class Service { static var shared = Service() }`. `shared` — actor-isolated. Доступ снаружи: `await Service.shared`. Static members `@globalActor` класса — все изолированы на этом global actor. `nonisolated static` — исключение.

**Что такое assumeIsolated?**
> `MainActor.assumeIsolated { updateUI() }` — говорит компилятору "мы уже на этом actor, проверять не нужно". Runtime crash если не так. Используется в legacy code или callbacks где компилятор не может доказать isolation статически.

**Как реализовать actor с кэшем?**
```swift
actor Cache<Key: Hashable, Value> {
    private var storage: [Key: Value] = [:]
    
    func get(_ key: Key) -> Value? { storage[key] }
    
    func set(_ key: Key, value: Value) { storage[key] = value }
    
    func getOrCompute(_ key: Key, compute: () async -> Value) async -> Value {
        if let cached = storage[key] { return cached }
        let value = await compute()  // ← reentrancy: другой caller мог тоже войти!
        storage[key] = value         // безопасно — перезапись идемпотентна
        return value
    }
}
```

---

## Sendable

### Beginner

**Что такое `Sendable`?**
> Протокол-маркер: тип безопасен для передачи через concurrency boundaries (между акторами, в Tasks). `struct Config: Sendable { let value: Int }`. Компилятор проверяет что `Sendable` типы не создадут data races.

**Зачем нужен `Sendable`?**
> Гарантия thread safety при передаче данных между concurrent контекстами. `Task { await actor.method(data) }` — data должна быть `Sendable`. Предотвращает передачу mutable shared state между задачами. Compile-time safety вместо runtime crash.

**Какие типы автоматически `Sendable`?**
> Value types (struct, enum) если все поля `Sendable`: `Int`, `String`, `Bool`, `Double`, `Array<Sendable>`. `@Sendable` функции. Actors. Immutable classes (`final class` с только `let` свойствами). Tuple, Optional если элементы Sendable.

---

### Middle

**Что такое `@Sendable` closure?**
> Замыкание безопасное для передачи через concurrency boundaries. `Task { @Sendable in ... }`. Не может захватывать mutable non-Sendable state. `Task.init` принимает `@Sendable () async throws -> T`. Компилятор проверяет захватываемые переменные.

**Что такое `@unchecked Sendable`?**
> `final class ThreadSafeCache: @unchecked Sendable { private let lock = NSLock(); private var storage: [String: Any] = [:] }`. Отключает compiler checks. Разработчик берёт ответственность за thread safety. Использовать только когда знаешь что делаешь.

**Как Sendable связан с data races?**
> Sendable = тип не создаст data race при concurrent access. Value types — копируются (нет shared state). Actors — сериализуют доступ. Классы без Sendable — могут иметь shared mutable state → data race. Swift 6 strict: non-Sendable через boundary = error.

**Почему class по умолчанию не Sendable?**
> Class — reference type: несколько concurrent задач могут получить одну ссылку и конкурентно мутировать state → data race. Поэтому class требует явного подтверждения thread safety через `Sendable` conformance.

**Как сделать class Sendable?**
> Вариант 1: `final class` с только `let` properties (immutable). Вариант 2: `@unchecked Sendable` + внутренняя синхронизация (lock, serial queue). Вариант 3: превратить в `actor`. Вариант 4: превратить в `struct`.

**Что такое region-based isolation?**
> Swift 6: компилятор отслеживает "регионы" где живут значения. Non-Sendable тип может передаваться через boundary если компилятор доказывает что source region больше не имеет доступа (transfer). Менее строго чем полный запрет.

**Как Sendable проверяется компилятором?**
> В Swift 5 — warnings. В Swift 6 — errors (strict concurrency). Проверки: передача non-Sendable через actor boundary, захват non-Sendable в `@Sendable` closure, возврат non-Sendable из actor. `@preconcurrency` — постепенная миграция.

**Как Sendable взаимодействует с generics?**
> `func process<T: Sendable>(_ value: T) async { await actor.store(value) }`. Constraint `T: Sendable` гарантирует безопасность. Conditional Sendable: `extension Array: Sendable where Element: Sendable { }` — уже реализовано в stdlib.

**Когда нужно явно указать `@unchecked Sendable`?**
> Когда тип thread-safe через механизмы невидимые компилятору: внешние C locks, OS-level sync, custom atomic operations. Например: `URLSession`, `NSCache` — `@unchecked Sendable` в Foundation. Предпочитай actor или struct где возможно.

**Как Sendable влияет на NSObject subclasses?**
> NSObject subclasses — не автоматически Sendable (mutable reference type). Для `@unchecked Sendable` — нужно доказать thread safety. Многие UIKit классы — `@MainActor` вместо Sendable (доступ только с main thread).

---

### Senior

**Как строгая проверка Sendable влияет на migration существующего кода?**
> Swift 6 breaking: весь код должен быть thread safe. Паттерны: 1) `@preconcurrency import` для legacy modules. 2) `@MainActor` для UI кода. 3) Structs вместо classes. 4) Actors для shared mutable state. 5) `@unchecked Sendable` как временная заглушка.

**Что такое transferring modifier (SE-0430)?**
> Swift 6.0: `func process(_ value: transferring MyObject)`. Параметр "передаётся" в функцию — caller больше не использует его. Не требует Sendable. Компилятор доказывает что нет aliasing. Позволяет передавать non-Sendable types безопасно.

**Как region isolation работает в Swift 6?**
> Компилятор отслеживает "регионы изоляции" для каждого значения. При передаче через concurrency boundary — проверяет что значение не используется в source region после передачи. Менее строго чем Sendable: позволяет передавать non-Sendable если нет aliasing.

**Что такое sending parameter?**
> `func store(_ value: sending MyObject)` — значение "отправляется" в функцию, ownership передаётся. Caller не может использовать значение после. Позволяет безопасно передавать non-Sendable через actor boundaries без полного Sendable требования.

**Как Sendable взаимодействует с actor isolation?**
> Actor-isolated state не нужно быть Sendable — доступ сериализован. Но данные передаваемые В actor или ИЗ actor через async методы — должны быть Sendable. Actor сам по себе Sendable (ссылка на actor безопасна для sharing).

---

## AsyncSequence & AsyncStream

### Beginner

**Что такое `AsyncSequence`?**
> Протокол для последовательностей значений приходящих асинхронно во времени. `protocol AsyncSequence { associatedtype Element; func makeAsyncIterator() -> Iterator }`. Примеры: `URLSession.bytes`, `NotificationCenter.notifications`, `FileHandle.bytes`.

**Как итерировать AsyncSequence?**
> `for await value in asyncSequence { process(value) }`. При отмене Task — цикл завершается. `for try await` — для throwing sequences. Внутри итератора: компилятор вызывает `makeAsyncIterator()` и затем `next()` повторно.

**Чем AsyncSequence отличается от обычной Sequence?**
> Sequence: значения доступны синхронно, `next()` возвращает сразу. AsyncSequence: значения приходят асинхронно, `next() async` — приостанавливает пока следующее значение не будет готово. Не нужно polling — push модель.

---

### Middle

**Как реализовать собственный AsyncSequence?**
```swift
struct CountdownSequence: AsyncSequence {
    typealias Element = Int
    let from: Int
    
    struct AsyncIterator: AsyncIteratorProtocol {
        var current: Int
        mutating func next() async -> Int? {
            guard current > 0 else { return nil }
            try? await Task.sleep(for: .seconds(1))
            defer { current -= 1 }
            return current
        }
    }
    
    func makeAsyncIterator() -> AsyncIterator { AsyncIterator(current: from) }
}
```

**Что такое `AsyncStream`?**
> Удобный способ создать AsyncSequence без реализации протокола вручную. `AsyncStream<Int> { continuation in continuation.yield(1); continuation.yield(2); continuation.finish() }`. Значения буферизуются. Есть throwing вариант: `AsyncThrowingStream`.

**Как создать AsyncStream из completion handler API?**
```swift
func locationStream() -> AsyncStream<CLLocation> {
    AsyncStream { continuation in
        let manager = CLLocationManager()
        let delegate = LocationDelegate {
            continuation.yield($0)
        }
        manager.delegate = delegate
        manager.startUpdatingLocation()
        continuation.onTermination = { _ in manager.stopUpdatingLocation() }
    }
}
```

**Что такое `AsyncThrowingStream`?**
> Как `AsyncStream` но может бросить ошибку: `continuation.finish(throwing: error)`. Итерация: `for try await value in stream`. Финишировать с ошибкой или без: `continuation.finish()` (успех) или `continuation.finish(throwing: error)`.

**Как отменить AsyncSequence итерацию?**
> Отмена Task содержащего `for await` — автоматически завершает итерацию. `for await value in sequence.prefix(10)` — остановиться после 10. `AsyncStream.onTermination` вызывается при cancellation. Внутри итератора: check `Task.isCancelled`.

**Что такое `AsyncStream.Continuation`?**
> Объект для отправки значений в AsyncStream извне. `.yield(value)` — отправить значение. `.finish()` — завершить последовательность. `.finish(throwing:)` — завершить с ошибкой. `.onTermination` — handler когда stream завершается (cancellation или finish).

**Как `AsyncStream` помогает оборачивать delegate API?**
> Delegate callbacks → yields в continuation. Пример: `NotificationCenter`, `CLLocationManager`, `AVAudioEngine`. Создаёшь AsyncStream, в нём настраиваешь delegate, в callbacks — `continuation.yield(value)`. Чисто, без callback hell.

**Как работает `for await` loop под капотом?**
> Компилятор трансформирует в: `var iter = sequence.makeAsyncIterator(); while let value = try await iter.next() { body(value) }`. Каждый вызов `iter.next()` — suspension point. Cancellation проверяется при каждом вызове `next()`.

**Что такое `makeAsyncIterator`?**
> Метод `AsyncSequence` создающий iterator: `func makeAsyncIterator() -> AsyncIterator`. Вызывается один раз в начале `for await`. Итератор — `AsyncIteratorProtocol` с `mutating func next() async throws -> Element?`. `nil` = конец последовательности.

**Как объединить несколько AsyncSequence?**
```swift
// Последовательно:
for await value in firstSequence.chain(secondSequence) { }

// Одновременно (нет встроенного merge, нужна реализация):
let combined = AsyncStream<Int> { cont in
    Task { for await v in seq1 { cont.yield(v) } }
    Task { for await v in seq2 { cont.yield(v) } }
}
```

**Что такое `AsyncChannel` из swift-async-algorithms?**
> Из пакета `swift-async-algorithms`: `AsyncChannel<Element>` — multi-producer, single-consumer канал. `send(_:)` — с backpressure (ждёт пока consumer не примет). `finish()` — завершить. Безопаснее чем AsyncStream для concurrent sends.

---

### Senior

**Как реализовать backpressure для AsyncStream?**
> `AsyncStream(bufferingPolicy:)`: `.unbounded` (по умолчанию, нет backpressure), `.bufferingNewest(N)` — хранить N последних, `.bufferingOldest(N)` — хранить N первых. Настоящий backpressure: `AsyncChannel` из swift-async-algorithms с `await send()`.

**Как AsyncStream взаимодействует со structured concurrency?**
> AsyncStream живёт независимо от создавшей Task (unstructured). `onTermination` вызывается при отмене итерирующей Task. Паттерн: создавать AsyncStream внутри actor, yield в actor methods. Cancellation через Task propagates к for-await.

**Что такое buffering policy для AsyncStream?**
> Контролирует буфер при backpressure. `.unbounded` — неограниченный буфер (риск OOM). `.bufferingNewest(4)` — при переполнении удаляет старые. `.bufferingOldest(4)` — при переполнении отбрасывает новые. `.dropNewest` / `.dropOldest` — явно.

**Как реализовать Combine Publisher как AsyncSequence?**
```swift
extension Publisher where Failure == Never {
    var values: AsyncPublisher<Self> { AsyncPublisher(self) }  // встроено в Combine!
}

// Использование:
let cancellable = Just(42)
for await value in Just(42).values {
    print(value)
}
```

**Как тестировать AsyncSequence?**
```swift
func test_countdownSequence() async {
    var results: [Int] = []
    for await value in CountdownSequence(from: 3) {
        results.append(value)
    }
    XCTAssertEqual(results, [3, 2, 1])
}

// Для AsyncStream с Task:
func test_asyncStream() async {
    let stream = makeTestStream(values: [1, 2, 3])
    var received: [Int] = []
    for await value in stream.prefix(3) {
        received.append(value)
    }
    XCTAssertEqual(received, [1, 2, 3])
}
```

---

## GCD — DispatchQueue

### Beginner

**Что такое DispatchQueue?**
> Объект управляющий очередью задач для выполнения. Задачи выполняются в порядке FIFO. Два типа: serial (одна за одной) и concurrent (параллельно). Основа GCD (Grand Central Dispatch) — C-level threading API от Apple.

**Что такое main queue?**
> `DispatchQueue.main` — serial очередь выполняющаяся на main thread. Все UI обновления — только здесь. `DispatchQueue.main.async { label.text = "done" }`. Синхронный вызов с main queue (на main thread) → deadlock.

**Что такое global queue?**
> `DispatchQueue.global(qos: .userInitiated)` — concurrent очередь системного пула. Разные QoS уровни. Не создаёт новый поток — использует thread pool. `DispatchQueue.global()` = `.default` QoS.

**Чем serial queue отличается от concurrent queue?**
> Serial: задачи выполняются одна за другой, порядок гарантирован, один поток в каждый момент. Concurrent: несколько задач одновременно, порядок не гарантирован, несколько потоков. Main queue — serial, global queues — concurrent.

**Что такое `async` для DispatchQueue?**
> `queue.async { work() }` — добавляет задачу в очередь и немедленно возвращает. Не ждёт выполнения. Caller продолжает работу параллельно. Не блокирует текущий поток.

**Что такое `sync` для DispatchQueue?**
> `queue.sync { work() }` — добавляет задачу и **блокирует** текущий поток до завершения. Гарантирует что работа выполнена до продолжения. Вернёт результат через return. Опасно если queue == текущая (deadlock).

---

### Middle

**Что произойдёт при вызове `sync` на main queue из main queue?**
> Deadlock: main thread ждёт задачу в main queue, но main queue не может выполнить задачу пока main thread не освободится. Взаимная блокировка. Приложение зависнет. `DispatchQueue.main.sync { }` из main thread → всегда deadlock.

**Что такое QoS (Quality of Service)?**
> Приоритет работы для планировщика. Определяет насколько срочно система должна выполнить работу. Влияет на: scheduling priority, CPU time, I/O bandwidth. Не абсолютный — относительный приоритет между задачами.

**Какие уровни QoS существуют?**
> От высокого к низкому: `.userInteractive` (UI, анимации), `.userInitiated` (ответ на действие пользователя), `.default` (общий), `.utility` (progress bar, загрузка), `.background` (backup, sync), `.unspecified` (нет QoS). Каждый следующий дешевле по ресурсам.

**Что такое DispatchGroup?**
> Группировка нескольких async задач для ожидания их завершения: `let group = DispatchGroup(); group.enter(); async { work(); group.leave() }; group.notify(queue: .main) { allDone() }`. Или `group.wait()` для синхронного ожидания.

**Как DispatchGroup используется для ожидания нескольких задач?**
```swift
let group = DispatchGroup()
var results: [String] = []
let lock = NSLock()

for url in urls {
    group.enter()
    URLSession.shared.dataTask(with: url) { data, _, _ in
        lock.withLock { results.append(parse(data)) }
        group.leave()
    }.resume()
}

group.notify(queue: .main) {
    updateUI(results)
}
```

**Что такое DispatchSemaphore?**
> Примитив синхронизации: `let sem = DispatchSemaphore(value: N)`. `sem.wait()` — декрементирует, блокирует если 0. `sem.signal()` — инкрементирует, разблокирует ждущих. Применения: ограничение параллелизма, ожидание async операций (антипаттерн на main thread).

**Что такое barrier task?**
> `queue.async(flags: .barrier) { writeOp() }` — барьер в concurrent queue. Все предыдущие задачи завершаются, потом выполняется barrier задача, потом следующие. Паттерн: concurrent reads + barrier writes = thread-safe read-write.

**Как работает `asyncAfter`?**
> `DispatchQueue.main.asyncAfter(deadline: .now() + 2) { doWork() }` — выполнить через 2 секунды. Не блокирует текущий поток. Точность не гарантирована (может выполниться чуть позже). Для отмены: `DispatchWorkItem` + `cancel()`.

**Что такое DispatchWorkItem?**
> Обёртка над задачей с возможностью отмены и уведомления: `let item = DispatchWorkItem { doWork() }; queue.async(execute: item); item.cancel()`. `item.notify(queue: .main) { done() }` — callback по завершении. `item.wait()` — синхронное ожидание.

**Как отменить DispatchWorkItem?**
> `item.cancel()` — помечает как cancelled. Если ещё не начала выполняться — пропускается. Если уже выполняется — `item.isCancelled` становится `true`, код должен сам проверять. Нет принудительного прерывания.

---

### Senior

**Что такое deadlock и как его спровоцировать с GCD?**
> Взаимная блокировка: два потока ждут друг друга. `DispatchQueue.main.sync {}` из main thread. `serialQueue.sync { serialQueue.sync {} }` — вложенный sync на той же serial queue. Избегай: никогда `sync` на текущей queue, никогда `sync` из main thread.

**Как DispatchQueue внутренне управляет потоками?**
> GCD поддерживает thread pool (до 64 потоков на iOS). При добавлении задачи: если есть свободный поток — использует его, иначе создаёт новый (до лимита). Serial queue: всегда один поток за раз (но поток может меняться). Thread explosion — антипаттерн.

**Что такое thread pool при использовании GCD?**
> Общий пул потоков для всех global queues. Размер динамический: увеличивается при нагрузке. `sync` блокирует поток — GCD может создать новый компенсирующий поток (thread explosion). Лимит ~64 потока. Thread switching = overhead.

**Как QoS propagation работает?**
> При добавлении задачи в queue: QoS задачи влияет на QoS queue temporarily. Если `.userInitiated` задача ждёт `.background` queue — QoS `.background` повышается (priority donation). Аналогично Swift Concurrency priority propagation.

**Что такое target queue?**
> `DispatchQueue(label: "my", target: otherQueue)` — задачи из "my" queue выполняются на `otherQueue`. Иерархия очередей. Изменить: `queue.setTarget(queue: targetQueue)`. Используется для группировки QoS. `barrier` на hierarchy — на всю цепочку.

**Как barrier влияет на concurrent queue?**
> `queue.async(flags: .barrier) { ... }` — все pending задачи до barrier выполняются, barrier выполняется эксклюзивно, потом следующие задачи продолжают concurrent выполнение. Reader-Writer: async (reads) + barrier async (writes).

**Что такое DispatchSource?**
> Мониторинг системных событий через GCD: `DispatchSource.makeFileSystemObjectSource(fileDescriptor:)`, `makeTimerSource()`, `makeSignalSource()`. Пример: `DispatchSource.makeTimerSource()` как альтернатива Timer. Не блокирует, callback через handler.

**Как реализовать reader-writer lock через DispatchQueue?**
```swift
class ThreadSafeCache<Key: Hashable, Value> {
    private var cache: [Key: Value] = [:]
    private let queue = DispatchQueue(label: "cache", attributes: .concurrent)
    
    func get(_ key: Key) -> Value? {
        queue.sync { cache[key] }  // concurrent read
    }
    
    func set(_ key: Key, value: Value) {
        queue.async(flags: .barrier) { self.cache[key] = value }  // exclusive write
    }
}
```

**Чем GCD отличается от Swift Concurrency в управлении потоками?**
> GCD: thread pool неограниченный (до 64), блокирующий sync создаёт потоки, нет автоматической отмены, нет structured hierarchy. Swift Concurrency: пул = CPU cores, потоки не блокируются (cooperative), структурированная отмена, priority inheritance. SC эффективнее.

**Как перейти с GCD кода на async/await?**
> `DispatchQueue.main.async { }` → `@MainActor func` или `await MainActor.run { }`. Completion handlers → `withCheckedContinuation`. `DispatchGroup` → `async let` или `TaskGroup`. `DispatchSemaphore` → structured concurrency. `DispatchQueue.async` → `Task { }`.

**Как правильно использовать GCD и async/await вместе?**
> Не смешивай `DispatchQueue.sync` с async/await — deadlock риск. `DispatchQueue.async` внутри async функции — потеря structured concurrency. Правило: на границе GCD↔async используй `withCheckedContinuation`. Постепенно мигрируй GCD → async/await.

---

## Race Condition & Deadlock

### Beginner

**Что такое race condition?**
> Ситуация когда результат программы зависит от непредсказуемого порядка выполнения concurrent операций. `var x = 0; Thread1: x += 1; Thread2: x += 1; // x может быть 1 или 2`. Непредсказуемо, воспроизводится редко — сложно отлаживать.

**Что такое deadlock?**
> Взаимная блокировка: два или более потока ждут ресурсы друг друга → никто не может продолжить. `Thread1: lock A, wait B. Thread2: lock B, wait A`. Приложение зависает. Нет ошибки — просто висит. Детектируется Thread Sanitizer или через hang reports.

**Как deadlock возникает с DispatchQueue?**
> `DispatchQueue.main.sync { }` с main thread. Вложенный `serialQueue.sync { serialQueue.sync { } }`. `semaphore.wait()` на main thread + `semaphore.signal()` в task ждущей main thread. Правило: никогда `sync` на ту же queue с которой вызываешь.

---

### Middle

**Как избежать race condition в iOS?**
> Использовать: actor (Swift Concurrency), serial DispatchQueue, NSLock/OSAllocatedUnfairLock, DispatchSemaphore, barrier tasks. Предпочитай value types (struct) — нет shared mutable state. `@MainActor` для UI state. Thread Sanitizer для детектирования.

**Что такое data race?**
> Конкретный тип race condition: два потока одновременно обращаются к одной памяти, хотя бы один пишет, без синхронизации. В Swift 6: compile-time ошибка. В Swift 5: Thread Sanitizer (TSan) детектирует в runtime. Data race → undefined behavior.

**Как Thread Sanitizer помогает найти race conditions?**
> TSan: инструментирует все memory accesses. Детектирует concurrent read+write без синхронизации. Включить: Edit Scheme → Diagnostics → Thread Sanitizer. Overhead ~5-15x замедление. Только в debug. Показывает stack trace обоих конкурирующих потоков.

**Что такое priority inversion?**
> Low-priority task держит ресурс нужный high-priority task. High-priority ждёт → фактически работает с low priority. Решения: priority inheritance (GCD, OS X), priority donation (Swift Concurrency), избегать длинного удержания lock на низком QoS.

**Как семафор помогает решить проблему race condition?**
> `DispatchSemaphore(value: 1)` как mutex: `sem.wait(); criticalSection(); sem.signal()`. Гарантирует что только один поток в критической секции. Но `wait()` блокирует поток — не используй на main thread или в async контексте.

**Что такое mutual exclusion?**
> Гарантия что только один поток одновременно выполняет критическую секцию. Реализации: NSLock, OSAllocatedUnfairLock, serial DispatchQueue, actor, @synchronized (ObjC). Основа thread safety для shared mutable state.

---

### Senior

**Чем отличается race condition от data race?**
> Data race — конкретный технический случай: concurrent memory access без sync. Race condition — логическая ошибка: результат зависит от timing. Data race → UB (memory corruption). Race condition → неправильный результат но без UB. Data race → всегда race condition. Race condition не обязательно data race.

**Как Actor решает проблему data races?**
> Actor isolation: только одна coroutine одновременно выполняет code на actor. Нет concurrent access к actor state. Компилятор статически проверяет что нет прямого доступа к actor state снаружи. `Sendable` гарантирует безопасность передаваемых данных.

**Как доказать отсутствие race condition в коде?**
> Формальные методы: model checking. Практически: TSan в тестах (не гарантия). Инспекция: каждый shared mutable state должен иметь явную стратегию синхронизации. Swift 6 strict concurrency: compile-time доказательство для основных случаев. Code review с фокусом на concurrency.

**Что такое lock-free programming?**
> Алгоритмы без locks, используют atomic operations (CAS — Compare And Swap). Преимущества: нет deadlock, нет priority inversion, высокая производительность. Сложность: ABA problem, memory ordering. В Swift: через `Atomics` из swift-atomics пакета.

**Что такое atomic operations и как они реализованы в Swift?**
> Операции гарантированно выполняющиеся как единое неделимое целое. `swift-atomics` пакет: `ManagedAtomic<Int>`. `load(ordering:)`, `store(_:ordering:)`, `compareExchange(expected:desired:ordering:)`. Базируются на `_Atomic` LLVM intrinsics → CPU-level atomic instructions (LOCK prefix на x86, LDREX/STREX на ARM).

**Как OSAllocatedUnfairLock отличается от NSLock?**
> `OSAllocatedUnfairLock` (iOS 16+): хранится на стеке, не требует heap allocation, faster (unfair — нет FIFO гарантии). `NSLock`: heap-allocated Objective-C object, fair (FIFO), чуть медленнее. Для hot paths — `OSAllocatedUnfairLock`. Для совместимости — `NSLock`.

**Как реализовать thread-safe счётчик без actor?**
```swift
// С OSAllocatedUnfairLock (iOS 16+):
final class Counter: @unchecked Sendable {
    private var _value = 0
    private let lock = OSAllocatedUnfairLock()
    
    var value: Int { lock.withLock { _value } }
    func increment() { lock.withLock { _value += 1 } }
}

// Или через actor (предпочтительно):
actor Counter {
    private(set) var value = 0
    func increment() { value += 1 }
}
```

---

## Operations

### Beginner

**Что такое `Operation`?**
> Абстракция единицы работы: `class MyOperation: Operation { override func main() { doWork() } }`. OOP wrapper над задачей. Поддерживает: отмену (`cancel()`), зависимости между задачами, приоритеты, состояния. Мощнее чем GCD block.

**Что такое `OperationQueue`?**
> Очередь `Operation` объектов: `let queue = OperationQueue(); queue.addOperation(myOp)`. Управляет параллелизмом: `queue.maxConcurrentOperationCount`. `OperationQueue.main` — на main thread. Автоматически управляет потоками.

**Чем Operation отличается от DispatchQueue?**
> Operation: OOP, отмена, зависимости между задачами, KVO для состояния, приоритеты. DispatchQueue: функциональный, легче, быстрее, меньше overhead. Operations для сложной orchestration задач. GCD для простого dispatch. Под капотом OperationQueue использует GCD.

---

### Middle

**Что такое зависимости между операциями?**
> `opB.addDependency(opA)` — opB начнётся только после завершения opA. Можно строить граф: `C.addDependency(A); C.addDependency(B)` — C ждёт A и B. `removeDependency(_:)` — убрать зависимость. Зависимости cross-queue работают.

**Как отменить операцию?**
> `operation.cancel()` — устанавливает `isCancelled = true`. Не прерывает принудительно — операция должна сама проверять: `if isCancelled { return }` в `main()`. `queue.cancelAllOperations()` — отменить все. Зависимые операции: если dependency отменена — зависимая всё равно запустится (проверяй `dependency.isCancelled`).

**Как реализовать асинхронную Operation?**
```swift
class AsyncOperation: Operation {
    private var _isExecuting = false
    private var _isFinished = false
    
    override var isAsynchronous: Bool { true }
    override var isExecuting: Bool { _isExecuting }
    override var isFinished: Bool { _isFinished }
    
    override func start() {
        willChangeValue(forKey: "isExecuting")
        _isExecuting = true
        didChangeValue(forKey: "isExecuting")
        
        doAsyncWork { [weak self] in
            self?.complete()
        }
    }
    
    func complete() {
        willChangeValue(forKey: "isExecuting")
        willChangeValue(forKey: "isFinished")
        _isExecuting = false
        _isFinished = true
        didChangeValue(forKey: "isExecuting")
        didChangeValue(forKey: "isFinished")
    }
}
```

**Что такое operation states (ready, executing, finished)?**
> `isReady`: готова к запуску (все dependencies выполнены). `isExecuting`: выполняется сейчас. `isFinished`: завершена (успешно или с отменой). `isCancelled`: отменена. State transitions KVO-наблюдаемы. Для async operation — нужно вручную уведомлять через KVO.

**Как ограничить параллелизм через OperationQueue?**
> `queue.maxConcurrentOperationCount = 4` — не более 4 одновременных операций. `= 1` — serial queue (одна за одной). `= OperationQueue.defaultMaxConcurrentOperationCount` — система решает. Полезно для: ограничения сетевых запросов, memory pressure.

**Что такое `BlockOperation`?**
> Удобный wrapper для замыканий: `BlockOperation { doWork() }`. Можно добавить несколько блоков: `op.addExecutionBlock { moreWork() }` — выполняются concurrent. Поддерживает dependencies как обычная Operation. Быстрее чем создавать subclass.

---

### Senior

**Когда предпочесть OperationQueue вместо GCD?**
> Используй OperationQueue когда нужно: зависимости между задачами (A должен завершиться до B), отмена с проверкой состояния, переиспользуемые задачи (subclassing), KVO для мониторинга прогресса. GCD — для простого async dispatch без сложной orchestration.

**Как реализовать retry через Operation?**
```swift
class RetryableOperation: Operation {
    private let maxRetries: Int
    private var retryCount = 0
    
    override func main() {
        while !isCancelled && retryCount < maxRetries {
            do {
                try performWork()
                return  // success
            } catch {
                retryCount += 1
                Thread.sleep(forTimeInterval: Double(retryCount) * 0.5)  // backoff
            }
        }
    }
}
```

**Как реализовать граф зависимостей через Operations?**
```swift
// Параллельная загрузка + последовательная обработка
let downloadA = DownloadOperation(url: urlA)
let downloadB = DownloadOperation(url: urlB)
let processA = ProcessOperation(input: downloadA)
let processB = ProcessOperation(input: downloadB)
let merge = MergeOperation()

processA.addDependency(downloadA)
processB.addDependency(downloadB)
merge.addDependency(processA)
merge.addDependency(processB)

let queue = OperationQueue()
queue.addOperations([downloadA, downloadB, processA, processB, merge], waitUntilFinished: false)
```

**Как Operations взаимодействуют с async/await?**
> Нет прямой интеграции. Паттерн: создать async wrapper: `func execute(_ operation: Operation) async { await withCheckedContinuation { cont in operation.completionBlock = { cont.resume() }; operationQueue.addOperation(operation) } }`. Или мигрировать на TaskGroup постепенно.

**Как отслеживать прогресс Operation?**
> `Progress` object: `let progress = Progress(totalUnitCount: 100); operation.progress = progress`. KVO на `progress.fractionCompleted`. `OperationQueue` + `Progress` hierarchy. Или custom KVO properties в Operation subclass. В Swift: Combine `@Published var progress: Double`.

---

*Конец документа · Concurrency & GCD*
