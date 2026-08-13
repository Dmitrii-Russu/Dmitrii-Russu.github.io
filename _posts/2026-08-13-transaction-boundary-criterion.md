---
title: "\"Grouping SQL Queries\" Is Not a Transaction Boundary. Here's the Real Criterion"
date: 2026-08-13 12:00:00 +0200
categories: [Architecture, Transactions]
tags: [transactions, spring, jpa, repository-pattern, layered-architecture]
---
In short: **a transaction boundary isn't determined by which SQL queries physically execute together, but by which component knows about the business invariant that ties multiple repository calls together.** In data-centric code these two things coincide — which is why the classic examples become confusing the moment a layered architecture appears: the repository stops being the place where business meaning lives, and only the service still knows what must happen — or not happen — as a single unit.

## The Intuition Everyone Starts With

Open almost any explanation of database transactions and you'll see the same example: BEGIN, a couple of UPDATEs, COMMIT. Debit one account, credit another, wrap both queries in a transaction so that either both happen or neither does.

The conclusion everyone draws from this is simple, and at this level entirely correct: a transaction is a group of SQL queries that must succeed or roll back together. The natural next step is to treat "which queries to group" as the whole question — find the operations that touch related data, wrap them, done.

The problem isn't that this reasoning is wrong. The problem is that it's only correct under a condition nobody states explicitly: **that the SQL script *is* the business operation.** In pure data-centric code there simply is no other artifact where the semantics of "a transfer" could live — the script *is* the transfer. The business operation and the database operation are literally the same thing. There's nothing to separate, because there's only one layer.

## What Changes Once a Repository Appears

The moment a layered architecture appears, this condition quietly stops holding, and almost no one revisits the rule that depended on it.

In a layered application, the repository typically doesn't know what a "transfer" is. It knows how to save and read data — that's its job. A repository method is parameterized by what to read or write: an id, an amount, an entity. This isn't an accident of coding style — look at the signatures of any standard repository interface (say, `save(entity)`, `findById(id)`, `deleteById(id)` in Spring Data JPA): they're typed around data, not around scenarios, because the repository's role was defined that way from the start. **It has no parameter for "why" it was called, and no way to express that this particular update only makes sense in the presence of another one.** This isn't an oversight someone forgot to fix — it's literally what keeping a repository a pure persistence component means.

So business logic accumulates where it *can* be expressed — in the service. That's also where the business operation itself now lives, as a named thing: "transfer" is no longer a script, it's a method. The individual repository calls the service makes are no longer atomic units of meaning — they're primitives the service composes to satisfy an invariant that only it knows about.

This is exactly what the classic example never shows — because in the classic example there was never any composition to begin with: one script, one transaction, one obvious boundary.

## The Criterion That Survives the Split

Once the business operation and the database operation stop being the same artifact, **"group the SQL queries" isn't so much wrong as it is the wrong question.** Queries don't decide anything, they just execute. The question that actually determines the boundary is:

**Which component knows which operations need to be consistent with each other?**

In a typical layered architecture, that boundary runs through the service — that's where the repository's CRUD primitives get assembled into a use case, and that's where the service knows the invariant "both movements happen together or not at all."

This doesn't mean a repository can't know *any* invariants — it can very well enforce local persistence invariants like "balance cannot go negative" or a uniqueness constraint. The point is different: the moment a repository starts defining a business invariant that spans multiple operations, it has already stopped being a pure persistence primitive in the architectural sense being discussed here.

This also reframes propagation, though that topic deserves its own treatment. Formally, REQUIRED is the rule "join the current transaction if one exists; otherwise create a new one." But behind that rule is the same question as above: who — the outer call or the inner call — has priority in defining the boundary. REQUIRED gives priority to whoever already opened the transaction: the inner call joins rather than insisting on its own. REQUIRES_NEW does the opposite — it suspends the current transaction and creates a new one, meaning the inner call insists on its own boundary regardless of what's already open. Neither is really about SQL queries at all — both are about which component decides what's atomic with what.

## Why This Confusion Is So Persistent

None of this is obvious in hindsight, and it's worth saying why the misleading intuition is so sticky, rather than chalking it up to a simple mistake.

Data-centric code isn't a strawman — it's a real, valid way to write software, and for a long time it was the default. In that world, the rule "group the queries that must execute together" isn't a simplification of the truth, it *is* the truth, because there's no second layer where a business operation could hide. The rule only becomes wrong when it's carried over, unexamined, into an architecture where the premise it depended on — one artifact, one layer — no longer holds. **The rule didn't get worse. The ground it stood on shifted.**

That's usually how outdated criteria survive: not because anyone actively defends them, but because the conditions that made them true stop being visible once the architecture they matched has disappeared.
