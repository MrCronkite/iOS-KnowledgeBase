# UIKit & SwiftUI — Вопросы и ответы
> iOS-собеседование · UIKit · SwiftUI · AutoLayout · Animation
> Уровни: Beginner · Middle · Senior

---

## UIView & UIViewController

### Beginner

**Что такое UIView?**
> Прямоугольная область на экране. Базовый строительный блок UI: отображает контент, обрабатывает события касания, управляет subviews. Каждый элемент UI (кнопка, лейбл, изображение) — UIView или его подкласс. Использует `CALayer` для рендеринга.

**Что такое UIViewController?**
> Управляет одним экраном (или частью экрана) приложения. Содержит корневой `view`, управляет его жизненным циклом, обрабатывает events, координирует переходы между экранами. Тонкий слой между моделью и view — в идеале.

**Что такое view hierarchy?**
> Дерево UIView объектов: каждый view может иметь superview (родитель) и subviews (дочерние). Рендеринг — рекурсивно сверху вниз. Hit testing — от листьев к корню. `addSubview`, `removeFromSuperview`, `insertSubview(at:)` — управление иерархией.

**Что такое lifecycle UIViewController?**
```
init → loadView → viewDidLoad → viewWillAppear → viewDidAppear
                                     ↑↓
                              viewWillDisappear → viewDidDisappear
                              (→ deinit если deallocated)
```
> `viewDidLoad` — один раз, `viewWillAppear`/`viewDidAppear` — каждое появление на экране.

**В каком методе лучше добавлять subviews?**
> `viewDidLoad` — лучший выбор. Вызывается один раз после загрузки view в память. View уже существует (`self.view != nil`). Не `viewWillAppear` — вызывается каждый раз при появлении, добавишь subviews повторно. Не `init` — view ещё не создан.

**Что такое `viewDidLoad`, `viewWillAppear`, `viewDidAppear`?**
> `viewDidLoad` — view загружен в память (один раз). Настройка UI, setup. `viewWillAppear` — view вот-вот появится (каждый раз). Обновить данные, начать анимации. `viewDidAppear` — view уже виден (каждый раз). Запустить анимации, начать воспроизведение видео.

**Что такое `viewWillLayoutSubviews` и `viewDidLayoutSubviews`?**
> `viewWillLayoutSubviews` — вызывается перед layout subviews, размеры ещё не финальные. `viewDidLayoutSubviews` — после layout, размеры финальные. Здесь можно читать `frame` для настройки зависящей от размера. Вызывается при каждом layout pass (rotation, keyboard).

**Чем `presentViewController` отличается от `pushViewController`?**
> `push` — добавляет в navigation stack (UINavigationController), есть back button, горизонтальная анимация. `present` — modal presentation, появляется поверх, нет автоматического back, различные стили (fullscreen, sheet, popover). Push требует NavigationController.

---

### Middle

**Что такое view controller containment?**
> Механизм встраивания одного VC в другой. `addChild`, `view.addSubview`, `didMove(toParent:)`. Parent VC управляет lifecycle child VC. Примеры: UITabBarController, UINavigationController, UIPageViewController. Позволяет переиспользовать VC как компоненты.

**Как добавить child view controller?**
```swift
func add(_ child: UIViewController) {
    addChild(child)                          // 1. уведомить о добавлении
    view.addSubview(child.view)              // 2. добавить view
    child.view.frame = containerView.bounds // 3. настроить frame
    child.didMove(toParent: self)            // 4. завершить добавление
}

func remove(_ child: UIViewController) {
    child.willMove(toParent: nil)     // 1. уведомить об удалении
    child.view.removeFromSuperview()  // 2. убрать view
    child.removeFromParent()          // 3. завершить удаление
}
```

**Что такое `loadView`?**
> Вызывается когда `view` запрашивается впервые и равен nil. По умолчанию загружает из NIB/Storyboard или создаёт пустой UIView. Override для программного создания view: `override func loadView() { view = MyCustomView() }`. Не вызывай `super.loadView()` при override.

**Чем `viewDidLoad` отличается от `viewWillAppear`?**
> `viewDidLoad`: один раз, view в памяти, данные ещё не актуальны (нет данных из сети). Настройка subviews, constraints, delegates. `viewWillAppear`: каждый раз перед показом. Обновление данных, refresh UI, показ/скрытие navigation bar. Данные должны быть актуальны.

**Что такое `setNeedsLayout` и `layoutIfNeeded`?**
> `setNeedsLayout()` — помечает view как требующий layout на следующем run loop цикле (asynchronous, дёшево). `layoutIfNeeded()` — немедленно выполняет pending layout (synchronous, дорого). Для анимации constraints: изменить constraint → `layoutIfNeeded()` внутри `UIView.animate`.

**Что такое `setNeedsDisplay`?**
> Помечает что содержимое view изменилось и нужно перерисовать на следующем render cycle. Вызывает `draw(_:)` (drawRect). Не вызывай `draw` напрямую. Для view на основе `CALayer` — `setNeedsDisplay()` тоже пометит layer для перерисовки.

**Как работает rendering pipeline в UIKit?**
> 1) UIKit изменяет view tree. 2) `layoutIfNeeded()` выполняет layout pass. 3) `draw(_:)` рисует в backing store. 4) CoreAnimation собирает layer tree. 5) Render server (отдельный процесс) выполняет compositing через Metal/GPU. 6) Результат на экран. Всё в рамках 16ms (60fps).

**Что такое `drawRect` / `draw(_:)`?**
> Метод для custom drawing через CoreGraphics: `override func draw(_ rect: CGRect)`. Используй `UIGraphicsGetCurrentContext()`. Вызывается render system при перерисовке. Дорого: каждый вызов создаёт backing store. Предпочитай `CALayer`/`UIImageView` для статики.

**Что такое `CALayer` и как она связана с UIView?**
> UIView — обёртка над `CALayer`. Каждый UIView имеет backing `layer: CALayer`. Layer — реальный рендерер: хранит bitmap, управляет composite. UIView: gesture handling, lifecycle, AutoLayout. CALayer: визуальное представление, анимации, compositing. Прямая работа с layer — быстрее.

**Как реализовать кастомный UIView?**
```swift
class RatingView: UIView {
    var rating: Int = 0 { didSet { setNeedsLayout() } }
    private var starLayers: [CAShapeLayer] = []
    
    override init(frame: CGRect) {
        super.init(frame: frame)
        setup()
    }
    required init?(coder: NSCoder) {
        super.init(coder: coder)
        setup()
    }
    
    private func setup() { backgroundColor = .clear }
    
    override func layoutSubviews() {
        super.layoutSubviews()
        updateStars()
    }
    
    override var intrinsicContentSize: CGSize { CGSize(width: 150, height: 30) }
}
```

**Что такое `prepareForInterfaceBuilder`?**
> `@IBDesignable` + `prepareForInterfaceBuilder()` — код выполняется в Interface Builder для live preview. Позволяет видеть кастомный view в IB без запуска приложения. `@IBInspectable` — свойства видимые и редактируемые в IB.

---

### Senior

**Как работает hit testing в UIKit?**
> `hitTest(_:with:)` вызывается рекурсивно начиная с `keyWindow`. Алгоритм: если `isHidden`, `alpha < 0.01`, или `!isUserInteractionEnabled` → вернуть nil. Если touch не в `bounds` → nil. Иначе: проверить subviews в обратном порядке (последний добавленный = верхний). Вернуть самый глубокий hit view.

**Что такое `UIResponder chain`?**
> Цепочка объектов для обработки events: UIView → superview → ... → UIViewController → UIWindow → UIApplication → AppDelegate. Event передаётся по цепочке пока не обработан. `UIView` расширяет `UIResponder`. `becomeFirstResponder()` — захватить ввод (клавиатура).

**Как реализовать custom transition?**
```swift
// 1. Animator
class SlideAnimator: NSObject, UIViewControllerAnimatedTransitioning {
    func transitionDuration(...) -> TimeInterval { 0.3 }
    func animateTransition(using ctx: UIViewControllerContextTransitioning) {
        let toView = ctx.view(forKey: .to)!
        ctx.containerView.addSubview(toView)
        toView.frame.origin.x = ctx.containerView.bounds.width
        UIView.animate(withDuration: 0.3, animations: {
            toView.frame.origin.x = 0
        }, completion: { _ in ctx.completeTransition(true) })
    }
}
// 2. Delegate
class VC: UIViewController, UIViewControllerTransitioningDelegate {
    func animationController(forPresented...) -> UIViewControllerAnimatedTransitioning? {
        SlideAnimator()
    }
}
```

**Что такое `UIPresentationController`?**
> Управляет custom presentation: как выглядит presenting VC во время presentation, dimming view, размер presented VC. Создаётся через `UIViewControllerTransitioningDelegate.presentationController(forPresented:...)`. Управляет containerView, добавляет chrome (тень, фон).

**Как работает `UIKit` render loop?**
> CADisplayLink тригерит render loop ~60/120 раз/сек. 1) Commit phase: UIKit commit pending changes (frame, backgroundColor). 2) Render phase (в render server): decode images, rasterize layers. 3) Display phase: compositor combinesayers → framebuffer → экран. Всё за 16.7ms (60fps).

**Что такое offscreen rendering и как избежать?**
> Рендеринг вне основного framebuffer (дополнительный pass на GPU). Причины: `layer.cornerRadius` + `clipsToBounds`, `layer.shadow`, `layer.mask`, `layer.shouldRasterize = false` с opacity. Избегать: `layer.shadowPath` (предрасчитанный путь), `UIBezierPath` clip, `shouldRasterize = true` для статики.

**Что такое `CALayer` backing store?**
> Bitmap buffer в памяти где layer хранит rendered content. Создаётся при `draw(_:)` или при установке `contents`. Размер: `bounds.size × contentsScale`. `drawsAsynchronously = true` — рисовать в background (не всегда помогает). Освобождается при memory pressure.

**Как `CoreAnimation` взаимодействует с `UIKit`?**
> UIKit — высокоуровневая обёртка над CA. `UIView.animate` создаёт `CABasicAnimation`. Каждый UIView change в animation block → implicit CA animation. CA работает в render server (XPC). UIKit на main thread, CA рендеринг — на GPU без блокировки main thread.

**Как реализовать parallax эффект?**
```swift
// Через UIMotionEffect:
let horizontal = UIInterpolatingMotionEffect(keyPath: "center.x", type: .tiltAlongHorizontalAxis)
horizontal.minimumRelativeValue = -20
horizontal.maximumRelativeValue = 20
imageView.addMotionEffect(horizontal)

// Или через scroll:
func scrollViewDidScroll(_ scrollView: UIScrollView) {
    let offset = scrollView.contentOffset.y
    backgroundImageView.transform = CGAffineTransform(translationX: 0, y: offset * 0.5)
}
```

**Как реализовать кастомный `UIControl`?**
```swift
class ThumbSlider: UIControl {
    var value: Float = 0
    
    override func beginTracking(_ touch: UITouch, with event: UIEvent?) -> Bool {
        // touch начался внутри control
        return true  // продолжать tracking
    }
    override func continueTracking(_ touch: UITouch, with event: UIEvent?) -> Bool {
        let point = touch.location(in: self)
        value = Float(point.x / bounds.width)
        sendActions(for: .valueChanged)
        return true
    }
    override func endTracking(_ touch: UITouch?, with event: UIEvent?) {
        sendActions(for: .editingDidEnd)
    }
}
```

**Что такое `UIViewRepresentable` и когда использовать?**
> Протокол для интеграции UIKit view в SwiftUI: `makeUIView(context:)` — создать UIView, `updateUIView(_:context:)` — обновить при изменении state. Нужен когда SwiftUI не имеет аналога: `UITextView`, `MKMapView`, `WKWebView`, кастомные камеры.

---

## UITableView & UICollectionView

### Beginner

**Что такое UITableView?**
> Прокручиваемый список строк (rows) сгруппированных в секции. Вертикальная прокрутка. DataSource предоставляет данные, Delegate обрабатывает события. Использует cell reuse для производительности. Styles: `.plain`, `.grouped`, `.insetGrouped`.

**Что такое UICollectionView?**
> Гибкий прокручиваемый список с кастомным layout. Supports: грид, карусель, waterfall, list. Более мощный чем UITableView. Layout отделён от данных (`UICollectionViewLayout`). Может прокручиваться горизонтально, вертикально, свободно.

**Что такое `UITableViewDataSource`?**
> Протокол предоставляющий данные: `numberOfSections(in:)`, `tableView(_:numberOfRowsInSection:)`, `tableView(_:cellForRowAt:)`. Обязательные методы: последние два. Delegate — для обработки событий (выбор, свайп, высота). DataSource — данные, Delegate — поведение.

**Что такое cell reuse?**
> Оптимизация: вместо создания новой ячейки для каждой строки — переиспользовать невидимые ячейки (уехавшие за экран). `dequeueReusableCell(withIdentifier:)` — взять из пула или создать новую. Экономит память и время создания объектов.

**Как зарегистрировать и dequeue ячейку?**
```swift
// Регистрация (в viewDidLoad):
tableView.register(MyCell.self, forCellReuseIdentifier: "MyCell")
// Или XIB:
tableView.register(UINib(nibName: "MyCell", bundle: nil), forCellReuseIdentifier: "MyCell")

// Dequeue (в cellForRowAt):
let cell = tableView.dequeueReusableCell(withIdentifier: "MyCell", for: indexPath) as! MyCell
cell.configure(with: data[indexPath.row])
return cell
```

---

### Middle

**Как правильно реализовать cell reuse?**
> 1) Всегда регистрировать ячейку. 2) В `cellForRowAt` — dequeue, затем configure. 3) В `prepareForReuse` — сбрасывать все изменяемые состояния. 4) Не хранить IndexPath в ячейке (устаревает при обновлениях). 5) Использовать `dequeueReusableCell(withIdentifier:for:)` (с IndexPath) — не падает при отсутствии регистрации.

**Что такое `prepareForReuse`?**
> Вызывается перед тем как ячейка вернётся из dequeue пула. Место для сброса состояния: отменить изображение загрузку, обнулить labels, снять gesture recognizers. Не вызывай `configure` здесь. Вызывается автоматически при `dequeueReusableCell`.

**Что такое `estimatedRowHeight`?**
> Приблизительная высота строки для быстрого scroll indicator calculation без реального layout всех строк. `tableView.estimatedRowHeight = 80` + `tableView.rowHeight = UITableView.automaticDimension` = self-sizing cells. Неправильный estimate → прыжки scroll indicator.

**Как реализовать self-sizing cells?**
```swift
tableView.rowHeight = UITableView.automaticDimension
tableView.estimatedRowHeight = 80  // приблизительная высота

// В ячейке: constraints top-to-bottom через весь contentView
// Последний constraint должен быть до bottomAnchor contentView
// priority: 999 для bottom constraint если есть конфликт
```

**Что такое `UICollectionViewFlowLayout`?**
> Стандартный layout для сеток: rows/columns с фиксированным или динамическим размером. `itemSize`, `minimumInteritemSpacing`, `minimumLineSpacing`, `sectionInset`. Поддерживает header/footer. Основа для большинства grid layout. Vertical или horizontal scroll direction.

**Что такое `UICollectionViewCompositionalLayout`?**
> Мощный современный layout (iOS 13+). Составной из: Item → Group → Section → Layout. Каждая секция независима. Поддерживает: nested groups, orthogonal scrolling sections, decorations. Заменяет кастомные layouts в большинстве случаев.

```swift
let itemSize = NSCollectionLayoutSize(widthDimension: .fractionalWidth(0.5),
                                      heightDimension: .fractionalHeight(1.0))
let item = NSCollectionLayoutItem(layoutSize: itemSize)
let groupSize = NSCollectionLayoutSize(widthDimension: .fractionalWidth(1.0),
                                       heightDimension: .absolute(80))
let group = NSCollectionLayoutGroup.horizontal(layoutSize: groupSize, subitems: [item])
let section = NSCollectionLayoutSection(group: group)
return UICollectionViewCompositionalLayout(section: section)
```

**Что такое `DiffableDataSource`?**
> UITableViewDiffableDataSource / UICollectionViewDiffableDataSource (iOS 13+). Замена delegate-based DataSource. Автоматически анимирует изменения через diff алгоритм. `apply(snapshot)` — вместо `reloadData`. Thread safe.

**Что такое `NSDiffableDataSourceSnapshot`?**
> Состояние данных для DiffableDataSource: секции и items. `var snapshot = NSDiffableDataSourceSnapshot<Section, Item>(); snapshot.appendSections([.main]); snapshot.appendItems(items); dataSource.apply(snapshot, animatingDifferences: true)`.

**Как реализовать drag & drop в коллекции?**
```swift
// Drag:
collectionView.dragDelegate = self
func collectionView(_ cv: UICollectionView, itemsForBeginning session: UIDragSession, at indexPath: IndexPath) -> [UIDragItem] {
    let item = UIDragItem(itemProvider: NSItemProvider(object: data[indexPath.item] as NSString))
    return [item]
}
// Drop:
collectionView.dropDelegate = self
func collectionView(_ cv: UICollectionView, performDropWith coordinator: UICollectionViewDropCoordinator) {
    // реорганизовать данные, apply snapshot
}
```

**Как оптимизировать прокрутку UITableView?**
> 1) Использовать `estimatedRowHeight` для быстрого layout. 2) Декодировать изображения в background thread. 3) Избегать `clipsToBounds` + `cornerRadius` (offscreen rendering). 4) `prepareForReuse` — отменять pending network requests. 5) Тяжёлые вычисления — в background, UI update — в main. 6) `cell.layer.shouldRasterize = true` для сложных статичных ячеек.

---

### Senior

**Как работает prefetching в UICollectionView?**
> `UICollectionViewDataSourcePrefetching` (iOS 10+): `prefetchItems(at:)` — заранее загрузить данные для upcoming cells. `cancelPrefetching(forItemsAt:)` — отменить если scroll изменил направление. Нужно: `collectionView.isPrefetchingEnabled = true`. Используй для preload images, network requests.

**Что такое `UICollectionViewLayout` и как создать кастомный?**
```swift
class WaterfallLayout: UICollectionViewLayout {
    override func prepare() { /* вычислить атрибуты всех items */ }
    
    override var collectionViewContentSize: CGSize { /* общий размер контента */ }
    
    override func layoutAttributesForElements(in rect: CGRect) -> [UICollectionViewLayoutAttributes]? {
        /* вернуть атрибуты для items в rect */
    }
    
    override func layoutAttributesForItem(at indexPath: IndexPath) -> UICollectionViewLayoutAttributes? {
        /* атрибуты конкретного item */
    }
    
    override func shouldInvalidateLayout(forBoundsChange newBounds: CGRect) -> Bool { true }
}
```

**Как реализовать sticky headers?**
> CompositionalLayout: `section.boundarySupplementaryItems` + `NSCollectionLayoutBoundarySupplementaryItem` с `.orthogonalScrollingBehavior`. Или: `sectionHeadersPinToVisibleBounds = true` в FlowLayout. Или кастомный layout: в `layoutAttributesForElements` — проверять offset и adjusting header frame.

**Как работает `UICollectionViewDiffableDataSource` под капотом?**
> При `apply(snapshot)`: сравнивает новый snapshot с текущим через hashable identifiers. Вычисляет diff: insertions, deletions, moves, reconfigurations. Применяет изменения через batch updates. Нет конфликтов как с `beginUpdates/endUpdates`. Thread safe — apply можно вызывать с любого thread.

**Что такое section snapshot в DiffableDataSource?**
> iOS 14+: `NSDiffableDataSourceSectionSnapshot<Item>` — snapshot для одной секции. Поддерживает иерархические данные (outline): `append(_ items:, to parent:)`. Expand/collapse через `expand/collapse`. Используется с `UICollectionViewListConfiguration` для outline views.

**Как работает reconfiguration vs reload в DiffableDataSource?**
> `reloadItems` — dequeue новую ячейку + полная перерисовка. `reconfigureItems` (iOS 15+) — обновить существующую ячейку на месте без анимации и без dequeue. Быстрее. Использовать когда item identity та же, только данные изменились (счётчик, статус).

**Как оптимизировать декодирование изображений в scroll?**
```swift
// Декодировать в background:
DispatchQueue.global(qos: .userInitiated).async {
    let image = UIImage(data: data)
    // Force decode (draw into CGContext):
    let decodedImage = image?.preparingForDisplay()
    DispatchQueue.main.async {
        cell.imageView.image = decodedImage
    }
}
// Или использовать ImagePipeline (Nuke/Kingfisher) — делают это автоматически
```

**Что такое `UICollectionViewListConfiguration`?**
> iOS 14+: настройка для list-стиля коллекции. `UICollectionLayoutListConfiguration(appearance: .insetGrouped)` создаёт UITableView-подобный layout через UICollectionView. Поддерживает: swipe actions, accessories (disclosure, checkmark), separators, trailing/leading swipe.

**Что такое `CellContentConfiguration` / `UIContentConfiguration`?**
> iOS 14+: протокол для конфигурации внешнего вида ячейки. `UIListContentConfiguration.cell()` — стандартная конфигурация. `var content = cell.defaultContentConfiguration(); content.text = "Title"; content.image = icon; cell.contentConfiguration = content`. Заменяет прямой доступ к `cell.textLabel`.

**Как реализовать infinite scroll?**
```swift
func scrollViewDidScroll(_ scrollView: UIScrollView) {
    let offsetY = scrollView.contentOffset.y
    let contentHeight = scrollView.contentSize.height
    let screenHeight = scrollView.frame.height
    
    if offsetY > contentHeight - screenHeight * 1.5 {
        guard !isLoading else { return }
        isLoading = true
        loadNextPage { [weak self] newItems in
            self?.items.append(contentsOf: newItems)
            self?.applySnapshot()
            self?.isLoading = false
        }
    }
}
```

---

## AutoLayout

### Beginner

**Что такое AutoLayout?**
> Система описания UI через constraints (ограничения) между view. Вместо явных frame — описываешь отношения: "эта кнопка на 16pt от края", "эта view равна по ширине родителю". Адаптируется к разным размерам экрана, orientations, локализациям.

**Что такое constraint?**
> Линейное уравнение: `item1.attribute relation multiplier × item2.attribute + constant`. Например: `button.leading = superview.leading + 16`. `NSLayoutConstraint` — класс. Constraints решаются Cassowary solver → финальные frames.

**Что такое `NSLayoutConstraint`?**
> Класс описывающий constraint: `NSLayoutConstraint(item: button, attribute: .leading, relatedBy: .equal, toItem: superview, attribute: .leading, multiplier: 1, constant: 16)`. Или через anchors (предпочтительно): `button.leadingAnchor.constraint(equalTo: superview.leadingAnchor, constant: 16)`.

**Что такое intrinsic content size?**
> Естественный размер view на основе его содержимого. `UILabel` — размер текста. `UIButton` — текст + инсеты. `UIImageView` — размер изображения. `UIView` — нет (CGSize.zero). AutoLayout использует как "предпочтительный" размер когда нет явных constraints на размер.

**Что такое hugging priority?**
> Content Hugging Priority: насколько view "сопротивляется" растяжению сверх intrinsic content size. Высокий (751) — view остаётся маленьким. Низкий (250) — view растягивается. Используется когда два view с гибкими размерами делят пространство.

**Что такое compression resistance?**
> Content Compression Resistance Priority: насколько view "сопротивляется" сжатию ниже intrinsic content size. Высокий — view не сжимается. Низкий — view сжимается. Противоположно hugging. Label: compression 750 (не обрезать текст), hugging 250 (можно растянуть).

---

### Middle

**Что такое layout ambiguity?**
> Неоднозначность: constraints не определяют уникальное положение/размер view (несколько решений). `view.hasAmbiguousLayout` — проверить. `exerciseAmbiguityInLayout()` — случайно переключить варианты (debug). Причина: недостаточно constraints. Решение: добавить constraints или приоритеты.

**Как отлаживать broken constraints?**
> Console: `[LayoutConstraints] Unable to simultaneously satisfy constraints.` — скопировать адрес: `po 0x6000...` в LLDB. Xcode: `Debug → View Hierarchy`. Методы: `UIView.hasAmbiguousLayout`, breakpoint на `UIViewAlertForUnsatisfiableConstraints`. Добавить `accessibilityIdentifier` для идентификации.

**Что такое Safe Area?**
> Зона экрана не перекрытая системным UI (status bar, navigation bar, tab bar, home indicator). `view.safeAreaInsets` — отступы. `safeAreaLayoutGuide` — для constraints: `view.leadingAnchor.constraint(equalTo: view.safeAreaLayoutGuide.leadingAnchor)`. Заменяет deprecated `topLayoutGuide`.

**Что такое `layoutMargins`?**
> Отступы внутри view для child content: `view.layoutMargins = UIEdgeInsets(top: 8, left: 16, bottom: 8, right: 16)`. `layoutMarginsGuide` — для constraints. `directionalLayoutMargins` — leading/trailing (RTL-aware). UITableViewCell: `contentView.layoutMarginsGuide` для правильных отступов.

**Что такое `UIStackView`?**
> View для автоматического layout subviews в ряд или колонку. `axis`, `distribution`, `alignment`, `spacing`. Не рисует ничего сам — управляет layout других views. Скрытый subview: `view.isHidden = true` → stackView автоматически уберёт из layout. Мощный инструмент уменьшения числа constraints.

**Как `UIStackView` упрощает AutoLayout?**
> Один StackView заменяет N constraints между N views. Автоматически обрабатывает: spacing, alignment, distribution. `isHidden` на subview — layout перестраивается без constraints изменений. `addArrangedSubview` / `removeArrangedSubview` — динамические изменения без constraint hell.

**Что такое priority для constraints?**
> `UILayoutPriority`: 1...1000. Required (1000) — нарушение = лог + игнорирование. Optional (< 1000) — solver старается выполнить, но может нарушить. `defaultHigh` = 750, `defaultLow` = 250. Используй для disambiguation и для constraints которые "должны быть но могут быть нарушены".

**Как анимировать изменение constraints?**
```swift
// Способ 1: изменить constant, потом animate layoutIfNeeded
heightConstraint.constant = 200

UIView.animate(withDuration: 0.3) {
    self.view.layoutIfNeeded()
}

// Способ 2: activate/deactivate constraints
NSLayoutConstraint.deactivate([smallConstraint])
NSLayoutConstraint.activate([largeConstraint])

UIView.animate(withDuration: 0.3) {
    self.view.layoutIfNeeded()
}
```

**Что такое `translatesAutoresizingMaskIntoConstraints`?**
> По умолчанию `true`: UIKit генерирует constraints из `autoresizingMask` для обратной совместимости. Конфликтует с ручными constraints. При программном AutoLayout: `view.translatesAutoresizingMaskIntoConstraints = false` — обязательно перед добавлением constraints. В IB: устанавливается автоматически.

**Как программно создавать constraints?**
```swift
// Anchors (предпочтительный современный способ):
NSLayoutConstraint.activate([
    button.topAnchor.constraint(equalTo: view.safeAreaLayoutGuide.topAnchor, constant: 16),
    button.leadingAnchor.constraint(equalTo: view.leadingAnchor, constant: 16),
    button.trailingAnchor.constraint(equalTo: view.trailingAnchor, constant: -16),
    button.heightAnchor.constraint(equalToConstant: 44)
])
```

**Как реализовать dynamic layout при появлении клавиатуры?**
```swift
NotificationCenter.default.addObserver(self, selector: #selector(keyboardWillShow),
    name: UIResponder.keyboardWillShowNotification, object: nil)

@objc func keyboardWillShow(_ notification: Notification) {
    let keyboardHeight = (notification.userInfo?[UIResponder.keyboardFrameEndUserInfoKey] as? CGRect)?.height ?? 0
    bottomConstraint.constant = -keyboardHeight
    UIView.animate(withDuration: 0.3) { self.view.layoutIfNeeded() }
}
```

---

### Senior

**Как работает constraint solver (Cassowary)?**
> Cassowary — incremental constraint solving algorithm. Представляет constraints как систему линейных уравнений. При изменении одного constraint — пересчитывает только affected views (incremental, не полный пересчёт). O(n) в среднем для sparse constraint systems. Быстрее чем наивный linear solver.

**Как реализовать кастомный layout без AutoLayout?**
```swift
override func layoutSubviews() {
    super.layoutSubviews()
    let width = bounds.width
    let itemWidth = (width - spacing * CGFloat(count + 1)) / CGFloat(count)
    for (i, subview) in arrangedSubviews.enumerated() {
        subview.frame = CGRect(
            x: spacing + CGFloat(i) * (itemWidth + spacing),
            y: spacing,
            width: itemWidth,
            height: bounds.height - spacing * 2
        )
    }
}
```

**Когда AutoLayout становится узким местом производительности?**
> При: сотнях constraints в одном pass, сложных constraint graphs с циклами, frequent layout passes в scroll (каждый frame). Профилировать: Time Profiler → `layoutSubviews`. Решения: кэшировать calculated frames, использовать `systemLayoutSizeFitting` для precalculation, manual layout в hot paths.

**Что такое `UIViewPropertyAnimator` и как он связан с constraints?**
> Интерактивный анимационный движок. `let animator = UIViewPropertyAnimator(duration: 0.3, curve: .easeOut) { self.view.layoutIfNeeded() }`. `animator.fractionComplete = gesture.value` — scrubbing. `animator.continueAnimation(withTimingParameters:durationFactor:)` — продолжить после scrub. Для gesture-driven animations.

**Как реализовать fluid interface с constraint анимациями?**
```swift
// Pan gesture + UIViewPropertyAnimator + scrubbing:
var animator: UIViewPropertyAnimator?

@objc func handlePan(_ gesture: UIPanGestureRecognizer) {
    switch gesture.state {
    case .began:
        topConstraint.constant = expandedY
        animator = UIViewPropertyAnimator(duration: 0.5, dampingRatio: 0.8) {
            self.view.layoutIfNeeded()
        }
        animator?.pauseAnimation()
    case .changed:
        let translation = gesture.translation(in: view).y
        animator?.fractionComplete = translation / (expandedY - collapsedY)
    case .ended:
        animator?.continueAnimation(withTimingParameters: nil, durationFactor: 1)
    default: break
    }
}
```

**Что такое NSLayoutDimension и как использовать multiplier?**
> `button.widthAnchor.constraint(equalTo: view.widthAnchor, multiplier: 0.5)` — ширина = 50% от superview. Multiplier изменить нельзя после создания — нужно деактивировать и создать новый. Используй для responsive layouts без hardcoded размеров.

---

## Animation

### Beginner

**Как создать базовую UIView анимацию?**
```swift
UIView.animate(withDuration: 0.3) {
    self.button.alpha = 0
    self.button.transform = CGAffineTransform(scaleX: 0.9, y: 0.9)
}
```
> Все изменения анимируемых свойств внутри блока — интерполируются. Анимируемые свойства: `frame`, `bounds`, `center`, `transform`, `alpha`, `backgroundColor`.

**Что такое `UIView.animate`?**
> Класс-метод для создания implicit UIKit анимации. Параметры: `duration`, `delay`, `options` (repeat, autoreverse, curveEaseIn и др.), `animations` замыкание (изменения), `completion` (по завершении). Под капотом создаёт `CABasicAnimation` для каждого свойства.

**Что такое duration и delay?**
> `duration` — длительность анимации в секундах. `delay` — задержка перед началом. `UIView.animate(withDuration: 0.3, delay: 0.1, options: [], animations: { }, completion: nil)`. `delay: 0` — немедленный старт. Для staggered animations: разные delays для каждого view.

**Что такое `completion` handler анимации?**
> Замыкание вызываемое когда анимация завершилась. `completion: { finished in if finished { self.doNextStep() } }`. `finished = false` если анимация была прервана (другой анимацией или explicit stop). Используй для chaining анимаций.

---

### Middle

**Что такое spring animation?**
```swift
UIView.animate(
    withDuration: 0.6,
    delay: 0,
    usingSpringWithDamping: 0.6,    // 0=bouncy, 1=no bounce
    initialSpringVelocity: 0.8,     // начальная скорость
    options: .curveEaseOut,
    animations: { self.button.center.y -= 100 }
)
// iOS 17+:
UIView.animate(springDuration: 0.5, bounce: 0.3) {
    self.button.transform = .identity
}
```

**Что такое keyframe animation?**
```swift
UIView.animateKeyframes(withDuration: 1.0, delay: 0) {
    UIView.addKeyframe(withRelativeStartTime: 0, relativeDuration: 0.3) {
        self.view.alpha = 0.5
    }
    UIView.addKeyframe(withRelativeStartTime: 0.3, relativeDuration: 0.4) {
        self.view.transform = CGAffineTransform(scaleX: 1.5, y: 1.5)
    }
    UIView.addKeyframe(withRelativeStartTime: 0.7, relativeDuration: 0.3) {
        self.view.alpha = 1.0
        self.view.transform = .identity
    }
}
```

**Что такое `UIViewPropertyAnimator`?**
> Интерактивный, прерываемый анимационный движок (iOS 10+). Можно: pause, resume, reverse, scrub (изменить `fractionComplete`). `addAnimations` — добавить анимации позже. `addCompletion` — callback. Идеален для gesture-driven animations и interactive transitions.

**Что такое `CAAnimation`?**
> Базовый класс Core Animation анимаций. Иерархия: `CAAnimation` → `CAPropertyAnimation` → `CABasicAnimation` / `CAKeyframeAnimation`. Также: `CAAnimationGroup`, `CATransition`. Работают напрямую с `CALayer`. Не блокируют основной поток — выполняются в render server.

**Что такое `CABasicAnimation`?**
```swift
let animation = CABasicAnimation(keyPath: "opacity")
animation.fromValue = 1.0
animation.toValue = 0.0
animation.duration = 0.3
animation.timingFunction = CAMediaTimingFunction(name: .easeInEaseOut)
animation.fillMode = .forwards
animation.isRemovedOnCompletion = false
layer.add(animation, forKey: "fadeOut")
```

**Что такое `CAKeyframeAnimation`?**
```swift
let bounce = CAKeyframeAnimation(keyPath: "transform.scale")
bounce.values = [1.0, 1.3, 0.9, 1.1, 1.0]
bounce.keyTimes = [0, 0.3, 0.6, 0.8, 1.0]
bounce.duration = 0.5
bounce.timingFunctions = [.easeOut, .easeIn, .easeOut, .easeIn].map {
    CAMediaTimingFunction(name: $0)
}
layer.add(bounce, forKey: "bounce")
```

**Что такое implicit vs explicit Core Animation?**
> Implicit: изменение animatable CALayer property автоматически анимируется (если есть active CATransaction). `layer.opacity = 0` → fade out. Не работает для UIView.layer — UIView отключает implicit animations в actions. Explicit: создаёшь `CAAnimation` явно и добавляешь через `layer.add`.

**Что такое `CATransaction`?**
> Группа CA изменений: `CATransaction.begin(); CATransaction.setAnimationDuration(0.5); layer.opacity = 0; CATransaction.commit()`. Все изменения в transaction — одна batch. Отключить анимацию: `CATransaction.setDisableActions(true)`. UIView.animate создаёт transaction под капотом.

**Как реализовать custom transition между view controllers?**
```swift
// 1. Animator (продолжение из раздела UIViewController)
// 2. Interactive transition:
class InteractiveTransition: UIPercentDrivenInteractiveTransition {
    var hasStarted = false
    var shouldFinish = false
    
    @objc func handlePan(_ gesture: UIPanGestureRecognizer) {
        let progress = gesture.translation(in: gesture.view).x / gesture.view!.bounds.width
        switch gesture.state {
        case .began: hasStarted = true; viewController.dismiss(animated: true)
        case .changed: shouldFinish = progress > 0.5; update(progress)
        case .ended: shouldFinish ? finish() : cancel(); hasStarted = false
        default: cancel()
        }
    }
}
```

**Что такое presentation controller?**
> `UIPresentationController` управляет presentation: containerView, chrome (dimming), размер presented VC. `frameOfPresentedViewInContainerView` — размер. `presentationTransitionWillBegin/DidEnd` — добавить/убрать chrome. Создаётся через `transitioningDelegate.presentationController(forPresented:...)`.

---

### Senior

**Как работает animation rendering pipeline?**
> 1) `CATransaction.commit()` → CA фиксирует изменения. 2) Render server получает layer tree. 3) GPU: rasterize layers → composite → framebuffer. 4) Display controller: framebuffer → экран. Анимации — в render server: CPU не участвует после setup. Поэтому CA анимации не блокируются даже при busy main thread.

**Что такое layer-backed vs non-layer-backed view?**
> На iOS все UIView layer-backed (всегда имеют CALayer). На macOS: `wantsLayer = true` делает NSView layer-backed. Layer-backed: CA compositing, анимации. Non-layer-backed (только macOS): direct drawing в window backing store. iOS: всегда layer-backed — нет разницы.

**Как реализовать particle effect через CAEmitterLayer?**
```swift
let emitter = CAEmitterLayer()
emitter.emitterPosition = CGPoint(x: view.bounds.midX, y: -10)
emitter.emitterShape = .line
emitter.emitterSize = CGSize(width: view.bounds.width, height: 1)

let cell = CAEmitterCell()
cell.birthRate = 5
cell.lifetime = 7.0
cell.velocity = 100
cell.velocityRange = 50
cell.emissionLongitude = .pi  // направление вниз
cell.scale = 0.05
cell.scaleRange = 0.02
cell.contents = UIImage(named: "snowflake")?.cgImage
emitter.emitterCells = [cell]
view.layer.addSublayer(emitter)
```

**Что такое Metal и как он связан с CoreAnimation?**
> Metal — low-level GPU API. CoreAnimation использует Metal (или OpenGL на старых устройствах) для compositing. UIKit rendering: UIKit → CA → Metal Render. Прямое использование Metal: `CAMetalLayer` — layer с Metal drawable. Нужно для: custom rendering, shaders, 3D, высокая частота обновлений.

**Как отлаживать анимацию через Instruments?**
> Core Animation instrument: FPS counter, dropped frames, offscreen rendering (красные области). Time Profiler: `CA::Render::` вызовы. Debug → View Hierarchy: Color Blended Layers (синий = no blend, красный = blend overhead). Color Offscreen Rendered — жёлтый = offscreen pass.

**Что такое `CADisplayLink`?**
```swift
var displayLink: CADisplayLink?

func startAnimation() {
    displayLink = CADisplayLink(target: self, selector: #selector(update))
    displayLink?.preferredFrameRateRange = CAFrameRateRange(minimum: 60, maximum: 120, preferred: 120)
    displayLink?.add(to: .main, forMode: .common)
}

@objc func update(displayLink: CADisplayLink) {
    let deltaTime = displayLink.targetTimestamp - displayLink.timestamp
    // обновить состояние анимации
    setNeedsDisplay()
}

func stopAnimation() { displayLink?.invalidate(); displayLink = nil }
```
> Синхронизирован с display refresh rate. `timestamp` — прошедшее время. `targetTimestamp` — когда следующий frame. Идеален для custom game loops, custom animations синхронизированных с экраном.

**Как реализовать `Animatable` протокол в SwiftUI и использовать в UIKit-like кастомном рендеринге?**
> `CALayer` + `CADisplayLink` для custom animations: обновлять custom property в `update` callback, вычислять interpolated value между start/end на основе `progress`. `CALayer.add(animation, forKey:)` для встроенных CA animations. Для custom layer properties: override `action(forKey:)`.

---

## SwiftUI — State Management

### Beginner

**Что такое `@State` в SwiftUI?**
> Property wrapper для локального изменяемого состояния view. `@State private var count = 0`. При изменении — view перерисовывается. Хранится вне структуры view (в SwiftUI graph). `$count` — `Binding<Int>`. Только для value types (Int, String, Bool, struct).

**Что такое `@Binding`?**
> Reference к state другого view. Двустороннее: изменение propagates обратно. `@Binding var isOn: Bool`. Передаётся через `$`: `Toggle(isOn: $isOn)`. Не владеет данными — только ссылка. Используй когда child view должен изменять state parent.

**Что такое `@ObservedObject`?**
> Property wrapper для внешнего `ObservableObject`. View обновляется при `@Published` изменениях. Не владеет объектом — не создаёт, не хранит lifetime. `@ObservedObject var viewModel: ViewModel`. Объект должен существовать пока view активен.

**Что такое `@StateObject`?**
> Создаёт и владеет `ObservableObject`. Инициализируется один раз и переживает rebuilds view. `@StateObject private var vm = ViewModel()`. В отличие от `@ObservedObject` — object не пересоздаётся при redraw. Использовать когда view должен владеть объектом.

**Что такое `@EnvironmentObject`?**
> Получает `ObservableObject` из SwiftUI environment. `@EnvironmentObject var store: AppStore`. Передаётся через `.environmentObject(store)` выше в иерархии. Не нужно передавать через init chain. Краш если не добавлен в environment.

**В чём разница между `@State` и `@StateObject`?**
> `@State` — для value types (Int, String, struct). `@StateObject` — для reference types (class conforming ObservableObject). `@State` хранит значение. `@StateObject` хранит ссылку на объект. Оба переживают rebuilds view. Оба — ownership (view создаёт и владеет).

---

### Middle

**Чем `@ObservedObject` отличается от `@StateObject`?**
> `@StateObject`: view создаёт объект, владеет им, объект живёт пока view в памяти. `@ObservedObject`: view получает объект снаружи, не владеет, объект может быть уничтожен. Правило: создаёшь сам → `@StateObject`. Получаешь снаружи → `@ObservedObject`.

**Что такое `@Environment`?**
> Читает значение из SwiftUI environment по key type: `@Environment(\.colorScheme) var colorScheme`. Встроенные keys: `\.dismiss`, `\.openURL`, `\.locale`, `\.colorScheme`. Кастомный key через `EnvironmentKey`. Изменяется через `.environment(\.key, value)`.

**Как кастомный `EnvironmentKey` создаётся?**
```swift
struct ThemeKey: EnvironmentKey {
    static let defaultValue: Theme = .light
}

extension EnvironmentValues {
    var theme: Theme {
        get { self[ThemeKey.self] }
        set { self[ThemeKey.self] = newValue }
    }
}

// Использование:
struct ContentView: View {
    @Environment(\.theme) var theme
}
// Установка:
ContentView().environment(\.theme, .dark)
```

**Что такое `@Observable` (iOS 17+)?**
> Swift macro заменяющий `ObservableObject` + `@Published`. `@Observable class ViewModel { var count = 0 }`. Нет нужды в `@Published` — все properties автоматически наблюдаемы. View обновляется только при изменении используемых properties (fine-grained). Нет Combine под капотом.

**Чем `@Observable` отличается от `ObservableObject`?**
> `ObservableObject` + `@Published`: Combine-based, `objectWillChange` publisher, view перерисовывается при любом `@Published` change. `@Observable` (iOS 17+): macro-based, fine-grained tracking (перерисовка только при используемых properties), быстрее, проще. `@StateObject`/`@ObservedObject` → `@State`/обычная переменная.

**Что такое `@Bindable`?**
> iOS 17+: создаёт Binding к `@Observable` классу. `@Bindable var model: MyModel; TextField("Name", text: $model.name)`. Для `ObservableObject` — использовали `$object.property` через `@StateObject`/`@ObservedObject`. `@Bindable` — аналог для нового `@Observable`.

**Что такое `@Published` и как оно связано с ObservableObject?**
> Property wrapper из Combine: при изменении свойства отправляет событие через `objectWillChange` publisher. `class VM: ObservableObject { @Published var count = 0 }`. SwiftUI view подписывается на `objectWillChange` и перерисовывается. Любое `@Published` изменение → redraw всего view.

**Что такое `objectWillChange`?**
> Publisher в `ObservableObject`: `var objectWillChange: ObservableObjectPublisher`. Отправляет `.send()` перед каждым изменением `@Published` свойства. Можно вызвать вручную: `objectWillChange.send()`. SwiftUI view подписывается и перерисовывается. Реализован через Combine.

**Как тестировать SwiftUI views с state?**
```swift
// ViewInspector library или:
func test_counterViewModel() {
    let vm = CounterViewModel()
    XCTAssertEqual(vm.count, 0)
    vm.increment()
    XCTAssertEqual(vm.count, 1)
}

// Для @Observable:
@Test func counterIncrement() async {
    let vm = CounterViewModel()
    vm.increment()
    #expect(vm.count == 1)
}
```

**Что такое `PreferenceKey`?**
> Механизм передачи данных снизу вверх в иерархии view (обратный Direction от Environment). `struct HeightKey: PreferenceKey { static var defaultValue: CGFloat = 0; static func reduce(value: inout CGFloat, nextValue: () -> CGFloat) { value = max(value, nextValue()) } }`. `.preference(key: HeightKey.self, value: height)`. `.onPreferenceChange(HeightKey.self) { height in }`.

---

### Senior

**Как SwiftUI определяет какие views нужно перерисовать?**
> Body вызывается при изменении любой зависимости view (state, binding, env). SwiftUI сравнивает новое и старое дерево view (diffing). Неизменившиеся subtrees пропускаются. `Equatable` conformance на View позволяет оптимизировать через `.equatable()`. Fine-grained с `@Observable` (iOS 17+).

**Что такое identity в SwiftUI?**
> Identity определяет что view "тот же самый" между renders. При сохранении identity — view обновляется (animations, state preserved). При смене identity — view уничтожается и создаётся заново (state сбрасывается). Два типа: structural (position in view tree) и explicit (`.id(value)`).

**Что такое structural identity vs explicit identity?**
> Structural: position в дереве view. `if condition { Text("A") } else { Text("B") }` — два разных positions = две разные identities (даже если выглядят одинаково). Explicit: `.id(value)` — принудительная identity. При смене value — view пересоздаётся (сброс state, animations).

**Как `@State` хранится под капотом?**
> SwiftUI хранит `@State` значения в AttributeGraph — графе зависимостей. При создании view node — создаётся storage slot. При rebuild view — storage переиспользуется (не сбрасывается). Доступ через `_state._value`. Swift runtime знает о @State через property wrapper mechanism.

**Как реализовать unidirectional data flow в SwiftUI?**
```swift
// Action → State → View → Action
enum Action { case increment, decrement, reset }

@Observable class Store {
    private(set) var count = 0
    
    func dispatch(_ action: Action) {
        switch action {
        case .increment: count += 1
        case .decrement: count -= 1
        case .reset: count = 0
        }
    }
}

struct CounterView: View {
    let store: Store
    var body: some View {
        Button("+") { store.dispatch(.increment) }
        Text("\(store.count)")
    }
}
```

**Что такое TCA (The Composable Architecture) и как оно работает?**
> Библиотека от pointfree.co: State + Action + Reducer + Store. `Reducer` — pure function: `(inout State, Action) -> Effect`. `Store` — держит state, принимает actions, запускает effects. Composable: feature = отдельный Reducer, можно compose. Testable: pure reducers легко тестировать.

**Как SwiftUI батчит обновления?**
> SwiftUI собирает все изменения state за один run loop iteration и делает одну перерисовку. `withAnimation { state1 = ..; state2 = .. }` — один animation block. `@Observable` macros — fine-grained: перерисовывает только views использующие изменившееся property. `MainActor` гарантирует serial updates.

**Что такое `withTransaction`?**
> Выполняет изменения с кастомным `Transaction`: `var transaction = Transaction(); transaction.animation = .spring(); withTransaction(transaction) { isExpanded = true }`. `Transaction` — metadata для animations и transitions. Передаётся через preference system. `.transaction { $0.animation = nil }` — отключить анимацию.

**Как реализовать side effects в SwiftUI?**
> `.task {}` — async side effect при появлении view (отменяется при disappear). `.onChange(of: value) { }` — при изменении value. `.onReceive(publisher)` — Combine publisher. В TCA: `Effect` возвращается из Reducer. В MVVM: `ViewModel.load()` вызывается из `.onAppear` или `.task`.

---

## SwiftUI — Navigation

### Beginner

**Что такое `NavigationStack`?**
> iOS 16+ замена `NavigationView`. `NavigationStack(path: $path) { RootView() }`. Программная навигация через `NavigationPath`. Поддерживает deep linking, serializable state. Убирает deprecation warnings от NavigationView.

**Что такое `NavigationLink`?**
> View создающий переход в Navigation stack: `NavigationLink("Detail", destination: DetailView())` или `NavigationLink(value: item) { Label(...) }`. С `value` — нужно `.navigationDestination(for:)`. Работает только внутри NavigationStack/NavigationView.

**Что такое `NavigationPath`?**
> Type-erased коллекция значений representing navigation stack: `@State var path = NavigationPath()`. `path.append(item)` — перейти вперёд. `path.removeLast()` — назад. `path.removeLast(path.count)` — к root. Сериализуется через `Codable` для state restoration.

**Что такое `NavigationSplitView`?**
> iOS 16+: sidebar + detail layout (iPad, Mac). `NavigationSplitView { Sidebar() } detail: { Detail() }`. `NavigationSplitView { List } content: { MidColumn } detail: { DetailView }` — трёхколоночный. Автоматически адаптируется к iPhone (как NavigationStack).

---

### Middle

**Как реализовать deep linking через NavigationStack?**
```swift
@State var path = NavigationPath()

// Обработка URL:
.onOpenURL { url in
    if let route = Route(url: url) {
        path = NavigationPath([route])
    }
}

// Destinations:
.navigationDestination(for: Route.self) { route in
    switch route {
    case .profile(let id): ProfileView(id: id)
    case .settings: SettingsView()
    }
}
```

**Что такое `navigationDestination(for:destination:)`?**
> iOS 16+: регистрирует destination для типа значений в NavigationStack. `NavigationLink(value: user)` + `.navigationDestination(for: User.self) { user in UserView(user: user) }`. Type-safe: разные типы → разные destinations. Работает через весь stack.

**Как программно навигировать через NavigationPath?**
```swift
@State var path = NavigationPath()

// Вперёд:
path.append(UserRoute.profile(id: "123"))

// Назад на один экран:
path.removeLast()

// К корню:
path = NavigationPath()

// Несколько экранов сразу:
path = NavigationPath([Route.home, Route.profile("123")])
```

**Чем NavigationStack отличается от NavigationView?**
> NavigationView (deprecated iOS 16): нет программного контроля стека, проблемы с двойным push, нет type-safe destinations. NavigationStack: programmatic path (`NavigationPath`), type-safe destinations, deep link support, state restoration. Используй NavigationStack для iOS 16+.

**Что такое `toolbar` modifier?**
```swift
.toolbar {
    ToolbarItem(placement: .navigationBarTrailing) {
        Button("Save") { save() }
    }
    ToolbarItem(placement: .navigationBarLeading) {
        EditButton()
    }
    ToolbarItemGroup(placement: .bottomBar) {
        Spacer()
        Button("Action") { }
    }
}
```

**Как реализовать sheet, popover, fullScreenCover?**
```swift
// Sheet:
.sheet(isPresented: $showSheet) { SheetView() }
.sheet(item: $selectedItem) { item in ItemView(item: item) }

// Full Screen:
.fullScreenCover(isPresented: $showFullScreen) { FullView() }

// Popover (iPad или iPhone в compact):
.popover(isPresented: $showPopover) { PopoverContent() }

// Dismiss из presented view:
@Environment(\.dismiss) var dismiss
Button("Close") { dismiss() }
```

**Как реализовать Coordinator паттерн в SwiftUI?**
```swift
@Observable
class AppCoordinator {
    var path = NavigationPath()
    var sheet: SheetDestination?
    
    func showProfile(_ user: User) { path.append(Route.profile(user)) }
    func showSettings() { sheet = .settings }
    func popToRoot() { path = NavigationPath() }
}

struct AppView: View {
    @State var coordinator = AppCoordinator()
    var body: some View {
        NavigationStack(path: $coordinator.path) {
            HomeView()
                .navigationDestination(for: Route.self) { route in
                    coordinator.view(for: route)
                }
        }
        .sheet(item: $coordinator.sheet) { coordinator.sheetView(for: $0) }
        .environment(coordinator)
    }
}
```

---

### Senior

**Как NavigationPath сериализуется?**
> `NavigationPath` содержит `CodableRepresentation` если все элементы Codable. `path.codable` → encode/decode. `UserDefaults.standard.set(try? JSONEncoder().encode(path.codable), forKey: "navPath")`. При восстановлении: `path = NavigationPath(try JSONDecoder().decode(...))`. State restoration на уровне navigation.

**Что такое `Hashable` requirement для NavigationStack?**
> Значения передаваемые в `NavigationLink(value:)` должны быть `Hashable`. SwiftUI использует hash для сравнения и анимации transitions. `Codable` нужен для `NavigationPath` serialization. Enum Routes: `enum Route: Hashable, Codable { case profile(id: String) }`.

**Как реализовать глубокую иерархию навигации?**
> Используй `NavigationPath` с enum: каждый case = экран. `path.append(Route.level1); path.append(Route.level2(data))`. Для cross-feature: coordinator владеет path. Для deep link: парсируй URL в `[Route]`, `path = NavigationPath(routes)`. Не вкладывай NavigationStack друг в друга.

**Как обработать back button в SwiftUI?**
```swift
// Кастомный back button:
.navigationBarBackButtonHidden(true)
.toolbar {
    ToolbarItem(placement: .navigationBarLeading) {
        Button(action: { dismiss() }) {
            HStack { Image(systemName: "chevron.left"); Text("Back") }
        }
    }
}

// Intercepting back gesture:
// Нет прямого API — используй .interactiveDismissDisabled() для sheets
// Для NavigationStack: нет официального API, нужны workarounds
```

---

## SwiftUI — Animations

### Beginner

**Как создать анимацию в SwiftUI?**
```swift
@State var isExpanded = false

Button("Animate") {
    withAnimation(.spring(duration: 0.5)) {
        isExpanded.toggle()
    }
}

RoundedRectangle(cornerRadius: 10)
    .frame(height: isExpanded ? 200 : 100)
    .animation(.easeInOut(duration: 0.3), value: isExpanded)
```

**Что такое `.animation()` modifier?**
> Анимирует изменения specific value: `.animation(.easeIn, value: isVisible)`. Применяется к view: при изменении `isVisible` — анимирует все animatable свойства этого view. Привязан к конкретному значению (безопаснее чем старый `.animation()` без value).

**Что такое `withAnimation`?**
> Функция-обёртка: все state изменения внутри блока → анимация. `withAnimation(.spring()) { isExpanded = true; color = .red }`. В отличие от `.animation(value:)` — анимирует все изменения в блоке. Можно вызвать из любого места (не только в body).

---

### Middle

**Чем `.animation(value:)` отличается от `.animation()`?**
> `.animation()` (deprecated): анимирует ВСЕ изменения view при любом rebuild — непредсказуемо. `.animation(.spring(), value: isExpanded)`: анимирует только когда `isExpanded` изменяется — предсказуемо, безопасно. Всегда используй новый вариант с `value`.

**Что такое `@Namespace`?**
> Property wrapper создающий пространство имён для `matchedGeometryEffect`: `@Namespace private var animation`. Связывает views в разных местах иерархии. Нужен для hero animations между экранами или для анимации перемещения элементов в списке.

**Что такое `matchedGeometryEffect`?**
```swift
@Namespace private var heroAnimation

// Source view:
Image("photo")
    .matchedGeometryEffect(id: "photo", in: heroAnimation)

// Target view (в другом месте/экране):
Image("photo")
    .matchedGeometryEffect(id: "photo", in: heroAnimation)
    .frame(maxWidth: .infinity)
```
> Анимирует переход между двумя views с одинаковым id в одном namespace. Автоматически interpolates position, size, shape. Hero animation эффект.

**Что такое `Transition`?**
> Анимация появления/исчезновения view. `.transition(.slide)`, `.transition(.opacity)`, `.transition(.scale)`. Применяется с `if condition { View().transition(.slide) }` внутри `withAnimation`. SwiftUI анимирует insertion (появление) и removal (исчезновение).

**Как создать кастомный Transition?**
```swift
struct RotateTransition: Transition {
    func body(content: Content, phase: TransitionPhase) -> some View {
        content
            .rotationEffect(.degrees(phase.isIdentity ? 0 : 90))
            .opacity(phase.isIdentity ? 1 : 0)
    }
}

// Использование:
Text("Hello").transition(RotateTransition())
```

**Что такое `AnyTransition`?**
> Type-erased wrapper для `Transition`. `AnyTransition.slide`, `.opacity`, `.scale`. Комбинация: `.asymmetric(insertion: .slide, removal: .opacity)`. `.combined(with:)` — два transition вместе. Используй когда нужно хранить transition в переменной.

**Что такое `Animation.spring`?**
> Физически реалистичная пружинная анимация. `Animation.spring(duration: 0.5, bounce: 0.3)` (iOS 17+). Или: `Animation.spring(response: 0.5, dampingFraction: 0.7)`. `response` — скорость, `dampingFraction` (0=бесконечный bounce, 1=без bounce). `.interpolatingSpring(stiffness:damping:)` — физические параметры.

**Что такое `phaseAnimator` (iOS 17+)?**
```swift
// Анимирует через несколько фаз последовательно:
Text("✓")
    .phaseAnimator([false, true]) { content, isGreen in
        content.foregroundStyle(isGreen ? .green : .primary)
    } animation: { phase in
        phase ? .spring(duration: 0.3) : .linear(duration: 0.1)
    }
```
> Циклически проходит через массив фаз, анимируя каждый переход.

**Что такое `keyframeAnimator` (iOS 17+)?**
```swift
Circle()
    .keyframeAnimator(initialValue: AnimState()) { content, value in
        content
            .offset(y: value.offsetY)
            .scaleEffect(value.scale)
    } keyframes: { _ in
        KeyframeTrack(\.offsetY) {
            LinearKeyframe(-50, duration: 0.3)
            SpringKeyframe(0, duration: 0.4)
        }
        KeyframeTrack(\.scale) {
            LinearKeyframe(1.2, duration: 0.2)
            SpringKeyframe(1.0, duration: 0.5)
        }
    }
```

**Что такое `withTransaction`?**
> Кастомизирует animation для блока изменений: `var t = Transaction(animation: .none); withTransaction(t) { state = newValue }`. Отключить анимацию: `withTransaction(Transaction(animation: nil))`. Передаётся через preference system в дочерние views.

---

### Senior

**Как SwiftUI анимации соотносятся с CoreAnimation?**
> SwiftUI анимации конвертируются в CoreAnimation: `withAnimation(.spring())` → `CASpringAnimation`. SwiftUI отслеживает animatable properties, SwiftUI вычисляет from/to values, CA делает interpolation на GPU. Не все SwiftUI анимации — прямые CA: custom Animatable может вычисляться в CPU.

**Что такое `AnimatableData`?**
> Протокол `Animatable`: `associatedtype AnimatableData: VectorArithmetic`. `animatableData` — текущее значение для interpolation. SwiftUI interpolates между двумя `animatableData` значениями. Для кастомных анимаций нужно conformance.

**Как реализовать `Animatable` протокол?**
```swift
struct WaveShape: Shape, Animatable {
    var amplitude: Double
    var frequency: Double
    
    var animatableData: AnimatablePair<Double, Double> {
        get { AnimatablePair(amplitude, frequency) }
        set { amplitude = newValue.first; frequency = newValue.second }
    }
    
    func path(in rect: CGRect) -> Path {
        // рисуем волну с текущими amplitude и frequency
        var path = Path()
        // ...
        return path
    }
}

// Анимация:
withAnimation(.easeInOut(duration: 2)) {
    amplitude = 30
    frequency = 3
}
```

**Что такое `AnimatablePair`?**
> Composable `VectorArithmetic` для двух значений: `AnimatablePair<Double, Double>`. Для трёх: `AnimatablePair<Double, AnimatablePair<Double, Double>>`. Используется в `animatableData` когда нужно анимировать несколько независимых значений одновременно.

**Как реализовать сложный gesture-driven animation?**
```swift
@State private var offset: CGSize = .zero
@State private var isDragging = false

var drag: some Gesture {
    DragGesture()
        .onChanged { value in
            withAnimation(.interactiveSpring()) {
                offset = value.translation
                isDragging = true
            }
        }
        .onEnded { value in
            withAnimation(.spring(duration: 0.4, bounce: 0.3)) {
                if abs(value.predictedEndTranslation.width) > 100 {
                    offset = CGSize(width: value.predictedEndTranslation.width > 0 ? 400 : -400, height: 0)
                } else {
                    offset = .zero
                }
                isDragging = false
            }
        }
}

Card()
    .offset(offset)
    .scaleEffect(isDragging ? 1.05 : 1.0)
    .gesture(drag)
```

**Что такое `TimelineView` и как использовать для анимации?**
```swift
TimelineView(.animation) { timeline in
    let phase = timeline.date.timeIntervalSinceReferenceDate
    Circle()
        .offset(x: cos(phase) * 50, y: sin(phase) * 50)
}
// .animation schedule — обновляется каждый frame (как CADisplayLink)
// Для custom real-time animations без CADisplayLink
```

---

*Конец документа · UIKit & SwiftUI*
