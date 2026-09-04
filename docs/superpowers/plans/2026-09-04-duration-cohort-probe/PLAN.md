# Duration Cohort Probe Implementation Plan

> **For agentic workers:** Run `/gflow:status --feature duration-cohort-probe` to find the
> next unchecked task. Implement one task at a time. Run `/gflow:check` before every commit.

**Goal:** When a Flow account renders no duration control, `--duration` fails in
milliseconds with a message that names cohort variance — instead of hanging ~40 s and
telling the user Google changed their frontend and to file a bug.

**Architecture:** One new probe and one new message constant, both inside
`api/transports/ui_automation_video.py`. `errors.py` gets **zero diff** — the cohort
wording is passed per-raise as `remediation_hint=`, following the existing
`ENTITY_ATTACH_DRIFT_HINT` pattern (27 such call sites repo-wide), because
`UiSelectorDriftError._default_remediation` is shared by all ten
`_probe_selector_cascade` callers and "re-run without `--duration`" is nonsense for
`image_mode_tab` or `count_tab`. No selector is scoped or deleted, and no exit code is added.

**Predict verdict:** 🟡 **CAUTION — 6/10** (2026-09-04). Average 8.0 across five personas,
downgraded 2 because the Devil's Advocate found a simpler approach the other four missed.

**Scenario:** `SCENARIO.md` in this directory — 14 scenarios, 3 Critical.

## Risk register

| Severity | Risk | Mitigation |
|---|---|---|
| **High** | A closed popover reads as "no duration control", turning a race into a false cohort verdict | Task 3 gates the zero on `[role='menu']` being present; S2 is a must-cover test |
| **High** | The original proposal (scope all 10 candidates) defends a collision the evidence rules out — the composer chip only carries `Ns` on cohorts that *have* the control | Dropped from scope. Not implemented. |
| Medium | Over-scoping or over-tightening creates a false-negative class on cohorts we have no account for | No selector is changed. The probe only decides *when to stop early*; the existing cascade is untouched. |
| Medium | The new message reaches MCP agents verbatim through the queued-job failure envelope | Task 5 writes it for both audiences and updates `docs/MCP.md:86`, which currently claims a 400 rather than a worker failure |
| Low | `remediation_hint` regression for the other nine cascade callers | `errors.py` untouched; Task 6 asserts the class default is unchanged |

**Deliberately out of scope**
- **Scoping the cascade to `[role='menu']`** — predict ruled the hazard out on the evidence.
- **The `aria-selected` read-back** — Task 7, gated on the $0 probe (below); not started.
- **`_set_output_count`** — same unscoped-cascade shape, and a wrong match there *doubles*
  spend. Filed as its own issue; it deserves its own predict round rather than riding along.

---

## File structure

### New files
```
docs/superpowers/plans/2026-09-04-duration-cohort-probe/SCENARIO.md
  (done) 14 scenarios across 8 active dimensions
tests/api/test_duration_cohort_probe.py
  Red tests for the probe, the message, and the ordering guarantee
```

### Modified files
```
src/gflow_cli/api/transports/ui_automation_video.py
  + DURATION_TAB_ABSENT_HINT constant beside ENTITY_ATTACH_DRIFT_HINT (:454)
  + _duration_row_present(page) -> bool | None   (None = menu not open)
  ~ _select_video_duration: probe before the cascade; pass remediation_hint=
  ~ duration_set event gains verified=
docs/MCP.md + website/docs/MCP.md
  ~ :86 no longer claims the rejection is always a 400 up front
```

---

## Task 1 — Test scaffold (red)

**What:** Red tests for every Critical/High scenario before any production code.

**Files:** `tests/api/test_duration_cohort_probe.py`

**Steps:**
- [ ] Fake page exposing `locator(...).count()` and `[role='menu']` presence
- [ ] S1: no duration row + open menu → raises `UiSelectorDriftError`
- [ ] S2: menu **absent** → does NOT report absence; retries/opens first
- [ ] S11: row present → probe returns True and the cascade still runs
- [ ] S14: the other nine cascade callers keep the default remediation

**Tests:** all red, no production code in this commit.

## Task 2 — `DURATION_TAB_ABSENT_HINT`

**What:** The cohort-aware message, as a module constant.

**Files:** `ui_automation_video.py` (beside `ENTITY_ATTACH_DRIFT_HINT`, :454)

**Steps:**
- [ ] Names cohort variance, not a frontend change
- [ ] States nothing was submitted and no credits were spent
- [ ] Gives the action: re-run without `--duration`, or `--model omni-flash`
- [ ] Says when filing a bug **is** warranted (UI shows the row for this account+model)
- [ ] Written for a human **and** an MCP agent — it reaches agents verbatim
- [ ] ASCII + em-dash only (Windows `cp1252` consoles)

**Tests:** assert the string contains no `--duration`-specific advice in the *class* default.

## Task 3 — `_duration_row_present()`

**What:** The fail-fast probe. ~40,000 ms → ~5 ms on affected cohorts.

**Files:** `ui_automation_video.py`

**Steps:**
- [ ] Non-waiting `count()` on `[role='menu'] [role='tab']:text-matches('^\d+s$')`
- [ ] Return `None` when `[role='menu']` itself is absent — **never** conflate with zero
- [ ] Precedent to follow: `_media_panel_present` (:1362)
- [ ] Locale-invariant: `^\d+s$` matched in the `ru` capture beside Cyrillic siblings

**Tests:** S1, S2, S8, S11 go green.

## Task 4 — Wire it into `_select_video_duration`

**What:** Probe first; on a definite zero, raise with the new hint instead of walking ten candidates.

**Files:** `ui_automation_video.py`

**Steps:**
- [ ] Probe before the cascade; `None` → existing open/retry path, unchanged
- [ ] Raise with `remediation_hint=DURATION_TAB_ABSENT_HINT`; exit code stays 23
- [ ] Assert ordering: `configure_video_settings` still precedes `_submit_and_poll`
- [ ] `duration_set` gains `verified=`; add `duration_readback_unverified` warning path

**Tests:** S1, S3, S12 green; S5 (chain/movie pre-flight) unchanged.

## Task 5 — MCP surface mirror

**What:** Not optional — Task 4 changes what an agent sees.

**Files:** `docs/MCP.md`, `website/docs/MCP.md`, `mcp/tools.py` docstrings

**Steps:**
- [ ] `docs/MCP.md:86` no longer claims rejection is always a 400 "rather than failing
      inside the worker" — with cohort variance, `omni-flash` *does* fail in the worker
- [ ] Tool description (`tools.py:940`) and the `duration` docstring (`:988`) stop
      claiming unconditional clip-length selection
- [ ] Confirm the queued payload round trip is untouched (`tools.py:922` ↔ `codec.py:244`)
- [ ] Walk the six mirror axes in `skills/check/SKILL.md` step 1b
- [ ] Regenerate `website/docs/`

**Tests:** `tests/mcp/test_cli_parity.py` + `test_server.py` green.

## Task 6 — Gates

**Steps:**
- [ ] `/gflow:check` fully green
- [ ] `errors.py` diff is empty — verify with `git diff --stat`
- [ ] CHANGELOG `[Unreleased]` entry
- [ ] `/gflow:branch-review` before the PR

## Task 7 — `aria-selected` read-back — **GATED, do not start**

**Blocked on:** the $0 stray-element probe on `denon82` (~12 lines added to `_menu_state`
in `scripts/dev/capture_video_model_capability_matrix.py`, replaying the cascade
document-wide). Needs a real Chrome session; a maintainer must run it.

**Kill condition:** `strays == []` for every `veo_3_1_*` ⇒ **close this task unstarted.**
The read-back would then add a false-negative class (cohorts rendering the row without
`aria-selected`) to defend a hazard that does not exist.

**If strays are found:** reuse the retry at `drivers/agentic.py:416-433`, failing **soft**
on a missing attribute — warn and proceed, never turn a working run into exit 23 (S4).
