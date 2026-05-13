# Contributing to SmartCart

## Branch Strategy

| Branch pattern | Purpose |
|----------------|---------|
| `main` | Always deployable. Protected. No direct pushes. |
| `feature/<issue>-<description>` | All new work |
| `fix/<issue>-<description>` | Bug fixes |
| `chore/<issue>-<description>` | Tooling, config, cleanup |
| `docs/<issue>-<description>` | Documentation only |
| `release/v<semver>` | Release preparation |

## Workflow

1. Create a GitHub Issue before starting any work
2. Branch from `main` — never from another feature branch
3. Make small, focused commits following Conventional Commits
4. Open a PR — link it to the issue
5. CI must be green before merge
6. Squash merge to keep `main` history clean
7. Delete the branch after merge

## Commit Message Format

    <type>(<scope>): <short imperative description>

    <body: WHY this change was made. Reference ADR if applicable.>

    Refs #<issue-number>

### Types

| Type | Use |
|------|-----|
| `feat` | New feature or behaviour |
| `fix` | Bug fix |
| `chore` | Tooling, config, no production code change |
| `docs` | Documentation only |
| `test` | Tests only, no production code change |
| `refactor` | Neither fix nor feat |
| `ci` | CI/CD pipeline changes |
| `perf` | Performance improvement |
| `style` | Formatting only, no logic change |

### Examples

    feat(auth): add JWT refresh token rotation with Redis TTL
    fix(order): prevent double-advance on saga state via @Version lock
    chore(platform): add commitlint and husky pre-commit hook
    docs(adr): add ADR-001 repository topology decision

## Code Standards

### Java
- Java 21. Records, sealed classes, pattern matching, virtual threads.
- No Lombok. No field injection. Constructor injection only.

### TypeScript
- Strict mode. No `any`. No `as` casts without explanatory comment.
- ESLint + Prettier enforced.

### Python
- Type hints throughout. Pydantic at all I/O boundaries.
- Ruff + Black enforced. No bare `except` clauses.

## Quality Bar

Every service must have before it is considered done:

- README, CHANGELOG, OpenAPI or AsyncAPI spec
- Dockerfile: multi-stage, distroless or slim base, non-root user
- `/health/live` and `/health/ready` endpoints
- Structured JSON logging with correlation IDs
- Prometheus `/metrics` endpoint
- OpenTelemetry tracing
- Unit tests with branch coverage >= 80%
- Integration tests using Testcontainers
- CI pipeline: lint -> test -> SonarQube -> build -> Trivy -> push

## Architecture Decisions

Any non-trivial technical decision requires an ADR in `docs/adr/`.
Use the template at `docs/adr/TEMPLATE.md`.
ADRs are numbered sequentially: ADR-001, ADR-002, etc.

## Money

NEVER use float or double for monetary values.

- Java: `java.math.BigDecimal`
- TypeScript: `Decimal` from `decimal.js`
- Python: `decimal.Decimal`

All amounts are in INR. Scale: 4 decimal places. HALF_UP rounding.
JSON wire format: amount as STRING e.g. "1250.0000".
