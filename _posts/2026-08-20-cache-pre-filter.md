---
title: "Cache as a Pre-Filter Before a Database Constraint"
date: 2026-08-20 08:00:00 +0300
categories: [Architecture, Caching]
tags: [caching, spring, database, data-integrity, cache-consistency]
---
In short: a cache can decide the outcome of an operation before the database is ever asked. Checking a cache before a write looks like a pure performance trick — skip the redundant query, save a round-trip. But a cache hit doesn't just skip the database, it replaces its decision. The database constraint always guarantees data integrity. But a cache hit can decide success or rejection on its own — without the database ever being consulted.

There's a common scenario: before an INSERT, the application checks whether a record already exists.

For example:

```java
public void create(String name, String email) {
    if (userRepository.existsByEmail(email)) {
        throw new IllegalStateException("Customer already exists: " + email);
    }

    userRepository.save(new User(name, email));
}
```

At first glance this looks reasonable. But `existsByEmail()` is an extra round-trip to the database.

```
request
   ↓
SELECT EXISTS(...)
   ↓
INSERT
```

If a significant share of requests are duplicates, the database ends up doing extra work that could sometimes be cut off earlier.

This is where a different idea comes in: **use the cache as a pre-filter.**

## A Working Example

```java
@Repository
@RequiredArgsConstructor
public class CustomerRepository {
    private final JdbcClient jdbc;
    private final CacheManager cacheManager;

    public Customer findById(String id) {
        Cache cache = cacheManager.getCache("customer");
        Customer cached = cache.get(id, Customer.class);

        if (cached != null) { return cached; }

        var sql = "SELECT * FROM customer WHERE id = :id";
        Customer customer = jdbc.sql(sql)
                                .param("id", id)
                                .query(Customer.class)
                                .single();

        cache.put(customer.id(), customer);
        cache.put(customer.email(), customer);

        return customer;
    }

    public Customer save(Customer customer) {
        Cache cache = cacheManager.getCache("customer");
        Customer cached = cache.get(customer.email(), Customer.class);

        if (cached != null) {
            throw new IllegalStateException(
                "Customer already exists: " + customer.email()
            );
        }

        var sql = "INSERT INTO customer VALUES (:id, :name, :email)";

        try {
            jdbc.sql(sql).paramSource(customer).update();
        } catch (DataIntegrityViolationException e) {
            throw new IllegalStateException(
                "Customer already exists: " + customer.email(), e
            );
        }

        cache.put(customer.id(), customer);
        cache.put(customer.email(), customer);

        return customer;
    }
}
```

The method is called `save()`, but it doesn't behave like a classic Spring/JPA `save()` (insert-or-update). It behaves like `createOrReject()`: if the email has already been seen, the attempt to create a record is rejected outright — not silently turned into a read of the existing one. This is a deliberate choice for a *creation* operation, not a universal substitute for update: if the business logic needs a real upsert, a pre-filter keyed on email is not the right tool for that.

Both paths — a cache hit and a UNIQUE constraint violation — throw the same exception type with the same meaning. The calling code doesn't care whether the operation was cut off at the cache level or at the database level — the error semantics are identical.

The same `Cache` region is indexed by two different keys at once — `id` and `email`. This is intentional: `findById` looks up by id, `save` checks for duplicates by email, and both paths share the same cache.

Table schema:

```sql
CREATE TABLE IF NOT EXISTS customer (
    id         UUID PRIMARY KEY,
    name       VARCHAR(255) NOT NULL,
    email      VARCHAR(255) NOT NULL UNIQUE
);
```

`UNIQUE(email)` here isn't a formality. It's precisely this constraint that remains the last resort if the pre-filter fails to catch a duplicate.

## Two Different Levels of "Guarantee" — Stated Once, for Good

Before going further, it's worth pinning down one thing that's otherwise easy to blur.

**A database constraint guarantees data integrity.** `UNIQUE(email)` guarantees exactly one thing: the `customer` table will never contain two rows with the same email. This guarantee is absolute and independent of cache state, because the only physical path to creating a record still goes through an `INSERT` checked against the constraint.

**A cache can influence operation semantics.** But the decision "was the record successfully created, or rejected" is no longer purely a data-integrity question — it's an *operation semantics* question: did this particular caller get the outcome that the system's actual state warranted at that moment? And here, as will become clear below, the cache is not just "an optimization in front of the DB" — it's a participant in the decision.

```
DB constraint  → guarantees data integrity
Cache          → may influence operation semantics
```

This distinction matters to hold onto from the start: further down there's a counter-example (a stale positive cache entry) that might look like it refutes the idea that "the DB is the authoritative source of truth." It doesn't — it's a refinement. The DB remains authoritative for *integrity*, but it does not solely control what the caller *sees*.

## Verifying the Behavior With Tests

Tests that demonstrate the behavior explicitly:

```java
@SpringBootTest
class CustomerRepositoryTest {

    @Autowired
    private CustomerRepository repository;

    @Autowired
    private CacheManager cacheManager;

    @Autowired
    private JdbcClient jdbc;

    private String uniqueEmail() {
        return UUID.randomUUID() + "@gmail.com";
    }

    @BeforeEach
    void clearCache() {
        var cache = cacheManager.getCache("customer");
        if (cache != null) { cache.clear(); }
    }

    @Test
    void secondInsertWithSameEmailIsRejectedByCache() {
        String email = uniqueEmail();

        repository.save(
            new Customer(UUID.randomUUID().toString(), "jack", email)
        );

        assertThrows(
            IllegalStateException.class, () -> repository.save(
                new Customer(UUID.randomUUID().toString(), "jack-dup", email)
            )
        );
    }

    @Test
    void findByIdReturnsCachedValueWithoutHittingDbTwice() {
        String email = uniqueEmail();
        Customer saved = repository.save(
            new Customer(UUID.randomUUID().toString(), "jack", email)
        );

        // warm the cache with the first call
        Customer first = repository.findById(saved.id());
        assertEquals(saved, first);

        // remove the row from the DB, bypassing the repository —
        // if the second findById actually hit the DB, it would fail,
        // since the row no longer exists
        jdbc.sql("DELETE FROM customer WHERE id = :id")
                .param("id", saved.id())
                .update();

        Customer second = repository.findById(saved.id());

        // since a result was still returned while the row no longer
        // exists in the DB — second came strictly from the cache
        assertEquals(saved, second);
    }

    @Test
    void staleCachedEmailCausesFalseRejection() {
        String email = uniqueEmail();
        Customer saved = repository.save(
            new Customer(UUID.randomUUID().toString(), "jack", email)
        );

        // warm up the positive email entry in the cache
        repository.findById(saved.id());

        // delete the row directly from the DB, bypassing save()/the repository —
        // simulating an external mutation (another service, a manual DELETE, a migration)
        jdbc.sql("DELETE FROM customer WHERE id = :id")
                .param("id", saved.id())
                .update();

        // the email no longer exists in the DB, so a new insert with the
        // same email should be allowed — but the cache still thinks it's taken
        assertThrows(
            IllegalStateException.class, () -> repository.save(
                new Customer(UUID.randomUUID().toString(), "someone-else", email)
            )
        );

        // and this is a false reject: confirm the email is actually free in the DB
        Optional<Customer> inDb = jdbc.sql(
            "SELECT * FROM customer WHERE email = :email"
        )
                .param("email", email)
                .query(Customer.class)
                .optional();

        assertTrue(
            inDb.isEmpty(),
            "email is free in the DB, but save() rejected the operation anyway"
        );
    }

    @Test
    void concurrentInsertsWithSameEmail_neverBothSucceed() throws InterruptedException {
        String email = uniqueEmail();
        int threads = 2;

        ExecutorService pool = Executors.newFixedThreadPool(threads);
        CountDownLatch ready = new CountDownLatch(threads);
        CountDownLatch go = new CountDownLatch(1);
        List<Future<Boolean>> results = new ArrayList<>();

        for (int i = 0; i < threads; i++) {
            results.add(pool.submit(() -> {
                ready.countDown();
                go.await();
                try {
                    repository.save(new Customer(UUID.randomUUID().toString(), "racer", email));
                    return true;
                } catch (IllegalStateException e) {
                    return false;
                }
            }));
        }

        ready.await();
        go.countDown();
        pool.shutdown();

        long successCount = results.stream()
                .map(f -> {
                    try {
                        return f.get();
                    } catch (Exception e) {
                        throw new RuntimeException(e);
                    }
                })
                .filter(Boolean::booleanValue)
                .count();

        assertEquals(
            1, successCount,
            "exactly one insert should succeed, the second should be rejected"
        );
    }
}
```

**Test 1** proves the pre-filter itself: the second `save()` with the same email is rejected without ever reaching the `INSERT`.

**Test 2** proves not just a matching result, but the actual fact of hitting the cache: after the first `findById`, the row is physically deleted from the DB behind the repository's back, and the second `findById` still successfully returns the same data. If the second call had actually gone to the DB, `.single()` would have thrown due to the missing row — the test would fail. The fact that it passes is literal proof that the second call never touched the DB.

**Test 3** demonstrates the flip side of the same mechanism: if the cache holds a stale positive entry, `save()` rejects the operation without even looking at the DB — even though the email is actually free in the DB at that point. The test explicitly checks both facts: that `save()` threw, and that there was no actual conflict in the DB at that moment. This isn't a hypothetical risk — it's reproducible behavior of the current implementation.

**Test 4** — `concurrentInsertsWithSameEmail_neverBothSucceed` — runs two threads simultaneously trying to create a record with the same email, and verifies exactly one succeeds. The test proves the invariant: of two concurrent `save()` calls with the same email, no more than one succeeds. It deliberately does not assert *which* mechanism produced that outcome — cache rejection or database constraint.

## The Cache Does Not Replace the Database Constraint

This is a key point — but, as noted above, it's true specifically for data integrity, not for the operation's semantics as a whole.

```
cache miss
    ↓
   DB decides           ← integrity and semantics are decided together

cache hit
    ↓
   operation rejected   ← the DB is never consulted
```

If the cache is empty or the entry has fallen out of it — yes, the request really does reach the database, and the database decides. But if the cache holds a stale positive entry (the email is in the cache, even though it's no longer in the DB), something different happens: `save()` throws and returns, without ever consulting the DB.

```
DB:       email does NOT exist
Cache:    email exists (stale entry)

save()
   ↓
cache hit
   ↓
reject
   ↓
DB was never checked
```

Data integrity isn't violated here — a duplicate row in the table is still impossible. But this *is* a false rejection of a legitimate operation: a positive cache hit becomes the final decision on semantics by itself, even though the DB remains solely responsible for integrity.

> **A stale positive cache entry is not harmless: unlike a cache miss, it can cause a false rejection without consulting the database. Therefore this pattern requires an explicit freshness/invalidation strategy — or acceptance of false rejects.**

In practice, this means the pattern, as shown, is acceptable only under one of two conditions:

- the cache is updated/invalidated by the same system that writes to the DB (i.e., no external mutation can desynchronize them), or
- the application deliberately accepts the possibility of occasional false rejections as a trade-off — for example, because a user retry is cheap.

If neither condition holds — say, there are external services or migrations that can delete a row behind this code's back — the pattern needs either a TTL, explicit cache invalidation on external mutations, or should be dropped for `save()` altogether.

## Why Does This Even Make Sense?

The point isn't that an in-memory check is always faster than a database round-trip — that depends on the setup. A local database with a warm connection pool and a good index on `email` can answer `SELECT EXISTS` in fractions of a millisecond, comparable to a cache lookup — especially if the cache is distributed (Redis) and adds its own network overhead.

A more precise way to put it: **the cache lets you avoid a round-trip to downstream storage for already-known duplicates.** This is a logical property of the pattern — true regardless of configuration. Whether that translates into a concrete latency win, and how large, is an empirical question — and the answer comes not from architecture but from a benchmark in a specific environment: a specific database, a specific index, a specific cache type, and a specific ratio of duplicate requests.

## What About Multiple Instances?

With a local cache like Caffeine, each application instance has its own state:

```
Instance A → Cache A
Instance B → Cache B
```

An entry present in Cache A isn't necessarily known to Cache B. This doesn't break uniqueness in the DB — the constraint still guarantees that. But the risk of a false reject becomes even more localized and unpredictable: an entry can be stale in Cache A but current in Cache B, and the outcome of `save()` starts depending on which instance the request landed on.

If filtering needs to work consistently across multiple application instances, a distributed cache like Redis can be used — but it, too, doesn't eliminate the need for the explicit invalidation strategy described above.

## Why Does the Record Get Cached Only After the DB Write?

Because it must first successfully pass through authoritative storage:

```java
Customer customer = ...; // after a successful INSERT / SELECT
cache.put(customer.id(), customer);
cache.put(customer.email(), customer);
```

We don't want to put a record into the cache before the operation is confirmed. This lets the cache act specifically as a reflection of already successfully processed data, not as a source of truth — with the caveat that this "reflection" can drift from the actual DB state over time, and that drift is exactly what creates the false-reject risk discussed above.

In more complex transactional scenarios, the exact moment of syncing the cache with the commit requires a separate solution, but for a simple operation the idea stays the same.

## Where Does a Bloom Filter Fit In?

A Bloom filter solves a similar problem — but with a fundamentally safer asymmetry:

```
Bloom filter: "definitely absent" → skip DB (safe, DB not needed)
              "maybe present"     → DB (final decision always with the DB)
```

The key difference: a positive Bloom filter result does not finalize a negative outcome. It only lowers confidence and passes the decision on to the DB. A false positive here means "possibly exists, needs checking" — not "definitely exists, reject."

The plain cache used in `save()` above has the opposite semantics:

```
Cache:  "hit"  → reject (the DB is never consulted)
        "miss" → DB (final decision rests with the DB)
```

A positive result here cuts the chain short and finalizes the outcome itself. That's exactly why a plain cache in this role can't be fully equated with a Bloom filter, even though the tasks look similar: with a Bloom filter, the risk of a false rejection is architecturally excluded by construction; with a point-cache, it isn't — and has to be closed separately (TTL, invalidation, or a deliberate acceptance of the risk).

## When the Pattern Fits — and When It Doesn't

**Good fit when:**

- duplicate requests are common;
- the lookup key is stable (e.g., email doesn't change between attempts);
- a cache hit genuinely means "it's acceptable to reject this operation" — i.e., the cache and DB are guaranteed not to diverge, or divergence is an acceptable business risk;
- the database constraint remains the authoritative mechanism for data integrity.

**Poor fit when:**

- every request must reach the database (e.g., audit or billing);
- a real upsert is needed — the payload must be applied even if the key has been seen before;
- the record can be deleted or modified behind this code's back, and a false reject is unacceptable;
- the record has side effects that cannot be silently skipped.

## The Main Point

The cache is not obligated to answer the question:

> "Does this record already exist in the DB?"

That question should be answered by the database constraint — and for data integrity, that's exactly how it works: the constraint remains the sole source of truth, regardless of cache state.

But the question "does this particular caller get success or rejection?" is, in the current implementation, sometimes answered by the cache itself, without asking the DB. This is a deliberate trade-off, not a side effect — and it should be explicit, not implied.

The pre-filter answers a different question:

> "Is there any point in sending this operation further at all?"

The idea isn't to replace `existsByEmail()` with `cache.get(email)` as a new uniqueness mechanism. The idea is this:

> **Use the cache to filter. Use the database to guarantee — for the positive path. For the negative path, the cache itself becomes the decision-maker, and that requires its own consistency guarantees.**

The cache can be a fast filter in front of the database. But data integrity is still guaranteed by the database — even if a cache hit sometimes decides the outcome before the request ever gets there.
