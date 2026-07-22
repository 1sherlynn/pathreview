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

## Week 8 — Reproduction & solution planning

**Reproduction commit link:** https://github.com/1sherlynn/pathreview/commit/ff6066e9e4ed8ca92237b8ee0f426dc1608a9188

**Reproduction summary:**
I wrote `scripts/repro_issue_71.py` to fire known injection payloads at
`PromptDefense`; **12/12 attacks bypassed `is_injection_attempt()`** (and
`sanitize()` left the instructions intact) because the detection patterns are
anchored to a leading newline, so start-of-input and inline attacks never match.

**PLAN.md link:** https://github.com/1sherlynn/pathreview/blob/test/71-prompt-injection-red-team/PLAN.md

**Walkthrough video (recommended):** None

**Blockers or open questions:**

_Open — false positives._ The real risk isn't missing an attack, it's flagging a
real applicant. Resumes legitimately say "system", "execute", "override". But
reproduction made me realise the keyword isn't the signal — sentence shape is.
"I built a distributed system" describes work; "Ignore all previous instructions"
gives the model an order and refers to the conversation itself. So I'll match on
command-verb-plus-meta-word ("ignore … instructions") and direct address ("you
are now") rather than bare keywords. To keep this honest I'm writing the benign
examples _first_, deliberately stuffed with the tricky words, and wiring them
into the suite so any benign match fails CI — that way "how strict" gets answered
by a test run instead of my gut. I can hit 100% on my versioned attack list
because it's a fixed set; I can't prove zero false positives on all resume text
ever, so I'll only claim what the benign examples check.

_Decided — CI runs on every PR, no path filter._ I'd planned to scope the job to
`safety/` and dug into it instead. Filtering at the workflow level (`on:
pull_request: paths:`) means the workflow never starts on unrelated PRs, so it
never reports back to GitHub — a required check then hangs at "waiting for
status" with no red X to click, and the documented fix is a second dummy workflow
just to answer. Filtering at the job level (`dorny/paths-filter`) is safe but
pointless here: my suite is plain pytest with no database, so it finishes in
under a minute, and the helper job that decides whether to skip costs ~20s of
that. The deciding argument was correctness, not mechanics — my attack examples
live in `tests/fixtures/injection_attempts/` and the suite in `tests/security/`,
neither of which is under `safety/`, so a `safety/`-only filter would skip the
security tests exactly when I add a new attack. It also goes stale: once
`PromptDefense` is wired into the pipeline (out of scope here), a change in
`ingestion/` could break detection and the filter would quietly skip the check
that caught it. The issue asks for it to run "on every PR that touches
`safety/`" — running on every PR is a superset. Cost of being wrong either way is
lopsided: 40 extra seconds vs. a stuck PR or a security check that silently does
nothing. Revisit only if the suite gets slow, and then at the job level.

_Correction from Week 7:_ I claimed "zero test coverage," but reproduction showed
a (tautological) unit test file already exists — the red-team suite #71 asks for
still doesn't.
