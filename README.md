# Options Trade Log

A Pine Script v6 indicator for TradingView that provides a persistent, live-updating on-chart log for options entries derived from Elliott Wave W3 signals. Built as an indicator for TradingView.

---

## The Problem

Elliott Wave trading requires monitoring signals across multiple timeframes simultaneously — an entry signal on the 5-minute chart may only be worth acting on when the 15-minute and 60-minute structure is also aligned. TradingView doesn't provide a native way to surface cross-timeframe signal history in a single view. Traders either context-switch between chart tabs (losing the moment) or maintain a separate spreadsheet log (disconnected from live price action).

This indicator solves that by embedding two persistent log tables directly on the chart:

- **Entry Log** — captures W3 signals at the entry timeframe (5m intraday / 30m swing) with live status badges as price reaches T1, target, or stop levels
- **Options Trade Log** — captures structure-timeframe confirmations (15m / 60m) and surfaces the options-specific context a trader needs: contract type, strike, expiry, T1/T2 price targets, estimated time to completion, and Friday expiry warnings

Both tables update live on every bar close without any manual input.

---

## Design Decisions

### Column Hierarchy: Type Before Signal

The Options Trade Log leads with **Type** (CALL / PUT) at column 2, followed by **Signal** (the W3 label) at column 3. This ordering reflects the primacy of the action over its source: a trader scanning the log mid-session needs to know *what to do* (buy a call, buy a put) before they need to know *why* (W3 bullish signal on the 15m). The Signal column provides audit context, not decision input.

This mirrors standard card-UI patterns where the primary CTA is presented first, with supporting metadata below or behind.

### Color System

The palette inherits Bob EW's exact color tokens to maintain visual continuity across a multi-indicator chart. Both indicators share the same TradingView dark theme and the same row coloring convention, so a trader's eyes never have to recalibrate when scanning between the two.

| Token | Hex | Role |
|---|---|---|
| `C_BULL` | `#089981` | Bull signal rows, active bullish state |
| `C_BEAR` | `#F23645` | Bear signal rows, active bearish state |
| `C_WARN` | `#E29400` | Friday expiry ⚠️ flag |
| `C_INACTIVE` | `#5D606B` | TF and ETA columns — context metadata, not actionable |
| `C_COMPLETE` | `#1B5E20` | *(Planned v1.5)* Row state when full target is hit |
| `C_GOLD` | `#D4AF37` | Section header labels |

Row backgrounds use `color.new(C_BULL/C_BEAR, 85)` — 85% transparency — so the signal color reads as a tint rather than a fill, keeping price data on the chart visible beneath the table.

### Emoji Communication Layer

Emoji are used as status indicators rather than text labels because they render at consistent size across all TradingView themes and screen DPIs, unlike text which can overflow narrow columns.

| Emoji | Meaning | Trigger |
|---|---|---|
| ✅ | T1 hit — first target reached, consider partial exit | `close >= t1_f` (bull) |
| 🎯 | Full target hit — trade complete | `close >= target_f` (bull) |
| 🛑 | Stop hit — exit signal | `close <= stop_f` (bull) |
| ⚠️ | Friday warning — avoid current-week expiry | `dayofweek == friday AND hour >= 15 ET` |

These four were chosen to avoid collision with emoji used in the parent Bob EW indicator's Action Bar, which uses directional arrows and color-coded wave labels. The status set here is intentionally orthogonal: progress states, not signal states.

---

## Engineering Notes

### 1. Pivot-Based Fibonacci Targets (replaces ATR approximations)

**The problem.** The original implementation derived target and stop prices from ATR (Average True Range) multiples — a common approximation when precise pivot data isn't available. This produced target levels that diverged from the parent indicator's actual ZigZag pivot levels, undermining the log's value as a decision tool. If the numbers on the log don't match the lines on the chart, traders stop trusting the log.

**The fix.** `mtfW3Signal()` was extended to return a 7-tuple instead of 3 values, adding the W2 pivot price and W1 wave size per direction. All target and stop levels are now computed as Fibonacci projections from those anchors:

```
Stop    = W2_low − 0.100 × W1   (tight structural buffer just below W2)
T1      = W2_low + 1.000 × W1   (equal-move projection)
T2      = W2_low + 1.272 × W1   (shallow extension)
Target  = W2_low + 1.618 × W1   (full W3 extension)
```

The four new values are held as `var float` locals inside the function so they persist at their most recent confirmed signal value — callers always receive a valid pivot set, never `na`, after the first signal fires. ATR remains as a fallback on the initial bar only, before the first signal populates the pivot vars.

This keeps the Trade Log's price levels mathematically consistent with the parent indicator without requiring access to its private internal arrays — a clean boundary between two independently distributed scripts.

### 2. Score Denominator Fix — 6 → 8

**The problem.** The confluence grade thresholds were written for a denominator of 8 (intraday A+ requires 7/8 aligned timeframes). But the count formula only included 6 timeframes — the 10m and 1W TFs were fetched via `request.security()` but their boolean aliases were never defined and never included in `bullCount` / `bearCount`. This meant the maximum achievable score was 6, making A+ structurally unreachable — a silent regression where the feature appeared to work but never fired at full quality.

**The fix.** `bull10` / `bear10` were defined alongside the other TF aliases, and both 10m and 1W were added back into the intraday count formula. `maxCount` is now correctly 8, and the existing grade thresholds (≥7 for A+, ≥6 for A, etc.) function as designed.

### 3. Status-Flip Dedup Bug — `.set()` on Every Tick

**The problem.** The dedup guard that prevents duplicate log entries on the same bar used a `var int[]` array to track the last-logged bar index. The guard's `.set()` call was running inside a block that evaluated on every price tick rather than once per confirmed bar close. On a volatile bar, this caused the log entry check to fire dozens of times per bar — and on the bar where a new signal appeared, the entry was written, then the guard was overwritten with the current bar index, then the check fired again and the entry was written a second time.

This is the same class of bug as a UI component that fires `onChange` on every keystroke rather than `onBlur` — the state mutation is happening at the wrong lifecycle event.

**The fix.** All signal detection and log writes are gated through `barstate.isconfirmed`, which evaluates to `true` exactly once per bar (on the final tick of the confirmed close). The dedup guard's `.set()` only runs inside that gate, so each bar index is written once and the entry is created once.

---

## Screenshots

### Entry Log — Live Status Badges
![Entry Log](docs/screenshots/entry-log.png)
*Entry Log showing 5m signals with live ✅ T1, 🎯 target, and 🛑 stop badges updating as price moves.*

### Options Trade Log — Full Column View
![Options Trade Log](docs/screenshots/options-trade-log.png)
*Options Trade Log with Type, Strike, Expiry, T1, T2, TF, and ETA columns. Grade A+ rows reflect 7/8 or 8/8 confluence.*

### Multi-Timeframe Confluence Score
![MTF Confluence](docs/screenshots/mtf-confluence.png)
*Score column showing 7/8 (A+) vs. 4/8 (B+) confluence across intraday and swing timeframes.*

---

## Disclaimer

This is a personal technical tool built for learning and exploration. Nothing in this repository constitutes financial advice. Use at your own discretion.

---

## Publishing This Repo

### Option A — Private GitHub Repository

**Create the repo**
1. Go to [github.com/new](https://github.com/new)
2. Name it `options-trade-log` (or similar)
3. Set visibility to **Private**
4. Skip the README / .gitignore initializers (this repo has both)
5. Click **Create repository**

**Push the code**
```bash
cd options-trade-log
git init
git add .
git commit -m "Initial release — v1.4"
git branch -M main
git remote add origin https://github.com/jamesmccleary/options-trade-log.git
git push -u origin main
```

**Invite the reviewer as a collaborator**
1. Go to **Settings → Collaborators** in your repo
2. Click **Add people**
3. Enter their GitHub username or email
4. They'll receive an email invitation — they need to accept before they can view the repo

---

### Option B — Secret GitHub Gist

A secret Gist is quicker to share and doesn't require the reviewer to accept an invitation. It's not indexed by search engines but anyone with the URL can view it.

1. Go to [gist.github.com](https://gist.github.com)
2. Set the filename to `Options_Trade_Log_v1_4.pine`
3. Paste the contents of `src/Options_Trade_Log_v1_4.pine`
4. Click **Create secret gist**
5. Copy the URL from your browser and share it directly

> **Note:** Secret Gists are not private — anyone with the URL can view them. If you want true access control, use Option A.
