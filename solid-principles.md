# SOLID Principles in C#: A Practical Guide

A comprehensive guide to understanding and applying SOLID principles in C# with real-world examples, design patterns, and best practices for writing clean, maintainable code.

## 📚 Table of Contents

- [Overview](#overview)
- [What is SOLID?](#what-is-solid)
- [The Five Principles](#the-five-principles)
  - [Single Responsibility Principle](#1️⃣-single-responsibility-principle-srp)
  - [Open/Closed Principle](#2️⃣-openclosed-principle-ocp)
  - [Liskov Substitution Principle](#3️⃣-liskov-substitution-principle-lsp)
  - [Interface Segregation Principle](#4️⃣-interface-segregation-principle-isp)
  - [Dependency Inversion Principle](#5️⃣-dependency-inversion-principle-dip)
- [Applying SOLID Together](#applying-solid-together)
- [Practical Tips](#practical-tips)
- [Resources](#resources)

## Overview

SOLID principles are the foundation of clean, maintainable object-oriented design. This repository provides practical C# examples that demonstrate how to apply these principles in real-world scenarios, helping you write better software that's easier to test, maintain, and extend.

## What is SOLID?

SOLID is an acronym representing five fundamental design principles:

- **S** - Single Responsibility Principle
- **O** - Open/Closed Principle  
- **L** - Liskov Substitution Principle
- **I** - Interface Segregation Principle
- **D** - Dependency Inversion Principle

These principles, introduced by Robert C. Martin (Uncle Bob), help developers create software that is:
- **Maintainable** - Easy to understand and modify
- **Testable** - Simple to write unit tests for
- **Flexible** - Can adapt to changing requirements
- **Scalable** - Grows without becoming unwieldy

## The Five Principles

### 1️⃣ Single Responsibility Principle (SRP)

> **"A class should have one, and only one, reason to change."**

Each class should focus on a single task or responsibility.

#### ❌ Violation Example

```csharp
// BAD: This class does too many things
public class UserService
{
    public void RegisterUser(string email, string password)
    {
        // Validation
        if (string.IsNullOrEmpty(email))
            throw new ArgumentException("Email required");
        
        // Password hashing
        var hashedPassword = BCrypt.HashPassword(password);
        
        // Database access
        using var connection = new SqlConnection("...");
        // ... save to database
        
        // Email sending
        var smtp = new SmtpClient("smtp.gmail.com");
        smtp.Send(new MailMessage(...));
        
        // Logging
        File.AppendAllText("log.txt", $"User {email} registered");
    }
}
```

**Problems:**
- Mixes validation, data access, email, and logging
- Hard to test individual pieces
- Changes to any concern require modifying this class
- Tight coupling to implementations

#### ✅ Following SRP

```csharp
// GOOD: Each class has a single responsibility

public class UserValidator
{
    public ValidationResult Validate(string email, string password)
    {
        var errors = new List<string>();
        
        if (string.IsNullOrEmpty(email))
            errors.Add("Email is required");
        
        if (password.Length < 8)
            errors.Add("Password must be at least 8 characters");
        
        return new ValidationResult
        {
            IsValid = errors.Count == 0,
            Errors = errors
        };
    }
}

public interface IPasswordHasher
{
    string Hash(string password);
    bool Verify(string password, string hash);
}

public interface IUserRepository
{
    Task<User> CreateAsync(User user);
    Task<User?> GetByEmailAsync(string email);
}

public interface IEmailService
{
    Task SendWelcomeEmailAsync(string email);
}

public class UserRegistrationService
{
    private readonly UserValidator _validator;
    private readonly IPasswordHasher _passwordHasher;
    private readonly IUserRepository _userRepository;
    private readonly IEmailService _emailService;
    
    public async Task<Result<User>> RegisterUserAsync(string email, string password)
    {
        // Validate
        var validationResult = _validator.Validate(email, password);
        if (!validationResult.IsValid)
            return Result<User>.Failure(validationResult.Errors);
        
        // Create user
        var user = new User
        {
            Email = email,
            PasswordHash = _passwordHasher.Hash(password)
        };
        
        await _userRepository.CreateAsync(user);
        await _emailService.SendWelcomeEmailAsync(email);
        
        return Result<User>.Success(user);
    }
}
```

**Benefits:**
- Each class has one reason to change
- Easy to test components independently
- Can swap implementations easily
- Better code organization

---

### 2️⃣ Open/Closed Principle (OCP)

> **"Software entities should be open for extension, but closed for modification."**

You should be able to add new functionality without changing existing code.

#### ❌ Violation Example

```csharp
// BAD: Must modify to add new discount types
public class DiscountCalculator
{
    public decimal Calculate(Order order, string discountType)
    {
        if (discountType == "Seasonal")
            return order.Total * 0.10m;
        else if (discountType == "Premium")
            return order.Total * 0.20m;
        else if (discountType == "FirstOrder")
            return 50;
        // Adding new discount? Must modify this method!
        
        return 0;
    }
}
```

#### ✅ Following OCP

```csharp
// GOOD: Extend by adding new classes
public interface IDiscountStrategy
{
    decimal Calculate(Order order);
}

public class SeasonalDiscount : IDiscountStrategy
{
    public decimal Calculate(Order order) => order.Total * 0.10m;
}

public class PremiumCustomerDiscount : IDiscountStrategy
{
    public decimal Calculate(Order order) => order.Total * 0.20m;
}

public class FirstOrderDiscount : IDiscountStrategy
{
    public decimal Calculate(Order order) => 50;
}

public class DiscountCalculator
{
    private readonly IEnumerable<IDiscountStrategy> _strategies;
    
    public DiscountCalculator(IEnumerable<IDiscountStrategy> strategies)
    {
        _strategies = strategies;
    }
    
    public decimal Calculate(Order order)
    {
        return _strategies.Sum(s => s.Calculate(order));
    }
}

// Add new discount without modifying existing code
public class BulkOrderDiscount : IDiscountStrategy
{
    public decimal Calculate(Order order)
    {
        return order.Items.Count >= 10 ? order.Total * 0.15m : 0;
    }
}
```

**Benefits:**
- Add new features without touching existing code
- Reduces risk of introducing bugs
- Follows the Single Responsibility Principle
- Makes the system more maintainable

---

### 3️⃣ Liskov Substitution Principle (LSP)

> **"Objects of a superclass should be replaceable with objects of its subclasses without breaking the application."**

Derived classes must be substitutable for their base classes.

#### ❌ Violation Example

```csharp
// BAD: Square violates LSP
public class Rectangle
{
    public virtual int Width { get; set; }
    public virtual int Height { get; set; }
    
    public int CalculateArea() => Width * Height;
}

public class Square : Rectangle
{
    public override int Width
    {
        get => base.Width;
        set
        {
            base.Width = value;
            base.Height = value; // Violates expectations!
        }
    }
    
    public override int Height
    {
        get => base.Height;
        set
        {
            base.Height = value;
            base.Width = value; // Violates expectations!
        }
    }
}

// This breaks LSP
void ResizeRectangle(Rectangle rect)
{
    rect.Width = 5;
    rect.Height = 10;
    
    // Expected area: 50
    // Actual for Square: 100 (broken!)
    Console.WriteLine(rect.CalculateArea());
}
```

#### ✅ Following LSP

```csharp
// GOOD: Use composition instead
public interface IShape
{
    int CalculateArea();
}

public class Rectangle : IShape
{
    public int Width { get; set; }
    public int Height { get; set; }
    
    public int CalculateArea() => Width * Height;
}

public class Square : IShape
{
    public int Side { get; set; }
    
    public int CalculateArea() => Side * Side;
}

// Or use a common abstraction
public abstract class Shape
{
    public abstract int CalculateArea();
}

public class RectangleShape : Shape
{
    public int Width { get; set; }
    public int Height { get; set; }
    
    public override int CalculateArea() => Width * Height;
}

public class SquareShape : Shape
{
    public int Side { get; set; }
    
    public override int CalculateArea() => Side * Side;
}
```

**Key Points:**
- Subtypes must honor the contract of the base type
- Don't strengthen preconditions or weaken postconditions
- Behavior should be consistent and expected
- Prefer composition over inheritance when types aren't truly substitutable

---

### 4️⃣ Interface Segregation Principle (ISP)

> **"A client should not be forced to depend on interfaces it does not use."**

Create small, focused interfaces rather than large, general-purpose ones.

#### ❌ Violation Example

```csharp
// BAD: Fat interface with too many responsibilities
public interface IWorker
{
    void Work();
    void Eat();
    void Sleep();
    void GetPaid();
    void TakeVacation();
}

// Robot doesn't eat or sleep!
public class Robot : IWorker
{
    public void Work() { /* ... */ }
    public void GetPaid() { /* ... */ }
    
    public void Eat() => throw new NotImplementedException(); // Bad!
    public void Sleep() => throw new NotImplementedException(); // Bad!
    public void TakeVacation() => throw new NotImplementedException(); // Bad!
}
```

#### ✅ Following ISP

```csharp
// GOOD: Small, focused interfaces
public interface IWorkable
{
    void Work();
}

public interface IFeedable
{
    void Eat();
}

public interface ISleepable
{
    void Sleep();
}

public interface IPayable
{
    void GetPaid();
}

// Human implements all interfaces
public class Human : IWorkable, IFeedable, ISleepable, IPayable
{
    public void Work() { /* ... */ }
    public void Eat() { /* ... */ }
    public void Sleep() { /* ... */ }
    public void GetPaid() { /* ... */ }
}

// Robot only implements what it needs
public class Robot : IWorkable, IPayable
{
    public void Work() { /* ... */ }
    public void GetPaid() { /* ... */ }
}

// Manager can work with specific capabilities
public class WorkManager
{
    public void ManageWork(IWorkable worker)
    {
        worker.Work();
    }
    
    public void ProcessPayroll(IPayable employee)
    {
        employee.GetPaid();
    }
}
```

**Benefits:**
- Clients depend only on methods they use
- Reduces coupling between components
- Easier to implement and test
- More flexible and maintainable

---

### 5️⃣ Dependency Inversion Principle (DIP)

> **"High-level modules should not depend on low-level modules. Both should depend on abstractions."**

Depend on interfaces/abstractions, not concrete implementations.

#### ❌ Violation Example

```csharp
// BAD: High-level class depends on low-level implementations
public class ProductService
{
    private readonly SqlServerDatabase _database;
    private readonly FileLogger _logger;
    
    public ProductService()
    {
        _database = new SqlServerDatabase(); // Tight coupling!
        _logger = new FileLogger(); // Tight coupling!
    }
    
    public async Task<Product> GetProductAsync(int id)
    {
        _logger.Log($"Fetching product {id}");
        return await _database.GetByIdAsync<Product>(id);
    }
}
```

**Problems:**
- Can't swap database or logger implementations
- Hard to test (can't mock dependencies)
- Tight coupling to specific implementations
- Violates OCP

#### ✅ Following DIP

```csharp
// GOOD: Depend on abstractions
public interface IDataAccess
{
    Task<T> GetByIdAsync<T>(int id) where T : class;
    Task SaveAsync<T>(T entity) where T : class;
}

public interface ILogger
{
    void Log(string message);
}

// High-level module
public class ProductService
{
    private readonly IDataAccess _dataAccess;
    private readonly ILogger _logger;
    
    public ProductService(IDataAccess dataAccess, ILogger logger)
    {
        _dataAccess = dataAccess;
        _logger = logger;
    }
    
    public async Task<Product> GetProductAsync(int id)
    {
        _logger.Log($"Fetching product {id}");
        return await _dataAccess.GetByIdAsync<Product>(id);
    }
}

// Low-level implementations
public class SqlServerDataAccess : IDataAccess
{
    public async Task<T> GetByIdAsync<T>(int id) where T : class
    {
        // SQL Server implementation
        return default;
    }
    
    public async Task SaveAsync<T>(T entity) where T : class
    {
        // SQL Server save
    }
}

public class FileLogger : ILogger
{
    public void Log(string message)
    {
        File.AppendAllText("log.txt", $"{DateTime.Now}: {message}\n");
    }
}

public class DatabaseLogger : ILogger
{
    private readonly DbContext _context;
    
    public void Log(string message)
    {
        _context.Logs.Add(new LogEntry { Message = message });
        _context.SaveChanges();
    }
}

// DI Configuration
services.AddScoped<ILogger, DatabaseLogger>();
services.AddScoped<IDataAccess, SqlServerDataAccess>();
services.AddScoped<ProductService>();
```

**Benefits:**
- Easy to swap implementations
- Testable (can inject mocks)
- Loose coupling
- Follows Open/Closed Principle

---

## Applying SOLID Together

Here's how all five principles work together in a complete example:

```csharp
// S - Single Responsibility: Each interface has one purpose
public interface IOrderValidator
{
    ValidationResult Validate(Order order);
}

public interface IInventoryService
{
    Task<bool> ReserveItemsAsync(Order order);
}

public interface IPaymentGateway
{
    Task<PaymentResult> ProcessPaymentAsync(decimal amount);
}

public interface INotificationService
{
    Task SendOrderConfirmationAsync(Order order);
}

// O - Open/Closed: Extend by adding new processors
public abstract class OrderProcessor
{
    public abstract Task<bool> CanProcess(Order order);
    public abstract Task ProcessAsync(Order order);
}

public class StandardOrderProcessor : OrderProcessor
{
    public override Task<bool> CanProcess(Order order) =>
        Task.FromResult(order.Total < 1000);
    
    public override async Task ProcessAsync(Order order)
    {
        // Standard processing
        await Task.CompletedTask;
    }
}

public class ExpressOrderProcessor : OrderProcessor
{
    public override Task<bool> CanProcess(Order order) =>
        Task.FromResult(order.IsExpress);
    
    public override async Task ProcessAsync(Order order)
    {
        // Express processing with priority
        await Task.CompletedTask;
    }
}

// L - Liskov: All OrderProcessors are substitutable

// I - Interface Segregation: Small, focused interfaces

// D - Dependency Inversion: Depend on abstractions
public class OrderService
{
    private readonly IOrderValidator _validator;
    private readonly IInventoryService _inventory;
    private readonly IPaymentGateway _paymentGateway;
    private readonly INotificationService _notifications;
    private readonly IEnumerable<OrderProcessor> _processors;
    
    public OrderService(
        IOrderValidator validator,
        IInventoryService inventory,
        IPaymentGateway paymentGateway,
        INotificationService notifications,
        IEnumerable<OrderProcessor> processors)
    {
        _validator = validator;
        _inventory = inventory;
        _paymentGateway = paymentGateway;
        _notifications = notifications;
        _processors = processors;
    }
    
    public async Task<Result<Order>> PlaceOrderAsync(Order order)
    {
        // Validate
        var validationResult = _validator.Validate(order);
        if (!validationResult.IsValid)
            return Result<Order>.Failure(validationResult.Errors);
        
        // Reserve inventory
        if (!await _inventory.ReserveItemsAsync(order))
            return Result<Order>.Failure("Items not available");
        
        // Process payment
        var paymentResult = await _paymentGateway.ProcessPaymentAsync(order.Total);
        if (!paymentResult.Success)
            return Result<Order>.Failure("Payment failed");
        
        // Find and use appropriate processor
        var processor = await FindProcessorAsync(order);
        await processor.ProcessAsync(order);
        
        // Send confirmation
        await _notifications.SendOrderConfirmationAsync(order);
        
        return Result<Order>.Success(order);
    }
    
    private async Task<OrderProcessor> FindProcessorAsync(Order order)
    {
        foreach (var processor in _processors)
        {
            if (await processor.CanProcess(order))
                return processor;
        }
        
        return _processors.First(); // Default
    }
}
```

## Summary

| Principle | Key Takeaway | Main Benefit |
|-----------|-------------|--------------|
| **SRP** | One class, one responsibility | Easier to maintain and test |
| **OCP** | Extend without modifying | Add features safely |
| **LSP** | Subtypes are substitutable | Reliable polymorphism |
| **ISP** | Small, focused interfaces | Clients use only what they need |
| **DIP** | Depend on abstractions | Loose coupling, flexible design |

## Practical Tips

### Getting Started

1. **Start with SRP** - It's the foundation of good design
2. **Use interfaces liberally** - They enable DIP and ISP
3. **Favor composition over inheritance** - Helps with OCP and LSP
4. **Write tests** - SOLID code is inherently testable
5. **Refactor iteratively** - Don't over-engineer from the start

### Code Review Checklist

- [ ] Does each class have a single, clear responsibility?
- [ ] Can I add new features without modifying existing code?
- [ ] Are derived classes truly substitutable for their base classes?
- [ ] Do interfaces have focused, cohesive sets of methods?
- [ ] Do high-level modules depend on abstractions, not implementations?

### When to Apply SOLID

**Apply SOLID when:**
- Building production applications
- Code will be maintained long-term
- Multiple developers work on the codebase
- Requirements are likely to change
- Testing is important

**Consider simpler approaches for:**
- Throwaway prototypes
- Very small scripts
- Single-use utilities
- When over-engineering would add unnecessary complexity

## Resources

### Books
- [Clean Architecture](https://www.amazon.com/Clean-Architecture-Craftsmans-Software-Structure/dp/0134494164) by Robert C. Martin
- [Clean Code](https://www.amazon.com/Clean-Code-Handbook-Software-Craftsmanship/dp/0132350882) by Robert C. Martin
- [Design Patterns](https://www.amazon.com/Design-Patterns-Elements-Reusable-Object-Oriented/dp/0201633612) by Gang of Four

### Online Resources
- [SOLID Principles - Microsoft Docs](https://docs.microsoft.com/en-us/dotnet/architecture/modern-web-apps-azure/architectural-principles)
- [Refactoring Guru - Design Principles](https://refactoring.guru/design-patterns/principles)
- [Uncle Bob's Blog](https://blog.cleancoder.com/)

### Video Tutorials
- [SOLID Principles Explained](https://www.youtube.com/watch?v=HLFbeC78YlU) - Derek Banas
- [Clean Code](https://www.youtube.com/watch?v=7EmboKQH8lM) - Uncle Bob

## Contributing

Contributions are welcome! Please feel free to submit a Pull Request with:
- Additional examples
- Better explanations
- Real-world case studies
- Corrections or improvements

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

## Author

**Atesh** - Building Better Software

---

⭐ Found this helpful? Star the repo and share with developers who want to write cleaner code!

**Tags:** `csharp` `solid` `design-patterns` `clean-code` `software-architecture` `best-practices` `dotnet`

---

> "SOLID is a journey, not a destination. Start small, refactor continuously, and your codebase will thank you!"
