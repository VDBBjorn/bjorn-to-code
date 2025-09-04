---
title: 'Own Your Boundary: Component Tests for Confident Refactoring'
published: 2025-09-02
draft: false
description: "Replace 50-line boilerplate integration tests with structured, readable component tests. Test business behavior instead of implementation details using a step-based approach that doesn't break when you refactor."
series: 'Component tests'
tags: ['dotnet', 'testing', 'integration-tests', 'component-tests', 'bdd']
---
Integration tests should give you confidence to deploy. Instead, they're usually a maintenance nightmare.

## The Problem: Integration Test Hell

Here's what most integration tests look like:

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

Every test is 50+ lines of boilerplate. You can't see what the test is actually testing. Copy-paste logic everywhere. Change one thing, break five tests.

**The real cost isn't just maintenance time**. 
- Your deployment pipeline becomes fragile - tests fail on unrelated changes, blocking releases. 
- Teams avoid refactoring because it means fixing dozens of tests. 
- Architecture decisions get driven by "what won't break the tests" instead of "what's the right design."

Meanwhile, your unit tests are brittle - every refactoring breaks them because they test implementation details, not behavior. You can't deploy with confidence because you're not testing the system as a whole.

The result: technical debt accumulates, velocity drops, and your test suite becomes an obstacle to good architecture instead of enabling it.

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

[Fact]
public Task WhenCreateEventWithDuplicateName_Then_ValidationError()
{
    return new Scenario(_context)
        .Arrange(new SeedEvent().WithName("Tech Conference"))
        .Act(new CreateEvent().WithName("Tech Conference").WithDate("2025-11-20"))
        .Assert(new RequestFailed().WithError("DUPLICATE_EVENT_NAME"));
}
```

Now you can **immediately see what each test does**. The noise is gone. The test names become living documentation of your business rules.

## The Solution: Component Tests

This approach uses what we call "component tests" - integration tests scoped to a single component that you own.

**Why not just better unit tests?** Unit tests will always break when you refactor because they're coupled to implementation details. Plus, they require extensive mocking and stubbing of dependencies - creating another layer of maintenance burden and potential false positives. End-to-end tests are too slow and fragile for rapid feedback. Component tests hit the sweet spot: they test real behavior through your actual API with real dependencies, but stay scoped to what you control.

**The game-changer is refactoring confidence.** Component tests enabled us to aggressively refactor internal implementation while maintaining behavior contracts. We could completely restructure our domain model, swap out infrastructure components, and optimize performance - all while the tests continued to pass because the business behavior remained unchanged.

:::bjorn
The real value of component tests isn’t just catching bugs. It’s the **confidence to refactor boldly**, knowing your business behavior is protected.
:::

### Testing from the Outside In

Instead of mocking every dependency and testing individual classes, we test complete user journeys against the real application. A component test verifies that when you call your service's API with specific inputs, you get the expected outputs and side effects.

### The Right Scope

Think of it this way: if you're responsible for the "Basket" service, your component tests verify that service works correctly in integration - but they don't test the downstream "Inventory Service" or "Payment Service" that other teams own.

This scoping is crucial because:
- **You own the test failures** - when a component test breaks, it's your code that needs fixing
- **Fast feedback loops** - tests run in your development environment with your build
- **Clear ownership** - each team maintains tests for their own components

## How Component Tests Work

Instead of cramming everything into test methods, extract the logic into reusable steps:

- **Background** steps set up common test context across related tests
- **Arrange steps** set up test-specific data and state
- **Act steps** perform the operation being tested  
- **Assert steps** verify the outcome and side effects

Each step is a small class that does one thing well. The same steps get reused across multiple tests.

## Where Component Tests Fit in the Test Stack

Write each scenario at the lowest level that can express the behavior without mocking or depending on private structure. A scenario should live in exactly one layer.

### Where Each Test Type Fits

**Unit tests** prove your pure domain logic—think calculation rules, invariants, and complex branching. Use them for pure functions, but avoid mock-heavy tests that just restate your implementation.

**Component tests** are your main safety net for refactoring. They run at the boundary you own (like your API or handler), with real dependencies wired up (database, serializers, DI). Focus on business behaviors, side effects, validation, and state transitions. Don’t call other services—stub at the boundary.

**End-to-end tests** cover a handful of critical user journeys that span multiple services (signup, purchase, billing). Keep just 1–3 happy paths and a key failure case. Avoid a matrix of permutations already covered by component tests.

**Contract/consumer tests** (optional) lock down request/response expectations between services, without the full cost of E2E. Use them to verify versioned payload shapes and semantics, but don’t recheck business rules already covered by component tests.

## Why Not BDD Tools Like SpecFlow?

This approach shares similarities with BDD (Behavior Driven Development), but we chose not to use tools like SpecFlow or [Reqnroll](https://reqnroll.net/). SpecFlow for .NET is no longer actively maintained, and frankly, our approach gives you much more flexibility with less complexity.

BDD tools often become maintenance headaches themselves. You're maintaining Gherkin syntax, step definitions, and the glue code between them. Our component test approach gives you the readability benefits of BDD while staying in pure C# with full IDE support, refactoring capabilities, type safety, and debugging with breakpoints.

## Why This Works

**Developer-owned**: These tests live in your component's repository and run as part of your build. When they fail, you know it's your code that needs attention.

**Reusability**: That `SeedEvent` step? Used across all different kinds of tests. Change the seeding logic once, all tests update.

**Readability**: Even non-developers can read these tests and understand exactly what business scenarios are covered.

**Maintainability**: When something breaks, you know exactly which business scenario failed, not which HTTP endpoint returned a different JSON structure.

**Debuggability**: Set breakpoints in steps, not buried in 50-line test methods.

## A Word of Caution: Don't Overextend

You *could* theoretically extend this framework to test across multiple real components - essentially running end-to-end tests using the same step-based approach. However, **be very careful here**.

Keep component tests focused on what you own. Let other teams use the tools they're comfortable with for broader integration scenarios.

## The Transformation

**Before**: "We can't refactor anything because it breaks all our unit tests."

**After**: "Our component tests let us refactor with confidence while ensuring the system still works."

Component tests transformed our testing strategy from testing implementation details to testing business behavior. We went from unit tests that broke on every change to robust integration tests that only broke when actual behavior changed.

Most importantly: **we could deploy automatically with confidence** because we knew our component hadn't changed for people using it from the outside.

The tests became our safety net for refactoring, our insurance policy when deploying and our documentation for new team members.

---

In the next post, I'll share how to implement this step-based testing framework in your .NET projects, complete with code examples and best practices.