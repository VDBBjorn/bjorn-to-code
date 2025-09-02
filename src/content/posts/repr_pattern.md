---
title: 'REPR pattern'
published: 2025-09-01
draft: false
description: 'Learn how the REPR pattern can simplify your API design by focusing on user intentions.'
tags: ['dotnet', 'architecture']
---

# REPR Pattern

I used to build APIs the traditional way. Fat controllers with 10+ methods. AddTicket, RemoveTicket, UpdateTicket, GetTickets, SearchTickets... you know the drill.

Each controller method needed different dependencies injected. The constructor grew to 8 parameters. Adding a new endpoint meant touching a class that already handled six other operations.

Then I discovered the REPR pattern (Request-Endpoint-Response). Instead of grouping operations by "entity," group them by "what the user wants to do."

## One Endpoint, One Class

Here's how "add ticket to basket" looks with REPR:

```csharp
// AddToBasket.cs - Everything for this one request

public sealed record AddToBasketRequest
{
    public required Guid TicketId { get; init; }
    public required int Quantity { get; init; }
}

public sealed record AddToBasketResponse
{
    public required Guid BasketItemId { get; init; }
    public required decimal TotalPrice { get; init; }
}

public class AddToBasketValidator : AbstractValidator<AddToBasketRequest>
{
    public AddToBasketValidator()
    {
        RuleFor(x => x.Quantity).GreaterThan(0);
        RuleFor(x => x.TicketId).NotEmpty();
    }
}

internal sealed class AddToBasketEndpoint : IEndpoint
{
    public void MapEndpoint(IEndpointRouteBuilder app)
    {
        app.MapPost("/basket/items", Handler);
    }
    
    private static async Task<IResult> Handler(
        AddToBasketRequest request,
        BasketDbContext dbContext,
        IValidator<AddToBasketRequest> validator)
    {
        var validationResult = await validator.ValidateAsync(request);
        if (!validationResult.IsValid)
            return Results.ValidationProblem(validationResult.ToDictionary());

        var ticket = await dbContext.Tickets
            .Where(t => t.Id == request.TicketId)
            .Select(t => new { t.Id, t.Price, t.IsAvailable })
            .FirstOrDefaultAsync();

        if (ticket == null || !ticket.IsAvailable)
            return Results.Problem("Ticket not available", statusCode: 400);

        var basketItem = new BasketItem
        {
            Id = Guid.CreateVersion7(),
            TicketId = request.TicketId,
            Quantity = request.Quantity,
            UnitPrice = ticket.Price
        };

        dbContext.BasketItems.Add(basketItem);
        await dbContext.SaveChangesAsync();

        var response = new AddToBasketResponse
        {
            BasketItemId = basketItem.Id,
            TotalPrice = basketItem.UnitPrice * basketItem.Quantity
        };

        return Results.Ok(response);
    }
}
```

Everything for this operation lives in one file. The Request model, validation, endpoint mapping, and business logic.

## Why REPR Works

**Single Responsibility**: Each endpoint class does exactly one thing. Need to change how "add to basket" works? Open AddToBasket.cs. Done.

**No Dependency Bloat**: Each endpoint only injects what it needs. AddToBasket needs DbContext and a validator. RemoveFromBasket might need something completely different.

**Easy Testing**: Want to test the add-to-basket logic? You're testing one focused class, not a controller that handles six different operations.

**Team Scalability**: Multiple developers can work on different operations without merge conflicts. Sarah works on AddToBasket.cs, Tom works on CreateOrder.cs.

## What About Code Duplication?

Some code might get duplicated across endpoints. That's okay. Duplication is cheaper than the complexity of shared services that try to do too much.

## The Mental Shift

Instead of thinking:
- "I need a BasketController with Add, Remove, Clear methods"

Think:
- "Users want to add items to their basket" → AddToBasket endpoint
- "Users want to remove items" → RemoveFromBasket endpoint  
- "Users want to clear their basket" → ClearBasket endpoint

Each operation becomes a focused, testable, maintainable piece of code.

---