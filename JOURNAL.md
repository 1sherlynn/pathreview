# Module 3 Journal

A running record of my Module 3 work on PathReview.

## Week 7 — Issue selection

**Issue link:** https://github.com/ascherj/pathreview/issues/71

**Issue title:** Implement a red-teaming test suite for the prompt injection defense

**Tier:** [ ] Tier 1  [ ] Tier 2  [x] Tier 3

**Problem summary:**
PathReview has a `PromptDefense` class in `safety/prompt_defense.py` that is
supposed to catch prompt-injection attempts in user input, but nothing actually
proves it works — `tests/security/` contains only an empty `__init__.py`, so the
defense has zero test coverage. The current defense is also thin: it strips a few
template/HTML characters in `sanitize()` and matches a short hand-written list of
regex patterns in `is_injection_attempt()`, which won't catch many well-known
attack styles. My task is to build an automated red-teaming suite that throws a
curated set of real injection attacks (role-switching, "ignore previous
instructions", template/Jinja injection, code-execution attempts, etc.) at the
safety layer and asserts every one is blocked. A successful fix gives us
`tests/security/test_prompt_injection.py` plus attack fixtures under
`tests/fixtures/injection_attempts/`, wired to run in CI on any PR that touches
`safety/`, so future changes to the defense can't silently regress.

**Branch name:** test/71-prompt-injection-red-team

**Setup confirmation:** [x] App runs locally at localhost:5173

**Cohort ledger:** [x] Issue added to cohort ledger

### "Is this right for me?" — scope reasoning

- **Tier / effort:** This is a Tier-3 issue (est. 7–10 hrs), the most ambitious
  tier. I chose it deliberately over a quick Tier-1 fix because it's *additive* —
  I'm writing a new test suite and fixtures, not rewriting production logic, so
  the blast radius is low even though the effort is higher.
- **Do I understand the code it touches?** Yes — it's contained to
  `safety/prompt_defense.py` (the code under test) and new files under
  `tests/security/` and `tests/fixtures/injection_attempts/`. I've read
  `prompt_defense.py` and understand its two entry points (`sanitize` and
  `is_injection_attempt`).
- **Clear definition of done?** Yes — a curated attack set where every attack is
  asserted blocked, plus a CI trigger scoped to `safety/`. Easy to tell when it's
  finished.
- **Risk I'm watching:** the suite may surface attacks the current defense does
  *not* block. If so, I'll document those as findings rather than silently
  weakening the tests, and decide with the issue whether hardening the defense is
  in scope.
