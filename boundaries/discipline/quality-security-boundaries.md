# Quality & Security Boundaries

**Quality and security are enforced by automation. CI is the authority. Human review validates intent; it does not substitute for automated gates.**

**Applies when:** always — any codebase with a CI pipeline (and if there's no CI pipeline yet, that's the first finding).

## Why it matters

- Manual review degrades under deadline pressure; the same reviewer who blocked a security issue last week may approve a similar one this week.
- A pre-commit hook that runs different checks than CI creates surprises in pull requests and erodes developer trust.
- Security findings accepted silently — even with good intentions — accumulate until they are too expensive to fix.

## The judgment

**What's automated, what's reviewed:**

- Automated: linting, formatting, type checking, tests, dependency vulnerability scans, security static analysis, naming and structure checks.
- Reviewed: architectural intent, design alternatives, business correctness.

**Pre-commit ↔ CI alignment:**

- Pre-commit and CI run the same categories of checks. Pre-commit is for fast feedback; CI is for authority.
- A check that runs only in CI surprises developers at PR time. A check that runs only locally can be bypassed.

**Vulnerability handling:**

- High and critical findings block merge. No silent acceptance.
- Suppressions require an inline justification and a tracked remediation date. Open-ended ignores are forbidden.

**Override policy:**

- No manual override of CI. A failing check is a failing PR. Exceptions require an explicit, documented design decision, not a deadline.
- A 70% coverage gate degrades to a 70% coverage gate with `--ignore-pattern` exclusions for "flaky" files within 18 months. The gate without an audit of what's actually excluded is theatre.

## Signals of violation in an audited codebase

- A CI workflow with `continue-on-error: true` or `soft_fail: true` on a security check, without a tracked deadline to remove it.
- A pre-commit hook that runs only formatting; the heavy checks live in CI only.
- Suppression comments without justification or tracked tickets.
- A merge to main with failing required checks via admin override.
- A dependency advisory of high severity older than the team's stated remediation window.

## Minimum viable shape

```
Pre-commit hook ↔ CI workflow → same categories, same severity rules
High / critical security findings → merge blocker, no override
Suppressions → inline justification + tracked remediation date
Tests, lint, format, type-check → run on every PR
CI decision → final; no manual override path
```

**Severity floor if violated:** P1 — CI gate drift is invisible until a high-severity advisory ships to production. P0 if `continue-on-error` or `soft_fail` is set on a security check without a tracked deadline. May step down by one tier in pre-revenue internal tools.
