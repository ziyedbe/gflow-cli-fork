# Scenario: fail-fast duration-control probe + cohort-aware exit-23 remediation

> Input: `/gflow:predict` verdict **🟡 CAUTION 6/10** (2026-09-04). The proposal
> was reshaped by that round and this analysis covers the reshaped version, not
> the original. Two things the predict council **removed** are deliberately absent
> below: scoping all ten cascade candidates to `[role='menu']`, and rewriting
> `UiSelectorDriftError._default_remediation` in `errors.py`.

## What is actually being built

1. **A fail-fast existence probe.** Before walking the duration cascade, one
   non-waiting `count()` on `[role='menu'] [role='tab']:text-matches('^\d+s$')`.
   Zero ⇒ this cohort renders no duration row; raise immediately instead of
   spending 10 candidates × `timeout_ms=4000`. Precedent: `_media_panel_present`
   (`ui_automation_video.py:1362`).
2. **A cohort-aware message**, passed per-raise as `remediation_hint=` with a
   `DURATION_TAB_ABSENT_HINT` constant beside `ENTITY_ATTACH_DRIFT_HINT`
   (`ui_automation_video.py:454`). **No `errors.py` diff** — that default is
   shared by all ten `_probe_selector_cascade` call sites.
3. **A read-back after the click**, reusing the `aria-selected` retry at
   `drivers/agentic.py:416-433`, failing *soft* when the attribute is absent.

## Gate before any of it

Predict's Devil's Advocate rated the underlying hazard **LIKELY, not CONFIRMED**.
The $0 stray-element probe on `denon82` (add ~12 lines to `_menu_state` in
`scripts/dev/capture_video_model_capability_matrix.py`, replay the cascade
document-wide, dump matches) decides whether item 3 ships at all. **Kill
condition:** `strays == []` for every `veo_3_1_*` ⇒ drop the read-back, ship 1+2.

## Coverage map

| Dimension | Active? | Why |
|---|---|---|
| D1 Auth & session | ❌ | No auth surface. Predict persona 2 verified: zero `cookie/token/bearer/sapisid/recaptcha` hits in the edit region. |
| D2 WAF / reCAPTCHA | ⚠️ low | No new requests. The probe *reduces* DOM churn; only a fail-fast that changes retry cadence could move per-profile heat. |
| D3 Selector drift & locale | ✅ **central** | The probe is a new selector, and `^\d+s$` is a text match on a page that localises everything around it. |
| D4 Batch / resume | ✅ | Chain and movie run N links; a mid-loop abort after link 0 has billed is the expensive case. |
| D5 Concurrency & Page pool | ⚠️ low | UI video uses `self._page` under `_generate_lock`, never `_checkout_page` — no `QueueFull`. But the 40 s is held under that lock, so N looped runs pay N × 40 s serially. |
| D6 Data layer | ❌ | No `DataStore` callsite, no schema, no recorder. |
| D7 Error propagation & exit codes | ✅ **central** | Exit 23, `remediation_hint`, Problem Details shape. |
| D8 Cross-platform paths | ⚠️ low | Message text only (em-dash / `PYTHONUTF8` on Windows consoles). |
| D9 Transport edge cases | ✅ | Popover open/closed races, the toggle trap. |
| D10 Headless vs headed | ✅ | The probe runs in a real browser; a headless or migrated host must not reach it. |
| D11 Input validation | ⚠️ low | `(4,6,8,10)` membership already guarded upstream at three layers. |
| D12 Observability | ✅ **central** | `duration_set` currently fires on a bare click with no read-back — the event lies today. |
| D13 MCP parity | ✅ **central** | `remediation_hint` reaches agents **verbatim** through the queued-job failure envelope. |

## Scenario table

| # | Dim | Scenario | Severity | Expected behaviour | Test category |
|---|---|---|---|---|---|
| S1 | D3/D12 | Cohort has **no** duration row; user passes `--duration 6` on `t2v` with **no** `--model` (guards bail on `resolved is None`, so it reaches the browser) | **Critical** | Probe returns 0 → `UiSelectorDriftError` (exit 23) in **~5 ms**, pre-submit, $0, message names cohort variance not frontend drift | unit (mocked page) + live |
| S2 | D9 | Popover never opened (mode/overlay race); probe queries a closed menu | **Critical** | Must **not** read 0-as-absence. Distinguish "menu missing" from "menu present, no duration row" — `[role='menu']` count 0 ⇒ retry/open, not a cohort verdict | unit |
| S3 | D12 | Click lands but nothing changes; `aria-selected` stays `false` | **High** | `duration_set` must carry `verified=False`, or raise — today it logs success unconditionally | unit |
| S4 | D3 | Cohort renders the row but with **no** `aria-selected` attribute (segmented control) | **High** | Fail **soft**: `log.warning('duration_readback_unverified')`, proceed. Never turn a working run into exit 23 | unit |
| S5 | D4 | `movie`/`chain` manifest, `duration` on link 3; links 0–2 already rendered and billed | **High** | Pre-flight rejects before link 0, or the abort is explicitly documented as post-spend. `reject_unusable_links` already validates — confirm the probe does not move the failure later | unit + BDD |
| S6 | D13 | Same failure via MCP `gflow_generate_video`, `wait=True` | **High** | Queued-job failure envelope carries `exit_code: 23`, `retryable`, and the **new** `remediation_hint` verbatim (`worker/daemon.py:441-464`) | unit |
| S7 | D13 | Same via MCP with `wait=False` | **Medium** | Agent gets a handle; **there is no MCP tool to read task status**, so the failure is invisible on that path. Document it; do not silently regress | doc + issue |
| S8 | D3 | `ru` / `pt-BR` locale, cohort **has** the row | **High** | `^\d+s$` still matches — the ru capture kept `4s/6s/8s` Latin beside `Изображение`/`Видео`. Probe must not depend on any translated word | live (both locales) |
| S9 | D9 | The composer chip (`button[aria-haspopup='menu']`, label `Vídeo · 720p · 8s…`) is a **toggle** | **Medium** | Probing must not click it; clicking closes the popover. Predict persona 2 named this as the one real collider — it only carries `8s` on cohorts that *have* the control | unit |
| S10 | D10 | Account migrated to `flow.google.com` | **Medium** | `FlowHostMigratedError` (exit 36) at `ui_automation_video.py:1463` fires **first**; the probe is never reached | unit (already covered — assert no regression) |
| S11 | D7 | `--model omni-flash` on a cohort that *does* render the row, `--duration 10` | **Medium** | Unaffected happy path. The probe must not add latency here (~5 ms budget) | unit + live |
| S12 | D5 | 10 looped runs on an affected cohort | **Medium** | 10 × 40 s ≈ 6.7 min of dead wait today, all under `_generate_lock`. After the probe: ~10 × 5 ms | perf assertion |
| S13 | D8 | Windows console, `PYTHONUTF8` unset | **Low** | New message must not carry characters that raise `UnicodeEncodeError` under `cp1252` | unit |
| S14 | D7 | `remediation_hint` for the **other nine** `_probe_selector_cascade` callers | **Low** | Unchanged. This is the regression the predict council blocked: rewriting the class default would make "re-run without `--duration`" nonsense for `image_mode_tab` / `count_tab` | unit |

## Must-cover acceptance criteria

1. Probe returns **0 only when the menu is open** — S2 is the correctness crux;
   a closed-popover false zero would report a cohort difference that is really a race.
2. Failure stays **pre-submit and $0** — assert ordering against
   `configure_video_settings` → `_attach_media_inputs` → `_submit_and_poll`.
3. Exit code stays **23**; no new class, no `EXIT_CODE_MAP` entry (predict D4:
   nothing branches on it, so a new code is an unearned contract change).
4. `errors.py` has **zero diff**.
5. `duration_set` gains `verified`; a new `duration_readback_unverified` warning exists.
6. The message is written for **both** a human and an MCP agent, since it reaches
   agents verbatim.
7. Latency on the affected path drops from ~40 s (80 s with the changelog-overlay
   retry) to single-digit ms — assert it, do not claim it.

## BDD skeleton (`tests/features/`)

```gherkin
Scenario: a cohort without a duration control fails fast and spends nothing
  Given a Flow settings popover that renders no duration tabs
  When the user runs "gflow video t2v --duration 6" with no --model
  Then the run aborts before any generation is submitted
  And the exit code is 23
  And the error names account/cohort variance rather than a frontend change
  And no credits are spent

Scenario: a closed popover is not mistaken for an absent duration control
  Given the settings popover has not opened yet
  When the duration probe runs
  Then it does not report the control as absent
  And it retries opening the popover before deciding

Scenario: a duration click that does not take is not logged as success
  Given a duration tab that stays aria-selected="false" after the click
  When the transport selects a duration
  Then "duration_set" is not emitted with verified=true

Scenario: a cohort whose tabs carry no aria-selected still succeeds
  Given a duration row rendered without aria-selected attributes
  When the transport selects a duration
  Then the run proceeds
  And a duration_readback_unverified warning is emitted
```

## KNOWN_ISSUES cross-references

- **"Video duration control is absent on some account cohorts"** — rewritten in
  #655 (this session). S1's message must match what that row now promises: exit
  23, pre-submit, no credits, and *cohort variance, not a bug to file*.
- **#639 / migrated host** — S10. `FlowHostMigratedError` must keep winning the race.
- **#634** — chain/movie mid-spend, S5.

## Open question for PLAN

`_set_output_count` (`ui_automation_video.py:1636-1644`) builds the *same* inline
unscoped text cascade, clicks bare, and its own comment claims "the same
pre-spend contract as the duration probe (#288)". A wrong match there **doubles**
spend rather than mis-sizing one clip. Predict's Architect flagged it; it is
strictly worse than the bug being fixed. In scope, or filed as its own issue?
