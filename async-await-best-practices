# C# Async/Await Best Practices Guide

A comprehensive guide to mastering asynchronous programming in C# with practical examples, common pitfalls, and real-world patterns.

## 📚 Table of Contents

- [Overview](#overview)
- [Key Concepts](#key-concepts)
- [Best Practices](#best-practices)
- [Common Pitfalls](#common-pitfalls)
- [Real-World Patterns](#real-world-patterns)
- [Performance Tips](#performance-tips)
- [Debugging](#debugging)
- [Quick Reference](#quick-reference)
- [Resources](#resources)

## Overview

This repository contains examples and guidelines for writing effective asynchronous code in C#. Whether you're building web APIs, desktop applications, or libraries, understanding async/await is essential for creating responsive and scalable applications.

## Key Concepts

### What Happens Behind the Scenes?

When you use `async`/`await`:
1. Method executes synchronously until the first `await`
2. If the operation isn't complete, the method returns an incomplete `Task`
3. When the operation completes, the continuation is scheduled
4. The rest of the method executes on the captured context

```csharp
public async Task<string> FetchDataAsync()
{
    // Doesn't block the thread
    var data = await httpClient.GetStringAsync("https://api.example.com/data");
    
    // Resumes on captured context
    return data.ToUpper();
}
```

## Best Practices

### ✅ Use ConfigureAwait(false) in Libraries

Avoid capturing the synchronization context when you don't need it:

```csharp
public async Task<User> GetUserAsync(int id)
{
    var response = await httpClient.GetAsync($"/users/{id}")
        .ConfigureAwait(false);
    var json = await response.Content.ReadAsStringAsync()
        .ConfigureAwait(false);
    return JsonSerializer.Deserialize<User>(json);
}
```

**When to use:** Library code, background services  
**When to avoid:** UI code (WPF, WinForms), ASP.NET Framework

### ✅ Avoid Async Void

```csharp
// ❌ Bad - Can't await, exceptions crash the app
public async void ProcessDataAsync() { }

// ✅ Good - Returns Task, can be awaited
public async Task ProcessDataAsync() { }

// ✅ Exception - Event handlers must be async void
private async void Button_Click(object sender, EventArgs e)
{
    try
    {
        await ProcessDataAsync();
    }
    catch (Exception ex)
    {
        // Handle gracefully
    }
}
```

### ✅ Parallel Operations with Task.WhenAll

```csharp
// ❌ Sequential - 3 seconds for 3 requests
foreach (var id in ids)
{
    var user = await GetUserAsync(id);
    users.Add(user);
}

// ✅ Parallel - ~1 second for 3 requests
var tasks = ids.Select(id => GetUserAsync(id));
var users = await Task.WhenAll(tasks);
```

### ✅ Support Cancellation

```csharp
public async Task<List<Product>> SearchProductsAsync(
    string query, 
    CancellationToken cancellationToken = default)
{
    cancellationToken.ThrowIfCancellationRequested();
    
    var response = await httpClient.GetAsync(
        $"/products?q={query}", 
        cancellationToken);
    
    return await JsonSerializer.DeserializeAsync<List<Product>>(
        await response.Content.ReadAsStreamAsync(),
        cancellationToken: cancellationToken);
}
```

### ✅ Stream Data with IAsyncEnumerable

```csharp
public async IAsyncEnumerable<LogEntry> StreamLogsAsync(
    [EnumeratorCancellation] CancellationToken cancellationToken = default)
{
    var reader = await GetLogReaderAsync();
    
    while (await reader.ReadAsync(cancellationToken))
    {
        yield return reader.Current;
    }
}

// Usage
await foreach (var log in StreamLogsAsync())
{
    Console.WriteLine(log.Message);
}
```

### ✅ Use ValueTask for Hot Paths

```csharp
public async ValueTask<int> GetCachedValueAsync(string key)
{
    if (_cache.TryGetValue(key, out var value))
    {
        return value; // No heap allocation when cached
    }
    return await _database.GetValueAsync(key);
}
```

## Common Pitfalls

### 🚫 The Deadlock Trap

```csharp
// ❌ DEADLY - Will deadlock!
public string GetDataSync()
{
    return GetDataAsync().Result; // DON'T DO THIS
}

// ✅ Solutions:
// 1. Make it async all the way
public async Task<string> GetDataAsync()
{
    return await FetchDataAsync();
}

// 2. Use ConfigureAwait(false) in called methods
// 3. Avoid mixing sync and async code
```

### 🚫 Async Over Sync

```csharp
// ❌ Bad - Fake async
public async Task<int> CalculateAsync(int value)
{
    await Task.Delay(0); // Pointless!
    return value * 2;
}

// ✅ Good - Just make it synchronous
public int Calculate(int value)
{
    return value * 2;
}
```

### 🚫 Fire and Forget

```csharp
// ❌ Bad - Exceptions swallowed
public void ProcessData()
{
    _ = ProcessDataAsync(); // Dangerous!
}

// ✅ Good - Proper error handling
public void ProcessData()
{
    _ = ProcessDataSafeAsync();
}

private async Task ProcessDataSafeAsync()
{
    try
    {
        await ProcessDataAsync();
    }
    catch (Exception ex)
    {
        _logger.LogError(ex, "Processing failed");
    }
}
```

## Real-World Patterns

### Retry with Exponential Backoff

```csharp
public async Task<T> RetryAsync<T>(
    Func<Task<T>> operation,
    int maxRetries = 3,
    CancellationToken cancellationToken = default)
{
    for (int i = 0; i < maxRetries; i++)
    {
        try
        {
            return await operation();
        }
        catch (Exception ex) when (i < maxRetries - 1)
        {
            var delay = TimeSpan.FromSeconds(Math.Pow(2, i));
            await Task.Delay(delay, cancellationToken);
        }
    }
    return await operation();
}
```

### Timeout Pattern

```csharp
public async Task<T> WithTimeoutAsync<T>(Task<T> task, TimeSpan timeout)
{
    using var cts = new CancellationTokenSource(timeout);
    var completedTask = await Task.WhenAny(task, Task.Delay(timeout, cts.Token));
    
    if (completedTask != task)
    {
        throw new TimeoutException($"Operation timed out after {timeout}");
    }
    
    cts.Cancel();
    return await task;
}
```

### Rate Limiting (Bulkhead Pattern)

```csharp
public class RateLimitedService
{
    private readonly SemaphoreSlim _semaphore;
    
    public RateLimitedService(int maxConcurrent = 5)
    {
        _semaphore = new SemaphoreSlim(maxConcurrent, maxConcurrent);
    }
    
    public async Task<T> ExecuteAsync<T>(
        Func<Task<T>> operation,
        CancellationToken cancellationToken = default)
    {
        await _semaphore.WaitAsync(cancellationToken);
        try
        {
            return await operation();
        }
        finally
        {
            _semaphore.Release();
        }
    }
}
```

### Lazy Async Initialization

```csharp
public class ExpensiveResource
{
    private readonly Lazy<Task<Connection>> _connection;
    
    public ExpensiveResource()
    {
        _connection = new Lazy<Task<Connection>>(InitializeConnectionAsync);
    }
    
    private async Task<Connection> InitializeConnectionAsync()
    {
        await Task.Delay(2000); // Expensive initialization
        return new Connection();
    }
    
    public async Task<string> GetDataAsync()
    {
        var connection = await _connection.Value;
        return await connection.FetchDataAsync();
    }
}
```

## Performance Tips

### Avoid Unnecessary State Machines

```csharp
// ❌ Creates state machine unnecessarily
public async Task<User> GetUserAsync(int id)
{
    return await _repository.GetByIdAsync(id);
}

// ✅ Pass through - no state machine
public Task<User> GetUserAsync(int id)
{
    return _repository.GetByIdAsync(id);
}
```

### Use ArrayPool for Buffers

```csharp
public async Task<byte[]> ProcessDataAsync(Stream stream)
{
    var buffer = ArrayPool<byte>.Shared.Rent(4096);
    try
    {
        int bytesRead = await stream.ReadAsync(buffer, 0, buffer.Length);
        var result = new byte[bytesRead];
        Array.Copy(buffer, result, bytesRead);
        return result;
    }
    finally
    {
        ArrayPool<byte>.Shared.Return(buffer);
    }
}
```

## Debugging

### Enable Async Call Stack in Visual Studio

1. Go to **Debug → Windows → Call Stack**
2. Check **Show External Code**
3. View the full async call chain

### Async-Friendly Logging

```csharp
public async Task<User> GetUserAsync(int userId)
{
    using var scope = _logger.BeginScope(
        new Dictionary<string, object>
        {
            ["UserId"] = userId,
            ["CorrelationId"] = Guid.NewGuid()
        });
    
    _logger.LogInformation("Fetching user {UserId}", userId);
    
    try
    {
        var user = await _repository.GetByIdAsync(userId);
        _logger.LogInformation("User {UserId} fetched successfully", userId);
        return user;
    }
    catch (Exception ex)
    {
        _logger.LogError(ex, "Failed to fetch user {UserId}", userId);
        throw;
    }
}
```

## Quick Reference

### ✅ DO

- Use `async`/`await` all the way down
- Add `ConfigureAwait(false)` in library code
- Use `Task.WhenAll` for parallel operations
- Provide `CancellationToken` parameters
- Use `ValueTask` for high-performance scenarios
- Use `IAsyncEnumerable` for streaming data
- Handle exceptions properly

### ❌ DON'T

- Use `async void` (except event handlers)
- Use `.Result` or `.Wait()` in async code
- Create fake async methods with `Task.Delay(0)`
- Swallow exceptions in fire-and-forget scenarios
- Mix synchronous and asynchronous code

### Performance Metrics

| Scenario | Sequential | Parallel (Task.WhenAll) |
|----------|-----------|------------------------|
| 3 API calls (1s each) | 3 seconds | ~1 second |
| Task allocations (1M cached hits) | ~24 MB | ~0 MB (ValueTask) |

## Resources

- [Microsoft Async/Await Best Practices](https://docs.microsoft.com/en-us/archive/msdn-magazine/2013/march/async-await-best-practices-in-asynchronous-programming)
- [ConfigureAwait FAQ](https://devblogs.microsoft.com/dotnet/configureawait-faq/)
- [Task-based Asynchronous Pattern (TAP)](https://docs.microsoft.com/en-us/dotnet/standard/asynchronous-programming-patterns/task-based-asynchronous-pattern-tap)
- [Stephen Cleary's Blog on Async](https://blog.stephencleary.com/)

## Contributing

Contributions are welcome! Please feel free to submit a Pull Request.

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

## Author

**Atesh** - Building Better .NET Applications

---

⭐ Found this helpful? Star the repo and share with your team!

**Tags:** `csharp` `async` `await` `dotnet` `performance` `best-practices`
