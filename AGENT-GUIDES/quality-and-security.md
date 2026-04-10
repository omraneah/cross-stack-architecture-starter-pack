# Agent Guide: Quality & Security Enforcement

**ARD Source:** `quality-security-boundaries.md`
**Scope:** All stacks — backend, frontend, mobile, infrastructure

---

## Why This Boundary Exists

**Concrete failure modes this boundary prevents:**

1. **Silent quality degradation over time.** Without automated gates, quality degrades one small compromise at a time — a test skipped here, a lint warning ignored there. Each individual bypass seems minor. After six months, the codebase has hundreds of suppressed warnings, flaky tests left in place, and no one knows where the quality floor is.

2. **Security vulnerability merged under review pressure.** A reviewer approving a PR with a known high-severity vulnerability (e.g., a vulnerable dependency, a security misconfiguration) under deadline pressure makes a calculation: "I'll fix it later." Later never comes. The vulnerability is exploited or found in a security audit months after the merge.

3. **Local checks pass but CI fails — development friction.** When local pre-commit hooks check different rules than CI, developers discover CI failures only after pushing. The feedback loop is long. Engineers work around it by pushing more commits to fix CI, which adds noise to the git history and delays merges.

4. **Human reviewer enforcing mechanical rules.** A code reviewer who manually checks variable naming, test coverage thresholds, or import style is doing automated work. This wastes review capacity on mechanics instead of intent, architecture, and correctness. And it is inconsistent — different reviewers have different thresholds.

5. **Flaky tests creating false confidence.** A test suite with flaky tests that are periodically skipped or re-run until passing creates false confidence. Engineers merge code trusting the tests passed, but the tests were not actually passing — they were being ignored.

---

## The Mental Model

Quality and security enforcement is a **pipeline with three layers**, each with a distinct role:

```
Layer 1: Pre-commit hooks
  → Catches issues before commit
  → Fast feedback (runs in seconds)
  → Developer's last personal gate

Layer 2: CI / PR workflows
  → Authoritative merge gate
  → Runs same checks as pre-commit
  → Cannot be bypassed by any individual

Layer 3: System-level gates
  → Long-term health visibility
  → Cross-project trend monitoring
  → Detects gradual degradation
```

**CI is the authority.** Pre-commit is convenience. System-level gates are monitoring. None of the three can be substituted for the others.

---

## Invariants

**1. CI is the only authority for merge decisions.**

- **Violation:** A PR is merged with failing checks under the justification "it's a minor issue I'll fix in the next PR."
- **Cost:** Every exception teaches the team that the gates are negotiable. Within months, the gates are effectively advisory. Quality degrades rapidly once the psychological contract is broken.

**2. Failing tests block merge — no exceptions.**

- **Violation:** A test is marked with `.skip` or `xtest()` without a tracked follow-up issue, and the PR is merged.
- **Cost:** The skipped test no longer protects the behavior it was testing. The behavior can be broken in future changes with no automated warning. The test debt accumulates.

**3. High and critical security vulnerabilities block merge.**

- **Violation:** A dependency with a known critical CVE is added or a known security misconfiguration is introduced, and the PR is approved because "we'll fix the security issue separately."
- **Cost:** The vulnerability is now in production. The "separate fix" PR is deprioritized. The vulnerability window extends from hours to weeks.

**4. Pre-commit hooks and CI run the same categories of checks.**

- **Violation:** Pre-commit runs only formatting, but CI also runs tests and security scans. Developers experience no test failures locally and discover them only after pushing.
- **Cost:** The development loop degrades. More commits are needed per PR to fix CI. Engineers learn to ignore local hooks because they don't represent reality.

**5. Pre-commit hooks cannot be bypassed.**

- **Violation:** An engineer uses `--no-verify` to commit, bypassing hooks, because they are in a hurry.
- **Cost:** The bypassed commit may introduce a quality or security issue. The hook's purpose — catching problems early — is undermined. If bypass becomes a habit, hooks are effectively disabled.

**6. Security soft-fail mode is a temporary state, not a permanent setting.**

- **Violation:** A security scan is configured with `soft_fail: true` permanently because addressing existing findings is deemed too costly.
- **Cost:** New security issues are introduced with no gate blocking them. The soft-fail setting accumulates technical debt. When a breach occurs, the audit trail shows that the security gate was explicitly disabled.

---

## Decision Protocol

**IF you are adding a new check or rule:**
→ THEN add it to both pre-commit hooks and CI in the same PR. They must stay aligned.
→ CHECK: After this change, do pre-commit and CI check the same set of rules? If not, align them.

**IF a test is flaky and you are tempted to skip it:**
→ THEN investigate the flakiness cause. If you cannot fix it now, document it in a tracked issue and leave the test enabled but note the tracking. Do not `.skip` without a traceable follow-up.
→ CHECK: Is there a tracked issue for this test's flakiness? If not, create one before touching the test.

**IF a security scan reports a high or critical finding:**
→ THEN it must be resolved before the PR is merged. "I'll fix it later" is not acceptable. If the finding is a false positive, add an explicit suppression with a comment explaining why.
→ CHECK: Is the finding a real vulnerability? If yes, fix it. If it is a documented false positive, add a suppression comment. Never ignore it silently.

**IF you are setting up a new project:**
→ THEN configure pre-commit hooks and CI workflows before writing any feature code. The quality gates are part of project setup, not an afterthought.
→ CHECK: Does the project have pre-commit hooks? Does it have a CI pipeline? Are they checking the same things? All three must be yes before first feature code.

**IF a PR check is failing:**
→ THEN fix the underlying issue. Do not disable the check, exclude the file, or add a suppression unless it is a documented false positive.
→ CHECK: Why is the check failing? Is it a real issue or a tooling misconfiguration? Fix the real issue or fix the tooling — do not bypass it.

---

## Generation Rules

**Always include (when generating project setup or CI configuration):**
- Pre-commit hook configuration that runs: linting, formatting, tests (or test hook), security scan
- CI workflow that runs the same categories: linting, formatting, tests, security scans, IaC validation (if applicable)
- Security scan with hard-fail for high and critical findings (no `soft_fail: true` as permanent state)
- Test step that is merge-blocking (no `continue-on-error: true` on tests)
- IaC workflows: `fmt -check`, `init`, `validate`, security scan before plan/apply

**Never generate:**
- CI configuration with `continue-on-error: true` on test steps
- Security scan configuration with `soft_fail: true` as a permanent setting
- Pre-commit hook configuration that checks fewer things than CI
- Committed code with `// eslint-disable` or equivalent without an explanatory comment
- Tests with `.skip` or `xit` without a tracked issue reference in a comment

**Verify before outputting:**
- Does CI have a merge-blocking test step? → Add if not.
- Does CI have a high/critical security scan without soft-fail? → Add if not or remove soft-fail.
- Are pre-commit hooks aligned with CI checks? → If CI checks more things, add them to pre-commit.
- Are any test skips in the code accompanied by a tracked issue? → If not, flag them.

---

## Self-Check Before Submitting

1. Do pre-commit hooks and CI run the same categories of quality checks? → If not, align them.
2. Are all tests passing (none skipped without tracked issues)? → If not, investigate and fix.
3. Are high and critical security findings resolved or explicitly documented as false positives? → If not, fix before submitting.
4. Is any CI check configured with `continue-on-error`, `soft_fail`, or equivalent bypass? → If yes, remove the bypass.
5. Is any new check added to CI but not to pre-commit hooks? → If yes, add it to pre-commit hooks.
6. Is any linting or formatting suppression added without an explanatory comment? → If yes, add the explanation.
7. Is the security gate in the IaC pipeline configured to hard-fail on high/critical findings? → If in soft-fail mode, flag as tech debt requiring resolution.

---

## Common Violations and Corrections

**Violation 1:** Test skipped without tracked issue.
```typescript
// WRONG
it.skip('should return tenant-scoped data', () => { ... });
```
**Correction:** Either fix the test, or leave it enabled and create a tracking issue. If a temporary skip is unavoidable, reference the issue in the test file.
```typescript
// ACCEPTABLE TEMPORARY STATE (with tracking)
// TODO: [ISSUE-123] Re-enable when async timing issue is resolved
it.skip('should return tenant-scoped data', () => { ... });
```

---

**Violation 2:** Security gate set to soft-fail permanently.
```yaml
# WRONG — permanent soft-fail hides real security issues
- name: Security scan
  uses: security-scanner-action@v1
  with:
    soft_fail: true
```
**Correction:** Set `soft_fail: false`. Address the existing findings. If findings cannot be addressed immediately, track each one as a known issue and resolve them in a dedicated sprint.

---

**Violation 3:** Pre-commit hooks only run formatting; CI also runs tests and security.
**Correction:** Update pre-commit configuration to include the same test and security scan steps that run in CI. Developers discover test failures locally before pushing.

---

**Violation 4:** ESLint rule disabled without explanation.
```typescript
// WRONG
// eslint-disable-next-line @typescript-eslint/no-explicit-any
const data: any = response;
```
**Correction:** Either fix the underlying type issue (preferred), or if the disable is genuinely necessary, add a comment explaining why and what the acceptable risk is.

---

**Violation 5:** PR merged with a failing lint check because "it's just a style issue."
**Correction:** No merger rationalizations. All failing checks block merge. A "style issue" that passes in review today becomes a pattern tomorrow. The CI gate is the contract — the human reviewer validates intent, not style.
