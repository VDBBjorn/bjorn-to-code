---
title: 'Component tests: from zero to refactoring confidence in 10 minutes'
published: 2025-09-16
draft: true
description: "Get your component tests running in minutes. Learn the practical setup with ServiceFixture, steps, and scenarios using xUnit v3 and the Microsoft Testing Platform."
series: 'Component tests'
tags: ['testing']
---

You've seen why component tests are useful. Now let's build them 😎

## The modern stack: xUnit v3 + Microsoft Testing Platform

We're using **xUnit v3** with the **Microsoft Testing Platform**. Your test projects become standalone executables - no external runners needed.

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net9.0</TargetFramework>
    <OutputType>Exe</OutputType>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="Microsoft.AspNetCore.Mvc.Testing" />
    <PackageReference Include="xunit.v3" />
    <PackageReference Include="Testcontainers.PostgreSql" />
  </ItemGroup>
</Project>
```

Run tests with `dotnet run` or execute the built `.exe` directly. Faster startup, better performance, cleaner architecture.

## ServiceFixture: your API boundary

Define what's real vs fake in your tests:

```csharp
public sealed class ServiceFixture : WebApplicationFactory<Program>, IServiceFixture, IAsyncLifetime
{
    private readonly DatabaseFixture _databaseFixture = new();

    protected override void ConfigureWebHost(IWebHostBuilder builder)
    {
        builder
            .ConfigureTestServices(services =>
            {
                // Real HTTP client → your test server
                services.AddHttpClient<EventsHttpClient>()
                    .ConfigureHttpClient(c => c.BaseAddress = new Uri("http://localhost"))
                    .ConfigurePrimaryHttpMessageHandler(sp => 
                        ((TestServer)sp.GetRequiredService<IServer>()).CreateHandler());

                // Fake time for predictable tests
                services.AddTransient<TimeProvider>(_ => new FakeTimeProvider());
            })
            .ConfigureAppConfiguration((_, config) =>
            {
                config.AddInMemoryCollection(new Dictionary<string, string>
                {
                    ["ConnectionStrings:eventsdb"] = _databaseFixture.ConnectionString
                });
            });
    }

    public async ValueTask InitializeAsync()
    {
        await _databaseFixture.InitializeAsync();
        _ = Services; // Trigger host creation
    }

    public async ValueTask DisposeAsync()
    {
        await _databaseFixture.DisposeAsync();
        await base.DisposeAsync();
    }
}
```

**What's real**: Your application code, database, HTTP routing, dependency injection.  
**What's fake**: External APIs, time, file system.

## Real database with Testcontainers

```csharp
public sealed class DatabaseFixture : IAsyncLifetime
{
    private static readonly PostgreSqlContainer SqlContainer = new PostgreSqlBuilder()
        .WithImage("postgres:17.5-alpine")
        .WithDatabase("eventsdb")
        .Build();

    public string ConnectionString => SqlContainer.GetConnectionString();

    public async ValueTask InitializeAsync() => await SqlContainer.StartAsync();
    public async ValueTask DisposeAsync() => await SqlContainer.DisposeAsync();
}
```

**Testcontainers** runs a real PostgreSQL instance in Docker. Starts once (~2-3 seconds), reused across your test session.

### Why not EF InMemory?

EF's InMemory provider is a testing trap:

- **No SQL validation** - Your test passes, production fails with "invalid SQL syntax"
- **No constraints** - Foreign keys, unique constraints ignored
- **Different behavior** - Transactions, null handling, everything differs

**Testcontainers = real confidence**. Same database engine, same constraints, same behavior.

## Your first component test

```csharp
public class CreateEventTests(ServiceFixture serviceFixture, ITestOutputHelper testOutputHelper)
{
    private readonly ScenarioContext _scenarioContext = new(serviceFixture, testOutputHelper);

    [Fact]
    public Task WhenCreateEventValid_Then_EventIsCreated()
    {
        return new Scenario(_scenarioContext)
            .Act(new CreateEventStep())
            .Assert(new RequestSucceeded(), new EventIsCreated());
    }
}
```

**Three parts**: ScenarioContext (state), Scenario (orchestration), Steps (actions).

## Building reusable steps

### Act: Make the API call

```csharp
public class CreateEventStep : IActStep
{
    public async Task Execute(ScenarioContext context, ILogger logger)
    {
        var utcNow = context.GetRequiredService<TimeProvider>().GetUtcNow();
        var result = await context.GetRequiredService<EventsHttpClient>()
            .CreateEvent(new CreateEventRequest
            {
                Name = "Test Event",
                Description = "This is a test event.",
                Start = new DateTimeOffset(2025, 07, 07, 10, 0, 0, TimeSpan.Zero),
                End = new DateTimeOffset(2025, 07, 07, 12, 0, 0, TimeSpan.Zero),
            });

        context.StoreOperationResult(result);
    }
}
```

### Assert: Verify the outcome

```csharp
public class EventIsCreated : IAssertStep
{
    public async Task Execute(ScenarioContext context, ILogger logger)
    {
        var response = context.GetOperationResponse() as Response<EventCreatedResponse>;
        Assert.NotNull(response);

        var eventsClient = context.GetRequiredService<EventsHttpClient>();
        var fetchedEvent = await eventsClient.GetEventById(response.Body.Id);

        Assert.NotNull(fetchedEvent);
    }
}
```

To verify you basically have two options:
1. **Through the API** - Use your HTTP client to fetch and verify data
2. **Directly in the database** - Query the database to verify state
3. **Combination** - Use API for main verification, DB for edge cases

In an ideal world, you verify through the API. But sometimes direct DB checks are simpler and more reliable.

## Composing complex scenarios

```csharp
[Fact]
public Task WhenEventWithSameNameExists_Then_ErrorIsReturned()
{
    return new Scenario(_scenarioContext)
        .Arrange(new SeedEvent { Name = "Duplicate Event" })
        .Act(new CreateEventStep { Name = "Duplicate Event" })
        .Assert(new RequestFailed().WithError("EVENT-NAME-ALREADY-EXISTS"));
}
```

Each step is reusable. Build your step library once, compose everywhere.

## Background steps for common setup

```csharp
private readonly ScenarioContext _scenarioContext = new ScenarioContext(serviceFixture, testOutputHelper)
    .Background(
        new SeedVenue().WithName("Conference Room"),
        new SeedUser().WithEmail("test@example.com")
    );
```

Background runs before every test. Change seeding logic once, not in 50 tests.

---
