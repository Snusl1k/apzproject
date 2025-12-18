# FluentValidation - Розширена система валідації

Документація системи валідації даних з використанням FluentValidation, кастомних розширень та бізнес-правил.

## Огляд

FluentValidation забезпечує type-safe, fluent API для побудови правил валідації. Проєкт використовує FluentValidation для:
- Валідації команд CQRS (MediatR)
- Кастомних бізнес-правил
- Складних умовних валідацій
- Вкладеної валідації колекцій

### Реєстрація в DI

```csharp
services.AddValidatorsFromAssembly(typeof(DependencyInjection).Assembly);
```

Автоматично реєструє всі класи, що наслідують `AbstractValidator<T>`.

---

## Валідатори команд (Commands)

### 1. CreateOrderCommandValidator
**Файл:** `CafeOrders.Application/Orders/Commands/CreateOrderCommand.cs`

```csharp
public class CreateOrderCommandValidator : AbstractValidator<CreateOrderCommand>
{
    public CreateOrderCommandValidator()
    {
        RuleFor(x => x.CustomerName).NotEmpty().MaximumLength(100);
        RuleFor(x => x.Lines).NotEmpty();
        RuleForEach(x => x.Lines).ChildRules(line =>
        {
            line.RuleFor(l => l.MenuItemId).NotEmpty();
            line.RuleFor(l => l.Quantity).GreaterThan(0);
        });
    }
}
```

**Правила:**
- `CustomerName` - обов'язкове, максимум 100 символів
- `Lines` - не порожня колекція
- Кожен `Line.MenuItemId` - обов'язковий
- Кожен `Line.Quantity` - більше 0

### 2. UpdateOrderStatusCommandValidator
**Файл:** `CafeOrders.Application/Orders/Commands/UpdateOrderStatusCommand.cs`

```csharp
public class UpdateOrderStatusCommandValidator : AbstractValidator<UpdateOrderStatusCommand>
{
    public UpdateOrderStatusCommandValidator()
    {
        RuleFor(x => x.OrderId).NotEmpty();
        RuleFor(x => x.Status).IsInEnum();
    }
}
```

**Правила:**
- `OrderId` - обов'язковий GUID
- `Status` - валідне значення enum OrderStatus

### 3. CreateMenuItemCommandValidator
**Файл:** `CafeOrders.Application/MenuItems/Commands/CreateMenuItemCommand.cs`

```csharp
public class CreateMenuItemCommandValidator : AbstractValidator<CreateMenuItemCommand>
{
    public CreateMenuItemCommandValidator()
    {
        RuleFor(x => x.Name).NotEmpty().MaximumLength(100);
        RuleFor(x => x.Description).MaximumLength(500);
        RuleFor(x => x.Price).GreaterThan(0m);
        RuleFor(x => x.CategoryId).NotEmpty();
    }
}
```

**Правила:**
- `Name` - обов'язкове, максимум 100 символів
- `Description` - максимум 500 символів
- `Price` - більше 0
- `CategoryId` - обов'язковий GUID

---

## Валідатори запитів (Queries)

### 1. GetOrdersQueryValidator
**Файл:** `CafeOrders.Application/Orders/Queries/GetOrdersQuery.cs`

```csharp
public class GetOrdersQueryValidator : AbstractValidator<GetOrdersQuery>
{
    public GetOrdersQueryValidator()
    {
        // Query без параметрів - завжди валідний
    }
}
```

**Призначення:** Query без параметрів завжди валідний. Валідатор створений для консистентності системи.

### 2. GetOrderByIdQueryValidator
**Файл:** `CafeOrders.Application/Orders/Queries/GetOrderByIdQuery.cs`

```csharp
public class GetOrderByIdQueryValidator : AbstractValidator<GetOrderByIdQuery>
{
    public GetOrderByIdQueryValidator()
    {
        RuleFor(x => x.OrderId)
            .NotEmpty()
            .WithMessage("ID замовлення обов'язковий");
    }
}
```

**Правила:**
- `OrderId` - обов'язковий GUID

### 3. GetOrdersWithFilterQueryValidator
**Файл:** `CafeOrders.Application/Orders/Queries/GetOrdersWithFilterQuery.cs`

Розширений валідатор для Query з фільтрацією та пагінацією:

```csharp
public record GetOrdersWithFilterQuery(
    string? Status,
    DateTime? FromDate,
    DateTime? ToDate,
    int? PageNumber,
    int? PageSize
);
```

**Правила:**
- `Status` - один з: Pending, InProgress, Completed, Cancelled (опціонально)
- `FromDate` - не старіша за 1 рік (опціонально)
- `ToDate` - має бути ≥ FromDate (опціонально)
- `PageNumber` - більше 0 (опціонально)
- `PageSize` - від 1 до 100 (опціонально)

### 4. GetMenuItemsQueryValidator
**Файл:** `CafeOrders.Application/MenuItems/Queries/GetMenuItemsQuery.cs`

Query без параметрів - завжди валідний.

### 5. GetMenuItemByIdQueryValidator
**Файл:** `CafeOrders.Application/MenuItems/Queries/GetMenuItemByIdQuery.cs`

**Правила:**
- `MenuItemId` - обов'язковий GUID

### 6. GetMenuItemsByCategoryQueryValidator
**Файл:** `CafeOrders.Application/MenuItems/Queries/GetMenuItemsByCategoryQuery.cs`

```csharp
public record GetMenuItemsByCategoryQuery(
    Guid CategoryId,
    int? PageNumber,
    int? PageSize,
    bool? OnlyAvailable
);
```

**Правила:**
- `CategoryId` - обов'язковий GUID
- `PageNumber` - більше 0 (опціонально)
- `PageSize` - від 1 до 100 (опціонально)
- `OnlyAvailable` - опціонально

### 7. GetMenuCategoriesQueryValidator
**Файл:** `CafeOrders.Application/MenuItems/Queries/GetMenuCategoriesQuery.cs`

Query без параметрів - завжди валідний.

### 8. GetMenuCategoryByIdQueryValidator
**Файл:** `CafeOrders.Application/MenuItems/Queries/GetMenuCategoryByIdQuery.cs`

**Правила:**
- `CategoryId` - обов'язковий GUID

---

## Валідатори DTO

### 1. OrderDtoValidator
**Файл:** `CafeOrders.Application/Validation/Validators/OrderDtoValidator.cs`

Комплексний валідатор для OrderDto з вкладеною валідацією OrderLines:

```csharp
public class OrderDtoValidator : AbstractValidator<OrderDto>
{
    public OrderDtoValidator()
    {
        RuleFor(x => x.Id).NotEmpty();
        RuleFor(x => x.CustomerName).NotEmpty().Length(2, 100);
        RuleFor(x => x.Status).NotEmpty().Must(IsValidOrderStatus);
        RuleFor(x => x.CreatedAtUtc).NotEmpty().LessThanOrEqualTo(DateTime.UtcNow);
        RuleFor(x => x.Total).GreaterThanOrEqualTo(0);
        RuleFor(x => x.Lines).NotEmpty().Must(lines => lines.Count <= 50);
        RuleForEach(x => x.Lines).SetValidator(new OrderLineDtoValidator());
    }
}
```

**Правила:**
- `Id` - обов'язковий
- `CustomerName` - обов'язкове, 2-100 символів
- `Status` - валідне значення enum (Pending, InProgress, Completed, Cancelled)
- `CreatedAtUtc` - не в майбутньому
- `Total` - не негативне
- `Lines` - 1-50 позицій, кожна валідується через OrderLineDtoValidator

### 2. OrderLineDtoValidator
**Файл:** `CafeOrders.Application/Validation/Validators/OrderDtoValidator.cs`

Валідатор для окремої позиції замовлення:

```csharp
public class OrderLineDtoValidator : AbstractValidator<OrderLineDto>
{
    public OrderLineDtoValidator()
    {
        RuleFor(x => x.MenuItemId).NotEmpty();
        RuleFor(x => x.MenuItemName).NotEmpty().Length(1, 100);
        RuleFor(x => x.Quantity).ReasonableOrderQuantity(); // 1-50
        RuleFor(x => x.UnitPrice).GreaterThan(0).LessThanOrEqualTo(10000);
        RuleFor(x => x.LineTotal).GreaterThan(0).Equal(x => x.UnitPrice * x.Quantity);
    }
}
```

**Правила:**
- `MenuItemId` - обов'язковий
- `MenuItemName` - обов'язкове, 1-100 символів
- `Quantity` - 1-50 шт
- `UnitPrice` - 0-10000
- `LineTotal` - має дорівнювати UnitPrice * Quantity

### 3. MenuItemDtoValidator
**Файл:** `CafeOrders.Application/Validation/Validators/MenuItemDtoValidator.cs`

```csharp
public class MenuItemDtoValidator : AbstractValidator<MenuItemDto>
{
    public MenuItemDtoValidator()
    {
        RuleFor(x => x.Id).NotEmpty();
        RuleFor(x => x.Name).NotEmpty().Length(3, 100).Matches(@"^[\p{L}\s\d\-'""()]+$");
        RuleFor(x => x.Description).MaximumLength(500);
        RuleFor(x => x.Price).ReasonableCafePrice(); // 0.01-1000
        RuleFor(x => x.CategoryId).NotEmpty();
        RuleFor(x => x.CategoryName).NotEmpty().Length(2, 50);
    }
}
```

**Правила:**
- `Id` - обов'язковий
- `Name` - 3-100 символів, літери/цифри/базова пунктуація
- `Description` - максимум 500 символів
- `Price` - 0.01-1000 грн
- `CategoryId` - обов'язковий
- `CategoryName` - 2-50 символів

### 4. MenuCategoryDtoValidator
**Файл:** `CafeOrders.Application/Validation/Validators/MenuCategoryDtoValidator.cs`

```csharp
public class MenuCategoryDtoValidator : AbstractValidator<MenuCategoryDto>
{
    public MenuCategoryDtoValidator()
    {
        RuleFor(x => x.Id).NotEmpty();
        RuleFor(x => x.Name).NotEmpty().Length(2, 50).Matches(@"^[\p{L}\s\-']+$");
        RuleFor(x => x.Description).MaximumLength(500);
    }
}
```

**Правила:**
- `Id` - обов'язковий
- `Name` - 2-50 символів, тільки літери/пробіли/дефіси/апострофи
- `Description` - максимум 500 символів

### 5. CreateOrderLineDtoValidator
**Файл:** `CafeOrders.Application/Validation/Validators/CreateOrderLineDtoValidator.cs`

Валідатор для CreateOrderLineDto (використовується в CreateOrderCommand):

```csharp
public class CreateOrderLineDtoValidator : AbstractValidator<CreateOrderLineDto>
{
    public CreateOrderLineDtoValidator()
    {
        RuleFor(x => x.MenuItemId).NotEmpty();
        RuleFor(x => x.Quantity).GreaterThan(0).LessThanOrEqualTo(50);
    }
}
```

**Правила:**
- `MenuItemId` - обов'язковий
- `Quantity` - 1-50 шт

---

## Кастомні розширення валідації

**Файл:** `CafeOrders.Application/Validation/CustomValidationExtensions.cs`

### 1. UkrainianPhoneNumber
Валідація українських номерів телефону.

```csharp
RuleFor(x => x.CustomerPhone)
    .UkrainianPhoneNumber();
```

**Допустимі формати:**
- `+380501234567` (13 символів)
- `380501234567` (12 символів)
- `0501234567` (10 символів)

**Приклади:**
```csharp
✅ "+380501234567"
✅ "380671234567"
✅ "0631234567"
✅ "+380 50 123 45 67" // з пробілами
✅ "0(50)123-45-67" // з дужками та дефісами
❌ "123456"
❌ "+380"
❌ "0501234" // занадто короткий
```

### 2. RestrictedEmailDomain
Обмеження email доменів.

```csharp
RuleFor(x => x.Email)
    .RestrictedEmailDomain("gmail.com", "outlook.com", "company.com");
```

**Використання:**
- Обмеження корпоративними доменами
- Блокування безкоштовних email сервісів

### 3. MustBeInFuture
Дата має бути в майбутньому.

```csharp
RuleFor(x => x.DeliveryTime)
    .MustBeInFuture()
    .When(x => x.DeliveryTime.HasValue);
```

**Використання:**
- Дата доставки
- Час бронювання
- Дедлайни

### 4. DuringBusinessHours
Валідація робочих годин (08:00-22:00).

```csharp
RuleFor(x => x.DeliveryTime)
    .DuringBusinessHours()
    .When(x => x.DeliveryTime.HasValue);
```

**Використання:**
- Час доставки
- Час візиту
- Бронювання столиків

### 5. ReasonableCafePrice
Ціна в розумних межах для кафе (0.01 - 1000 грн).

```csharp
RuleFor(x => x.Price)
    .ReasonableCafePrice();
```

**Діапазон:** 0.01₴ - 1000₴

### 6. ReasonableOrderQuantity
Кількість позицій у замовленні (1-50).

```csharp
RuleFor(x => x.Quantity)
    .ReasonableOrderQuantity();
```

**Діапазон:** 1-50 штук

### 7. NotOlderThanDays
Дата не старіша за X днів.

```csharp
RuleFor(x => x.OrderDate)
    .NotOlderThanDays(30);
```

**Використання:**
- Обмеження дат в історії
- Валідність документів

### 8. UkrainianTextLength
Довжина тексту з українськими символами.

```csharp
RuleFor(x => x.Description)
    .UkrainianTextLength(10, 500);
```

**Підтримує:**
- Кириличні символи
- Латиницю
- Цифри
- Базову пунктуацію

### 9. NotEmptyWithMaxCount
Колекція не порожня з максимальною кількістю.

```csharp
RuleFor(x => x.Lines)
    .NotEmptyWithMaxCount<CreateOrderCommand, IReadOnlyCollection<OrderLine>, OrderLine>(20);
```

---

## Розширені валідатори

### CreateOrderWithValidationCommand
**Файл:** `CafeOrders.Application/Orders/Commands/CreateOrderWithValidationCommand.cs`

Демонструє всі типи валідації:

```csharp
public record CreateOrderWithValidationCommand(
    string CustomerName,
    string CustomerPhone,
    string? CustomerEmail,
    DateTime? DeliveryTime,
    IReadOnlyCollection<CreateOrderLineDto> Lines
);

public class CreateOrderWithValidationCommandValidator : AbstractValidator<CreateOrderWithValidationCommand>
{
    public CreateOrderWithValidationCommandValidator()
    {
        // Базова валідація
        RuleFor(x => x.CustomerName)
            .NotEmpty()
            .Length(2, 100)
            .Matches(@"^[\p{L}\s\-']+$");

        // Кастомне розширення
        RuleFor(x => x.CustomerPhone)
            .NotEmpty()
            .UkrainianPhoneNumber();

        // Умовна валідація
        RuleFor(x => x.DeliveryTime)
            .MustBeInFuture()
            .When(x => x.DeliveryTime.HasValue);

        RuleFor(x => x.DeliveryTime)
            .DuringBusinessHours()
            .When(x => x.DeliveryTime.HasValue);

        // Валідація колекції
        RuleFor(x => x.Lines)
            .NotEmpty()
            .Must(lines => lines.Count <= 20);

        // Вкладена валідація
        RuleForEach(x => x.Lines).ChildRules(line =>
        {
            line.RuleFor(l => l.MenuItemId).NotEmpty();
            line.RuleFor(l => l.Quantity).ReasonableOrderQuantity();
        });

        // Складне правило
        RuleFor(x => x.Lines)
            .Must(lines => lines.Select(l => l.MenuItemId).Distinct().Count() == lines.Count)
            .WithMessage("Замовлення містить дублікати позицій меню");
    }
}
```

### CreateMenuItemWithValidationCommand
**Файл:** `CafeOrders.Application/MenuItems/Commands/CreateMenuItemWithValidationCommand.cs`

```csharp
public record CreateMenuItemWithValidationCommand(
    string Name,
    string Description,
    decimal Price,
    Guid CategoryId,
    bool IsAvailable,
    int? PreparationTimeMinutes
);

public class CreateMenuItemWithValidationCommandValidator : AbstractValidator<CreateMenuItemWithValidationCommand>
{
    public CreateMenuItemWithValidationCommandValidator()
    {
        RuleFor(x => x.Name)
            .NotEmpty()
            .Length(3, 100)
            .Matches(@"^[\p{L}\s\d\-'""()]+$");

        RuleFor(x => x.Price)
            .ReasonableCafePrice();

        RuleFor(x => x.PreparationTimeMinutes)
            .InclusiveBetween(1, 120)
            .When(x => x.PreparationTimeMinutes.HasValue);

        // Складне правило: якщо позиція недоступна, час приготування не вказується
        RuleFor(x => x.PreparationTimeMinutes)
            .Null()
            .When(x => !x.IsAvailable)
            .WithMessage("Час приготування не вказується для недоступних позицій");
    }
}
```

---

## Демонстраційний контролер

**Файл:** `CafeOrders.Api/Controllers/ValidationDemoController.cs`

### Endpoints

#### 1. POST /api/validation-demo/validate-order
Валідація замовлення з детальними помилками.

**Request:**
```json
{
  "customerName": "Іван Петренко",
  "customerPhone": "+380501234567",
  "customerEmail": "ivan@example.com",
  "deliveryTime": "2025-12-20T14:30:00",
  "lines": [
    { "menuItemId": "...", "quantity": 2 }
  ]
}
```

**Response (успіх):**
```json
{
  "isValid": true,
  "message": "✅ Замовлення валідне!",
  "validationDetails": {
    "customerName": "Іван Петренко",
    "customerPhone": "+380501234567",
    "itemCount": 1,
    "totalQuantity": 2
  }
}
```

**Response (помилка):**
```json
{
  "isValid": false,
  "message": "❌ Валідація не пройдена",
  "errors": [
    {
      "property": "CustomerPhone",
      "error": "має бути валідним українським номером телефону",
      "attemptedValue": "123456"
    }
  ]
}
```

#### 2. POST /api/validation-demo/validate-menu-item
Валідація позиції меню.

#### 3. GET /api/validation-demo/test-phone?phone=+380501234567
Тестування валідації телефону.

#### 4. GET /api/validation-demo/test-delivery-time?deliveryTime=2025-12-25T15:00
Тестування валідації часу доставки.

#### 5. GET /api/validation-demo/test-price?price=45.50
Тестування валідації ціни.

#### 6. GET /api/validation-demo/order-validation-rules
Отримати список всіх правил валідації для замовлення.

#### 7. GET /api/validation-demo/menu-item-validation-rules
Отримати список всіх правил валідації для позиції меню.

#### 8. POST /api/validation-demo/test-multiple-errors
Демонстрація множинних помилок валідації.

---

## Типи валідацій FluentValidation

### 1. Базові правила (Built-in Rules)

```csharp
RuleFor(x => x.Name).NotEmpty();              // Не порожнє
RuleFor(x => x.Name).NotNull();               // Не null
RuleFor(x => x.Email).EmailAddress();         // Email формат
RuleFor(x => x.Age).GreaterThan(18);          // Більше за
RuleFor(x => x.Price).GreaterThanOrEqualTo(0); // Більше або дорівнює
RuleFor(x => x.Discount).LessThan(100);       // Менше за
RuleFor(x => x.Name).Length(3, 100);          // Довжина рядка
RuleFor(x => x.Description).MaximumLength(500); // Максимальна довжина
RuleFor(x => x.Code).MinimumLength(6);        // Мінімальна довжина
RuleFor(x => x.Status).IsInEnum();            // Валідне значення enum
RuleFor(x => x.Price).InclusiveBetween(0.01m, 1000m); // Діапазон
```

### 2. Регулярні вирази (Regex)

```csharp
RuleFor(x => x.Name)
    .Matches(@"^[\p{L}\s\-']+$")
    .WithMessage("Може містити тільки літери, пробіли, дефіси та апострофи");

RuleFor(x => x.PostalCode)
    .Matches(@"^\d{5}$")
    .WithMessage("Поштовий індекс має містити 5 цифр");
```

### 3. Кастомні правила (Custom Rules)

```csharp
RuleFor(x => x.Lines)
    .Must(lines => lines.Select(l => l.MenuItemId).Distinct().Count() == lines.Count)
    .WithMessage("Замовлення містить дублікати");

RuleFor(x => x.Password)
    .Must(password => ContainsUpperCase(password) && ContainsDigit(password))
    .WithMessage("Пароль має містити великі літери та цифри");
```

### 4. Умовні правила (Conditional Validation)

```csharp
// Валідація тільки якщо умова виконується
RuleFor(x => x.DeliveryTime)
    .MustBeInFuture()
    .When(x => x.DeliveryTime.HasValue);

// Інакше (Unless)
RuleFor(x => x.ShippingAddress)
    .NotEmpty()
    .Unless(x => x.IsPickup);
```

### 5. Вкладена валідація (Nested Validation)

```csharp
// Валідація кожного елемента колекції
RuleForEach(x => x.Lines).ChildRules(line =>
{
    line.RuleFor(l => l.MenuItemId).NotEmpty();
    line.RuleFor(l => l.Quantity).GreaterThan(0);
});

// Валідація через окремий валідатор
RuleFor(x => x.Address).SetValidator(new AddressValidator());
```

### 6. Колекції (Collections)

```csharp
RuleFor(x => x.Lines)
    .NotEmpty()
    .WithMessage("Замовлення має містити хоча б одну позицію");

RuleFor(x => x.Items)
    .Must(items => items.Count <= 20)
    .WithMessage("Максимум 20 позицій");
```

### 7. Порівняння властивостей (Property Comparison)

```csharp
RuleFor(x => x.EndDate)
    .GreaterThan(x => x.StartDate)
    .WithMessage("Дата завершення має бути пізніше дати початку");

RuleFor(x => x.PasswordConfirmation)
    .Equal(x => x.Password)
    .WithMessage("Паролі не співпадають");
```

---

## Best Practices

### ✅ Рекомендації

1. **Відокремлюйте валідатори**
   - Один валідатор на один Command/DTO
   - Не змішуйте бізнес-логіку з валідацією

2. **Використовуйте кастомні розширення**
   - Створюйте reusable extension methods для спільних правил
   - Зберігайте в `Validation/CustomValidationExtensions.cs`

3. **Зрозумілі повідомлення**
   ```csharp
   // ✅ Зрозуміло
   .WithMessage("Номер телефону має бути в форматі +380XXXXXXXXX")
   
   // ❌ Незрозуміло
   .WithMessage("Invalid phone")
   ```

4. **Умовна валідація**
   - Використовуйте `.When()` для optional полів
   - Використовуйте `.Unless()` для виключень

5. **DRY принцип**
   ```csharp
   // ✅ Reusable extension
   RuleFor(x => x.Phone).UkrainianPhoneNumber();
   
   // ❌ Дублювання коду
   RuleFor(x => x.Phone).Matches(@"^(\+380|380|0)\d{9}$");
   ```

### ❌ Антипатерни

1. **Не валідуйте бізнес-логіку у валідаторах**
   ```csharp
   // ❌ Бізнес-логіка
   RuleFor(x => x.OrderId)
       .Must(id => _repository.Exists(id))
       .WithMessage("Замовлення не існує");
   
   // ✅ Перевірка у Handler
   var order = await _repository.GetByIdAsync(orderId);
   if (order == null) throw new NotFoundException();
   ```

2. **Не робіть DB запити у валідаторах**
   - Валідатори мають бути швидкими
   - DB перевірки виконуйте у Handler

3. **Не ігноруйте помилки валідації**
   - MediatR автоматично викликає валідатори
   - Обробляйте ValidationException в middleware

---

## Інтеграція з MediatR

FluentValidation автоматично інтегрується з MediatR через pipeline behavior:

```csharp
// У DependencyInjection.cs
services.AddMediatR(typeof(DependencyInjection).Assembly);
services.AddValidatorsFromAssembly(typeof(DependencyInjection).Assembly);
```

**Workflow:**
1. Controller отримує Command
2. MediatR Pipeline викликає відповідний Validator
3. Якщо валідація не пройдена → ValidationException
4. Якщо валідація успішна → Handler виконується

---

## HTTP тести

Всі тести доступні в [test-validation.http](test-validation.http):

```http
### Валідне замовлення
POST http://localhost:5000/api/validation-demo/validate-order
Content-Type: application/json

{
  "customerName": "Іван Петренко",
  "customerPhone": "+380501234567",
  "lines": [...]
}

### Некоректний телефон
POST http://localhost:5000/api/validation-demo/validate-order
{
  "customerPhone": "123456"
}

### Тест валідації ціни
GET http://localhost:5000/api/validation-demo/test-price?price=45.50
```

---

## Підсумок

| Категорія | Кількість | Приклади |
|-----------|-----------|----------|
| **Валідатори команд (Commands)** | 5 | CreateOrder, UpdateOrderStatus, CreateMenuItem, CreateOrderWithValidation, CreateMenuItemWithValidation |
| **Валідатори запитів (Queries)** | 8 | GetOrders, GetOrderById, GetOrdersWithFilter, GetMenuItems, GetMenuItemById, GetMenuItemsByCategory, GetMenuCategories, GetMenuCategoryById |
| **Валідатори DTO** | 5 | OrderDto, OrderLineDto, MenuItemDto, MenuCategoryDto, CreateOrderLineDto |
| **Кастомні розширення** | 9 | UkrainianPhoneNumber, MustBeInFuture, ReasonableCafePrice, DuringBusinessHours, NotOlderThanDays, тощо |
| **Demo endpoints** | 8 | validate-order, test-phone, test-price, order-validation-rules, тощо |

**Загальна статистика:**
- ✅ **18 валідаторів** для всіх Commands, Queries та DTOs
- ✅ **9 кастомних розширень** для специфічних бізнес-правил
- ✅ **100% покриття** - кожен Command/Query/DTO має валідатор
- ✅ **Автоматична інтеграція** з MediatR pipeline
- ✅ **Консистентна система** валідації в усьому проєкті

**Головні переваги:**
- ✅ Type-safe validation
- ✅ Reusable custom rules
- ✅ Clear error messages (українською)
- ✅ Conditional validation
- ✅ Collection validation
- ✅ Automatic MediatR integration
