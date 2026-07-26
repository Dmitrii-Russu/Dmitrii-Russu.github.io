---
title: "\"The service is getting bloated\" — move the logic into the entity? Here's the real criterion"
date: 2026-07-26 12:00:00 +0200
categories: [Architecture, Domain Modeling]
tags: [ddd, java, aggregate, transaction-script, cqrs]
---

# "The service is getting bloated — move the logic into the entity" is not a criterion. Here's the one that actually works

**In short:** an aggregate is worth loading only when its state is required to decide whether the new state is valid. If an operation is fully determined by its input and doesn't check any domain invariants, loading the aggregate is just an expensive intermediate step — not a sign of "doing DDD right."

---

Open almost any article or talk on Domain-Driven Design, and you'll run into the same piece of advice again and again:

> If a service starts growing and business logic accumulates in it — move that logic into the entity.

This has become such a common cliché that it's turned into a default rule: the service is bloated → therefore encapsulation is broken somewhere → therefore the logic must move into the model. The problem is that this criterion describes a symptom, not a cause. The size of a service says nothing about where the logic actually belongs — it only shows that there's a lot of code, and a lot of code can be written correctly or incorrectly regardless of architectural style.

A more useful question is different:

> **Does this operation require the aggregate's state to make its decision?**

If the operation must verify that the new state doesn't violate a domain rule, the state is needed, and loading the aggregate is justified. If the decision is fully derived from the input parameters and doesn't depend on what's already stored in the object, the state isn't needed, and the aggregate becomes an unnecessary step.

This is precisely the question that separates cases where a Rich Domain Model is warranted from cases where a Transaction Script or a plain Application Use Case would be simpler and more efficient — far more precisely than the rule "the service got too big."

It's worth stressing: this isn't about pitting the two approaches against each other. In *Patterns of Enterprise Application Architecture*, Martin Fowler treats Transaction Script, Domain Model, Table Module, and Service Layer as different ways of organizing domain logic, each with its own area of applicability — not as stages in an evolution from "bad" code to "good" code.

More on this: Martin Fowler — *Patterns of Enterprise Application Architecture* (Domain Logic Patterns), https://martinfowler.com/eaaCatalog/

---

## This choice has a cost

Moving behavior inside an aggregate means that before the operation runs, the aggregate must be loaded from storage:

> load the aggregate → execute the behavior → save the aggregate

The reason for that load isn't encapsulation for its own sake — it's that without the aggregate's state, the operation's correctness can't be verified.

This echoes an observation from Fowler:

> *"There's an initial cost in complexity of database access for a domain model which pays off if and only if you have a lot of domain logic."*

A rich domain model requires a more complex data-access mechanism, and that added complexity pays off only when there's enough domain logic to justify it.

More on this: Martin Fowler — *Domain Logic and SQL*, https://martinfowler.com/articles/dblogic.html

Every time we move behavior inside an aggregate, we're implicitly deciding: from now on, this operation requires loading the aggregate first.

---

## But not every operation needs this

Take a piece of code that's common in DDD-style projects:

```java
Portfolio portfolio = portfolioRepository.getById(portfolioId);

portfolio.accrueEarnings(period, totalAmount);

portfolioRepository.save(portfolio);
```

Inside the aggregate:

```java
public void accrueEarnings(Period period, Money totalAmount) {
    for (Share share : shares) {
        Money amount = totalAmount.multiply(share.getPercentage());
        share.increaseAccrued(amount);
    }
}
```

It looks perfectly "DDD-ish": the logic lives inside the entity, the service stays thin. But it's worth asking a simple question:

**What decision does the aggregate actually make here?**

Look closely at the method, and the answer is: none. The percentages are already known and don't change. No invariants are checked. The method simply iterates over a collection and computes a few numbers — an operation fully determined by its input and by data fetched directly from storage, without loading the aggregate.

To get here, we had to load the entire object graph — including shares that never participate in the computation — mutate a few numbers, and save the aggregate again. The rule is formally satisfied: the logic "lives inside the model." But in substance, the aggregate isn't acting as a carrier of a decision — it's acting as an expensive container for arithmetic.

Costs like these can be entirely justified — but only when it's genuinely impossible to decide whether the new state is valid without the aggregate's state. In this case it isn't: the result is fully derived from the input parameters.

---

## So where does the real boundary sit?

A fair question follows: is there a case where a check simply *cannot* be pushed down into the database — via a constraint, a trigger, a stored procedure?

Practically any invariant can, in principle, be implemented at the database level: triggers and stored procedures in most systems are full-fledged programming languages. So the question isn't one of feasibility — it's about choosing the most appropriate place to implement the rule, weighing maintainability, the expressiveness of the model, performance requirements, and the architecture of the specific system.

---

## What if the aggregate is made small?

DDD proponents have a fair response to this critique: aggregates should be as small as possible. Vaughn Vernon develops this recommendation extensively in the *Effective Aggregate Design* series and in *Implementing Domain-Driven Design*.

A smaller aggregate genuinely means less data to load, a smaller object graph, a smaller consistency boundary, and a lower chance of transaction conflicts.

But it's important not to confuse cause and effect here. Shrinking the aggregate reduces the *cost* of loading it — it doesn't eliminate the *need* to load it wherever invariants must be checked. Shrinking aggregates is a way to lower the price of the decision, not a way to change the criterion for making it.

---

## A separate case — read operations

If an operation changes nothing and simply answers a question about current state, it doesn't need an aggregate at all. Loading a full domain model only to discard it right after reading a few values means paying the cost of loading it for no benefit whatsoever.

This is exactly why Command Query Responsibility Segregation (CQRS) emerges naturally here. Commands use aggregates when invariants need protecting. Queries work with projections, read models, or dedicated SQL queries. This isn't a separate architectural rule — it's a direct consequence of the criterion already established.

---

## One more nuance

Even when all business logic is encapsulated inside the aggregate and every invariant is checked successfully in memory, that still doesn't mean the system is fully protected.

Between the read and the write, another transaction can change the data — the classic race-condition problem. Some invariants, by their nature, belong not to the object model but to the storage mechanism.

That's exactly why real systems rely heavily on UNIQUE, CHECK, and FOREIGN KEY constraints, atomic SQL operations, optimistic and pessimistic locking, and MVCC. These mechanisms exist not because DDD fell short, but because some rules simply cannot be reliably verified in memory between a read and a write.

More on this: Vlad Mihalcea — *How to prevent race conditions*, https://vladmihalcea.com/race-condition/

---

## Decision table

| Question | Tool |
|---|---|
| Does the decision depend on current state? | Aggregate |
| Is the decision determined by input? | Transaction Script |
| Read-only? | Read Model |
| Need an atomic guarantee? | DB Constraint / SQL |

---

## Conclusion

Encapsulating behavior inside an aggregate is one of the most powerful tools in Domain-Driven Design. But every powerful tool has a cost: moving logic into an aggregate means it has to be loaded before the operation can run.

That cost is justified when it buys you invariant checking. If it doesn't, a natural question follows: why load the aggregate at all? Testability isn't a real argument here — both approaches are equally well covered by unit tests; the difference is only in the shape of the tests.

In every other case, a Transaction Script, an Application Use Case, or a dedicated SQL statement isn't a compromise — it's the more appropriate engineering choice. Each of these tools exists to solve a different problem, and good architecture doesn't start with picking a favorite pattern — it starts with understanding what problem each tool solves and what price you pay for using it.
