# PLAN.md — Issue #71: Red-teaming test suite for the prompt injection defense

**Issue:** https://github.com/ascherj/pathreview/issues/71
**Branch:** `test/71-prompt-injection-red-team`
**Tier:** 3 (est. 7–10 hrs)

---

## 1. What the issue asks for

Add an automated red-team test suite that fires a curated set of **known prompt
injection attacks** at the safety layer (`safety/prompt_defense.py`) and verifies
each one is blocked. The suite must run in CI on every PR that touches `safety/`.
Target files named in the issue:

- `tests/security/test_prompt_injection.py`
- `tests/fixtures/injection_attempts/`

---

## 2. Reproduction (done — Week 8)

Reproduced locally with `scripts/repro_issue_71.py` (committed for evidence).

**Findings — the defense is much weaker than the issue implies:**

1. **12 / 12 real-world attacks BYPASS `is_injection_attempt()`** (returns
   `False`). Examples that slip through: `"Ignore all previous instructions..."`
   at the start of input, inline `"...ignore the previous instructions..."`,
   `"System: ..."` at the start, DAN-style jailbreaks, prompt-leak requests,
   spaced delimiters `{ { 7*7 } }`, and Unicode fullwidth braces `｛｛7*7｝｝`.
2. **Root cause:** every instruction/role pattern is anchored to a leading
   newline (e.g. `r"\n\s*(?:Ignore|Forget|Disregard|Override)"`), so any attack
   at the **start** of the string, or phrased inline, never matches. Whole attack
   *categories* (jailbreak personas, prompt-leak, encoding, homoglyphs) have **no
   pattern at all**.
3. **`sanitize()` only strips template/angle-bracket characters** — it leaves
   instruction-style injections completely intact:
   `sanitize("Ignore all previous instructions...")` returns the string unchanged.
4. **The existing `tests/unit/test_prompt_defense.py` is tautological** — every
   "malicious" fixture is hand-built to match a pattern the regex already
   contains, so it proves nothing about real robustness. It also ships with **1
   pre-existing failing test** (`test_whitespace_variations_detected`: `System :`
   with spaces before the colon isn't matched).
5. **No red-team suite exists** — `tests/security/` contains only `__init__.py`,
   `tests/fixtures/injection_attempts/` does not exist, and `.github/workflows/ci.yml`
   has no job or path-trigger for `safety/`.

> **Correction from Week 7:** my Week 7 problem summary said the defense had
> "zero test coverage." Reproduction proved that wrong — there *is* a unit test
> file, it's just tautological and lives in `tests/unit/`, not the red-team suite
> `tests/security/` that #71 asks for. Documenting the gap between plan and
> reality rather than rewriting history.

---

## 3. Scope decision

The issue says "verifies that all are blocked," but reproduction shows the
current defense blocks almost none of the curated attacks. So this issue is **not
tests-only** — to make the suite pass, the defense itself must be hardened. Scope:

**In scope**
- A curated, **versioned** corpus of known-attack fixtures, organized by category.
- A parametrized red-team suite asserting every corpus attack is blocked, plus a
  benign corpus asserting **no false positives**.
- Hardening `safety/prompt_defense.py` (`INJECTION_PATTERNS` + `sanitize`) enough
  to block the curated corpus without flagging the benign corpus.
- A CI job that runs the suite, triggered on PRs touching `safety/`.
- Fixing the pre-existing failing unit test so the suite is green.

**Out of scope (note in PR, possibly file follow-up issues)**
- "Solving" prompt injection in general — the corpus is a fixed, documented set,
  not a claim of completeness.
- **Wiring `PromptDefense` into the review pipeline** — it is currently orphaned
  (called nowhere). Testing it in isolation is still valuable; integrating it is a
  separate concern and a separate issue.

---

## 4. Files to change

| File | Change | New? |
|------|--------|------|
| `tests/fixtures/injection_attempts/` | Attack corpus + benign corpus (see §5) | new dir |
| `tests/security/test_prompt_injection.py` | Parametrized red-team suite, `@pytest.mark.security` | new |
| `tests/security/conftest.py` | Fixture-loader that reads the corpus | new |
| `safety/prompt_defense.py` | De-anchor patterns, add attack categories, Unicode-normalize + neutralize in `sanitize()` | modify |
| `tests/unit/test_prompt_defense.py` | Fix the pre-existing failing test; align with new behavior | modify |
| `.github/workflows/ci.yml` | Add `security` job scoped to `safety/` changes | modify |
| `scripts/repro_issue_71.py` | Reproduction evidence (already committed); delete before final PR | remove later |

---

## 5. Sub-tasks (in order)

1. **Design the corpus format.** One file per attack category under
   `tests/fixtures/injection_attempts/` (e.g. `role_switching.txt`,
   `instruction_override.txt`, `jailbreak.txt`, `delimiter_smuggling.txt`,
   `prompt_leak.txt`, `encoding_homoglyph.txt`), one payload per line, plus a
   `benign/` subfolder of legitimate resume/portfolio snippets. Document where
   each attack came from (OWASP LLM01, known jailbreak lists).
2. **Write the fixture loader** (`tests/security/conftest.py`) that yields
   `(category, payload)` tuples for attacks and benign inputs separately.
3. **Write the failing suite first (red).** `test_prompt_injection.py`
   parametrized: every attack asserts `is_injection_attempt(...) is True`; every
   benign input asserts `is_injection_attempt(...) is False`. Confirm it fails,
   reproducing the gap as automated tests.
4. **Harden `is_injection_attempt`.** Remove the `\n` anchors; add case-insensitive
   detection for the missing categories; Unicode-normalize (NFKC) before matching
   so homoglyph/fullwidth attacks collapse to ASCII; collapse internal whitespace
   so `System  :` and `{ { }}` are caught. Iterate until the attack corpus is green.
5. **Tune against the benign corpus.** Add/adjust benign snippets that use words
   like *system*, *execute*, *disregard*, *override* in innocent contexts; tighten
   patterns until false positives = 0. This is the hard part.
6. **Harden `sanitize()`** so it neutralizes (not just detects) the delimiter and
   role-marker classes; keep it idempotent (there's a test for that).
7. **Fix `test_whitespace_variations_detected`** and reconcile the unit tests with
   the new, stricter behavior.
8. **Add the CI job.** New `security` job in `ci.yml` running
   `pytest tests/security -m security`. Trigger only when `safety/` (or the suite)
   changes — see risk R4 for the mechanism.
9. **Full local gate:** `make check && make test-unit && pytest tests/security -v`.
   Remove `scripts/repro_issue_71.py`. Open PR with the reproduction table.

---

## 6. Risks & edge cases (already visible)

- **R1 — False positives are the real difficulty.** Resume/portfolio text
  legitimately contains "system", "execute", "override", "disregard". Stricter
  detection risks flagging honest users. Mitigation: mandatory benign corpus in
  the suite; false positives fail CI just like misses do.
- **R2 — "Block all attacks" is unbounded.** Prompt injection is adversarial and
  never "done." Mitigation: freeze scope to a **documented, versioned corpus**;
  the PR claims "blocks this curated set," not "is unbreakable."
- **R3 — Detect vs. neutralize.** `is_injection_attempt` (boolean flag) and
  `sanitize` (transform) are two different defenses. Decide per test what "blocked"
  means; likely assert detection for the suite, and separately assert `sanitize`
  neutralizes delimiter-class payloads.
- **R4 — CI path-scoping is a GitHub Actions gotcha.** Workflow-level `paths:`
  filters skip the *whole* workflow, and a "skipped" required check can block
  merges. Job-level path filtering needs `dorny/paths-filter` (or running the suite
  unconditionally and accepting it's fast). Decide during sub-task 8; leaning
  toward running it always since unit-speed tests are cheap.
- **R5 — Unicode normalization can distort legitimate content.** NFKC changes some
  valid characters. Since this runs only on the injection-check path (not on stored
  user data), the blast radius is limited, but note it.
- **R6 — Orphaned defense.** Because `PromptDefense` is wired into nothing, a green
  suite proves the component works, not that the app is protected. Call this out
  explicitly in the PR so it isn't mistaken for end-to-end coverage.

---

## 7. Definition of done

- [ ] `tests/security/test_prompt_injection.py` runs the full attack corpus; all
      curated attacks are blocked and all benign inputs pass.
- [ ] `safety/prompt_defense.py` hardened; the 12 reproduced bypasses now blocked.
- [ ] `tests/unit/test_prompt_defense.py` fully green (pre-existing failure fixed).
- [ ] CI runs the security suite on PRs touching `safety/`.
- [ ] `make check && make test-unit && pytest tests/security` all pass locally.
- [ ] Reproduction script removed; PR describes the before/after with the bypass table.
