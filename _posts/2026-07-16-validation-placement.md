---
title: Criteria for Placing Validation in an Application
date: 2026-07-16 12:00:00 +0200
categories: [Architecture, Persistence]
tags: [validation, transaction-script, rich-domain-model, fat-database, concurrency, database, java]
---

## 1. Context and Scope

The question "where should validation live" is usually reduced to a dispute between Fat Database, Transaction Script, and Rich Domain Model. In practice this is the wrong framing: the architectural style only determines where the *bulk* of business logic lives, not where a *specific* check should be placed. Below is a model that approaches this decision through three questions: rule type, concurrency risk, and the role of the check (advisory vs. authoritative).

An application consists of two main parts: the database and the application code. This document is about placing **state invariants** and **business-logic parameters**, i.e. conditions that data and operations on it must satisfy. It does not cover syntactic input validation (email format, string length, required fields): that kind of validation is always performed at the system boundary, in a DTO/controller, and is out of scope here.

Possible architectural styles for placing business logic:

- **Fat database.** All validation and business logic is delegated to the database; the application code can be a single flat controller.
- **Transaction script.** Validation lives in a service and/or controller.
- **Rich domain model.** Validation is encapsulated inside entities.

The chosen style determines where the *bulk* of business logic lives, but it doesn't resolve the separate question of where a *specific* check should be placed. Below are three questions asked of every rule in turn. The rule's type determines which protection mechanisms are even applicable, so these are not three independent measurements but a chain of successively narrowing decisions. They are brought together into a single decision tree at the end of the document.

## 2. First Question: Rule Type

### 2.1. Structural Invariants

A condition checked at the level of a single record or a relationship between records, requiring no knowledge of the business process: uniqueness, referential integrity, value ranges, type conformance, a condition expressible as a CHECK (balance ≥ 0, percentages sum to 100). Such a condition is naturally checked by the DBMS.

### 2.2. Behavioral (Aggregate) Invariants

A condition that concerns not a single record but the consistency of an aggregate or a sequence of operations: "an order must contain at least one line item," "an application cannot move to status 'paid' without first passing through 'confirmed'," "the sum of ledger entries in a transaction must balance." Some of these conditions can formally be encoded in the database (triggers, stored procedures), but that requires moving aggregate logic into a layer not designed to express it. Such invariants generally belong in code (a transaction script or a domain entity); at best, the database can back them up with a projection, e.g. a materialized field with a CHECK (see section 7).

### 2.3. Contextual Business Rules

A condition that depends on variables the database cannot access directly: user role, pricing tier, the stage of an external process, data from a third-party service. Such a rule belongs in code.

### 2.4. Technique: Decomposing a Composite Rule

Many real-world rules are composite and don't fall cleanly into a single category above. A classic example: "you cannot withdraw more than the account balance." This is simultaneously a structural invariant (the balance cannot go negative) and a contextual rule (the limit depends on the overdraft, customer role, account type). In such cases the rule is split into layers:

- **invariant core**: the part that holds for any customer and any system state (balance ≥ 0), which belongs as a CHECK constraint in the database;
- **contextual layer**: the part that depends on business variables (overdraft limit), which belongs in code.

Practical test: if you strip away all business context (roles, pricing tiers, feature flags), does the condition still make sense and still hold? If yes, it's the core. If it disappears along with the context, it's the contextual layer.

### 2.5. Criterion

A rule is first checked for compositeness and split if needed. Each part is then classified into one of three classes: a structural invariant is a candidate for a database constraint; a behavioral invariant usually stays in code, with the database optionally backing it up via a projection; a contextual rule belongs in code. The wording around constraints is deliberately cautious ("candidate"): whether the specific DBMS can express it as a constraint, the cost of maintaining the schema, and how often the rule changes are additional factors in the final decision.

## 3. Second Question: Concurrency

Regardless of which class a rule falls into, a separate question arises: can it be violated by a concurrent write, i.e. a situation where two simultaneous processes read the same state, each independently decides its own write is valid, and both write, so that the resulting state violates the rule.

There's a temptation to conclude that since the check already exists in code, duplicating it in the database is redundant. Practice shows the opposite. Incidents such as Flexcoin and GitHub demonstrate that inadequate control of concurrent access leads to data loss or integrity violations, precisely in cases where a check in code was formally present but not atomic with respect to the write.

This is not about "defense in depth" in the sense of redundant duplication for extra safety. The read → decide → write pattern provides no guarantee against races no matter how many layers of checking exist in code: there is always a window between the read and the write in which the state can change. The only way to close that window is with database-level mechanisms: constraints, transactions, isolation levels, locks, or optimistic concurrency control.

### 3.1. A Caveat on Duplication: Where Protecting the Core Ends

The requirement for database-level protection is unconditional, but only for the **invariant core**: here duplication is not optional, because no other guarantee of atomicity exists. For the **contextual layer**, duplicating the check in the database is not recommended by default: its cost (a second place to maintain, the risk of the two getting out of sync as the rule changes) usually outweighs the benefit.

An important nuance: a lock protecting the core automatically protects the contextual part only if the contextual data is read inside the same transaction, after that lock has been acquired. If contextual data (say, an overdraft limit) lives in another table and is read before the balance row is locked, the race remains: a concurrent process can change the limit in between. In that case there are three options, and choosing among them is a deliberate trade-off, not something automatic:

1. read the contextual data after acquiring the lock, within the same transaction (if architecturally feasible);
2. raise the isolation level to serializable for that operation (more expensive in terms of performance, see section 7);
3. protect the contextual data with its own versioning mechanism, independent of the core.

**Example (option 3).** An online store: a 10% discount for VIP customers is a contextual rule, not a structural invariant, so there's no need to duplicate it as a separate database constraint. The risk: a customer's VIP status is changed in the admin panel while they're placing an order. The solution is not to move the "VIP discount" rule into the database, but to add a `version` column to the user record and, when placing the order, run `UPDATE users SET version = version + 1 WHERE id = $1 AND version = $2` within the same transaction as the charge. This is an optimistic lock on the contextual data, separate from the lock protecting the balance: the contextual rule is protected without moving its logic into the database.

### 3.2. Criterion

If a rule (or its invariant core) is subject to races, database-level protection is mandatory regardless of architectural style. For the contextual part, duplication is justified only if it is not covered by the lock protecting the core (see 3.1) and the risk of violating it is critical.

## 4. Third Question: Authoritative vs. UI Convenience

An **authoritative check** is the point whose result finally decides the fate of an operation. There is exactly one such point per rule (given sections 2–3: for an invariant core under race risk, that point is the database).

An **advisory check** is a fast check in the UI or at the controller entry point, before the database is reached. Its purpose is to keep the user from going through the whole flow just to be rejected where the error could have been shown immediately. Put simply, advisory is a preliminary, non-binding answer ("probably fine"), while authoritative is the final verdict. Advisory does not replace authoritative and may duplicate its logic, which is fine, since it serves a different purpose.

Criterion: advisory checks are added for UX reasons, not for reliability. Their absence is not critical to system integrity; their presence does not remove the requirements from sections 2–3.

## 5. Final Decision Tree

The questions from sections 2–4 are not asked in parallel but in sequence: the answer to the first one (rule type) narrows the set of applicable protection mechanisms in the second one (section 7). So the tree below is a chain of narrowing decisions, not three independent measurements.

1. **Is the rule composite?** Split it into an invariant core and a contextual layer (2.4), then apply steps 2–4 to each part separately.
2. **Which of the three classes does the rule belong to?** (see section 2)
   - Structural invariant → CHECK/constraint in the database, an authoritative candidate.
   - Behavioral (aggregate) invariant → code (service/entity); if the invariant can be expressed as a projection, add a CHECK on it as a backstop (section 7).
   - Contextual rule → authoritative logic strictly in code; the database is no help here.
3. **Can a concurrent write violate the rule?**
   - Yes, invariant core → database-level protection is mandatory (choice of mechanism in section 7).
   - Yes, contextual part → check whether it is read after the core's lock, within the same transaction (3.1); if not, and the risk is critical, a separate versioning mechanism is needed; otherwise it can stay in code.
4. **Is instant feedback needed in the UI?** → add an advisory check. This is an addition, not a replacement for steps 2–3.

**Cross-cutting examples:**

| Rule | Type | Race risk? | Solution |
|---|---|---|---|
| User email is unique | Structural invariant | Yes (double registration) | unique constraint in the database |
| Money transfer between accounts | Structural core (balance ≥ 0) + context (limits) | Yes | SELECT FOR UPDATE on the accounts, limits checked in code within the same transaction |
| Booking flight seats | Structural invariant (remaining_seats ≥ 0) | Yes, high contention | optimistic lock (version) or CHECK + retry |
| Order contains ≥1 line items | Behavioral (aggregate) invariant | Usually no (checked at the use-case boundary) | code (the Order entity, on creation/checkout) |
| GitHub incident (simplified) | — | read → check → write without locking | illustrates what results from ignoring section 3 |

## 6. Fit with Architectural Styles

- **Fat database.** This style exists not because constraints happen to cover structural invariants well, but because all business logic is deliberately centralized in the database, via triggers and stored procedures, including behavioral and contextual rules. Structural invariants (2.1) are naturally checked by the DBMS regardless of the chosen style: this is not a privilege of Fat Database, but a consequence of it. The style's real problem shows up precisely with behavioral and contextual rules: moving that logic into the database scales poorly and complicates maintenance.
- **Transaction script.** The bulk of logic lives in the service/controller layer. For an invariant core under race risk, the database-level requirement remains mandatory regardless of style.
- **Rich domain model.** Encapsulating validation in an entity addresses code cohesion and readability, but does not remove the requirement from section 3: encapsulation is a guarantee within a single in-memory process, not a guarantee of atomicity across concurrent transactions.

The choice of style determines where the bulk of business logic lives, but it does not remove the need to protect, in the database, an invariant core that is subject to concurrent writes. This requirement is the same for both transaction script and rich domain model.

## 7. Database-Level Mechanisms

| Type of risk | Mechanism | When to use |
|---|---|---|
| Uniqueness/structural violations | unique constraint, foreign key, CHECK constraint | Whenever applicable, the cheapest and most reliable solution. Example: a unique constraint on email |
| Lost update, low contention | optimistic lock (version column) | Record editing by a user with a delay between read and write. Example: VIP discount, low-load seat booking |
| High contention, critical integrity | SELECT FOR UPDATE (pessimistic lock) within a transaction | Money, stock levels, counters on a shared resource. Example: debiting a balance |
| Complex invariants spanning several tables | serializable isolation, or an explicit transaction with a check before commit | When a constraint isn't expressive enough. Caution: serializable isolation in PostgreSQL under load produces frequent `serialization_failure` errors and requires retry logic on the application side. It's not a universal solution, but a last resort |
| Aggregate invariants (sum of children = parent) | a materialized field, updated within the same transaction as the child records, plus a CHECK on that field | When recalculating the aggregate on the fly is expensive, but a guarantee is needed at commit time |

This section is the second step, after section 3: first the *necessity* of database-level protection is established, then the *specific mechanism*.

## 8. Scope of Applicability

These criteria are not a rigid standard but a frame of reference for deciding on each rule individually. For an invariant core subject to races (section 3), the requirement for database-level protection remains unconditional: it is backed by the cost of real-world failures. For behavioral and contextual rules (2.2–2.3), a team may deliberately deviate from the recommendations, for example forgoing a CHECK constraint on an aggregate in favor of eventual consistency, if that trade-off is deliberately reflected in the overall system design and the risk of temporary inconsistency is acceptable to the business.
