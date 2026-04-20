# 05 — Conventions

> Project-wide rules. Short, enforced, boring on purpose.

---

## 1. Code style

### .NET
- C# 13, nullable reference types **on** everywhere
- `ImplicitUsings` on
- `file`-scoped namespaces
- Use `record` for DTOs, `sealed class` for entities and services
- Guard clauses with `ArgumentException.ThrowIfNullOrWhiteSpace` / `ArgumentNullException.ThrowIfNull`
- `Guid.CreateVersion7()` for all new primary keys (lexicographically sortable, better for indexes)
- No `async void`, no `.Result`, no `.Wait()`
- Format on save: `dotnet format`

### TypeScript / Svelte
- Strict mode on (`"strict": true` in `tsconfig.json`)
- ESLint + Prettier, format on save
- Prefer `type` over `interface` except when declaration merging needed
- No `any`. Use `unknown` and narrow
- Svelte 5 runes syntax (`$state`, `$derived`, `$effect`)

### Common
- 2 spaces indent (TS/JSON/YAML), 4 spaces (C#)
- LF line endings, UTF-8, trailing newline
- `.editorconfig` committed to repo

---

## 2. Git

### Branches
- `main` — deployable
- `feat/<short-name>` — features
- `fix/<short-name>` — bug fixes
- `chore/<short-name>` — tooling, deps, docs

### Commits (Conventional Commits)
```
<type>(<scope>): <subject>

[body]

[footer]
```

Types: `feat`, `fix`, `chore`, `docs`, `refactor`, `test`, `perf`, `build`, `ci`.

Examples:
```
feat(sites): add province code lookup endpoint
fix(auth): handle missing realm_access claim from lightweight tokens
chore(deps): bump Keycloak.AuthServices 2.9.1 → 2.9.2
```

Body explains **why**, not what. The diff shows the what.

### PRs
Optional for solo work, but if used: PR title = commit-style summary. Description lists the user-visible change and any migration/ops implications.

---

## 3. Database migrations

- One logical change per migration
- Never edit a migration after it has run anywhere outside your laptop
- If production has drifted, add a new corrective migration, don't rewrite history
- Name migrations descriptively: `AddSiteTierColumn`, not `Update1`
- Every migration must be reversible (`Down` implemented) unless the data loss is intentional and documented

```bash
# Add
dotnet ef migrations add AddSiteTierColumn \
    --project src/JetX.Infrastructure \
    --startup-project src/JetX.Api

# Remove (before commit)
dotnet ef migrations remove \
    --project src/JetX.Infrastructure \
    --startup-project src/JetX.Api

# Bundle for prod
dotnet ef migrations bundle --force \
    --project src/JetX.Infrastructure \
    --startup-project src/JetX.Api \
    -o build/migrator/efbundle
```

---

## 4. Testing policy

- Domain: unit test every non-trivial method, especially invariants
- Application: integration test every handler against a real PostgreSQL via Testcontainers
- Api: smoke test the critical endpoints with a fake auth scheme
- Frontend: component tests with Vitest + Testing Library for anything with logic, not for styling
- Target: every bug that reaches prod gets a regression test before the fix merges

Run locally:
```bash
dotnet test
npm test --prefix frontend
```

---

## 5. Secrets

### Never
- Commit secrets, even to a private repo
- Log tokens, passwords, or the `Authorization` header value
- Paste a production secret into a chat, ticket, or commit message
- Use the same client secret across dev/staging/prod

### Always
- Dev: `dotnet user-secrets` for .NET, `.env` (gitignored) for Node
- Prod: `/etc/jetx/*.env`, root-owned, group-readable by `jetx` (mode 640)
- Rotate if a secret is known to have leaked, and within 90 days otherwise
- Keep a password manager entry per production secret with rotation date

---

## 6. Logging

Log at these levels:

| Level       | When                                                              |
|-------------|-------------------------------------------------------------------|
| `Error`     | Unhandled exception, failed external call with retries exhausted  |
| `Warning`   | Expected but unusual: 4xx from dependency, quota near limit       |
| `Information` | One line per request (Serilog does this), app lifecycle events  |
| `Debug`     | Verbose; disabled in prod                                          |
| `Trace`     | Never in prod                                                      |

Always include correlation IDs. ASP.NET Core's `TraceIdentifier` is fine for starters; add `X-Request-Id` pass-through on Caddy for end-to-end tracing.

**Don't log:**
- Access tokens, refresh tokens, ID tokens, authorization headers
- User passwords (you won't have any anyway — Keycloak owns them)
- Full request bodies on write endpoints (risk of logging secrets)
- PII beyond username/email unless specifically needed for the log's purpose

---

## 7. API design

- REST. No GraphQL unless there's a real reason
- Plural resource nouns: `/api/sites`, `/api/cameras`
- `GET` list, `GET /{id}` detail, `POST` create, `PUT /{id}` replace, `PATCH /{id}` partial update, `DELETE /{id}`
- Return `ProblemDetails` (RFC 7807) for errors. ASP.NET Core does this with `AddProblemDetails`
- Paginate list endpoints: `?page=1&pageSize=50`, cap `pageSize` at 200
- Version via URL only when you must: `/api/v2/sites`. Prefer additive changes
- Every endpoint annotated with `.WithName()`, `.WithTags()`, and typed results for good OpenAPI output

---

## 8. Authorization rules

- Never check roles inside handlers. Use `RequireAuthorization("<policy>")` on endpoints
- Policies named by capability (`"sites:read"`) not by role (`"staff"`)
- Row-level filtering (e.g. "user can only see sites they're assigned to") lives in the Application layer, fed by `ICurrentUser`
- No "superuser bypass" flag. `platform-admin` role enforces itself

---

## 9. Documentation

- This `docs/` folder is canonical. Update it when behavior changes
- Each bounded context gets a short `README.md` in its folder explaining the domain vocabulary
- ADRs (architecture decision records) in `docs/adr/NNNN-title.md` for any decision that's hard to reverse. Use the short MADR format

---

## 10. Dependencies

- Pin exact versions in `package-lock.json` / `packages.lock.json`
- Monthly sweep: `dotnet list package --outdated`, `npm outdated`
- Major upgrades land in their own PR with explicit changelog review
- Prefer Microsoft/owner-maintained packages over community wrappers for core concerns (auth, data)
- Avoid abandoned packages (last release > 18 months, no open maintainer response)

---

## 11. What to avoid (JetX-specific)

- **Don't replicate Keycloak user data into `jetx_app`.** Store `keycloak_sub` (Guid) as a foreign key and look up display info on demand. The one exception is a periodic sync job if you need the username indexed for search
- **Don't reinvent permissions.** Every new capability is a new client role in Keycloak + one-line policy in .NET
- **Don't couple domain entities to Pi node IDs, camera stream URLs, or NVR paths.** Those live in infrastructure tables, joined by ID. The domain should survive a complete reorg of the physical fleet
- **Don't build a "framework" inside this framework.** If you find yourself writing `ISiteServiceBaseAbstractFactory<T>`, stop. Write the specific thing
- **Don't skip migrations in dev.** Make them part of the loop so they're never scary in prod
