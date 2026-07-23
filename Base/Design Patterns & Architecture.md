# Design Patterns & Architecture — Вопросы и ответы
> iOS-собеседование · Design Patterns · Architecture · SOLID · DI
> Уровни: Beginner · Middle · Senior · Staff

---

## Creational Patterns (Порождающие)

### Beginner

**Что такое Singleton?**
> Паттерн гарантирующий что класс имеет только один экземпляр и предоставляет глобальную точку доступа к нему. Примеры в iOS: `URLSession.shared`, `UserDefaults.standard`, `NotificationCenter.default`, `FileManager.default`.

**Как реализовать Singleton в Swift?**
```swift
// Базовый Singleton:
final class NetworkManager {
    static let shared = NetworkManager()  // ленивая инициализация, thread-safe
    private init() { }  // запрет создания снаружи
    
    func fetch(url: URL) { }
}
NetworkManager.shared.fetch(url: url)

// Thread-safe без static let (старый способ через dispatch_once):
// static let уже thread-safe в Swift — инициализируется один раз через dispatch_once

// Singleton с конфигурацией:
final class AppConfig {
    static let shared = AppConfig()
    private init() { }
    
    var baseURL = URL(string: "https://api.example.com")!
    var timeout: TimeInterval = 30
}
```

**Что такое Factory Method?**
> Определяет интерфейс для создания объекта, но позволяет подклассам решать какой класс инстанцировать. Фабричный метод делегирует создание объектов подклассам.
```swift
// Протокол с фабричным методом:
protocol Button {
    func render()
    func onClick()
}

class IOSButton: Button {
    func render() { print("Render iOS button") }
    func onClick() { print("iOS tap") }
}

class AndroidButton: Button {
    func render() { print("Render Android button") }
    func onClick() { print("Android click") }
}

// Factory Method:
protocol Dialog {
    func createButton() -> Button  // фабричный метод
    func render() {
        let button = createButton()
        button.render()
    }
}

class IOSDialog: Dialog {
    func createButton() -> Button { IOSButton() }
}
```

**Что такое Abstract Factory?**
> Создаёт семейства связанных объектов не указывая конкретных классов. Как фабрика фабрик.
```swift
// Абстрактная фабрика UI компонентов:
protocol UIFactory {
    func createButton() -> Button
    func createTextField() -> TextField
    func createAlert() -> Alert
}

class LightThemeFactory: UIFactory {
    func createButton() -> Button { LightButton() }
    func createTextField() -> TextField { LightTextField() }
    func createAlert() -> Alert { LightAlert() }
}

class DarkThemeFactory: UIFactory {
    func createButton() -> Button { DarkButton() }
    func createTextField() -> TextField { DarkTextField() }
    func createAlert() -> Alert { DarkAlert() }
}

// Использование:
class App {
    private let factory: UIFactory
    init(factory: UIFactory) { self.factory = factory }
    
    func buildUI() {
        let button = factory.createButton()
        let field = factory.createTextField()
        // всё создано в одном стиле
    }
}
```

**Что такое Builder паттерн?**
> Разделяет конструирование сложного объекта от его представления. Позволяет создавать разные представления одного и того же объекта.
```swift
// Builder для сложного URLRequest:
struct URLRequestBuilder {
    private var url: URL
    private var method = "GET"
    private var headers: [String: String] = [:]
    private var body: Data?
    private var timeout: TimeInterval = 30
    
    init(url: URL) { self.url = url }
    
    func method(_ method: String) -> Self {
        var copy = self; copy.method = method; return copy
    }
    func header(_ key: String, _ value: String) -> Self {
        var copy = self; copy.headers[key] = value; return copy
    }
    func body(_ data: Data) -> Self {
        var copy = self; copy.body = data; return copy
    }
    func timeout(_ timeout: TimeInterval) -> Self {
        var copy = self; copy.timeout = timeout; return copy
    }
    
    func build() -> URLRequest {
        var request = URLRequest(url: url, timeoutInterval: timeout)
        request.httpMethod = method
        request.httpBody = body
        headers.forEach { request.setValue($1, forHTTPHeaderField: $0) }
        return request
    }
}

// Использование (fluent interface):
let request = URLRequestBuilder(url: URL(string: "https://api.com/users")!)
    .method("POST")
    .header("Content-Type", "application/json")
    .header("Authorization", "Bearer token")
    .body(jsonData)
    .timeout(60)
    .build()
```

---

### Middle

**Чем Singleton вреден для тестирования?**
```swift
// Проблема — глобальное состояние:
class UserManager {
    static let shared = UserManager()
    var currentUser: User?
}

class ViewModel {
    func loadData() {
        // Нельзя подменить в тестах!
        let user = UserManager.shared.currentUser
    }
}

// Проблемы:
// 1. Нельзя подменить на Mock в тестах
// 2. Тесты влияют друг на друга (shared state)
// 3. Порядок тестов важен (flaky tests)
// 4. Нельзя запустить параллельно
// 5. Скрытая зависимость — не видна в сигнатуре

// Решение: DI
class ViewModel {
    private let userManager: UserManagerProtocol
    init(userManager: UserManagerProtocol = UserManager.shared) {
        self.userManager = userManager
    }
}
```

**Как заменить Singleton на DI?**
```swift
// 1. Создать протокол:
protocol UserManagerProtocol {
    var currentUser: User? { get }
    func login(credentials: Credentials) async throws -> User
    func logout()
}

// 2. Singleton реализует протокол:
final class UserManager: UserManagerProtocol {
    static let shared = UserManager()
    private init() { }
    var currentUser: User?
    // ...
}

// 3. Передавать через DI:
class ProfileViewModel {
    private let userManager: UserManagerProtocol
    
    init(userManager: UserManagerProtocol) {
        self.userManager = userManager
    }
}

// Production: ProfileViewModel(userManager: UserManager.shared)
// Tests: ProfileViewModel(userManager: MockUserManager())
```

**Что такое Factory для создания View Controllers?**
```swift
// Factory скрывает детали создания VC:
protocol ViewControllerFactory {
    func makeProfileVC(userId: Int) -> ProfileViewController
    func makeSettingsVC() -> SettingsViewController
    func makeLoginVC() -> LoginViewController
}

class AppViewControllerFactory: ViewControllerFactory {
    private let dependencies: AppDependencies
    
    init(dependencies: AppDependencies) {
        self.dependencies = dependencies
    }
    
    func makeProfileVC(userId: Int) -> ProfileViewController {
        let vm = ProfileViewModel(
            userId: userId,
            userService: dependencies.userService,
            imageLoader: dependencies.imageLoader
        )
        return ProfileViewController(viewModel: vm)
    }
    
    func makeSettingsVC() -> SettingsViewController {
        SettingsViewController(
            settingsService: dependencies.settingsService,
            analyticsService: dependencies.analyticsService
        )
    }
    
    func makeLoginVC() -> LoginViewController {
        let vm = LoginViewModel(authService: dependencies.authService)
        return LoginViewController(viewModel: vm)
    }
}

// Coordinator использует factory:
class AppCoordinator {
    private let factory: ViewControllerFactory
    
    func showProfile(userId: Int) {
        let vc = factory.makeProfileVC(userId: userId)
        navigationController.push(vc, animated: true)
    }
}
```

**Как реализовать Builder для сложных объектов?**
```swift
// Шаг-за-шагом Builder (как в тестах):
class AlertBuilder {
    private var title: String = ""
    private var message: String?
    private var actions: [UIAlertAction] = []
    private var style: UIAlertController.Style = .alert
    
    func title(_ title: String) -> Self {
        self.title = title
        return self
    }
    func message(_ message: String) -> Self {
        self.message = message
        return self
    }
    func action(title: String, style: UIAlertAction.Style = .default, handler: (() -> Void)? = nil) -> Self {
        let action = UIAlertAction(title: title, style: style) { _ in handler?() }
        actions.append(action)
        return self
    }
    func destructive(title: String, handler: @escaping () -> Void) -> Self {
        action(title: title, style: .destructive, handler: handler)
    }
    func cancel(title: String = "Отмена") -> Self {
        action(title: title, style: .cancel)
    }
    
    func build() -> UIAlertController {
        let alert = UIAlertController(title: title, message: message, preferredStyle: style)
        actions.forEach { alert.addAction($0) }
        return alert
    }
}

// Использование:
let alert = AlertBuilder()
    .title("Удалить аккаунт?")
    .message("Это действие необратимо")
    .destructive(title: "Удалить") { self.deleteAccount() }
    .cancel()
    .build()
present(alert, animated: true)
```

**Что такое Prototype паттерн?**
> Создание новых объектов клонированием существующих вместо создания через конструктор.
```swift
// В Swift — через NSCopying или кастомный clone():
protocol Cloneable {
    func clone() -> Self
}

class UserProfile: Cloneable {
    var name: String
    var age: Int
    var avatar: UIImage?
    
    init(name: String, age: Int) {
        self.name = name
        self.age = age
    }
    
    func clone() -> UserProfile {
        let copy = UserProfile(name: name, age: age)
        copy.avatar = avatar  // shallow copy — делим UIImage
        return copy
    }
}

// Struct в Swift — встроенный Prototype:
struct Config {
    var baseURL: String
    var timeout: TimeInterval
    var headers: [String: String]
}

var defaultConfig = Config(baseURL: "https://api.com", timeout: 30, headers: [:])
var testConfig = defaultConfig  // копия!
testConfig.baseURL = "https://test-api.com"
// defaultConfig.baseURL — "https://api.com" — не изменился
```

**Как реализовать кастомный init factory?**
```swift
// Static factory methods:
extension URLRequest {
    static func jsonGET(url: URL, token: String) -> URLRequest {
        var request = URLRequest(url: url)
        request.httpMethod = "GET"
        request.setValue("application/json", forHTTPHeaderField: "Accept")
        request.setValue("Bearer \(token)", forHTTPHeaderField: "Authorization")
        return request
    }
    
    static func jsonPOST<T: Encodable>(url: URL, body: T, token: String) throws -> URLRequest {
        var request = URLRequest(url: url)
        request.httpMethod = "POST"
        request.setValue("application/json", forHTTPHeaderField: "Content-Type")
        request.setValue("Bearer \(token)", forHTTPHeaderField: "Authorization")
        request.httpBody = try JSONEncoder().encode(body)
        return request
    }
}

// Использование:
let request = URLRequest.jsonGET(url: usersURL, token: authToken)
let postRequest = try URLRequest.jsonPOST(url: createURL, body: newUser, token: authToken)
```

---

### Senior

**Как реализовать type-safe builder в Swift?**
```swift
// Используя phantom types — compile-time проверка:
struct BuilderState {}
struct WithTitle: BuilderState {}
struct WithMessage: BuilderState {}
struct Ready: BuilderState {}

struct NotificationBuilder<State> {
    private var title: String = ""
    private var message: String = ""
    private var sound: UNNotificationSound = .default
    
    // Доступен только когда State == BuilderState (начальное):
    func title(_ title: String) -> NotificationBuilder<WithTitle> {
        var builder = NotificationBuilder<WithTitle>()
        builder.title = title
        return builder
    }
}

extension NotificationBuilder where State == WithTitle {
    // Доступен только после установки title:
    func message(_ message: String) -> NotificationBuilder<Ready> {
        var builder = NotificationBuilder<Ready>()
        builder.title = self.title
        builder.message = message
        return builder
    }
}

extension NotificationBuilder where State == Ready {
    // build доступен только когда State == Ready:
    func build() -> UNNotificationContent {
        let content = UNMutableNotificationContent()
        content.title = title
        content.body = message
        content.sound = sound
        return content
    }
}

// Компилятор не позволит вызвать build() без title и message:
let content = NotificationBuilder<BuilderState>()
    .title("Новое сообщение")    // → NotificationBuilder<WithTitle>
    .message("У вас 3 уведомления")  // → NotificationBuilder<Ready>
    .build()                      // ✅ доступен

// let broken = NotificationBuilder<BuilderState>().build()  // ❌ ошибка компиляции
```

**Как Factory паттерн взаимодействует с DI?**
```swift
// DI Container как Abstract Factory:
final class DIContainer {
    private var registrations: [ObjectIdentifier: () -> Any] = [:]
    
    func register<T>(_ type: T.Type, factory: @escaping () -> T) {
        registrations[ObjectIdentifier(type)] = factory
    }
    
    func resolve<T>(_ type: T.Type) -> T {
        guard let factory = registrations[ObjectIdentifier(type)] as? () -> T else {
            fatalError("No registration for \(type)")
        }
        return factory()
    }
}

// Регистрация зависимостей:
let container = DIContainer()
container.register(NetworkClient.self) { URLSessionNetworkClient() }
container.register(UserRepository.self) {
    RemoteUserRepository(client: container.resolve(NetworkClient.self))
}
container.register(UserViewModel.self) {
    UserViewModel(repository: container.resolve(UserRepository.self))
}

// Factory + DI = создание VC с зависимостями:
class ScreenFactory {
    private let container: DIContainer
    init(container: DIContainer) { self.container = container }
    
    func makeUserList() -> UserListViewController {
        let vm = container.resolve(UserViewModel.self)
        return UserListViewController(viewModel: vm)
    }
}
```

**Что такое Object Pool паттерн и где применяется в iOS?**
```swift
// Переиспользование дорогих объектов вместо создания новых:
class ObjectPool<T> {
    private var available: [T] = []
    private var inUse: [T] = []
    private let factory: () -> T
    private let reset: (T) -> Void
    
    init(initialSize: Int, factory: @escaping () -> T, reset: @escaping (T) -> Void) {
        self.factory = factory
        self.reset = reset
        available = (0..<initialSize).map { _ in factory() }
    }
    
    func acquire() -> T {
        let obj = available.isEmpty ? factory() : available.removeLast()
        inUse.append(obj)
        return obj
    }
    
    func release(_ obj: T) {
        inUse.removeAll { $0 as AnyObject === obj as AnyObject }
        reset(obj)
        available.append(obj)
    }
}

// iOS использует Object Pool в:
// 1. UITableView/UICollectionView — cell reuse pool
//    dequeueReusableCell — берёт из пула
//    prepareForReuse — сброс состояния
//
// 2. CALayer backing stores — переиспользование буферов
// 3. Metal: MTLCommandBuffer pools
// 4. Thread pools в GCD

// Кастомный пул для WebSocket соединений:
let connectionPool = ObjectPool(
    initialSize: 3,
    factory: { WebSocketConnection() },
    reset: { $0.clearCallbacks() }
)
let conn = connectionPool.acquire()
defer { connectionPool.release(conn) }
```

---

## Behavioral Patterns (Поведенческие)

### Beginner

**Что такое Observer паттерн?**
> Определяет зависимость один-ко-многим: при изменении состояния одного объекта все зависимые автоматически уведомляются.
```swift
// В iOS реализован через:
// 1. NotificationCenter
// 2. KVO (Key-Value Observing)
// 3. Delegate
// 4. Combine (@Published, PassthroughSubject)
// 5. Swift Observable (@Observable)

// Кастомная реализация:
protocol Observer: AnyObject {
    func update(event: String, data: Any?)
}

class EventEmitter {
    private var observers: [String: [WeakRef<Observer>]] = [:]
    
    func subscribe(_ observer: Observer, to event: String) {
        observers[event, default: []].append(WeakRef(observer))
    }
    func emit(event: String, data: Any? = nil) {
        observers[event]?.forEach { $0.value?.update(event: event, data: data) }
    }
}
```

**Что такое Delegate паттерн?**
> Объект делегирует часть функциональности другому объекту. Loose coupling: делегирующий знает только протокол, не конкретную реализацию.
```swift
// Протокол делегата:
protocol DownloadManagerDelegate: AnyObject {
    func downloadDidStart(_ manager: DownloadManager)
    func downloadDidProgress(_ manager: DownloadManager, progress: Double)
    func downloadDidComplete(_ manager: DownloadManager, url: URL)
    func downloadDidFail(_ manager: DownloadManager, error: Error)
}

class DownloadManager {
    weak var delegate: DownloadManagerDelegate?  // weak — избегаем retain cycle
    
    func startDownload(from url: URL) {
        delegate?.downloadDidStart(self)
        // ... загрузка ...
        delegate?.downloadDidProgress(self, progress: 0.5)
        delegate?.downloadDidComplete(self, url: localURL)
    }
}

// Реализация:
class ViewController: DownloadManagerDelegate {
    let manager = DownloadManager()
    
    override func viewDidLoad() {
        manager.delegate = self
        manager.startDownload(from: url)
    }
    
    func downloadDidComplete(_ manager: DownloadManager, url: URL) {
        // обновить UI
    }
}
```

**Что такое Strategy паттерн?**
> Определяет семейство алгоритмов, инкапсулирует каждый и делает взаимозаменяемыми.
```swift
// Стратегии сортировки:
protocol SortStrategy {
    func sort<T: Comparable>(_ array: [T]) -> [T]
}

class BubbleSort: SortStrategy {
    func sort<T: Comparable>(_ array: [T]) -> [T] { /* bubble sort */ array }
}

class QuickSort: SortStrategy {
    func sort<T: Comparable>(_ array: [T]) -> [T] { array.sorted() }
}

class DataProcessor {
    private var sortStrategy: SortStrategy
    
    init(strategy: SortStrategy = QuickSort()) {
        sortStrategy = strategy
    }
    
    func setStrategy(_ strategy: SortStrategy) {
        sortStrategy = strategy
    }
    
    func process<T: Comparable>(_ data: [T]) -> [T] {
        sortStrategy.sort(data)
    }
}

// В Swift через замыкания (Strategy без класса):
class Sorter {
    var strategy: ([Int]) -> [Int] = { $0.sorted() }
    func sort(_ array: [Int]) -> [Int] { strategy(array) }
}
let sorter = Sorter()
sorter.strategy = { $0.sorted(by: >) }  // изменить стратегию
```

**Что такое Command паттерн?**
> Инкапсулирует запрос как объект. Позволяет параметризовать клиентов с разными запросами, ставить в очередь, логировать, поддерживать отмену.
```swift
protocol Command {
    func execute()
    func undo()
}

class MoveCommand: Command {
    private let view: UIView
    private let offset: CGPoint
    private var previousPosition: CGPoint = .zero
    
    init(view: UIView, offset: CGPoint) {
        self.view = view
        self.offset = offset
    }
    
    func execute() {
        previousPosition = view.center
        view.center = CGPoint(x: view.center.x + offset.x,
                              y: view.center.y + offset.y)
    }
    
    func undo() {
        view.center = previousPosition
    }
}

class CommandHistory {
    private var history: [Command] = []
    
    func execute(_ command: Command) {
        command.execute()
        history.append(command)
    }
    
    func undo() {
        history.popLast()?.undo()
    }
    
    func redo() { /* если нужно */ }
}
```

---

### Middle

**Как Delegate реализован в UIKit?**
> UIKit широко использует Delegate для кастомизации поведения без subclassing:
```swift
// UITableView использует два протокола:
// UITableViewDataSource — данные (что показывать)
// UITableViewDelegate  — поведение (как реагировать)

class MyViewController: UIViewController,
    UITableViewDataSource,
    UITableViewDelegate {
    
    // DataSource — обязательные:
    func tableView(_ tv: UITableView, numberOfRowsInSection section: Int) -> Int { data.count }
    func tableView(_ tv: UITableView, cellForRowAt indexPath: IndexPath) -> UITableViewCell {
        let cell = tv.dequeueReusableCell(withIdentifier: "Cell", for: indexPath)
        cell.textLabel?.text = data[indexPath.row]
        return cell
    }
    
    // Delegate — опциональные:
    func tableView(_ tv: UITableView, didSelectRowAt indexPath: IndexPath) {
        showDetail(for: data[indexPath.row])
    }
    func tableView(_ tv: UITableView, heightForRowAt indexPath: IndexPath) -> CGFloat { 60 }
}

// UITextField, UIScrollView, URLSession — все через Delegate
// weak var delegate — всегда weak чтобы избежать retain cycle
```

**Чем Delegate отличается от Notification?**
| | Delegate | NotificationCenter |
|--|--|--|
| Связь | Один к одному | Один ко многим |
| Типизация | Строгая (протокол) | Слабая (userInfo: Any?) |
| Coupling | Делегирующий знает о delegate | Полное разделение |
| Ответ | Может возвращать значение | Нет возврата |
| Использование | UITableView, UITextField | Keyboard, AppState |

```swift
// Delegate — когда нужен ответ от одного:
protocol ImagePickerDelegate: AnyObject {
    func didPickImage(_ image: UIImage)  // ответ
}

// Notification — когда много слушателей:
NotificationCenter.default.post(name: .userDidLogin, object: nil)
// Любой объект в приложении может подписаться
```

**Что такое Chain of Responsibility?**
> Передаёт запрос по цепочке обработчиков. Каждый обработчик решает — обработать или передать дальше.
```swift
// Responder chain в UIKit — готовый пример:
// Touch event: UIView → superview → UIViewController → UIWindow → UIApplication

// Кастомная реализация:
protocol Handler: AnyObject {
    var next: Handler? { get set }
    func handle(request: Int) -> String?
}

class BaseHandler: Handler {
    var next: Handler?
    
    func handle(request: Int) -> String? {
        next?.handle(request: request)
    }
}

class MonkeyHandler: BaseHandler {
    override func handle(request: Int) -> String? {
        if request < 10 { return "Monkey handled \(request)" }
        return super.handle(request: request)
    }
}

class DogHandler: BaseHandler {
    override func handle(request: Int) -> String? {
        if request < 50 { return "Dog handled \(request)" }
        return super.handle(request: request)
    }
}

// Настройка цепочки:
let monkey = MonkeyHandler()
let dog = DogHandler()
monkey.next = dog

monkey.handle(request: 5)   // "Monkey handled 5"
monkey.handle(request: 30)  // "Dog handled 30"
monkey.handle(request: 100) // nil
```

**Что такое State машина?**
> Объект меняет поведение при изменении внутреннего состояния. Переходы между состояниями чётко определены.
```swift
// Состояния загрузки:
enum LoadingState {
    case idle
    case loading
    case success(data: [User])
    case failure(error: Error)
}

// Переходы:
// idle → loading (при запросе)
// loading → success (при ответе)
// loading → failure (при ошибке)
// success/failure → loading (при повторном запросе)
// success/failure → idle (при сбросе)
```

**Как реализовать State Machine в Swift?**
```swift
// Через enum с методами:
enum TrafficLight {
    case red, yellow, green
    
    var next: TrafficLight {
        switch self {
        case .red: return .green
        case .green: return .yellow
        case .yellow: return .red
        }
    }
    
    var duration: TimeInterval {
        switch self {
        case .red: return 60
        case .green: return 45
        case .yellow: return 5
        }
    }
}

// Через паттерн State (объекты):
protocol OrderState {
    func confirm(order: Order)
    func pay(order: Order)
    func ship(order: Order)
    func cancel(order: Order)
}

class Order {
    var state: OrderState = PendingState()
    
    func confirm() { state.confirm(order: self) }
    func pay() { state.pay(order: self) }
    func ship() { state.ship(order: self) }
    func cancel() { state.cancel(order: self) }
}

class PendingState: OrderState {
    func confirm(order: Order) {
        print("Order confirmed")
        order.state = ConfirmedState()
    }
    func pay(order: Order) { print("Cannot pay unconfirmed order") }
    func ship(order: Order) { print("Cannot ship unconfirmed order") }
    func cancel(order: Order) {
        print("Order cancelled")
        order.state = CancelledState()
    }
}

class ConfirmedState: OrderState {
    func confirm(order: Order) { print("Already confirmed") }
    func pay(order: Order) {
        print("Payment received")
        order.state = PaidState()
    }
    func ship(order: Order) { print("Pay first") }
    func cancel(order: Order) {
        print("Order cancelled")
        order.state = CancelledState()
    }
}
```

**Что такое Template Method?**
> Определяет скелет алгоритма в базовом классе, делегируя некоторые шаги подклассам.
```swift
// Базовый класс с шаблонным методом:
class DataImporter {
    // Template Method — финальный, не переопределяется:
    final func importData(from url: URL) {
        let rawData = readData(from: url)     // шаг 1 — переопределяемый
        let parsed = parseData(rawData)       // шаг 2 — переопределяемый
        validateData(parsed)                  // шаг 3 — опциональный hook
        saveData(parsed)                      // шаг 4 — общий
    }
    
    func readData(from url: URL) -> Data {
        try! Data(contentsOf: url)
    }
    
    func parseData(_ data: Data) -> [Record] {
        fatalError("Must override")  // abstract
    }
    
    func validateData(_ records: [Record]) { }  // hook — опциональный
    
    private func saveData(_ records: [Record]) {
        // общая логика сохранения
    }
}

class CSVImporter: DataImporter {
    override func parseData(_ data: Data) -> [Record] {
        // парсинг CSV
        return []
    }
}

class JSONImporter: DataImporter {
    override func parseData(_ data: Data) -> [Record] {
        // парсинг JSON
        return []
    }
    
    override func validateData(_ records: [Record]) {
        // дополнительная валидация для JSON
    }
}
```

**Что такое Mediator паттерн?**
> Определяет объект-посредник инкапсулирующий взаимодействие между объектами. Уменьшает связность — объекты не знают друг о друге.
```swift
// Mediator для компонентов формы:
protocol FormComponent: AnyObject {
    var mediator: FormMediator? { get set }
    func notify(event: String)
}

protocol FormMediator: AnyObject {
    func notify(sender: FormComponent, event: String)
}

class LoginForm: FormMediator {
    private weak var emailField: TextFieldComponent?
    private weak var passwordField: TextFieldComponent?
    private weak var loginButton: ButtonComponent?
    
    func notify(sender: FormComponent, event: String) {
        // Когда изменилось поле — проверить валидность кнопки:
        if event == "textChanged" {
            let isValid = !(emailField?.text.isEmpty ?? true) &&
                         (passwordField?.text.count ?? 0) >= 6
            loginButton?.setEnabled(isValid)
        }
        
        if event == "buttonTapped" {
            performLogin()
        }
    }
}

// В iOS: NSNotificationCenter, Coordinator паттерн = Mediator
```

**Что такое Visitor паттерн?**
> Позволяет добавлять новые операции к классам без их изменения, через отдельный объект-посетитель.
```swift
// Иерархия элементов формы:
protocol FormElement {
    func accept(visitor: FormVisitor)
}

struct TextInput: FormElement {
    let value: String
    func accept(visitor: FormVisitor) { visitor.visit(self) }
}

struct Checkbox: FormElement {
    let isChecked: Bool
    func accept(visitor: FormVisitor) { visitor.visit(self) }
}

// Visitor — добавляем операцию без изменения классов:
protocol FormVisitor {
    func visit(_ element: TextInput)
    func visit(_ element: Checkbox)
}

class ValidationVisitor: FormVisitor {
    var errors: [String] = []
    
    func visit(_ element: TextInput) {
        if element.value.isEmpty { errors.append("Field is empty") }
    }
    func visit(_ element: Checkbox) {
        if !element.isChecked { errors.append("Must accept terms") }
    }
}

class SerializationVisitor: FormVisitor {
    var data: [String: Any] = [:]
    
    func visit(_ element: TextInput) { data["text"] = element.value }
    func visit(_ element: Checkbox) { data["checked"] = element.isChecked }
}
```

**Что такое Iterator паттерн?**
> Предоставляет способ последовательного доступа к элементам без раскрытия внутреннего представления.
```swift
// Swift Sequence + IteratorProtocol = Iterator паттерн:
struct Fibonacci: Sequence, IteratorProtocol {
    private var a = 0, b = 1
    private let limit: Int
    private var count = 0
    
    init(limit: Int) { self.limit = limit }
    
    mutating func next() -> Int? {
        guard count < limit else { return nil }
        count += 1
        let result = a
        (a, b) = (b, a + b)
        return result
    }
}

// Использование через for-in:
for fib in Fibonacci(limit: 10) {
    print(fib)  // 0, 1, 1, 2, 3, 5, 8, 13, 21, 34
}

// Кастомный итератор для дерева (обход в ширину):
struct BFSIterator<T>: IteratorProtocol {
    private var queue: [TreeNode<T>]
    
    init(root: TreeNode<T>) { queue = [root] }
    
    mutating func next() -> T? {
        guard !queue.isEmpty else { return nil }
        let node = queue.removeFirst()
        queue.append(contentsOf: node.children)
        return node.value
    }
}
```

---

### Senior

**Как реализовать type-safe event bus?**
```swift
// Type-safe event bus через generics:
final class EventBus {
    private var handlers: [ObjectIdentifier: [(Any) -> Void]] = [:]
    
    func subscribe<E>(_ eventType: E.Type, handler: @escaping (E) -> Void) -> AnyCancellable {
        let key = ObjectIdentifier(eventType)
        let wrappedHandler: (Any) -> Void = { event in
            guard let typedEvent = event as? E else { return }
            handler(typedEvent)
        }
        handlers[key, default: []].append(wrappedHandler)
        
        // Возвращаем cancellable:
        return AnyCancellable { [weak self] in
            self?.handlers[key]?.removeLast()
        }
    }
    
    func publish<E>(_ event: E) {
        let key = ObjectIdentifier(type(of: event) as Any.Type)
        handlers[key]?.forEach { $0(event) }
    }
}

// Типизированные события:
struct UserLoggedIn { let user: User }
struct CartItemAdded { let product: Product; let quantity: Int }

let bus = EventBus()

let sub = bus.subscribe(UserLoggedIn.self) { event in
    print("User logged in: \(event.user.name)")
}

bus.publish(UserLoggedIn(user: currentUser))  // type-safe!
// bus.publish("wrong type")  // отправит, но обработчик не сработает
```

**Как реализовать Undo/Redo через Command?**
```swift
// Полная реализация Undo/Redo:
protocol UndoableCommand {
    func execute()
    func undo()
    var description: String { get }
}

class UndoManager {
    private var history: [UndoableCommand] = []
    private var redoStack: [UndoableCommand] = []
    private let maxHistory = 50
    
    var canUndo: Bool { !history.isEmpty }
    var canRedo: Bool { !redoStack.isEmpty }
    
    func execute(_ command: UndoableCommand) {
        command.execute()
        history.append(command)
        redoStack.removeAll()  // новое действие очищает redo
        if history.count > maxHistory { history.removeFirst() }
    }
    
    func undo() {
        guard let command = history.popLast() else { return }
        command.undo()
        redoStack.append(command)
    }
    
    func redo() {
        guard let command = redoStack.popLast() else { return }
        command.execute()
        history.append(command)
    }
    
    func undoHistory() -> [String] { history.map(\.description).reversed() }
}

// Конкретные команды для текстового редактора:
class InsertTextCommand: UndoableCommand {
    private let textView: UITextView
    private let text: String
    private let range: NSRange
    var description: String { "Insert '\(text)'" }
    
    init(textView: UITextView, text: String, at range: NSRange) {
        self.textView = textView
        self.text = text
        self.range = range
    }
    
    func execute() {
        textView.textStorage.insert(NSAttributedString(string: text), at: range.location)
    }
    
    func undo() {
        let deleteRange = NSRange(location: range.location, length: text.count)
        textView.textStorage.deleteCharacters(in: deleteRange)
    }
}
```

**Как State паттерн соотносится со Swift enum?**
```swift
// Enum — идеальная реализация State в Swift:
enum NetworkState<T> {
    case idle
    case loading
    case success(T)
    case failure(Error)
    
    var isLoading: Bool {
        if case .loading = self { return true }
        return false
    }
    
    var value: T? {
        if case .success(let value) = self { return value }
        return nil
    }
    
    func map<U>(_ transform: (T) -> U) -> NetworkState<U> {
        switch self {
        case .idle: return .idle
        case .loading: return .loading
        case .success(let value): return .success(transform(value))
        case .failure(let error): return .failure(error)
        }
    }
}

// Преимущества enum-based State Machine:
// ✅ Exhaustive switch — компилятор требует обработки всех состояний
// ✅ Associated values — данные привязаны к состоянию
// ✅ Нет невалидных состояний (isLoading = true, data != nil — невозможно)
// ✅ Value semantics — thread safe при правильном использовании
```

**Когда использовать Strategy vs Dependency Injection?**
```
Strategy:
  • Алгоритм меняется в runtime
  • Нужно переключать реализацию динамически
  • Объект сам контролирует когда менять стратегию
  Example: сортировка, форматирование, валидация

DI:
  • Зависимость фиксирована на время жизни объекта
  • Внешний код контролирует реализацию
  • Тестируемость — подмена моками
  Example: NetworkClient, Repository, Analytics

Граница размыта:
  // Strategy:
  var sortStrategy: SortStrategy = QuickSort()
  func sort() { sortStrategy.sort(data) }
  
  // DI:
  let networkClient: NetworkClientProtocol  // через init, не меняется
  
  // Часто DI используется для Strategy:
  // Передаём стратегию через DI, но можем менять в runtime
```

---

## Structural Patterns (Структурные)

### Beginner

**Что такое Adapter паттерн?**
> Конвертирует интерфейс одного класса в интерфейс который ожидает клиент. Позволяет работать несовместимым интерфейсам вместе.
```swift
// Адаптируем legacy API к современному:
// Legacy:
class XMLParser {
    func parseXML(_ xml: String) -> [String: String] { [:] }
}

// Наш интерфейс:
protocol DataParser {
    func parse(_ data: Data) -> [String: String]
}

// Adapter:
class XMLParserAdapter: DataParser {
    private let xmlParser = XMLParser()
    
    func parse(_ data: Data) -> [String: String] {
        let xml = String(data: data, encoding: .utf8) ?? ""
        return xmlParser.parseXML(xml)
    }
}

// UIKit → SwiftUI через UIViewRepresentable — тоже Adapter!
struct MapViewAdapter: UIViewRepresentable {
    func makeUIView(context: Context) -> MKMapView { MKMapView() }
    func updateUIView(_ uiView: MKMapView, context: Context) { }
}
```

**Что такое Facade паттерн?**
> Предоставляет упрощённый интерфейс к сложной подсистеме.
```swift
// Facade для медиа операций:
class MediaFacade {
    private let camera = CameraManager()
    private let imageProcessor = ImageProcessor()
    private let fileManager = LocalFileManager()
    private let uploader = CloudUploader()
    
    // Простой интерфейс вместо работы с 4 классами:
    func captureAndUploadPhoto() async throws -> URL {
        let image = try await camera.capture()
        let processed = imageProcessor.resize(image, to: CGSize(width: 1080, height: 1080))
        let compressed = imageProcessor.compress(processed, quality: 0.8)
        let localURL = try fileManager.save(compressed)
        let cloudURL = try await uploader.upload(from: localURL)
        return cloudURL
    }
}

// Пример в iOS SDK:
// AVFoundation — Facade над CoreMedia, CoreAudio, CoreVideo
// UIKit — Facade над CoreGraphics, CoreAnimation, CoreText
```

**Что такое Decorator паттерн?**
> Динамически добавляет объекту новую функциональность путём обёртки в объект-декоратор.
```swift
// Декораторы для NetworkClient:
protocol NetworkClient {
    func fetch(_ url: URL) async throws -> Data
}

// Базовая реализация:
class URLSessionClient: NetworkClient {
    func fetch(_ url: URL) async throws -> Data {
        let (data, _) = try await URLSession.shared.data(from: url)
        return data
    }
}

// Декоратор — логирование:
class LoggingClient: NetworkClient {
    private let wrapped: NetworkClient
    init(_ client: NetworkClient) { wrapped = client }
    
    func fetch(_ url: URL) async throws -> Data {
        print("→ Request: \(url)")
        let start = Date()
        let data = try await wrapped.fetch(url)
        print("← Response: \(url) [\(Date().timeIntervalSince(start):.2f)s, \(data.count) bytes]")
        return data
    }
}

// Декоратор — кэширование:
class CachingClient: NetworkClient {
    private let wrapped: NetworkClient
    private var cache: [URL: (data: Data, timestamp: Date)] = [:]
    private let ttl: TimeInterval
    
    init(_ client: NetworkClient, ttl: TimeInterval = 300) {
        wrapped = client
        self.ttl = ttl
    }
    
    func fetch(_ url: URL) async throws -> Data {
        if let cached = cache[url],
           Date().timeIntervalSince(cached.timestamp) < ttl {
            return cached.data
        }
        let data = try await wrapped.fetch(url)
        cache[url] = (data, Date())
        return data
    }
}

// Декоратор — retry:
class RetryClient: NetworkClient {
    private let wrapped: NetworkClient
    private let maxRetries: Int
    
    init(_ client: NetworkClient, maxRetries: Int = 3) {
        wrapped = client
        self.maxRetries = maxRetries
    }
    
    func fetch(_ url: URL) async throws -> Data {
        var lastError: Error?
        for attempt in 1...maxRetries {
            do {
                return try await wrapped.fetch(url)
            } catch {
                lastError = error
                if attempt < maxRetries {
                    try await Task.sleep(for: .seconds(Double(attempt)))
                }
            }
        }
        throw lastError!
    }
}

// Композиция декораторов:
let client: NetworkClient = RetryClient(
    CachingClient(
        LoggingClient(
            URLSessionClient()
        ),
        ttl: 600
    ),
    maxRetries: 3
)
```

---

### Middle

**Что такое Composite паттерн?**
> Компонует объекты в древовидные структуры для представления иерархий. Клиент одинаково работает с отдельными объектами и группами.
```swift
// Файловая система:
protocol FileSystemItem {
    var name: String { get }
    var size: Int { get }
    func display(indent: String)
}

class File: FileSystemItem {
    let name: String
    let size: Int
    
    init(name: String, size: Int) {
        self.name = name
        self.size = size
    }
    
    func display(indent: String = "") {
        print("\(indent)📄 \(name) (\(size) bytes)")
    }
}

class Directory: FileSystemItem {
    let name: String
    private var children: [FileSystemItem] = []
    
    init(name: String) { self.name = name }
    
    var size: Int { children.reduce(0) { $0 + $1.size } }
    
    func add(_ item: FileSystemItem) { children.append(item) }
    func remove(_ name: String) { children.removeAll { $0.name == name } }
    
    func display(indent: String = "") {
        print("\(indent)📁 \(name)/")
        children.forEach { $0.display(indent: indent + "  ") }
    }
}

// UIView hierarchy = Composite!
// Каждый UIView может содержать subviews и сам является view
```

**Что такое Bridge паттерн?**
> Разделяет абстракцию от реализации так чтобы они могли изменяться независимо.
```swift
// Без Bridge: N абстракций × M реализаций = N×M классов
// С Bridge: N + M классов

// Абстракция — View:
protocol Shape {
    var renderer: Renderer { get }  // bridge к реализации
    func draw()
    func resize(factor: Double)
}

// Реализация — рендерер:
protocol Renderer {
    func renderCircle(radius: Double)
    func renderRect(width: Double, height: Double)
}

class VectorRenderer: Renderer {
    func renderCircle(radius: Double) { print("Vector circle r=\(radius)") }
    func renderRect(width: Double, height: Double) { print("Vector rect \(width)×\(height)") }
}

class RasterRenderer: Renderer {
    func renderCircle(radius: Double) { print("Raster circle r=\(radius)") }
    func renderRect(width: Double, height: Double) { print("Raster rect \(width)×\(height)") }
}

// Конкретные фигуры:
class Circle: Shape {
    let renderer: Renderer
    var radius: Double
    
    init(renderer: Renderer, radius: Double) {
        self.renderer = renderer
        self.radius = radius
    }
    
    func draw() { renderer.renderCircle(radius: radius) }
    func resize(factor: Double) { radius *= factor }
}

// Изменяем рендерер независимо от фигуры:
let circle = Circle(renderer: VectorRenderer(), radius: 5)
circle.draw()
// Потом заменяем рендерер без изменения Circle
```

**Что такое Proxy паттерн?**
> Предоставляет суррогат или заместитель другому объекту для контроля доступа.
```swift
// Типы Proxy:
// 1. Virtual Proxy — ленивая инициализация
// 2. Protection Proxy — контроль доступа
// 3. Remote Proxy — локальный представитель удалённого объекта
// 4. Caching Proxy — кэширование

// Virtual Proxy — ленивая загрузка изображения:
class LazyImageLoader {
    private let url: URL
    private var _image: UIImage?
    
    init(url: URL) { self.url = url }
    
    var image: UIImage {
        if _image == nil {
            print("Loading image from \(url)...")
            _image = UIImage()  // загрузка
        }
        return _image!
    }
}

// Protection Proxy — контроль доступа:
protocol DataService {
    func getData() -> [String]
    func deleteAll()
}

class RealDataService: DataService {
    func getData() -> [String] { ["data1", "data2"] }
    func deleteAll() { print("All data deleted!") }
}

class SecureDataProxy: DataService {
    private let service = RealDataService()
    private let currentUser: User
    
    init(user: User) { currentUser = user }
    
    func getData() -> [String] {
        guard currentUser.isAuthenticated else { return [] }
        return service.getData()
    }
    
    func deleteAll() {
        guard currentUser.isAdmin else {
            print("Access denied: admin required")
            return
        }
        service.deleteAll()
    }
}
```

**Как Facade используется для сетевого слоя?**
```swift
// Facade скрывает: URLSession, Codable, error mapping, retry, auth:
class APIClient {
    private let session: URLSession
    private let baseURL: URL
    private let tokenProvider: TokenProvider
    private let decoder = JSONDecoder()
    
    init(baseURL: URL, tokenProvider: TokenProvider) {
        self.baseURL = baseURL
        self.tokenProvider = tokenProvider
        
        let config = URLSessionConfiguration.default
        config.timeoutIntervalForRequest = 30
        session = URLSession(configuration: config)
    }
    
    // Простой интерфейс — Facade:
    func get<T: Decodable>(_ path: String) async throws -> T {
        try await request(.get, path: path, body: Optional<Never>.none)
    }
    
    func post<Body: Encodable, T: Decodable>(_ path: String, body: Body) async throws -> T {
        try await request(.post, path: path, body: body)
    }
    
    func delete(_ path: String) async throws {
        let _: EmptyResponse = try await request(.delete, path: path, body: Optional<Never>.none)
    }
    
    // Скрытая сложность:
    private func request<Body: Encodable, T: Decodable>(
        _ method: HTTPMethod,
        path: String,
        body: Body?
    ) async throws -> T {
        var request = URLRequest(url: baseURL.appendingPathComponent(path))
        request.httpMethod = method.rawValue
        request.setValue("application/json", forHTTPHeaderField: "Content-Type")
        request.setValue("Bearer \(tokenProvider.token)", forHTTPHeaderField: "Authorization")
        
        if let body = body {
            request.httpBody = try JSONEncoder().encode(body)
        }
        
        let (data, response) = try await session.data(for: request)
        
        guard let httpResponse = response as? HTTPURLResponse else {
            throw APIError.invalidResponse
        }
        
        switch httpResponse.statusCode {
        case 200...299:
            return try decoder.decode(T.self, from: data)
        case 401:
            throw APIError.unauthorized
        case 404:
            throw APIError.notFound
        default:
            throw APIError.serverError(httpResponse.statusCode)
        }
    }
}

// Использование Facade — просто:
let users: [User] = try await api.get("/users")
let user: User = try await api.post("/users", body: newUserRequest)
```

**Как Decorator реализовать через протоколы?**
```swift
// Протокол как абстракция:
protocol TextTransformer {
    func transform(_ text: String) -> String
}

// Базовая реализация:
struct IdentityTransformer: TextTransformer {
    func transform(_ text: String) -> String { text }
}

// Декораторы через протокол:
struct UppercaseDecorator: TextTransformer {
    private let wrapped: TextTransformer
    init(_ transformer: TextTransformer) { wrapped = transformer }
    func transform(_ text: String) -> String { wrapped.transform(text).uppercased() }
}

struct TrimDecorator: TextTransformer {
    private let wrapped: TextTransformer
    init(_ transformer: TextTransformer) { wrapped = transformer }
    func transform(_ text: String) -> String { wrapped.transform(text).trimmingCharacters(in: .whitespaces) }
}

struct PrefixDecorator: TextTransformer {
    private let wrapped: TextTransformer
    private let prefix: String
    init(_ transformer: TextTransformer, prefix: String) {
        wrapped = transformer
        self.prefix = prefix
    }
    func transform(_ text: String) -> String { prefix + wrapped.transform(text) }
}

// Композиция:
let transformer: TextTransformer = PrefixDecorator(
    UppercaseDecorator(
        TrimDecorator(IdentityTransformer())
    ),
    prefix: ">>> "
)
transformer.transform("  hello world  ")  // ">>> HELLO WORLD"
```

**Что такое Flyweight паттерн и где он используется в UIKit?**
```swift
// Flyweight: разделяет общее состояние между множеством объектов
// Intrinsic state (общее, неизменяемое) — в Flyweight
// Extrinsic state (уникальное) — передаётся при вызове

// UIKit примеры:
// UITableViewCell — dequeue переиспользует ячейки (cell pool = Flyweight)
// UIFont — одинаковые шрифты разделяют данные
// UIImage(named:) — кэширует изображения (imageName = Flyweight key)
// NSString interning — одинаковые строки могут делить память

// Кастомный Flyweight для текстовых атрибутов:
struct TextAttributes {
    let font: UIFont
    let color: UIColor
    let alignment: NSTextAlignment
}

class TextAttributesCache {
    private var cache: [String: TextAttributes] = [:]
    
    func attributes(font: UIFont, color: UIColor, alignment: NSTextAlignment) -> TextAttributes {
        let key = "\(font.fontName)-\(font.pointSize)-\(color)-\(alignment.rawValue)"
        
        if let cached = cache[key] { return cached }
        
        let attrs = TextAttributes(font: font, color: color, alignment: alignment)
        cache[key] = attrs
        return attrs
    }
}

// Flyweight для частиц в игре:
struct ParticleType {  // Flyweight — общие данные
    let texture: UIImage
    let color: UIColor
    let mass: Float
}

struct Particle {  // Extrinsic state — уникальные данные
    let type: ParticleType  // ссылка на Flyweight
    var position: CGPoint
    var velocity: CGVector
    var lifetime: Float
}
```

---

## Architecture

### MVC, MVP, MVVM, VIPER

### Beginner

**Что такое MVC?**
> Model-View-Controller — базовая архитектура Apple. Model — данные и бизнес-логика. View — отображение. Controller — посредник между M и V.
```
Model ←→ Controller ←→ View
         ↑_________________↑
         (иногда View → Controller напрямую)
```

**Что такое Model, View, Controller?**
```swift
// Model — данные:
struct User: Codable {
    let id: Int
    let name: String
    let email: String
}

// View — только UI, без логики:
class UserView: UIView {
    let nameLabel = UILabel()
    let emailLabel = UILabel()
    
    func configure(name: String, email: String) {
        nameLabel.text = name
        emailLabel.text = email
    }
}

// Controller — связывает M и V:
class UserViewController: UIViewController {
    private var user: User?
    private let userView = UserView()
    private let service = UserService()
    
    func loadUser(id: Int) {
        service.fetchUser(id: id) { [weak self] user in
            self?.user = user
            self?.userView.configure(name: user.name, email: user.email)
        }
    }
}
```

**Какая основная проблема MVC в iOS?**
> **Massive View Controller** — ViewController берёт на себя слишком много обязанностей: networking, parsing, business logic, navigation, data formatting — вместо только координации M и V.

**Что такое Massive View Controller?**
```swift
// Anti-pattern — ViewController делает всё:
class ProfileViewController: UIViewController,
    UITableViewDataSource, UITableViewDelegate,
    UIImagePickerControllerDelegate,
    CLLocationManagerDelegate {
    
    // Networking:
    func loadProfile() { URLSession.shared.dataTask... }
    
    // Business logic:
    func validateEmail(_ email: String) -> Bool { ... }
    func calculateAge(from birthday: Date) -> Int { ... }
    
    // Data formatting:
    func formatDate(_ date: Date) -> String { ... }
    
    // Navigation:
    func showSettings() { navigationController?.push... }
    
    // All delegate methods...
    // 1000+ lines!
}
```

**Что такое MVP?**
> Model-View-Presenter. Presenter берёт логику из Controller. View — пассивный (только отображает, не содержит логики). Presenter не зависит от UIKit — легко тестируется.
```
View ←→ Presenter → Model
```

**Что такое MVVM?**
> Model-View-ViewModel. ViewModel содержит бизнес-логику и подготовку данных для View. Binding между View и ViewModel (через Combine, @Published). ViewModel не зависит от UIKit.
```
View ←(binding)→ ViewModel → Model
```

---

### Middle

**Чем MVVM решает проблемы MVC?**
```swift
// MVC — логика в Controller:
class UserVC: UIViewController {
    func loadUser() {
        URLSession.shared.dataTask(with: url) { data, _, error in
            let user = try! JSONDecoder().decode(User.self, from: data!)
            DispatchQueue.main.async {
                self.nameLabel.text = "\(user.firstName) \(user.lastName)"
                self.ageLabel.text = "Возраст: \(Calendar.current.component(.year, from: Date()) - user.birthYear)"
            }
        }
    }
}

// MVVM — логика в ViewModel, Controller тонкий:
class UserViewModel: ObservableObject {
    @Published var displayName = ""
    @Published var ageText = ""
    @Published var isLoading = false
    
    func loadUser(id: Int) async {
        isLoading = true
        let user = try! await userService.fetchUser(id: id)
        displayName = "\(user.firstName) \(user.lastName)"  // форматирование в VM
        ageText = "Возраст: \(age(from: user.birthYear))"
        isLoading = false
    }
}

class UserVC: UIViewController {
    private let viewModel = UserViewModel()
    // Только binding — никакой логики
    override func viewDidLoad() {
        viewModel.$displayName.assign(to: \.text, on: nameLabel).store(in: &cancellables)
        Task { await viewModel.loadUser(id: userId) }
    }
}
```

**Что такое ViewModel?**
> Слой между View и Model. Содержит: бизнес-логику, форматирование данных для отображения, состояние UI (isLoading, error), команды (методы вызываемые из View). Не содержит: UIKit импорты, ссылки на View.

**Что такое data binding?**
```swift
// Combine binding:
viewModel.$userName
    .receive(on: DispatchQueue.main)
    .assign(to: \.text, on: nameLabel)
    .store(in: &cancellables)

// SwiftUI binding через @Published:
@ObservedObject var viewModel: UserViewModel
Text(viewModel.userName)  // автоматически обновляется

// Closure binding (простейший способ):
class ViewModel {
    var onUserLoaded: ((User) -> Void)?
    
    func loadUser() {
        service.fetch { [weak self] user in
            self?.onUserLoaded?(user)
        }
    }
}
```

**Что такое VIPER?**
> View-Interactor-Presenter-Entity-Router. Самая строгая архитектура. Каждый модуль = папка с 5 файлами + протоколы.
```
View ↔ Presenter ↔ Interactor ↔ Entity (Model)
       ↕
      Router (навигация)
```

**Что такое Interactor, Presenter, Router в VIPER?**
```swift
// Interactor — бизнес-логика:
class UserInteractor: UserInteractorInput {
    weak var presenter: UserInteractorOutput?
    private let userService: UserServiceProtocol
    
    func fetchUser(id: Int) {
        Task {
            do {
                let user = try await userService.fetchUser(id: id)
                await presenter?.didFetchUser(user)
            } catch {
                await presenter?.didFailFetchUser(error)
            }
        }
    }
}

// Presenter — форматирует данные:
class UserPresenter: UserPresenterInput, UserInteractorOutput {
    weak var view: UserViewInput?
    var router: UserRouterInput?
    var interactor: UserInteractorInput?
    
    func viewDidLoad() { interactor?.fetchUser(id: currentUserId) }
    
    func didFetchUser(_ user: User) {
        let viewModel = UserViewModel(
            name: "\(user.firstName) \(user.lastName)",
            email: user.email
        )
        view?.showUser(viewModel)
    }
    
    func didTapEditButton() { router?.navigateToEditUser() }
}

// Router — навигация:
class UserRouter: UserRouterInput {
    weak var viewController: UIViewController?
    
    static func build(userId: Int) -> UIViewController {
        let vc = UserViewController()
        let presenter = UserPresenter()
        let interactor = UserInteractor()
        let router = UserRouter()
        
        vc.presenter = presenter
        presenter.view = vc
        presenter.interactor = interactor
        presenter.router = router
        interactor.presenter = presenter
        router.viewController = vc
        
        return vc
    }
    
    func navigateToEditUser() {
        let editVC = EditUserRouter.build()
        viewController?.navigationController?.pushViewController(editVC, animated: true)
    }
}
```

**Что такое Clean Architecture?**
```
Layers (от внешних к внутренним):
  Frameworks & Drivers (UI, DB, Network)
      ↓
  Interface Adapters (Controllers, Presenters, Gateways)
      ↓
  Use Cases (Application Business Rules)
      ↓
  Entities (Enterprise Business Rules)

Dependency Rule: зависимости только внутрь!
```
```swift
// Entity — чистая бизнес-сущность:
struct User {
    let id: UUID
    let name: String
    let email: Email  // value object
}

// Use Case:
class FetchUserUseCase {
    private let repository: UserRepository  // протокол, не реализация
    
    init(repository: UserRepository) { self.repository = repository }
    
    func execute(id: UUID) async throws -> User {
        let user = try await repository.fetchUser(id: id)
        // бизнес-правила:
        guard user.isActive else { throw UserError.accountDeactivated }
        return user
    }
}

// Repository (протокол в Domain, реализация в Data):
protocol UserRepository {
    func fetchUser(id: UUID) async throws -> User
}

class RemoteUserRepository: UserRepository {
    func fetchUser(id: UUID) async throws -> User {
        // HTTP запрос → маппинг в Entity
    }
}
```

**Что такое Coordinator паттерн?**
```swift
// Coordinator управляет навигацией, освобождая VC:
protocol Coordinator: AnyObject {
    var childCoordinators: [Coordinator] { get set }
    func start()
}

class AppCoordinator: Coordinator {
    var childCoordinators: [Coordinator] = []
    private let window: UIWindow
    private let navigationController = UINavigationController()
    
    init(window: UIWindow) { self.window = window }
    
    func start() {
        window.rootViewController = navigationController
        showAuth()
    }
    
    private func showAuth() {
        let coordinator = AuthCoordinator(navigationController: navigationController)
        coordinator.delegate = self
        childCoordinators.append(coordinator)
        coordinator.start()
    }
    
    private func showMain() {
        let coordinator = MainCoordinator(navigationController: navigationController)
        childCoordinators.append(coordinator)
        coordinator.start()
    }
}

extension AppCoordinator: AuthCoordinatorDelegate {
    func authDidComplete() {
        childCoordinators.removeAll { $0 is AuthCoordinator }
        showMain()
    }
}
```

---

### Senior

**Как выбрать архитектуру для большого проекта?**
```
Критерии выбора:
  Размер команды:
    1-2 разработчика → MVVM
    3-5 → MVVM + Coordinator / Clean Architecture
    5+  → Clean Architecture + VIPER или TCA
  
  Сложность домена:
    CRUD приложение → MVC/MVVM
    Сложная бизнес-логика → Clean Architecture
    Много состояний → TCA
  
  Тестируемость:
    MVVM: тестировать ViewModel ✅
    VIPER: тестировать каждый слой ✅✅
    TCA: reducers = pure functions ✅✅✅
  
  Скорость разработки:
    MVC > MVVM > VIPER (boilerplate)
    TCA = высокий порог вхождения
  
  Вопросы для принятия решения:
  - Сколько экранов?
  - Насколько сложна бизнес-логика?
  - Нужен ли offline?
  - Насколько важна тестируемость?
  - Опыт команды?
```

**Что такое TCA (The Composable Architecture)?**
```swift
import ComposableArchitecture

// State:
struct CounterState: Equatable {
    var count = 0
    var isLoading = false
}

// Actions:
enum CounterAction {
    case increment
    case decrement
    case loadButtonTapped
    case loadResponse(Result<Int, Error>)
}

// Reducer — pure function:
let counterReducer = Reducer<CounterState, CounterAction, CounterEnvironment> { state, action, env in
    switch action {
    case .increment:
        state.count += 1
        return .none
        
    case .decrement:
        state.count -= 1
        return .none
        
    case .loadButtonTapped:
        state.isLoading = true
        return env.fetchCount()
            .receive(on: env.mainQueue)
            .catchToEffect(CounterAction.loadResponse)
        
    case .loadResponse(.success(let count)):
        state.isLoading = false
        state.count = count
        return .none
        
    case .loadResponse(.failure):
        state.isLoading = false
        return .none
    }
}

// View:
struct CounterView: View {
    let store: Store<CounterState, CounterAction>
    
    var body: some View {
        WithViewStore(store) { viewStore in
            VStack {
                Text("\(viewStore.count)")
                Button("+") { viewStore.send(.increment) }
                Button("-") { viewStore.send(.decrement) }
            }
        }
    }
}
```

**Что такое unidirectional data flow?**
```
Однонаправленный поток данных:

Action → Reducer → State → View → Action → ...

Преимущества:
  • Предсказуемость: одинаковый action → всегда одинаковый state
  • Time-travel debugging: можно воспроизвести любое состояние
  • Легко тестировать: reducer = pure function
  • Нет скрытых мутаций state

Реализации в iOS:
  • TCA (The Composable Architecture)
  • ReSwift (Redux-like)
  • Elm Architecture
  • SwiftUI @Observable + unidirectional обычного
```

---

### Staff

**Как масштабировать архитектуру для команды 50+ разработчиков?**
```
Принципы:
  1. Vertical Slices: каждая фича — изолированный модуль
     Features/Auth/, Features/Feed/, Features/Profile/
  
  2. Shared Core: переиспользуемые компоненты
     SharedUI/, NetworkKit/, Analytics/, StorageKit/
  
  3. Feature Dependencies: через протоколы, не конкретные типы
     Auth.isLoggedIn: → AnyPublisher<Bool, Never>
  
  4. Interface Segregation: разные подмодули для интерфейса и реализации
     AuthInterface/ (только протоколы) ← зависят другие фичи
     Auth/          (реализация)       ← зависит только App
  
  5. Dependency Graph: строго однонаправленный
     App → Features → Shared → Core
     НЕТ: Feature A ↔ Feature B (только через Shared)

  6. Build Metrics: время компиляции, size
     Цель: любой модуль < 5 мин компиляции
     Параллельная компиляция через модуляризацию
```

**Как минимизировать время компиляции через модуляризацию?**
```
Стратегии:
  1. SPM/CocoaPods модули: параллельная компиляция
     Один монолит: 15 мин → 10 SPM модулей: 3 мин (параллельно)
  
  2. Precompiled frameworks: зависимости не пересобираются
     
  3. Избегать @_exported import — цепные перекомпиляции
  
  4. Explicit @inlinable только когда нужно
  
  5. Typealiases в public API вместо прямого использования внутренних типов
  
  6. Измерять: -Xfrontend -debug-time-function-bodies
     Найти самые медленные выражения для компилятора
  
  7. Whole-module optimization только для Release
  
  Инструменты:
    XCLogParser — анализ build логов
    BuildTimeAnalyzer — найти медленные функции
    Periphery — найти неиспользуемый код
```

---

## SOLID

### Beginner

**Что такое SOLID?**
> 5 принципов объектно-ориентированного программирования для создания поддерживаемого кода:
- **S** — Single Responsibility Principle
- **O** — Open/Closed Principle
- **L** — Liskov Substitution Principle
- **I** — Interface Segregation Principle
- **D** — Dependency Inversion Principle

**Что такое Single Responsibility Principle?**
> Класс должен иметь только одну причину для изменения — одну ответственность.
```swift
// ❌ Нарушение SRP:
class UserManager {
    func fetchUser(id: Int) -> User { /* сеть */ }
    func saveToDatabase(_ user: User) { /* БД */ }
    func sendWelcomeEmail(to user: User) { /* email */ }
    func formatUserForDisplay(_ user: User) -> String { /* форматирование */ }
    // 4 причины для изменения!
}

// ✅ SRP:
class UserNetworkService { func fetchUser(id: Int) async throws -> User { } }
class UserRepository { func save(_ user: User) { } }
class EmailService { func sendWelcome(to user: User) { } }
class UserFormatter { func format(_ user: User) -> String { } }
```

**Что такое Open/Closed Principle?**
> Классы открыты для расширения, закрыты для модификации.
```swift
// ❌ Нарушение OCP — добавление нового типа требует изменения класса:
class DiscountCalculator {
    func calculate(type: String, price: Double) -> Double {
        switch type {
        case "student": return price * 0.9
        case "senior": return price * 0.85
        // добавление "employee" требует менять этот класс!
        default: return price
        }
    }
}

// ✅ OCP — расширяем через протоколы:
protocol DiscountStrategy {
    func apply(to price: Double) -> Double
}
class StudentDiscount: DiscountStrategy { func apply(to price: Double) -> Double { price * 0.9 } }
class SeniorDiscount: DiscountStrategy { func apply(to price: Double) -> Double { price * 0.85 } }
class EmployeeDiscount: DiscountStrategy { func apply(to price: Double) -> Double { price * 0.7 } }

class DiscountCalculator {
    func calculate(strategy: DiscountStrategy, price: Double) -> Double {
        strategy.apply(to: price)
    }
}
// Новый тип скидки — новый класс, не изменяем DiscountCalculator
```

**Что такое Liskov Substitution Principle?**
> Объекты подклассов должны заменять объекты базовых классов без нарушения корректности программы.
```swift
// ❌ Нарушение LSP:
class Rectangle {
    var width: Double
    var height: Double
    var area: Double { width * height }
    init(width: Double, height: Double) { self.width = width; self.height = height }
}

class Square: Rectangle {
    override var width: Double {
        didSet { height = width }  // нарушает ожидания Rectangle
    }
}

// Код ломается:
func testRectangle(_ rect: Rectangle) {
    rect.width = 4
    rect.height = 5
    assert(rect.area == 20)  // Square: width = 5, height = 5, area = 25 ≠ 20!
}
```

---

### Middle

**Что такое Interface Segregation Principle?**
> Клиенты не должны зависеть от интерфейсов которые они не используют. Разбивай толстые протоколы на маленькие.
```swift
// ❌ Нарушение ISP — толстый протокол:
protocol Worker {
    func work()
    func eat()
    func sleep()
}

class Robot: Worker {
    func work() { print("Working") }
    func eat() { fatalError("Robots don't eat!") }  // нарушение!
    func sleep() { fatalError("Robots don't sleep!") }
}

// ✅ ISP — разбить на малые протоколы:
protocol Workable { func work() }
protocol Feedable { func eat() }
protocol Restable { func sleep() }

class Human: Workable, Feedable, Restable {
    func work() { }
    func eat() { }
    func sleep() { }
}

class Robot: Workable {
    func work() { }
    // Не реализует eat/sleep — и не должен!
}
```

**Что такое Dependency Inversion Principle?**
> Модули высокого уровня не должны зависеть от модулей низкого уровня. Оба должны зависеть от абстракций.
```swift
// ❌ Нарушение DIP:
class UserViewModel {
    private let service = MySQLUserService()  // зависит от конкретной реализации!
}

// ✅ DIP — через протоколы:
protocol UserServiceProtocol {
    func fetchUsers() async throws -> [User]
}

class UserViewModel {
    private let service: UserServiceProtocol  // зависит от абстракции
    init(service: UserServiceProtocol) { self.service = service }
}

class MySQLUserService: UserServiceProtocol { /* ... */ }
class FirebaseUserService: UserServiceProtocol { /* ... */ }
// Можно подменять без изменения UserViewModel
```

**Как SOLID применяется в iOS разработке?**
```
SRP → Разделять VC на ViewModel + DataSource + Navigator
OCP → Протоколы для расширения без изменения кода
LSP → Корректные иерархии классов, не ломать behaviour
ISP → Маленькие протоколы делегатов
DIP → DI через протоколы вместо конкретных классов
```

**Как реализовать DIP через протоколы в Swift?**
```swift
// Вся система через протоколы:
protocol AuthService {
    func login(email: String, password: String) async throws -> User
    func logout() async
}

protocol UserStorage {
    func save(_ user: User) throws
    func load() throws -> User?
}

protocol Analytics {
    func track(_ event: String, properties: [String: Any])
}

// ViewModel зависит только от протоколов:
class LoginViewModel {
    private let authService: AuthService
    private let storage: UserStorage
    private let analytics: Analytics
    
    init(
        authService: AuthService,
        storage: UserStorage,
        analytics: Analytics
    ) {
        self.authService = authService
        self.storage = storage
        self.analytics = analytics
    }
}

// Конкретные реализации зависят от потребности:
// Production: FirebaseAuth, CoreDataStorage, MixpanelAnalytics
// Tests: MockAuth, InMemoryStorage, MockAnalytics
```

**Что такое composition over inheritance?**
```swift
// ❌ Inheritance — жёсткая иерархия:
class Animal { func move() { } }
class Pet: Animal { func cuddle() { } }
class Dog: Pet { func bark() { } }
class GuideDog: Dog { func guide() { } }
// Хочешь Wild Swimming Animal? Не можешь без изменения иерархии!

// ✅ Composition — гибкая система:
protocol Movable { func move() }
protocol Swimmable { func swim() }
protocol Flyable { func fly() }
protocol Talkable { func talk() }

struct Duck: Movable, Swimmable, Flyable {
    func move() { }
    func swim() { }
    func fly() { }
}

struct Parrot: Movable, Flyable, Talkable {
    func move() { }
    func fly() { }
    func talk() { }
}
// Любая комбинация без изменения иерархии!
```

---

### Senior

**Как SOLID взаимодействует с Protocol-Oriented Programming?**
```swift
// POP + SOLID = мощная комбинация:

// ISP через protocol composition:
typealias NetworkClientProtocol = Requestable & Cancellable & Configurable

// OCP через protocol extension:
protocol Cacheable {
    var cacheKey: String { get }
}

extension Cacheable where Self: Identifiable {
    var cacheKey: String { "\(type(of: self))_\(id)" }
    // Новая функциональность без изменения типов!
}

// DIP через associated types:
protocol Repository {
    associatedtype Entity
    func fetch(id: UUID) async throws -> Entity
    func save(_ entity: Entity) async throws
}

class UserRepository: Repository {
    typealias Entity = User
    func fetch(id: UUID) async throws -> User { /* ... */ }
    func save(_ entity: User) async throws { /* ... */ }
}
```

**Как нарушение LSP может привести к runtime ошибкам?**
```swift
// Пример с UIViewController:
class BaseViewController: UIViewController {
    func showError(_ message: String) {
        let alert = UIAlertController(title: "Error", message: message, preferredStyle: .alert)
        present(alert, animated: true)
    }
}

class ChildViewController: BaseViewController {
    override func showError(_ message: String) {
        // Нарушение LSP: не вызываем super и не показываем alert!
        print("Ignoring error: \(message)")
        // Код использующий BaseViewController ожидает показа Alert!
    }
}

// Правильно: вызывать super или полностью заменять поведение
// согласно контракту базового класса
```

**Как SOLID влияет на testability?**
```
SRP → Маленькие классы = маленькие тесты = быстро понять что тестировать
OCP → Можно добавить поведение через Stub без изменения тестируемого кода
LSP → Уверенность что Mock ведёт себя как реальный объект
ISP → Маленькие протоколы = маленькие Mock (не имплементировать 20 методов)
DIP → DI = можно подменить любую зависимость в тесте

Метрика: если тест требует сложной setup — нарушен SRP или DIP
```

---

## Dependency Injection

### Beginner

**Что такое Dependency Injection?**
> Техника при которой объект получает зависимости извне вместо создания их самостоятельно. Контроль инверсируется — создатель объекта контролирует зависимости, не сам объект.

**Какие виды DI существуют?**
```swift
// 1. Constructor Injection — через init (предпочтительный):
class UserViewModel {
    private let service: UserService
    init(service: UserService) { self.service = service }
}

// 2. Property Injection — через свойство:
class UserViewModel {
    var service: UserService!  // обычно Optional или var
}
let vm = UserViewModel()
vm.service = RealUserService()

// 3. Method Injection — через параметр метода:
class DataProcessor {
    func process(data: Data, validator: DataValidator) -> Result<Void, Error> {
        validator.validate(data)
    }
}

// 4. Interface Injection (через протокол):
protocol ServiceInjectable {
    func inject(service: UserService)
}
```

**Что такое constructor injection?**
```swift
// Самый надёжный способ DI:
// 1. Зависимость обязательна (не Optional)
// 2. Объект сразу готов к работе
// 3. Immutable зависимость (let)
// 4. Легко заметить зависимости

class OrderService {
    private let repository: OrderRepository
    private let paymentGateway: PaymentGateway
    private let notificationService: NotificationService
    
    init(
        repository: OrderRepository,        // 3 обязательные зависимости
        paymentGateway: PaymentGateway,     // видно в сигнатуре!
        notificationService: NotificationService
    ) {
        self.repository = repository
        self.paymentGateway = paymentGateway
        self.notificationService = notificationService
    }
}
```

---

### Middle

**Чем DI помогает тестируемости?**
```swift
// Без DI — нельзя тестировать изолированно:
class OrderService {
    private let gateway = StripePaymentGateway()  // реальные деньги в тестах!
    func charge(amount: Double) { gateway.charge(amount) }
}

// С DI — подменяем в тестах:
protocol PaymentGateway { func charge(amount: Double) throws }
class StripeGateway: PaymentGateway { /* реальная реализация */ }
class MockPaymentGateway: PaymentGateway {
    var chargeCallCount = 0
    var lastAmount: Double?
    func charge(amount: Double) { chargeCallCount += 1; lastAmount = amount }
}

class OrderService {
    private let gateway: PaymentGateway
    init(gateway: PaymentGateway) { self.gateway = gateway }
    func charge(amount: Double) throws { try gateway.charge(amount: amount) }
}

// Тест без реальных денег:
func test_charge_callsGateway() throws {
    let mock = MockPaymentGateway()
    let sut = OrderService(gateway: mock)
    try sut.charge(amount: 99.99)
    XCTAssertEqual(mock.chargeCallCount, 1)
    XCTAssertEqual(mock.lastAmount, 99.99)
}
```

**Как реализовать простой DI контейнер?**
```swift
final class Container {
    private var factories: [ObjectIdentifier: () -> Any] = [:]
    private var singletons: [ObjectIdentifier: Any] = [:]
    
    // Transient — новый объект при каждом resolve:
    func register<T>(_ type: T.Type, factory: @escaping () -> T) {
        factories[ObjectIdentifier(type)] = factory
    }
    
    // Singleton — один объект для всех:
    func registerSingleton<T>(_ type: T.Type, factory: @escaping () -> T) {
        factories[ObjectIdentifier(type)] = { [weak self] in
            let key = ObjectIdentifier(type)
            if let existing = self?.singletons[key] as? T { return existing }
            let instance = factory()
            self?.singletons[key] = instance
            return instance
        }
    }
    
    func resolve<T>(_ type: T.Type) -> T {
        guard let factory = factories[ObjectIdentifier(type)],
              let instance = factory() as? T else {
            fatalError("No registration for \(type)")
        }
        return instance
    }
}

// Использование:
let container = Container()
container.register(NetworkClient.self) { URLSessionNetworkClient() }
container.registerSingleton(UserRepository.self) {
    RemoteUserRepository(client: container.resolve(NetworkClient.self))
}

let repo = container.resolve(UserRepository.self)
```

**Что такое service locator и чем он отличается от DI?**
```swift
// Service Locator — глобальный реестр сервисов (анти-паттерн):
class ServiceLocator {
    static var shared = ServiceLocator()
    private var services: [String: Any] = [:]
    
    func register<T>(_ service: T) {
        services[String(describing: T.self)] = service
    }
    func resolve<T>() -> T {
        services[String(describing: T.self)] as! T
    }
}

class UserViewModel {
    func loadUser() {
        // Зависимость скрыта! Не видна в сигнатуре
        let service: UserService = ServiceLocator.shared.resolve()
    }
}

// Отличия:
// Service Locator: зависимости скрыты, глобальное состояние = плохо для тестов
// DI: зависимости явные (в init/параметрах), легко тестировать
//
// Service Locator нарушает DIP и затрудняет тестирование
// DI делает зависимости явными и подменяемыми
```

**Как реализовать DI без библиотек?**
```swift
// Pure Swift DI через factory closures:
struct AppDependencies {
    // Lazy инициализация:
    lazy var networkClient: NetworkClient = URLSessionNetworkClient()
    
    lazy var userRepository: UserRepository = {
        RemoteUserRepository(client: networkClient)
    }()
    
    lazy var authService: AuthService = {
        FirebaseAuthService(repository: userRepository)
    }()
    
    lazy var userViewModel: UserViewModel = {
        UserViewModel(
            authService: authService,
            analytics: analyticsService
        )
    }()
    
    lazy var analyticsService: Analytics = MixpanelAnalytics()
}

// При старте приложения:
var dependencies = AppDependencies()
let rootVC = RootViewController(viewModel: dependencies.userViewModel)
```

**Что такое Swinject?**
```swift
// Популярный DI фреймворк для Swift:
import Swinject

let container = Container()

// Регистрация:
container.register(NetworkClient.self) { _ in URLSessionNetworkClient() }
container.register(UserRepository.self) { r in
    RemoteUserRepository(client: r.resolve(NetworkClient.self)!)
}
container.register(UserViewModel.self) { r in
    UserViewModel(repository: r.resolve(UserRepository.self)!)
}.inObjectScope(.container)  // singleton scope

// Resolve:
let vm = container.resolve(UserViewModel.self)!

// Object scopes:
// .transient   — новый объект каждый раз
// .container   — singleton в рамках контейнера
// .weak        — слабая ссылка, пересоздаётся если освобождён
// .graph       — singleton в рамках одного resolve дерева
```

**Как DI взаимодействует с SwiftUI Environment?**
```swift
// SwiftUI Environment — встроенный DI механизм:

// 1. Через EnvironmentKey:
struct NetworkClientKey: EnvironmentKey {
    static let defaultValue: NetworkClient = URLSessionNetworkClient()
}

extension EnvironmentValues {
    var networkClient: NetworkClient {
        get { self[NetworkClientKey.self] }
        set { self[NetworkClientKey.self] = newValue }
    }
}

struct UserListView: View {
    @Environment(\.networkClient) var networkClient
    
    var body: some View { /* ... */ }
}

// Внедрение:
UserListView()
    .environment(\.networkClient, MockNetworkClient())  // в тестах

// 2. Через EnvironmentObject:
class AppDependencies: ObservableObject {
    let userService: UserService
    let analyticsService: Analytics
    
    init(userService: UserService, analyticsService: Analytics) {
        self.userService = userService
        self.analyticsService = analyticsService
    }
}

@main struct MyApp: App {
    var body: some Scene {
        WindowGroup {
            ContentView()
                .environmentObject(AppDependencies(
                    userService: RemoteUserService(),
                    analyticsService: MixpanelAnalytics()
                ))
        }
    }
}

struct ProfileView: View {
    @EnvironmentObject var deps: AppDependencies
    var body: some View { /* deps.userService... */ }
}
```

---

### Senior

**Как реализовать type-safe DI контейнер?**
```swift
// Через @resultBuilder и KeyPath:
@propertyWrapper
struct Injected<T> {
    private let container: DIContainer
    private let type: T.Type
    
    var wrappedValue: T { container.resolve(type) }
    
    init(_ type: T.Type, container: DIContainer = .shared) {
        self.type = type
        self.container = container
    }
}

// Использование:
class UserViewModel {
    @Injected(UserRepository.self) private var repository
    @Injected(Analytics.self) private var analytics
    
    // Автоматически резолвится из контейнера
}

// Type-safe через generics:
struct Dependency<T> {
    private let resolve: () -> T
    
    init(_ resolve: @escaping () -> T) {
        self.resolve = resolve
    }
    
    var value: T { resolve() }
}

class MyClass {
    private let userService: Dependency<UserServiceProtocol>
    
    init(userService: Dependency<UserServiceProtocol>) {
        self.userService = userService
    }
    
    func doWork() {
        userService.value.fetchUser(id: 1)
    }
}
```

**Как управлять lifetime объектов через DI?**
```swift
enum Scope {
    case transient   // новый каждый раз
    case singleton   // один на всё приложение
    case weak        // слабая ссылка — пересоздаётся если не retained
    case perRequest  // один на scope (например на один экран)
}

class ScopedContainer {
    private var registrations: [ObjectIdentifier: (Scope, () -> Any)] = [:]
    private var singletons: [ObjectIdentifier: Any] = [:]
    private weak var weakStorage: AnyObject?
    
    func register<T>(_ type: T.Type, scope: Scope = .transient, factory: @escaping () -> T) {
        registrations[ObjectIdentifier(type)] = (scope, factory)
    }
    
    func resolve<T>(_ type: T.Type) -> T {
        let key = ObjectIdentifier(type)
        guard let (scope, factory) = registrations[key] else {
            fatalError("Not registered: \(type)")
        }
        
        switch scope {
        case .transient:
            return factory() as! T
            
        case .singleton:
            if let existing = singletons[key] as? T { return existing }
            let instance = factory() as! T
            singletons[key] = instance
            return instance
            
        case .weak:
            // Более сложная реализация со слабыми ссылками
            return factory() as! T
            
        case .perRequest:
            // Scope Container pattern
            return factory() as! T
        }
    }
}
```

**Как реализовать circular dependency resolution?**
```swift
// Circular: A зависит от B, B зависит от A

// Проблема:
// class A { init(b: B) { } }
// class B { init(a: A) { } }
// Нельзя создать ни один!

// Решение 1: Property injection для одной из зависимостей:
class A {
    var b: B?  // property injection
}
class B {
    let a: A   // constructor injection
    init(a: A) { self.a = a }
}

let a = A()
let b = B(a: a)
a.b = b  // замыкаем круг после создания

// Решение 2: Lazy resolution через замыкание:
class LazyContainer {
    func register<T>(_ type: T.Type, factory: @escaping (LazyContainer) -> T) { }
    func lazy<T>(_ type: T.Type) -> () -> T { { self.resolve(type) } }
}

class A {
    private let getB: () -> B  // ленивое получение B
    init(getB: @escaping () -> B) { self.getB = getB }
    func useB() { getB().doSomething() }
}

// Решение 3: Mediator — убрать прямую зависимость через посредника
```

**Как DI влияет на модуляризацию?**
```swift
// Модуль должен экспортировать только протоколы, не реализации:

// AuthInterface module (протоколы):
public protocol AuthService {
    func login(credentials: Credentials) async throws -> User
    func currentUser() -> User?
    func logout()
}

// Auth module (реализация):
// НЕ экспортирует FirebaseAuthService — только регистрирует в контейнере
public class AuthModuleConfigurator {
    public static func register(in container: DIContainer) {
        container.register(AuthService.self) { _ in
            FirebaseAuthService()  // скрыта от внешнего мира
        }
    }
}

// Feed module зависит от AuthInterface, не от Auth:
// import AuthInterface (не import Auth)
class FeedViewModel {
    private let auth: AuthService  // протокол из AuthInterface
}

// App level собирает всё:
AuthModuleConfigurator.register(in: container)
FeedModuleConfigurator.register(in: container)
// Feed получает AuthService через контейнер не зная о Firebase
```

---

*Конец документа · Design Patterns & Architecture*
