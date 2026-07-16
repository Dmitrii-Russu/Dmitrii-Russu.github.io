---
title: "One-to-Many in Java: When a Collection in the Parent Is Justified — and When It Is Not"
date: 2026-07-16 13:00:00 +0200
categories: [Architecture, Persistence]
tags: [java, jpa, hibernate, spring-data-jdbc, jdbc, one-to-many, ddd]
---

The relational model stores the FK on the child table side. In Java we have two ways to reflect this relationship: a collection in the parent entity (`@OneToMany` / `List<Child>`) or a reference in the child (`@ManyToOne` / `long parentId`). The choice affects write behaviour — and this is precisely where most decisions are made without sufficient justification.

---

## The test that gives an unambiguous answer

Vlad Mihalcea puts it this way: the `@ManyToOne` association is the most natural way to map a one-to-many database relationship and is generally the most efficient alternative.

Practical criterion: if you remove the collection and replace it with a separate query, which business rule stops working?

If the answer is "none — the list will simply be fetched by a separate query" — the collection is not needed as part of the model.

If the answer is "an invariant will be violated" — the collection is justified. Typical cases:

- the child entity has no meaning without the parent (value object in DDD terms)
- a business rule requires atomic validation over the entire group (for example, a maximum number of line items in an order)
- the lifecycle of child objects is fully controlled by the parent

---

## How this affects writes

**Reference in the child entity (`@ManyToOne`).** When writing, the child entity can be persisted independently of the parent. The parent object is not loaded. No preliminary fetch — no extra SQL.

**Collection in the parent entity (`@OneToMany`).** Here behaviour depends on the tool.

*JPA/Hibernate* tracks collection changes through a snapshot and generates the necessary `INSERT`/`UPDATE`/`DELETE` statements. Depending on the scenario and mapping configuration, this may require loading the current state of the collection. Additionally, correct entity identity implementation is required — in particular, `equals`/`hashCode` when using `Set` and working with detached objects.

*Spring Data JDBC* does not compute the diff between the old and new state of the child collection. Instead, the child elements of the aggregate are deleted and re-inserted. This is predictable and straightforward, but means a full rewrite on every `save`. For small aggregates this is the norm; for large ones — a conscious constraint.

*JdbcClient / JdbcTemplate* does not manage the graph automatically. If you choose a collection in the parent, you write the synchronisation logic by hand. With a reference in the child — the task is trivial: a single `INSERT` or `UPDATE` specifying the `parent_id`.

---

## Practical conclusion

If business rules do not require aggregate consistency — use a reference in the child entity. It is simpler, cheaper on writes, and introduces no implicit dependencies on collection-loading mechanisms.

If consistency of a group of objects within a single transaction is essential — the collection is justified. In that case, choose your tool deliberately: JPA gives you fine-grained changes, Spring Data JDBC gives you a full rewrite, plain JDBC gives you full control at the cost of manual implementation.

Many developers start modelling with the question "how do I map a FK in Java". A more productive question is: "which objects must change consistently within a single transaction". The answer often automatically suggests whether a collection in the parent is needed — or whether a reference in the child is sufficient.

---

*References:*
- Vlad Mihalcea — [The best way to map a @OneToMany association with JPA and Hibernate](https://vladmihalcea.com/the-best-way-to-map-a-onetomany-association-with-jpa-and-hibernate/)
- My previous article — [JPA handles full graphs well. What about partial ones?](https://www.linkedin.com/pulse/jpa-handles-full-graphs-well-what-partial-ones-dmitrii-russu-py1cf/)
