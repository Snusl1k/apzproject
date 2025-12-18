# CafeOrders Clean Architecture

ASP.NET Core Web API за чистою архітектурою для кафе: меню, категорії, замовлення зі статусами. CQRS (MediatR), FluentValidation, EF Core з SQLite (фолбек InMemory).

## Проекти
- **CafeOrders.Api** – REST, Swagger, health checks.
- **CafeOrders.Application** – CQRS, валідація, DTO, контракти, сервіси, обробники подій.
- **CafeOrders.Domain** – DDD сутності (MenuCategory, MenuItem, Order, OrderLine), статуси замовлення, доменні події.
- **CafeOrders.Infrastructure** – EF Core SQLite (файл `cafeorders.db`) або InMemory, DI, сидування даних, диспетчер подій.

## Архітектурні патерни
- **Clean Architecture** – розділення на шари Domain, Application, Infrastructure, API.
- **Domain-Driven Design (DDD)** – агрегати з фабриками, інваріантами та поведінкою.
- **CQRS** – розділення команд та запитів через MediatR.
- **Event-Driven Design** – доменні події (OrderCreatedEvent) з обробниками для логування та сповіщень.

## Запуск
1. Відновити пакети: `dotnet restore`
2. Зібрати рішення: `dotnet build`
3. Запустити API (SQLite за замовчуванням): `dotnet run --project CafeOrders.Api`
  - Якщо HTTPS порт не визначено, можна вимкнути редирект або явно задати урли: `dotnet run --project CafeOrders.Api --urls "http://localhost:5000;https://localhost:5001"`
4. Swagger: https://localhost:5001/swagger або http://localhost:5000/swagger (після старту).

## Основні ендпоїнти
- `GET /health` – перевірка стану.
- `GET /api/categories` – перелік категорій меню.
- `GET /api/menuitems` – список позицій меню з категоріями.
- `POST /api/menuitems` – створити позицію меню.
  - Body: `{ "name": "Flat White", "description": "...", "price": 3.5, "categoryId": "<guid>" }`
- `GET /api/orders` – список замовлень з рядками та статусом.
- `POST /api/orders` – створити замовлення.
  - Body: `{ "customerName": "Jane", "lines": [{ "menuItemId": "<guid>", "quantity": 2 }] }`
- `PATCH /api/orders/{id}/status` – оновити статус замовлення.
  - Body: `{ "status": "InProgress" }` або Completed/Cancelled/Pending.

## Event-Driven Design
При створенні замовлення через `POST /api/orders`:
1. **Order агрегат** породжує подію `OrderCreatedEvent`.
2. **EventDispatcher** автоматично знаходить та викликає всі зареєстровані обробники:
   - `OrderLoggingEventHandler` – логує деталі замовлення для аудиту.
   - `OrderNotificationEventHandler` – симулює відправку сповіщення (email/SMS/push).
3. Події обробляються асинхронно після збереження замовлення в базу.

Лог виводить інформацію про створене замовлення:
```
📝 [EVENT] Замовлення створено: ID={guid}, Клієнт=Іван, Сума=7.50, Позицій=2
🔔 [NOTIFICATION] Нове замовлення #{guid} від клієнта Іван на суму 7.50. Потрібно опрацювати!
```

## Dependency Injection
Проєкт демонструє всі три життєві цикли DI:
- **Transient** – новий екземпляр кожного разу
- **Scoped** – один екземпляр на HTTP запит (DbContext, Repositories, Services)
- **Singleton** – один екземпляр на весь застосунок (ConfigurationCache)

Тестові endpoints:
- `GET /api/di-lifetime` – детальна демонстрація життєвих циклів
- `GET /api/di-lifetime/quick` – швидка перевірка

Детальніше: [DI-LIFETIMES.md](DI-LIFETIMES.md)

## Generic Repository Pattern
Реалізовано Generic Repository Pattern для уніфікації CRUD операцій:
- **IRepository<T>** – generic інтерфейс з базовими методами
- **Repository<T>** – базова реалізація для всіх сутностей
- **GenericOrderRepository** – конкретний репозиторій з додатковими методами

Базові методи: `GetById`, `GetAll`, `Add`, `Update`, `Delete`, `Find`, `Exists`, `Count`

Демонстраційні endpoints (використовують `IRepository<Order>`):
- `GET /api/repository-demo/orders` – GetAllAsync()
- `GET /api/repository-demo/orders/{id}` – GetByIdAsync()
- `POST /api/repository-demo/orders` – AddAsync()
- `PUT /api/repository-demo/orders/{id}/status` – UpdateAsync()
- `DELETE /api/repository-demo/orders/{id}` – DeleteAsync()
- `GET /api/repository-demo/orders/pending` – FindAsync() з lambda
- `GET /api/repository-demo/statistics` – CountAsync(), ExistsAsync()

## Unit of Work Pattern
Реалізовано Unit of Work для координації кількох репозиторіїв та управління транзакціями:
- **IUnitOfWork** – інтерфейс для координації репозиторіїв
- **UnitOfWork** – реалізація з управлінням транзакціями (Begin, Commit, Rollback)
- Доступ до репозиторіїв: `Orders`, `MenuItems`, `MenuCategories`
- Атомарність: всі зміни зберігаються разом або відкочуються разом

Методи: `SaveChangesAsync`, `BeginTransactionAsync`, `CommitTransactionAsync`, `RollbackTransactionAsync`

Демонстраційні endpoints (координація через `IUnitOfWork`):
- `GET /api/unit-of-work/orders` – доступ до Orders через UnitOfWork
- `GET /api/unit-of-work/statistics` – доступ до 3-х репозиторіїв
- `POST /api/unit-of-work/create-order-with-validation` – координація Orders + MenuItems
- `POST /api/unit-of-work/create-full-workflow` – координація 3-х репозиторіїв (Category → MenuItem → Order)
- `GET /api/unit-of-work/test-rollback` – демонстрація відкату транзакції

## AutoMapper Integration
Проєкт використовує AutoMapper для автоматичного маппінгу Domain сутностей в DTO:
- **OrderMappingProfile** – маппінг Order → OrderDto з обчисленням Total та вкладеними OrderLines
- **MenuItemMappingProfile** – маппінг MenuItem → MenuItemDto з CategoryName
- **MenuCategoryMappingProfile** – маппінг MenuCategory → MenuCategoryDto

Mapping profiles автоматично реєструються через `services.AddAutoMapper(Assembly)`.

Демонстраційні endpoints (використання `IMapper`):
- `GET /api/automapper-demo/orders` – маппінг Order → OrderDto
- `GET /api/automapper-demo/orders/{id}` – маппінг з вкладеними OrderLines
- `GET /api/automapper-demo/menu-items` – маппінг MenuItem → MenuItemDto
- `GET /api/automapper-demo/categories` – маппінг MenuCategory → MenuCategoryDto
- `GET /api/automapper-demo/statistics` – множинний маппінг колекцій
- `POST /api/automapper-demo/orders` – створення замовлення з маппінгом response
- `GET /api/automapper-demo/mapping-comparison` – порівняння підходів Map() vs ProjectTo()

**HTTP тести**: [test-automapper.http](test-automapper.http)

**Детальна документація:** [AUTOMAPPER-PROFILES.md](AUTOMAPPER-PROFILES.md)

## FluentValidation
Повна система валідації для всіх Commands, Queries та DTOs:
- **Валідатори команд (5)** – CreateOrder, UpdateOrderStatus, CreateMenuItem, CreateOrderWithValidation, CreateMenuItemWithValidation
- **Валідатори запитів (8)** – GetOrders, GetOrderById, GetOrdersWithFilter, GetMenuItems, GetMenuItemById, GetMenuItemsByCategory, GetMenuCategories, GetMenuCategoryById
- **Валідатори DTO (5)** – OrderDto, OrderLineDto, MenuItemDto, MenuCategoryDto, CreateOrderLineDto
- **Кастомні розширення (9)** – UkrainianPhoneNumber, MustBeInFuture, ReasonableCafePrice, DuringBusinessHours, NotOlderThanDays, ReasonableOrderQuantity, UkrainianTextLength, RestrictedEmailDomain, NotEmptyWithMaxCount
- **100% покриття** – кожен клас має валідатор

Валідатори автоматично інтегруються з MediatR pipeline.

Демонстраційні endpoints:
- `POST /api/validation-demo/validate-order` – валідація замовлення з детальними помилками
- `POST /api/validation-demo/validate-menu-item` – валідація позиції меню
- `GET /api/validation-demo/test-phone?phone=+380501234567` – тест валідації телефону
- `GET /api/validation-demo/test-delivery-time` – тест валідації часу доставки
- `GET /api/validation-demo/test-price?price=45.50` – тест валідації ціни
- `GET /api/validation-demo/order-validation-rules` – список правил для замовлення
- `POST /api/validation-demo/test-multiple-errors` – демонстрація множинних помилок

**HTTP тести**: [test-validation.http](test-validation.http)

**Детальна документація:** [FLUENTVALIDATION.md](FLUENTVALIDATION.md)
## Caching System
Багатошарова система кешування для підвищення продуктивності:
- **IMemoryCache** – in-memory кешування на рівні сервера
- **ICacheService** – абстракція з додатковими можливостями (GetOrCreate, інвалідація за префіксом)
- **Response Caching** – HTTP кешування на рівні middleware
- **Cache Keys** – централізовані константи для уникнення помилок
- **Cache Settings** – налаштування термінів кешування (5 хв / 15 хв / 1 год / 24 год)

**Переваги**: Зменшення часу відповіді з ~1500ms до ~1ms (speedup 1000-1600x), зниження навантаження на БД на 70-90%

Демонстраційні endpoints:
- `GET /api/cache-demo/basic-demo` – базова робота Set + Get
- `GET /api/cache-demo/get-or-create` – GetOrCreate pattern (викличте двічі для порівняння)
- `GET /api/cache-demo/collection-demo` – кешування колекцій
- `GET /api/cache-demo/performance-comparison` – порівняння performance з/без кешу
- `DELETE /api/cache-demo/invalidate/{key}` – інвалідація конкретного ключа
- `DELETE /api/cache-demo/invalidate-prefix/{prefix}` – масова інвалідація за префіксом
- `GET /api/cache-demo/statistics` – статистика кешу
- `DELETE /api/cache-demo/clear-all` – очистити весь кеш

**HTTP тести**: [test-caching.http](test-caching.http)

**Детальна документація**: [CACHING.md](CACHING.md)

**Memcached**: увімкнути через `Caching:Provider = "Memcached"` у `appsettings*.json` (сервери в секції `Memcached:Servers`).


## Примітки
- SQLite файл `cafeorders.db` у корені; якщо рядок підключення не заданий, використовується InMemory.
- Після старту створюються категорії та стартові позиції меню.
- FluentValidation, MediatR, AutoMapper, сервіси та обробники подій реєструються через `AddApplication()`, контекст та диспетчер через `AddInfrastructure()`.
