---
title: 'Own your boundary: component tests for confident refactoring'
published: 2025-09-02
draft: false
description: "Gain the confidence to refactor boldly. Component tests verify business behavior without breaking on implementation changes, filling the gap between fragile unit tests and slow E2E tests."
series: 'Component tests'
tags: ['dotnet', 'testing', 'integration-tests', 'component-tests', 'bdd']
---

You want to refactor your code, but you're worried you might break something important. How do you know your changes won't cause problems?

**The problem: your tests work against you, not for you.**

## Your tests are fighting refactoring

**Unit tests break when you refactor** because they test implementation, not behavior. Move a class? Fix 15 tests. Change your domain model? Rewrite all your mocks.

**E2E tests are too slow** to run during development. You find out they're broken after you push to CI.

You need tests that prove your business logic still works after refactoring, without breaking every time you move code around.

## Component tests: test your boundary, not your internals

Component tests solve this by testing **what you own** through **your API boundary**:

```csharp
// Instead of testing internal classes with mocks...
var service = new EventService(mockRepo, mockValidator);
var result = service.CreateEvent(request);

// Test through your actual API
POST /events { "name": "Tech Conference", "date": "2025-10-15" }
```

**Key principle**: Test behavior through the boundary you control. Everything inside your component is real (database, DI, domain logic). Everything outside is stubbed (external APIs, other services).

This means you can refactor your internals without breaking tests, while still proving your business logic works.

## Make them readable with steps

The problem with most integration tests? They're unreadable:

```csharp
[Fact]
public async Task CreateEvent_WhenValidRequest_ReturnsEventId()
{
    // 20 lines of HTTP setup
    // 10 lines of request building
    // 15 lines of response parsing
    // 10 lines of database verification
}
```

**Step-based tests are readable:**

```csharp
[Fact]
public Task CreateEvent_Success()
{
    return new Scenario()
        .Arrange(new AsUser("john@example.com"))
        .Act(new CreateEvent().WithName("Tech Conference"))
        .Assert(
            new RequestSucceeded(),
            new EventExists().WithName("Tech Conference")
        );
}
```

Each step is a reusable class. Change how you seed users? Update one `AsUser` class, not 50 tests.

Component tests become your main safety net for refactoring. They sit between unit tests (pure domain logic) and E2E tests (critical user journeys), focusing on business behaviors through your API.

:::bjornthinking
Component tests give you **refactoring confidence**: the ability to improve your code structure while proving business behavior stays intact.
:::

**Why this works:**

**Developer-owned** - These tests live in your repo and run with your build. When they fail, it's your code.

**Reusable** - That `CreateEvent` step? Used across permission tests, validation tests, and happy path tests. Build once, use everywhere.

**Maintainable** - When business rules change, update the relevant steps. When implementation changes, tests keep passing.

## The result

**Before**: "We can't refactor because our tests will break."

**After**: "Our tests prove the behavior works. Refactor away."

Component tests become your safety net for refactoring and your documentation for new team members.

---

Next post: How to implement this with real code examples.