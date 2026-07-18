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

- [ ] Load algorithm and reference definitions only from the Rust bridge.
- [ ] Wire reference comparisons to real captured input windows, not static benchmark payloads.
- [ ] Define and persist real calibration labels.
- [ ] Implement calibration runs with train/holdout splits from local metric history.
- [ ] Show calibration outputs only after a completed local calibration run.

## Home, Coach, More

- [ ] Make Home widgets share the same live/local/bridge/unavailable data contracts as Health.
- [ ] Ensure Coach prompts explicitly receive current provenance and missing-data states.
- [ ] Replace remaining placeholder routes with empty states or real screens.
- [ ] Remove debug preview-only strings from runtime surfaces before TestFlight builds.
