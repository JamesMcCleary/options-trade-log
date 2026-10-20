# Changelog

All notable changes to Options Trade Log.

---

## [1.4] — Current

### Changed
- Options Trade Log column order: **Type** (CALL/PUT) now leads at col 2, **Signal** (W3 label) moves to col 3. Type is the primary action; Signal is supporting context.

### Added
- `C_COMPLETE` (`#1B5E20`) color token added to the palette. Planned for use as a row-level background state when a trade hits its full target, replacing the translucent bull/bear tint. Implementation deferred to v1.5.

### Fixed
- Annotated `_w{tf}r` W1 bar count variables as tuple-position placeholders on non-ETA timeframes. Pine Script v6 requires full tuple destructure; only `_w15r` (intraday ETA) and `_w60r` (swing ETA) are consumed.

---

## [1.3]

### Added
- **TF column** (col 12): records the structure timeframe the signal was generated on (15m intraday / 60m swing), stored at log time so historical rows remain accurate if the mode is changed mid-session.
- **ETA column** (col 13): estimates time to trade completion as a formatted range (e.g., `"2-3 hrs"`, `"1-2 days"`). Derived from the structure TF's W1 bar count projected at 1.0× to 1.618× (the expected W3 duration range).
- `etaStr()` helper function: formats a lo/hi minute range into `"X-Y min"`, `"X-Y hrs"`, or `"X-Y days"` using `math.ceil` for conservative (trader-safe) rounding.
- `tl_tf` and `tl_dur` string arrays to persist TF and ETA values per log row.

### Changed
- Table widened from 12 → 14 columns. Both section header merges updated to span full width.

---

## [1.2]

### Added
- **T1 column** (col 10) and **T2 column** (col 11) to the Options Trade Log, exposing the intermediate and secondary Fibonacci price targets directly in the table.
- `tl_t1_f` float array to store T1 values per row (T2 and Target were already stored; T1 was computed but not persisted).
- Per-target status badges: T1 cell shows `✅` on hit; T2 cell shows `✅` on T2 hit, `🎯` on full target hit.

### Changed
- Strike column no longer carries status badges (previously stacked with T2 badge). Status is now distributed across T1 and T2 cells where it is directly associated with the relevant price level.
- `log_entry()` signature updated: `t1_f` parameter added between `expiry` and `t2_f`.
- Table widened from 10 → 12 columns.

---

## [1.1]

### Fixed
- **Pivot-based Fibonacci targets** replace ATR approximations. `mtfW3Signal()` now returns a 7-tuple including W2 pivot price and W1 wave size per direction. Targets and stops are computed as Fibonacci projections from the W2 anchor (1.000×, 1.272×, 1.618× for T1/T2/Target; −0.1× for stop). ATR values remain as a fallback only until the first confirmed signal populates the pivot vars.
- **Score denominator corrected from 6 to 8** (intraday). `bull10` / `bear10` boolean aliases were missing — the 10m and 1W timeframes were fetched but excluded from `bullCount` / `bearCount`. Both are now included, bringing `maxCount` to 8 and making the A+ threshold (≥7/8) reachable.
- **Status-flip dedup bug resolved.** The `_bull_logged` / `_bear_logged` dedup guard was evaluating `.set()` on every price tick rather than once per bar close, causing duplicate log entries on signal bars. All detection and log writes are now gated through `barstate.isconfirmed`.

### Added
- Directional grade computation: `_bull_grade` and `_bear_grade` replace the undirected `_liveGrade`. Each grade is computed via `gradeFromScore()` and then passed through `applyHTF()`, which caps A/A+ to B+ when neither 1D nor 1W is aligned with the signal direction.
- Entry Log (5m/30m) and Options Log (15m/60m) now source targets from their own respective TF pivot data rather than sharing a single set of levels.
