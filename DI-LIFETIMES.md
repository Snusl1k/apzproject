# Dependency Injection (DI) - Життєві цикли сервісів

## ✅ Реалізовані життєві цикли DI в проєкті

ASP.NET Core має вбудований DI контейнер з трьома типами життєвих циклів:

---

### 1. **TRANSIENT** ⚡
**Новий екземпляр при кожному запиті з DI контейнера**

#### Коли використовувати:
- Легкі stateless сервіси
- Сервіси без спільного стану
- Об'єкти з коротким життям

#### Реалізація:
```csharp
services.AddTransient<ITransientOperationService, TransientOperationService>();
```

#### Приклад:
- [TransientOperationService.cs](CafeOrders.Application/Services/Lifetime/TransientOperationService.cs)

---

### 2. **SCOPED** 🔄
**Один екземпляр на HTTP запит**

#### Коли використовувати:
- Робота з базою даних (DbContext)
- Repository pattern
- Unit of Work
- Сервіси, що потребують спільного стану в межах запиту

#### Реалізація:
```csharp
services.AddScoped<IScopedOperationService, ScopedOperationService>();
services.AddScoped<IOrderRepository, OrderRepository>();
services.AddScoped<IOrderService, OrderService>();
```

#### Приклади в проєкті:
- [ScopedOperationService.cs](CafeOrders.Application/Services/Lifetime/ScopedOperationService.cs)
- [OrderService.cs](CafeOrders.Application/Services/OrderService.cs) - бізнес-логіка
- [OrderRepository.cs](CafeOrders.Infrastructure/Persistence/Repositories/OrderRepository.cs) - доступ до БД
- `CafeOrdersDbContext` - EF Core контекст (завжди Scoped!)

---

### 3. **SINGLETON** 🌍
**Один екземпляр на весь час життя застосунку**

#### Коли використовувати:
- Thread-safe сервіси
- Кеші та конфігурації
- Stateless допоміжні сервіси
- Сервіси, що не зберігають стан конкретного запиту

#### ⚠️ ВАЖЛИВО:
- Має бути thread-safe
- НЕ може залежати від Scoped сервісів (DbContext)
- Створюється при старті застосунку

#### Реалізація:
```csharp
services.AddSingleton<ISingletonOperationService, SingletonOperationService>();
services.AddSingleton<IConfigurationCacheService, ConfigurationCacheService>();
```

#### Приклади:
- [SingletonOperationService.cs](CafeOrders.Application/Services/Lifetime/SingletonOperationService.cs)
- [ConfigurationCacheService.cs](CafeOrders.Infrastructure/Services/ConfigurationCacheService.cs) - thread-safe кеш

---

## 📊 Порівняння життєвих циклів

| Життєвий цикл | Створення | Час життя | Приклади використання |
|---------------|-----------|-----------|----------------------|
| **Transient** | При кожному запиті з DI | До завершення використання | Легкі stateless операції |
| **Scoped** | Один раз на HTTP запит | До завершення запиту | DbContext, Repositories, бізнес-логіка |
| **Singleton** | При старті застосунку | Весь час роботи | Кеші, конфігурації, логери |

---

## 🧪 Тестування життєвих циклів

### Демонстраційний endpoint:
```
GET /api/di-lifetime
```

Викличте цей endpoint кілька разів та порівняйте GUID:

- **Transient**: Завжди новий GUID при кожному запиті
- **Scoped**: Однаковий GUID в межах одного запиту, різний між запитами
- **Singleton**: Завжди той самий GUID (створюється один раз при старті)

### Швидка перевірка:
```
GET /api/di-lifetime/quick
```

---

## 📝 Реєстрація в проєкті

### Application Layer ([DependencyInjection.cs](CafeOrders.Application/DependencyInjection/DependencyInjection.cs)):
```csharp
// TRANSIENT
services.AddTransient<ITransientOperationService, TransientOperationService>();

// SCOPED (типово для бізнес-логіки)
services.AddScoped<IMenuService, MenuService>();
services.AddScoped<IOrderService, OrderService>();
services.AddScoped<ILifetimeDemoService, LifetimeDemoService>();

// SINGLETON
services.AddSingleton<ISingletonOperationService, SingletonOperationService>();
```

### Infrastructure Layer ([DependencyInjection.cs](CafeOrders.Infrastructure/DependencyInjection/DependencyInjection.cs)):
```csharp
// SCOPED (DbContext та repositories)
services.AddDbContext<CafeOrdersDbContext>(); // Завжди Scoped!
services.AddScoped<IOrderRepository, OrderRepository>();

// SINGLETON (кеш конфігурацій)
services.AddSingleton<IConfigurationCacheService, ConfigurationCacheService>();
```

---

## ⚠️ Типові помилки та як їх уникнути

### 1. Captive Dependency
❌ **НЕ РОБІТЬ ТАК:**
```csharp
// Singleton залежить від Scoped - ПОМИЛКА!
public class MySingletonService
{
    private readonly DbContext _context; // DbContext - Scoped!
}
```

✅ **ПРАВИЛЬНО:**
```csharp
// Singleton використовує IServiceProvider для створення scope
public class MySingletonService
{
    private readonly IServiceProvider _serviceProvider;
    
    public void DoWork()
    {
        using var scope = _serviceProvider.CreateScope();
        var context = scope.ServiceProvider.GetRequiredService<DbContext>();
    }
}
```

### 2. Thread Safety
Singleton сервіси мають бути thread-safe (використовуйте `ConcurrentDictionary`, `lock`, тощо).

### 3. Memory Leaks
Transient сервіси з `IDisposable` автоматично звільняються контейнером наприкінці scope.

---

## 🎯 Best Practices

1. **Типово використовуйте Scoped** для бізнес-логіки
2. **Singleton тільки для stateless** сервісів
3. **Transient для легких** операцій
4. **DbContext завжди Scoped** (вбудовано в EF Core)
5. **Repositories - Scoped** (працюють з DbContext)
6. **Уникайте Captive Dependencies** (Singleton → Scoped)
