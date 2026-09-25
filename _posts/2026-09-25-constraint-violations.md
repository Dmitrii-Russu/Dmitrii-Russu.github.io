---
title: "Constraint Violations: Why the Database Still Gets the Final Say"
date: 2026-09-25 08:00:00 +0300
categories: [Architecture, Backend]
tags: [spring, postgresql, validation, data-integrity, rest-api]
---

In short: input validation is a filter, not a guarantee. It rejects malformed requests before they cost you anything, but it can't see what's already in the table. Uniqueness and integrity are facts about storage state, and only storage can enforce them — which means the database ends up checking some of the same things your code already checked, on purpose.

You insert a customer with an email that's already taken. The server answers with a 500. The client learns nothing: not what went wrong, not what to fix. Let's look at where each kind of error actually belongs, and how to turn a constraint violation into a response someone can act on.

## Jakarta Validation: checking at the boundary

The cost of bad data grows with how far it travels. An empty name caught in the controller costs one check. The same name that reaches the data access layer costs an open transaction, a round trip to the database, and an exception that has to be caught and unwrapped across the stack.

That's why Jakarta Validation sits at the entry point:

```java
public record CustomerRequest(
        @NotBlank @Size(min = 3, max = 20) String name,
        @NotBlank @Email String email) {}
```

Garbage doesn't get in, and the validator collects every format error in one pass rather than one at a time. The principle is simple: reject as early and as cheaply as possible.

## What the validator can't see

A validator checks an object in isolation. It has no idea what's already in the table, so uniqueness is out of reach for it. "This email is already registered" is a fact about the state of storage, not about the shape of a field — and no amount of annotations on a DTO will tell you that.

## SELECT before INSERT doesn't guarantee anything

The obvious next move is to check uniqueness in the service before writing:

```java
@Service
@RequiredArgsConstructor
public class CustomerService {
    private final CustomerRepository repository;

    public void create(Customer customer) {
        if (repository.existsByEmail(customer.email())) {
            throw new CustomerAlreadyExistsException(...);
        }
        repository.insert(customer);
    }
}
```

The object itself can't run this check — it doesn't see other objects, only itself. So the check moves outside it, into a service or a domain service behind a repository port. Wherever it lives, it's still a `SELECT` followed by an `INSERT`, and that pairing has two problems.

First, it's an extra query on every write. Second, and more serious: there's a window between the `SELECT` and the `INSERT`. Two concurrent requests can both see "email is free" and both try to insert. Under concurrent access the violation happens anyway, so the real guarantee has to live in the database — a pre-check just adds a query without removing the risk.

Which gives the actual division of labor: the database enforces uniqueness, and the application translates whatever error comes back.

## Naming the constraints

A constraint name is a contract between the schema and the code, so name them on purpose instead of letting Postgres generate one:

```sql
CREATE TABLE IF NOT EXISTS customer (
    id    INT  NOT NULL,
    name  TEXT NOT NULL,
    email TEXT NOT NULL,

    CONSTRAINT pk_customer PRIMARY KEY (id),
    CONSTRAINT uq_customer_email UNIQUE (email),
    CONSTRAINT chk_customer_name_min_length CHECK (LENGTH(TRIM(name)) >= 3),
    CONSTRAINT chk_customer_name_max_length CHECK (LENGTH(TRIM(name)) <= 20),
    CONSTRAINT chk_customer_email_format
        CHECK (email ~ '^[^@\s]+@[^@\s]+\.[^@\s]+$')
);
```

## An entity identifier is not an error vocabulary

Spring converts JDBC failures into `DataIntegrityViolationException`, and request validation failures into `MethodArgumentNotValidException`. Both are technical — the framework has no idea what your domain calls things, so the business interpretation is on you. Three small classes carry it, with no dependency on Spring or JDBC:

```java
public class CustomerNotFoundException extends RuntimeException {
    public CustomerNotFoundException(Integer id) {
        super("Customer with id " + id + " not found");
    }
}

public class CustomerAlreadyExistsException extends RuntimeException {
    public CustomerAlreadyExistsException(String message, Throwable cause) {
        super(message, cause);
    }
}

public class InvalidCustomerException extends RuntimeException {
    public InvalidCustomerException(String message, Throwable cause) {
        super(message, cause);
    }
}
```

The original exception is kept as the `cause`, so when the domain exception gets logged, the root cause is still visible in the stack trace.

## The translator

Postgres hands back the SQLState and the name of the constraint that was violated. That's enough to pick the right domain exception:

```java
@Component
public class CustomerConstraintViolationTranslator {

    public RuntimeException translate(DataIntegrityViolationException e, Customer customer) {
        if (!(e.getMostSpecificCause() instanceof PSQLException pgEx)) {
            return e;
        }
        var error = pgEx.getServerErrorMessage();
        if (error == null) {
            return e;
        }
        var constraint = error.getConstraint() == null ? "" : error.getConstraint();

        return switch (pgEx.getSQLState()) {
            case "23502" -> invalid("Field cannot be null: " + error.getColumn(), e);

            case "23505" -> switch (constraint) {
                case "pk_customer" -> exists(
                        "Customer with id " + customer.id() + " already exists", e);
                case "uq_customer_email" -> exists(
                        "Customer with this email is already registered", e);
                default -> e;
            };

            case "23514" -> switch (constraint) {
                case "chk_customer_name_min_length" -> invalid(
                        "Customer name is too short (minimum 3 characters)", e);
                case "chk_customer_name_max_length" -> invalid(
                        "Customer name is too long (maximum 20 characters)", e);
                case "chk_customer_email_format" -> invalid("Invalid email format", e);
                default -> e;
            };

            default -> e;
        };
    }
    // invalid(...) and exists(...) build the domain exceptions
}
```

An unrecognized violation comes back untouched and ends up as a 500. That's deliberate: a constraint someone forgot to map should be loud, not quietly disguised as a 400.

## Repository and handler

```java
@Repository
@RequiredArgsConstructor
public class CustomerInsertRepository {
    private final JdbcClient jdbc;
    private final CustomerConstraintViolationTranslator translator;

    public void insert(Customer customer) {
        var sql = "INSERT INTO customer (id, name, email) VALUES (:id, :name, :email)";
        try {
            jdbc.sql(sql).paramSource(customer).update();
        } catch (DataIntegrityViolationException e) {
            throw translator.translate(e, customer);
        }
    }
}
```

```java
@RestControllerAdvice
public class CustomerExceptionHandler {

    @ExceptionHandler(CustomerNotFoundException.class)
    public ProblemDetail handleNotFound(CustomerNotFoundException ex) {
        return ProblemDetail.forStatusAndDetail(HttpStatus.NOT_FOUND, ex.getMessage());
    }

    @ExceptionHandler(CustomerAlreadyExistsException.class)
    public ProblemDetail handleAlreadyExists(CustomerAlreadyExistsException ex) {
        return ProblemDetail.forStatusAndDetail(HttpStatus.CONFLICT, ex.getMessage());
    }

    @ExceptionHandler(InvalidCustomerException.class)
    public ProblemDetail handleInvalid(InvalidCustomerException ex) {
        return ProblemDetail.forStatusAndDetail(HttpStatus.BAD_REQUEST, ex.getMessage());
    }
}
```

The chain is now traceable end to end: database, Spring's own exception, domain exception, `ProblemDetail`.

```
POST /customers  (email already taken)
409 {"detail":"Customer with this email is already registered", "status":409, ...}

POST /customers  (id already taken)
409 {"detail":"Customer with id 1 already exists", "status":409, ...}

GET /customers/999
404 {"detail":"Customer with id 999 not found", "status":404, ...}
```

A two-character name never reaches the database at all — `@Size` stops it, and the client gets a 400 straight from `MethodArgumentNotValidException`. `chk_customer_name_min_length` only fires if a value reaches the repository bypassing that check. When it does, the translator produces an `InvalidCustomerException` and the same 400, with the message "Customer name is too short (minimum 3 characters)."

## Why keep the constraint if validation already covers it

This is usually the point where someone asks why bother, if `@Size` already rejects a short name. Because validation and the database are answering different questions. Validation asks: is this request well-formed? The constraint asks: is the data in this row still valid, no matter which path put it there? A batch import, a script run directly against the database, a second service writing to the same table, a validation rule someone loosened by mistake — none of them go through your controller, and all of them can still hit the constraint.

| Level | What it checks | When it fires |
|---|---|---|
| Jakarta Validation | format and bounds of input | in the controller, earliest |
| Domain and service layer | invariants that need context | on create and update |
| Database | declared integrity constraints (`UNIQUE`, `CHECK`, `NOT NULL`, `FK`) | at write time |

The database catches whatever the layers above missed: a race on `UNIQUE`, direct table access, a validation rule that quietly went missing somewhere upstream. It's also the only layer that sees the full state of the data at once, not just the one request in front of it.

## What this doesn't give you for free

**One violation per statement.** A single `INSERT` fails on the first constraint it hits, and which one that is isn't predictable in advance. Collecting every error at once is Jakarta's job, not the database's.

**Duplicated numbers.** "Minimum 3 characters" lives in the SQL and in the message text, and there's no clean way to derive one from the other — `pg_get_constraintdef` returns an expression, not a sentence a user can read. Keep the numbers as constants and add a test that inserts a too-short name and checks the message it gets back. The same goes for `@Email` on the DTO and the regex in the schema: two independent implementations of "valid email," easy to let drift apart without noticing.

**Constraint names are load-bearing.** Rename one in a migration without updating the translator, and the next violation quietly becomes a 500. Worth a test that would catch it.

## The takeaway

Catch format errors at the door — it's cheap and keeps obvious garbage out. Leave uniqueness and integrity to the database — a pre-check can't close the race, only the constraint can. Translate what comes back into exceptions your domain actually speaks, and let `ProblemDetail` carry that to the client. The database ends up checking some of what the layers above already checked. That's not redundancy for its own sake — it's the one layer that doesn't have to trust anything upstream of it.

How do you draw this line in your own services — everything in the database, or do you lean more on the application layer? Curious how others split it.

Code: https://github.com/java-backend-architecture/constraint-violation-translation
