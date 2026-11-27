---
layout: post
title: "Integrity Violation Handling Policy in Spring Boot + PostgreSQL: 409 for Uniqueness Violations, 500 for Schema Mismatch"
date: 2026-11-27
description: "Classifying database constraint violations by who consumes the error: 409 for UNIQUE, 500 with diagnostics for CHECK and NOT NULL that duplicate input validation. Uses getConstraint() and getColumn() instead of parsing exception messages."
tags: [java, spring, spring-boot, postgresql, jdbc, jpa, error-handling, validation]
---

**Diagnostics via `getConstraint()` and `getColumn()` instead of parsing exception messages**

Full code and dependency versions: <https://github.com/java-backend-architecture/integrity-violation-direct-handling>

The task: when a uniqueness constraint (`UNIQUE` or `PRIMARY KEY`) is violated, return a clear HTTP 409 to the client. When a `CHECK` or `NOT NULL` constraint that duplicates input validation is violated, return HTTP 500 and write diagnostic information to the log for the developer. No parsing of exception messages.

## 1. Why Both Layers of Protection Are Needed

Client-side validation can be bypassed, so the server validates input on its own. Server-side validation does not guarantee integrity either: the database can be written to bypassing the application, and check-then-insert is vulnerable to concurrent requests. Database constraints are therefore mandatory. Together, the two layers provide defense in depth.

## 2. Consumers of Error Information

Two parties learn about an error: the client (to fix their input) and the developer (to debug). For this task, we are interested in three types of database constraints: `UNIQUE`, `CHECK`, and `NOT NULL`.

| Situation | Client response | Log |
|---|---|---|
| Jakarta Validation error | 400, the request never reaches the database | not needed |
| Uniqueness violation | 409 + field label, if the constraint is known | info, expected situation |
| `CHECK` or `NOT NULL` violation duplicating input validation | 500 Internal error | error, `reason=validation_schema_mismatch` + constraint or column |
| Other integrity violation (`id = NULL`, standalone business `CHECK`, FK) | 500 Internal error | error, `reason=integrity_violation` + constraint or column |
| Other database access failure (not an integrity violation) | 500 Database failure | error + stack trace |
| Unexpected error | 500 Internal error | error + stack trace |

In this example, any uniqueness violation (`UNIQUE` or `PRIMARY KEY`) is treated as a client conflict and results in 409. For a constraint without a label (for example, `PRIMARY KEY`), the response is generic: `Resource already exists`.

Here, `CHECK` and `NOT NULL` duplicate the structural constraints of input validation (length, required fields, normalization), not all Jakarta rules: for example, `@Email` is not repeated in the schema. Therefore, a violation of such a constraint after validation has passed indicates a mismatch between the code and the schema: the client cannot fix anything, and the details are needed by the developer. Standalone business `CHECK` constraints that are not reflected in the DTO require a different policy.

FK violations (`23503`) are not covered in this example: the schema has a single table. In a real API, a reference from a client request to a nonexistent entity may be handled as 404, 409, or 422, depending on the contract, and this requires a separate policy. Until one exists, such a violation results in 500 as an "other integrity violation".

## 3. Validation and Schema

```java
public record UserRequest(
        @NotBlank @Size(min = 2, max = 5)
        String name,
        @NotBlank @Email @Size(min = 11, max = 12)
        String email
) {
    public UserRequest {
        name = name == null ? null : name.trim();
        email = email == null ? null : email.trim().toLowerCase(Locale.ROOT);
    }
}
```

The limits are deliberately small to make the tests easier to follow.

```sql
create table if not exists users (
    id    uuid default gen_random_uuid(),
    name  text not null,
    email text not null,

    constraint pk_users primary key (id),
    constraint uq_users_name unique (name),
    constraint uq_users_email unique (email),

    constraint ck_users_name_no_outer_whitespace check (name !~ '^\s|\s$'),
    constraint ck_users_email_normalized check (email = lower(email) and email !~ '\s'),
    constraint ck_users_name_length check (length(name) between 2 and 5),
    constraint ck_users_email_length check (length(email) between 11 and 12)
);
```

All `UNIQUE` and `CHECK` constraints are named. For `UNIQUE`, the constraint name identifies the field that caused the conflict; for `CHECK`, it shows which rule was violated. In this example, each constraint applies to a single column.

`CHECK` does not transform anything. Normalization is done by the repository (`trim`, `lower`), while the schema guarantees that the chosen normalization invariants hold: otherwise, another write path would store `JACK@MAIL.RU` alongside `jack@mail.ru`. Why a regex rather than `trim()`, why `Locale.ROOT`, and what to do about collations are described in the project's README.

**`NOT NULL`: the column name instead of the constraint name.** For a native `NOT NULL` (SQLState `23502`), PostgreSQL reports the column name rather than the constraint name: `getConstraint()` returns `null`, and `getColumn()` returns the column name. That is why `NOT NULL` stays native, and a `CHECK` of the form `(x is not null)` is not needed just to get a name. For `UNIQUE` and `CHECK`, the error is identified by the constraint name; for `NOT NULL`, by the column name.

## 4. Why SQLState Alone Is Not Enough

`PRIMARY KEY` also implies `NOT NULL`: a direct `insert into users (id, ...) values (null, ...)` produces `23502` on `id`, even though `id` is not part of the DTO (an explicit `NULL` overrides `DEFAULT`). A standalone business `CHECK` also produces `23514`. Therefore, only violations of known rules are treated as a mismatch: for `CHECK`, by constraint name; for `NOT NULL`, by column name. Some of the values in these sets are unreachable via HTTP because Jakarta catches them earlier; the sets protect against other write paths and against DTO changes.

`validation_schema_mismatch` denotes the category of known constraints that duplicate input validation. If a request arrived over HTTP and passed Jakarta Validation, such a violation indicates a mismatch between validation and the schema.

## 5. Extracting the Constraint and Column Names

Many examples parse the name out of the exception message. This is fragile: the message format depends on the server version and locale. The driver offers a structured alternative: `PSQLException.getServerErrorMessage()` with its `getConstraint()` and `getColumn()` methods.

The code is split into three parts:

- **`ErrorDetailsExtractor`** extracts the SQLState, constraint name, and column name from the exception. It is the only place that knows about `PSQLException`.
- **`IntegrityPolicy`** knows the SQLState codes: it detects duplicates, classifies violations, and stores the labels for 409 responses. It consists of static functions and constants, with no Spring and no driver dependency.
- **`WebExceptionHandler`** turns the result into an HTTP response and a log entry. It does not import driver classes and contains no SQLState codes.

```java
public final class ErrorDetailsExtractor {

    private static final String UNKNOWN = "unknown";

    private ErrorDetailsExtractor() {}

    public static Details extract(DataAccessException e) {
        var cause = NestedExceptionUtils.getMostSpecificCause(e);

        var serverError = cause instanceof PSQLException pe
                ? pe.getServerErrorMessage()
                : null;

        var sqlState = cause instanceof SQLException se
                ? orUnknown(se.getSQLState())
                : UNKNOWN;

        var constraint = serverError != null
                ? orUnknown(serverError.getConstraint())
                : UNKNOWN;

        var column = serverError != null
                ? orUnknown(serverError.getColumn())
                : UNKNOWN;

        return new Details(sqlState, constraint, column);
    }

    private static String orUnknown(String value) {
        return value != null ? value : UNKNOWN;
    }

    public record Details(String sqlState, String constraint, String column) {}
}
```

`"unknown"` is returned if PostgreSQL did not send a value or if the root cause is not a driver exception.

The classification is tied to the `users` table. In an application with several tables, column names and `CHECK` constraint names may repeat, so the table name (`getTable()`) should be added to `Details`.

```java
public final class IntegrityPolicy {

    // A new uniqueness constraint that needs its own label
    // must be added here, otherwise the client will get "Resource already exists"
    private static final Map<String, String> CONSTRAINT_LABELS = Map.of(
            "uq_users_name", "Name",
            "uq_users_email", "Email"
    );

    // Constraints and columns that duplicate structural rules
    // and input normalization.
    // Some of them are unreachable via HTTP (Jakarta catches them earlier).
    // The set protects against other write paths and DTO changes.
    private static final Set<String> VALIDATION_CHECKS = Set.of(
            "ck_users_name_no_outer_whitespace",
            "ck_users_email_normalized",
            "ck_users_name_length",
            "ck_users_email_length"
    );

    private static final Set<String> VALIDATED_COLUMNS = Set.of("name", "email");

    private IntegrityPolicy() {}

    public static String duplicateLabel(String constraint) {
        return CONSTRAINT_LABELS.getOrDefault(constraint, "Resource");
    }

    // 23505 = unique_violation
    public static boolean isDuplicate(String sqlState) {
        return "23505".equals(sqlState);
    }

    // 23514 = check_violation, 23502 = not_null_violation.
    // Only violations of rules that duplicate input validation
    // are treated as a mismatch.
    public static String reason(String state, String constraint, String column) {
        return switch (state) {
            case "23514" -> VALIDATION_CHECKS.contains(constraint)
                    ? "validation_schema_mismatch" : "integrity_violation";
            case "23502" -> VALIDATED_COLUMNS.contains(column)
                    ? "validation_schema_mismatch" : "integrity_violation";
            default -> "integrity_violation";
        };
    }
}
```

**The handler.** The decision to return 409 is based on SQLState `23505` (via `IntegrityPolicy.isDuplicate`), not on the exception type. Which subclass of `DataIntegrityViolationException` is thrown (`DuplicateKeyException` or the generic type) depends on the data access layer and configuration, so a single handler for the base type covers both `JdbcClient` and JPA.

```java
@Slf4j
@RestControllerAdvice
public class WebExceptionHandler extends ResponseEntityExceptionHandler {

    @ExceptionHandler(DataIntegrityViolationException.class)
    public ProblemDetail integrity(DataIntegrityViolationException e, HttpServletRequest req) {
        var details = ErrorDetailsExtractor.extract(e);

        var state = details.sqlState();
        var constraint = details.constraint();
        var column = details.column();

        if (IntegrityPolicy.isDuplicate(state)) {
            log.info(
                    "event=duplicate_key method={} path={} constraint={}",
                    req.getMethod(), req.getRequestURI(), constraint
            );

            var label = IntegrityPolicy.duplicateLabel(constraint);
            return ProblemDetail.forStatusAndDetail(
                    HttpStatus.CONFLICT, label + " already exists"
            );
        }

        var reason = IntegrityPolicy.reason(state, constraint, column);

        // The exception is not passed: PostgreSQL's DETAIL may contain row values
        log.error(
                "event=db_integrity_violation reason={} sqlstate={} method={} path={} constraint={} column={}",
                reason, state, req.getMethod(), req.getRequestURI(), constraint, column
        );

        return ProblemDetail.forStatusAndDetail(
                HttpStatus.INTERNAL_SERVER_ERROR, "Internal error"
        );
    }

    @ExceptionHandler(DataAccessException.class)
    ProblemDetail dataAccess(DataAccessException e, HttpServletRequest req) {
        log.error("event=db_failure method={} path={}", req.getMethod(), req.getRequestURI(), e);
        return ProblemDetail.forStatusAndDetail(
                HttpStatus.INTERNAL_SERVER_ERROR, "Database failure"
        );
    }

    @ExceptionHandler(Exception.class)
    ProblemDetail unexpected(Exception e, HttpServletRequest req) {
        log.error("event=unexpected_error method={} path={}", req.getMethod(), req.getRequestURI(), e);
        return ProblemDetail.forStatusAndDetail(
                HttpStatus.INTERNAL_SERVER_ERROR, "Internal error"
        );
    }
}
```

The repository (`JdbcClient`) wraps values in `trim(?)` and `lower(trim(?))` on insert; its code is on GitHub.

## 6. Limitations and Caveats

**Coupling to PostgreSQL.** The driver is isolated in `ErrorDetailsExtractor`, only `IntegrityPolicy` knows the SQLState codes, and the handler works with the classification result. However, the codes themselves and the semantics of `getConstraint()` and `getColumn()` are PostgreSQL-specific: if you change the DBMS, the extractor and the policy must be revisited, while the handler stays as is. For an application that is deliberately tied to PostgreSQL, this is an acceptable cost.

**Domain exceptions.** Here, the presentation layer consumes infrastructure diagnostics directly: the HTTP adapter itself uses this information. Translating to a domain exception is justified when there is a consumer in the core that makes a different decision on conflict (more details: <https://dmitrii-russu.github.io/posts/database-exception-to-domain-exception-spring/>). Here, there is no such consumer. If one appears, the repository catches `DataIntegrityViolationException`, calls the same `ErrorDetailsExtractor` and `IntegrityPolicy`, and throws a domain exception; the classes themselves and their tests do not change.

**Caveats:**

- Jakarta Validation exceptions do not need separate mapping: the advice extends `ResponseEntityExceptionHandler`, which returns a `ProblemDetail`.
- **PII in logs.** For integrity violations, the exception is not passed to the logger: its message may contain PostgreSQL's `DETAIL` with row values (email, name). For database failures and unexpected errors, the stack trace is kept.
- **Database failure vs Internal error.** Any other database access failure returns `Database failure`: the client and monitoring can see that the problem lies in the data layer, without any further details. If the API must not disclose even that, the response should be unified with `Internal error`.
- **Account enumeration.** `Email already exists` is convenient for the client, but in registration and password recovery flows it allows probing for the existence of accounts. The level of detail depends on the API's security requirements.

## 7. Tests

The main tests run against a real PostgreSQL (zonky embedded-postgres) and verify the actual constraints. Mocks are used only in `DbFailureTest`, for failures that cannot be reproduced on a real database. The rest of the test code is in the repository: <https://github.com/java-backend-architecture/integrity-violation-direct-handling>

| Check | What it confirms |
|---|---|
| `UNIQUE`, length, `NOT NULL` via the repository (`ConstraintNameTest`) | Constraint and column names are obtained from the driver; the test fails if a constraint is renamed in `schema.sql` |
| Direct `INSERT` bypassing the DTO and the repository | The schema itself rejects non-normalized data (`john\t`, `BOB@MAIL.RU`). Each case violates exactly one constraint, otherwise the result would depend on the order in which `CHECK` constraints are evaluated |
| `id = NULL` | `NOT NULL` on `id` yields `integrity_violation`, not `validation_schema_mismatch` |
| `IntegrityPolicyTest`, `ErrorDetailsExtractorTest` (plain JUnit) | Classification as pure functions; for a `SQLException` that is not a `PSQLException`, the extractor does not fail and returns `unknown` for the PostgreSQL fields |
| `WebExceptionHandlerTest` (no Spring context) | `UNIQUE` without a constraint name yields 409 with the generic label `Resource already exists`; FK yields 500 with `reason=integrity_violation` |
| JPA + Hibernate on a real database | A single handler works regardless of the exception subclass |
| Emoji over HTTP (below) | The Java–PostgreSQL mismatch is real |

**The mismatch is reachable.** Java counts UTF-16 code units, while PostgreSQL counts characters: `String.length()` returns 2 for an emoji, whereas PostgreSQL's `length()` returns 1. A name consisting of a single emoji passes `@Size(min = 2)` and fails on `ck_users_name_length`. The test goes through the whole stack: HTTP, validation, controller, repository, real database. The client receives a 500 without details, and the log contains everything needed.

```java
@Test
void validationAndSchemaMismatchGoesToIntegrityHandler(CapturedOutput output) throws Exception {
    post(json("\\uD83D\\uDE00", "emo@mail.ru")) // JSON escape of a surrogate pair
            .andExpect(status().isInternalServerError())
            .andExpect(jsonPath("$.detail").value("Internal error"))
            .andExpect(content().string(not(containsString("ck_users"))));
    assertThat(output)
            .contains("event=db_integrity_violation")
            .contains("reason=validation_schema_mismatch")
            .contains("constraint=ck_users_name_length");
}
```

## 8. Summary

A database constraint violation should be classified based on who consumes the error and what the constraint means: `UNIQUE` (SQLState `23505`) yields 409; `CHECK` and `NOT NULL` that duplicate validation yield 500 with `validation_schema_mismatch`; everything else yields 500 with `integrity_violation`. The decision does not depend on whether `JdbcClient` or JPA throws the exception, and the schema guarantees that the chosen normalization invariants hold, as demonstrated by the direct `INSERT` test.
