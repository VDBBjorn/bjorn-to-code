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

I introduced what we called "component tests" - tests that exercise the entire service from the outside in. Instead of mocking every dependency and testing individual classes, we test complete user journeys against the real application.

These tests gave us something the unit tests never could: **confidence to refactor**. As long as the external behavior remained the same, we could restructure the internals without breaking tests.

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

private readonly ScenarioContext _context = new ScenarioContext(serviceFixture, testOutputHelper)
        .Background(
            new SeedTenant().WithName("Pied Piper"),
            new SeedUser().WithName("Richard Hendricks")
        );

[Fact]
public Task WhenCreateEventValid_Then_EventIsCreated()
{
    return new Scenario(_context)
        .Arrange(
            new AsUser("Richard Hendricks")
        )
        .Act(new CreateEventStep().WithName("Tech Conference 2025"))
        .Assert(
            new RequestSucceeded(),
            new EventIsCreated().WithName("Tech Conference 2025")
        );
}

[Fact]
public Task WhenEventWithSameNameExists_Then_ErrorIsReturned()
{
    return new Scenario(_context)
        .Arrange(new SeedEvent().WithName("Tech Conference 2025"))
        .Act(new CreateEventStep().WithName("Tech Conference 2025"))
        .Assert(new RequestFailed().WithError("EVENT-NAME-ALREADY-EXISTS"));
}
```

Now you can **immediately see what each test does**. The noise is gone. The test names become the documentation.

## Component tests

Instead of cramming everything into test methods, extract the logic into reusable steps:

- **Background** steps set up common test context, including a tenant per test
- **Arrange steps** set up test data
- **Act steps** perform the operation being tested  
- **Assert steps** verify the outcome

Each step is a small class that does one thing well. The same steps get reused across multiple tests.

## Why This Works

**Reusability**: That `SeedEvent` step? Used in all kinds of different tests. Change the seeding logic once, all tests update.

**Readability**: Even non-developers can read these tests and understand exactly what scenarios are covered.

**Maintainability**: When something breaks, you know exactly which business scenario failed.

**Debuggability**: Set breakpoints in steps, not buried in 50-line test methods.

## The Transformation

Before: "We can't refactor anything because it breaks all our unit tests."

After: "Our component tests let us refactor with confidence while ensuring the system still works."

Component tests transformed our testing strategy from testing implementation details to testing business behavior. We went from brittle unit tests that broke on every change to robust component tests that only broke when actual behavior changed.

Most importantly: **we could deploy with confidence** because we knew the entire system worked together, not just individual pieces in isolation.

---
