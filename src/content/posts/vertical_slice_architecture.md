---
title: 'Vertical Slice Architecture'
published: 2025-09-01
draft: false
description: 'Stop organizing by technical concerns. Start organizing by business features. Learn how vertical slice architecture makes your .NET code more maintainable.'
tags: ['dotnet', 'architecture']
---

# Vertical Slice Architecture

In my last post, I showed how the REPR pattern (Request-Endpoint-Response) solved the fat controller problem. One operation, one focused class.

But what happens when you have 50+ endpoints? Where do you put them all?

If you just dump them in a folder, you're back to the same problem - hunting through files to find what you need. You need an organization strategy that scales.

That's where Vertical Slice Architecture comes in.

## The Problem with Technical Organization

Most codebases organize by technical concerns:

```
Controllers/
├── BasketController.cs
├── OrderController.cs  
├── CustomerController.cs
Services/
├── BasketService.cs
├── OrderService.cs
├── CustomerService.cs
Models/
├── BasketModels.cs
├── OrderModels.cs
├── CustomerModels.cs
```

To understand "add to basket," you're jumping between Controllers, Services, and Models folders. Three different places, organized by what the code *is* rather than what it *does*.

## Organizing by Business Capability

With Vertical Slice Architecture, you organize by business capability instead:

```
Features/
├── Basket/
│   ├── AddToBasket/
│   │   └── AddToBasket.cs
│   ├── RemoveFromBasket/
│   │   └── RemoveFromBasket.cs
│   └── ClearBasket/
│       └── ClearBasket.cs
├── Orders/
│   ├── CreateOrder/
│   │   └── CreateOrder.cs
│   ├── CancelOrder/
│   │   └── CancelOrder.cs
│   └── GetOrderHistory/
│       └── GetOrderHistory.cs
└── Shared/
    └── BasketDbContext.cs
```

Want to understand basket functionality? Everything's in the Basket folder. Need to modify "add to basket"? Open Features/Basket/AddToBasket/AddToBasket.cs. Done.

## The Hierarchy That Actually Makes Sense

This structure maps well to how businesses actually think:

- **Business Domain** (Basket, Orders) → What area of the business
- **Feature Operations** (AddToBasket, CreateOrder) → What the user wants to do
- **Implementation** (the REPR endpoint) → How we make it happen

We're not organizing by "controllers" and "services." We're organizing by business capabilities and user intentions.

## It's Not Just Folder Structure

VSA isn't about moving files around. It's about three core principles:

**1. Maximize coupling within a slice, minimize coupling between slices**
Everything for "add to basket" lives together. But "add to basket" doesn't depend on "create order."

**2. Axis of change**  
Things that change together should be near each other. When basket business rules change, you work in the Basket folder.

**3. Each slice chooses its own complexity**
AddToBasket might be simple CRUD. CreateOrder might have rich domain logic with multiple entities. That's fine - each slice uses what it needs.

## Our Real Structure

Here's what we actually built:

```csharp
// Features/Basket/AddToBasket/AddToBasket.cs
public static class AddToBasket
{
    public record Request(Guid TicketId, int Quantity);
    public record Response(Guid BasketItemId, decimal TotalPrice);
    
    public class Validator : AbstractValidator<Request>
    {
        public Validator()
        {
            RuleFor(x => x.Quantity).GreaterThan(0);
            RuleFor(x => x.TicketId).NotEmpty();
        }
    }
    
    public class Endpoint : IEndpoint
    {
        public void MapEndpoint(IEndpointRouteBuilder app)
        {
            app.MapPost("/basket/items", Handler);
        }
        
        private static async Task<IResult> Handler(
            Request request,
            BasketDbContext db,
            IValidator<Request> validator)
        {
            // Implementation here...
        }
    }
}
```

Everything for this feature in one file. Request, Response, Validation, Endpoint, Handler. When someone needs to change how "add to basket" works, they open this file and everything they need is right here.

## What About Shared Code?

"But what about the database context? What about shared utilities?"

We keep shared infrastructure in a Shared folder. DbContext, logging, common utilities - things that truly are shared across business domains.

The key word is "truly." Don't share code just because two slices look similar. Share it when you actually need to maintain consistency across domains.

## When Features Get Complex

Simple CRUD operations work great as single files. But what about complex business logic?

```
Features/
├── Orders/
│   ├── CreateOrder/
│   │   ├── CreateOrder.cs              # REPR endpoint
│   │   └── CreateOrderValidator.cs     # Slice-specific validation
│   ├── CancelOrder/
│   │   └── CancelOrder.cs
│   ├── UpdateOrderStatus/
│   │   └── UpdateOrderStatus.cs  
│   └── Shared/                         # Shared within this domain
│       ├── OrderAggregate.cs           # Rich domain model
│       └── OrderEvents.cs              # Domain events
```

Notice the domain aggregate lives in Orders/Shared - it's used by multiple slices within the same business domain, but not across different domains.

Each slice gets the focused REPR endpoint. Complex domain logic gets shared at the domain level, not the slice level.

## The Real Benefits

**For New Team Members:**
Look at the folder structure. You immediately understand what the application does. No hunting through layers to find business logic.

**For Feature Development:**
Working on basket functionality? You live in the Basket folder. No context switching between Controllers, Services, and Models.

**For Debugging:**
Bug in "add to basket"? Open AddToBasket.cs. Everything that could cause the bug is right there.

**For Code Reviews:**
Changes are localized. The reviewer understands exactly what business capability is being modified.

## Start Small

You don't need to restructure your entire codebase. Try this for your next feature:

1. Create a Features folder
2. Add your business domain (e.g., Notifications)  
3. Create one feature slice (e.g., SendWelcomeEmail)
4. Put everything for that feature in one file

See how it feels. See how your team reacts when they need to find and modify that code later.

## It's About Mental Models

Traditional architecture forces you to think in technical layers. VSA lets you think in business terms.

Instead of "I need to add a method to the controller, update the service, and modify the model," you think "I need to create a new feature for email notifications."

The code organization matches how you think about the problem.

---

*Coming up: How to handle shared business logic and complex domain rules within vertical slices, and when to evolve simple REPR endpoints into richer architectures.*