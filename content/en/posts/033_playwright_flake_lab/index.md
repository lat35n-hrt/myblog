+++
date = '2026-01-08T20:36:45+09:00'
draft = false
title = 'Playwright Flake Lab'
categories = ["Playwright"]
+++



# Creating an Environment Where Clicks Occasionally Fail to Study Playwright Behavior

When E2E tests "occasionally fail," it undermines development speed and trust before any discussion about product quality can even begin.
Meanwhile, discussions about flakes tend to rely on subjective impressions and gut feelings.

So this time, I built a "Flake Lab" that intentionally injects flakes, quantifies success rates and failure reasons over 50 runs, and demonstrates that the success rate improves under identical conditions after fixes.

The goal of this PoC isn't "test failures are unavoidable." It's to make failure patterns reproducible, identify the root causes, and show how to eliminate them through design—all within a tight scope.

## Goals

- Reproduce UI flakes (blocked clicks) with configurable probability
- Run 50 times to aggregate success rates and failure reasons
- Keep traces and logs as evidence
- Demonstrate that "waiting for the right thing" improves success rates

## Approach: "Generate," "Measure," and "Eliminate" Flakes

The structure is straightforward:

1. Inject flakes (intentionally break things)
2. Run 50 times under identical conditions to aggregate success rates and failure reasons
3. Confirm failure modes through logs
4. Design proper wait conditions, apply improvements, and re-measure under the same conditions

"Reproducibility," "classification," "evidence," and "improvement" all contained in a single PoC.

## Entry Point for the PoC

This Flake Lab's main component is separated from pytest's auto-discovery and explicitly runs Playwright from a dedicated runner.

```bash
# Flake Lab
python scripts/run_flake_trials.py
```

## Flake Injection Method: Transient Overlay (Click Interception)

This time, I'm tracking behavior with a simple UI flake pattern.

- Insert a **transient overlay (full-screen overlay)** briefly
- The overlay captures pointer events, blocking clicks
- The overlay exists for only 300ms
- Probability of showing the overlay per trial is 0.3
- Fixed random seed ensures reproducibility

Technically, the Playwright side inserts `#__flake_overlay` into the DOM and removes it after a set duration.

Intentionally creates event (click) interception.

```python
# inject overlay
def inject_overlay(page, duration_ms: int) -> None:
    page.evaluate(
        """
        (durationMs) => {
          const id = "__flake_overlay";
          const old = document.getElementById(id);
          if (old) old.remove();

          const el = document.createElement("div");
          el.id = id;
          el.style.position = "fixed";
          el.style.inset = "0";
          el.style.background = "rgba(0,0,0,0.01)";
          el.style.zIndex = "2147483647";
          el.style.pointerEvents = "auto";
          document.body.appendChild(el);

          setTimeout(() => { el.remove(); }, durationMs);
        }
        """,
        duration_ms,
    )
```

## Test Flow: Single Representative Path

The target flow is fixed to one path:

```
List → Detail → Approve
```

This "representative flow" is repeated 50 times, with each trial's results recorded in JSONL.
For failures, traces can also be saved (if configured).

## Measurement Prerequisites: Exclude Server Startup Noise (Health Check)

To exclude measurement noise (server not started or still starting), `GET /health` is checked before trials.
The implementation uses only standard library components, with a 2-second timeout and immediate failure for any non-200 HTTP status.

```python
def assert_server_up():
    with urlopen(f"{BASE_URL}/health", timeout=2) as r:
        if r.status != 200:
            raise RuntimeError(f"Server not ready: {r.status}")
```

This allows us to focus the flake discussion on UI/synchronization issues.

## Execution and Measurement: Control via Environment Variables

Execution conditions are fixed through environment variables:

- `BASE_URL`: Target URL for testing
- `FLAKE_STRATEGY`: Click strategy (naive / wait / off)
- `FLAKE_OVERLAY_MS`: Overlay display duration (e.g., 300ms)
- `FLAKE_OVERLAY_PROB`: Probability of injecting overlay (e.g., 0.3)
- `TEST_FLAKE_SEED`: Random seed (e.g., 42)

The key point is comparing before/after using the same seed / prob / duration.
Without this, "it just happened to work" or "we got lucky this time" creeps in.

## Implementation Details: Probabilistic Overlay Injection, Difference Is "What to Wait For"

### Overlay Injection Is "Probabilistic," Not "Every Time"

The overlay is injected only when the following conditions are met for each trial:

- `FLAKE_STRATEGY != "off"`
- AND `random() < FLAKE_OVERLAY_PROB` (e.g., 0.3)

Whether injection occurred is logged as `overlay_injected` in the JSONL.

```python
# overlay injection logic (excerpt)
overlay_injected = False

if FLAKE_STRATEGY != "off" and (_rng.random() < FLAKE_OVERLAY_PROB):
    overlay_injected = True
    inject_overlay(page, FLAKE_OVERLAY_MS)
```

### The Before/After Difference Is "Synchronization Design Before Clicking"

The improvement isn't "blind retries" but clarifying what to wait for.

- **naive**: If the injected overlay is still active during the click attempt, it may fail
- **wait**: If injected, wait for the overlay to disappear before clicking

```python
# before/after branching (excerpt)
if overlay_injected and FLAKE_STRATEGY == "wait":
    # improved: wait overlay to disappear
    page.locator("#__flake_overlay").wait_for(
        state="detached",
        timeout=FLAKE_OVERLAY_MS + 2000
    )

if FLAKE_STRATEGY == "naive" and overlay_injected:
    # intentionally fragile: short timeout so click fails while overlay exists
    btn.click(timeout=200)
else:
    btn.click()
```

## Results: Before / After Comparison Under Identical Conditions (50 Runs)

### Before (naive): 60% Success Rate

Execution command:

```bash
BASE_URL=http://127.0.0.1:8004 \
FLAKE_STRATEGY=naive \
FLAKE_OVERLAY_MS=300 \
FLAKE_OVERLAY_PROB=0.3 \
TEST_FLAKE_SEED=42 \
python scripts/run_flake_trials.py
```

Output (50 runs):

```
Runs: 50
OK: 30, FAIL: 20, SuccessRate: 60.0%
Failure reasons:
  - click_timeout: 20
```

With naive, when the overlay is injected, clicks are blocked and `click_timeout` occurs.
A typical case of "the button is visible, but a transparent element is covering it, preventing clicks."

### After (wait): 100% Success Rate

Execution command:

```bash
BASE_URL=http://127.0.0.1:8004 \
FLAKE_STRATEGY=wait \
FLAKE_OVERLAY_MS=300 \
FLAKE_OVERLAY_PROB=0.3 \
TEST_FLAKE_SEED=42 \
python scripts/run_flake_trials.py
```

Output (50 runs):

```
Runs: 50
OK: 50, FAIL: 0, SuccessRate: 100.0%
```

Improvement details:

1. For trials where overlay is injected, wait for `#__flake_overlay` to be removed (detached) from the DOM before clicking
2. Then click

"What to wait for" is fixed to the overlay's removal. The results confirm that flakes can be eliminated through synchronization design rather than relying on retries.

## Log Collection: JSONL and Playwright Call Logs

Comparing logs output to JSON files.

### Before (naive): click_timeout with overlay (click interception is the cause)
```json
{"run_id": 45, "ok": false, "error_type": "click_timeout", "error_message": "Locator.click: Timeout 200ms exceeded.\nCall log:\n  - waiting for locator(\"button#approve\")\n    - locator resolved to <button id=\"approve\">Approve</button>\n  - attempting click action\n    2 × waiting for element to be visible, enabled and stable\n      - element is visible, enabled and stable\n      - scrolling into view if needed\n      - done scrolling\n      - <div id=\"__flake_overlay\"></div> intercepts pointer events\n    - retrying click action\n    - waiting 20ms\n    - waiting for element to be v", "elapsed_ms": 1627.665, "base_url": "http://127.0.0.1:8004", "ts_epoch_ms": 1767687410485, "step": "click_approve", "flake_strategy": "naive", "overlay_injected": true, "overlay_ms": 300, "overlay_prob": 0.3, "test_flake_seed": 42}
```

What this call log shows is that while the button itself is judged to be "visible, enabled and stable," `#__flake_overlay` intercepts pointer events and blocks the click.
In other words, a typical UI flake: "the element is visible but cannot be clicked."

### Before (naive): OK without overlay
```json
{"run_id": 49, "ok": true, "error_type": "", "error_message": "", "elapsed_ms": 4023.302, "base_url": "http://127.0.0.1:8004", "ts_epoch_ms": 1767687421734, "step": "wait_approved", "flake_strategy": "naive", "overlay_injected": false, "overlay_ms": 300, "overlay_prob": 0.3, "test_flake_seed": 42}
```

### After (wait): OK even with overlay (waiting fixed to overlay removal)
```json
{"run_id": 2, "ok": true, "error_type": "", "error_message": "", "elapsed_ms": 3706.224, "base_url": "http://127.0.0.1:8004", "ts_epoch_ms": 1767687944178, "step": "wait_approved", "flake_strategy": "wait", "overlay_injected": true, "overlay_ms": 300, "overlay_prob": 0.3, "test_flake_seed": 42}
```

With the wait strategy, success is confirmed even when overlay injection occurs, using the same schema. The synchronization design change—waiting for `#__flake_overlay` to disappear (detached) before clicking—is reflected.

## Why This PoC Is Useful

The essence of flakes is usually one of the following:

- The state isn't ready yet (waiting for the wrong thing)
- UI changes momentarily (animations, overlays, spinners)
- Async processing completion is non-deterministic (API delays, rendering delays)

In this PoC, rather than leaving flakes to "chance":

1. Fix reproduction conditions (seed / prob / duration)
2. Classify failures (reason)
3. Keep evidence (logs / trace)
4. Apply countermeasures and demonstrate with success rates (before/after)

These steps were confirmed within a minimal scope.

## Rendering Method (Jinja2) and Current Conclusions

Note that what this PoC addresses is not the "rendering method" but a DOM/event-level flake: event interception (click blocking) at the DOM level.

The UI in this case has a simple structure, and enabling Jinja2 (server-side template) didn't substantially change the DOM structure or flow, so similar failure modes and improvement effects were reproducible.

However, if UI implementation changes alter DOM structure, selectors, or timing—as in realistic tests—the wait condition assumptions would also change, requiring readjustment.

## Appendix: Directory Structure (Minimal)

Since the measurement center of this article is the runner, it runs with the following minimal structure:

```
scripts/run_flake_trials.py: 50-run runner (overlay injection, aggregation, JSONL recording)
artifacts/runs/YYYYMMDD_HHMMSS_*: execution results (JSONL, traces if needed)
```

(The UI implementation exists on the application side pointed to by `BASE_URL`. The PoC's focus is "reproducing click interception → identifying cause → improving through synchronization design.")

## Summary

- Flakes aren't "bad luck"—they're targets for reproduction, classification, evidence, and improvement
- Click interception via overlay was injected probabilistically, and success rates were measured over 50 runs
  - naive: 60% (20 click_timeouts)
  - Waiting for overlay: 100% (reproduced under identical conditions)
- Through logs (JSONL) and Playwright call logs, the improvement discussion was grounded in observable data
```