---
title: 'Own Your Boundary: Component Tests for Confident Refactoring'
published: 2025-09-02
draft: false
description: "Gain the confidence to refactor boldly. Component tests verify business behavior without breaking on implementation changes, filling the gap between fragile unit tests and slow E2E tests."
series: 'Component tests'
tags: ['dotnet', 'testing', 'integration-tests', 'component-tests', 'bdd']
---

You want to refactor your code, but you're paralyzed by fear. Will your changes break something? Are you preserving the same business behavior? 

**The problem: neither unit tests nor E2E tests give you the confidence you need.**

## The Core Problem: No Safety Net for Refactoring

Here's what happens when you try to refactor:

**Unit tests break on every change** because they're coupled to implementation details. Move a method between classes? Fix 10 tests. Restructure your domain model? Rewrite dozens of mocks. Unit tests tell you if your internal structure changed, not if your business behavior is preserved.

**E2E tests are too slow and fragile**. They live outside your development environment, require multiple services running, take minutes to execute, and are often owned by QA teams. You don't run them locally—you only find out they're broken when CI fails.

**The gap in the middle is killing your velocity.** You need tests that verify business behavior without breaking on implementation changes, but you also need them to run fast in your development environment.

## What You Actually Need

You need tests that give you **refactoring confidence**:
- Test behavior, not implementation details
- Run fast during development (seconds, not minutes)  
- Only test what you own and control
- Don't break when you restructure internal code

This is where component tests shine.

## The Reality: Your Current Integration Tests Are Fighting You

When you try to write integration tests today, you end up with this:

```csharp
[Fact]
public async Task CreateEvent_WhenValidRequest_ReturnsEventId()
{
    // 20 lines of HTTP client setup
    // 10 lines of request building  
    // 5 lines of calling the API
    // 15 lines of parsing and asserting the response
    // 10 lines of verifying database state
}
```

Every test is 50+ lines of boilerplate. You can't see what's being tested. Change one thing, break five tests. These tests don't give you confidence—they give you maintenance burden.

## What If Tests Looked Like This Instead?

```csharp
private readonly ScenarioContext _context = new ScenarioContext()
        .Background(
            new SeedUser().WithName("John Smith")
        );

[Fact]
public Task WhenCreateEvent_Then_EventIsCreated()
{
    return new Scenario(_context)
        .Arrange(new AsUser("John Smith"))
        .Act(new CreateEvent().WithName("Tech Conference").WithDate("2025-10-15"))
        .Assert(
            new RequestSucceeded(),
            new EventExists().WithName("Tech Conference")
        );
}
```

Now you can **immediately see what behavior is being tested**. This test doesn't care about your internal class structure, your database schema, or how you implement the business logic. It only cares that when someone creates an event, the event gets created. 

**Refactor away—this test will still pass as long as the behavior is preserved.**

## The Solution: Component Tests

Component tests are integration tests scoped to a single component that you own. They test real behavior through your actual API with real dependencies, but stay focused on what you control.

**The game-changer is refactoring confidence.** Component tests enabled us to aggressively refactor internal implementation while maintaining behavior contracts. We could completely restructure our domain model, swap out infrastructure components, and optimize performance - all while the tests continued to pass because the business behavior remained unchanged.

:::bjorn
The real value of component tests isn't just catching bugs. It's the **confidence to refactor boldly**, knowing your business behavior is protected.
:::

## How Component Tests Work

Instead of cramming everything into test methods, extract the logic into reusable steps:

- **Background** steps set up common test context across related tests
- **Arrange steps** set up test-specific data and state
- **Act steps** perform the operation being tested  
- **Assert steps** verify the outcome and side effects

Each step is a small class that does one thing well. The same steps get reused across multiple tests.

## Where Component Tests Fit in the Test Stack

**Unit tests** prove your pure domain logic—think calculation rules, invariants, and complex branching. Use them for pure functions, but avoid mock-heavy tests that just restate your implementation.

**Component tests** are your main safety net for refactoring. They run at the boundary you own (like your API or handler), with real dependencies wired up (database, serializers, DI). Focus on business behaviors, side effects, validation, and state transitions. Don't call other services—stub at the boundary.

**End-to-end tests** cover a handful of critical user journeys that span multiple services. Keep just 1–3 happy paths and a key failure case. These often live outside your dev environment and are owned by QA teams.

**Contract/consumer tests** (optional) lock down request/response expectations between services, without the full cost of E2E.

## Why This Works

**Developer-owned**: These tests live in your component's repository and run as part of your build. When they fail, you know it's your code that needs attention.

**Reusability**: That `SeedEvent` step? Used across all different kinds of tests. Change the seeding logic once, all tests update.

**Readability**: Even non-developers can read these tests and understand exactly what business scenarios are covered.

**Maintainability**: When something breaks, you know exactly which business scenario failed, not which HTTP endpoint returned a different JSON structure.

## The Transformation

**Before**: "We can't refactor anything because it breaks all our unit tests."

**After**: "Our component tests let us refactor with confidence while ensuring the system still works."

Component tests became our safety net for refactoring, our insurance policy when deploying, and our documentation for new team members.

---

In the next post, I'll share how to implement this step-based testing framework in your .NET projects, complete with code examples and best practices.