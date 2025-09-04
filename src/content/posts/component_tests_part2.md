---
title: 'Component Tests - Technical Foundation'
published: 2025-09-09
draft: true
description: 'The technical foundation that makes component tests possible - .NET 10, TestContainers, and the essential patterns.'
series: 'Component tests'
tags: ['dotnet', 'testing', 'integration-tests', 'component-tests', 'testcontainers']
---

In the [previous post](/posts/20250902_component_tests_part1), we saw how component tests can transform your testing strategy from implementation-focused unit tests to behavior-focused integration tests. Now let's look at the technical foundation that makes this possible.

## The Testing Stack

The component test "framework" is built on a few key pillars:

**.NET 10**: The latest version gives us the performance and features we need for complex integration scenarios. We're using the native testing platform that comes with modern .NET rather than external frameworks.

**TestContainers**: No more "works on my machine" database issues or non-migrated local databases. Every test runs against a real PostgreSQL (or SQL Server) instance in Docker. Fresh database, consistent state, full isolation.

**In-Memory messaging**: We use in-memory message brokers to simulate asynchronous communication between services. This allows us to test message-driven architectures without the overhead of real message brokers.

## Test Architecture Overview

```csharp
public class CreateEventTests(ServiceFixture serviceFixture, ITestOutputHelper testOutputHelper)
{
    private readonly ScenarioContext _context = new ScenarioContext(serviceFixture, testOutputHelper);

    [Fact]
    public Task WhenCreateEventValid_Then_EventIsCreated()
    {
        return new Scenario(_context)
            .Act(new CreateEventStep())
            .Assert(
                new RequestSucceeded(),
                new EventIsCreated()
            );
    }

    [Fact]
    public Task WhenEventWithSameNameExists_Then_ErrorIsReturned()
    {
        return new Scenario(_context)
            .Arrange(new SeedEvent())
            .Act(new CreateEventStep())
            .Assert(new RequestFailed().WithError("EVENT-NAME-ALREADY-EXISTS"));
    }
}
```

## The ServiceFixture

Instead of spinning up the entire application repeatedly, we use a shared fixture that manages your application lifecycle:

```csharp
public class ServiceFixture : IAsyncLifetime
{
    private PostgreSqlContainer _database;
    private WebApplicationFactory<Program> _factory;
    
    public IServiceProvider Services => _factory.Services;

    public async Task InitializeAsync()
    {
        // Spin up PostgreSQL in Docker
        _database = new PostgreSqlBuilder()
            .WithImage("postgres:15")
            .WithDatabase("testdb")
            .WithUsername("test")
            .WithPassword("test")
            .Build();

        await _database.StartAsync();

        // Build your application with test database
        _factory = new WebApplicationFactory<Program>()
            .WithWebHostBuilder(builder =>
            {
                builder.UseConnectionString(_database.GetConnectionString());
                // Configure test-specific overrides
            });

        // Run migrations, seed test data
        await SetupDatabase();
    }
}
```

## The Scenario

The heart of the component testing framework is the `Scenario` class. It orchestrates your test steps in a readable, structured way:

```csharp
public sealed class Scenario(ScenarioContext context)
{
    private List<IArrangeStep>? _arrangeSteps;
    private List<IActStep>? _actSteps;
    private List<IAssertStep>? _assertSteps;

    public Scenario Arrange(params IArrangeStep[] arrangeSteps)
    {
        _arrangeSteps ??= [];
        _arrangeSteps.AddRange(arrangeSteps);
        return this;
    }

    public Scenario Act(params IActStep[] actSteps)
    {
        _actSteps ??= [];
        _actSteps.AddRange(actSteps);
        return this;
    }

    public Task Assert(params IAssertStep[] assertSteps)
    {
        _assertSteps ??= [];
        _assertSteps.AddRange(assertSteps);
        return Execute();
    }
}
```

The scenario logs each phase clearly:
- 😶‍🌫️ Background (shared setup)
- 🔧 Arrange (test-specific setup)  
- 🚀 Act (the operation being tested)
- 🔎 Assert (verification)

## The Steps

Each step in the tests is a small class with a single responsibility:

```csharp
public class CreateEventStep : IActStep
{
    public async Task Execute(ScenarioContext scenarioContext, ILogger logger)
    {
        var result = await scenarioContext.GetRequiredService<EventsHttpClient>()
            .CreateEvent(new CreateEventRequest
            {
                Name = "Test Event",
                Description = "This is a test event.",
                Start = DateTimeOffset.UtcNow.AddHours(1),
                End = DateTimeOffset.UtcNow.AddHours(2),
                Address = "123 Test St, Test City",
                Latitude = 12.34,
                Longitude = 56.78
            });

        scenarioContext.StoreOperationResult(result);
    }
}
```

The step interfaces are simple contracts:

```csharp
public interface IActStep : IStep
{
    Task Execute(ScenarioContext scenarioContext, ILogger logger);
}
```

- **Background steps** can be used to set up common preconditions for your tests. They run before any specific test steps and can include things like seeding the database or configuring the application state.
- **Arrange steps** prepare the necessary context for the test. This can include setting up mock objects, configuring services, or any other setup required before the action is performed.
- **Act steps** perform the actual operation being tested. They should be focused on a single action and should not include any assertions or verifications.
- **Assert steps** verify the outcome of the test. They should be independent of the act steps and should only check the final state of the system.

## ScenarioContext: Your Test Gateway

The `ScenarioContext` becomes your gateway to the application and test state:

```csharp
public class ScenarioContext
{
    private readonly Dictionary<string, object> _storage = new();
    
    public T GetRequiredService<T>() where T : notnull
    {
        return _serviceScope.ServiceProvider.GetRequiredService<T>();
    }
    
    public void Set<T>(T value, string key)
    {
        _storage[key] = value ?? throw new ArgumentNullException(nameof(value));
    }
    
    public T Get<T>(string key)
    {
        return (T)_storage[key];
    }
}
```

## Why This Works

**Real Application**: You're testing through HTTP clients against your real application. Same routes, same serialization, same business logic.

**Isolated Infrastructure**: Each test fixture gets its own database container. Tests can run in parallel without conflicts.

**Readable Intent**: The test clearly shows what business scenario it's testing. The step names become living documentation.

**Maintainable Steps**: Change how events are created? Update one step class. Change the API? Update the HTTP client. Tests rarely need to change unless behavior actually changes.

## Performance through parallel execution

Integration tests often lack the ability to run in parallel, leading to longer feedback cycles. By leveraging TestContainers and a multi-tenant architecture, we can achieve significant performance improvements:

- **Shared Database**: All tests share a single database instance, reducing overhead.
- **Isolated Test Data**: Each test runs in its own tenant, ensuring data isolation without the need for separate databases.
- **Fast Execution**: Tests can run concurrently, making better use of available resources.

## Step Execution Logging

The framework provides rich logging for each step execution:

```
😶‍🌫️ Background
   🐾 SeedUser > {"Name":"Test User","Role":"Admin"}
🔧 Arrange
   🐾 SeedEvent > {"Name":"Existing Event"}
🚀 Act
   🐾 CreateEventStep > {"Name":"Test Event"}
🔎 Assert
   🐾 RequestFailed > {"ExpectedError":"EVENT-NAME-ALREADY-EXISTS"}
   ⏱️ RequestFailed completed in 45 ms
```

You can see exactly what each step did (and how long it took if needed).

## Step Execution Logging

This foundation gives you everything you need to start writing component tests that actually add value. The step pattern keeps tests maintainable, the scenario pattern keeps them readable, and TestContainers keeps them reliable.

The goal isn't just to have tests that pass - it's to have tests that give you confidence to deploy and refactor without fear.