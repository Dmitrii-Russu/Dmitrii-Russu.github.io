---
layout: post
title: "When Should a Database Exception Be Translated into a Domain Exception? (Spring)"
date: 2026-11-27
description: "Spring already translates database exceptions into DataAccessException. When is a further translation into a domain exception justified? A criterion based on who consumes the exception."
tags: [java, spring, exceptions, architecture, clean-architecture, postgresql]
---

**If the only party that receives information about a violation is the HTTP handler (it builds the response for the client and writes diagnostics for the developer), does the database exception need to be translated into a domain exception?**

Jakarta Validation checks incoming data, but on its own it does not guarantee uniqueness. You can run a pre-check query against the database, but it does not replace a UNIQUE constraint.

**In short:** translation is justified when something inside the core consumes the exception. If the only consumer is the HTTP handler, Spring's `DataAccessException` already does the job.

## 1. Two Ways to Enforce Uniqueness

**In application code.** Before writing, the application checks whether the value already exists and throws its own exception if it does. A `@RestControllerAdvice` turns that exception into an HTTP response. But the check does not protect against concurrent inserts, so the UNIQUE constraint in the database is needed anyway.

**With a database constraint.** On a violation, PostgreSQL throws its own exception, and Spring translates it into its `DataAccessException` hierarchy (here, `DataIntegrityViolationException`). This is more than a rename: Spring provides a single hierarchy that frees your code from depending on a specific data access technology. The original cause is preserved, so database-specific details can still be retrieved when needed. In my example, the same handler is verified with both `JdbcClient` and Hibernate/JPA.

## 2. Who Consumes the Exception?

When the application-level check detects the duplicate, the handler receives an application exception. When the database detects it, the handler receives a Spring exception. Why translate the latter into a domain exception?

The usual answer is Clean Architecture: the core must not depend on infrastructure, and dependencies point inward. Nobody disputes that. Let's look at the concrete case.

The exception travels up the stack from the repository to the advice. The layers in between neither import nor catch it. `DataAccessException` is an unchecked exception, so it does not have to be declared in method signatures. The core's code does not depend on its type.

The dependency rule protects the core. The controller and the advice sit outside it and already use Spring MVC and `@RestControllerAdvice`. An adapter is supposed to know about infrastructure, so Spring in the controller and the advice is a design decision, not a violation. **Which type of exception reaches the controller is a matter of the controller's convenience, not of the core's safety.**

## 3. Can Anything Get Worse?

Can anything get worse because the handler receives a Spring exception rather than a domain one?

The code confirms only the first item below, along with the classification of violations by SQLState and constraint name. The rest is argued by reasoning: the example has no core layer and no tests for it.

- Will the handler need to change when moving from JDBC to JPA? No. The same handler is verified with both mechanisms.
- Independence of the core? The core's code does not depend on the exception type.
- Testing the core in isolation? The core does not need the exception type.
- Transaction rollback? If an unchecked exception propagates out of a transactional method, Spring's default rules roll the transaction back. No domain semantics are needed for that.
- Replacing the web framework? You would have to change the web adapter and its integration with the new framework. Swapping the exception type does not remove that work.

In my example, changing the database vendor would still require changes: Spring's hierarchy hides the data access technology, but not the differences between databases. You would have to update the class that extracts details from the cause (SQLState, constraint name) and the classification policy. A translator producing a domain exception would still have to extract the same technical data and classify the violation, so it does not remove the dependency on a specific database.

People say that depending on Spring is bad, but the harm is usually asserted rather than demonstrated. Name the property that gets worse when the adapter receives a Spring exception, and it can be checked. Above, five possible consequences were examined, and none of them gets worse.

## 4. When There Is a Consumer in the Core

Translation is justified when internal logic must make a different decision on a conflict. Then the exception has a consumer in the core, and you need your own application-level contract (an exception or a plain operation result).

The wish to test the reaction to a conflict does not by itself require a domain exception. A collection-based fake lets you test application logic, but it does not throw database exceptions and does not reproduce the real behavior of UNIQUE and concurrent inserts. That can only be verified by integration tests against a real database.

## 5. Why Add Another Translation?

The chain looks like this:

1. The repository receives a Spring exception.
2. It translates it into its own exception.
3. The exception travels up the stack to the advice.
4. The advice builds the response.

**Who inside the application consumes the new exception, and what decision will it make based on it?**

If there is no such consumer, the translation is infrastructure for its own sake. We are paid to solve problems, not to maximize the number of exception-translation layers.

---

## Code

- [integrity-violation-direct-handling](https://github.com/java-backend-architecture/integrity-violation-direct-handling)
