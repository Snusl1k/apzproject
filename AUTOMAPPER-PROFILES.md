# AutoMapper Profiles - Конфігурації маппінгу

Документація всіх конфігурацій AutoMapper для маппінгу Domain сутностей ↔ DTO.

## Огляд

AutoMapper автоматизує конвертацію між Domain моделями (Entities) та Data Transfer Objects (DTOs). Всі профілі розташовані в `CafeOrders.Application/Mapping/`.

### Реєстрація в DI

```csharp
services.AddAutoMapper(typeof(DependencyInjection).Assembly);
```

AutoMapper автоматично сканує assembly та реєструє всі класи, що наслідують `Profile`.

---

## 1. OrderMappingProfile

**Файл:** `CafeOrders.Application/Mapping/OrderMappingProfile.cs`

### Маппінги Domain → DTO (для Read операцій)

#### Order → OrderDto
```csharp
CreateMap<Order, OrderDto>()
    .ForMember(dest => dest.Status, opt => opt.MapFrom(src => src.Status.ToString()))
    .ForMember(dest => dest.Total, opt => opt.MapFrom(src => src.Total))
    .ForMember(dest => dest.Lines, opt => opt.MapFrom(src => src.Lines));
```

**Використання:**
- `GetOrdersQuery` - повертає список замовлень
- `CreateOrderCommand` - response після створення
- `AutoMapperDemoController` - демонстрація маппінгу

**Особливості:**
- `Status` конвертується з enum в string (`OrderStatus.Pending` → `"Pending"`)
- `Total` обчислюється автоматично через властивість Domain моделі
- `Lines` колекція автоматично маппиться через `OrderLine → OrderLineDto`

#### OrderLine → OrderLineDto
```csharp
CreateMap<OrderLine, OrderLineDto>()
    .ForMember(dest => dest.MenuItemName, opt => opt.MapFrom(src => src.MenuItemName))
    .ForMember(dest => dest.LineTotal, opt => opt.MapFrom(src => src.LineTotal));
```

**Використання:**
- Автоматично викликається при маппінгу `Order → OrderDto` для колекції `Lines`

**Особливості:**
- `LineTotal` обчислюється в Domain моделі як `UnitPrice * Quantity`

### Маппінги DTO → Domain (для Create/Update операцій)

#### OrderDto → Order (зворотний маппінг)
```csharp
CreateMap<OrderDto, Order>()
    .ForMember(dest => dest.Status, opt => opt.MapFrom(src => Enum.Parse<OrderStatus>(src.Status)))
    .ForMember(dest => dest.Lines, opt => opt.Ignore())
    .ForMember(dest => dest.DomainEvents, opt => opt.Ignore())
    .ForMember(dest => dest.Total, opt => opt.Ignore());
```

**Обмеження:**
- ❌ **Не використовується в production** через DDD pattern
- Order має private constructor та фабричний метод `Order.Create()`
- `Lines` не маппляться напряму, додаються через `Order.AddLine()`
- `DomainEvents` управляється всередині агрегату

**Рекомендація:**
Для створення Order використовуйте фабричний метод `Order.Create()` у CommandHandler.

---

## 2. MenuItemMappingProfile

**Файл:** `CafeOrders.Application/Mapping/MenuItemMappingProfile.cs`

### Маппінги Domain → DTO (для Read операцій)

#### MenuItem → MenuItemDto
```csharp
CreateMap<MenuItem, MenuItemDto>()
    .ForMember(dest => dest.CategoryName, 
        opt => opt.MapFrom(src => src.Category != null ? src.Category.Name : string.Empty));
```

**Використання:**
- `GetMenuItemsQuery` - повертає список позицій меню
- `CreateMenuItemCommand` - response після створення
- `AutoMapperDemoController` - демонстрація

**Особливості:**
- `CategoryName` витягується з navigation property `Category.Name`
- Безпечна обробка null (якщо Category не завантажено через EF Include)

**Приклад використання в Query Handler:**
```csharp
var menuItems = await _context.MenuItems
    .Include(m => m.Category) // Важливо для CategoryName
    .ToListAsync(cancellationToken);

var dtos = _mapper.Map<IReadOnlyCollection<MenuItemDto>>(menuItems);
```

### Маппінги DTO → Domain (для Update операцій)

#### MenuItemDto → MenuItem
```csharp
CreateMap<MenuItemDto, MenuItem>()
    .ForMember(dest => dest.Category, opt => opt.Ignore())
    .ForMember(dest => dest.CategoryId, opt => opt.MapFrom(src => src.CategoryId))
    .ForMember(dest => dest.Id, opt => opt.Ignore());
```

**Використання:**
- Update операції (якщо буде додано `UpdateMenuItemCommand`)

**Обмеження:**
- `Category` не маппиться (navigation property)
- `Id` не змінюється при Update

### Command → Domain (для Create операцій)

#### CreateMenuItemCommand → MenuItem
```csharp
// Примітка: MenuItem має private constructor і фабричний метод Create()
// Використовуйте MenuItem.Create() у CommandHandler замість автомаппінгу
```

**Рекомендація:**
```csharp
// ✅ Правильно (в CreateMenuItemCommandHandler)
var entity = MenuItem.Create(
    request.Name, 
    request.Description, 
    request.Price, 
    request.CategoryId);

// ❌ Неправильно (AutoMapper не може через private constructor)
var entity = _mapper.Map<MenuItem>(request);
```

---

## 3. MenuCategoryMappingProfile

**Файл:** `CafeOrders.Application/Mapping/MenuCategoryMappingProfile.cs`

### Маппінги Domain → DTO (для Read операцій)

#### MenuCategory → MenuCategoryDto
```csharp
CreateMap<MenuCategory, MenuCategoryDto>();
```

**Використання:**
- `GetMenuCategoriesQuery` - повертає список категорій
- `AutoMapperDemoController` - демонстрація

**Особливості:**
- Простий маппінг без спеціальних конфігурацій
- Всі властивості маппляться автоматично (convention-based)

### Маппінги DTO → Domain (для Update операцій)

#### MenuCategoryDto → MenuCategory
```csharp
CreateMap<MenuCategoryDto, MenuCategory>()
    .ForMember(dest => dest.Items, opt => opt.Ignore())
    .ForMember(dest => dest.Id, opt => opt.Ignore());
```

**Використання:**
- Update операції (якщо буде додано `UpdateMenuCategoryCommand`)

**Обмеження:**
- `Items` не маппиться (навігаційна колекція MenuItem)
- `Id` не змінюється при Update

---

## Використання в коді

### 1. Injection в Query/Command Handlers

```csharp
public class GetOrdersQueryHandler : IRequestHandler<GetOrdersQuery, IReadOnlyCollection<OrderDto>>
{
    private readonly IApplicationDbContext _context;
    private readonly IMapper _mapper; // ← Inject IMapper

    public GetOrdersQueryHandler(IApplicationDbContext context, IMapper mapper)
    {
        _context = context;
        _mapper = mapper;
    }

    public async Task<IReadOnlyCollection<OrderDto>> Handle(
        GetOrdersQuery request, 
        CancellationToken cancellationToken)
    {
        var orders = await _context.Orders
            .Include(o => o.Lines)
            .ToListAsync(cancellationToken);

        // AutoMapper: List<Order> → List<OrderDto>
        return _mapper.Map<IReadOnlyCollection<OrderDto>>(orders);
    }
}
```

### 2. Використання в Controllers

```csharp
[ApiController]
[Route("api/orders")]
public class OrdersController : ControllerBase
{
    private readonly IMapper _mapper;

    public OrdersController(IMapper mapper)
    {
        _mapper = mapper;
    }

    [HttpGet("{id}")]
    public async Task<ActionResult<OrderDto>> GetOrder(Guid id)
    {
        var order = await _repository.GetByIdAsync(id);
        if (order == null) return NotFound();

        // AutoMapper: Order → OrderDto
        var dto = _mapper.Map<OrderDto>(order);
        return Ok(dto);
    }
}
```

### 3. Маппінг колекцій

```csharp
// Список
List<Order> orders = await _repository.GetAllAsync();
var orderDtos = _mapper.Map<List<OrderDto>>(orders);

// IReadOnlyCollection
IReadOnlyCollection<MenuItem> menuItems = await _repository.GetAllAsync();
var menuItemDtos = _mapper.Map<IReadOnlyCollection<MenuItemDto>>(menuItems);

// Array
MenuItem[] items = menuItems.ToArray();
var itemDtos = _mapper.Map<MenuItemDto[]>(items);
```

### 4. Маппінг одного об'єкту

```csharp
Order order = await _repository.GetByIdAsync(orderId);
OrderDto orderDto = _mapper.Map<OrderDto>(order);
```

---

## Best Practices

### ✅ Коли використовувати AutoMapper

1. **Domain → DTO (Read операції)**
   - Query handlers (GetOrdersQuery, GetMenuItemsQuery)
   - API responses
   - Демонстраційні endpoints

2. **DTO → Domain (Update операції)**
   - Оновлення існуючих entities
   - Patch операції

3. **Колекції**
   - Маппінг списків, масивів, IEnumerable

### ❌ Коли НЕ використовувати AutoMapper

1. **DDD Aggregates з private constructors**
   - Використовуйте фабричні методи: `Order.Create()`, `MenuItem.Create()`
   - AutoMapper не може обійти encapsulation

2. **Entity створення через Commands**
   - `CreateOrderCommand` → викликайте `Order.Create()` напряму
   - `CreateMenuItemCommand` → викликайте `MenuItem.Create()` напряму

3. **Складна бізнес-логіка**
   - Якщо створення entity потребує validation, event dispatching
   - Краще написати explicit код у handler

### 🔍 Важливі моменти

#### Navigation Properties
Завжди використовуйте `.Include()` для завантаження навігаційних властивостей:

```csharp
// ✅ Правильно
var menuItems = await _context.MenuItems
    .Include(m => m.Category) // CategoryName буде доступний
    .ToListAsync();

var dtos = _mapper.Map<List<MenuItemDto>>(menuItems);
// dtos[0].CategoryName = "Beverages"

// ❌ Неправильно (CategoryName буде порожнім)
var menuItems = await _context.MenuItems.ToListAsync();
var dtos = _mapper.Map<List<MenuItemDto>>(menuItems);
// dtos[0].CategoryName = "" ← Category == null
```

#### Enum Conversions
AutoMapper автоматично конвертує enum ↔ string:

```csharp
// Order.Status (OrderStatus enum) → OrderDto.Status (string)
OrderStatus.Pending → "Pending"
OrderStatus.InProgress → "InProgress"

// Зворотна конвертація
"Pending" → OrderStatus.Pending
```

#### Computed Properties
Використовуйте `.MapFrom()` для властивостей, що обчислюються:

```csharp
CreateMap<Order, OrderDto>()
    .ForMember(dest => dest.Total, 
        opt => opt.MapFrom(src => src.Total)); // src.Total обчислюється в Domain
```

---

## Тестування

### Unit тести для Profiles

```csharp
[Fact]
public void OrderMappingProfile_ShouldHaveValidConfiguration()
{
    var configuration = new MapperConfiguration(cfg => 
        cfg.AddProfile<OrderMappingProfile>());
    
    configuration.AssertConfigurationIsValid(); // Перевірка всіх маппінгів
}

[Fact]
public void Order_Should_Map_To_OrderDto()
{
    // Arrange
    var mapper = new MapperConfiguration(cfg => 
        cfg.AddProfile<OrderMappingProfile>()).CreateMapper();
    
    var order = Order.Create("Test Customer");
    order.AddLine(Guid.NewGuid(), "Coffee", 3.50m, 2);

    // Act
    var dto = mapper.Map<OrderDto>(order);

    // Assert
    Assert.Equal(order.Id, dto.Id);
    Assert.Equal("Test Customer", dto.CustomerName);
    Assert.Equal(7.00m, dto.Total);
    Assert.Single(dto.Lines);
}
```

### Integration тести з контролерами

Перевірте HTTP тести: [test-automapper.http](test-automapper.http)

---

## Підсумок

| Profile | Domain → DTO | DTO → Domain | Command → Domain |
|---------|--------------|--------------|------------------|
| **OrderMappingProfile** | ✅ Order → OrderDto<br>✅ OrderLine → OrderLineDto | ⚠️ Не рекомендується (private ctor) | ⚠️ Використовуйте Order.Create() |
| **MenuItemMappingProfile** | ✅ MenuItem → MenuItemDto | ✅ MenuItemDto → MenuItem | ⚠️ Використовуйте MenuItem.Create() |
| **MenuCategoryMappingProfile** | ✅ MenuCategory → MenuCategoryDto | ✅ MenuCategoryDto → MenuCategory | ⚠️ Використовуйте MenuCategory.Create() |

**Головний принцип:** AutoMapper відмінно працює для Domain → DTO (Read), але для Create операцій завжди використовуйте фабричні методи Domain моделей.
