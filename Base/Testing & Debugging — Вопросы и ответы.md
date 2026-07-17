# Testing & Debugging — Вопросы и ответы
> iOS-собеседование · XCTest · UI Tests · LLDB · Instruments · Debugging
> Уровни: Beginner · Middle · Senior · Staff

---

## Unit Tests

### Beginner

**Что такое unit тест?**
> Автоматическая проверка одной изолированной единицы кода (функция, метод, класс) в контролируемых условиях. Цель: убедиться что каждый компонент работает корректно независимо от других. Быстро выполняются, воспроизводимы, не зависят от сети/БД/файловой системы.

**Что такое `XCTestCase`?**
> Базовый класс для тестов в Apple's XCTest framework. Наследуй для создания тест-сьюта:
```swift
class CalculatorTests: XCTestCase {
    // Методы начинающиеся с test — автоматически запускаются
    func testAddition() {
        XCTAssertEqual(2 + 2, 4)
    }
    
    func testDivisionByZero() {
        // тест кейс
    }
}
```
> Xcode обнаруживает все классы наследующие XCTestCase и методы с префиксом `test`.

**Что такое `XCTAssert`?**
> Семейство функций для проверок в тестах. При невыполнении — тест падает с описанием:
```swift
XCTAssertEqual(result, 42)               // равенство
XCTAssertNotEqual(a, b)                  // неравенство
XCTAssertTrue(condition)                 // истинность
XCTAssertFalse(condition)                // ложность
XCTAssertNil(value)                      // nil
XCTAssertNotNil(value)                   // не nil
XCTAssertGreaterThan(a, b)               // a > b
XCTAssertLessThanOrEqual(a, b)           // a <= b
XCTAssertThrowsError(try dangerousFunc()) // бросает ошибку
XCTAssertNoThrow(try safeFunc())          // не бросает
XCTFail("Этот код не должен выполняться") // принудительный провал
```

**Что такое `setUp` и `tearDown`?**
```swift
class UserServiceTests: XCTestCase {
    var sut: UserService!  // System Under Test
    var mockRepo: MockUserRepository!
    
    override func setUp() {
        super.setUp()
        // Вызывается ПЕРЕД каждым тестом
        // Инициализация зависимостей, настройка моков
        mockRepo = MockUserRepository()
        sut = UserService(repository: mockRepo)
    }
    
    override func tearDown() {
        // Вызывается ПОСЛЕ каждого теста
        // Очистка ресурсов, сброс состояния
        sut = nil
        mockRepo = nil
        super.tearDown()
    }
    
    // Async версии (Swift 5.5+):
    override func setUp() async throws { }
    override func tearDown() async throws { }
    
    // Один раз для всего класса:
    override class func setUp() { }    // перед всеми тестами класса
    override class func tearDown() { } // после всех тестов класса
}
```

**Что такое test target?**
> Отдельный target в Xcode для тестов. Два типа: `Unit Testing Bundle` (быстрые, без UI) и `UI Testing Bundle` (медленные, с симулятором). Тесты не входят в production binary. Настройка: Project → Targets → + → Unit Test Bundle. `@testable import ModuleName` — доступ к `internal` API.

**Что такое `@testable import`?**
> Позволяет в тест-таргете обращаться к `internal` символам модуля (без изменения access level):
```swift
@testable import MyApp  // в тест файле

// Теперь можно обращаться к internal:
let vm = UserListViewModel()  // если internal — ок в тестах
vm.internalMethod()           // доступно
```
> `open` и `public` — доступны и без `@testable`. `private` и `fileprivate` — недоступны даже с `@testable`.

---

### Middle

**Что такое Mock, Spy, Stub, Fake?**
> Все — Test Doubles (замены реальных зависимостей в тестах):

| Тип | Описание | Цель |
|-----|----------|------|
| **Stub** | Возвращает заранее заданные данные | Изолировать тест от реальной зависимости |
| **Mock** | Stub + проверяет что методы вызваны правильно | Верифицировать поведение |
| **Spy** | Записывает вызовы для последующей проверки | Интроспекция без strict expectations |
| **Fake** | Рабочая упрощённая реализация | In-memory DB вместо реальной |

```swift
// Stub — фиксированный ответ:
class StubNetworkClient: NetworkClient {
    func fetch(url: URL) async throws -> Data {
        return Data("{\"id\": 1}".utf8)  // всегда возвращает это
    }
}

// Spy — записывает вызовы:
class SpyAnalytics: Analytics {
    var trackedEvents: [String] = []
    func track(_ event: String) { trackedEvents.append(event) }
}

// Mock — stub + верификация:
class MockNetworkClient: NetworkClient {
    var fetchCallCount = 0
    var lastURL: URL?
    var stubbedData: Data = Data()
    var shouldThrow = false
    
    func fetch(url: URL) async throws -> Data {
        fetchCallCount += 1
        lastURL = url
        if shouldThrow { throw NetworkError.notFound }
        return stubbedData
    }
}
```

**В чём разница между Mock и Stub?**
> **Stub** — пассивный: просто возвращает данные, не проверяет как его используют. **Mock** — активный: содержит expectations о том, как его должны вызвать, и проверяет это. Stub изолирует, Mock верифицирует поведение. В Swift чаще используют Spy паттерн (записывать + проверять в assert).

**Как тестировать асинхронный код?**
```swift
// Способ 1: async throws тест (Swift 5.5+) — предпочтительно:
func test_fetchUser_success() async throws {
    let sut = UserService(client: StubClient())
    let user = try await sut.fetchUser(id: 1)
    XCTAssertEqual(user.name, "Alice")
}

// Способ 2: XCTestExpectation (для completion handlers):
func test_fetchUser_withCallback() {
    let exp = expectation(description: "user fetched")
    var result: User?
    
    sut.fetchUser(id: 1) { user in
        result = user
        exp.fulfill()
    }
    
    waitForExpectations(timeout: 1)
    XCTAssertNotNil(result)
}

// Способ 3: Combine тест:
func test_publisherEmitsValue() {
    let exp = expectation(description: "value received")
    var received: [Int] = []
    
    [1,2,3].publisher
        .sink { received.append($0) }
        .store(in: &cancellables)
    
    exp.fulfill()
    waitForExpectations(timeout: 0.1)
    XCTAssertEqual(received, [1,2,3])
}
```

**Что такое `XCTestExpectation`?**
```swift
// Базовое использование:
func test_asyncOperation() {
    let exp = expectation(description: "operation completes")
    
    asyncOperation { result in
        XCTAssertNotNil(result)
        exp.fulfill()  // сигнал что условие выполнено
    }
    
    // Ждём максимум 3 секунды:
    waitForExpectations(timeout: 3) { error in
        if let error = error { XCTFail("Timeout: \(error)") }
    }
}

// Несколько ожиданий:
let exp1 = expectation(description: "first")
let exp2 = expectation(description: "second")
wait(for: [exp1, exp2], timeout: 5)

// N вызовов:
let exp = expectation(description: "called 3 times")
exp.expectedFulfillmentCount = 3

// Инвертированное (убедиться что НЕ вызовется):
let exp = expectation(description: "should not be called")
exp.isInverted = true
wait(for: [exp], timeout: 0.5)
```

**Как тестировать throws функции?**
```swift
// Проверить что бросает:
func test_parse_invalidJSON_throwsError() throws {
    let invalidData = Data("not json".utf8)
    XCTAssertThrowsError(try parseJSON(invalidData)) { error in
        XCTAssertEqual(error as? ParseError, ParseError.invalidFormat)
    }
}

// Проверить что НЕ бросает:
func test_parse_validJSON_succeeds() throws {
    let validData = Data("{\"id\": 1}".utf8)
    XCTAssertNoThrow(try parseJSON(validData))
}

// Или использовать try напрямую (тест упадёт при ошибке):
func test_parse_returnsCorrectValue() throws {
    let data = Data("{\"id\": 42}".utf8)
    let result = try parseJSON(data)  // throws → тест failure
    XCTAssertEqual(result.id, 42)
}
```

**Что такое test coverage?**
> Процент кода покрытого тестами. Включить: Edit Scheme → Test → Options → Code Coverage. Просматривать: Report Navigator → Coverage. Типы: Line coverage (строки), Branch coverage (ветви if/else), Function coverage.
```
Цели:
  Business logic (ViewModel, UseCase): 80–90%
  UI код: 40–60%
  Data models: 70–80%
  Весь проект: 60–70% разумная цель
  
НЕ гонись за 100% — возвраты diminishing returns
Важнее качество тестов чем количество
```

**Как тестировать View Model?**
```swift
// ViewModel с протоколом для тестирования:
protocol UserRepositoryProtocol {
    func fetchUsers() async throws -> [User]
}

class UserListViewModel: ObservableObject {
    @Published var users: [User] = []
    @Published var error: String?
    @Published var isLoading = false
    
    private let repository: UserRepositoryProtocol
    
    init(repository: UserRepositoryProtocol) {
        self.repository = repository
    }
    
    func loadUsers() async {
        isLoading = true
        do {
            users = try await repository.fetchUsers()
        } catch {
            self.error = error.localizedDescription
        }
        isLoading = false
    }
}

// Mock репозиторий:
class MockUserRepository: UserRepositoryProtocol {
    var usersToReturn: [User] = []
    var errorToThrow: Error?
    var fetchCallCount = 0
    
    func fetchUsers() async throws -> [User] {
        fetchCallCount += 1
        if let error = errorToThrow { throw error }
        return usersToReturn
    }
}

// Тесты:
class UserListViewModelTests: XCTestCase {
    var sut: UserListViewModel!
    var mockRepo: MockUserRepository!
    
    override func setUp() {
        super.setUp()
        mockRepo = MockUserRepository()
        sut = UserListViewModel(repository: mockRepo)
    }
    
    func test_loadUsers_onSuccess_updatesUsers() async {
        // Given:
        let expectedUsers = [User(id: 1, name: "Alice")]
        mockRepo.usersToReturn = expectedUsers
        
        // When:
        await sut.loadUsers()
        
        // Then:
        XCTAssertEqual(sut.users, expectedUsers)
        XCTAssertNil(sut.error)
        XCTAssertFalse(sut.isLoading)
    }
    
    func test_loadUsers_onFailure_setsError() async {
        // Given:
        mockRepo.errorToThrow = URLError(.notConnectedToInternet)
        
        // When:
        await sut.loadUsers()
        
        // Then:
        XCTAssertTrue(sut.users.isEmpty)
        XCTAssertNotNil(sut.error)
    }
    
    func test_loadUsers_setsLoadingState() async {
        // Проверить что isLoading меняется корректно:
        XCTAssertFalse(sut.isLoading)
        let task = Task { await self.sut.loadUsers() }
        // isLoading = true во время загрузки...
        await task.value
        XCTAssertFalse(sut.isLoading)
    }
}
```

**Что такое Dependency Injection для тестирования?**
> DI = передача зависимостей через init/property вместо создания внутри класса. Позволяет в тестах подменить реальные зависимости на моки:
```swift
// ❌ Плохо: зависимость создаётся внутри — нельзя подменить:
class BadViewModel {
    private let service = NetworkService()  // hardcoded
}

// ✅ Хорошо: зависимость передаётся снаружи:
class GoodViewModel {
    private let service: NetworkServiceProtocol
    
    init(service: NetworkServiceProtocol = NetworkService()) {
        self.service = service
    }
}

// В production: GoodViewModel() — использует реальный сервис
// В тестах: GoodViewModel(service: MockNetworkService())
```

**Как тестировать CoreData?**
```swift
// Используй in-memory store — быстро, без файла:
class CoreDataTests: XCTestCase {
    var container: NSPersistentContainer!
    var context: NSManagedObjectContext!
    
    override func setUp() {
        super.setUp()
        container = NSPersistentContainer(name: "MyModel")
        
        // In-memory конфигурация:
        let description = NSPersistentStoreDescription()
        description.type = NSInMemoryStoreType
        container.persistentStoreDescriptions = [description]
        
        container.loadPersistentStores { _, error in
            XCTAssertNil(error)
        }
        context = container.viewContext
    }
    
    func test_createUser_savesSuccessfully() throws {
        let user = User(context: context)
        user.name = "Alice"
        user.age = 30
        
        try context.save()
        
        let request = User.fetchRequest()
        let users = try context.fetch(request)
        
        XCTAssertEqual(users.count, 1)
        XCTAssertEqual(users.first?.name, "Alice")
    }
    
    override func tearDown() {
        context = nil
        container = nil
        super.tearDown()
    }
}
```

---

### Senior

**Как тестировать Actor-изолированный код?**
```swift
actor Counter {
    private(set) var value = 0
    func increment() { value += 1 }
    func reset() { value = 0 }
}

class CounterTests: XCTestCase {
    // async тесты автоматически поддерживают await:
    func test_increment_increasesValue() async {
        let counter = Counter()
        await counter.increment()
        await counter.increment()
        let value = await counter.value
        XCTAssertEqual(value, 2)
    }
    
    // @MainActor проверка:
    @MainActor
    func test_mainActorProperty() {
        let vm = MainActorViewModel()
        vm.update()  // синхронно — мы на MainActor
        XCTAssertEqual(vm.title, "Updated")
    }
    
    // Тест с TaskGroup:
    func test_concurrentIncrements() async {
        let counter = Counter()
        await withTaskGroup(of: Void.self) { group in
            for _ in 0..<100 {
                group.addTask { await counter.increment() }
            }
        }
        let value = await counter.value
        XCTAssertEqual(value, 100)  // actor гарантирует отсутствие race
    }
}
```

**Что такое async тесты с `async throws`?**
```swift
class AsyncTests: XCTestCase {
    // Просто добавь async throws — XCTest автоматически поддерживает:
    func test_fetchUser_returnsCorrectData() async throws {
        let service = UserService(client: StubClient())
        let user = try await service.fetchUser(id: 1)
        XCTAssertEqual(user.name, "Alice")
        // Нет expectation, нет waitForExpectations — всё синхронно читается
    }
    
    // Тест с timeout (Swift 5.9+):
    func test_slowOperation_completesInTime() async throws {
        let result = try await withTimeout(seconds: 2) {
            try await slowOperation()
        }
        XCTAssertNotNil(result)
    }
}
```

**Как реализовать Protocol-based mocking?**
```swift
// 1. Протокол для каждой зависимости:
protocol NetworkClient {
    func fetch<T: Decodable>(_ request: URLRequest) async throws -> T
}

// 2. Mock реализация:
class MockNetworkClient: NetworkClient {
    var responses: [URLRequest: Any] = [:]
    var errors: [URLRequest: Error] = [:]
    var requests: [URLRequest] = []
    
    func fetch<T: Decodable>(_ request: URLRequest) async throws -> T {
        requests.append(request)
        if let error = errors[request] { throw error }
        if let response = responses[request] { return response as! T }
        fatalError("No stub for \(request)")
    }
    
    func stub<T>(_ value: T, for request: URLRequest) {
        responses[request] = value
    }
}

// 3. Использование:
func test_fetchUser() async throws {
    let mock = MockNetworkClient()
    let expectedUser = User(id: 1, name: "Alice")
    mock.stub(expectedUser, for: userRequest(id: 1))
    
    let service = UserService(client: mock)
    let user = try await service.fetchUser(id: 1)
    
    XCTAssertEqual(user, expectedUser)
    XCTAssertEqual(mock.requests.count, 1)
}
```

**Что такое Snapshot Testing?**
> Тестирование скриншотов UI: при первом запуске — сохраняет reference snapshot. При последующих — сравнивает pixel by pixel. Обнаруживает неожиданные визуальные регрессии.
```swift
// Используя swift-snapshot-testing (pointfree):
import SnapshotTesting

class ViewSnapshotTests: XCTestCase {
    func test_userCard_light() {
        let view = UserCardView(user: .fixture())
        // Первый запуск: сохраняет снимок
        // Последующие: сравнивает
        assertSnapshot(matching: view, as: .image(on: .iPhone13))
    }
    
    func test_userCard_dark() {
        let view = UserCardView(user: .fixture())
        assertSnapshot(
            matching: view,
            as: .image(on: .iPhone13, traits: .init(userInterfaceStyle: .dark))
        )
    }
    
    // SwiftUI View:
    func test_loginScreen() {
        let view = LoginView()
        assertSnapshot(matching: view, as: .image(layout: .device(config: .iPhone13)))
    }
}
```

**Как тестировать Combine Publishers?**
```swift
class CombineTests: XCTestCase {
    var cancellables = Set<AnyCancellable>()
    
    // Синхронный (Just, [].publisher и т.д.):
    func test_mapOperator() {
        var received: [Int] = []
        [1, 2, 3].publisher
            .map { $0 * 2 }
            .sink { received.append($0) }
            .store(in: &cancellables)
        XCTAssertEqual(received, [2, 4, 6])
    }
    
    // Async publisher через values:
    func test_asyncPublisher() async {
        let values = await [1, 2, 3].publisher
            .map { $0 * 2 }
            .values
            .reduce(into: []) { $0.append($1) }
        XCTAssertEqual(values, [2, 4, 6])
    }
    
    // С expectation:
    func test_futurePublisher() {
        let exp = expectation(description: "received")
        var result: Int?
        
        Future<Int, Never> { promise in
            DispatchQueue.main.asyncAfter(deadline: .now() + 0.1) {
                promise(.success(42))
            }
        }
        .sink { value in
            result = value
            exp.fulfill()
        }
        .store(in: &cancellables)
        
        waitForExpectations(timeout: 1)
        XCTAssertEqual(result, 42)
    }
    
    // Тест @Published:
    func test_viewModelPublished() {
        let vm = SearchViewModel()
        var states: [Bool] = []
        
        vm.$isLoading
            .sink { states.append($0) }
            .store(in: &cancellables)
        
        vm.search("query")
        
        XCTAssertEqual(states.first, false)   // начальное
        XCTAssertEqual(states.dropFirst().first, true)  // во время поиска
    }
}
```

**Что такое property-based testing?**
> Вместо конкретных примеров — генерируются сотни случайных входных данных. Проверяется что свойство (инвариант) выполняется для всех. Библиотека: `SwiftCheck`.
```swift
// Инвариант: reversing twice = original
property("reversing twice equals identity") <- forAll { (xs: [Int]) in
    xs.reversed().reversed() == xs
}

// Инвариант: sorted array is non-decreasing
property("sort is non-decreasing") <- forAll { (xs: [Int]) in
    let sorted = xs.sorted()
    return zip(sorted, sorted.dropFirst()).allSatisfy { $0 <= $1 }
}

// Без библиотеки — базовое fuzzing:
func test_parser_handlesArbitraryInput() {
    for _ in 0..<1000 {
        let randomString = String((0..<Int.random(in: 0...100))
            .map { _ in Character(UnicodeScalar(Int.random(in: 32...126))!) })
        // Не должен крашиться:
        _ = try? parser.parse(randomString)
    }
}
```

**Как организовать тесты для большого приложения?**
```
Структура:
MyApp/
├── Sources/
│   ├── Features/Auth/
│   ├── Features/Feed/
│   └── Core/
└── Tests/
    ├── UnitTests/
    │   ├── Features/AuthTests/
    │   ├── Features/FeedTests/
    │   └── Core/
    ├── IntegrationTests/
    │   └── NetworkTests/
    └── UITests/
        └── Flows/

Принципы:
• Один тест-файл на один production-файл
• Fixtures/Helpers в отдельном shared target
• Общие моки — в TestSupport модуле
• Название: <TestedClass>Tests.swift
• Нейминг: test_methodName_condition_expectedResult

Категоризация по скорости:
• Fast Unit Tests: < 10ms
• Slow Unit Tests: > 10ms (CoreData, Keychain)  
• Integration Tests: реальная сеть/БД
• UI Tests: полный симулятор
```

**Что такое test double hierarchy?**
```
Все называются Test Doubles:
  Dummy     — объект который просто передаётся, не используется
  Stub      — возвращает заготовленные ответы
  Spy       — записывает вызовы для последующей проверки
  Mock      — ожидания встроены в объект (verifiable)
  Fake      — рабочая упрощённая реализация

Практика в Swift:
  Stub: протокол с фиксированными return values
  Spy:  протокол + массив записанных вызовов
  Mock: Spy + assertions внутри методов
  Fake: InMemoryDatabase, StubServer
```

**Как реализовать contract testing?**
> Тестирование что клиент и сервер соблюдают общий контракт (API):
```swift
// Contract: /users/{id} возвращает User с полями id, name, email
class UserAPIContractTests: XCTestCase {
    
    // Provider side (сервер):
    func test_getUserEndpoint_returnsValidSchema() async throws {
        let response = try await realAPIClient.fetchUser(id: 1)
        
        // Проверить схему ответа:
        XCTAssertNotNil(response.id)
        XCTAssertFalse(response.name.isEmpty)
        XCTAssertTrue(response.email.contains("@"))
    }
    
    // Consumer side (клиент):
    func test_clientParsesUserResponse() throws {
        // Фиксированный ответ сервера из контракта:
        let contractJSON = """
        {"id": 1, "name": "Alice", "email": "alice@example.com"}
        """.data(using: .utf8)!
        
        let user = try JSONDecoder().decode(User.self, from: contractJSON)
        XCTAssertEqual(user.id, 1)
        XCTAssertEqual(user.name, "Alice")
    }
}
```

**Как тестировать сетевой слой?**
```swift
// 1. URLProtocol stub — перехватывает URLSession запросы:
class MockURLProtocol: URLProtocol {
    static var requestHandler: ((URLRequest) throws -> (HTTPURLResponse, Data))?
    
    override class func canInit(with request: URLRequest) -> Bool { true }
    override class func canonicalRequest(for request: URLRequest) -> URLRequest { request }
    
    override func startLoading() {
        guard let handler = MockURLProtocol.requestHandler else { return }
        do {
            let (response, data) = try handler(request)
            client?.urlProtocol(self, didReceive: response, cacheStoragePolicy: .notAllowed)
            client?.urlProtocol(self, didLoad: data)
            client?.urlProtocolDidFinishLoading(self)
        } catch {
            client?.urlProtocol(self, didFailWithError: error)
        }
    }
    
    override func stopLoading() {}
}

class NetworkTests: XCTestCase {
    var session: URLSession!
    
    override func setUp() {
        let config = URLSessionConfiguration.ephemeral
        config.protocolClasses = [MockURLProtocol.self]
        session = URLSession(configuration: config)
    }
    
    func test_fetchUser_success() async throws {
        // Given:
        MockURLProtocol.requestHandler = { request in
            let response = HTTPURLResponse(
                url: request.url!,
                statusCode: 200,
                httpVersion: nil,
                headerFields: nil
            )!
            let data = Data("""{"id":1,"name":"Alice"}""".utf8)
            return (response, data)
        }
        
        // When:
        let client = NetworkClient(session: session)
        let user: User = try await client.fetch(URL(string: "https://api.example.com/users/1")!)
        
        // Then:
        XCTAssertEqual(user.id, 1)
        XCTAssertEqual(user.name, "Alice")
    }
    
    func test_fetchUser_404_throwsError() async {
        MockURLProtocol.requestHandler = { _ in
            let response = HTTPURLResponse(url: URL(string: "https://api.com")!, statusCode: 404, httpVersion: nil, headerFields: nil)!
            return (response, Data())
        }
        
        let client = NetworkClient(session: session)
        do {
            let _: User = try await client.fetch(URL(string: "https://api.example.com/users/999")!)
            XCTFail("Должен бросить ошибку")
        } catch {
            XCTAssertEqual((error as? NetworkError), .notFound)
        }
    }
}
```

---

### Staff

**Как добиться 80% test coverage в legacy codebase?**
```
Стратегия:
1. Не пытаться покрыть всё сразу — это ловушка
2. Определить критические пути (оплата, авторизация, core flow)
3. Покрывать при каждом изменении: "Boy Scout Rule"
   — изменяешь метод → пиши тест для него
4. Characterization tests: тест СУЩЕСТВУЮЩЕГО поведения (даже если оно неправильное)
   — потом рефакторинг безопасно
5. Инжектировать зависимости постепенно через Extract Protocol
6. Feature flags: новый код → всегда с тестами
7. Метрика в CI: coverage не должна падать при каждом PR
```

**Что такое mutation testing?**
> Проверка качества тестов: автоматически вносятся мутации в исходный код (меняет `+` на `-`, `>` на `<`), запускаются тесты. Если тесты проходят с мутацией — тесты слабые (не ловят баги). Инструмент: `muter` для Swift.
```
// Исходный код:
func isAdult(_ age: Int) -> Bool { age >= 18 }

// Мутация 1: >= → >
func isAdult(_ age: Int) -> Bool { age > 18 }
// Если тест не падает при age=18 — тест неполный!

// Мутация 2: >= → <=  
func isAdult(_ age: Int) -> Bool { age <= 18 }
// Хорошие тесты должны поймать это
```

**Как интегрировать тесты в CI/CD pipeline?**
```yaml
# GitHub Actions пример:
name: Tests

on: [push, pull_request]

jobs:
  unit-tests:
    runs-on: macos-14
    steps:
      - uses: actions/checkout@v4
      
      - name: Run Unit Tests
        run: |
          xcodebuild test \
            -scheme MyApp \
            -destination 'platform=iOS Simulator,name=iPhone 15' \
            -enableCodeCoverage YES \
            -resultBundlePath TestResults.xcresult \
            | xcbeautify
      
      - name: Check Coverage Threshold
        run: |
          xcrun xccov view --report --json TestResults.xcresult \
            | python3 check_coverage.py --threshold 70
      
      - name: Upload Results
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: test-results
          path: TestResults.xcresult

  ui-tests:
    runs-on: macos-14
    # Запускать только на main/develop — UI тесты медленные
    if: github.ref == 'refs/heads/main'
    steps:
      - name: Run UI Tests
        run: |
          xcodebuild test \
            -scheme MyAppUITests \
            -destination 'platform=iOS Simulator,name=iPhone 15'
```

**Как реализовать performance regression testing?**
```swift
// XCTest Performance Tests:
func test_sortLargeArray_performance() {
    let array = (0..<10000).map { _ in Int.random(in: 0...1000000) }
    
    measure {
        _ = array.sorted()  // замеряет этот блок 10 раз
    }
    // Xcode автоматически сравнивает с baseline
    // Падает если время увеличилось > threshold
}

// Кастомные метрики:
func test_jsonParsing_performance() {
    let data = loadLargeJSONFixture()
    
    let options = XCTMeasureOptions()
    options.iterationCount = 5
    
    measure(metrics: [XCTCPUMetric(), XCTMemoryMetric()], options: options) {
        _ = try? JSONDecoder().decode([User].self, from: data)
    }
}

// Установить baseline: Editor → Set Performance Baseline
// При регрессии (> 10% замедление) — тест падает
```

---

## UI Tests

### Beginner

**Что такое UI тесты?**
> Автоматизация взаимодействия пользователя с реальным приложением через симулятор/устройство. Медленные (секунды на тест), но проверяют end-to-end flow. Используют Accessibility API. Запуск: UITesting Bundle target в Xcode.

**Что такое `XCUIApplication`?**
```swift
class LoginUITests: XCTestCase {
    let app = XCUIApplication()
    
    override func setUp() {
        super.setUp()
        continueAfterFailure = false  // стоп при первом провале
        
        // Настройка перед запуском:
        app.launchArguments = ["--uitesting"]
        app.launchEnvironment = ["API_BASE_URL": "http://localhost:8080"]
        
        app.launch()  // запустить приложение
    }
    
    // Другие приложения:
    let safari = XCUIApplication(bundleIdentifier: "com.apple.mobilesafari")
    let springboard = XCUIApplication(bundleIdentifier: "com.apple.springboard")
}
```

**Что такое accessibility identifier?**
```swift
// В production коде (View):
button.accessibilityIdentifier = "loginButton"
// SwiftUI:
Button("Login") { }.accessibilityIdentifier("loginButton")

// В UI тесте:
app.buttons["loginButton"].tap()

// Почему accessibilityIdentifier лучше текста:
// ✅ Не ломается при переводе (локализация)
// ✅ Не ломается при изменении текста
// ✅ Работает независимо от visibility
// ❌ Нужно поддерживать синхронность
```

**Как запустить UI тест?**
```swift
class OnboardingTests: XCTestCase {
    func test_onboarding_completesSuccessfully() {
        let app = XCUIApplication()
        app.launch()
        
        // Нажать кнопку:
        app.buttons["getStartedButton"].tap()
        
        // Ввести текст:
        let emailField = app.textFields["emailField"]
        emailField.tap()
        emailField.typeText("user@example.com")
        
        // Проверить что элемент существует:
        XCTAssertTrue(app.staticTexts["Welcome"].exists)
        
        // Свайп:
        app.swipeLeft()
        
        // Скролл:
        app.scrollViews.firstMatch.swipeUp()
    }
}
```

---

### Middle

**Как тестировать навигацию?**
```swift
func test_tapLogin_navigatesToHome() {
    let app = XCUIApplication()
    app.launch()
    
    // Заполнить форму:
    app.textFields["email"].tap()
    app.textFields["email"].typeText("user@test.com")
    app.secureTextFields["password"].typeText("password123")
    
    // Нажать:
    app.buttons["loginButton"].tap()
    
    // Проверить навигацию:
    let homeTitle = app.navigationBars["Home"]
    XCTAssertTrue(homeTitle.waitForExistence(timeout: 3))
    
    // Проверить что экрана логина нет:
    XCTAssertFalse(app.buttons["loginButton"].exists)
}

// Назад:
app.navigationBars.buttons.firstMatch.tap()
// или:
app.navigationBars["Title"].buttons["Back"].tap()
```

**Что такое `XCUIElement`?**
```swift
// Элемент UI — proxy до реального view
let button = app.buttons["loginButton"]

// Типы элементов:
app.buttons["OK"]
app.textFields["username"]
app.secureTextFields["password"]
app.staticTexts["Welcome"]
app.images["avatar"]
app.tables.cells.firstMatch
app.collectionViews.cells["cellId"]
app.switches["notificationSwitch"]
app.sliders["volumeSlider"]
app.segmentedControls["tabs"]
app.alerts.firstMatch

// Запросы:
app.buttons.count                    // количество кнопок
app.cells.matching(identifier: "id") // по identifier
app.descendants(matching: .button)   // все кнопки в иерархии

// Состояние:
button.exists          // существует ли
button.isEnabled       // доступна ли
button.isHittable      // можно ли нажать
button.value           // значение (для switch/slider)
button.label           // текст
```

**Как симулировать user interactions?**
```swift
// Касания:
element.tap()
element.doubleTap()
element.twoFingerTap()
element.press(forDuration: 2.0)     // долгое нажатие
element.tap(withNumberOfTaps: 3, numberOfTouches: 1)

// Текст:
textField.typeText("Hello World")
textField.clearAndEnterText("New text")  // кастомный метод
app.keys["delete"].tap()                 // backspace

// Жесты:
element.swipeLeft()
element.swipeRight()
element.swipeUp()
element.swipeDown()
element.pinch(withScale: 2, velocity: 1)   // zoom in
element.rotate(CGFloat.pi / 2, withVelocity: 1)  // поворот

// Скролл до элемента:
element.scroll(byDeltaX: 0, deltaY: 200)

// Координаты:
let coord = element.coordinate(withNormalizedOffset: CGVector(dx: 0.5, dy: 0.5))
coord.tap()
```

**Что такое `waitForExistence`?**
```swift
// Ждать появления элемента (async операции, анимации):
let element = app.buttons["submitButton"]
XCTAssertTrue(element.waitForExistence(timeout: 5))

// Ждать исчезновения:
let spinner = app.activityIndicators.firstMatch
XCTAssertTrue(spinner.waitForExistence(timeout: 2))
// Дождаться пока пропадёт:
let predicate = NSPredicate(format: "exists == false")
expectation(for: predicate, evaluatedWith: spinner)
waitForExpectations(timeout: 5)

// Удобный extension:
extension XCUIElement {
    func waitForDisappearance(timeout: TimeInterval = 3) -> Bool {
        let predicate = NSPredicate(format: "exists == false")
        let exp = XCTNSPredicateExpectation(predicate: predicate, object: self)
        return XCTWaiter.wait(for: [exp], timeout: timeout) == .completed
    }
}
```

**Как stub сетевые запросы в UI тестах?**
```swift
// Способ 1: Launch Arguments + MockServer:
// В тесте:
app.launchArguments = ["--uitesting", "--mock-network"]

// В приложении:
#if DEBUG
if CommandLine.arguments.contains("--mock-network") {
    URLProtocol.registerClass(MockURLProtocol.self)
}
#endif

// Способ 2: Local server (Swifter/Embassy):
class APITests: XCTestCase {
    var server: HttpServer!
    
    override func setUp() {
        server = HttpServer()
        server["/users"] = { _ in .ok(.json(["id": 1, "name": "Alice"])) }
        try? server.start(8080)
        
        let app = XCUIApplication()
        app.launchEnvironment["BASE_URL"] = "http://localhost:8080"
        app.launch()
    }
    
    override func tearDown() { server.stop() }
}

// Способ 3: Environment variable:
app.launchEnvironment["USE_MOCK_DATA"] = "true"
// В коде:
if ProcessInfo.processInfo.environment["USE_MOCK_DATA"] == "true" {
    return mockUsers
}
```

**Что такое Page Object Model в UI тестах?**
```swift
// Инкапсулировать взаимодействие с каждым экраном в отдельный класс:

class LoginScreen {
    private let app: XCUIApplication
    
    init(app: XCUIApplication) { self.app = app }
    
    // Elements:
    var emailField: XCUIElement { app.textFields["emailField"] }
    var passwordField: XCUIElement { app.secureTextFields["passwordField"] }
    var loginButton: XCUIElement { app.buttons["loginButton"] }
    var errorMessage: XCUIElement { app.staticTexts["errorMessage"] }
    
    // Actions:
    @discardableResult
    func login(email: String, password: String) -> HomeScreen {
        emailField.tap()
        emailField.typeText(email)
        passwordField.tap()
        passwordField.typeText(password)
        loginButton.tap()
        return HomeScreen(app: app)
    }
    
    func verifyErrorMessage(_ text: String) {
        XCTAssertTrue(errorMessage.waitForExistence(timeout: 2))
        XCTAssertEqual(errorMessage.label, text)
    }
}

class HomeScreen {
    private let app: XCUIApplication
    init(app: XCUIApplication) { self.app = app }
    var isDisplayed: Bool { app.navigationBars["Home"].exists }
}

// Тест читается как user story:
func test_validLogin_showsHome() {
    let login = LoginScreen(app: app)
    let home = login.login(email: "user@test.com", password: "password123")
    XCTAssertTrue(home.isDisplayed)
}
```

---

### Senior

**Как оптимизировать время UI тестов?**
```
1. Параллельный запуск (Xcode parallel testing)
2. Отключить анимации:
   app.launchArguments += ["-UIAnimationEnabled", "false"]

3. Использовать launch arguments вместо UI для настройки состояния:
   // Вместо: логиниться через UI каждый раз
   app.launchArguments = ["--logged-in", "--skip-onboarding"]

4. Разбить на группы: smoke (5 мин) / full (30 мин)
5. Запускать только smoke на каждом PR, full — ночью
6. Page Object: переиспользование кода
7. Убрать waitForExistence с большими timeout — использовать предиктивные паузы
8. Один симулятор — один тест (нет shared state)
```

**Как параллельно запускать UI тесты?**
```
Xcode: Edit Scheme → Test → Info → Options
  ✅ Execute in parallel on Simulator

Командная строка:
xcodebuild test \
  -scheme MyApp \
  -destination 'platform=iOS Simulator,name=iPhone 15' \
  -parallel-testing-enabled YES \
  -maximum-parallel-testing-workers 4

Важно:
  • Тесты должны быть независимы (нет shared state)
  • Каждый тест — чистый запуск app.launch()
  • Нет файлов/UserDefaults без очистки
  • Уникальные test data для каждого теста
```

**Как использовать launch arguments для конфигурации?**
```swift
// В тесте:
app.launchArguments += ["--uitesting"]
app.launchArguments += ["--reset-state"]
app.launchArguments += ["--logged-in"]
app.launchEnvironment["USER_ID"] = "123"
app.launchEnvironment["BASE_URL"] = "http://localhost:8080"

// В приложении (AppDelegate или @main):
struct MyApp: App {
    init() {
        setupForUITesting()
    }
    
    func setupForUITesting() {
        let args = ProcessInfo.processInfo.arguments
        let env = ProcessInfo.processInfo.environment
        
        if args.contains("--reset-state") {
            UserDefaults.standard.removePersistentDomain(forName: Bundle.main.bundleIdentifier!)
        }
        if args.contains("--logged-in") {
            AuthService.shared.setMockToken("test-token")
        }
        if let baseURL = env["BASE_URL"] {
            APIConfig.baseURL = baseURL
        }
    }
}
```

**Что такое screenshot testing?**
```swift
// Встроенный в XCTest:
func test_loginScreen_appearance() {
    let app = XCUIApplication()
    app.launch()
    
    // Сделать скриншот:
    let screenshot = app.screenshot()
    let attachment = XCTAttachment(screenshot: screenshot)
    attachment.name = "Login Screen"
    attachment.lifetime = .keepAlways
    add(attachment)  // сохраняется в результаты теста
}

// С использованием swift-snapshot-testing (UI Test версия):
func test_homeScreen_matchesSnapshot() {
    let app = XCUIApplication()
    app.launch()
    
    // Перейти на нужный экран...
    
    assertSnapshot(
        matching: app.screenshot().image,
        as: .image(precision: 0.99)  // 99% pixel совпадение
    )
}
```

---

## Отладка (Debugging)

### Breakpoints — типы и настройка

**Что такое breakpoint и как добавить?**
> Точка остановки выполнения программы для инспекции состояния. Клик на номер строки в Xcode — красная точка. При достижении строки — выполнение останавливается, можно смотреть переменные, вызывать LLDB команды.

**Типы breakpoints в Xcode:**
```
1. Line Breakpoint — стандартный, по строке кода
   Добавить: клик на номер строки

2. Conditional Breakpoint — остановка только при условии
   ПКМ на breakpoint → Edit → Condition: "count > 10"
   
3. Exception Breakpoint — останавливается на брошенных исключениях
   Breakpoint Navigator (+) → Exception Breakpoint
   Ловит: Objective-C exceptions, Swift errors
   
4. Symbolic Breakpoint — по имени функции/метода
   (+) → Symbolic → Symbol: "-[UIView layoutSubviews]"
   
5. Runtime Issue Breakpoint — ловит предупреждения runtime
   (+) → Runtime Issue
   
6. Swift Error Breakpoint — останавливается когда Swift бросает Error
   (+) → Swift Error
   
7. Watchpoint — при изменении значения переменной в памяти
   В Variables View: ПКМ на переменную → Watch
```

**Как настроить actions для breakpoint?**
```
ПКМ → Edit Breakpoint:
  Condition:  "userId == 42"    — останавливать только при условии
  Ignore:     10 first times    — пропустить первые N раз
  Action:     Debugger Command  — выполнить LLDB команду
              Log Message       — напечатать в консоль без остановки
              Shell Command     — выполнить скрипт
              Sound             — воспроизвести звук
  
  ✅ Automatically continue — не останавливать, только выполнить action
  (как logging без printf)
```

---

### LLDB — основные команды

**Навигация по стеку и выполнению:**
```bash
# Продолжить выполнение:
continue  # или: c

# Следующая строка (не входить в функцию):
next      # или: n

# Войти в функцию:
step      # или: s

# Выйти из текущей функции:
finish    # или: f

# Запустить до определённой строки:
thread until 42

# Стек вызовов:
bt        # backtrace — полный стек
bt 5      # только 5 frames
frame select 2  # перейти к frame 2

# Список потоков:
thread list
thread select 2  # переключиться на поток 2
```

**Просмотр и изменение переменных:**
```bash
# Напечатать значение:
p variable            # print — быстро
p self.count
p array[0]

# Pretty print (использует LLDB formatters):
po variable           # print object — для объектов

# Подробный вывод:
frame variable        # все локальные переменные
frame variable name   # конкретная

# Изменить значение во время отладки:
expression count = 42
e count = 42          # короткая форма
expression self.isEnabled = true

# Вызвать метод:
expression self.reload()
e print(self.description)

# Swift-специфичное:
expression import UIKit
e let $vc = unsafeBitCast(0x7f8a1234, to: UIViewController.self)
```

**Работа с памятью:**
```bash
# Адрес объекта:
p unsafeBitCast(myObject, to: Int.self)  # или просто po myObject увидишь адрес

# Прочитать память:
memory read 0x7f8a1234
memory read --size 4 --format x 0x7f8a1234  # hex, 4 байта

# Найти объект по адресу:
e let $obj = unsafeBitCast(0x7f8a1234, to: AnyObject.self)
po $obj

# Heap info:
heap -t NSObject     # все NSObject на heap (macOS)
```

**Полезные LLDB команды:**
```bash
# Список всех breakpoints:
breakpoint list

# Удалить breakpoint:
breakpoint delete 1

# Отключить/включить:
breakpoint disable 1
breakpoint enable 1

# Watchpoint:
watchpoint set variable self.count
watchpoint set expression -w write -- &self.count

# Искать символ:
image lookup -n viewDidLoad
image lookup -rn "UIView.*layout"

# Список всех модулей:
image list

# Тип переменной:
frame variable -T myVar

# Alias для частых команд:
command alias reload expression self.collectionView.reloadData()
```

**Работа с View Hierarchy из LLDB:**
```bash
# Напечатать view hierarchy:
po self.view.recursiveDescription()

# Найти view по адресу:
e let $view = unsafeBitCast(0x7f8a1234, to: UIView.self)
po $view

# Изменить цвет view без перекомпиляции:
e self.view.backgroundColor = UIColor.red

# Forcerender UI:
expression CATransaction.flush()

# Показать frame всех subviews:
po UIApplication.shared.keyWindow?.recursiveDescription()
```

---

### Инструменты отладки Xcode

**Debug Navigator:**
```
CPU — загрузка процессора в реальном времени
Memory — использование памяти (важно: нет утечек!)
Energy — энергопотребление
Disk — I/O операции
Network — сетевая активность
FPS — Frames Per Second (60 или 120 — норма)

Как читать:
  Memory растёт постоянно → утечка памяти
  CPU spike → тяжёлая операция на main thread
  FPS drops → UI bottleneck
```

**Memory Graph Debugger:**
```
Debug → Memory Graph (кнопка в Debug Bar)

Показывает:
  • Все объекты в памяти
  • Graф ссылок между объектами
  • Retain cycles (помечены "!")
  
Как искать утечку:
  1. Запустить Memory Graph
  2. Найти объект которого не должно быть
  3. Посмотреть кто держит ссылку
  4. Найти цикл
```

**View Hierarchy Debugger:**
```
Debug → View Hierarchy (кнопка в Debug Bar)
Или: Debug → View Debugging → Capture View Hierarchy

Возможности:
  • 3D разбивка слоёв
  • Просмотр constraints каждого view
  • Автоматическое определение ambiguous layout
  • Показывает hidden views
  
Горячие клавиши в View Debugger:
  Scroll: масштаб 3D
  Option + перетащить: вращение
  Show/Hide clipped content
```

**Console — полезные трюки:**
```swift
// Логирование без printf:
// Breakpoint action → Log Message:
// "User count: @count@" — вставляет значение переменной

// OSLog (предпочтительно для production):
import OSLog
let logger = Logger(subsystem: "com.myapp", category: "network")

logger.debug("Request started: \(url)")
logger.info("User logged in: \(userId)")
logger.warning("Slow response: \(duration)s")
logger.error("Failed to parse: \(error)")
logger.critical("Database corrupted!")

// Фильтрация в Console.app:
// subsystem:com.myapp category:network
```

**Thread Sanitizer (TSan):**
```
Включить: Edit Scheme → Run → Diagnostics → Thread Sanitizer

Обнаруживает:
  • Data races (concurrent read+write без синхронизации)
  • Использование освобождённой памяти из другого потока
  
Как читать output:
  WARNING: ThreadSanitizer: Swift access race
  Read of size 8 at 0x... by thread 2:
  Previous write at 0x... by thread 1:
  
Overhead: ~5-15x замедление — только в Debug
```

**Address Sanitizer (ASan):**
```
Включить: Edit Scheme → Run → Diagnostics → Address Sanitizer

Обнаруживает:
  • Use after free
  • Heap buffer overflow
  • Stack buffer overflow
  • Use of uninitialised memory
  
Для Swift: реже нужен (ARC), но для C/C++ bridging — незаменим
Overhead: ~2x замедление памяти
```

**Undefined Behavior Sanitizer (UBSan):**
```
Edit Scheme → Diagnostics → Undefined Behavior Sanitizer

Обнаруживает:
  • Integer overflow
  • Null pointer dereference
  • Invalid type casts
  • Misaligned pointers
```

---

### Instruments — основные шаблоны

**Time Profiler:**
```
Когда использовать: приложение тормозит, лагает scroll, медленный запуск

Как читать:
  • Flame Graph: прямоугольник = функция, ширина = время
  • Найди самые широкие блоки на main thread
  • "Hide System Libraries" — показать только твой код
  
Горячие клавиши:
  Command+2: Call Tree
  Command+3: Flame Graph
  Option+Command+W: Filter by heaviest stack
  
Типичные проблемы:
  layoutSubviews занимает > 1ms
  JSON decode на main thread
  Core Data fetch на main thread
  Image decode на main thread
```

**Allocations:**
```
Когда: растёт память, OOM crashes

Режимы:
  All Allocations — все объекты в памяти
  Live Allocations — только живые объекты
  
Heap Shot — снимок состояния кучи:
  1. Mark Generation (кнопка)
  2. Выполни действие подозрительное
  3. Mark Generation снова
  4. Сравни: что появилось и не исчезло?
  
Backtrace: где был создан объект
Filter: имя класса для поиска
```

**Leaks:**
```
Когда: подозрение на memory leak

Автоматически находит:
  • Retain cycles
  • Объекты недостижимые из root (но retain count > 0)
  
Leaks Checker запускается автоматически каждые 10 секунд
Красная X в timeline — обнаружена утечка

Graph view: визуальный граф retain cycle
  Найди объект с "leak" иконкой
  Проследи ссылки — найди цикл
```

**Network:**
```
Когда: медленные запросы, много requests

Показывает:
  • Все HTTP/HTTPS запросы
  • Duration, size, status code
  • Request/Response headers и body
  
Фильтр по домену: filter bar
Сортировка по duration: найди медленные
```

**Core Data:**
```
Когда: медленные fetch, долгие сохранения

Показывает:
  • SQL запросы
  • Fetch duration
  • Fault counts (N+1 problem!)
  
N+1 проблема:
  Fetch 100 users, потом для каждого fetch avatar
  → 101 запрос вместо 1 с JOIN
  Решение: prefetchRelationshipKeyPaths
```

---

### Полезные консольные команды Xcode

**В Debug Console (при остановке на breakpoint):**
```bash
# Текущий view controller:
po UIApplication.shared.keyWindow?.rootViewController

# Navigation stack:
po (UIApplication.shared.keyWindow?.rootViewController as? UINavigationController)?.viewControllers

# UserDefaults:
po UserDefaults.standard.dictionaryRepresentation()

# Все subviews:
po self.view.subviews

# Frame view в window координатах:
po self.button.convert(self.button.bounds, to: nil)

# Вызвать метод и увидеть результат:
expression -l Swift -- import UIKit
expression -l Swift -- UIColor.red.cgColor

# Добавить subview без перезапуска:
expression self.view.addSubview(UIView(frame: CGRect(x: 0, y: 0, width: 100, height: 100)))
expression CATransaction.flush()  // форс рендер
```

**Launch Arguments полезные:**
```bash
# Показать все Core Data SQL:
-com.apple.CoreData.SQLDebug 1

# Verbose Core Data:
-com.apple.CoreData.Logging.stderr 1

# UIAutomation logging:
-UIViewLayoutFeedbackLoopDebuggingThreshold 100

# Auto Layout conflicts verbose:
-NSConstraintBasedLayoutVisualizeMutuallyExclusiveConstraints YES

# Disable animations (для тестов):
-UIAnimationEnabled NO

# Slow animations (для debugging):
# Hardware → Slow Animations в симуляторе (⌘T)
```

---

### Шпаргалка: Где искать проблему

```
Приложение тормозит:
  → Time Profiler → main thread bottleneck

Растёт память:
  → Memory Graph Debugger → retain cycle
  → Instruments Allocations → Heap Shot

Приложение крашится:
  → Crash Report в Xcode (Window → Devices)
  → Exception Breakpoint → смотреть стек
  → lldb: bt (backtrace при краше)

Data race:
  → Thread Sanitizer включить
  → Run → Diagnostics

Медленный UI:
  → Debug → View Debugging
  → Core Animation instrument
  → Slow Animations в симуляторе

Сетевые проблемы:
  → Network instrument
  → Charles Proxy (MITM SSL)
  → po URLSession.shared.configuration

Не работает constraint:
  → View Hierarchy Debugger
  → Breakpoint на UIViewAlertForUnsatisfiableConstraints
  → po view.hasAmbiguousLayout()

Не вызывается метод:
  → Symbolic Breakpoint на имя метода
  → lldb: image lookup -n methodName
```

---

*Конец документа · Testing & Debugging*
