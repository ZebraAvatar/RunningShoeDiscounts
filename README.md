# RunningShoeDiscounts

State and tooling for the daily running-shoe deal board (US men's 12.5).
Repo: https://github.com/ZebraAvatar/RunningShoeDiscounts
Board: https://claude.ai/code/artifact/2d9671d4-0b8c-48a1-bb34-9ce0c42c8933

## Files

- `TASK_PROMPT.md` — the scheduled-task prompt. Source of truth for how a run behaves.
- `registry.json` — per-site knowledge: which URLs work, which are blocked, how each site exposes discounts, per-site gotchas. Grows every run.
- `reviews.json` — cache of running-press review dates. Immutable facts; never look one up twice.
- `state/` — daily snapshots and the qualifying-rows state file (currently `running-shoe-deals-log.md` in Google Drive; migrate here).
- `snapshots/` — raw HTML per URL per day. Makes staleness *visible*: if promo markup is byte-identical across a promotion boundary, it's static template text, not a live signal.

**Never commit a token to this repo.**

## Why this exists

On 2026-09-09 the board priced 11 rows with a "20% off in cart" discount that did not apply. Three rows (Takumi Sen 11, Hyperion Max 3, Paradigm 8) didn't clear 30% without it and should never have been listed. The runs of 09-10, 09-11 and 09-12 repeated it.

Root cause: the extraction subagent's prompt named the expected answer — `whether the page shows an "Extra 20% Off in Cart" badge` — so a small model confirmed a string it had been handed. Last run's finding was laundered into this run's evidence via the prompt itself. The same tool output also contained back-computed fake MSRPs ("~$138.28 (implied -35%)"), which should have discredited the whole extraction.

## The rules that follow from it

1. **Never name the expected answer in an extraction prompt.** Ask "quote any promotional text verbatim," never "does it show the 20% badge?"
2. **Never carry a value forward.** Prior state is for diffing only.
3. **Qualify on the visible price.** Codes and cart discounts annotate a row; they never put one on the board.
4. **Fail loud.** An unreadable field is "could not read," never a blank and never a default.
5. **Report what's listed.** Don't adjudicate whether a sale is really on — that isn't observable and isn't the job.

## Split of labour

Deterministic (code): fetching, snapshotting, diffing, JSON-LD extraction, percentages, the $130/30% filter, first-seen dates, price history, review-cache lookups.

Model: judging the diff (running shoe or lifestyle? which generation?), discovery of new sites and URL patterns, the one-sentence takes, writing the page.

Discovery runs weekly, not daily, and its output is **registry entries, not deals** — a working URL added to `registry.json` pays off on every subsequent daily run.

## Playwright

Chromium is preinstalled at `/opt/pw-browsers/chromium` (`PLAYWRIGHT_BROWSERS_PATH` is set; never run `playwright install`). Needed wherever size availability is JS-rendered — Zappos above all, where 11 of 16 product pages returned unreadable selectors to a plain fetch.

Silent-failure discipline, in order of how often it bites: assert instead of defaulting on a missing selector; wait for the value, not the element, so you don't capture a `$0.00` placeholder; re-query after re-renders rather than holding stale handles; screenshot every extraction; bound-check values (a running shoe is not $4 and not $9,000); never wrap an extraction in a bare try/except.

## Not yet built

The scraper. It needs the registry populated by a real canvassing pass first — writing it against guessed URL patterns would just encode today's 404s.

## Robots

REI's main category, ASICS, and Woot are robots-disallowed. That stays true regardless of tooling; log them as coverage gaps rather than routing around them.
