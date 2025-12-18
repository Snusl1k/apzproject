# 🚀 Швидкий старт - Тестування CafeOrders API

## 📋 Список всіх HTTP тестів

### 1. **Основний функціонал** - [CafeOrders.Api.http](CafeOrders.Api/CafeOrders.Api.http)
Базові операції з меню та замовленнями.

### 2. **Кешування** - [test-caching.http](test-caching.http) ⭐ NEW
Демонстрація системи кешування з порівнянням продуктивності.

**Рекомендований порядок тестування**:
```
1. GET /api/cache-demo/performance-comparison  👈 Почніть з цього!
   - Побачите різницю: ~1500ms vs ~1ms (1500x швидше)

2. GET /api/cache-demo/get-or-create
   - Перший виклик: повільно (~2 секунди)
   - Другий виклик: миттєво (<1ms) з кешу

3. GET /api/cache-demo/statistics
   - Перегляд існуючих ключів кешу

4. DELETE /api/cache-demo/invalidate/demo:expensive-data
   - Видалення конкретного ключа

5. GET /api/cache-demo/statistics
   - Перевірка що ключ видалено
```

### 3. **FluentValidation** - [test-validation.http](test-validation.http)
100% покриття валідацією всіх команд, запитів та DTO.

**Цікаві тести**:
- `POST /api/validation-demo/test-multiple-errors` - множинні помилки валідації
- `GET /api/validation-demo/test-phone?phone=+380501234567` - валідація українського номера

### 4. **AutoMapper** - [test-automapper.http](test-automapper.http)
Автоматичний маппінг Domain → DTO з вкладеними об'єктами.

**Найкращі приклади**:
- `GET /api/automapper-demo/orders/{id}` - маппінг з OrderLines та обчисленням Total
- `GET /api/automapper-demo/mapping-comparison` - порівняння Map() vs ProjectTo()

### 5. **Unit of Work** - [test-unitofwork.http](test-unitofwork.http)
Координація кількох репозиторіїв та управління транзакціями.

**Ключові тести**:
- `POST /api/unit-of-work/create-full-workflow` - створення Category → MenuItem → Order в одній транзакції
- `GET /api/unit-of-work/test-rollback` - демонстрація rollback

### 6. **Generic Repository** - [test-repository.http](test-repository.http)
Generic Repository Pattern з CRUD операціями.

### 7. **Event-Driven Design** - [test-events.http](test-events.http)
Обробка доменних подій (OrderCreatedEvent → Logging + Notification).

## 🎯 Швидкий тест кешування (5 хвилин)

### Крок 1: Запустіть застосунок
```bash
dotnet run --project CafeOrders.Api --urls "http://localhost:5000"
```

### Крок 2: Відкрийте test-caching.http

### Крок 3: Виконайте Performance Comparison
```http
GET http://localhost:5000/api/cache-demo/performance-comparison
```

**Очікуваний результат**:
```json
{
  "message": "Performance порівняння: Cache vs No Cache",
  "results": [
    { "request": "Without Cache", "time": "1500 ms", "source": "Database" },
    { "request": "With Cache", "time": "1 ms", "source": "Cache" },
    { "request": "With Cache (2nd time)", "time": "0 ms", "source": "Cache" }
  ],
  "speedup": "1500x швидше з кешем",
  "recommendation": "✅ Кешування рекомендується для цієї операції"
}
```

### Крок 4: Тест GetOrCreate
```http
GET http://localhost:5000/api/cache-demo/get-or-create
```

**Виконайте ДВІЧІ**:
- 1-й виклик: ~2000ms (створення даних)
- 2-й виклик: <1ms (з кешу)

### Крок 5: Інвалідація кешу
```http
DELETE http://localhost:5000/api/cache-demo/invalidate-prefix/demo:
```

Видалить всі ключі з префіксом `demo:`.

## 📊 Корисні endpoint'и

### Перевірка здоров'я
```http
GET http://localhost:5000/health
```

### Swagger UI
```
http://localhost:5000/swagger
```

### Статистика кешу
```http
GET http://localhost:5000/api/cache-demo/statistics
```

### DI Lifetimes Demo
```http
GET http://localhost:5000/api/di-lifetime
```

## 💡 Поради

### 1. Використовуйте VS Code REST Client
Встановіть розширення **REST Client** для зручного виконання HTTP запитів прямо з `.http` файлів.

### 2. Порівняйте performance
Виконайте `performance-comparison` кілька разів - побачите стабільні результати кешування.

### 3. Моніторинг логів
Під час роботі з кешем дивіться на лог застосунку:
```
Cache HIT for key: demo:expensive-data    ← Дані з кешу
Cache MISS for key: demo:expensive-data   ← Дані з БД
Cache SET for key: demo:expensive-data    ← Збереження в кеш
```

### 4. Інвалідація
Після `POST`/`PUT`/`DELETE` операцій тестуйте інвалідацію:
```http
POST /api/orders                          ← Створити замовлення
DELETE /api/cache-demo/invalidate-prefix/orders:  ← Інвалідувати кеш
GET /api/orders                           ← Дані оновляться
```

## 📖 Детальна документація

- **Кешування**: [CACHING.md](CACHING.md)
- **Валідація**: [FLUENTVALIDATION.md](FLUENTVALIDATION.md)
- **AutoMapper**: [AUTOMAPPER-PROFILES.md](AUTOMAPPER-PROFILES.md)
- **SOLID**: [SOLID-REFACTORING.md](SOLID-REFACTORING.md)
- **DI Lifetimes**: [DI-LIFETIMES.md](DI-LIFETIMES.md)

## 🎓 Навчальні цілі

Цей проєкт демонструє:

✅ **Clean Architecture** - розділення на 4 шари  
✅ **DDD** - агрегати з поведінкою та інваріантами  
✅ **CQRS** - розділення команд та запитів (MediatR)  
✅ **Event-Driven Design** - доменні події з обробниками  
✅ **Repository Pattern** - Generic + Specific repositories  
✅ **Unit of Work** - координація репозиторіїв + транзакції  
✅ **Dependency Injection** - всі 3 lifetimes (Transient/Scoped/Singleton)  
✅ **AutoMapper** - Domain → DTO з вкладеними об'єктами  
✅ **FluentValidation** - 100% покриття з кастомними розширеннями  
✅ **Caching** - In-Memory + Response Cache для продуктивності  
✅ **SOLID принципи** - 5 виправлених порушень  

---

**Автор**: CafeOrders Development Team  
**Версія**: 1.0  
**Дата**: 2025-12-18

💡 **Порада**: Почніть з [test-caching.http](test-caching.http) щоб побачити реальну різницю в продуктивності!
