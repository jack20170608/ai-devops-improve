---
applyTo: "**/*.java"
---

# Java development instructions

Human-readable source: `10-Devops-SOP/02-java-coding-standard.md`.

If these rules conflict with the build configuration, an ADR, or a more specific instruction, stop and report the conflict.

## Before editing

- Read the target type, callers, tests, package structure, and module build file.
- Reuse existing project patterns before introducing an abstraction or dependency.
- For a defect, add a test that reproduces the failure before changing production behavior.
- Make the smallest complete change; do not modify unrelated code.

## Architecture

Organize code by business capability:

```text
orders/
├── api/
├── application/
├── domain/
└── persistence/
```

- Domain code must not import HTTP, ORM, persistence, or framework-specific packages.
- Controllers must call application use cases, not repositories or databases.
- Application code coordinates use cases and transaction boundaries.
- Persistence adapters implement interfaces owned by the calling business module.
- Keep dependency directions explicit and acyclic.
- Use the narrowest visibility that works.
- Do not create generic `util`, `helper`, `manager`, `common`, or `misc` dumping grounds.

## Types and methods

- Use the repository formatter, UTF-8, braces for control flow, and no wildcard imports.
- Name types with domain nouns, methods with verbs, and boolean methods as predicates.
- Make nullability explicit on public and cross-module interfaces.
- Return empty collections instead of `null`.
- Use `Optional<T>` only for return values that may be absent.
- Prefer value objects for identifiers, money, status, and constrained values.
- Use `BigDecimal` or a Money type for currency; never use `double`.
- Prefer immutable records or final classes.
- Defensively copy mutable inputs and outputs.
- Use `Instant` or `OffsetDateTime` for time points.

Avoid:

```java
void createOrder(String id, double amount, String currency, boolean force);
```

Prefer:

```java
CreateOrderResult createOrder(
    OrderId orderId,
    Money amount,
    CreationPolicy policy);
```

## Errors and resources

- Catch only exceptions that can be handled meaningfully.
- Never use an empty catch block or return fallback success after failure.
- Do not catch `Throwable` or `Error` in ordinary application code.
- Preserve the original cause when translating exceptions.
- Use try-with-resources for every `AutoCloseable`.
- Propagate `InterruptedException` or restore the interrupt flag.
- Log an exception once at the place responsible for handling or observing it.

```java
try {
  repository.save(order);
} catch (SQLException cause) {
  throw new OrderPersistenceException(
      "Unable to store order " + order.id(), cause);
}
```

## Concurrency and external calls

- Prefer no shared state, then immutable state, then high-level concurrency abstractions.
- Give every remote call, blocking I/O operation, lock wait, and future wait a timeout.
- Bound thread pools, queues, database connections, and downstream concurrency.
- Do not use `Thread.stop()` or assume `volatile` makes compound operations atomic.
- Test concurrency with latches, barriers, futures, and deterministic timeouts, not `sleep()`.

## Logging and security

- Use structured, parameterized logging.
- Never log passwords, tokens, session IDs, private keys, authorization headers, connection strings, complete request bodies, or unnecessary personal data.
- Do not concatenate untrusted values into SQL, shell commands, expressions, or log formats.
- Use parameterized queries.
- Validate trust-boundary input for type, length, range, allowed values, and business rules.
- Enforce object ownership and tenant authorization in the query or transaction.

Unsafe:

```java
String sql = "SELECT * FROM account WHERE customer_id = '" + customerId + "'";
```

Safe:

```java
try (PreparedStatement statement = connection.prepareStatement(
    "SELECT account_id FROM account WHERE customer_id = ?")) {
  statement.setString(1, customerId);
}
```

## Tests and completion

- Add unit tests for domain rules and integration tests for database, messaging, and HTTP behavior.
- Add negative tests for validation, authorization, timeout, retry, and concurrency behavior.
- Control time, randomness, UUIDs, and external I/O through replaceable dependencies.
- Do not leave `@Disabled` tests without an issue, owner, reason, and expiry.
- Run the smallest relevant tests, then the repository's required full verification command.
- Do not declare completion if required checks fail or were silently skipped.

