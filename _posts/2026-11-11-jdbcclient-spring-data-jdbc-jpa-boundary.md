---
layout: post
title: "JdbcClient, Spring Data JDBC, and JPA: Where the Boundary Between the Domain Model and Persistence Lies"
date: 2026-11-11
description: "What a data access technology imposes on your architecture before you write your first repository: JPA, Spring Data JDBC, and JdbcClient compared through the boundary between the domain model and persistence."
tags: [java, spring, jpa, jdbc, architecture, ddd]
---

**What does a data access technology impose on your architecture before you write your first repository?**

Even if all SQL lives directly in services, a boundary between the domain model and persistence is implicitly present. The question is who drew it: you or the technology.

**In short:** JPA draws it through entities, Spring Data JDBC through aggregates, and JdbcClient leaves the decision to you, along with responsibility for the result.

## 1. The Interview Answer

The question usually goes like this: "When should you choose JPA, and when JDBC?"

The answer is often built around API features: dirty checking, lazy loading, CRUD repositories, relationship management, and writing SQL.

I would start with a different question: is the application ready to accept the persistence model the technology offers, or does this boundary need to be defined independently?

**Short answer: JPA is justified when managing a graph of entities and its state genuinely simplifies the application.** We will look at the details below.

## 2. JPA: Two Ways to Organize the Domain Model

JPA provides object-relational mapping (ORM), built on entities, their identity, relationships, and state management.

A JPA entity has specific structural requirements:

- the class must not be `final`;
- a public or protected no-argument constructor is required;
- methods and persistent instance variables must not be `final`;
- Java records cannot be used as entities.

These requirements are established by Jakarta Persistence 3.2, § 2.1 [1]. Starting with version 3.2, records are allowed as `@Embeddable` (§ 2.7) [2], but not as `@Entity`.

ORM providers such as Hibernate use a persistence context, object state management, proxies, and lazy-loading mechanisms. This affects application behavior, especially when entities are used outside the data access layer.

When designing an application with JPA, the developer effectively chooses between two options.

### Option 1. The JPA Entity Is Also the Domain Model

```text
Database
    ↕ ORM mapping
JPA Entity = Domain Model
    ↕
Application
```

This approach avoids an additional Entity → Domain Model conversion.

However, the domain model takes on the characteristics of the ORM:

- the class structure must account for JPA requirements;
- relationships and data loading depend on the provider's mechanisms;
- lazy loading can affect the execution of services and controllers;
- changes to managed objects are detected through dirty checking;
- the moment changes are synchronized with the database depends on flush;
- business behavior and the ORM contract end up in the same classes.

For example, accessing a lazy collection can trigger an additional SQL query. If an Entity is passed between application layers, the specifics of its state and loading can become part of the behavior of those layers.

There is no additional Entity → Domain mapping here, but the domain model accepts the constraints of the ORM.

### Option 2. The Domain Model Is Separate from the JPA Entity

Another approach is to create separate models for business logic and data storage.

```text
Database
    ↕ ORM mapping
JPA Entity
    ↕ Entity ↔ Domain mapping
Domain Model
    ↕
Application
```

Two transformations appear here:

1. Relational data is mapped to a JPA Entity by the ORM.
2. The JPA Entity is converted to a Domain Model by the application.

On write, the reverse transformation is performed.

This architecture isolates the domain from the persistence context, ORM annotations, lazy collections, and other entity management specifics.

However, the price of this isolation is additional mapping code and the maintenance of two models.

Thus, JPA allows both the direct use of an Entity as the Domain Model and an additional boundary between them.

## 3. Spring Data JDBC: An Aggregate-Based Persistence Model

Spring Data JDBC offers a different model of persistence organization.

Its central abstraction is the aggregate root.

An aggregate is a group of objects whose consistency is ensured within atomic changes. A repository is organized around the aggregate root.

For example:

```text
Order
 └── OrderItems
```

An aggregate defines a consistency boundary; it does not merely reflect the structure of foreign keys between tables.

In Spring Data JDBC, all component entities that belong to an aggregate are treated as part of a single aggregate. References to other aggregates are represented separately, for example through `AggregateReference`, which stores the identifier of another aggregate [5].

Spring Data JDBC maps relational data directly to aggregate objects. It requires no JPA entities, persistence context, ORM proxies, or dirty checking. The aggregate itself can be the domain model.

```java
public interface OrderRepository
        extends CrudRepository<Order, UUID> {
}
```

However, the application accepts the aggregate persistence model.

In Spring Data Relational 4.1.2 (Spring Data JDBC), when an existing aggregate is saved:

1. The component entities that belong to the aggregate are deleted.
2. The aggregate root record is updated.
3. The child entities are inserted again.

This behavior is described in the Persisting Entities section [6].

It applies to entities inside the aggregate. References to independent aggregates via `AggregateReference` do not mean that those aggregates are included in the saved object.

If only one child record has changed, the persistence mechanism may recreate the remaining records of the aggregate. For models where child records frequently change independently, this can be excessive.

Spring Data JDBC also supports optimistic locking through `@Version` on the aggregate root [6]. On save, the version is checked and incremented automatically.

Thus, Spring Data JDBC maps domain objects directly, but assumes a particular organization of the domain and particular persistence semantics.

## 4. JdbcClient: The Application Defines the Data Access Model

`JdbcClient` is a fluent API on top of `JdbcTemplate` and `NamedParameterJdbcTemplate`. It provides execution of SQL queries and updates, parameter binding, and result mapping.

At the same time, JdbcClient does not introduce its own model of entities, aggregate roots, or lifecycle of persistent objects.

The developer independently defines the structure of SQL queries, write operations, read and write models, the conditions under which operations execute, and the rules for handling conflicts [7].

### The Boundary Between the Domain Model and Persistence

Without a dedicated data access layer, JPA still brings its model of entities, identity, and lifecycle into the application.

With JdbcClient, the developer decides how data is converted into objects and where the persistence boundary lies.

Consider restoring a domain object.

Suppose `Owner` is an immutable domain model with a private constructor and its own behavior:

```java
public final class Owner {

    private final UUID id;
    private final String name;

    private Owner(UUID id, String name) {
        if (name == null || name.isBlank()) {
            throw new IllegalArgumentException(
                    "Owner name must not be blank");
        }

        this.id = id;
        this.name = name.trim();
    }

    public static Owner restore(UUID id, String name) {
        return new Owner(id, name);
    }

    public Owner rename(String newName) {
        return new Owner(id, newName);
    }

    public UUID id() {
        return id;
    }

    public String name() {
        return name;
    }
}
```

This implementation of `Owner` does not meet the requirements for a JPA Entity: the class is declared `final`, it has no public or protected no-argument constructor, and with FIELD access its `final` fields cannot be persistent fields either.

JdbcClient allows such an object to be restored directly:

```java
public Optional<Owner> findById(UUID id) {
    return jdbc.sql("""
            SELECT id, name
            FROM owners
            WHERE id = :id
            """)
            .param("id", id)
            .query((rs, rowNum) -> Owner.restore(
                    rs.getObject("id", UUID.class),
                    rs.getString("name")))
            .optional();
}
```

Here the `RowMapper` restores the domain object directly. No intermediate ORM entity is required.

**JdbcClient does not eliminate mapping. It allows you to avoid introducing a separate ORM entity model, which, with a JPA Entity and a Domain Model kept separate, would require an additional conversion.**

Spring Data JDBC also allows data to be mapped directly to aggregate objects. Its difference is that the application accepts the aggregate persistence model.

The retrieved object can be modified through a domain method and saved with an explicit SQL operation.

The `rename()` method creates a new `Owner` object, preserving the original identifier. Therefore, the SQL operation changes only the name:

```java
Owner renamed = owner.rename(newName);

int updated = jdbc.sql("""
        UPDATE owners
        SET name = :name
        WHERE id = :id
        """)
        .param("id", renamed.id())
        .param("name", renamed.name())
        .update();
```

This simplified example has no version check: with concurrent renames, the last update may overwrite the previous one (last-write-wins). If this behavior is unacceptable, a version check can be included directly in the SQL, as shown below.

### Write Operation Semantics

The difference is even more evident when implementing domain operations.

Suppose an order can be moved to the `PAID` state only if it is in the `NEW` state.

```java
public boolean markPaid(UUID orderId) {
    return jdbc.sql("""
            UPDATE orders
            SET status = 'PAID',
                paid_at = CURRENT_TIMESTAMP
            WHERE id = :id
              AND status = 'NEW'
            """)
            .param("id", orderId)
            .update() == 1;
}
```

The condition for a valid state transition is included directly in the SQL operation.

The database checks the condition and performs the change within a single statement. This operation requires no prior read of the entity, no subsequent modification of its state, and no dirty checking.

The number of affected rows determines the result: `1` means the transition was performed; `0` means the order does not exist or is no longer in the `NEW` state.

In JPA, the same operation can be implemented by loading the entity and changing its state, relying on dirty checking. Between the read and the write, another transaction may change the order's state. `@Version` makes it possible to detect such a conflict when changes are synchronized [3], and pessimistic locking limits concurrent access. With a conditional `UPDATE`, the check of transition validity and the change are performed by a single SQL statement, without a preceding SELECT. The difference is not whether the operation can be performed, but where the transition condition lives: in application code or directly in the SQL operation.

For example, in PostgreSQL at the `READ COMMITTED` isolation level, a second transaction attempting to modify a row locked by the first waits for the first to finish. If the first transaction commits the `NEW` → `PAID` transition, PostgreSQL re-evaluates the `WHERE` condition against the updated version of the row. If the condition no longer holds, the second operation updates zero rows [8].

JPA can also express such a conditional change through bulk JPQL or native SQL. However, bulk JPQL bypasses the automatic optimistic locking check: the version must be checked and incremented manually. In addition, the persistence context is not synchronized with the result of a bulk update, so entities that are already loaded may contain stale state.

This is described in Jakarta Persistence 3.2, § 4.11 — Bulk Update and Delete Operations [4].

Thus, conditional SQL allows a domain rule to be expressed directly in the storage operation. In JPA, the same semantics can live in entity modification code or in a bulk query.

### The Price of Freedom

When using JdbcClient, the developer is personally responsible for the correctness of the SQL, conflict handling, transaction consistency, and verification of operation results.

Optimistic locking via a version must be implemented directly in SQL:

```sql
UPDATE owners
SET name = :name,
    version = version + 1
WHERE id = :id
  AND version = :expectedVersion
```

If the UPDATE changed no rows, this may mean either that the record does not exist or that the version does not match. If necessary, the application must distinguish between these cases with an additional check.

JPA and Spring Data JDBC provide ready-made `@Version` mechanisms. JdbcClient leaves their implementation to the developer.

Transaction boundaries can still be managed by the standard Spring infrastructure through `@Transactional`.

## 5. Comparison of Architectural Consequences

| Criterion | JdbcClient | Spring Data JDBC | JPA |
|---|---|---|---|
| Persistence model | SQL and mapping | Aggregates | ORM entities |
| Unit of persistence | Application SQL operation | Aggregate root | Managed entities |
| Mapping to the domain model | Directly via RowMapper | Directly into the aggregate | Entity or Entity → Domain |
| Change tracking | Explicit write operations | No dirty checking; the aggregate is saved as a whole | Persistence context and dirty checking |
| Optimistic locking | Explicit implementation in SQL | `@Version` on the aggregate root | `@Version` on the entity |

## 6. When Is an ORM Truly Justified?

An ORM is justified when the domain model is naturally representable as a graph of related objects with identity, relationships, and behavior, and managing the state of that graph simplifies the implementation of write operations.

This fit can be assessed using three criteria.

**1. The domain genuinely corresponds to an object model.**

The relationships and behavior of objects reflect business logic, and the object graph is the natural representation of the domain rather than a technical wrapper over tables.

**2. The team is ready to accept the JPA Entity as the domain model, or to knowingly pay for the additional conversion between models.**

When an Entity is used directly, the domain model accepts the constraints of the ORM. When the models are separated, additional conversions appear.

**3. Unit of Work and dirty checking genuinely simplify write operations.**

Automatic change tracking should reduce the complexity of the application, not merely replace explicit SQL operations.

If the application requires strict isolation of the domain model from persistence, and the read and write models differ significantly, an ORM can add work: an additional entity model appears, along with conversions between it and the application model.

Spring Data JDBC is a good fit when aggregate boundaries match the requirements of the domain and are the natural unit of persistence.

JdbcClient is especially interesting when the application needs to define data operations itself, does not need ORM mechanisms, or does not want to accept the aggregate persistence model.

The SQL-first approach is especially convenient with CQRS: read models can be designed directly for specific query scenarios and do not have to repeat the structure of the aggregates used for writing. JPA also supports DTO projections, so the advantage here lies in direct control over the SQL and the shape of the result.

## 7. Architectural Conclusion

JPA offers a model of managed entities. The developer can use an Entity directly as the domain model or separate it from the domain with an additional conversion layer.

Spring Data JDBC maps data directly to aggregate objects, but assumes the aggregate model and the corresponding persistence semantics.

JdbcClient leaves the definition of SQL, data models, and write operations to the developer. It allows query results to be converted directly into domain objects without introducing a mandatory intermediate ORM model.

**Even without a dedicated data access layer, an architectural boundary exists. With JPA, it runs through the entity model and the mechanisms for managing entity state. With JdbcClient, its position and the way it is implemented are determined by the application itself.**

Choosing a data access technology is a decision about where the boundary between the domain model and the persistence model lies and who is responsible for organizing it.

**The tool follows from the architectural decision.**

---

## Sources

- [1] Jakarta Persistence 3.2 Specification, § 2.1 The Entity Class
- [2] Jakarta Persistence 3.2 Specification, § 2.7 Embeddable Classes
- [3] Jakarta Persistence 3.2 Specification, § 3.5.1–3.5.2 Optimistic Locking
- [4] Jakarta Persistence 3.2 Specification, § 4.11 Bulk Update and Delete Operations
- [5] Spring Data Relational 4.1.2, Domain Driven Design and Relational Databases
- [6] Spring Data Relational 4.1.2, Persisting Entities and Optimistic Locking
- [7] Spring Framework 7.0 Reference, JDBC Core Classes and JdbcClient
- [8] PostgreSQL 18 Documentation, § 13.2.1 Read Committed Isolation Level
