---
tags: [php, symfony, doctrine, caching, phpstan]
author: Nicolas Mugnier
categories: architecture
description: "Retour d'expérience sur l'optimisation de jobs background et de cache dans une application PHP/Symfony : itération cursor-based, Messenger, cache par tags et règle PHPStan custom."
locale: en_US
image: /assets/img/scaling-background-jobs-and-caching.png
---

# Scaling Background Jobs and Caching in a PHP Platform: Patterns That Worked

When a background job that "works fine" starts crashing at scale, the fix is rarely a single change. It is usually a chain of architectural decisions — each one unlocking the next. This article walks through five of those decisions, made over several months on a PHP/Symfony platform, to turn a fragile synchronous pipeline into a resilient, cache-aware, statically verified system. Each part can be read independently, but together they tell the story of how a series of small, deliberate improvements compounded into a fundamentally different architecture.

---

## Part 1 — From Full Loads to Cursor-Based Iteration in Doctrine

### The problem with loading everything

When you need to synchronize a large dataset from your database into an external search engine, the naive approach is tempting: fetch all records, loop over them, push each one. For a while, it works perfectly.

Then your dataset grows.

At some point, your query returns tens of thousands of rows. Doctrine hydrates each row into a fully populated entity — with all its associations, collections, and metadata eagerly loaded. PHP's memory usage climbs steadily. The process hits the memory limit. The command crashes halfway through. You restart it, it crashes again at a different point. You bump the memory limit. It crashes later. You are now playing whack-a-mole with a fundamental architectural problem.

The root issue is simple: you are loading the entire dataset into memory before processing a single record. In Big O terms, memory usage is **O(n)** where *n* is the total number of records. As your dataset doubles, your memory requirement doubles — and eventually exceeds PHP's memory limit.

### Why OFFSET pagination isn't the answer

The first fix most developers reach for is OFFSET-based pagination — split the query into pages using `LIMIT 100 OFFSET 0`, then `LIMIT 100 OFFSET 100`, and so on. It is familiar, easy to reason about, and works fine at small scale.

The problem is that `OFFSET` is a lie. When you write `OFFSET 50000`, you are not asking the database to start at row 50,000. You are asking it to scan and discard the first 50,000 rows before returning your 100. The further you paginate, the more rows are scanned on every request. At scale, each page becomes progressively slower — a full table scan disguised as pagination.

To fetch page *p* of size *k*, the database must scan *p × k* rows. The cost of a single page is **O(p × k)**, and the total cost to iterate the entire dataset across all *n/k* pages is **O(n²/k)** — quadratic in the number of records. Memory is bounded at **O(k)** per page, but the time cost grows explosively.

The correct alternative is **keyset pagination**, also called **cursor-based pagination**. Instead of telling the database "skip N rows", you tell it "give me rows where `id > :last_seen_id`". The database uses the primary key index directly and jumps immediately to the right position — no scanning, no discarding. Each page costs **O(k + log n)**: a B-tree index seek in **O(log n)** to find the starting point, then a sequential scan of *k* rows. The total cost to iterate the entire dataset is **O(n + (n/k) × log n)** — essentially linear.

```sql
SELECT DISTINCT user_id
FROM profiles
WHERE /* your filters */
AND user_id > :lastUserId
ORDER BY user_id
LIMIT :batchSize
```

Each page starts exactly where the last one ended. The cursor is simply the last ID you processed.

### Two generators for two use cases

PHP's `\Generator` is the natural fit for exposing this pattern. A generator is lazy — it computes the next value only when the caller asks for it, and suspends execution between yields. This means memory usage stays bounded regardless of the total dataset size.

Two methods were introduced on the repository:

`iterateAll(batchSize: 100)` yields fully hydrated entities one batch at a time. Critically, after each batch is yielded and the caller has processed it, the Doctrine Entity Manager is explicitly cleared to detach all entities from the identity map and release their memory before the next page:

```php
public function iterateAll(array $filters = [], int $batchSize = 100): \Generator
{
    $lastId = 0;

    do {
        $ids = $this->getDistinctIdsAfter($baseQuery, $parameters, $lastId, $batchSize);

        if (empty($ids)) {
            break;
        }

        yield from $this->buildEntitiesForIds($ids);

        $lastId = \end($ids);
        $this->getEntityManager()->clear(); // free memory before next batch
    } while (\count($ids) === $batchSize);
}
```

`iterateUserIds(batchSize: 500)` is a lighter variant that yields only `int[]` arrays of IDs, without hydrating any entities at all. This is used by the dispatch command, which only needs identifiers to put on a message queue — it has no need to load the full entity graph.

The result is a process that can iterate over 100,000 records with the same memory footprint as iterating over 100. Memory usage is **O(k)** where *k* is the batch size — a constant that you control, independent of *n*. The peak memory usage is determined by the batch size, not by the size of the dataset.

To summarize the three approaches:

| Approach | Time (per page) | Time (total) | Memory |
|---|---|---|---|
| Full load | O(n) | O(n) | O(n) |
| OFFSET pagination | O(p × k) | O(n²/k) | O(k) |
| Keyset pagination | O(k + log n) | O(n + (n/k) × log n) | O(k) |

---

## Part 2 — Moving Heavy Background Jobs to Symfony Messenger

### The synchronous trap

Even with memory-efficient iteration, running a large background job synchronously in a console command has a fundamental reliability problem: the command is only as resilient as its weakest moment.

One slow external API call, one transient network error, one unexpected exception — and the entire job halts. You have to figure out where it stopped, reason about partial state, and restart manually. There is no retry mechanism, no isolation between failures, and no way to parallelise work.

The architectural answer is to stop doing the work in the command and instead dispatch the work to be done asynchronously by **Symfony Messenger** workers.

### Batch dispatch and bulk processing

The refactored architecture splits the work into two stages: a lightweight dispatcher and a batch handler that processes each group as a unit.

<div class="mermaid">
flowchart LR
    CMD[Console Command] -->|iterateUserIds| GEN[ID Generator]
    GEN -->|batch of 500 IDs| B1[ProcessBatchRequest]
    GEN -->|batch of 500 IDs| B2[ProcessBatchRequest]
    GEN -->|batch of 500 IDs| B3[ProcessBatchRequest]
    B1 --> H1[Batch Handler]
    B2 --> H2[Batch Handler]
    B3 --> H3[Batch Handler]
    H1 --> DB[(Database)]
    H1 -->|bulkUpsert| SE[Search Engine]
    H2 --> DB
    H2 -->|bulkUpsert| SE
    H3 --> DB
    H3 -->|bulkUpsert| SE
</div>

The command is now trivially simple. It iterates over the ID generator and dispatches one batch message per page. It completes in seconds:

```php
foreach ($this->repository->iterateUserIds(batchSize: 500) as $userIds) {
    $this->bus->dispatch(new ProcessBatchRequest($userIds));
    $batchCount++;
}
```

No entities are loaded. No external calls are made. The command's only job is to put work on the queue.

The batch handler receives a group of IDs and processes them as a single unit. On the database side, entities are loaded individually within the handler. But on the search engine side, all records are pushed in a single `bulkUpsert` call rather than one API call per record. At high volume, this makes a significant difference — network round-trips are the dominant cost, and batching reduces them by a factor equal to the batch size.

Each batch message is an independent, retryable unit of work. If a batch fails, it can be retried or moved to a dead-letter queue without affecting any other batch. Workers can process multiple batches in parallel across different processes or machines.

### Making the event-driven architecture async

The batch processing pipeline was only half of the picture. The application already had an event-driven architecture in place: when a user updated their profile, added an experience, or deleted a record, domain events were fired and subscribers would update the search index accordingly. The problem was that this entire flow — from event to database write to search engine API call — happened synchronously, inside the HTTP request cycle.

This meant that every user action that triggered an indexation paid the full cost of the search engine round-trip before the API could respond. On a single operation the overhead is small, but it adds up: a user saving a form would wait not just for the database write, but also for the external API call to the search engine. Latency crept into every mutation endpoint.

<div class="mermaid">
flowchart LR
    subgraph Before
        A1[User Action] --> A2[Event Subscriber]
        A2 --> A3[Database Write]
        A3 --> A4[Search Engine API Call]
        A4 --> A5[HTTP Response]
    end
    subgraph After
        B1[User Action] --> B2[Event Subscriber]
        B2 --> B3[Database Write]
        B3 --> B4[Dispatch Message]
        B4 --> B5[HTTP Response]
        B4 -.->|async| B6[Worker]
        B6 --> B7[Search Engine API Call]
    end
</div>

The fix was to apply the same Messenger pattern, but at the individual event level. Instead of calling the search engine synchronously inside the event subscriber, the subscriber now dispatches a single message per item onto the queue. The worker picks it up and performs the insert, update, or delete asynchronously.

The benefits are twofold. First, API response times improved immediately — the user's request completes as soon as the database write is done and the message is enqueued, without waiting for the search engine. Second, the search engine updates inherit all the resilience properties of async processing: automatic retries, failure isolation, and the ability to absorb temporary downstream outages without impacting the user-facing application.

### Isolating traffic with a dedicated transport

<div class="mermaid">
flowchart LR
    MSG[Message] --> T[job_processing transport]
    T -->|attempt 1| W[Worker]
    W -->|failure| R1["Retry #1 — 1 min"]
    R1 -->|failure| R2["Retry #2 — 3 min"]
    R2 -->|failure| R3["Retry #3 — 9 min"]
    R3 -->|failure| R4["Retry #4 — 27 min"]
    R4 -->|failure| DLQ[failed_job_processing]
    W -->|success| DONE[Done]
</div>

A subtle but important decision was giving this job its own dedicated Messenger transport, rather than sharing an existing queue.

Sharing queues across different message types creates coupling. A spike in one domain's message volume starves other domains. A consumer crash affects unrelated work. Tuning retry behaviour for one message type can break the semantics of another.

With a dedicated transport:

```yaml
job_processing:
    dsn: "%env(APP_MESSENGER_TRANSPORT_DSN)%"
    failure_transport: failed_job_processing
    options:
        queue_name: job_processing
        redeliver_timeout: 300
    retry_strategy:
        max_retries: 4
        delay: 60000       # start at 1 minute
        multiplier: 3      # exponential: 1min → 3min → 9min → 27min
        max_delay: 3600000 # never wait more than 1 hour
```

The exponential backoff is deliberate. Transient failures — rate limits, network blips, momentary service unavailability — typically resolve within minutes. Retrying immediately would pile additional pressure onto an already stressed downstream system. Waiting progressively longer gives it time to recover while still eventually processing the message. After four retries, the message lands in a dedicated failure transport where it can be inspected, understood, and manually re-dispatched once the root cause is resolved.

A marker interface routes all related messages to this transport through a single routing rule, keeping the configuration clean.

### Preserving side effects in an async context

Moving from synchronous to asynchronous introduced a subtle regression: analytics events that were fired synchronously inside the original event subscriber were lost in the migration.

In a synchronous flow, the subscriber has access to the current state: it knows whether the record existed before the operation, so it can fire a "created" or "deleted" event accordingly. In an async handler, this context is gone — the handler runs later, in a separate process, with no memory of the state at the time the domain event was raised.

The solution was to reconstruct the necessary state inside the handler itself by querying the current state of the search index before processing:

```php
$wasIndexed = $this->isIndexed($request->id);

try {
    $entity = $this->repository->findById($request->id);
    $this->searchGateway->insert($entity);

    if (!$wasIndexed) {
        $this->analytics->track($request->id, Events::CREATED);
    }
} catch (NotFoundException) {
    $this->searchGateway->delete($request->id);

    if ($wasIndexed) {
        $this->analytics->track($request->id, Events::DELETED);
    }
}
```

Before touching the search index, the handler checks whether the record is currently indexed. If the record exists in the database and was not indexed → this is a creation. If the record no longer exists and was indexed → this is a deletion. The analytics events fire with correct semantics regardless of when the handler runs relative to when the original domain event was raised.

---

## Part 3 — Profile Completion Cache: Making Expensive Aggregations Cheap

### Why aggregated scores are expensive

A profile completion score is not a single value stored in a column. It is computed by evaluating multiple independent data sections — each backed by its own database query — and aggregating the results into a percentage. In our case, 9 separate use cases are called, each fetching and evaluating a different piece of a user's profile. The result is then used to decide whether each section is filled, unfilled, or stale.

For a user viewing their own dashboard, this fan-out is acceptable. The cost is paid once per page view and the latency is hidden behind a loading state.

The problem emerges when this computation is triggered at scale in a background job. Our indexation pipeline calls profile completion for every user being indexed. With thousands of messages being processed in parallel, each one triggering 9 database queries, the load quickly becomes significant. Response times degrade. The database connection pool fills up. Workers slow down.

### A long-lived cache with tag-based invalidation

<div class="mermaid">
flowchart TB
    subgraph Read Path
        R1[GetProfileCompletion] -->|check| CACHE[(Cache)]
        CACHE -->|miss| R2[Compute from 9 use cases]
        R2 --> DB[(Database)]
        R2 -->|store with tags| CACHE
        CACHE -->|hit| R3[Return cached result]
    end
    subgraph Write Path
        W1[Mutation use case] -->|UpdateSection| DB
        W1 -->|InvalidateCache| INV["Invalidate tag: profile_completion_{userId}"]
        INV --> CACHE
    end
</div>

The right mental model for this kind of data is: it changes rarely and is read often. A user's profile completion rate only changes when they actively update one of their data sections. In between updates, the value is stable. Recomputing it on every access is pure waste.

The solution is aggressive caching with a 90-day TTL, backed by tag-based invalidation:

```php
#[Cache(ttl: 7776000, tags: [
    '"profile_completion_" ~ request',
    '"profile_completion_" ~ request.userId',
])]
public function execute(Request $request): Response
```

The two tags serve different invalidation scopes:

- `profile_completion_{userId}` — purge all cached completion data for a specific user, regardless of which sections were included in the request. This is the tag used by mutation use cases.
- `profile_completion_{request}` — a more granular tag that encodes both the user ID and the specific set of sections requested, via the request's `__toString()`. This ensures that if the set of sections changes (e.g. a new section is added to the profile), old cache entries with the previous section list are not incorrectly served.

### Wiring invalidation on the write side

Caching a read is only half the work. The other half is ensuring the cache is invalidated precisely when the underlying data changes. This is where many caching implementations fall apart — they cache aggressively but invalidate too broadly (wiping everything on any change) or not at all (serving stale data indefinitely).

Tag-based invalidation gives you surgical precision. Every mutation use case that modifies a profile section — create, edit, delete, save — carries an explicit cache invalidation declaration on its `execute()` method:

```php
#[InvalidateCache(tags: ['"profile_completion_" ~ request.userId'])]
public function execute(UpdateSectionRequest $request): SectionResponse
```

When a user saves their phone number, updates their bio, or adds a new experience, only their profile completion cache entry is invalidated. Every other user's cache remains untouched. The next call for that user recomputes from scratch and caches the result for another 90 days.

This work touched 18 mutation use cases across 9 data domains — not a small amount of boilerplate, but also not a complexity that can be abstracted away. Each mutation use case is independently responsible for declaring what it invalidates.

---

## Part 4 — Replacing a Hardcoded List with an Interface-Driven Registry

### The maintenance trap hiding in your constructor

There is a common pattern in codebases that starts innocent and quietly becomes a liability. It looks like this:

```php
public function __construct(int $userId)
{
    $this->sections = [
        SectionA::getSectionName(),
        SectionB::getSectionName(),
        SectionC::getSectionName(),
        // ... 6 more lines
    ];
}
```

This constructor knows too much. It encodes a business rule — "these are the sections that constitute a complete profile" — in a place that is hard to find, impossible to enforce, and easy to forget. Adding a new section means knowing this list exists, remembering where it lives, and hoping the next developer does too. It is a silent convention masquerading as code.

There is also a design problem: a request object is supposed to be a dumb data transfer object. It should carry values, not make decisions. Having it hard-code a list of use case class names is a responsibility it should not have.

### An interface as a first-class contract

The first step is to make the membership rule explicit:

```php
interface ProfileCompletionSection
{
    public static function getSectionName(): string;
}
```

Each relevant use case now declares `implements ProfileCompletionSection`. What was previously an implicit, undocumented convention — "I know this class is used for profile completion because it appears in that one constructor" — becomes a formal, statically verifiable declaration. PHPStan enforces that every implementing class exposes `getSectionName()`. The compiler is now your documentation.

The interface also serves as the hook for static analysis tooling, as we will see in Part 5.

### Symfony's tagged iterator as the wiring mechanism

With the interface in place, the next question is: how does the use case discover all implementors at runtime without building the list manually again?

The answer is Symfony's dependency injection container, which has native support for this pattern. Using `_instanceof` configuration, every class implementing the interface is automatically tagged:

```yaml
_instanceof:
    App\UseCase\ProfileCompletionSection:
        tags: ['app.profile_completion_section']
```

The use case that orchestrates profile completion then declares its dependency using `#[TaggedIterator]`:

```php
public function __construct(
    #[TaggedIterator('app.profile_completion_section')]
    private readonly iterable $sections,
    private readonly GetDatafieldCompletion $getDatafieldCompletion,
) {}
```

The container injects all tagged services automatically. Adding a new section to profile completion now requires exactly one change: adding `implements ProfileCompletionSection` to the relevant use case class. The tagged iterator picks it up. The request is built dynamically. No list to maintain.

### Navigating the service proxy layer

One non-obvious problem emerged: the injected `$sections` are not the real use case instances. They are proxy objects generated by the service proxy library, which wraps use cases to apply cross-cutting concerns like caching, security, and transaction management.

The proxy overrides instance methods to intercept calls. It does not — and cannot — override static methods, because PHP's static dispatch bypasses the virtual method table. Calling `$section::getSectionName()` on a proxy will fail or call the wrong method.

The solution is `get_parent_class()`. The proxy always extends the real class by exactly one level of inheritance. `get_parent_class($proxy)` returns the real class — the one that actually defines `getSectionName()` — and the static call resolves correctly:

```php
foreach ($this->sections as $section) {
    $realClass = \get_parent_class($section);
    if ($realClass === false) {
        continue;
    }
    $sectionNames[] = $realClass::getSectionName();
}
```

This is an honest acknowledgment of a leaky abstraction. The proxy layer is an implementation detail that bleeds into consuming code. The `get_parent_class()` call is documented with a comment explaining the assumption (proxy is exactly one level deep), and the `false` guard ensures PHPStan is satisfied and the code is safe if the assumption ever changes.

---

## Part 5 — PHPStan Rule: Enforcing Cache Invalidation at the Compiler Level

### The gap that architecture alone cannot close

The interface-driven registry solves the discoverability problem for the read path. But it quietly introduces a new risk on the write path.

When a developer adds a new `ProfileCompletionSection` implementor, they must also find every mutation use case that modifies that section's data and add the corresponding `#[InvalidateCache]` declaration. This is not obvious. There is no compiler warning, no test failure, no runtime error. The cache will simply serve stale completion rates, silently, until someone notices that user data is not updating correctly.

This is precisely the kind of bug that survives code review, passes all tests, makes it to production, and is discovered weeks later by a confused user. It is not a logic error — it is a missing declaration, which is the hardest category of bug to catch after the fact.

The correct answer is to make the compiler catch it before it ships.

### What the rule checks and how

`ProfileCompletionInvalidateCacheRule` is a custom PHPStan rule that operates on `ClassMethod` nodes. For every `execute()` method in the codebase, it applies four sequential filters:

**Is this a mutation use case?**
The class short name must start with `Create`, `Edit`, `Delete`, `Save`, or `Update`. Everything else is skipped immediately.

**Is it a datafield use case?**
The class must extend the base `DatafieldUseCase`. This filters out unrelated mutation classes that happen to share the same naming convention.

**Is it related to a profile completion section?**
This is the most interesting step. The rule extracts the "domain namespace" — the shared ancestor namespace that groups a section's Get and mutation use cases together. The algorithm strips the class short name and the verb folder from the fully qualified class name:

```
App\UseCase\Experience\CreateExperience\CreateExperience
  → remove class name  → App\UseCase\Experience\CreateExperience
  → remove verb folder → App\UseCase\Experience
```

This works for both nested structures (each verb in its own subfolder) and flat structures (all verbs in the same namespace):

```
App\UseCase\SocialLinks\DeleteSocialLinks
  → remove class name       → App\UseCase\SocialLinks
  → no verb folder to strip → App\UseCase\SocialLinks
```

The rule then checks via `get_declared_classes()` whether any `ProfileCompletionSection` implementor shares the same domain namespace. `GetExperiences` and `CreateExperience` both resolve to `App\UseCase\Experience`. The match is reliable across both structural patterns.

**Does the `execute()` method carry the right cache tag?**
The rule walks the AST of the method's attribute groups, finds `#[InvalidateCache]`, locates the `tags` named argument, iterates its array items, and checks whether any string value contains `profile_completion_`:

```php
foreach ($method->attrGroups as $attrGroup) {
    foreach ($attrGroup->attrs as $attr) {
        if ($attr->name->toString() !== InvalidateCache::class) {
            continue;
        }
        foreach ($attr->args as $arg) {
            if ($arg->name?->toString() !== 'tags') {
                continue;
            }
            foreach ($arg->value->items as $item) {
                if (
                    $item->value instanceof Node\Scalar\String_
                    && \str_contains($item->value->value, 'profile_completion_')
                ) {
                    return true; // valid
                }
            }
        }
    }
}
```

If this check fails, a descriptive error is emitted with a tip pointing to the exact fix:

```
Mutation use case "App\UseCase\Experience\CreateExperience" affects a profile
completion section but its execute() method is missing an #[InvalidateCache]
attribute with a tag containing "profile_completion_".

💡 Add #[InvalidateCache(tags: ['"profile_completion_" ~ request.userId'])]
   to the execute() method.
```

### Immediate payoff and long-term value

The rule caught a real violation on its first run — a `SavePhoneNumber` use case that was correctly invalidating the phone number cache but silently missing the profile completion invalidation. One line added, one bug prevented from reaching production.

The long-term value is structural. The rule runs in CI on every pull request. Any developer who adds a new `ProfileCompletionSection` implementor without wiring the corresponding cache invalidation on mutation use cases will see a build failure with a clear, actionable error message. The system is now self-enforcing — correctness is guaranteed by the architecture, not by tribal knowledge or code review vigilance.

This is what good static analysis tooling feels like at its best: not a list of style warnings, but a machine-readable encoding of architectural invariants that the compiler checks on your behalf.

---

## Conclusion

None of these patterns are novel. Cursor-based pagination, async messaging, tag-based caching, interface-driven registries, and custom static analysis rules are all well-documented techniques. What made the difference was applying them in sequence, where each solution created the conditions for the next. Memory-efficient iteration enabled async dispatch. Async dispatch exposed the need for caching. Caching required precise invalidation. Invalidation required a discoverable registry. And the registry needed a compiler-level safety net to stay correct over time. The lesson is not any single pattern — it is that architectural improvements compound when each one is designed with the next constraint in mind.
