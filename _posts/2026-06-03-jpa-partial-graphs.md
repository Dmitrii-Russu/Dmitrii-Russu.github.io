---
title: JPA handles full graphs well. What about partial ones?
date: 2026-06-03 12:00:00 +0200
categories: [Architecture, Persistence]
tags: [jdbc, jooq, jpa, blaze, java]
---

JPA does not provide a first-class model for partial nested graphs as a concept. For that you need JDBC (manual assembly), jOOQ (MULTISET), or Blaze Persistence (Entity Views).

Most discussions around persistence start with the wrong problem. We compare frameworks, SQL tools, ORMs… But the real problem is simpler and more fundamental:

A relational JOIN result is flat by default.
Applications need nested object graphs or specialized shapes of data.

## The relational reality

Consider a simple model:

```
Owner → Pet → Visit
```

In a relational database — three tables with foreign key relationships. After a JOIN:

```
owner_id | owner_name | pet_id | pet_name | visit_id | visit_date
1        | jack       | 1      | buddy1   | 1        | 2026-01-10
1        | jack       | 1      | buddy1   | 2        | 2026-02-15
1        | jack       | 2      | buddy2   | 3        | 2026-03-01
3        | bob        | 4      | hew1     | null     | null
```

Row duplication, Cartesian explosion, structural mismatch. This is not a framework bug — it is a fundamental impedance mismatch between the relational and object models.

## EntityGraph: controlling loading, not data shape

```java
@EntityGraph(attributePaths = { "pets", "pets.visits" })
Owner findById(String id);
```

One query, no N+1, full graph loaded.

But there is an important constraint to understand:

EntityGraph controls identity graph loading — which associations to fetch and when.
Projection controls the shape of the result — what form the data should take.

These are different responsibilities. EntityGraph answers the question "how do I load the full graph?" — but not "what if I need a different shape?"

For example:

```java
record OwnerListView(
    String id,
    String name,
    List<String> pets  // names only, not entities
)
```

Here we need neither Visit entities nor the full Pet model. This is where EntityGraph ends.

## The real problem: who builds the graph?

Who is responsible for assembling the nested object graph?

This is the axis of the architectural choice:

```
JDBC   → the application builds the graph
jOOQ   → the SQL abstraction builds the graph
Blaze  → the ORM builds the graph
```

Different tools solve the same problem at different architectural layers.

## Approach 1 — JDBC: the application builds the graph

SQL returns flat rows. The graph is assembled manually in a single pass — the accumulator pattern:

```java
static List<OwnerProjection> extractWithGraph(ResultSet rs) throws SQLException {
    Map<String, OwnerProjection> owners = new LinkedHashMap<>();

    while (rs.next()) {
        String ownerId = rs.getString("owner_id");

        OwnerProjection owner = owners.computeIfAbsent(
            ownerId, id -> OwnerProjection.of(id, rs.getString("owner_name"))
        );

        String petId = rs.getString("pet_id");
        if (petId != null) {
            PetProjection pet = owner.getOrCreatePet(petId, rs.getString("pet_name"));

            String visitId = rs.getString("visit_id");
            if (visitId != null) {
                pet.getOrCreateVisit(visitId, rs.getDate("visit_date").toLocalDate());
            }
        }
    }
    return List.copyOf(owners.values());
}
```

`getOrCreatePet` and `getOrCreateVisit` are `computeIfAbsent` calls inside each accumulator:

```java
class OwnerProjection {
    private final Map<String, PetProjection> pets = new LinkedHashMap<>();

    PetProjection getOrCreatePet(String id, String name) {
        return pets.computeIfAbsent(id, k -> PetProjection.of(id, name, this.id));
    }
}

class PetProjection {
    private final Map<String, VisitProjection> visits = new LinkedHashMap<>();

    VisitProjection getOrCreateVisit(String id, LocalDate date) {
        return visits.computeIfAbsent(id, k -> VisitProjection.of(id, date, this.id));
    }
}
```

`OwnerProjection`, `PetProjection`, `VisitProjection` — mutable accumulators hidden behind package-private visibility. Only immutable record types are exposed to the outside through `ViewMapper`.

Result of a single pass:

```
Owner(jack)
  └── Pet(buddy1)
      ├── Visit(2026-01-10)
      └── Visit(2026-02-15)
  └── Pet(buddy2)
      └── Visit(2026-03-01)
```

+ full control over SQL
+ single query, no N+1
+ no external dependencies
- intermediate classes required at every graph level
- manual deduplication via LinkedHashMap

[persistence-graph-extraction-jdbc](https://github.com/java-backend-architecture/persistence-graph-extraction-jdbc)

## Approach 2 — jOOQ MULTISET: the SQL abstraction builds the graph

jOOQ shifts graph assembly into the SQL layer. The query returns a nested structure directly:

```java
dsl.select(
    OWNERS.ID,
    OWNERS.NAME,
    multiset(
        select(
            PETS.ID, PETS.NAME, PETS.OWNER_ID,
            multiset(
                select(VISITS.ID, VISITS.DATE, VISITS.PET_ID)
                    .from(VISITS)
                    .where(VISITS.PET_ID.eq(PETS.ID))
            ).convertFrom(r -> r.map(Records.mapping(VisitView::new)))
        )
        .from(PETS)
        .where(PETS.OWNER_ID.eq(OWNERS.ID))
    ).convertFrom(r -> r.map(Records.mapping(PetView::new)))
)
.from(OWNERS)
.fetch(Records.mapping(OwnerView::new));
```

No intermediate projections, no manual deduplication. `Records.mapping` maps directly into record types.

For a flat list — even more concise:

```java
dsl.select(
    OWNERS.ID,
    OWNERS.NAME,
    multiset(
        select(PETS.NAME)
            .from(PETS)
            .where(PETS.OWNER_ID.eq(OWNERS.ID))
    ).convertFrom(r -> r.map(rec -> rec.get(PETS.NAME)))
)
.from(OWNERS)
.fetch(Records.mapping(OwnerListView::new));
```

+ no manual graph assembly
+ type safety at the table and column level
+ minimal supporting code
- requires jOOQ code generation
- field order in SELECT must match constructor parameter order — enforced at runtime only
- MULTISET in production requires PostgreSQL or another database with full support; H2 is used in the examples for test purposes only to keep local setup simple

[persistence-graph-extraction-jooq](https://github.com/java-backend-architecture/persistence-graph-extraction-jooq)

## Approach 3 — JPA + Blaze Persistence: the ORM builds the graph

Blaze introduces a declarative model through Entity Views:

```java
@EntityView(OwnerEntity.class)
interface OwnerEntityView {
    @IdMapping String getId();
    String getName();
    List<? extends PetEntityView> getPets();
}

@EntityView(PetEntity.class)
interface PetEntityView {
    @IdMapping String getId();
    String getName();
    @Mapping("owner.id") String getOwnerId();
    List<? extends VisitEntityView> getVisits();
}
```

The data shape is declared — Blaze generates optimized queries automatically. But this is not magic without configuration. Each view must be registered explicitly:

```java
@Bean
EntityViewManager entityViewManager(CriteriaBuilderFactory cbf) {
    EntityViewConfiguration config = EntityViews.createDefaultConfiguration();
    config.addEntityView(OwnerEntityView.class);
    config.addEntityView(PetEntityView.class);
    config.addEntityView(VisitEntityView.class);
    config.addEntityView(OwnerListEntityView.class);
    return config.createEntityViewManager(cbf);
}
```

Mapping to the application read model is still handled by `ViewMapper`:

```java
var setting = EntityViewSetting.create(OwnerEntityView.class);
var cb = cbf.create(em, OwnerEntity.class).where("id").eq(id);

return evm.applySetting(setting, cb)
    .getResultList()
    .stream()
    .findFirst()
    .map(ViewMapper::toView);
```

By default Blaze generates a single optimized query. When an EntityView contains multiple independent collections, it may issue a separate query for each. For full predicate type safety use the metamodel: `where(OwnerEntity_.id).eq(id)`

+ declarative projections
+ optimized queries generated automatically
+ no manual deduplication
- generated SQL is hidden — harder to reason about query shape and production performance
- complex graphs can produce query explosion — requires explicit verification in production
- requires BlazeConfig and registration of every view
- additional dependency

[persistence-graph-extraction-jpa](https://github.com/java-backend-architecture/persistence-graph-extraction-jpa)

## Comparison

| Criterion            | JDBC         | jOOQ          | JPA + Blaze       |
|----------------------|--------------|---------------|-------------------|
| SQL result           | flat rows    | nested        | per-view optimized|
| Graph built by       | application  | SQL layer     | ORM layer         |
| Queries to DB        | 1 (JOIN)     | 1 (MULTISET)  | 1 by default      |
| Intermediate classes | Projection   | none          | @EntityView       |
| SQL visible          | yes          | yes           | hidden            |
| Type safety          | none         | yes (codegen) | with metamodel    |
| Setup                | minimal      | codegen       | BlazeConfig       |

## When to use what

**JDBC** — when you need full control over SQL, want no extra dependencies, or are working with non-standard data shapes. The cost is intermediate classes and manual deduplication.

**jOOQ** — when you want type safety at the SQL level and are comfortable with code generation. The best balance between control and conciseness for complex multi-level projections.

**JPA + Blaze** — when you are already on JPA and need declarative projections with minimal manual mapping. Keep in mind: the SQL is hidden, and production behavior requires explicit verification.

## The key insight

Persistence is not about how you load data.
It is about who owns the responsibility for shaping it.

```
JDBC  → the application controls every step
jOOQ  → the SQL layer takes over graph assembly
Blaze → the ORM declares the shape and generates the query
```

Choosing a tool is choosing where graph construction responsibility lives. And that decision shapes the architecture of your entire persistence layer.

---

Fetching data is easy. Shaping it correctly — that is the real engineering challenge.
