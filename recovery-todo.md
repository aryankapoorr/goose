# Recovery V2 TODO

- HRV and resting HR trend rows are wired to bridge-backed daily series (`daily_recovery_metrics`/feature reports); Recovery V2 now also auto-runs the packet score bridge call on load so the current-day recovery score populates without a manual Packet Inputs run. Still open: the recovery score *trend* row has no daily history, since `metrics.recovery_score_from_features` only computes a single current window. A real multi-day score trend needs a new Rust rollup that reruns the scoring pipeline per historical day.
- Replace the zero vitals cards with trusted packet-derived respiratory rate, SpO2, and wrist-temperature fields once their semantics are verified.
- Add a real recovery timeline model that links the score to the primary sleep window and the packet/vitals inputs used by the score run.
- Add Recovery V2 snapshot tests or simulator screenshots for no-data, bridge-data, and packet-run-blocked states.
- Keep Recovery V2 free of fixture/sample values; show `0` or an empty state until trusted local data exists.
