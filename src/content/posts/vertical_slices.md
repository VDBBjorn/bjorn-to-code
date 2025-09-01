---
title: 'Vertical Slice Architecture: One Feature, One File'
published: 2025-09-01
draft: false
description: 'Stop organizing by technical concerns. Start organizing by business features. Learn how vertical slice architecture makes your .NET code more maintainable.'
tags: ['dotnet', 'architecture']
---

I used to work on a ticketing platform. Our services were textbook N-tier applications - API layer, CQRS layer, domain model, repository layer, database implementation.

Adding a single field to "add ticket to basket" meant touching every layer. A simple change became a massive PR.

That's when I discovered Jimmy Bogard's talk on [Vertical Slice Architecture](https://www.youtube.com/watch?v=SUiWfhAhgQw). Instead of organizing code by technical layers, organize it by business features.

## This Isn't Clean Architecture with Fewer Projects

Don't confuse Vertical Slice Architecture with flattening your Clean Architecture solution. The difference runs deeper than folder structure.

Clean Architecture forces abstractions whether you need them or not. Repository interfaces for simple CRUD operations. Service layers that just pass data through. Mappers between nearly identical objects. The architecture demands these layers exist.

With vertical slices, you're not forced into any particular pattern. Need to swap data sources? Add an interface. Just querying a database? Use Entity Framework directly. The choice is yours based on what the feature actually needs.

## One Feature, One File

Here's how "add ticket to basket" looks with vertical slices:

```csharp
// AddTicketToBasket.cs - Everything for this one feature

public sealed record AddTicketToBasketRequest
{
    public required Guid TicketId { get; init; }
    public required int Quantity { get; init; }
}

public sealed record TicketAddedResponse
{
    public required Guid BasketItemId { get; init; }
    public required decimal TotalPrice { get; init; }
}

public class AddTicketValidator : AbstractValidator<AddTicketToBasketRequest>
{
    public AddTicketValidator()
    {
        RuleFor(x => x.Quantity).GreaterThan(0);
        RuleFor(x => x.TicketId).NotEmpty();
    }
}

internal sealed class AddTicketEndpoint : IEndpoint
{
    public void MapEndpoint(IEndpointRouteBuilder app)
    {
        app.MapPost("/basket/items", async (
            AddTicketToBasketRequest request,
            IRequestHandler<AddTicketToBasketRequest, TicketAddedResponse> handler) =>
        {
            var response = await handler.Handle(request);
            return Results.Ok(response);
        });
    }
}

internal sealed class AddTicketHandler(BasketDbContext dbContext)
    : IRequestHandler<AddTicketToBasketRequest, TicketAddedResponse>
{
    public async Task<TicketAddedResponse> Handle(AddTicketToBasketRequest request)
    {
        var ticket = await dbContext.Tickets
            .Where(t => t.Id == request.TicketId)
            .Select(t => new { t.Id, t.Price, t.IsAvailable })
            .FirstOrDefaultAsync();

        if (ticket == null || !ticket.IsAvailable)
            throw new TicketNotAvailableException(request.TicketId);

        var basketItem = new BasketItem
        {
            Id = Guid.CreateVersion7(),
            TicketId = request.TicketId,
            Quantity = request.Quantity,
            UnitPrice = ticket.Price
        };

        dbContext.BasketItems.Add(basketItem);
        await dbContext.SaveChangesAsync();

        return new TicketAddedResponse
        {
            BasketItemId = basketItem.Id,
            TotalPrice = basketItem.UnitPrice * basketItem.Quantity
        };
    }
}
```

Everything for this feature is in one file. Want to understand how it works? Read this file. Want to add a field? Change it here. Done.

## What Changed

When I showed this to the team, everyone immediately got it. No more hunting for related classes. No more forgetting to update layer 4 when you changed layer 2.

With vertical slicing, each request can decide how it wants to handle its logic. You no longer need shared layers like repositories, services, and controllers everywhere. One slice might use Entity Framework, another could use raw SQL with Dapper, and a third might call an external API directly. Each feature chooses the approach that makes sense for its specific requirements.

Adding that field I mentioned? Now it's:
- Add to request model
- Use in handler
- One file. One PR.

## When It Works Best

Perfect for request/response operations:
- API integrations where external systems push data
- CRUD operations
- Simple workflows

## The Real Win

Your code becomes organized around the problems you're solving, not the technical patterns you're following.

Use Vertical Slice Architecture for your next project. Start with one feature in one file. See how much clearer your codebase becomes.

---
