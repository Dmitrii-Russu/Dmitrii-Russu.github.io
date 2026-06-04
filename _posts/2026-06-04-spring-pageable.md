---
title: Why I stopped exposing Spring Pageable in application layer contracts
date: 2026-06-04 12:00:00 +0200
categories: [Architecture, Persistence]
tags: [java, spring-boot, pagination, ddd, hexagonal-architecture, jdbc]
---

The use-case contract should describe what the application needs — not the API of the persistence framework it happens to use.

Spring Data pagination works well in CRUD applications. For many projects, Pageable in an application service is a reasonable compromise. The problem starts when Pageable becomes part of a public use-case contract:

```java
// use case depends on Spring Data
interface OwnerUseCase {
    Page<OwnerView> execute(Pageable pageable);
}
```

Now every caller — controller, scheduler, message listener — must know about Spring Data API. The application layer contract starts describing persistence framework details instead of application needs.

---

## Framework-independent pagination model

PageQuery, PageResult and SortRequest live in the application layer. No Spring Data dependencies — plain Java records:

```java
public record PageQuery(int page, int size, List<SortRequest> sort) {
    public static PageQuery of(int page, int size) {
        return new PageQuery(page, size, List.of());
    }
    public static PageQuery of(int page, int size, List<SortRequest> sort) {
        return new PageQuery(page, size, sort);
    }
}

public record PageResult<T>(List<T> content, int page, int size, long total) {}
```

SortRequest uses domain field names — no SQL, no infrastructure details:

```java
public record SortRequest(String field, Direction direction) {
    public enum Direction { ASC, DESC }

    private static final Set<String> ALLOWED_FIELDS = Set.of("id", "name");

    public SortRequest {
        if (!ALLOWED_FIELDS.contains(field))
            throw new IllegalArgumentException("Invalid sort field: " + field);
    }

    public static SortRequest asc(String field)  { return new SortRequest(field, Direction.ASC); }
    public static SortRequest desc(String field) { return new SortRequest(field, Direction.DESC); }
}
```

The caller uses domain language — no SQL aliases leak into the application layer:

```java
// single field
PageQuery.of(0, 10, List.of(SortRequest.asc("name")));

// multiple fields
PageQuery.of(0, 10, List.of(
    SortRequest.asc("name"),
    SortRequest.desc("id")
));
```

The read repository returns a read model directly — a CQRS approach where the repository intentionally returns a projection rather than an aggregate:

```java
public interface OwnerReadRepository {
    PageResult<OwnerView> findAllFlat(PageQuery request);
}
```

---

## Infrastructure adapter

At the infrastructure boundary, PageQuery is converted to SQL parameters. SQL aliases live here and nowhere else. The application layer never sees Pageable, SQL aliases, or a raw ResultSet:

```java
private static final Map<String, String> FIELD_MAP = Map.of(
    "id",   "o.id",
    "name", "o.name"
);

private static final String SELECT_PAGE = """
    SELECT o.id   AS owner_id,
           o.name AS owner_name
    FROM owners o
    ORDER BY %s
    LIMIT :limit OFFSET :offset
    """;

@Repository
public class JdbcOwnerReadRepository implements OwnerReadRepository {

    private String buildOrderBy(PageQuery request) {
        if (request.sort().isEmpty()) return "o.id ASC";
        return request.sort().stream()
                .map(s -> FIELD_MAP.get(s.field()) + " " + s.direction().name())
                .collect(Collectors.joining(", "));
    }

    @Override
    public PageResult<OwnerView> findAllFlat(PageQuery request) {
        int offset = request.page() * request.size();

        // orderBy contains only fields from FIELD_MAP + enum direction — no SQL injection possible
        String orderBy = buildOrderBy(request);

        List<OwnerView> content = jdbc.sql(SELECT_PAGE.formatted(orderBy))
                .param("limit",  request.size())
                .param("offset", offset)
                .query(OwnerProjection.class)
                .stream()
                .map(ViewMapper::toView)
                .toList();

        long total = jdbc.sql(COUNT_ALL).query(Long.class).single();

        return new PageResult<>(content, request.page(), request.size(), total);
    }
}
```

`OwnerProjection` and `ViewMapper` are package-private — they never leave the infrastructure package. In many cases, changing the persistence technology affects only the adapter while the application layer remains unchanged.

---

## Testability

The application service is tested without a Spring context:

```java
@Test
void returns_paginated_owners() {
    var request = PageQuery.of(0, 10);
    var expected = new PageResult<>(List.of(ownerView), 0, 10, 1L);

    when(repository.findAllFlat(request)).thenReturn(expected);

    var result = service.getOwners(request);

    assertThat(result.content().get(0).name()).isEqualTo("jack1");
    verify(repository).findAllFlat(request);
}
```

No Spring context, no Pageable, no Spring Data dependencies in the test.

---

## Trade-off

This approach introduces additional abstractions. In a simple CRUD application it may be unnecessary — and that is a valid choice.

In systems where architectural boundaries matter, an explicit pagination model keeps the application layer independent of the persistence framework. Worth noting: different bounded contexts may have different pagination semantics — cursor-based, keyset. PageQuery should not be shared across the whole system in that case.

One more thing — yes, PageQuery looks similar to Pageable. That is intentional. The goal is not to invent a new pagination model. The goal is to own the application contract instead of exposing Spring Data types.

---

Working examples:

- Flat pagination — PageQuery, PageResult, pure SQL:
[persistence-flat-pagination-jdbc](https://github.com/java-backend-architecture/persistence-flat-pagination-jdbc)

- Pagination with multi-field sorting and whitelist protection:
[persistence-flat-pagination-sorting-jdbc](https://github.com/java-backend-architecture/persistence-flat-pagination-sorting-jdbc)

- Pagination over an object graph (owner → pets → visits):
[persistence-graph-pagination-jdbc](https://github.com/java-backend-architecture/persistence-graph-pagination-jdbc)
