# The OODA Loop in Software Development: When the Right Tool Creates the Wrong Design

In military strategy, the OODA loop — **Observe, Orient, Decide, Act** — describes how to process information and adapt faster than your opponent. It translates well to software development, particularly when iterating on design decisions.

This article walks through a concrete example where using the "correct" tooling led to a subtle but significant architectural problem — and how recognizing it early produced a simpler, better design.

---

## The Feature

A **profile completion score** aggregates data from several independent sources: contact info, biography, skills, preferences, experiences, and so on. Each source is fetched by a dedicated use case. The score is expensive to compute, so it must be **cached** and **invalidated** whenever a contributing field changes.

---

## First Iteration — Using Standard Tooling

The codebase provides declarative `#[Cache]` and `#[InvalidateCache]` attributes. The natural approach is to reach for them.

- Tag all data-fetching use cases with a service tag (`datafield.profile_score`)
- Inject them into `GetProfileScore` via a tagged iterator
- Cache the result under **two tags**: one per userId, one per full request (which encodes the datafields list)
- On each mutation use case, declare `#[InvalidateCache]` pointing to the userId tag

```php
// GetProfileScoreRequest — __toString() encodes userId + hash of datafields list,
// so the cache auto-invalidates when the list of datafields changes
public function __toString(): string
{
    return sprintf('profile_score_%d_%s', $this->userId, md5(implode(',', $this->datafields)));
}

// GetProfileScore — two cache tags: one to invalidate by userId, one to invalidate by request content
#[Cache(tags: ['"profile_score_" ~ request.userId', '"profile_score_" ~ request'])]
public function execute(GetProfileScoreRequest $request): ProfileScoreResponse { ... }

// GetContactInfo — tagged as a datafield
class GetContactInfo extends GetDatafieldUseCase implements UseCase, ProfileScoreDatafield { ... }

// UpdateContactInfo — invalidates the score cache by userId
#[InvalidateCache(tags: ['"contact_info_" ~ request.userId', '"profile_score_" ~ request.userId'])]
public function execute(UpdateContactInfoRequest $request): ContactInfoResponse { ... }
```

It works. Tests pass. Cache is invalidated on mutations, and also automatically on datafields list changes.

---

## Observe — Something Feels Off

Drawing the dependency graph reveals the problem:

```
GetProfileScore          ──► GetContactInfo, GetBiography, GetSkills, ...

GetContactInfo           ──► ProfileScoreDatafield (interface)
GetBiography             ──► ProfileScoreDatafield (interface)

UpdateContactInfo        ──► "profile_score_" cache tag
UpdateBiography          ──► "profile_score_" cache tag
```

**Knowledge about the profile score has leaked into every use case that participates in it.** `GetContactInfo` is a generic use case — it should not know it contributes to a score. Yet removing it from the score would require modifying the use case itself. A custom PHPStan rule had to be written just to enforce correct attribute usage across all these files.

This is a **bidirectional dependency**: `GetProfileScore` depends on the datafields, and the datafields depend on `GetProfileScore`.

---

## Orient — What Is the Real Constraint?

The real requirement is simple: **only `GetProfileScore` should know what it depends on**. The individual use cases should be completely unaware they participate in a score computation.

The standard tooling works well for genuine cross-cutting concerns. But it breaks encapsulation when used to couple unrelated use cases through a shared tag. The score feature's caching strategy is the score feature's responsibility — not a distributed one.

---

## Decide — Centralize, Don't Distribute

The decision: drop `#[Cache]` and `#[InvalidateCache]` for the score, handle caching manually inside `GetProfileScore`, and centralize all knowledge there.

- `DATAFIELDS` — static list of which use cases contribute to the score
- `MUTABLE_DATAFIELDS` — static list of which mutations should trigger invalidation
- An event subscriber wires invalidation without touching the mutation use cases

---

## Act — The Refactored Design

```php
class GetProfileScore implements UseCase
{
    private const array DATAFIELDS = [GetContactInfo::class, GetBiography::class, ...];
    public const array MUTABLE_DATAFIELDS = [UpdateContactInfo::class, UpdateBiography::class, ...];

    public function execute(GetProfileScoreRequest $request): ProfileScoreResponse
    {
        $cacheKey = self::getCacheKey($request->userId);
        $cached = $this->cache->fetch($cacheKey);

        if ($cached === false) {
            $cached = $this->computeScore->execute(...);
            $this->cache->save($cacheKey, $cached, ttl: 90 * 24 * 3600);
        }

        return $cached;
    }

    public static function getCacheKey(int $userId): string
    {
        // Including field names in the key ensures the cache auto-invalidates
        // whenever the DATAFIELDS list is updated (field added or removed).
        return sha1(sprintf('profile_score_%d_%s', $userId, implode(',', self::getFieldNames())));
    }
}
```

```php
class ProfileScoreEventSubscriber implements EventSubscriberInterface
{
    public static function getSubscribedEvents(): array
    {
        // Derived entirely from GetProfileScore — single source of truth
        return array_fill_keys(
            array_map(fn($c) => $c . 'Event', GetProfileScore::MUTABLE_DATAFIELDS),
            'onMutation'
        );
    }

    public function onMutation(object $event): void
    {
        $this->cache->delete(GetProfileScore::getCacheKey($event->request->userId));
        $this->getProfileScore->execute(new GetProfileScoreRequest($event->request->userId));
    }
}
```

Individual use cases now have **zero knowledge** of the score feature.

---

## Cache Invalidation on Schema Changes — Both Iterations Solve It Differently

Both iterations handle the case where a developer adds or removes a datafield — but through very different means.

The first iteration used a second cache tag derived from `request.__toString()`, which encoded the datafields list as a hash. Mutation use cases invalidated by the userId tag, while a datafields list change would naturally produce a new tag, orphaning the old entry.

The second iteration encodes the same information directly in the cache key:

```php
public static function getCacheKey(int $userId): string
{
    return sha1(sprintf('profile_score_%d_%s', $userId, implode(',', self::getFieldNames())));
}
```

Same guarantee, no `__toString()` magic, no second tag — and the logic lives in the one class that owns the concern.

---

## The Result

|  | First iteration | Second iteration |
|---|---|---|
| Datafield awareness of score | Yes — via interface | None |
| Mutation awareness of score | Yes — via cache tag | None |
| Knowledge ownership | Distributed across ~15 files | Centralized in one class |
| Enforcement overhead | 185-line PHPStan rule | Not needed |
| Net code change | additions only | more deletions than additions |

---

## Takeaway

The OODA loop here is tight:

1. **Observe** — the implementation works, but score knowledge leaks across 15+ use cases
2. **Orient** — the real constraint is unidirectional dependency; the score should own its knowledge
3. **Decide** — abandon idiomatic tooling for this specific case, accept a deliberate "non-standard" local solution
4. **Act** — centralize, delete the PHPStan rule, simplify

The lesson is not that declarative caching attributes are bad. They are excellent when a use case genuinely owns its caching concern. The lesson is that **pragmatism sometimes means resisting the available tooling** when applying it distributes concerns that should stay centralized.

A feedback loop that leads you to delete more code than you write, remove an enforcement rule, and simplify a design is a feedback loop worth trusting.
