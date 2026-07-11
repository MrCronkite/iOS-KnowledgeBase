# Swift — Вопросы и краткие ответы
> Подготовка к iOS-собеседованию · Уровни: Beginner · Middle · Senior · Staff

---

## Variables & Constants

### Beginner

**Чем отличается `var` от `let` в Swift?**
> `var` — изменяемая переменная, значение можно менять после инициализации. `let` — константа, значение задаётся один раз и не меняется. Компилятор предупредит если `var` нигде не изменяется — лучше использовать `let`.

**Можно ли изменить значение константы `let` после инициализации?**
> Нет. После присвоения значения `let`-константе изменить его невозможно — компилятор выдаст ошибку. Исключение: если `let` — reference type (class), нельзя менять саму ссылку, но можно изменять свойства объекта.

**Что такое type inference в Swift?**
> Компилятор автоматически определяет тип переменной по значению, которым она инициализирована. `var x = 5` — компилятор выводит тип `Int`. Явное указание типа необязательно, но иногда делает код понятнее.

**Как явно указать тип переменной при объявлении?**
> `var name: String = "John"` или `var age: Int`. Тип указывается через двоеточие после имени переменной.

**Что такое `nil` и в каком контексте он используется?**
> `nil` — отсутствие значения. Используется только с Optional-типами (`String?`, `Int?`). Non-optional типы не могут быть `nil` — это гарантируется компилятором.

**Можно ли объявить переменную без начального значения?**
> Да, но только если тип указан явно: `var name: String`. При этом переменная должна быть инициализирована до первого использования, иначе — ошибка компилятора.

**Чем отличается объявление `var x = 5` от `var x: Int = 5`?**
> Результат одинаковый — в обоих случаях создаётся `Int`. Первый вариант использует type inference, второй — явную аннотацию типа. Разницы в рантайме нет.

**Что такое shadowing переменной?**
> Объявление новой переменной с тем же именем во вложенной области видимости. Внутри области используется новая переменная, снаружи — старая. Часто используется в `if let name = name { }`.

**Что произойдёт если обратиться к переменной до её инициализации?**
> Ошибка компиляции: `variable 'x' used before being initialized`. Swift требует инициализации до использования — это гарантия безопасности на уровне компилятора.

**Можно ли в Swift объявить несколько переменных в одной строке?**
> Да: `var a = 1, b = 2, c = 3` или `var x: Int, y: Int`. Типы могут быть разными при явном указании: `var name: String, age: Int`.

---

### Middle

**В чём разница между `let` для value type и reference type?**
> Для value type (`struct`, `Int`, `Array`) — нельзя ни изменить значение, ни присвоить новое. Для reference type (`class`) — нельзя заменить ссылку на другой объект, но можно изменять свойства самого объекта (они mutable).

**Почему `let` массив нельзя изменять, но `let` объект класса можно мутировать?**
> `Array` — value type. `let array` делает неизменяемой саму структуру. `class` — reference type. `let object` фиксирует адрес в памяти (ссылку), но содержимое объекта по этому адресу может меняться.

**Что такое stored property и computed property?**
> Stored property — реально хранит значение в памяти (`var name: String`). Computed property — вычисляется каждый раз при обращении через `get`/`set`, физически место в памяти не занимает.

**Как работает lazy stored property?**
> Инициализируется только при первом обращении, а не при создании объекта. Полезно для дорогостоящих объектов. Всегда `var`, не `let`. Не thread-safe по умолчанию.

**Когда стоит использовать `lazy var` вместо обычной переменной?**
> Когда инициализация дорогая (создание сложного объекта, загрузка данных) и свойство может не понадобиться вообще. Также когда для инициализации нужен `self` (в замыкании).

**Что такое willSet и didSet?**
> Property observers — код, выполняющийся при изменении значения. `willSet` вызывается **до** изменения (новое значение доступно как `newValue`), `didSet` — **после** (старое значение как `oldValue`).

**Чем отличается `willSet` от `didSet`?**
> `willSet` срабатывает перед записью нового значения, `didSet` — после. В `willSet` ещё доступно старое значение через `self.property`, в `didSet` — `oldValue` содержит предыдущее значение.

**Что такое property observer и когда он не вызывается?**
> Код, реагирующий на изменение значения. Не вызывается: при инициализации (в `init`), при изменении значения внутри `didSet` того же свойства (защита от рекурсии), при установке значения через `inout`.

**Можно ли добавить property observer к `let` свойству?**
> Нет. Observer имеет смысл только для изменяемых (`var`) свойств. `let` не меняется после инициализации — добавить `willSet`/`didSet` к нему нельзя, компилятор выдаст ошибку.

**Что такое global variable и в чём её отличие от local?**
> Global — объявлена на уровне модуля, видна везде. Local — внутри функции/блока, видна только там. Глобальные в Swift ленивые (lazy) и thread-safe по умолчанию, инициализируются один раз при первом обращении.

---

### Senior

**Как работает memory layout для хранения переменных в Swift?**
> Value types на стеке (если не escape), reference types — на куче. Размер определяется через `MemoryLayout<T>.size`. Компилятор может оптимизировать layout для выравнивания. Struct — непрерывный блок, содержащий поля.

**Почему `let` массивы в Swift — value semantic, а не reference?**
> `Array` реализован как struct с указателем на буфер в куче. COW (Copy-on-Write) делает его эффективным: копирование буфера происходит только при мутации. `let` запрещает мутацию → реального копирования не будет.

**Как компилятор оптимизирует неиспользуемые переменные?**
> При оптимизациях (Release) компилятор удаляет код, результат которого нигде не используется (dead code elimination). `-Onone` — без оптимизаций, все переменные хранятся. `-O` / `-Osize` — агрессивное удаление.

**Что такое `@discardableResult` и когда его применять?**
> Атрибут, подавляющий предупреждение компилятора когда результат функции не используется. Применять когда результат опционален по смыслу (например, `@discardableResult func save() -> Bool`).

**Как работает property observer в контексте наследования?**
> Дочерний класс может добавить `willSet`/`didSet` к унаследованному свойству через `override`. При этом вызываются оба observer — родительского и дочернего класса.

**Что произойдёт с didSet если изменить значение внутри самого didSet?**
> Swift защищает от бесконечной рекурсии: изменение значения внутри `didSet` **не** вызовет `didSet` повторно. Но `willSet` тоже не будет вызван. Значение просто обновится.

**Почему `lazy var` не может быть `let`?**
> Инициализация `lazy` происходит при первом обращении, то есть после создания объекта. `let` требует инициализации в `init`. Эти требования несовместимы. Кроме того, lazy подразумевает возможность "дозаписи" при первом доступе.

**Как Swift обрабатывает глобальные переменные (thread safety)?**
> Глобальные переменные инициализируются через `dispatch_once` аналог — один раз, thread-safely. Это гарантия Swift runtime. Сами значения не защищены от concurrent access — это задача разработчика.

**Что такое stored property с accessor?**
> Computed property с get и set. Физически ничего не хранит — get вычисляет значение, set записывает. Отличается от stored property + didSet тем, что нет реального хранилища.

**В чём разница между `@State` и обычной переменной в SwiftUI с точки зрения хранения?**
> Обычная переменная в struct-View уничтожается при каждом пересоздании View. `@State` хранится **вне** структуры — SwiftUI управляет хранилищем, и значение переживает пересоздание View.

---

## Memory Layout

### Beginner

**Что такое стек (stack) и куча (heap) в контексте памяти?**
> Stack — быстрая, LIFO-память для локальных переменных, управляется автоматически (push/pop). Heap — динамическая память для объектов с неизвестным временем жизни, управляется ARC. Stack быстрее, Heap гибче.

**Где хранятся value types в Swift?**
> Преимущественно на стеке — когда размер известен на этапе компиляции и объект не "убегает" из области видимости. Но если value type захвачен замыканием или хранится в классе — переезжает на кучу.

**Где хранятся reference types в Swift?**
> Всегда на куче. При создании объекта класса выделяется память на куче, а переменная хранит лишь указатель (8 байт на 64-бит) на этот объект.

**Что такое выравнивание памяти (alignment)?**
> Требование чтобы адрес начала типа в памяти был кратен определённому числу. `Int64` требует выравнивания по 8 байт. Нарушение выравнивания приводит к ошибке или снижению производительности.

**Сколько байт занимает `Bool` в Swift?**
> 1 байт (`MemoryLayout<Bool>.size == 1`). Хотя логически нужен лишь 1 бит, для выравнивания используется минимальная адресуемая единица — байт.

**Сколько байт занимает `Int` на 64-битной платформе?**
> 8 байт (64 бита). `Int` в Swift платформозависим: 4 байта на 32-битных, 8 байт на 64-битных устройствах. Для гарантированного размера используй `Int32`, `Int64`.

---

### Middle

**Как можно узнать размер типа в Swift?**
> Через `MemoryLayout<T>.size` — фактический размер в байтах. Также `MemoryLayout<T>.stride` — шаг в массиве (с учётом выравнивания), `MemoryLayout<T>.alignment` — требование выравнивания.

**Чем отличается `MemoryLayout<T>.size` от `MemoryLayout<T>.stride`?**
> `size` — реальное количество байт данных типа. `stride` — расстояние между элементами в массиве (size + padding для выравнивания). Для `Bool`: size=1, stride=1. Для struct с `Int8` и `Int64`: size=9, stride=16.

**Что такое `MemoryLayout<T>.alignment`?**
> Минимальное выравнивание адреса в памяти для типа T. Адрес переменной должен быть кратен этому числу. `Int64` — alignment=8, `Int8` — alignment=1, `Double` — alignment=8.

**Как enum с associated values хранится в памяти?**
> Содержит тег (discriminator, обычно 1 байт) для определения активного case, плюс буфер размером с наибольший associated value. Итоговый размер = max(associated values) + тег + выравнивание.

**Как struct хранится в памяти — всегда ли на стеке?**
> Нет. По умолчанию на стеке, но переезжает на кучу если: захвачен escaping-замыканием, хранится как свойство класса, используется через протокол (existential container), слишком большой для inline.

**Что такое existential container и как он устроен в памяти?**
> 40-байтная структура для хранения значения protocol type: 3 слова (24 байта) — value buffer или указатель на кучу, 1 слово — pointer на Value Witness Table, 1 слово — pointer на Protocol Witness Table(s).

**Как протоколы влияют на размер типа?**
> Конкретный тип — размер остаётся. Хранение как `any Protocol` — оборачивается в existential container (40 байт). Это boxing увеличивает размер и добавляет heap allocation для крупных типов.

**Что такое inline storage в existential containers?**
> Первые 3 слова (24 байта) existential container используются для хранения значения напрямую (inline), если оно помещается. Для больших значений — heap allocation, а buffer хранит указатель.

**Как Optional хранится в памяти?**
> `Optional<T>` — enum с двумя case: `.none` и `.some(T)`. Компилятор оптимизирует: для Reference types `Optional` не занимает дополнительного места (nil = нулевой указатель). Для value types — добавляется 1 байт тега.

**Что такое tagged pointer в Objective-C и есть ли аналог в Swift?**
> ObjC хранит маленькие объекты (NSNumber, NSDate) прямо в указателе, используя свободные биты адреса. Swift использует похожую оптимизацию для `Optional` reference types и некоторых enum.

---

### Senior

**Как Swift оптимизирует хранение enum с одним case без associated value?**
> Такой enum эквивалентен `Void` или empty struct — занимает 0 байт. Компилятор может полностью устранить его из рантайма.

**Почему large struct может всё равно храниться на куче?**
> При захвате escaping-замыканием struct копируется на кучу. При хранении в классе — живёт на куче вместе с классом. При использовании как `any Protocol` — boxing в existential container.

**Что такое copy elision и как Swift его применяет?**
> Оптимизация: вместо копирования значения компилятор конструирует его сразу в нужном месте. Swift применяет это при возврате значения из функции — избегает временной копии (NRVO, Named Return Value Optimization).

**Как работает memory layout для generic types?**
> Generic типы специализируются компилятором (monomorphization) — каждый `Array<Int>` и `Array<String>` имеет свой layout. Без специализации — indirect через Value Witness Table для определения размера и операций.

**Что такое `withUnsafePointer(to:)` и когда это нужно?**
> Предоставляет unsafe указатель на значение на время выполнения замыкания. Нужно для interop с C API, которые требуют сырые указатели. После замыкания поведение указателя не определено.

**Как устроен vtable для Swift классов?**
> Vtable — массив указателей на функции, по одному на каждый overridable метод. Хранится в метаданных типа. При виртуальном вызове: загружается указатель на vtable → смещение по индексу метода → вызов.

**Что такое witness table для протоколов?**
> Protocol Witness Table (PWT) — аналог vtable для протоколов. Содержит указатели на реализации всех требований протокола для конкретного типа. Value Witness Table (VWT) — операции с памятью (copy, destroy, size).

**Как Swift хранит функции (closure vs function pointer)?**
> Простая функция — "thin" pointer (1 слово). Замыкание — "thick" pointer (2 слова): указатель на код + указатель на контекст захваченных переменных. Контекст — ref-counted объект на куче.

**В чём разница между value buffer и heap allocation для existential?**
> Value buffer — 3 слова в existential для inline хранения малых типов (≤24 байта). Если тип больше — выделяется heap allocation, а в buffer хранится указатель. Heap allocation добавляет ref-counting overhead.

**Как инструмент Instruments помогает анализировать memory layout?**
> Allocations instrument показывает heap allocations по типам и размерам. Memory Graph Debugger — граф объектов с размерами. Leaks — неосвобождённые объекты. Позволяет найти неожиданный boxing и лишние allocations.

---

### Staff

**Как Swift ABI влияет на бинарную совместимость и memory layout?**
> Стабильный ABI (Swift 5.0+) гарантирует что layout типов из stdlib не меняется между версиями Swift. Это позволяет использовать предкомпилированные stdlib без перекомпиляции зависимостей.

**Что такое resilience boundaries в Swift и как они влияют на layout?**
> Граница между модулями, где layout типа может быть неизвестен на этапе компиляции. Внутри модуля — прямой доступ к полям. Через boundary — через accessor функции, чтобы изменение layout не сломало клиентов.

**Как library evolution mode меняет подход к memory layout?**
> `-enable-library-evolution` — компилятор не встраивает layout типов в клиентский код, а использует indirect access. Это позволяет менять layout в новых версиях библиотеки без перекомпиляции клиентов (ценой производительности).

**Что такое `@_fixed_layout` и `@frozen` и чем они отличаются?**
> `@frozen` (публичный API) — гарантирует неизменность layout типа, позволяет клиентам встраивать размер и поля напрямую. `@_fixed_layout` — внутренний атрибут с тем же смыслом. `@frozen enum` — клиент может exhaustively switch без `@unknown default`.

**Как Swift на Apple Silicon использует Tagged Pointer Optimization?**
> На ARM64e Apple использует pointer authentication (PAC). Swift и ObjC runtime используют свободные биты в адресном пространстве для хранения метаданных прямо в указателе, экономя heap allocations для малых объектов.

**Как устроен runtime dispatch в Swift по сравнению с Objective-C?**
> ObjC: `objc_msgSend` — поиск по имени метода в хеш-таблице isa цепочки, динамически разрешается. Swift: vtable — прямое смещение в таблице, статически известное. Swift значительно быстрее, ObjC гибче (swizzling, forward).

---

## Value vs Reference Types

### Beginner

**Что такое value type и reference type?**
> Value type — при присваивании создаётся независимая копия (struct, enum, tuple). Reference type — при присваивании копируется ссылка, оба указывают на один объект в памяти (class, closure).

**Какие типы в Swift являются value types?**
> `struct`, `enum`, `tuple`. Все примитивы: `Int`, `Double`, `Bool`, `String`. Коллекции: `Array`, `Dictionary`, `Set`. Большинство типов Swift Standard Library.

**Какие типы в Swift являются reference types?**
> `class`, `closure` (захватывающие переменные), `function`. Все типы из Foundation/UIKit/AppKit — NSObject subclasses. Акторы (`actor`) — тоже reference type.

**Что происходит при присвоении value type новой переменной?**
> Создаётся полная независимая копия. Изменение копии не влияет на оригинал и наоборот. Для коллекций с COW реальное копирование откладывается до момента мутации.

**Что происходит при присвоении reference type новой переменной?**
> Копируется только указатель. Обе переменные указывают на один объект в памяти. Изменение через одну переменную видно через другую. ARC увеличивает retain count.

**Почему String в Swift — value type?**
> Для предсказуемого поведения и thread safety. Каждое присвоение — независимая копия, нет неожиданных side effects. COW делает это эффективным: реальное копирование только при мутации.

---

### Middle

**Что такое copy-on-write (COW) и как оно работает?**
> Оптимизация: при присвоении копирование не происходит сразу — оба экземпляра разделяют внутренний буфер. При попытке мутации одного из них — проверяется уникальность ссылки (`isKnownUniquelyReferenced`). Если не уникальна — создаётся копия буфера.

**Какие стандартные типы Swift реализуют COW?**
> `Array`, `Dictionary`, `Set`, `String`, `Data` (Foundation). Все стандартные коллекции. Пользовательские struct — COW не автоматически, нужно реализовывать вручную.

**Как реализовать COW для собственного типа?**
> Завернуть mutable state в reference type (класс). В mutating методах проверять `isKnownUniquelyReferenced(&_storage)` — если `false`, создавать копию storage перед мутацией.

**Когда происходит фактическое копирование при COW?**
> При мутации (запись) когда retain count внутреннего буфера > 1. Если ты единственный владелец — мутация in-place без копирования. Чтение никогда не вызывает копирование.

**Как проверить, является ли ссылка уникальной (`isKnownUniquelyReferenced`)?**
> `isKnownUniquelyReferenced(&myObject)` — возвращает `true` если retain count == 1. Работает только с Swift native классами (не NSObject). Используется внутри COW реализаций.

**Чем struct лучше class с точки зрения thread safety?**
> Value type: каждый поток работает со своей копией — нет shared mutable state, нет data races по умолчанию. Class: все потоки могут читать/писать один объект — нужна синхронизация.

**Можно ли реализовать протокол с mutating методом на class?**
> Да, но ключевое слово `mutating` не нужно для класса (классы всегда mutable). Если протокол требует `mutating func`, class реализует его без `mutating`.

**Что такое mutating func и зачем нужен этот модификатор?**
> Для struct и enum метод по умолчанию не может изменять `self`. `mutating` явно разрешает мутацию self и его свойств. Swift подменяет self целиком после выполнения метода.

**Как value type ведёт себя внутри замыкания?**
> Захватывается по значению (копируется) в момент создания замыкания. Изменения внутри замыкания не видны снаружи и наоборот. Если нужно разделить состояние — используй `inout` или reference type.

**Почему Array в Swift — value type, но быстрый как reference type?**
> За счёт COW. Передача массива — копирование только 3 слов (pointer + size + capacity). Реальное дорогостоящее копирование буфера только при мутации, и только если есть другие владельцы.

---

### Senior

**Как Swift определяет когда нужно скопировать value type?**
> Компилятор делает escape analysis: если value type не "убегает" из области видимости (не передаётся escaping-замыканию, не сохраняется в классе) — остаётся на стеке, копирования нет. При передаче по значению в функцию — stack copy, дёшево.

**В каких ситуациях struct может вести себя как reference type?**
> Если struct содержит reference type свойство — это свойство разделяется между копиями. Два экземпляра struct будут иметь разные копии примитивов, но одинаковую ссылку на class-объект.

**Как nested reference type влияет на COW?**
> COW не распространяется автоматически на вложенные reference types. При копировании struct копируются ссылки на объекты, не сами объекты. Нужно реализовывать deep copy вручную если требуется полная изоляция.

**Что такое indirect enum и зачем он нужен?**
> `indirect enum` — associated values хранятся на куче, а не напрямую. Нужен для рекурсивных структур (linked list, tree), где enum содержит свой же тип. Без `indirect` — бесконечный размер в compile time.

**Как компилятор оптимизирует передачу крупных struct?**
> При передаче крупного struct компилятор может оптимизировать до передачи указателя (guaranteed by Swift calling conventions). Receiving side может использовать значение напрямую без копирования (copy elision, NRVO).

**Что такое in-place mutation optimization?**
> Если функция принимает value type и мутирует его — компилятор может оптимизировать чтобы mutation происходила in-place без создания копии, если доказана уникальность. Обычно через `inout`.

**Как выбор между struct и class влияет на производительность при многопоточности?**
> Struct: каждый поток со своей копией — нет lock contention, нет синхронизации. Быстрее при read-heavy workload. Class: shared state — нужны locks или actor, overhead синхронизации. Зато экономия памяти при большом shared state.

**Как value semantic помогает тестировать код?**
> Value types легко тестировать: нет скрытых зависимостей через shared references, нет нужды в mock объектах для изоляции, детерминированное поведение. Каждый тест получает свою независимую копию состояния.

**В чём разница между семантикой значения и ссылки в контексте SwiftUI?**
> SwiftUI View — struct (value semantic): каждый rebuild создаёт новый экземпляр, SwiftUI сравнивает и рендерит diff. `@State`, `@ObservedObject` — reference semantic хранилища, переживающие rebuild. Value семантика View ускоряет diff-алгоритм.

**Как actor model в Swift Concurrency связан с value/reference semantics?**
> Actor — reference type, но защищённый: внешний доступ к mutable state только через async вызовы (один за раз). `Sendable` protocol маркирует типы безопасные для передачи между акторами: value types автоматически Sendable, классы — нет.

---

## Struct

### Beginner

**Что такое struct в Swift?**
> Value type для группировки связанных данных и поведения. Копируется при присвоении. Не поддерживает наследование. Рекомендован Apple для большинства пользовательских типов.

**Какие члены может содержать struct?**
> Stored properties, computed properties, methods, subscripts, initializers, static properties/methods. Не может иметь deinit и не поддерживает наследование.

**Что такое memberwise initializer?**
> Автоматически генерируемый компилятором initializer, принимающий все stored properties в качестве параметров. `struct Point { var x, y: Int }` → автоматически `Point(x:y:)`. Исчезает при добавлении кастомного init в основном теле.

**Может ли struct наследовать другой struct?**
> Нет. Structs не поддерживают наследование вообще. Для переиспользования кода — composition (включение struct как свойства) или протоколы с default implementation.

**Может ли struct реализовывать протоколы?**
> Да, это основной механизм полиморфизма для struct. Struct может реализовывать любое количество протоколов. `struct Point: Hashable, Codable, CustomStringConvertible { ... }`

**Что означает `mutating` перед методом struct?**
> Разрешает методу изменять `self` и его stored properties. Без `mutating` метод компилируется как неизменяющий. При вызове `mutating` на `let` экземпляре — ошибка компилятора.

---

### Middle

**Чем struct отличается от class в Swift?**
> Struct: value type, копирование при присвоении, стек (обычно), нет наследования, нет deinit, нет ARC overhead, thread-safe по умолчанию. Class: reference type, ссылка при присвоении, куча, наследование, deinit, ARC.

**Когда предпочесть struct вместо class?**
> Apple рекомендует struct по умолчанию. Выбирай struct если: не нужно наследование, нужна value semantics, нет identity, нужен thread safety без синхронизации. Class — для ObjC interop, shared mutable state, identity важна.

**Может ли struct содержать reference type свойства?**
> Да. `struct Person { var name: String; var image: UIImage? }`. При копировании struct — примитивы копируются, ссылки на class-объекты (UIImage) разделяются между копиями (shallow copy).

**Как COW влияет на работу struct?**
> Стандартные коллекции внутри struct не копируются при копировании struct — они используют COW. Реальное копирование буфера коллекции происходит только при мутации одного из экземпляров.

**Что такое recursive struct и почему он невозможен напрямую?**
> Struct, содержащий себя как поле: `struct Node { var next: Node }`. Невозможен — бесконечный размер. Решение: `var next: Node?` (Optional делает размер конечным) или `class Node` для reference semantics.

**Как реализовать кастомный initializer не теряя memberwise?**
> Добавить кастомный init в extension, не в основном объявлении struct. Компилятор синтезирует memberwise только когда в основном теле нет кастомного init.

**Может ли struct реализовать Identifiable?**
> Да. `struct User: Identifiable { var id: UUID; var name: String }`. UUID обеспечивает уникальную идентичность, что важно для SwiftUI List и ForEach.

**Как Swift автоматически синтезирует Equatable для struct?**
> Если все stored properties соответствуют `Equatable` — компилятор автоматически генерирует `==` оператор, сравнивающий все поля. Достаточно объявить conformance: `struct Point: Equatable { var x, y: Int }`.

**Что такое static stored property в struct?**
> Свойство, принадлежащее типу, а не экземпляру. Одно на весь тип, хранится в специальной области памяти. `static var count = 0`. Инициализируется лениво и thread-safely.

**Почему SwiftUI рекомендует использовать struct для View?**
> Struct-View: лёгкие для создания и копирования, SwiftUI может эффективно сравнивать и делать diff, нет управления памятью. Каждый rebuild — новый экземпляр, SwiftUI сам решает что перерисовывать.

---

### Senior

**Как Swift хранит struct в памяти при передаче в функцию?**
> По умолчанию — копирование на стеке (pass by value). Компилятор может оптимизировать крупные structs до передачи указателя (без реального копирования). `inout` — передаёт адрес напрямую.

**Что такое `@inlinable` и как оно влияет на struct?**
> `@inlinable` публикует тело функции в binary interface, позволяя клиентскому коду встраивать её при компиляции. Для struct методов — клиент может оптимизировать работу со struct без вызова в библиотеку.

**Как struct ведёт себя при захвате в замыкании?**
> Захватывается по значению в момент создания замыкания — копия. Изменения после захвата не видны замыканию. Если нужно разделять изменения — используй `inout` или помести в class-обёртку.

**Что такое phantom type и как его реализовать через struct?**
> Generic struct с неиспользуемым type parameter для compile-time проверки: `struct Validated<T> { let value: String }`. `Validated<Email>` и `Validated<Username>` — разные типы, хотя содержат String. Предотвращает перепутывание на уровне типов.

**Как struct помогает избежать retain cycle?**
> Value types не участвуют в ARC, нет retain cycles. Если delegate или callback хранится в struct — нет сильных ссылок. Но если struct содержит class — эта class-ссылка участвует в ARC как обычно.

**Что происходит с struct при передаче через generic protocol?**
> Зависит от контекста: при конкретном generic (`func f<T: Equatable>(_ t: T)`) — специализация, прямой вызов. При `any Protocol` — boxing в existential, heap allocation, виртуальная диспетчеризация.

**Как динамическая диспетчеризация отличается для struct и class?**
> Struct: статическая диспетчеризация (compile-time) для конкретных типов — быстро, inline возможен. Class: динамическая через vtable — накладные расходы. Через протокол (any): witness table — тоже динамическая.

**Что такое stack promotion и когда struct остаётся на стеке?**
> Компилятор оставляет struct на стеке если: размер известен, не захвачен escaping-замыканием, не хранится в class, не используется как any Protocol. Stack allocation значительно быстрее heap.

**Как использовать struct для реализации State Machine?**
> Enum для состояний + struct для машины с методом `mutating func transition(event:)`. Или struct с computed property `state` и логикой переходов. Value semantics гарантируют предсказуемость без side effects.

**Как struct взаимодействует с ObjC runtime?**
> Нативные Swift structs недоступны в ObjC. Для interop нужно: обернуть в class, или использовать `@objc` (только для классов). Исключение: Swift struct может реализовывать ObjC protocol через `@objc` class-адаптер.

---

## Class

### Beginner

**Что такое class в Swift?**
> Reference type для сложных объектов с идентичностью, наследованием и разделяемым состоянием. Хранится на куче, управляется ARC. При присвоении копируется ссылка, а не объект.

**Чем class отличается от struct?**
> Class: reference type, наследование, deinit, ARC, heap, identity (===). Struct: value type, нет наследования, нет deinit, нет ARC, stack/heap, нет identity. Struct быстрее и безопаснее для однопоточного кода.

**Что такое наследование в Swift?**
> Класс-потомок наследует свойства, методы и initializers родителя. Swift поддерживает только одиночное наследование. `class Dog: Animal { }`. Потомок может override методы и добавлять новые.

**Что такое `override` ключевое слово?**
> Явно указывает что метод, свойство или subscript переопределяет реализацию из родительского класса. Обязателен — без него компилятор выдаст ошибку. Защищает от случайного перекрытия.

**Что такое `final` class?**
> Класс, который нельзя наследовать. `final class ViewController { }`. Методы `final` классов диспатчатся статически — быстрее vtable. Явный сигнал архитектурного решения.

**Что такое `super` в Swift?**
> Ссылка на родительский класс в контексте методов и инициализаторов потомка. `super.viewDidLoad()` — вызов родительской реализации. `super.init(frame:)` — обязательная цепочка инициализаторов.

**Как создать экземпляр класса?**
> `let object = MyClass()` или `let object = MyClass(parameter: value)`. В отличие от struct — нет автоматического memberwise initializer, нужно объявить `init` явно (или использовать значения по умолчанию).

**Что такое `deinit`?**
> Деструктор класса — вызывается непосредственно перед освобождением памяти объекта (когда retain count достигает 0). Место для cleanup: закрыть файлы, отменить подписки, освободить ресурсы.

---

### Middle

**Когда вызывается `deinit`?**
> Когда ARC определяет что retain count объекта стал 0 — все strong ссылки на него исчезли. Не гарантируется точный момент в многопоточной среде. Не вызывается если есть retain cycle.

**Что такое designated initializer и convenience initializer?**
> Designated — полностью инициализирует все stored properties, вызывает `super.init`. Convenience — вспомогательный, должен вызвать другой init (в конечном счёте designated) через `self.init`. Convenience помечается ключевым словом.

**Как работает цепочка инициализаторов в иерархии классов?**
> Designated init должен вызвать designated init родителя (`super.init`). Convenience init должен вызвать другой init того же класса (`self.init`). Гарантирует полную инициализацию перед использованием.

**Что такое two-phase initialization в Swift?**
> Фаза 1: все stored properties инициализируются снизу вверх по иерархии. Фаза 2: customization properties сверху вниз. Между фазами нельзя вызывать методы и обращаться к self — safety guarantee.

**Зачем нужен `required init`?**
> Гарантирует что все подклассы реализуют этот инициализатор. `required init(coder:)` — для NSCoding. При subclassing нужно явно реализовать с `required` модификатором, нельзя просто унаследовать.

**Что такое `@objc` и `dynamic` для класса?**
> `@objc` — экспортирует метод в ObjC runtime, позволяя использовать из ObjC кода. `dynamic` — форсирует динамическую диспетчеризацию через ObjC runtime. Нужно для KVO, method swizzling.

**Как работает `isKind(of:)` и `isMember(of:)`?**
> `isKind(of: Animal.self)` — true если объект этого класса **или любого подкласса**. `isMember(of: Animal.self)` — true только если объект **точно** этого класса (не подкласса). Аналог Swift — `is` оператор (как isKind).

**Что такое `type(of:)` в контексте полиморфизма?**
> Возвращает динамический тип объекта в рантайме. `type(of: animal)` для `Dog` вернёт `Dog.self`, даже если переменная объявлена как `Animal`. Используется для метапрограммирования.

**Как работает KVO и зачем `@objc dynamic`?**
> Key-Value Observing — механизм ObjC для наблюдения за изменениями свойств. Требует `@objc dynamic` потому что работает через ObjC runtime (method swizzling). Swift нативно предлагает `@Observable` и Combine как замену.

**Чем отличается `class var` от `static var`?**
> Оба принадлежат типу, а не экземпляру. `static var` — нельзя переопределить в подклассе. `class var` — можно переопределить (`override class var`). `static` в классах — синоним `final class`.

---

### Senior

**Как устроен vtable для Swift классов?**
> Vtable — непрерывный массив function pointers в метаданных типа. Каждый overridable метод имеет фиксированный slot. Вызов метода: `object.metadata.vtable[methodSlot]()`. Подкласс копирует vtable родителя и заменяет нужные slots.

**Что такое message passing в ObjC и как Swift от него отличается?**
> ObjC: `[obj method]` → `objc_msgSend` → поиск реализации по имени (`SEL`) в хеш-таблице `isa`-цепочки, динамически в рантайме. Swift: vtable lookup по числовому offset, статически известному на этапе компиляции. Swift в разы быстрее.

**Как работает `NSObject` subclassing в Swift?**
> NSObject даёт ObjC runtime базу: isa pointer, ref counting через ObjC, KVO, KVC поддержку. Swift методы на NSObject subclass — mixed: часть через vtable, часть через ObjC dispatch если `@objc`.

**Как `@objc` влияет на производительность?**
> `@objc` форсирует ObjC dynamic dispatch вместо Swift vtable. ObjC dispatch медленнее: хеш-таблица lookup, кешируется, но cache miss дорог. Без `@objc` — быстрый vtable. С `@objc` — overhead ~10-50ns на вызов.

**Что такое existential открытый тип и как он работает с классами?**
> `any Animal` — existential, тип неизвестен в compile time. `open(value as: any Animal)` (Swift 5.7) — открывает existential, давая доступ к конкретному типу внутри замыкания. Позволяет вызывать методы с Self-требованиями.

**Как работает deallocation для объектов с weak references?**
> При retain count → 0: объект deinitialized, но память не освобождается сразу. ARC обнуляет все weak references (через side table) и только потом освобождает память. Это гарантирует что weak refs не станут dangling pointers.

**Что такое `unmanaged` и когда он используется?**
> `Unmanaged<T>` — оборачивает reference type без ARC управления. Нужен для C API где передаёшь/получаешь указатель вручную. `passRetained` — +1 retain, `passUnretained` — без retain. `takeRetainedValue()` — -1 retain при получении.

**Как работает `objc_msgSend` под капотом?**
> Assembly функция: 1) загружает isa из первых байт объекта, 2) lookup в method cache isa-цепочки по selector (хеш), 3) при cache miss — поиск по всей иерархии, кеширование, 4) прыжок на IMP (implementation pointer).

**Как Swift interoperability влияет на дизайн классов?**
> Классы для ObjC interop должны наследоваться от NSObject, использовать `@objc` для видимых методов. Это ограничивает: нет generics в ObjC, нет Swift-only типов в сигнатурах, принудительный ObjC dispatch overhead.

**Что такое class cluster pattern и где Apple его применяет?**
> Паттерн: публичный абстрактный класс + приватные конкретные подклассы. Клиент работает только с абстрактным интерфейсом. Apple: `NSArray` (abstract) → `__NSArrayI`, `__NSArrayM` (concrete). `NSString` → несколько конкретных реализаций.

---

## Enum

### Beginner

**Что такое enum в Swift?**
> Value type для набора связанных именованных значений. Более мощный чем в других языках: поддерживает associated values, методы, computed properties, протоколы. Часто используется для State Machine и Error типов.

**Что такое raw value у enum?**
> Предустановленное значение конкретного типа для каждого case. `enum Direction: String { case north = "N" }`. Raw value должны быть уникальными. Доступны через `.rawValue` property и `init?(rawValue:)`.

**Какие типы могут быть raw value?**
> `Int`, `Double`, `String`, `Character` — и любой тип соответствующий `RawRepresentable`. Для `Int` — автоинкремент с 0. Для `String` — имя case как строка по умолчанию.

**Что такое associated value?**
> Дополнительные данные, прикреплённые к конкретному case: `case loading(progress: Double)`, `case error(Error)`. Разные case могут иметь разные типы. Доступны через pattern matching в switch.

**Как использовать switch с enum?**
> `switch direction { case .north: ... case .south: ... }`. Switch должен быть exhaustive (покрывать все case) или иметь `default`. С associated values: `case .loading(let progress):`.

**Что такое `CaseIterable`?**
> Протокол, добавляющий свойство `allCases: [Self]` — коллекцию всех case enum. Компилятор автоматически синтезирует для enum без associated values. Полезно для UI пикеров, тестов.

**Как создать enum с String raw value?**
> `enum Status: String { case active = "active"; case inactive = "inactive" }`. Если имена case совпадают со значениями — можно опустить: `enum Status: String { case active, inactive }`.

---

### Middle

**Чем отличаются raw value от associated value?**
> Raw value: одно фиксированное значение для case (задаётся при объявлении), одинаковый тип для всех case. Associated value: данные "в моменте" при создании case, разные типы для разных case, не фиксированы.

**Что такое recursive enum (`indirect enum`)?**
> Enum содержащий себя как associated value. `indirect enum Tree { case leaf(Int); case node(Tree, Tree) }`. `indirect` указывает хранить associated value через указатель (heap), делая размер конечным.

**Как enum реализует протоколы?**
> Как и struct/class — через `extension` или в объявлении. `enum Color: Equatable { }` — автосинтез. Для `Codable` — автосинтез если все associated values Codable. `CaseIterable` — только без associated values.

**Как работает pattern matching с enum?**
> `switch`, `if case`, `guard case`, `for case`. `if case .loading(let p) = state { }`. Поддерживает where clause: `case .error(let e) where e.code == 404`. `~=` оператор для кастомного matching.

**Что такое `@unknown default` в switch?**
> `@unknown default` — выдаёт warning если в будущем добавится новый case enum, но не требует exhaustive обработки сейчас. Отличие от `default` — компилятор предупреждает о непокрытых случаях.

**Как enum используется для моделирования состояния (State Machine)?**
> `enum LoadingState { case idle; case loading; case success(Data); case failure(Error) }`. Каждый case — состояние, associated values — данные этого состояния. Switch exhaustive — нельзя забыть обработать состояние.

**Что такое frozen enum и non-frozen enum?**
> `@frozen enum` — не добавит новых case в будущих версиях библиотеки, клиент может switch без `@unknown default`. Non-frozen (default для public в library evolution mode) — могут добавиться case, требуется `@unknown default`.

**Как автоматически синтезировать Equatable и Hashable для enum?**
> Объявить conformance — компилятор генерирует автоматически: `enum Color: Equatable, Hashable { case red, green, blue }`. Для enum с associated values — все они должны быть Equatable/Hashable.

**Как хранятся enum с associated values в памяти?**
> Discriminator (тег, обычно 1 байт) + буфер размером с наибольший associated value. Если associated value влезает в 8 байт — компилятор может упаковать тег вместе с данными. Размер: `MemoryLayout<MyEnum>.size`.

**Можно ли добавить stored property к enum?**
> Нет, только computed. Stored properties невозможны — enum не имеет фиксированного инициализатора всех свойств. Можно: computed property, static stored property (принадлежит типу), методы, инициализаторы.

---

### Senior

**Как enum с multiple associated values влияет на memory layout?**
> Размер = max из размеров всех associated value tuples + тег. Компилятор пытается упаковать тег в spare bits (свободные биты указателей или в Optional). `MemoryLayout<Enum>.size` покажет итоговый размер.

**Как использовать enum для type-safe URL routing?**
> `enum Route { case home; case profile(userId: Int); case settings }`. `func url(for route: Route) -> URL { switch route { ... } }`. Невозможно создать невалидный URL, рефакторинг безопасен — компилятор поймает все места.

**Что такое sealed class аналог в Swift через enum?**
> Swift enum с associated values — прямой аналог sealed class Kotlin. `enum Result<T> { case success(T); case failure(Error) }`. Exhaustive switch гарантирует обработку всех вариантов — то же что sealed class даёт.

**Как enum помогает заменить boolean blindness?**
> Вместо `func process(isActive: Bool, isAdmin: Bool)` → `enum UserRole { case active, inactive, admin }`. Невозможна ошибка перепутать параметры. `process(role: .admin)` читается как документация.

**Как реализовать State Machine через enum + протоколы?**
> Протокол State с методом `transition(event:) -> State`. Enum реализует протокол, каждый case — состояние. `switch self { case .idle: return event == .start ? .running : self }`. Compile-time проверка полноты переходов.

**Как работает exhaustive pattern matching и почему это важно?**
> Компилятор требует покрыть все case enum в switch — нельзя случайно пропустить состояние. При добавлении нового case — ошибка компиляции во всех switch без `default`. Это делает код поддерживаемым.

**Как создать generic enum?**
> `enum Result<Success, Failure: Error> { case success(Success); case failure(Failure) }`. Generic параметры указываются как type parameters. Swift стандартный `Result` именно так и реализован.

**Что такое discriminated union и как это реализовано в Swift enum?**
> Tagged union из функциональных языков — тип, который может быть одним из нескольких вариантов, каждый с собственными данными. Swift enum с associated values — прямая реализация discriminated union с тегом (discriminator).

**Как enum связан с Result type?**
> `Result<Success, Failure>` — enum с двумя case: `.success(Success)` и `.failure(Failure)`. Позволяет вернуть или значение, или ошибку из функции, моделировать async результаты, chain через `map`/`flatMap`.

**Как enum ведёт себя при ObjC interop?**
> Swift enum с raw value Int/String может быть `@objc`: `@objc enum Direction: Int { case north, south }`. Без raw value — не может быть `@objc`. Associated values — не поддерживаются в ObjC вообще.

---

## Protocol

### Beginner

**Что такое протокол в Swift?**
> Контракт, определяющий набор методов, свойств и требований. Не содержит реализации (но может через extension). Тип "обещает" предоставить реализацию. Основа Protocol-Oriented Programming в Swift.

**Как объявить протокол?**
> `protocol Drawable { func draw(); var color: Color { get } }`. Properties в протоколе указывают только get/set требования, без реализации. Методы — только сигнатуры.

**Как класс или struct реализует протокол?**
> `struct Circle: Drawable { func draw() { ... }; var color: Color = .red }`. Тип обязан реализовать все требования протокола. Partial implementation — ошибка компилятора.

**Что такое protocol conformance?**
> Соответствие типа протоколу — когда тип реализует все его требования. Можно в объявлении типа или через extension. `extension Circle: Equatable { }`.

**Можно ли протоколу иметь default implementation?**
> Да, через extension: `extension Drawable { func describe() { print("Drawing") } }`. Типы могут не реализовывать метод — используется default. Но могут переопределить.

**Что такое `@objc protocol`?**
> Протокол видимый из ObjC кода. Все требования становятся `@objc`. Может иметь optional requirements (`@objc optional func method()`). Реализовывать могут только классы (и NSObject subclass обычно).

---

### Middle

**Что такое protocol extension?**
> Extension на протокол, добавляющий default implementations методов и computed properties. `extension Collection { var isNotEmpty: Bool { !isEmpty } }`. Все типы соответствующие Collection автоматически получают `isNotEmpty`.

**Чем отличается protocol с default implementation от abstract class?**
> Protocol: нет наследования, множественная conformance, value types поддерживают. Abstract class: наследование, одиночное, только reference types, stored properties. Protocol + extension ≈ abstract class, но гибче и для value types.

**Что такое associated type в протоколе?**
> Placeholder для типа, который определяется при conformance: `protocol Container { associatedtype Item; func add(_ item: Item) }`. Конкретный тип Item определяется реализующим типом (inference или typealias).

**Что такое `Self` в протоколе?**
> Ссылка на конкретный тип соответствующий протоколу. `protocol Copyable { func copy() -> Self }`. Для class — может быть подкласс. Наличие Self или associated type делает протокол PAT (Protocol with Associated Type).

**Почему нельзя использовать протокол с associated type напрямую как тип переменной?**
> `var container: Container` — ошибка. Компилятор не знает Item в compile time, не может определить размер и layout. Решения: generics (`<T: Container>`), `any Container` (Swift 5.7+, boxing), или type erasure.

**Что такое protocol composition (`&`)?**
> Создание типа-пересечения нескольких протоколов: `func process(_ value: Codable & Hashable)`. Тип должен соответствовать обоим протоколам. Typealias: `typealias CodableHashable = Codable & Hashable`.

**Что такое conditional conformance?**
> Conformance к протоколу только при выполнении условий: `extension Array: Equatable where Element: Equatable`. `[Int]` — Equatable, `[UIView]` — нет. Мощный инструмент для generic кода.

**Как работает protocol witness table?**
> PWT — таблица функциональных указателей для каждого требования протокола. При вызове метода через протокол: lookup в PWT → вызов конкретной реализации. Аналог vtable для протоколов, но для любых типов.

**Что такое retroactive conformance?**
> Добавление conformance к стороннему типу из стороннего модуля. `extension UIColor: Codable { }`. Риск: конфликт если оба модуля добавят одинаковую conformance. Swift 5.7 требует `@retroactive` атрибут.

**Чем отличается `class` constraint на протоколе от `AnyObject`?**
> `protocol P: class { }` (устаревший синтаксис) и `protocol P: AnyObject { }` — одно и то же: только reference types могут реализовывать протокол. Нужно для weak property types и когда нужна reference semantics.

---

### Senior

**Что такое existential type и как оно влияет на производительность?**
> `any Protocol` — existential: значение боксируется в existential container (40 байт), dynamic dispatch через PWT, heap allocation для крупных значений. Overhead: ~3-5x медленнее конкретного типа. Предпочитай generics где возможно.

**Чем отличается `any Protocol` от `some Protocol`?**
> `any Protocol` — existential: тип неизвестен в compile time, dynamic dispatch, boxing, runtime overhead. `some Protocol` — opaque type: конкретный тип известен компилятору, static dispatch, нет boxing, inline. `some` быстрее.

**Как работает dynamic dispatch для протоколов?**
> При вызове метода через `any Protocol`: 1) загрузить Protocol Witness Table из existential container, 2) найти slot метода в PWT, 3) вызвать функцию по указателю. Нельзя inline, нельзя специализировать.

**Что такое static dispatch и когда он применяется для протоколов?**
> При использовании generics (`<T: Protocol>`) — компилятор знает конкретный тип, может inline и оптимизировать. Или при `some Protocol` — тоже статически. Быстро, без PWT lookup.

**Как protocol extension methods диспатчатся?**
> Метод из extension (не в протоколе) — статически (тип конкретен при compile time). Метод из протокола + override в extension — динамически через PWT. Важно: если метод только в extension (не в протоколе) — нет override через протокол тип.

**Что такое protocol-oriented programming и каковы его ограничения?**
> Подход Apple: вместо наследования классов — composition из протоколов с default implementations. Ограничения: нет stored state в протоколах, нет super вызовов, алмазная проблема в extensions, сложный type inference с PAT.

**Как использовать protocols для dependency inversion?**
> Высокоуровневый модуль зависит от абстракции (протокола), не от конкретного типа. `class ViewModel { let service: NetworkServiceProtocol }`. В тестах — mock, в prod — реальная реализация. DIP принцип.

**Что такое phantom protocols?**
> Протоколы без требований, используемые только как маркеры для type constraints. `protocol AdminUser {}; struct User: AdminUser {}`. Compile-time проверка: `func deleteAll<T: AdminUser>(_ user: T)`.

**Как реализовать type erasure через протоколы?**
> `AnyPublisher`, `AnySequence` паттерн: wrapper-struct хранит closure-обёртки на методы. `struct AnyAnimal: Animal { private let _speak: () -> String; func speak() -> String { _speak() } }`. Скрывает конкретный тип за протоколом.

**Как работают primary associated types (SE-0346)?**
> `protocol Collection<Element> { }` — синтаксис для указания "главного" associated type. Позволяет constrained existential: `any Collection<Int>`. Более читаемо чем сложные where clauses.

---

### Staff

**Как compiler решает какой метод вызвать при конфликте в protocol extension?**
> Приоритет: 1) реализация в самом типе, 2) extension конкретного типа, 3) extension протокола. При конфликте двух protocol extensions — compile error. Тип-требования vs extension: тип выигрывает при dynamic dispatch, extension — при static.

**Что такое protocol conformance checking во время компиляции?**
> Компилятор проверяет: все ли требования реализованы, правильные ли типы, выполнены ли where constraints. Ошибки — на этапе компиляции. Conditional conformance — проверяется при каждом использовании.

**Как Swift реализует conditional conformance под капотом?**
> Компилятор генерирует специализированную PWT для каждого конкретного conditional case. `Array<Int>: Equatable` — отдельная PWT с `==` использующей `Int.==`. Проверяется и генерируется в compile time, не runtime.

**Что такое `@_marker` protocol?**
> Внутренний атрибут Swift. Маркерный протокол без requirements, используемый только для type system constraints. `Sendable` реализован как `@_marker` — нет runtime представления, только compile-time проверка.

**Как designed protocol hierarchies влияют на binary size?**
> Каждая conformance генерирует PWT. Глубокие иерархии с множеством conformances = множество таблиц. Неиспользуемые conformances могут быть устранены linker dead stripping. `@objc` protocol conformances крупнее — ObjC metadata.

**Как протоколы взаимодействуют с generics для специализации?**
> `func sort<T: Comparable>(_ array: [T])` — при компиляции с конкретным `T` создаётся специализированная версия функции (monomorphization). Устраняет PWT overhead, позволяет inline. Увеличивает binary size но ускоряет выполнение.

---

## Extensions

### Beginner

**Что такое extension в Swift?**
> Механизм добавления функциональности к существующему типу (своему или чужому) без изменения исходного кода и без наследования. Можно добавить методы, computed properties, initializers, conformances.

**Что можно добавить через extension?**
> Computed properties, methods, initializers (для struct — non-designated), subscripts, nested types, protocol conformances. Нельзя: stored properties, property observers к существующим, designated initializers (для class).

**Можно ли добавить stored property через extension?**
> Нет. Stored properties требуют фиксированного layout типа в compile time — extension не может его изменить. Обходной путь: `objc_setAssociatedObject` для NSObject subclasses или external dictionary.

**Можно ли переопределить существующий метод через extension?**
> В extension нельзя использовать `override` для методов класса. Можно добавить новый метод с другой сигнатурой. Override только в наследовании (subclass), не в extension того же класса.

**Можно ли расширять чужие типы?**
> Да, это одна из главных возможностей. `extension String { func isPalindrome() -> Bool { } }`. Риск: retroactive conformance при добавлении protocol conformance к чужому типу.

---

### Middle

**Что такое retroactive conformance и каковы её риски?**
> `extension UIColor: Codable { }` — conformance в не своём модуле. Риски: конфликт если два модуля добавят одинаковую conformance (compile error или неопределённое поведение). Swift 5.7+: нужен `@retroactive` атрибут как явное подтверждение.

**Как extension влияет на access control?**
> Extension наследует access level типа, но можно ограничить: `private extension MyType { }` — все члены extension становятся private. Можно указать уровень для каждого члена отдельно.

**Чем отличается extension на struct от extension на protocol?**
> Extension на struct — добавляет функциональность конкретному типу. Extension на protocol — добавляет default implementation всем типам conforming. Protocol extension = ad-hoc polymorphism.

**Как использовать extension для организации кода?**
> Разбить реализацию по responsibilities: `// MARK: - UITableViewDataSource`, extension для protocol conformances, extension для private helpers. Улучшает читаемость без изменения функциональности.

**Что такое `where` clause в extension?**
> Ограничивает extension на подмножество типов: `extension Array where Element: Numeric { func sum() -> Element { reduce(0, +) } }`. Только `Array<Int>` (и другие Numeric) получат `sum()`, `Array<String>` — нет.

**Что такое conditional extension?**
> Extension с where clause — применяется только при выполнении условия. `extension Collection where Element: Equatable { func containsDuplicates() -> Bool { } }`. Мощный инструмент для generic программирования.

**Как extension помогает реализовать протоколы?**
> Выносить conformance в отдельный extension: `extension ViewController: UITableViewDelegate { ... }`. Каждый протокол — свой extension. Код организован по функциональности, а не по типу декларации.

**Можно ли добавить initializer через extension к struct?**
> Да, но только non-memberwise. Главное преимущество: компилятор сохранит autogenerated memberwise initializer. `extension Point { init(same value: Int) { self.init(x: value, y: value) } }`.

**Что такое protocol extension vs default implementation?**
> Одно и то же. Default implementation — это реализация метода в extension на протокол. `extension Printable { func describe() { print(self) } }`. Тип может переопределить или использовать default.

**Как extension влияет на ObjC interop?**
> `@objc extension` — все методы видны в ObjC. Extension на NSObject subclass — методы доступны через ObjC runtime если помечены `@objc`. Нельзя добавить `@objc` метод в extension non-NSObject класса.

---

### Senior

**Как extension влияет на binary size?**
> Каждый метод в extension компилируется как отдельная функция. Неиспользуемые — могут быть устранены dead stripping при -whole-module-optimization. Extension на generic type с where — генерирует код только для используемых специализаций.

**Что такое linker dead stripping и как extension влияет на него?**
> Linker удаляет символы на которые нет ссылок. Методы в extension — отдельные символы. При `-whole-module-optimization` компилятор видит весь модуль и агрессивнее устраняет неиспользуемый код из extensions.

**Как компилятор обрабатывает методы из extension при диспатче?**
> Метод требуемый протоколом + реализация в extension = через PWT (dynamic). Метод только в extension (не в протоколе) = через static dispatch (тип известен) или vtable (если override). Extension метод на протокол без override — всегда static.

**Что такое backdeploy extension method?**
> `@backDeployed(before: iOS 16.0)` — метод доступен на старых OS через copy в клиентский binary. Компилятор встраивает реализацию в app, если OS не предоставляет native версию. Swift 5.8+.

**Как использовать extension для feature slicing по условию компиляции?**
> `#if canImport(SwiftUI); extension View { func myModifier() -> some View { } }; #endif`. Или через `#if DEBUG` для отладочных extension. Позволяет платформо-специфичную функциональность без #if везде.

**Как generic extension работает с constrained types?**
> `extension Dictionary where Value: Collection { var totalCount: Int { values.reduce(0) { $0 + $1.count } } }`. Компилятор проверяет constraint при каждом использовании. Тело extension может использовать методы из constraint.

---

## Generics

### Beginner

**Что такое generics в Swift?**
> Механизм написания гибкого, переиспользуемого кода работающего с любым типом удовлетворяющим требованиям. `func swap<T>(_ a: inout T, _ b: inout T)` работает для Int, String, любого типа. Избегает дублирования кода.

**Как объявить generic функцию?**
> `func identity<T>(_ value: T) -> T { return value }`. `<T>` — type parameter. Называть можно T, U, Element или описательно. Компилятор выводит T из аргументов при вызове.

**Что такое type parameter?**
> Placeholder для конкретного типа в generic определении. `<T>`, `<Key, Value>`. При использовании generic — заменяется конкретным типом. Может иметь constraints: `<T: Comparable>`.

**Как generics помогают избежать дублирования кода?**
> Вместо `func maxInt(_ a: Int, _ b: Int) -> Int` и `func maxDouble(_ a: Double, _ b: Double) -> Double` — одна `func max<T: Comparable>(_ a: T, _ b: T) -> T`. Одна реализация для всех типов.

**Что такое generic constraint?**
> Ограничение на type parameter: `<T: Equatable>` — T должен соответствовать Equatable. `<T: Numeric>` — T должен быть числовым. Позволяет вызывать методы протокола внутри generic функции.

---

### Middle

**Что такое `where` clause в generics?**
> Дополнительные constraints: `func equal<T, U>(_ t: T, _ u: U) -> Bool where T: Equatable, T == U`. Позволяет выражать сложные требования. `where T.Element: Hashable` — constraint на associated type.

**Что такое generic type и как его объявить?**
> `struct Stack<Element> { private var items: [Element] = [] }`. Тип параметризован. `Stack<Int>` и `Stack<String>` — разные типы. Методы получают доступ к Element как к типу.

**Как работает специализация generic функций?**
> Компилятор создаёт отдельную версию функции для каждого конкретного типа (monomorphization). `sort<Int>`, `sort<String>` — разные бинарные функции. Быстро, inline возможен. Увеличивает binary size.

**Что такое type erasure и зачем она нужна?**
> Скрытие конкретного generic типа за абстракцией. `AnyPublisher<Int, Error>` скрывает конкретную цепочку Publishers. Нужно когда нельзя вернуть или хранить PAT напрямую. Реализуется через wrapper с closures.

**Как реализовать `AnyPublisher` паттерн вручную?**
> `struct AnyAnimal: Animal { private let _speak: () -> String; init<A: Animal>(_ animal: A) { _speak = animal.speak }; func speak() -> String { _speak() } }`. Wrapper хранит closures для каждого метода протокола.

**Что такое conditional conformance в контексте generics?**
> `extension Array: Equatable where Element: Equatable`. Компилятор генерирует conformance только когда условие выполнено. При работе с `[T]` где T: Equatable — автоматически `[T]: Equatable`.

**Как generics влияют на производительность?**
> Специализированные generics — как конкретный код, без overhead. Неспециализированные (через протокол без специализации) — witness table dispatch, boxing. WMO (Whole Module Optimization) увеличивает специализацию.

**Что такое monomorphization?**
> Процесс создания компилятором отдельных версий generic функции для каждого конкретного типа. `sort<Int>` и `sort<Double>` — два отдельных машинных кода. Максимальная производительность ценой binary size.

**Как работает generic protocol constraint?**
> `func process<T: Collection>(_ collection: T) where T.Element: Hashable`. Можно вызывать `collection.count`, `collection.contains` (методы Collection) и использовать `T.Element` как Hashable тип.

**Что такое associated type inference?**
> Компилятор выводит тип associated type из реализации требований протокола. `struct IntStack { var items: [Int]; mutating func push(_ item: Int) }` — Swift сам выводит `Element == Int` для протокола Stack.

---

### Senior

**Как Swift специализирует generics при компиляции?**
> При WMO (Whole Module Optimization) компилятор видит все вызовы и создаёт специализированные версии. Без WMO — специализация только внутри файла. Специализации для стандартных типов (Int, String) — всегда.

**Когда происходит boxing для generic types?**
> При использовании `any Protocol` (existential) — boxing всегда. При generic без специализации (library boundary) — через witness table без boxing для value types ≤ 3 слов. При прохождении через ABI boundary — boxing для крупных типов.

**Что такое opaque return type (`some`) и чем оно отличается от generic?**
> Generic: тип определяет caller. Opaque (`some`): тип определяет implementer, скрыт от caller. `func makeView() -> some View` — точный тип известен компилятору, но не API пользователю. Static dispatch.

**Как `primary associated types` улучшают использование протоколов в generic коде?**
> `protocol Collection<Element>` — `any Collection<Int>` вместо сложного `any Collection where Element == Int`. Более читаемый синтаксис для constrained existentials. Swift 5.7+.

**Что такое variadic generics (SE-0393)?**
> Параметры-пакеты типов: `func zip<each T>(_ values: repeat each T) -> (repeat each T)`. Позволяет функциям принимать произвольное число разнотипных аргументов. Swift 5.9+.

**Как компилятор обрабатывает глубоко вложенные generic типы?**
> `Optional<Array<Dictionary<String, Optional<Int>>>>` — каждый уровень добавляет complexity. Компилятор разворачивает рекурсивно. Может быть медленная компиляция (exponential в худшем). `typealias` упрощает.

**Что такое `@_specialize` attribute?**
> Форсирует компилятор создать специализацию: `@_specialize(where T == Int)`. Используется в stdlib для критичных путей. Гарантирует специализацию даже через ABI boundary где обычно нет.

**Как generics взаимодействуют с ARC?**
> Generic функция не знает является ли T reference или value type. Компилятор вставляет retain/release через value witness table. При специализации — прямые retain/release или их отсутствие (value types).

**Что такое same-type requirement в generic constraints?**
> `where T.Iterator.Element == U`: T и U должны иметь одинаковый associated type. `func zip<S1: Sequence, S2: Sequence>(_ s1: S1, _ s2: S2) where S1.Element == S2.Element`. Строгое выравнивание типов.

**Как реализовать heterogeneous collection через type erasure?**
> `protocol Drawable { func draw() }; struct AnyDrawable: Drawable { private let _draw: () -> Void; init<D: Drawable>(_ d: D) { _draw = d.draw }; func draw() { _draw() } }; var shapes: [AnyDrawable]`. Коллекция разных Drawable через wrapper.

---

## Associated Types

### Beginner

**Что такое associated type?**
> Placeholder для типа внутри протокола, определяемый при conformance. `protocol Container { associatedtype Item; func add(_ item: Item) }`. Делает протокол generic. Конкретный тип Item — при реализации.

**Как объявить associated type в протоколе?**
> `protocol IteratorProtocol { associatedtype Element; mutating func next() -> Element? }`. Можно задать constraint: `associatedtype Element: Equatable`. Или default: `associatedtype Element = Int`.

**Что такое type alias в контексте associated type?**
> При реализации протокола можно явно задать тип: `typealias Element = String`. Или Swift выведет его из реализации методов. Оба подхода эквивалентны.

**Как Swift инферирует associated type?**
> Из реализации методов: `struct IntIterator: IteratorProtocol { func next() -> Int? { ... } }` — Swift выводит `Element == Int` из возвращаемого типа `next()`. Явный typealias не нужен.

---

### Middle

**Почему нельзя использовать протокол с associated type как тип (`any Protocol`)?**
> До Swift 5.7 — нельзя совсем. С 5.7+ — только через constrained existential: `any Collection<Int>`. Без констрейнта `any Collection` — ошибка, компилятор не знает Element для определения размера и операций.

**Что такое primary associated type?**
> `protocol Collection<Element>` — синтаксис Swift 5.7. Делает один AT "главным". Позволяет: `any Collection<Int>`, `some Collection<String>`. Синтаксический сахар для наиболее часто используемого AT.

**Как использовать constrained existential (`any Collection<Int>`)?**
> `var c: any Collection<Int> = [1, 2, 3]`. Позволяет хранить любую Collection с Int элементами без generics. Overhead existential boxing. `c.count` работает, `c[0]` — нет (Index неизвестен).

**Что такое `where` clause для associated type?**
> `protocol Sequence where Element: Hashable`. Требование к AT при объявлении протокола. Или в extension: `extension Sequence where Element: Equatable { func containsDuplicates() -> Bool }`.

**Как associated types связаны с PAT (Protocol with Associated Type)?**
> PAT — протокол с хотя бы одним associated type (или Self requirements). Создаёт ограничения: нельзя использовать как тип переменной напрямую без wrapper или `any`. Требует generic или existential boxing.

**Что такое type aliasing associated type?**
> `typealias Element = Int` внутри реализующего типа. Явное указание компилятору. Нужно редко — обычно Swift выводит сам. Полезно при неоднозначности или для документирования.

**Как реализовать type erasure для PAT?**
> Wrapper-struct с closure-хранилищами: `struct AnyIterator<Element>: IteratorProtocol { private let _next: () -> Element?; mutating func next() -> Element? { _next() } }`. Скрывает конкретный тип, сохраняет функциональность.

**Что такое same-type constraint для associated types?**
> `func compare<C1: Collection, C2: Collection>(_ c1: C1, _ c2: C2) where C1.Element == C2.Element`. Обе коллекции должны иметь одинаковый Element. Compile-time проверка. Позволяет использовать == для элементов.

---

### Senior

**Как Swift 5.7 изменил работу с existentials и associated types?**
> Ввёл `any` keyword для явного boxed existential. Constrained existentials: `any Collection<Int>`. Opening existentials. Это сделало PAT более практичными без полного перехода на generics.

**Что такое boxed protocol type и как он отличается от generic?**
> `any Protocol` — boxed existential: конкретный тип неизвестен в compile time, dynamic dispatch, heap boxing. `<T: Protocol>` — generic: конкретный тип известен, static dispatch, специализация. Generic быстрее.

**Как recursive associated type constraint работает?**
> `protocol Node { associatedtype Child: Node }`. Рекурсивное требование: Child тоже должен быть Node. Компилятор проверяет при conformance. Используется для tree-подобных структур.

**Как multiple associated types влияют на type inference?**
> `protocol Mapping { associatedtype Input; associatedtype Output; func map(_ input: Input) -> Output }`. Компилятор должен вывести оба типа. Иногда нужны явные typealias или аннотации при неоднозначности.

**Что такое associated type default?**
> `associatedtype Index = Int` — если реализующий тип не указывает Index, используется Int. Убирает необходимость явно указывать для самого common case. Можно переопределить при нужде.

---

## Opaque Types & Existentials

### Beginner

**Что такое opaque type в Swift?**
> Тип возврата скрытый за `some Protocol`. `func makeView() -> some View`. Конкретный тип известен компилятору и не меняется, но скрыт от пользователя API. Обратная генерика — тип выбирает implementer.

**Что означает ключевое слово `some`?**
> "Некоторый конкретный тип соответствующий протоколу". `some View` — есть конкретный View тип, но какой — определяет функция, не caller. Компилятор знает тип, может оптимизировать. Static dispatch.

**Чем `some View` отличается от `View`?**
> `View` без `any` — до Swift 5.7 ошибка для PAT. `some View` — opaque type, компилятор знает конкретный тип. `any View` — existential, boxing, dynamic dispatch. `some` быстрее и не требует boxing.

**Что такое `any` keyword в Swift 5.7+?**
> Явный маркер existential type: `any Animal`, `any Codable`. Указывает что тип неизвестен в compile time, будет boxing. Обязателен для PAT в Swift 5.7+. Без `any` — неявный existential, deprecated.

---

### Middle

**Чем отличается `some Protocol` от `any Protocol`?**
> `some`: конкретный тип фиксирован (один за время работы), известен компилятору, static dispatch, inline. `any`: тип может быть любым соответствующим, dynamic dispatch, boxing, runtime overhead. `some` быстрее.

**Когда использовать `some` вместо `any`?**
> `some` — когда тип всегда один и тот же (возврат из функции, SwiftUI body). `any` — когда нужна гетерогенная коллекция разных типов или тип меняется в рантайме. Предпочитай `some` для производительности.

**Что такое reverse generics?**
> Другое название для opaque return type (`some`). Обычные generics: тип выбирает caller. Reverse/opaque generics: тип выбирает callee (implementer). Инвертированное направление абстракции.

**Как `some` помогает скрыть конкретный тип возврата?**
> `func makePublisher() -> some Publisher<Int, Never>`. Реализация может изменить цепочку операторов не ломая API. Пользователь видит только `some Publisher<Int, Never>`, не конкретный тип. ABI стабильность.

**Что такое existential opening?**
> Swift 5.7+: `func makeItSpeak(_ animal: any Animal) { if let concrete = animal as? any Animal { concrete.speak() } }`. Компилятор открывает existential давая доступ к конкретному типу внутри замыкания для вызова Self-требующих методов.

**Как работает `any` с protocol requirements?**
> `var animal: any Animal = Cat()`. Можно вызывать только те требования протокола которые не используют Self или AT. `animal.speak()` — ок если speak() -> Void. `animal.copy()` — нет если return Self.

**Что такое `opened existential`?**
> Временно "открытый" existential в контексте вызова. Компилятор создаёт локальный generic контекст где тип конкретен. Позволяет вызывать Self-требующие методы. `open(animal as! any Animal)` (Swift 5.7).

**Как `some` влияет на производительность vs `any`?**
> `some`: no boxing (если value type ≤ 24 байт), static dispatch, inline, специализация. `any`: existential container 40 байт, heap allocation для крупных, PWT lookup, нет inline. `some` быстрее в 3-10x для tight loops.

---

### Senior

**Как компилятор работает с opaque types внутри?**
> Создаёт "underlying type token" — внутреннее имя для конкретного типа. Использует при оптимизациях. Для ABI: opaque type — как generic с скрытым type argument. Клиент не знает тип, но компилятор — да.

**Что такое primary associated type и constrained existential?**
> `protocol Collection<Element>` — primary AT. `any Collection<Int>` — constrained existential. Объединяет удобство existential (разные типы) с частичной типизацией Element. Лучше чем untyped `any Collection`.

**Как `any` и `some` взаимодействуют с associated types?**
> `some Collection<Int>` — opaque, фиксированный тип. `any Collection<Int>` — existential boxed. В функции: `func process(_ c: some Collection<Int>)` — generic под капотом. `func process(_ c: any Collection<Int>)` — existential, slower.

**Что такое covariance и contravariance в контексте existentials?**
> `any Animal` — не является подтипом `any Mammal` даже если Mammal: Animal. Existentials не covariant по умолчанию. `[any Cat]` нельзя передать где ожидается `[any Animal]`. В Swift 5.7+ работа над этим продолжается.

**Как `any` влияет на dynamic dispatch?**
> `any Protocol` — всегда dynamic dispatch через Protocol Witness Table. Нельзя инлайнить. Нельзя специализировать. Компилятор не видит конкретный тип — никаких оптимизаций. Overhead vs static dispatch ~5-10x для trivial методов.

---

## Result Builders

### Beginner

**Что такое result builder?**
> Механизм создания DSL (Domain Specific Language) в Swift. Позволяет писать декларативный код внутри специального блока. `@ViewBuilder`, `@resultBuilder`. Компилятор трансформирует декларативный синтаксис в серию вызовов.

**Где result builder используется в SwiftUI?**
> `body: some View` неявно использует `@ViewBuilder`. `VStack { Text("a"); Text("b") }` трансформируется в `VStack(content: { ViewBuilder.buildBlock(Text("a"), Text("b")) })`. Также: `@SceneBuilder`, `@CommandsBuilder`.

**Что такое `@resultBuilder` attribute?**
> Атрибут для struct/enum/class объявляющий его result builder'ом. Тип должен содержать статические `buildBlock` методы. `@resultBuilder struct ViewBuilder { static func buildBlock<C: View>(_ c: C) -> C { c } }`.

**Как SwiftUI использует ViewBuilder?**
> `@ViewBuilder var body: some View { Text("Hello"); Image("icon") }` → компилятор вызывает `ViewBuilder.buildBlock(Text("Hello"), Image("icon"))` → возвращает `TupleView<(Text, Image)>`. Прозрачно для разработчика.

---

### Middle

**Как реализовать собственный result builder?**
> `@resultBuilder struct StringBuilder { static func buildBlock(_ parts: String...) -> String { parts.joined() } }`. Использование: `@StringBuilder func greeting() -> String { "Hello"; " "; "World" }` → "Hello World".

**Что такое `buildBlock`?**
> Обязательный статический метод result builder'а. Принимает результаты выражений из блока и объединяет их. Может быть перегружен для разного числа аргументов. `static func buildBlock(_ c1: C1, _ c2: C2) -> TupleView<(C1,C2)>`.

**Как result builder обрабатывает условные конструкции (`if/else`)?**
> Через `buildIf` и `buildEither(first:)/buildEither(second:)`. `if condition { Text("yes") } else { Text("no") }` → `buildEither(first: Text("yes"))` или `buildEither(second: Text("no"))`. Тип результата — `_ConditionalContent<C1,C2>`.

**Что такое `buildOptional` и `buildEither`?**
> `buildOptional` — для `if` без else: возвращает `C?`. `buildEither(first:)` и `buildEither(second:)` — для `if/else`: выбирают нужную ветку. Оба нужны для поддержки условий в result builder.

**Как реализовать поддержку `for` циклов в result builder?**
> Реализовать `buildArray(_ components: [Component]) -> Component`. Компилятор генерирует массив результатов цикла и передаёт в buildArray. `for item in items { Text(item) }` → `buildArray([Text(items[0]), Text(items[1]), ...])`.

**Что такое `buildFinalResult`?**
> Необязательный метод для финальной трансформации результата: `static func buildFinalResult<C: View>(_ component: C) -> some View`. Позволяет конвертировать внутренний тип в публичный тип возврата.

**Как result builder влияет на тип замыкания?**
> Параметр функции аннотированный `@ViewBuilder var content: () -> Content` — тело замыкания обрабатывается через ViewBuilder. Тип `Content` выводится как результат `buildBlock`. Компилятор трансформирует всё автоматически.

**Что такое `buildExpression`?**
> Метод для предварительной обработки каждого выражения: `static func buildExpression(_ expression: String) -> String`. Позволяет конвертировать expression перед передачей в buildBlock. Нужен для поддержки разных типов выражений.

**Как тестировать код, использующий result builder?**
> Тестировать результирующее значение, не трансформацию. `let view = MyBuilder.buildBlock(Text("A"), Text("B"))`. Или через XCTest snapshot. Трансформация — работа компилятора, не нужно тестировать.

**Как result builder используется в Swift Regex?**
> `Regex { OneOrMore(.digit); Optionally("-"); OneOrMore(.digit) }` — `RegexBuilder` DSL. Каждый компонент — `RegexComponent`. Компилятор складывает через `buildBlock` в `Regex` объект.

---

### Senior

**Как компилятор трансформирует result builder код?**
> 1) Выражения в блоке извлекаются как отдельные statements. 2) Каждое выражение — через `buildExpression`. 3) Условия — через `buildOptional`/`buildEither`. 4) Циклы — через `buildArray`. 5) Всё — через `buildBlock`. 6) Результат — через `buildFinalResult`.

**Что такое `buildArray` и когда он вызывается?**
> `static func buildArray(_ components: [Component]) -> Component`. Вызывается для `for-in` циклов в result builder. Компилятор собирает все итерации в массив, передаёт в buildArray. Без реализации — for-in не поддерживается.

**Как result builder помогает создавать DSL?**
> Декларативный синтаксис для domain-specific задач: SwiftUI layout, SQL запросы, HTML generation, test matchers. Пользователь пишет читаемый код, компилятор трансформирует в API вызовы. Меньше бойлерплейта.

**Как реализовать HTML builder через result builder?**
> `@resultBuilder struct HTMLBuilder { static func buildBlock(_ nodes: HTMLNode...) -> HTMLNode { .fragment(nodes) } }`. `div { h1("Title"); p("Body") }` → вызовы buildBlock → HTMLNode tree. Типобезопасный HTML.

**Как error handling работает внутри result builder?**
> По умолчанию result builder не пропагирует ошибки. Для поддержки throwing: реализовать `buildBlock` throwing версию. В SwiftUI — нет поддержки throws в body. Обходной путь: `.task {}` для async работы с ошибками.

**Как availability checking работает внутри result builder?**
> `@available(iOS 16, *) if #available(iOS 16, *) { NavigationStack { } } else { NavigationView { } }`. Compiler checks availability в каждой ветке result builder. `buildEither` обрабатывает ветки с разными availability требованиями.

---

## Property Wrappers

### Beginner

**Что такое property wrapper?**
> Механизм инкапсуляции логики доступа к свойству за переиспользуемой аннотацией. `@State var count = 0` — `@State` это property wrapper, добавляющий хранение и автоматическую перерисовку SwiftUI. Убирает boilerplate.

**Как объявить собственный property wrapper?**
> `@propertyWrapper struct Clamped { var wrappedValue: Int { didSet { wrappedValue = min(max(wrappedValue, 0), 100) } }; init(wrappedValue: Int) { self.wrappedValue = min(max(wrappedValue, 0), 100) } }`.

**Что такое `wrappedValue`?**
> Обязательное свойство property wrapper — реальное хранимое значение. Доступ к нему — через имя свойства без `$`. `@Clamped var progress = 0; progress = 150 // → 100`.

**Что такое `projectedValue`?**
> Опциональное свойство property wrapper для доступа через `$`. В SwiftUI `@State var count` → `$count` возвращает `Binding<Int>` (projectedValue). Тип projectedValue может быть любым.

**Как обратиться к projected value через `$`?**
> `$propertyName` — синтаксический сахар для доступа к `projectedValue`. `@State var name = ""`; `TextField("", text: $name)` — передаём `Binding<String>`, а не `String`.

---

### Middle

**Как реализовать property wrapper с атрибутами инициализации?**
> `@propertyWrapper struct Clamped { init(wrappedValue: Int, range: ClosedRange<Int>) { ... } }`. Использование: `@Clamped(range: 0...100) var progress = 50`. Дополнительные параметры передаются в атрибут.

**Что такое `@State`, `@Binding`, `@Published` — как они реализованы?**
> `@State` — wrapper хранит значение в SwiftUI storage, при изменении тригерит перерисовку. `@Binding` — projectedValue `@State` переменной, двусторонняя ссылка. `@Published` — wrapper из Combine, отправляет через `objectWillChange` publisher.

**Как property wrapper взаимодействует с протоколами?**
> Property wrapper не синтезируется через протоколы. `protocol Configurable { @State var isEnabled: Bool }` — ошибка компилятора. Протоколы не могут требовать property wrapper — только тип и get/set.

**Как тестировать код с property wrappers?**
> Тестировать логику через реальные значения. `@Clamped var value = 0; value = 200; XCTAssertEqual(value, 100)`. Или тестировать wrapper отдельно. SwiftUI-специфичные (State) сложнее — нужен hosting environment.

**Можно ли стекировать property wrappers?**
> Да: `@Clamped @UserDefault var value = 0`. Применяются снаружи внутрь: сначала UserDefault, результат — wrappedValue для Clamped. Ограничение: inner wrapper должен быть `var`, порядок важен.

**Что происходит с property wrapper при наследовании?**
> Подкласс не может изменить wrapper родительского свойства. Свойства с wrapper в суперклассе — унаследованы как есть. `override var wrappedProperty` — ошибка, нельзя override wrapped property.

**Как property wrapper влияет на Codable синтез?**
> Компилятор не синтезирует Codable для property wrapper автоматически — нужно реализовать вручную или обойти через `@CodableIgnored` паттерн. `encode(to:)` должен работать с `wrappedValue`.

**Что такое `@AppStorage` и как он реализован?**
> Property wrapper из SwiftUI над `UserDefaults`. `wrappedValue` читает/пишет в UserDefaults по ключу. `projectedValue` — `Binding<T>`. Наблюдает за изменениями через `NotificationCenter` и тригерит перерисовку View.

**Как property wrapper используется в аргументах функций?**
> Swift 5.7+: `func process(@Clamped value: Int)`. Wrapper применяется к аргументу при вызове. Полезно для валидации входных данных на уровне сигнатуры функции.

**Что такое attached macro vs property wrapper?**
> Property wrapper — runtime механизм через struct. Attached macro (Swift 5.9) — compile-time трансформация кода. Macro может генерировать stored properties, override, более мощный. `@Observable` — macro, не property wrapper.

---

### Senior

**Как компилятор трансформирует property wrapper?**
> `@State var count = 0` → `private var _count: State<Int> = State(wrappedValue: 0); var count: Int { get { _count.wrappedValue } set { _count.wrappedValue = newValue } }`. Компилятор генерирует backing storage и accessors.

**Как property wrapper влияет на ARC?**
> Backing storage (_count) — дополнительный объект. Если wrapper — class (как State в SwiftUI) — ARC overhead. Если wrapper — struct с reference type внутри — ARC для внутреннего объекта. Важно для performance-critical кода.

**Как реализовать thread-safe property wrapper?**
> `@propertyWrapper struct Protected<T> { private var _value: T; private let lock = NSLock(); var wrappedValue: T { get { lock.withLock { _value } }; set { lock.withLock { _value = newValue } } } }`. Lock защищает чтение и запись.

**Как property wrapper взаимодействует с Sendable?**
> Wrapper должен быть `Sendable` если используется с акторами. Backing storage должен быть thread-safe. `@State` в SwiftUI — MainActor-bound. Custom thread-safe wrapper должен явно conformить Sendable или использоваться только на одном акторе.

**Как реализовать property wrapper с reset функциональностью?**
> `@propertyWrapper struct Resettable<T> { let defaultValue: T; var wrappedValue: T; var projectedValue: () -> Void { { wrappedValue = defaultValue } } }`. `$myValue()` сбрасывает до дефолта через projectedValue closure.

---

## Access Control

### Beginner

**Какие уровни доступа есть в Swift?**
> 5 уровней от открытого к закрытому: `open` > `public` > `package` (Swift 5.9) > `internal` > `fileprivate` > `private`. По умолчанию — `internal`. Модификаторы ограничивают видимость за пределами определённой области.

**Что означает `private`?**
> Доступ только внутри объявляющего declaration и extension того же типа в том же файле. `private var count` в ClassA — не видно ClassB, не видно extension ClassA в другом файле.

**Что означает `fileprivate`?**
> Доступ только внутри одного source файла. Все типы и расширения в том же файле имеют доступ. Полезно когда несколько типов в одном файле должны взаимодействовать.

**Что означает `internal`?**
> Доступен из любого места в рамках одного модуля (target). Уровень по умолчанию. Невидим из других модулей (app для framework, test target для app).

**Что означает `public`?**
> Видим из других модулей, но нельзя наследовать класс или override методы за пределами модуля. Для библиотек/frameworks — публичный API. Внутри модуля — как internal.

**Что означает `open`?**
> Максимально открытый: видим и наследуем/override из других модулей. Только для классов и их членов. Явный сигнал что тип спроектирован для subclassing.

**Что является уровнем доступа по умолчанию?**
> `internal`. Всё что не помечено явно — доступно внутри модуля и невидимо снаружи. Удобно для internal кода приложения.

---

### Middle

**Чем отличается `private` от `fileprivate`?**
> `private` — только в пределах Declaration + extension того же типа в том же файле. `fileprivate` — весь файл. `private var x` в ClassA не видна ClassB в том же файле, но `fileprivate var y` — видна.

**Чем отличается `public` от `open`?**
> `public` — видимый снаружи модуля, но нельзя subclass/override вне модуля. `open` — разрешает subclassing и override извне. Apple предпочитает `public` чтобы контролировать inheritance. `open` — осознанное решение разрешить расширение.

**Как access control влияет на наследование?**
> Подкласс не может быть с более открытым доступом чем суперкласс. Override метода — не более открытый чем оригинал (если оригинал `internal` — override может быть `internal` или закрытее, не `public`).

**Как access control работает для extension?**
> Extension в том же модуле — может иметь свой access level. `private extension MyType { }` — все члены private. `extension MyType: PublicProtocol { }` — conformance добавляется даже если extension private (conformance видна по уровню протокола).

**Что такое `@testable import`?**
> `@testable import MyModule` в тест-таргете даёт доступ к `internal` символам как если бы они были `public`. `open` и `public` — без изменений. Позволяет тестировать internal API без изменения его access level.

**Как access control влияет на protocol conformance?**
> Conformance к публичному протоколу — видна на уровне типа. Методы в implementation должны быть хотя бы `internal`. Public протокол с public requirements — реализация тоже должна быть `public`.

**Что такое `private(set)`?**
> Getter — заданного уровня, setter — `private`. `public private(set) var count = 0` — читать может кто угодно снаружи модуля, изменять — только внутри типа. Идеальный паттерн для read-only API с internal mutable state.

**Как access control работает для nested типов?**
> Nested тип по умолчанию — `internal`. Не наследует уровень внешнего типа. `private struct Config { }` внутри `public class Manager { }` — Config не видна снаружи. Нужно явно задавать `public struct Config { }`.

**Как access control влияет на модульность?**
> Чёткие public интерфейсы модулей — меньше зависимостей, проще рефакторинг internal без изменения API. `internal` детали реализации — не утекают наружу. Модуль = unit с явными boundaries.

**Что произойдёт если public протокол требует internal метод?**
> Ошибка компилятора: public протокол не может требовать internal requirements — они были бы видны пользователям протокола, но недоступны для реализации в других модулях. Все протокол требования должны быть ≥ protocol access level.

---

### Senior

**Как access control влияет на бинарную совместимость?**
> Изменение `public` → `internal` — breaking change для бинарной совместимости (пользователи больше не видят символ). Изменение `internal` → `public` — не breaking (добавляем API). Вот почему нужно тщательно думать о public API изначально.

**Что такое `package` access level (Swift 5.9)?**
> Новый уровень между `internal` и `public`. Видим в пределах Swift Package (нескольких модулей одного пакета), но не снаружи. Полезно для library-internal utilities которые нужны разным таргетам пакета.

**Как спроектировать публичный API с правильным access control?**
> Принцип минимального доступа: начни с `private`/`internal`, открывай до `public` только необходимое. `public` protocol с необходимыми requirements, `internal` implementation details. `open` — только для точек расширения.

**Как access control взаимодействует с `@inlinable`?**
> `@inlinable` требует `public` или `@usableFromInline`. Встраивает тело функции в ABI. Делает implementation details видными клиентам (в binary), хотя access level остаётся `public`. Не меняет доступность, меняет visibility for optimization.

**Что такое `@usableFromInline`?**
> Делает `internal` символ доступным из `@inlinable` функций других модулей. Не меняет логический access level — через обычный код не доступен. Нужно для вспомогательных функций используемых в inlinable public API.

**Как тестировать private методы без нарушения инкапсуляции?**
> 1) `@testable import` для `internal`. 2) Тестировать через public/internal интерфейс (предпочтительно). 3) Вынести в отдельный internal тип. 4) `fileprivate` вместо `private` — видно из extension в том же файле. 5) Property injection в тестах.

---

## Closures

### Beginner

**Что такое замыкание (closure) в Swift?**
> Самодостаточный блок функциональности, который может быть передан и использован в коде. Может захватывать переменные из окружающего контекста. `{ (param) -> ReturnType in body }`. Функции — именованные замыкания.

**Как объявить замыкание?**
> `let greet: (String) -> String = { name in "Hello, \(name)!" }`. Или с типом из контекста: `let double = { (x: Int) -> Int in x * 2 }`. Без аргументов: `let action = { print("done") }`.

**Что такое trailing closure syntax?**
> Если последний аргумент функции — замыкание, его можно вынести за скобки: `array.sorted { $0 < $1 }` вместо `array.sorted(by: { $0 < $1 })`. Если единственный аргумент — скобки можно опустить совсем.

**Как передать замыкание как аргумент функции?**
> `func apply(_ operation: (Int) -> Int, to value: Int) -> Int { operation(value) }`. Вызов: `apply({ $0 * 2 }, to: 5)` или `apply(to: 5) { $0 * 2 }` (trailing closure).

**Что такое `$0`, `$1` shorthand argument names?**
> Сокращённые имена для аргументов замыкания в порядке следования. `{ $0 + $1 }` эквивалентно `{ a, b in a + b }`. Удобно для коротких замыканий, где имена не нужны.

---

### Middle

**Что такое capturing в замыканиях?**
> Замыкание захватывает переменные и константы из окружающего scope и может использовать их даже после завершения этого scope. `var count = 0; let increment = { count += 1 }`. count захвачен и изменяется.

**Как closure захватывает переменные?**
> По умолчанию — захватывает **ссылку** на переменную (для reference types) или копию в момент **изменения** (для value types внутри shared box). `var x = 0; let c = { print(x) }; x = 10; c() // 10`.

**Что такое strong, weak и unowned capture?**
> Strong (default): захватывает сильную ссылку — удерживает объект. Weak: `[weak self]` — захватывает слабую, self? может стать nil. Unowned: `[unowned self]` — как weak но non-optional, краш если объект освобождён.

**Что такое capture list?**
> Список `[weak self, unowned delegate]` перед параметрами замыкания. Определяет как захватывать переменные. `{ [weak self] in self?.doSomething() }`. Без capture list — default (strong) capture.

**Что такое retain cycle через замыкания и как его избежать?**
> A.closure = { A.method() } — closure захватывает A (strong), A держит closure (strong) → retain cycle, memory leak. Решение: `[weak self]` или `[unowned self]` в capture list. Проверяй через Instruments → Leaks.

**Что означает `@escaping`?**
> Замыкание помечается `@escaping` если оно может быть вызвано после возврата функции — сохранено в property, передано асинхронно. Без `@escaping` — non-escaping, компилятор знает что вызовется синхронно, оптимизирует.

**Когда замыкание является escaping?**
> Когда сохраняется за пределами вызова функции: в property класса, в DispatchQueue.async, в URLSession completionHandler, в массиве замыканий. Любое async callback — escaping.

**Что такое `@autoclosure`?**
> Автоматически оборачивает выражение в замыкание: `func log(_ message: @autoclosure () -> String)`. Вызов: `log("Value: \(heavyComputation())")` — вычисление произойдёт только если log решит его вычислить.

**Как `@autoclosure` влияет на вычисление аргументов?**
> Ленивое вычисление. `&&` оператор использует `@autoclosure`: правый аргумент вычисляется только если левый true. `assert` — @autoclosure, message не вычисляется в Release. Полезно для отложенного expensive computation.

**Что такое функции высшего порядка?**
> Функции принимающие функции как аргументы или возвращающие функции. `map`, `filter`, `reduce` — функции высшего порядка. `func apply(_ f: (Int) -> Int) -> (Int) -> Int { return f }` — возвращает функцию.

---

### Senior

**Как Swift оптимизирует замыкания без захвата?**
> Non-capturing closure — конвертируется в обычный function pointer (thin pointer, 1 слово). Нет heap allocation для контекста. `sorted(by: <)` — встроенный оператор, полностью оптимизируется в статический вызов.

**Что такое thin vs thick function pointer?**
> Thin: указатель на функцию без контекста (обычная функция, non-capturing closure) — 1 слово. Thick: 2 слова — указатель на код + указатель на захваченный контекст (heap allocation). Замыкания с захватом — thick.

**Как замыкание хранится в памяти?**
> Захватывающее замыкание: heap-allocated context объект содержит захваченные переменные + ref count. Thick pointer (2 слова) указывает на код и контекст. Context — ARC-managed. При копировании замыкания — копируется thick pointer, context retains.

**Что такое closure lifetime и как оно влияет на ARC?**
> Замыкание продлевает жизнь всех захваченных strong объектов. Пока замыкание живёт — живут захваченные объекты. escaping closure в long-lived property может неожиданно удерживать view controller.

**Как `[weak self]` влияет на работу замыкания если self уже освобождён?**
> `self` становится `nil`. `[weak self] in guard let self else { return }` — безопасный выход. Без guard и принудительного `self!` — краш. Важно: в async контексте self может стать nil между проверкой и использованием.

**Что такое `@Sendable` closure?**
> Closure безопасная для передачи через concurrency boundaries. `Task { @Sendable in ... }`. Не может захватывать mutable state без синхронизации. Компилятор проверяет Sendable conformance захваченных переменных.

**Как замыкания взаимодействуют со structured concurrency?**
> `Task {}` принимает `@escaping @Sendable () async throws -> T`. Замыкание должно быть Sendable. Захват `self` — self должен быть Sendable или actor-isolated. `async let` — замыкание выполняется в child task.

**Что такое noescape closure optimization?**
> Non-escaping closure (default) — компилятор знает что она не сохранится. Оптимизации: no heap allocation для context (стек), no retain/release, inline возможен. Значительно быстрее escaping.

**Как escaping closure влияет на компиляторные оптимизации?**
> Escaping: компилятор не знает когда и сколько раз вызовется. Нет inline. Context — heap. Захваченные объекты должны быть reference-counted. Компилятор не может доказать отсутствие side effects между вызовами.

**Что такое partial application функций?**
> Создание функции с некоторыми фиксированными аргументами. `let addFive = { add(5, $0) }` — partial application add. Curry pattern: `func add(_ a: Int) -> (Int) -> Int { { b in a + b } }`. `add(5)(3) == 8`.

---

## ARC & Memory Management

### Beginner

**Что такое ARC (Automatic Reference Counting)?**
> Механизм управления памятью в Swift/ObjC. Компилятор автоматически вставляет retain/release вызовы. Когда retain count объекта достигает 0 — объект освобождается. Не требует ручного управления (в отличие от C malloc/free).

**Как ARC управляет памятью?**
> При каждом новом владельце объекта (присвоение, передача) — retain count +1. При удалении владельца (конец scope, nil присвоение) — release, count -1. При count==0 — deinit вызывается, память освобождается.

**Что такое retain count?**
> Счётчик сильных ссылок на объект. Хранится в заголовке объекта на куче. При создании — 1. Каждая strong reference +1, освобождение -1. При достижении 0 — объект уничтожается.

**Что происходит когда retain count достигает 0?**
> 1) deinit вызывается. 2) Все strong references на другие объекты — release. 3) Память объекта возвращается в heap. 4) Weak references на этот объект обнуляются (→ nil). Происходит синхронно (не через GC).

**Что такое strong reference?**
> Default тип ссылки. `var object = MyClass()` — strong reference, удерживает объект живым пока существует ссылка. Влияет на retain count. Несколько strong references — объект живёт пока хотя бы одна существует.

**Что такое weak reference?**
> `weak var delegate: MyDelegate?` — не увеличивает retain count. Объект может быть освобождён даже при наличии weak ссылок. При освобождении объекта weak references → nil автоматически. Всегда Optional.

**Что такое unowned reference?**
> `unowned let parent: Parent` — не увеличивает retain count, но non-optional. Предполагает что объект живёт не меньше чем держатель ссылки. При доступе к уже освобождённому объекту — краш (dangling pointer).

---

### Middle

**Чем weak отличается от unowned?**
> weak: Optional, при освобождении объекта → nil, безопасно. unowned: non-optional, не обнуляется — если объект освобождён → краш при доступе. weak безопаснее, unowned — когда гарантирован lifetime (child → parent).

**Когда использовать weak, а когда unowned?**
> weak: когда lifetime неизвестен или объект может быть nil (delegate, обратные ссылки в опциональных отношениях). unowned: когда объект гарантированно живёт дольше владельца ссылки (child → parent, closure в методе объекта).

**Что такое retain cycle?**
> Два или более объекта держат strong ссылки друг на друга → ни один не освобождается → memory leak. A.b = B(); B.a = A() → оба с retain count ≥ 1 навсегда. Instruments → Leaks помогает найти.

**Как возникает retain cycle с замыканиями?**
> `class A { var closure: (() -> Void)?; func setup() { closure = { self.doWork() } } }`. A держит closure, closure держит A (через self) → retain cycle. Решение: `[weak self]` или `[unowned self]` в capture list.

**Как обнаружить retain cycle?**
> 1) Instruments → Leaks, 2) Memory Graph Debugger (Debug → Memory Graph), 3) Добавить `deinit { print("deallocated") }` и проверить вызов, 4) Memory использование в Debug Navigator.

**Что такое memory leak?**
> Память выделена но никогда не освобождается — retain count > 0 из-за retain cycle или потери последней ссылки без release. Увеличивает memory footprint приложения. Может привести к jetsam kill на iOS.

**Как Instruments помогает найти memory leak?**
> Instruments → Leaks: показывает объекты с ненулевым retain count которые недостижимы (leak). Memory Graph: визуальный граф ссылок — найди циклы. Allocations: отслеживай рост heap без соответствующего освобождения.

**Что такое weak delegate pattern?**
> `weak var delegate: SomeDelegate?` — стандартный паттерн для делегатов. Объект не должен владеть своим делегатом (иначе retain cycle). UITableView не удерживает tableView.delegate — это weak reference.

**Как правильно использовать `[weak self]` в async коде?**
> `Task { [weak self] in guard let self else { return }; await self.loadData() }`. Без `[weak self]` в Task — self удерживается пока Task не завершится. С `[weak self]` — Task не удерживает self, проверяй nil перед каждым использованием.

**Что происходит с weak reference после освобождения объекта?**
> Автоматически обнуляется → становится `nil`. Swift runtime использует side table для отслеживания weak references. При deallocate: все weak refs в side table → nil, потом память объекта освобождается.

---

### Senior

**Как ARC работает под капотом?**
> Компилятор вставляет `swift_retain` / `swift_release` вызовы. Retain count хранится в заголовке объекта (inline ref count). При переполнении — side table. Release: атомарно декрементирует count, при 0 — вызывает destroy function.

**Что такое side table в ARC?**
> Вспомогательная структура для объектов с weak/unowned references. Хранится отдельно от объекта. Содержит: strong ref count, weak ref count, unowned ref count, флаги. Объект в side table живёт пока есть weak ссылки.

**Что такое weak reference counting отдельно от strong?**
> Side table хранит отдельный weak count. Объект deinit когда strong count → 0, но память не освобождается пока weak count > 0 (иначе weak refs могут читать освобождённую память). Память освобождается когда оба count = 0.

**Как ARC взаимодействует со Swift Concurrency?**
> Actor isolates state — ARC для акторных объектов работает как обычно. Task захватывает объекты в замыканиях (strong). `[weak self]` в Task работает. `Sendable` constraint гарантирует безопасную передачу через boundaries.

**Что такое immortal object в Swift runtime?**
> Объект с "immortal" bitpattern в ref count — никогда не освобождается. Используется для глобальных объектов и статических данных. Стандартные literal types (пустые строки, числа) — immortal, не нужен ARC overhead.

**Как компилятор вставляет retain/release вызовы?**
> ARC optimization pass в SIL (Swift Intermediate Language): вставляет retain при каждом "use" ссылки, release при конце lifetime. Оптимизирует: устраняет парные retain/release, перемещает release позже в code flow.

**Что такое arc optimization pass?**
> LLVM pass убирающий лишние retain/release. `retain(x); release(x)` → ничего. Перемещает release к последнему use. Устраняет retain/release для short-lived объектов полностью. Значительно уменьшает overhead ARC.

**Как `[unowned self]` может привести к крашу?**
> Если self освобождается до завершения замыкания — доступ к unowned self → краш (EXC_BAD_ACCESS). Пример: анимация с unowned self, но view controller уже popped. Безопаснее использовать `[weak self]` с guard.

**Что такое владение (ownership) в Swift 5.9+?**
> Явный контроль над передачей/копированием значений. `consume x` — перемещает значение (x больше нельзя использовать). `copy x` — явная копия. `borrowing`/`consuming` параметры функций. Уменьшает лишние копии.

**Как `consume` и `copy` операторы влияют на ARC?**
> `consume x` — явное перемещение, компилятор не вставляет retain при передаче, x не может использоваться после. Уменьшает retain/release пары. `copy x` — явная копия там где компилятор мог бы оптимизировать. Для precision ownership.

---

### Staff

**Как ARC Swift отличается от ARC ObjC?**
> Swift ARC: compile-time вставка retain/release, оптимизирован компилятором. ObjC ARC: тоже compile-time, но через ObjC runtime (objc_retain/objc_release). Swift ARC быстрее — нет ObjC runtime overhead, лучше статический анализ.

**Что такое ownership manifesto в Swift?**
> Документ 2017 года описывающий будущую систему владения в Swift: move semantics, borrow checker, non-copyable types. Частично реализовано в Swift 5.9 (consume, borrowing/consuming). Цель: безопасность памяти без GC уровня Rust.

**Как `borrow` и `inout` связаны с ownership?**
> `inout` — передаёт эксклюзивный mutable доступ (временно "владеет" для мутации). Borrowing (Swift 5.9): `borrowing` параметр — read-only доступ без копирования. `consuming` — передаёт ownership, caller не может использовать значение после.

**Что такое non-escapable types и как они связаны с memory safety?**
> Типы которые не могут "убежать" из scope где созданы (Span, BufferView). Гарантия компилятором что не будет dangling reference. Используется для safe raw memory access без unsafe pointer. Swift 5.9+.

**Как move-only types изменяют модель владения?**
> Non-copyable (move-only) тип может быть только перемещён, не скопирован. Даёт уникальное владение как в Rust. `~Copyable` marker. Пример: `FileDescriptor` — ровно один владелец. При transfer — источник становится invalid.

**Как компилятор доказывает безопасность при ownership transfer?**
> Definitive initialization analysis: отслеживает состояние каждой переменной (initialized/consumed/moved). После `consume x` — x помечается как consumed, любой доступ к x → ошибка компилятора. Как borrow checker в Rust, но менее строгий.

---

## Copy-on-Write

### Beginner

**Что такое Copy-on-Write (COW)?**
> Оптимизация: при копировании value type реальное копирование откладывается до момента мутации. Копии разделяют внутренний буфер пока не нужна запись. Уменьшает число дорогих heap copies.

**Какие типы в Swift используют COW?**
> Стандартные коллекции: `Array`, `Dictionary`, `Set`, `String`, `Data`. Кастомные типы — COW нужно реализовывать вручную. `Int`, `Bool` и другие примитивы — не нуждаются (помещаются на стек целиком).

**Когда происходит копирование при COW?**
> При мутации (write) когда буфер разделяется между несколькими экземплярами. Если retain count буфера > 1 и нужна запись — создаётся уникальная копия. При чтении — никогда. При уникальном владении — мутация in-place.

---

### Middle

**Как реализовать COW для собственного типа?**
```swift
struct MyArray<T> {
    private var storage: ArrayStorage<T>
    mutating func append(_ element: T) {
        if !isKnownUniquelyReferenced(&storage) {
            storage = storage.copy()  // copy before write
        }
        storage.append(element)
    }
}
```

**Что такое `isKnownUniquelyReferenced`?**
> `isKnownUniquelyReferenced(&object) -> Bool` — проверяет что retain count объекта == 1 (только один владелец). Работает только с native Swift классами (не NSObject). Используется в COW чтобы решить нужно ли копировать.

**Как проверить, произошло ли копирование?**
> Добавить print в copy() метода storage, или breakpoint в буфере. Через Address Sanitizer или Instruments → Allocations отслеживай новые heap allocations при мутации. В тестах — сравнивать адреса буферов через `withUnsafeBytes`.

**Как COW влияет на производительность?**
> Read-heavy сценарии: нет копирования → отлично. Многократная мутация одного владельца: нет копирования → отлично. Мутация при нескольких владельцах: одно копирование перед первой мутацией → затем in-place. Итого намного лучше eager copy.

**Как вложенные типы влияют на COW?**
> `struct Container { var array: [Int] }`. Копирование Container — shallow copy. `array` разделяет буфер. Мутация `container1.array.append(1)` — COW срабатывает для array, не для Container. Каждая коллекция — отдельный COW unit.

**Как COW работает с `Array<T>` где T — class?**
> COW применяется к буферу Array (массиву указателей). Копирование Array — копируются указатели, не сами объекты (shallow). Объекты класса разделяются между копиями Array. Изменение элемента Array не вызывает COW Array, но изменяет shared object.

---

### Senior

**Как компилятор оптимизирует COW?**
> При WMO компилятор может доказать уникальность владения → пропустить `isKnownUniquelyReferenced` check. Escape analysis: если array не escape из scope — mutation in-place гарантирована. Reduce unnecessary copies через ownership analysis.

**Что такое buffer sharing в COW collections?**
> Несколько экземпляров `Array` могут указывать на один `_ContiguousArrayBuffer` объект. Ref count буфера отслеживает кол-во sharing. При 1 — мутация in-place. При >1 — copy буфера (O(n)) перед мутацией.

**Как реализовать COW для дерева данных?**
> Обернуть каждый узел в class (reference type). В struct-дереве — check uniqueness каждого узла перед мутацией. Рекурсивный COW: mutating метод check root uniqueness, при нужде копирует путь от root до изменяемого узла (path copy).

**Как многопоточность влияет на COW?**
> `isKnownUniquelyReferenced` не thread-safe — только для однопоточного использования. Concurrent read — безопасен (нет мутации). Concurrent write — нужна синхронизация на уровне владельца (actor, lock). COW не делает value types thread-safe автоматически.

**Как COW связан с value semantics гарантиями?**
> COW — implementation detail, не нарушающий value semantics. Наблюдаемое поведение: каждая копия независима. `var b = a; b.append(1)` — a не меняется. COW делает это эффективным, но семантически — полная независимость гарантирована.

---

## Static & Class Methods

### Beginner

**Что такое `static` метод?**
> Метод принадлежащий типу, а не экземпляру. Вызывается на типе: `MyType.staticMethod()`. Не имеет доступа к `self` (экземпляру). Работает для struct, enum, class.

**Что такое `class` метод?**
> Как `static`, но только для классов и может быть переопределён подклассами (`override`). `static func` в классе = `final class func` (нельзя override). `class func` — можно override.

**Чем `static func` отличается от `class func`?**
> `static func` — не может быть переопределён в подклассе (final). `class func` — может быть переопределён через `override`. В struct/enum только `static`, в class оба доступны.

**Можно ли переопределить `static` метод?**
> Нет. `static` = `final class` для классов. Для переопределяемых методов на уровне типа — используй `class func`.

---

### Middle

**В чём разница между `static` и `class` для properties?**
> `static var` — не может быть override. `class var` — только computed, может быть override подклассом. Stored properties на уровне типа — только `static`, не `class`. `class let` — не существует.

**Как `static` методы влияют на наследование?**
> `static` методы не наследуются в смысле override. Подкласс **скрывает** parent static метод своим одноимённым, не переопределяет. `MySubclass.method()` вызовет версию из MySubclass, `MyParent.method()` — из MyParent.

**Что такое `class var` в protocol?**
> Protocol требование для type property: `protocol P { static var value: Int { get } }`. Реализация в class может быть `class var value: Int { ... }` (если нужен override) или `static var value: Int = 0`.

**Как static методы используются в factory pattern?**
> `static func make(config: Config) -> Self` — factory создаёт экземпляр типа. `URLSession.shared`, `JSONDecoder()`, `Calendar.current`. Static factory может возвращать разные реализации скрытые за общим типом.

**Что такое `static let` singleton паттерн?**
> `class Manager { static let shared = Manager() }`. `static let` инициализируется лениво при первом доступе и thread-safely (через dispatch_once). Гарантирует единственный экземпляр. Простой и идиоматичный Swift singleton.

---

### Senior

**Как dispatch работает для static vs class методов?**
> `static func` — прямой static dispatch (адрес известен в compile time). `class func` без override — тоже может быть static или через vtable. `class func` с override — через vtable (class type metadata, не instance vtable).

**Когда static dispatch предпочтительнее dynamic?**
> Всегда быстрее (нет lookup). `final` классы, `static` методы, struct методы — static dispatch. В tight loops где вызывается миллионы раз — разница заметна. Компилятор может inline static dispatch.

**Как static методы влияют на testability?**
> Static методы — трудно подменить в тестах (нет injection). Предпочитай protocol + instance injection. Или `static var makeInstance: () -> Service = { ServiceImpl() }` — подменяемый в тестах factory.

**Что такое type-level polymorphism?**
> Полиморфизм на уровне типов через `class func` с override. `class Animal { class func sound() -> String { "..." } }; class Dog: Animal { override class func sound() -> String { "Woof" } }`. Вызов через тип: `Dog.sound()`.

---

## Defer

### Beginner

**Что такое `defer` в Swift?**
> Блок кода выполняющийся при выходе из текущей области видимости (scope) — неважно как: return, throw, break или просто конец блока. `defer { cleanup() }`.

**Когда выполняется блок `defer`?**
> В момент выхода из scope (функции, цикла, блока if). Гарантированно выполнится даже при throw (но не при `fatalError`). Выполняется после return expression но до фактического возврата из функции.

**Может ли быть несколько `defer` блоков?**
> Да, можно несколько `defer` в одном scope. Выполняются в обратном порядке объявления (LIFO — как стек): последний объявленный — первый выполнится.

---

### Middle

**В каком порядке выполняются несколько `defer` блоков?**
> LIFO (Last In, First Out) — обратный порядку объявления. `defer { A }; defer { B }; defer { C }` → при выходе: C, B, A. Аналогично стеку — последний зарегистрированный → первый вызванный.

**Как `defer` работает с `throw`?**
> При выбросе ошибки — `defer` блоки выполняются до propagation ошибки вверх. Это гарантирует cleanup даже при ошибках. `defer { file.close() }` — закроет файл независимо от того, бросила функция или нет.

**Как `defer` используется для cleanup ресурсов?**
> Открытие ресурса + defer для закрытия: `let file = openFile(); defer { file.close() }`. Блокировка + defer для разблокировки: `lock.lock(); defer { lock.unlock() }`. Код cleanup рядом с открытием — не забудешь.

**Что произойдёт с `defer` если функция вызывает `fatalError`?**
> `fatalError` не выполняет defer. `fatalError` прерывает выполнение процесса немедленно через `abort()`. Defer предназначен для нормального выхода и throw. Нельзя полагаться на defer при fatalError.

**Как `defer` работает внутри цикла?**
> `defer` в теле цикла выполняется при каждой **итерации** (при выходе из scope итерации). `for i in array { defer { print("end") }; process(i) }` — "end" печатается после каждого process(i). Не после цикла целиком.

**Что такое RAII паттерн и как `defer` его реализует?**
> RAII (Resource Acquisition Is Initialization) — ресурс захватывается при создании объекта и освобождается при уничтожении. `defer` — функциональный аналог: "захват" до defer, "освобождение" в defer блоке. Без создания wrapper объекта.

---

### Senior

**Как `defer` взаимодействует с async/await?**
> В async функции `defer` выполняется при выходе из suspend point или функции. Важно: defer выполняется на том же executor где была функция в момент выхода (может быть другой поток после await). Thread-safe cleanup в defer нужно проектировать осознанно.

**Как `defer` влияет на ARC и владение?**
> Объекты захваченные в defer удерживаются (strong) до выполнения defer блока. Это продлевает lifetime. Иногда нужно явно `let _ = capturedObject` чтобы продлить, или наоборот — nil перед defer чтобы освободить раньше.

**Как `defer` в async функции работает при отмене Task?**
> При cancellation Task (cooperative): если функция проверяет `Task.isCancelled` и throws `CancellationError` — defer выполнится как при любом throw. `withTaskCancellationHandler` + defer = правильный cleanup при cancellation.

---

## Error Handling

### Beginner

**Как Swift обрабатывает ошибки?**
> Через `throws`/`try`/`catch` механизм. Функция помечается `throws` если может бросить ошибку. Вызов — через `try`. Обработка — в `do { try f() } catch { }`. Или `try?` (Optional), `try!` (force, краш).

**Что такое `throws`?**
> Ключевое слово в сигнатуре функции: `func parse(_ data: Data) throws -> Model`. Указывает что функция может бросить ошибку (любого типа `Error`). Вызывающий обязан обработать через try/catch.

**Что такое `try`, `try?`, `try!`?**
> `try` — вызов throwing функции внутри do-catch, пробрасывает ошибку вверх. `try?` — преобразует результат в Optional (nil при ошибке, ошибка теряется). `try!` — force unwrap, краш при ошибке. Избегай `try!` в production.

**Как объявить свой тип ошибки?**
> `enum NetworkError: Error { case invalidURL; case timeout; case serverError(Int) }`. Любой тип conforming к `Error` protocol может использоваться как ошибка. `Error` — пустой протокол маркер.

**Что такое `Error` протокол?**
> Marker протокол: `public protocol Error: Sendable`. Пустой — не требует реализации. Тип conforming к Error может быть брошен через throw. `LocalizedError` расширяет с `errorDescription`, `failureReason`.

---

### Middle

**Что такое `Result<Success, Failure>`?**
> Enum: `.success(Success)` или `.failure(Failure)`. Позволяет передавать результат или ошибку как значение (не через throws). Удобно для async completion handlers, хранения в переменных, Combine.

**Чем `Result` отличается от `throws`?**
> throws: синхронный, ошибка пропагируется по стеку вызовов. Result: значение — можно хранить, передавать, создавать позже. Result используют где throws неудобен: async callbacks, stored results, delayed evaluation.

**Что такое `rethrows`?**
> `func map<T>(_ transform: (Element) throws -> T) rethrows -> [T]`. Функция "перебрасывает" ошибку только если переданное замыкание бросает. Если замыкание non-throwing — вызов функции тоже non-throwing. Нет лишнего try.

**Когда использовать `try?` vs `try!`?**
> `try?` — когда ошибка не важна или ожидается: `if let value = try? parse(data) { }`. `try!` — никогда в production. Только если 100% уверен что ошибки не будет (инициализация с hardcoded данными в тестах).

**Как обработать несколько типов ошибок?**
> `catch let error as NetworkError { }; catch let error as ParseError { }; catch { }`. Или конвертировать в единый тип через mapError. Или использовать `any Error` и проверять через `is`.

**Что такое `LocalizedError`?**
> Протокол расширяющий Error: `errorDescription: String?`, `failureReason: String?`, `recoverySuggestion: String?`. Реализуй для читаемых сообщений об ошибках. `error.localizedDescription` использует это.

**Как тестировать throwing функции?**
> `XCTAssertThrowsError(try dangerousFunc()) { error in XCTAssertEqual(error as? MyError, .expectedCase) }`. Или `XCTAssertNoThrow(try safeFunc())`. Для async: `await XCTAssertThrowsError(try await asyncFunc())`.

**Что такое typed throws (SE-0413)?**
> Swift 6: `func parse() throws(ParseError) -> Model`. Конкретный тип ошибки в сигнатуре. Позволяет catch без as?: `catch let error { error.specificProperty }`. Компилятор знает тип error в catch.

**Как `defer` работает с `throw`?**
> При throw из функции: 1) defer блоки выполняются в обратном порядке, 2) потом ошибка пропагируется вверх. Гарантирует cleanup. `defer { resource.release() }` выполнится даже если `throw` случится между открытием и закрытием.

**Как ошибки распространяются в async/await?**
> `async throws` функция — бросает ошибку как обычно. `try await dangerousFunc()` внутри do-catch или propagates вверх. CancellationError — бросается автоматически при cancellation. Structured concurrency: ошибка дочерней task = ошибка родительской.

---

### Senior

**Как typed throws изменяет ABI?**
> Untyped throws: ошибка передаётся через `swift_error` register + error type erased. Typed throws: компилятор знает конкретный тип ошибки — может оптимизировать (нет boxing для small error types). ABI изменяется для typed throws функций.

**Как реализовать error mapping между слоями?**
> `func domainFetch() throws(DomainError) { do { try dataSource.fetch() } catch let e as NetworkError { throw DomainError.network(e) } catch { throw DomainError.unknown } }`. Каждый слой имеет свой тип ошибки, маппинг при пересечении границ.

**Как error handling работает в Combine?**
> Publisher с `Failure != Never` — может завершиться ошибкой. `catch { _ in Just(fallback) }` — обработка с fallback. `mapError` — конвертация. После ошибки publisher завершается — нет дальнейших значений.

**Как создать иерархию ошибок для большого приложения?**
> Общий `AppError` enum на верхнем уровне. Feature-specific errors: `NetworkError`, `StorageError`, `AuthError`. Mapper функции при пересечении слоёв. Логирование на уровне handling. `LocalizedError` для UI-сообщений.

**Что такое error recovery vs error propagation?**
> Propagation: throw вверх по стеку — пусть вызывающий решает. Recovery: обработать здесь, вернуть дефолт, повторить попытку. Recovery предпочтительна когда знаешь как восстановиться. Propagation — когда нет.

**Как async throws функции оптимизированы компилятором?**
> В SIL: throwing путь — отдельная ветка. Компилятор оптимизирует happy path (без ошибки) отдельно. Error allocation: мелкие ошибки могут хранится на стеке (stack allocation) вместо heap. Typed throws дают больше возможностей оптимизации.

---

## Optional

### Beginner

**Что такое Optional в Swift?**
> Тип-обёртка для значения которое может отсутствовать. `Optional<Int>` = `Int?`. Либо `.some(value)`, либо `.none` (nil). Компилятор не позволяет использовать Optional как non-Optional без явного извлечения.

**Как объявить Optional переменную?**
> `var name: String? = "John"` или `var age: Int? = nil`. `?` после типа = Optional. По умолчанию Optional переменная без инициализации = nil.

**Что такое `nil`?**
> Отсутствие значения. В Swift — только для Optional типов. Не то же что NULL в C (указатель на 0) или null в Java. `nil` — литерал для `.none` case Optional enum.

**Что такое optional binding (`if let`)?**
> Безопасное извлечение Optional: `if let name = optionalName { print(name) }`. Если Optional содержит значение — создаётся константа name с этим значением. Если nil — блок пропускается.

**Что такое force unwrap (`!`)?**
> `optionalValue!` — принудительно извлекает значение. Краш если nil. Избегай в production. Использовать только когда 100% гарантировано ненулевое значение (редко). Замени на `guard let` или `if let`.

**Что такое optional chaining (`?.`)?**
> `user?.address?.city` — безопасная цепочка вызовов. Если любое звено nil — вся цепочка возвращает nil. Не бросает ошибку, не крашится. Результат — Optional.

**Что такое nil coalescing (`??`)?**
> `optionalValue ?? defaultValue` — если Optional nil, использует default. `let name = user.name ?? "Anonymous"`. Правый операнд вычисляется лениво (только если левый nil). Возвращает non-optional тип.

---

### Middle

**Как работает `guard let`?**
> `guard let value = optional else { return }`. При nil — выполняет else блок (обязательный выход). При не-nil — value доступна в оставшейся части функции. Инвертирует логику — happy path без вложенности.

**Что такое multiple optional binding?**
> `if let a = optA, let b = optB { }` — проверяет несколько Optional. Если хоть один nil — блок не выполняется. Короче чем вложенные if let. Можно смешивать: `if let a = optA, b > 0 { }`.

**Как Optional реализован в Swift (enum)?**
> `enum Optional<Wrapped> { case none; case some(Wrapped) }`. `nil` = `.none`. `"hello"` = `.some("hello")`. Все методы (`map`, `flatMap`) — реализованы через это enum. Синтаксический сахар скрывает детали.

**Что такое Optional chaining с функциями?**
> `user?.greet()` — если user nil, greet не вызывается, результат nil. Если greet() -> String, то `user?.greet()` -> `String?`. Если greet() -> Void, то `user?.greet()` -> `Void?` (игнорируется обычно).

**Что такое `flatMap` для Optional?**
> `optional.flatMap { value in anotherOptional(value) }`. Если optional nil → nil. Если some — передаёт value в замыкание. Если замыкание возвращает nil → nil. Избегает `Optional<Optional<T>>`.

**Как `map` работает для Optional?**
> `optional.map { value in transform(value) }`. Если nil → nil. Если some → применяет transform и оборачивает в some. `"5".map { Int($0) }` → `Optional<Optional<Int>>` (поэтому предпочитают flatMap для фаллибл трансформаций).

**Что такое implicitly unwrapped optional (`!` в типе)?**
> `var label: UILabel!` — Optional который автоматически unwrap при доступе. Не нужен `?` или `!` при использовании: `label.text = "x"`. Краш если nil при доступе. Нужен для @IBOutlet и двухфазной инициализации.

**Когда использовать implicitly unwrapped optional?**
> @IBOutlet — не nil после viewDidLoad. Dependency injection через init, где after-init гарантировано не nil. Avoid в обычном коде — риск краша. Предпочитай обычный Optional или non-optional.

**Как Optional влияет на Codable?**
> Optional свойство — если ключ отсутствует в JSON или null → nil. Non-optional — ошибка декодирования если ключ отсутствует или null. Можно переопределить через custom `init(from decoder:)`.

**Что такое optional protocol requirements?**
> `@objc protocol Delegate { @objc optional func method() }`. Только в `@objc` протоколах. Реализация необязательна. Вызов через optional chaining: `delegate?.method?()`. Избегай — предпочитай default implementation.

---

### Senior

**Как компилятор оптимизирует Optional?**
> Для reference types: Optional не добавляет дополнительной памяти — nil = нулевой указатель. Для Bool: 1 байт, nil = особое значение. Для enum Optional<MyEnum> — компилятор использует spare bits enum для кодирования none. Нет лишней памяти.

**Что такое `Optional<Optional<T>>` и как это обрабатывать?**
> `String?? = Optional<Optional<String>>`. Три состояния: `.none`, `.some(.none)`, `.some(.some(value))`. Возникает при `map` с Optional-возвращающим замыканием. Используй `flatMap` для схлопывания уровней.

**Как Optional взаимодействует с generics?**
> `func first<T>(_ array: [T]) -> T?` — generic Optional. Компилятор специализирует для конкретных T. `T?` как generic constraint: `where T? == U` — возможно. `Optional<T>` conforming protocols — через conditional conformance.

**Как `switch` работает с Optional?**
> `switch optionalValue { case .some(let v): ...; case .none: ... }` или `switch optionalValue { case let v?: ...; case nil: ... }`. `v?` — shorthand для `.some(let v)`. Pattern matching полностью поддерживается.

**Что такое optional tuple?**
> `var pair: (Int, String)? = nil`. Optional tuple — не то же что tuple Optionals. `(Int?, String?)` — всегда существует tuple, поля могут быть nil. `(Int, String)?` — весь tuple может быть nil.

---

## String & Collections

### Beginner

**Как создать строку в Swift?**
> `let s = "Hello"` — string literal. `let s = String(42)` — из числа. `let s = String(data: data, encoding: .utf8)` — из Data. Многострочная: `let s = """..."""`.

**Что такое String interpolation?**
> `"Hello, \(name)! Age: \(age)"` — встраивание выражений в строку. Компилятор превращает в серию append. Можно форматировать: `"\(value, specifier: "%.2f")"`. Работает с любым `CustomStringConvertible`.

**Что такое `Character` в Swift?**
> Один графемный кластер Unicode. `"é"` — один Character (e + combining accent). `"🇺🇸"` — один Character (два Unicode scalar). Не байт, не Unicode scalar — именно видимый символ.

**Как получить длину строки?**
> `string.count` — количество Characters (не байт). Для UTF-8 байт: `string.utf8.count`. Для UTF-16: `string.utf16.count`. Предпочитай `isEmpty` для проверки вместо `count == 0`.

**Что такое `String.Index`?**
> Непрозрачный индекс позиции в строке. Не `Int` — нельзя `string[2]`. Используется: `string[string.startIndex]`, `string.index(after: idx)`, `string[range]`. Отражает что String — коллекция Characters.

**Как сравнивать строки?**
> `==` — точное равенство (регистрозависимое). `string.lowercased() == other.lowercased()` — без учёта регистра. `string.compare(other, options: [.caseInsensitive])` для сложных сравнений. `<` для лексикографического порядка.

---

### Middle

**Почему доступ к символам строки по индексу не O(1) в Swift?**
> String хранится в UTF-8 (variable width encoding). Разные символы занимают 1-4 байта. Чтобы найти N-й Character — нужно пройти с начала (O(n)). `string[string.index(string.startIndex, offsetBy: 5)]` — O(n), не O(1).

**Что такое `Substring`?**
> Срез строки: `let sub = string[range]` — возвращает Substring. Разделяет buffer с оригиналом (нет копирования). Преобразование: `String(sub)` — создаёт копию. Не храни Substring долго — удерживает весь original String.

**Как работает String encoding в Swift?**
> Нативное хранение: UTF-8 (или ASCII-optimized). `.utf8` view — байты UTF-8. `.utf16` view — для ObjC/NSString interop. `.unicodeScalars` — Unicode scalar values. `.characters` (по умолчанию) — grapheme clusters.

**Что такое `StringProtocol`?**
> Протокол которому соответствуют и `String` и `Substring`. Позволяет писать функции принимающие оба: `func process<S: StringProtocol>(_ s: S)`. Избегает конвертацию Substring → String в API.

**Как эффективно конкатенировать строки?**
> `+` оператор: создаёт новую строку при каждом вызове — O(n). `+=`: при уникальном владении — in-place (COW). Много конкатенаций: `components.joined(separator: ",")` или через Array + joined. `String(repeating:count:)` для повторений.

**Что такое `NSString` и как он связан с `String`?**
> NSString — ObjC class для строк. Swift String ↔ NSString bridging автоматически. `string as NSString`, `(nsstring as String)`. NSString — UTF-16 internally. Swift String — UTF-8. Bridging при необходимости копирует.

**Как работать с Unicode в Swift?**
> Unicode scalar: `\u{1F600}` 😀. Combining characters: `"e\u{301}"` == `"é"` (оба один Character). `string.unicodeScalars` — для работы с scalar values. Normalization: `string.precomposedStringWithCanonicalMapping`.

**Что такое grapheme cluster?**
> Минимальная единица читаемого текста. Один или несколько Unicode scalars составляют grapheme cluster = один Character. `"🏳️‍🌈"` — один Character но несколько scalars (flag + rainbow + ZWJ sequences).

**Что такое `String.UTF8View`, `String.UTF16View`, `String.UnicodeScalarView`?**
> Разные представления (views) одной строки. `.utf8` — байты, для сетевого протокола/JSON. `.utf16` — для NSString interop, JavaScript. `.unicodeScalars` — Unicode code points. Каждый view — разный count для той же строки.

**Как работает string interpolation под капотом?**
> Компилятор трансформирует `"\(x) items"` в: `var result = ""; result += String(describing: x); result += " items"`. Через `DefaultStringInterpolation`. Можно кастомизировать через `StringInterpolationProtocol`.

---

### Senior

**Как Swift хранит строки в памяти (small string optimization)?**
> Small String Optimization (SSO): строки до 15 UTF-8 байт хранятся прямо в double-word (16 байт) без heap allocation. Флаг в старшем байте указывает на SSO vs heap. Крупные строки — heap-allocated buffer с ref counting (COW).

**Как `String` реализует `Collection` протокол?**
> `String` conforming к `Collection<Character>`. Index — не Int а `String.Index` (internal byte offset). `count` — O(n) (нужно подсчитать grapheme clusters). `startIndex`, `endIndex`, `index(after:)`. Поддерживает slicing через `string[range]`.

**Как работает строковый literal в Swift?**
> String literals — `StaticString` под капотом до конвертации в String. Компилятор хранит их в binary как UTF-8. При runtime: оборачивается в Swift String с pointer на binary data (без copy). Immutable string literals — immortal objects.

**Что такое Swift Regex (`/pattern/`)?**
> Compile-time строго-типизированные регулярные выражения. `/(\d+)-(\w+)/` — тип `Regex<(Substring, Substring, Substring)>`. Matches содержат именованные captured groups. Быстрее чем NSRegularExpression. iOS 16+.

**Как работает `Regex` builder?**
> `let regex = Regex { OneOrMore(.digit); "-"; Capture { OneOrMore(.word) } }`. Composable через result builder. Каждый компонент — `RegexComponent`. Compile-time типизация captures. Читаемее чем pattern string.

**Как эффективно парсить большие строки?**
> Не создавай Substring за Substring (удерживают оригинал). Используй `String.Index` напрямую. `Scanner` для structured parsing. Для binary data — `Data` вместо String. Stream parsing — не загружай всё в память.

---

## Collection Protocols

### Beginner

**Что такое `Collection` протокол?**
> Sequence с multi-pass и indicial access. `startIndex`, `endIndex`, subscript. Позволяет повторно итерировать. `Array`, `String`, `Dictionary` — Collection. `count`, `isEmpty`, `first`, `last`.

**Что такое `Sequence` протокол?**
> Тип который можно итерировать один раз: `makeIterator() -> Iterator`. `for-in` работает с Sequence. Нет гарантии multi-pass. `map`, `filter`, `reduce` — методы Sequence.

**В чём разница между `Sequence` и `Collection`?**
> Sequence — single-pass, нет индексов, нет `count`. Collection — multi-pass, индексированный доступ, `startIndex`/`endIndex`. Все Collection — Sequence, не наоборот. Network stream — Sequence но не Collection.

**Что такое `IteratorProtocol`?**
> `protocol IteratorProtocol { associatedtype Element; mutating func next() -> Element? }`. Базовый протокол итерации. `next()` возвращает следующий элемент или nil при завершении. Sequence использует IteratorProtocol.

---

### Middle

**Что такое `RandomAccessCollection`?**
> Collection с O(1) доступом по индексу и O(1) подсчётом расстояния между индексами. `Array` — RandomAccessCollection. `Dictionary` — нет (нет стабильного порядка). Нужно для эффективного binary search.

**Что такое `BidirectionalCollection`?**
> Collection которую можно итерировать в обоих направлениях: `index(before:)`. `Array`, `String` — Bidirectional. Позволяет `reversed()`, `last`, `dropLast`. `Dictionary` — нет (нет порядка).

**Что такое `MutableCollection`?**
> Collection позволяющая изменять элементы через subscript setter: `collection[index] = newValue`. `sort()` — требует MutableCollection. Нет требований к добавлению/удалению, только изменение существующих.

**Что такое `RangeReplaceableCollection`?**
> Collection поддерживающая добавление, удаление, замену диапазонов: `append`, `insert`, `remove`, `replaceSubrange`. `Array` и `String` — RangeReplaceableCollection. Полный набор мутирующих операций.

**Как реализовать собственную коллекцию?**
> Минимум: conformance к Collection: `startIndex`, `endIndex`, `subscript`, `index(after:)`. Дополнительно: BidirectionalCollection (`index(before:)`), RandomAccessCollection (`index(_:offsetBy:)`). Синтезируются: `count`, `isEmpty`, `map`, `filter` и др.

**Что такое `LazySequence` и когда её использовать?**
> `.lazy` property: `array.lazy.map { }.filter { }` — трансформации не вычисляются пока не нужен результат. Не создаёт промежуточные массивы. Эффективно для long chains на больших коллекциях. Только нужные элементы вычисляются.

**Как `map`, `filter`, `reduce` реализованы для Sequence?**
> В extension на Sequence. `map` — итерирует через makeIterator, трансформирует через замыкание, собирает в Array. `filter` — предикат, включает подходящие. `reduce` — аккумулирует через замыкание. Все через for-in loop внутри.

**Что такое `compactMap` и `flatMap` для коллекций?**
> `compactMap` — map + убрать nil (для Optional-returning transforms). `flatMap` — map + flatten вложенных коллекций в одну. `[[1,2],[3,4]].flatMap { $0 }` → `[1,2,3,4]`. Разные, не путай.

**Как работает `zip` для двух последовательностей?**
> `zip(s1, s2)` — создаёт Sequence пар `(s1[i], s2[i])`. Останавливается когда короткая заканчивается. Ленивый — не создаёт промежуточный массив. Разные типы: `zip([1,2,3], ["a","b","c"])` → `[(1,"a"), (2,"b"), (3,"c")]`.

**Что такое `AnySequence` и type erasure для Sequence?**
> `AnySequence<Element>` — type erasure wrapper для Sequence. Скрывает конкретный тип. `AnyIterator<Element>` — то же для Iterator. Нужно когда возвращаешь разные типы Sequence из функции.

---

### Senior

**Как реализовать ленивый (lazy) итератор?**
> Реализовать `IteratorProtocol` с `next()` вычисляющим следующее значение по требованию. Бесконечные последовательности: `struct Fibonacci { }; struct FibIterator { var a = 0, b = 1; mutating func next() -> Int? { let result = a; (a, b) = (b, a+b); return result } }`.

**Как Collection индексы влияют на производительность?**
> RandomAccessCollection: `index(_:offsetBy:)` — O(1). BidirectionalCollection без RandomAccess: `index(after:)` × n раз = O(n). `String.Index` — byte offset, но подсчёт n-й character — O(n). Используй правильный протокол для нужной производительности.

**Что такое slicing в коллекциях?**
> `collection[lower..<upper]` — возвращает `Slice<Collection>` (или специализированный SubSequence). Не копирует данные — view на оригинал. `String[range]` → `Substring`. `Array[range]` → `ArraySlice<T>`. Эффективно, но удерживает оригинал.

**Как `forEach` отличается от `for-in`?**
> Оба итерируют. Разница: `return` в `forEach { }` возвращает из замыкания, не из внешней функции. `break`/`continue` недоступны. `for-in` — контроль потока. `forEach` — функциональный стиль, лучше для chaining.

**Что такое `withContiguousStorageIfAvailable`?**
> Метод Sequence: если элементы хранятся в непрерывной памяти — вызывает замыкание с UnsafeBufferPointer. Позволяет SIMD оптимизации, прямой доступ к памяти. `Array` — всегда contiguous. `AnySequence` — не гарантировано.

---

## Hashable, Equatable, Comparable

### Beginner

**Что такое `Equatable`?**
> Протокол для сравнения на равенство: `static func == (lhs: Self, rhs: Self) -> Bool`. Нужен для `==` и `!=` операторов. Компилятор автосинтезирует для struct/enum если все поля Equatable.

**Что такое `Hashable`?**
> Протокол для вычисления хеш-значения: `func hash(into hasher: inout Hasher)`. Требует Equatable. Нужен для ключей Dictionary и элементов Set. Равные объекты обязаны иметь одинаковый хеш.

**Что такое `Comparable`?**
> Протокол для упорядочивания: `static func < (lhs: Self, rhs: Self) -> Bool`. Достаточно реализовать `<` — остальные операторы (>, <=, >=) синтезируются. Нужен для `sort()` без компаратора.

**Как Swift автоматически синтезирует `Equatable`?**
> Если тип соответствует Equatable и все хранимые свойства Equatable — компилятор генерирует `==` сравнивающий все поля по порядку. Достаточно `struct Point: Equatable { var x, y: Int }`.

---

### Middle

**Как реализовать `Equatable` вручную?**
```swift
struct Money: Equatable {
    var amount: Decimal
    var currency: String
    static func == (lhs: Money, rhs: Money) -> Bool {
        lhs.currency == rhs.currency && lhs.amount == rhs.amount
    }
}
```

**Что такое `Identifiable`?**
> Протокол с `id: ID` — уникальный идентификатор экземпляра. `protocol Identifiable { associatedtype ID: Hashable; var id: ID { get } }`. Используется в SwiftUI List, ForEach для отслеживания идентичности элементов.

**Как `Hashable` используется в `Set` и `Dictionary`?**
> Set и Dictionary используют хеш для быстрого поиска O(1). Хеш → bucket индекс. В bucket — линейный поиск через `==`. Хороший хеш = равномерное распределение = мало коллизий = быстрый доступ.

**Что такое `hash(into:)` функция?**
> Метод Hashable: `func hash(into hasher: inout Hasher) { hasher.combine(field1); hasher.combine(field2) }`. `Hasher` — accumulator хеша. `combine` добавляет вклад поля. Результат: `hasher.finalize()`.

**Что такое hash collisions и как они влияют на производительность?**
> Коллизия: разные ключи с одинаковым хешем. В том же bucket — линейный поиск через ==. Много коллизий → O(n) вместо O(1). Плохая хеш-функция (все хеши одинаковые) → Set/Dictionary как список.

**Как `Comparable` используется в сортировке?**
> `array.sorted()` — требует Comparable элементов. Использует `<` для сравнения. `array.sorted { $0 < $1 }` — явный компаратор. `min()`, `max()` — тоже требуют Comparable.

**Что такое `SortComparator`?**
> Swift 5.5+: протокол для переиспользуемых компараторов. `KeyPathComparator(\User.name)`, `KeyPathComparator(\User.age, order: .reverse)`. Можно комбинировать: `[nameComparator, ageComparator]`. Используется в `sorted(using:)`.

**Как реализовать custom ordering с `Comparable`?**
> `extension Priority: Comparable { static func < (lhs: Priority, rhs: Priority) -> Bool { lhs.rawValue < rhs.rawValue } }`. Или через enum с Int raw value + автоматический Comparable. Или кастомная логика порядка.

**Что означает правило что равные объекты должны иметь одинаковый hash?**
> Требование Hashable: `a == b → a.hashValue == b.hashValue`. Нарушение: объект в Set не найдётся после изменения хеша или при неправильной реализации. Обратное не обязательно: разные объекты могут иметь одинаковый хеш (коллизия OK).

**Как `Hashable` влияет на `Dictionary` performance?**
> Хороший хеш: O(1) в среднем. Все в одном bucket (плохой хеш): O(n). Автосинтез хеша в Swift использует `SipHash` — хорошее распределение. Для custom типов: combine все поля участвующие в `==`.

---

### Senior

**Как Swift рандомизирует хеши для безопасности (hash seed)?**
> `Hasher` использует random seed генерируемый при запуске процесса. Защита от HashDoS атак (намеренные коллизии). `hashValue` разный для одного объекта в разных запусках. Не используй hashValue для персистентного хранения.

**Как реализовать `Hashable` для типа с floating point?**
> Проблема: NaN != NaN, -0.0 == +0.0 но разные bits. `hash(into:)` для Float: `hasher.combine(value.bitPattern)` — нарушит правило для NaN. Лучше: `hasher.combine(value == 0 ? 0 : value.bitPattern)`. Или запретить NaN в типе.

**Как hash collision разрешается в Swift Dictionary?**
> Open addressing или separate chaining (зависит от реализации). Swift использует Robin Hood hashing — линейное зондирование с перестановкой для уменьшения variance. Load factor ~75% — resize при превышении.

**Что такое `Hasher` и как он работает?**
> `Hasher` — накопитель хеша. Использует SipHash-1-3 алгоритм с random seed. `combine<H: Hashable>(_ value: H)` — добавляет вклад. `finalize() -> Int` — финальный хеш. Порядок combine важен — `combine(a); combine(b) ≠ combine(b); combine(a)`.

---

*Конец раздела Swift*
