# A Practical Guide to Conventional Commits

Conventional Commits is a lightweight specification for writing structured commit messages. It makes changelogs easier to generate, history easier to read, and versioning easier to automate.

## The Format

```
<type>(<scope>): <message>
```

- **type** — what kind of change this is (required)
- **scope** — where in the codebase (optional)
- **message** — short imperative description (required)

Examples:

```
feat(auth): add OAuth2 login
fix(cart): correct total price calculation
chore: update composer dependencies
```

---

## The Spec: Only Two Official Types

The [Conventional Commits spec](https://www.conventionalcommits.org/) formally defines only two types:

- `feat` — adds user-facing behavior → triggers a **minor** semver bump
- `fix` — corrects a bug → triggers a **patch** semver bump

Everything else comes from the **Angular commit convention**, widely adopted as a de facto standard.

---

## The Angular Type Vocabulary

| Type | Purpose |
|---|---|
| `feat` | New feature |
| `fix` | Bug fix |
| `docs` | Documentation only |
| `style` | Formatting, whitespace (no logic change) |
| `refactor` | Restructure without feature/fix |
| `perf` | Performance improvement |
| `test` | Adding or fixing tests |
| `build` | Build system, dependencies |
| `ci` | CI/CD pipeline changes |
| `chore` | Other maintenance tasks |
| `revert` | Reverts a previous commit |

---

## `feat` vs `chore`

The key question: **would a user or API consumer notice this change?**

- **Yes** → `feat` (or `fix`)
- **No** → `chore`

```
feat: add job posting search index     ← new capability for users
chore: update CI node version          ← internal, invisible to users
```

`refactor` and `perf` are worth distinguishing from `chore` — they signal *why* the code changed without implying a visible behavior change.

---

## Scopes: Optional but Valuable

Scopes answer *"where?"* — a module, domain, or layer name.

```
feat(search): add faceted filtering
fix(user): prevent duplicate email registration
ci(docker): pin base image version
```

**Without scope** is equally valid when the context is obvious or the repo is small.

### Tips

- Be consistent — partial use of scopes is worse than none
- Keep them short: `auth`, `api`, `db`, `user`, `payment`
- Consider enforcing an allowed list in `commitlint` on larger projects

---

## Custom Types

The spec allows custom types. If your team maintains a content repo, `content(article): add guide to conventional commits` is valid — as long as your team aligns on the vocabulary and your linter allows it.

Only `feat` and `fix` carry reserved meaning (semver). Everything else is convention.

---

## Breaking Changes

Any type can signal a breaking change with a `!` or a `BREAKING CHANGE:` footer:

```
feat!: remove deprecated v1 endpoints

BREAKING CHANGE: /api/v1 is no longer available, migrate to /api/v2
```

This triggers a **major** semver bump regardless of type.

---

## Summary

| | Spec | Angular convention |
|---|---|---|
| Required types | `feat`, `fix` | + 9 more |
| Scope | Optional | Optional |
| Custom types | Allowed | Allowed |
| Breaking changes | `!` or footer | Same |

Start with the Angular vocabulary, add scopes when your repo grows, and define custom types only when none of the standard ones fit.
