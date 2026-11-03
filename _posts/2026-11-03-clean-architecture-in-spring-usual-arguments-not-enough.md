---
title: "Clean Architecture in Spring: Why the Usual Arguments in Its Favor Are Not Enough"
date: 2026-11-03 08:00:00 +0300
categories: [Architecture, Backend]
tags: [spring-boot, clean-architecture, dao, mockito, archunit, spring-data]
---

Testability, replaceable DAOs, interfaces: this is how Clean Architecture is usually justified. **But all of this is available in plain Spring Boot, without ports and mappers.** So what does it actually give you?

At its core, Clean Architecture is about isolating business logic.

If an application is mostly simple CRUD with no significant business rules, there may be nothing worth isolating in a separate core.

Growing business-logic complexity is often cited as a main reason to adopt Clean Architecture. But is that criterion enough on its own?

---

## Testability and Swappable Implementations

The main benefits usually claimed for Clean Architecture are the ability to swap infrastructure implementations without touching the application core, and the ability to test in isolation.

**Programming to an interface and the DAO pattern are enough for both.** Clean Architecture adds no technical capability here.

To replace dependencies in tests, you can use mocks (Mockito) or fakes. A fake gives you no more independence from Spring than Mockito does: both replace a dependency without starting the application context.

So **you don't need Clean Architecture to swap adapter implementations or to test in isolation.** It is enough to organize dependencies properly and not couple business logic directly to Spring's infrastructure mechanisms.

That said, a unit test with fakes starts neither a JPA provider nor Spring proxies. As a result, JPA annotations and `@Transactional` do not exhibit their infrastructure behavior in such a test. By itself, this is not a reason to move entities out of the core just to make unit testing possible.

---

## The Cost of Isolation

Spring provides ready-made data access support, transactions, pagination, and sorting.

Applying Clean Architecture means adapting part of this functionality, especially when Spring-specific types cross the core boundary. For example, `Pageable` and `Page` from Spring Data either leak into the core, or you need your own pagination types and a mapping between them.

On top of that come additional isolation elements: ports and mappers between layer models.

> All of this increases the number of files and packages, and adds classes and conversions whose place in the structure is not always obvious. **As a result, the code becomes harder to navigate and read.**

---

## Boundaries and Interface Placement

Application boundaries can be enforced with varying degrees of strictness: with ArchUnit tests, or with language and module-system features that enforce them at compile time.

If strict isolation is still needed, an example of compile-time isolation (with entities and transactional annotations moved out of the application layer) is described in [my article](https://dmitrii-russu.github.io/posts/clean-architecture-spring-boot/).

In Clean Architecture, port interfaces live on the inner layer's side, so that source-code dependencies point inward.

But consider an ordinary single-module Spring Boot application.

If a service depends on a DAO interface, moving that interface from an `infrastructure` package to an `application` package changes nothing about DI, implementation swapping, or unit testing.

Of course, the direction of source-code dependencies does change. But as long as the interface contract contains no infrastructure types, its location in an outer package does not get in the way of any of those operations.

So placing the interface in the core gives you no new capabilities for DI, substitution, or testing. **It serves something else: keeping the core independent of infrastructure. That is a matter of boundary enforcement.**

---

## What Clean Architecture Actually Gives You

These typical tasks are handled by plain Spring Boot, without Clean Architecture:

| Task | Plain Spring Boot |
|------|-------------------|
| Swapping an implementation (in tests and in production) | Interface + DI |
| Unit testing a service | Mockito / fake |
| Reusing one piece of business logic from several controllers | Service layer |
| Enforcing architectural boundaries | Package-private adapters, ArchUnit |

**Only the last task genuinely leads to a separate discussion about the need for stricter architectural discipline.**

But even that does not mean Clean Architecture is necessarily required.

Growing business-logic complexity alone is not enough. The practical value of strict isolation shows up when you need not just the ability to swap infrastructure implementations, but a guarantee that the application's internal rules stay independent of external mechanisms as the system evolves: for example, with several independently evolving modules or teams, or a large number of external adapters. Even then, Clean Architecture is not the only way to hold the boundaries: you can make adapters package-private and scan only the packages that contain interfaces and adapters. The compiler will then prevent any code from reaching an implementation directly, bypassing its interface, whether from the core or from any other package.

Clean Architecture offers a stricter organization of an application, but justifying it with testability, interfaces, and replaceable DAOs is not enough: **plain Spring Boot already provides all of that. What has to be shown is the additional practical value of strict isolation itself.**

I am not against Clean Architecture. But it should be applied deliberately, based on the application's actual requirements and the cost of reaching the required degree of isolation.
