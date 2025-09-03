---
title: 'Making Integration Tests Actually Maintainable'
published: 2025-09-02
draft: false
description: 'Stop writing brittle integration tests. Learn how component tests makes your tests readable, reusable, and actually maintainable.'
tags: ['dotnet', 'testing', 'integration-tests', 'component-tests']
---

Integration tests should give you confidence to deploy. Instead, they're usually a maintenance nightmare.

At a previous project I worked on, we had hundreds of unit tests testing internal components in isolation. But we had zero integration tests. The unit tests were brittle - every refactoring randomly broke tests because they were testing implementation details, not behavior.

We couldn't deploy with confidence because we weren't testing the system as a whole.

## Testing from the Outside In

We introduced what we called "component tests" - tests that exercise a single service end-to-end from the outside in. Instead of mocking every dependency and testing individual classes, we test complete user journeys against the real application.

A component test is essentially an integration test scoped to one component. You're testing your entire service in integration - database, message bus, external APIs - but you're not crossing service boundaries into other components.

These tests gave us something the unit tests never could: **confidence to refactor**. As long as the external behavior remained the same, we could restructure the internals without breaking tests.

## Why Not BDD Tools Like SpecFlow?

This approach shares similarities with BDD (Behavior Driven Development), but we chose not to use tools like SpecFlow. SpecFlow for .NET is no longer actively maintained, and frankly, our approach gives you much more flexibility with less complexity.

BDD tools often become maintenance headaches themselves. You're maintaining Gherkin syntax, step definitions, and the glue code between them. Our component test approach gives you the readability benefits of BDD while staying in pure C# with full IDE support, refactoring capabilities, and type safety.

## The Traditional Pain

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

## What If Tests Looked Like This?

```csharp
private readonly Scenario _context = new Scenario()
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

## Component Tests

Instead of cramming everything into test methods, extract the logic into reusable steps:

- **Background** steps set up common test context across related tests
- **Arrange steps** set up test-specific data and state
- **Act steps** perform the operation being tested  
- **Assert steps** verify the outcome and side effects

Each step is a small class that does one thing well. The same steps get reused across multiple tests.

## Why This Works

**Reusability**: That `SeedEvent` step? Used across booking tests, cancellation tests, capacity tests. Change the seeding logic once, all tests update.

**Readability**: Even non-developers can read these tests and understand exactly what business scenarios are covered.

**Maintainability**: When something breaks, you know exactly which business scenario failed, not which HTTP endpoint returned a different JSON structure.

**Debuggability**: Set breakpoints in steps, not buried in 50-line test methods.

## The Transformation

Before: "We can't refactor anything because it breaks all our unit tests."

After: "Our component tests let us refactor with confidence while ensuring the system still works."

Component tests transformed our testing strategy from testing implementation details to testing business behavior. We went from unit tests that broke on every change to robust component tests that only broke when actual behavior changed.

Most importantly: **we could deploy automatically with confidence** because we knew our component hadn't changed for people using it from the outside.

The tests became our safety net for refactoring, our insurance policy when deploying and our documentation for new team members.

---