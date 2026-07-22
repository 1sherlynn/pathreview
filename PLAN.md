## Solution plan

**Issue:** [#71 — Implement a red-teaming test suite for the prompt injection defense](https://github.com/ascherj/pathreview/issues/71)
_(Tier 3, est. 7–10 hrs · branch `test/71-prompt-injection-red-team`)_

### Understand

**What the issue asks for:** an automated red-team suite that fires a curated set
of known prompt-injection attacks at the safety layer and verifies each is
blocked, running in CI on every PR that touches `safety/`.

**Root cause (from reproduction, `scripts/repro_issue_71.py`):** the defense in
`safety/prompt_defense.py` is far weaker than the issue implies. Every
instruction/role pattern is anchored to a leading newline — e.g.
`r"\n\s*(?:Ignore|Forget|Disregard|Override)"` — so any attack at the **start** of
the input, or phrased inline, never matches. Whole attack categories (jailbreak
personas, prompt-leak, encoding, Unicode homoglyphs, spaced delimiters) have no
pattern at all. `sanitize()` only strips template/angle-bracket characters and
leaves instruction-style attacks untouched.

**Expected vs. actual:**
- _Expected:_ known injection attacks are detected/blocked; benign text is not.
- _Actual:_ **12/12 reproduced attacks bypass** `is_injection_attempt()`;
  `sanitize("Ignore all previous instructions…")` returns the string unchanged.
  The existing `tests/unit/test_prompt_defense.py` is tautological (every
  "malicious" fixture is built to match a pattern the regex already contains) and
  already ships with **1 failing test** (`test_whitespace_variations_detected`).

**Scope decision:** because the issue requires attacks to actually be *blocked* and
they currently aren't, this is tests **plus** hardening the defense — not
tests-only. Wiring the (currently orphaned) `PromptDefense` into the request
pipeline is **out of scope** (separate concern / follow-up issue).

### Map

Files, functions, and modules involved — and the specific files I expect to touch:

| File | Role in the change | New? |
|------|--------------------|------|
| `safety/prompt_defense.py` | Code under test. Modify `INJECTION_PATTERNS` + `sanitize()`; add Unicode/whitespace normalization | modify |
| `tests/security/test_prompt_injection.py` | The red-team suite, `@pytest.mark.security`, parametrized over the attack files | new |
| `tests/security/conftest.py` | Fixture loader for the tuning / holdout / benign files; also generates mutation variants | new |
| `tests/fixtures/injection_attempts/` | `tuning/` (attacks I tune against), `holdout/` (written first, not consulted while writing patterns), `benign/` (legitimate resume snippets) | new dir |
| `tests/unit/test_prompt_defense.py` | Fix 1 failing test **and 4 that pass vacuously** (lines 145, 187, 237, 249) | modify |
| `.github/workflows/ci.yml` | Add a `security` job, unfiltered (runs on every PR) | modify |
| `scripts/repro_issue_71.py` | Reproduction evidence (already committed); delete before final PR | remove |

### Plan

Six concrete sub-tasks, in order (tests before implementation):

1. **Build the attack, holdout, and benign files, plus the loader.** One file per
   attack category under `tests/fixtures/injection_attempts/tuning/`
   (role-switching, instruction-override, jailbreak, delimiter-smuggling,
   prompt-leak, encoding/homoglyph). Write `holdout/` **in the same sitting**,
   before any regex work — same categories, different payloads — then don't open
   it again until step 5. Write `benign/` (legitimate resume/portfolio snippets,
   deliberately loaded with "system", "execute", "override") **before** touching
   any pattern. Add `tests/security/conftest.py` to load all three, and to
   generate mechanical variants of each attack (case-flipped, extra internal
   spacing, NFKC homoglyph swap) so patterns are tested against strings I didn't
   hand-write.
2. **Write the suite red-first.** `test_prompt_injection.py`, parametrized.
   Assertion semantics (resolved — see Risks): "blocked" means
   `is_injection_attempt(attack) is True`, plus a separate assertion that
   `sanitize()` removes the imperative. `PromptDefense` has no caller, so the
   suite can only prove detection, not enforcement. Each benign input asserts
   `False`. Confirm it fails, reproducing the gap as automated tests.
3. **Harden `is_injection_attempt()`.** Remove the `\n` anchors; add the missing
   categories; NFKC-normalize and collapse internal whitespace before matching so
   `System :`, `{ { }}`, and `｛｛｝｝` are caught. Match on sentence shape — command
   verb plus meta word, or direct address — not bare keywords. Iterate until
   `tuning/` and its generated variants pass.
4. **Tune against the benign examples** until false positives = 0 (the hard part),
   then harden `sanitize()` to neutralize delimiter/role classes while staying
   idempotent and without ever emitting a marker that wasn't in the input.
5. **Generalisation gate + repair the unit file.** Now open `holdout/` and run it
   **once**. Record the catch rate in the PR — that number, not a green suite, is
   the real result of this issue. Under ~80% means the patterns are overfitted to
   strings I wrote, so return to step 3 and generalise rather than adding
   payload-specific patterns. Then repair `tests/unit/test_prompt_defense.py`: fix
   `test_whitespace_variations_detected` (failing), and give real assertions to
   the four vacuous tests — `test_sanitize_multiple_template_delimiters:145`,
   `test_benign_mentions_not_flagged:187`, `test_code_blocks_handled:237` (no
   assert at all), `test_sanitize_with_mixed_delimiters:249`.
6. **Add CI + finalize.** New `security` job running `pytest tests/security -m
   security`, unfiltered — it runs on every PR, like every other job in
   `.github/workflows/ci.yml` (see Risks for why not path-filtered). The
   `security` marker is already registered in `pyproject.toml`, so no config
   change is needed. Run `make check && make test-unit && pytest tests/security`;
   remove the repro script; open the PR with the before/after bypass table, the
   holdout catch rate, and a note that enforcement is a follow-up issue.

### Inputs & outputs

- **Input to the fix:** arbitrary user-controlled text (resume text, GitHub bio,
  portfolio blurbs) — both the curated attack strings and benign snippets.
- **Signatures that change (in `safety/prompt_defense.py`):**
  - `PromptDefense.is_injection_attempt(text: str) -> bool` — behavior changes:
    de-anchor patterns, add categories, so it returns `True` on the reproduced
    bypasses. Signature stays the same.
  - `PromptDefense.sanitize(text: str) -> str` — behavior changes: neutralize
    role/separator markers, not just delimiters; must stay idempotent.
  - New private helper, e.g. `_normalize(text: str) -> str` — NFKC-normalize +
    collapse internal whitespace, called before matching.
  - Data that changes: `INJECTION_PATTERNS` (expanded list) and the currently
    unused `DANGEROUS_CHARS` dict (wire in or remove).
- **Outputs / what the fix produces:** `is_injection_attempt` returns `True` for
  every attack file and `False` for every benign example; `sanitize` returns
  text with injection markers neutralized. Plus: new passing
  `tests/security/test_prompt_injection.py`, a green `tests/unit/` (including the
  four tests that currently pass vacuously), and a CI check guarding the injection
  defense. One further output that isn't a pass/fail: the **holdout catch rate**,
  reported in the PR — the share of attacks the hardened patterns catch without
  having been tuned on them. **No** API/DB/schema changes and no change to the
  review request flow.

### Risks & unknowns

- **False positives** — _where:_ `is_injection_attempt` in
  `safety/prompt_defense.py`. Resume text legitimately contains "system",
  "execute", "override", "disregard", so de-anchoring the patterns risks flagging
  honest users. _Mitigation:_ match on sentence shape, not bare keywords — a
  command verb plus a meta word ("ignore … instructions", "disregard … prompt")
  or direct address ("you are now"). "I built a distributed system" describes
  work; an attack gives the model an order and refers to the conversation itself.
  _Investigation path:_ write `tests/fixtures/injection_attempts/benign/` **first**,
  deliberately loaded with the tricky words, and run every new pattern against it;
  any match fails CI. I can hit 100% on the versioned attack files because that
  set is fixed; I can't prove zero false positives on all resume text, so I claim
  only what `benign/` checks.
- **`execute(` pattern is two-sided** — _where:_ the `(?:execute|run|eval)\s*\(`
  entry in `INJECTION_PATTERNS`. It misses real calls (`exec(`, `os.system(`) and
  false-positives on benign "I execute (on schedule) projects". _Path:_ decide
  whether to anchor to known dangerous callables or drop it.
- **Overfitting — the suite could end up as tautological as the one it replaces**
  — _where:_ the loop between `tests/fixtures/injection_attempts/tuning/` and
  `INJECTION_PATTERNS`. This is the biggest risk in the issue. If I write the
  attacks and then tune regexes until those exact strings match, the suite proves
  "my regex matches my strings" — which is the same defect I found in
  `tests/unit/test_prompt_defense.py`, just with the order reversed. It would be
  green and meaningless. _Mitigation:_ (a) `holdout/`, written before any regex
  work and opened once, at step 5; (b) mechanical variants generated in
  `conftest.py` rather than hand-written. _Investigation path:_ the holdout catch
  rate is the measurement — if it lags the tuning set badly, the patterns are
  memorising payloads, and the fix is to generalise the pattern, never to add a
  payload-specific one.
- **Detect vs. neutralize** — _where:_ `is_injection_attempt` (returns `bool`) vs
  `sanitize` (returns `str`). **Resolved: the suite asserts detection.** Since
  `PromptDefense` is imported by nothing, there is no caller that could block
  anything, so "blocked" can only mean "detected" here. Tests assert
  `is_injection_attempt(attack) is True`, plus a separate assertion that
  `sanitize()` strips the imperative. The PR states plainly that enforcement is
  the follow-up issue, so a green suite isn't mistaken for a protected app.
- **CI path-scoping** — _where:_ `.github/workflows/ci.yml`. **Resolved: no path
  filter, the job runs on every PR.** Workflow-level `paths:` means the workflow
  never starts on unrelated PRs and so never reports back — a required check then
  hangs at "waiting for status" with no failure to click into. Job-level filtering
  (`dorny/paths-filter`) is safe but doesn't pay for itself: the suite is plain
  pytest with no service containers, and the helper job that decides whether to
  skip costs most of the runtime it saves. The deciding argument is correctness —
  the attack files live in `tests/fixtures/injection_attempts/` and the suite in
  `tests/security/`, neither under `safety/`, so a `safety/`-only filter would
  skip the security tests exactly when a new attack is added. _Residual risk:_
  once `PromptDefense` is wired into the pipeline, changes in `ingestion/` or
  `core/` could affect detection — another reason the filter would go stale.
  Revisit only if the suite gets slow, and then at the job level.
- **Unicode normalization (NFKC)** — _where:_ the new `_normalize()` helper. NFKC
  can distort some legitimate characters; it runs only on the injection-check path
  (not on stored user data), so blast radius is limited. _Path:_ verify against the
  benign examples.
- **Orphaned defense** — _where:_ `PromptDefense` is imported by nothing in
  `core/`/`api/` (confirmed via grep). A green suite proves the component works,
  not that the app is protected; wiring it in is a separate issue — call this out
  in the PR.
- **"Block all attacks" is unbounded** — injection is adversarial and never
  "done"; I scope to the documented, version-controlled attack files under
  `tests/fixtures/injection_attempts/` and claim only that set is blocked.

### Edge cases

Inputs/states the fix must handle gracefully:
- Empty string and whitespace-only input → not flagged.
- Attack at the **start** of input (no leading newline) → flagged.
- Attack phrased **inline** mid-sentence → flagged.
- `System :` with spaces before the colon → flagged.
- Spaced delimiters `{ { 7*7 } }` and Unicode fullwidth `｛｛7*7｝｝` → flagged.
- Multi-line payloads (attack split across newlines) → flagged.
- Benign text using "system"/"execute"/"override" innocently → **not** flagged.
- `sanitize()` run twice → same result (idempotent).
- `sanitize()` must not **create** a marker by deleting text: `"IGIGNORENORE"`
  must not become `"IGNORE"`, and `"{{{{x}}}}"` must not collapse into a fresh
  `{{…}}`. Removal has to be single-pass or re-checked, not naive `.replace()`.
- Very long input (a 100KB pasted resume) → completes quickly; no catastrophic
  backtracking from the new patterns.
