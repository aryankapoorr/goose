# Remaining Data TODO

Runtime rule: do not show fabricated metric values. A surface may show live, local, bridge-derived, or unavailable data only.

## Sleep

- [ ] Import primary sleep directly from band packets and persist nightly sleep records locally.
- [ ] Populate sleep stage timeline from band-derived sleep stage output. Likely already correct once the score auto-runs (see below); re-check before assuming more work is needed.
- [ ] Build sleep trend history for score, time asleep, REM, deep, HR dip, sleep time, wake time, and latency. No `sleepTrendRowsForV2()` exists yet, and (like recovery) no bridge method returns a daily score series, so this needs a new Rust rollup.
- [x] Add real target sleep amount input and persist it. `OnboardingStorage.targetSleepMinutes` now backs both `SleepV2SleepNeededSheet` and `SleepV2AlarmSheet` via `@AppStorage`, so they stay in sync and persist across launches.
- [ ] Compute Sleep Needed from target sleep, sleep debt, recent sleep history, and planned wake time. Sleep debt fields already exist in the Rust sleep score output; still needs real wiring plus a planned-wake-time model.
- [ ] Compute Sleep Bank from target sleep amount and stored nightly sleep history. Blocked on persisted nightly sleep history above.
- [x] Derive sleep insights from actual sleep score components and confidence fields. `sleepInsightTopDriver()`/`sleepInsightConfidenceSummary()` now surface the real weighted score component with the largest deficit and the real `confidence_0_to_1` value instead of hardcoded text.

Also fixed: Sleep V2 now auto-runs the packet score bridge call on load (same fix as Recovery), and removed a hardcoded `92` fallback score that violated the no-fabricated-data rule above.

## Recovery

- [ ] Compute recovery score from local HRV, resting HR, respiratory rate, sleep, strain, and temperature inputs.
- [ ] Populate recovery history rows for score, HRV, resting HR, respiratory rate, SpO2, and wrist temperature.
- [ ] Resolve respiratory rate, SpO2, and wrist temperature packet semantics from band data.
- [ ] Replace manual vitals placeholders with packet-derived or user-entered values with provenance.

## Strain And Activity

- [x] Persist activity sessions with start/end, type, HR summary, zone durations, and sync status. Already fully built in Rust (`activity_sessions`/`activity_metrics` tables, `activity.*` bridge methods) and exercised by the live-workout recording flow. Calories are the one field still not attached to a session.
- [ ] Compute daily strain from activity sessions and HR load. The strain score today is a continuous motion+HR estimate; it does not use the persisted session/zone data at all. Needs real Rust algorithm work to combine the two.
- [x] Add real step count extraction from motion/history packets. Already real (`step_counter.rs`/`step_discovery.rs`/`step_motion_estimator.rs`); it just wasn't populating because Strain never called `refreshPacketInputsIfNeeded()`. Fixed below.
- [x] Add calorie/energy estimator from profile, HR, movement, and activity sessions. Already real (`energy_rollup.rs`); same auto-run gap as steps, now fixed.
- [ ] Build strain trends for score, exercise duration, daytime HR, total energy, and step count. Energy/step trend rows already work once inputs auto-run. `strain-score-trend` needs a new Rust "daily" field (same class as sleep/recovery). `daytime-hr-trend` needs a new daytime-HR rollup. `exercise-duration-trend` is Swift-only (aggregate `activity.list_sessions` per day) and still open.

Also fixed:
- Strain V2 had no `.onAppear` at all (unlike Recovery/Sleep) and never auto-ran the packet score bridge call, so the strain score stayed empty until a manual Packet Inputs run. It now calls `store.refreshPacketInputsIfNeeded()` and `store.refreshPacketScoresIfNeeded()` on load, plus refreshes the activity timeline.
- `strainDurationDisplayText()` was an unconditional `"--"` stub; it now sums real persisted session durations for the selected day.
- The Activities section was a hardcoded "No activities" card; it now lists real sessions from `activity.list_sessions_with_metrics` when present.
- The heart-rate-zones chart was 100% static (`"0 min"`, zero-width bars) and was also dead code in one place (rendered only from an unreachable branch for the `.strain` route); it now shows real per-zone minutes aggregated from persisted `hr_zone_N_duration` session metrics.
- `strainTargetDisplayText()` was left as `"--"` intentionally — there is no target-strain concept anywhere in the app yet (no setting, no Rust field), so returning empty is the honest behavior, not a bug.

## Stress And Energy Bank

- [ ] Persist daily stress windows instead of only computing the current day in memory. Raw HR already persists 7 days locally (`HeartRateSeriesStores.swift`); no Rust stress table exists yet. Still open.
- [x] Add activity masking to split activity stress from non-activity stress. `StressWindowPoint` now carries `isActivityWindow`, computed by overlap-testing each 10-min bucket against `activity.list_sessions_with_metrics` for that day. The `"non-activity-stress-trend"` row was previously mislabeled — it only excluded sleep windows, not activity time — and now correctly excludes both.
- [x] Use stored sleep windows for sleep stress instead of inferred clock windows. `stressAlgorithmSummary` now uses `store.primarySleepWindow(for:)` (the real bridge-derived sleep window) when available for that day, falling back to the `hour < 7 || hour >= 23` clock heuristic only when no scored sleep window exists.
- [ ] Persist Energy Bank history and compute long-range trends. Entirely unbuilt — no daily rollup table, no read-back bridge method. Still open; needs new Rust work.
- [ ] Calibrate Energy Bank charge/drain rates against stored recovery, sleep, and activity history. Blocked on the app's Calibration feature, which is itself unbuilt for any family. Still open; needs new Rust work.

Also fixed:
- `StressV2OverviewPage` had no `.onAppear` at all; it now refreshes packet inputs and scores like the other pages. Note this doesn't change the Stress page's own score (that's driven by the local HR cache, not packet scores) — it mainly keeps Energy Bank's recovery input and the coach's stress summary fresh.
- `RecoveryV2OverviewPage`'s `.onAppear` was calling `refreshPacketScoresIfNeeded()` but not `refreshPacketInputsIfNeeded()`, so HRV/resting-HR trend rows (which come from `daily_recovery_metrics`/packet-input reports, not packet scores) could stay empty if Recovery was opened before any other packet-input-triggering screen. Now fixed alongside the Stress page.
- Energy Bank silently seeded its whole day's charge/drain simulation from a hardcoded `55` whenever the recovery score was unavailable, and displayed it as a real percentage. It now returns an honest "No recovery score" empty state instead.

## Cardio Load

- [ ] Validate the local Cardio Load formula against multiple real workout sessions. Now unblocked — the formula runs on real device data instead of being unreachable; still needs validation against real workouts once available.
- [ ] Persist computed daily Cardio Load rows so charts do not recompute from raw sessions every render. Still recomputes from raw sessions every call; unchanged by this pass.
- [ ] Confirm HR zone durations from band activity metrics; keep HR fallback only as marked local estimate. The formula already prefers zone durations over the HRR-based HR fallback when available (`cardioLoadContribution`); still needs confirmation against real band data.
- [ ] Add migration/backfill for historical activity sessions once band import supports it. Unchanged.

Also fixed: `cardioLoadAlgorithmSummary()` was a stub that discarded its `range`/`calendar` arguments and unconditionally returned an empty/unavailable summary — every Cardio Load surface (`CardioLoadDetailSurface`, `cardioStatusRows()`, `cardioLoadSnapshot()`) was dead on arrival regardless of local data. Meanwhile a complete, real computation pipeline already existed unused in the same file: `cardioLoadActivitySessions`/`cardioLoadActivityMetricsByName` (real bridge session/metric reads), `cardioLoadContribution` (a real HRR-based or zone-duration-based per-session load formula), and `cardioLoadDailyComputations` (real acute/chronic load + training-status rollup) — plus matching helper functions (`cardioLoadVisibleDayCount`, `cardioLoadDayStarts`, `cardioLoadSessionIsUsable`, `cardioLoadTrainingStatus`, `cardioLoadTrendModel`) that were already written and also unused. `cardioLoadAlgorithmSummary()` now wires these together for real instead of stubbing them out.

## Algorithms, References, Calibration

- [x] Load algorithm and reference definitions only from the Rust bridge. Already done — `refreshBridgeCatalogs()` loads `metrics.built_in_definitions`/`metrics.reference_definitions`/`metrics.default_preferences` and is the only place these are ever assigned; no competing hardcoded list exists.
- [x] Wire reference comparisons to real captured input windows, not static benchmark payloads. `runReferenceComparisons()` was a stub that never touched the bridge. The real Rust comparison engine (`metrics.reference_compare`) was fully built and completely unused; the real per-family inputs it needs were already sitting in `packetScoreReports` from existing score runs. Now wired for real, plus added the missing nav link (`.referenceComparisons` existed as a route but nothing linked to it).
- [ ] Define and persist real calibration labels. Rust schema + bridge methods (`calibration.import_labels`, `calibration_labels` table) are fully built and unused, but there is no Swift UI to actually enter a label value today. Deferred — needs new screens, not just wiring.
- [ ] Implement calibration runs with train/holdout splits from local metric history. `calibration.evaluate_stored_labels` + a real chronological-split, leakage-checked linear-model evaluator are fully built and unused in Rust. Blocked on labels (above) and on `algorithm_runs` actually having rows — see the `persist_algorithm_run` fix below, which is prep work for this but not the full feature.
- [ ] Show calibration outputs only after a completed local calibration run. Falls out for free once the two items above are done. Currently gated on fake local booleans that render hardcoded numbers (`"1 label | manual"`, `"71.5 raw -> 74.2 / 100"`, etc.) — a real bug, but fixing it requires the labels UI above, so left as-is for now.

Also fixed: `runPacketScores()` never passed `persist_algorithm_run: true` to any of the four score bridge calls, so the real `algorithm_runs` table (the "local metric history" calibration needs) has never had a single row written to it. All four (sleep/recovery/strain/stress) now opt in, so real history starts accumulating going forward — pure prep work for the calibration item above, not a full feature.

## Home, Coach, More

- [x] Make Home widgets share the same live/local/bridge/unavailable data contracts as Health. Largely already true — Home builds its snapshots through the same `HealthDataStore` functions Health uses. Two minor, low-priority mismatches found and left as-is: `HomeDashboardView`'s strain-percent parsing duplicates `HealthDataStore.strainPercent(_:)` instead of calling it, and the Cardio Load sparkline draws a fabricated "confidence band" around its one real data point (cosmetic, no text claims a number).
- [x] Ensure Coach prompts explicitly receive current provenance and missing-data states. Found a real gap: Coach's tool context and highlight/data-gap list only covered sleep/recovery/strain/stress — Cardio Load and Energy Bank (both real, computed metrics with their own Home widgets) were completely invisible to Coach. Added `cardio_load`/`energy_bank` to `CoachLocalToolContext.loadStats()`'s scores/provenance and to `CoachOverviewSnapshot`'s highlights/gaps, using the existing real summary functions.
- [ ] Replace remaining placeholder routes with empty states or real screens. Checked all 13 `MoreRoute` cases — every one resolves to a real view, no dead routes or "Coming soon" strings anywhere. The only two non-functional rows (data export/deletion, support bundle composer) are already honestly disabled with truthful labels; making them work needs new Rust bridge methods, not a quick win. Nothing to fix here.
- [x] Remove debug preview-only strings from runtime surfaces before TestFlight builds. Checked — the only debug-only UI is correctly wrapped in `#if DEBUG`, and the only unreachable dead code (`MoreDataStore+Validation.swift`'s preview factories and a hardcoded fake `healthSyncCandidate` function) is never called from anywhere, so there's no runtime leak. Nothing to fix.

Also fixed: `HomeTimelineSection` hardcoded fixed clock times ("06:34", "17:00", "12:30") for its sleep/recovery/activity summary rows regardless of actual data or date — the same fabricated-value bug already fixed elsewhere (sleep's `?? 92`, energy bank's `?? 55`). These now show the real freshness label instead of a fake specific time.
