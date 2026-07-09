# Layer 2 Risk Scoring Tables

These files are the authoritative runtime scoring inputs for Layer 2.

They were derived from the legacy Layer 1 threat grading references, but they remove Layer 1 vote, confidence, structural agreement, BFT, and auto-containment columns. Layer 1 must not calculate or emit risk score, confidence score, priority, response mode, or containment decisions.

Layer 2 uses:

- `attack_vector_risk_scores.md` for MITRE ATT&CK base threat scores.
- `capec_risk_scores.md` for CAPEC base threat scores.
- `surface_multiplier_matrix.md` and `edge_case_matrix.md` only as context for surface and blast-radius reasoning. They do not provide runtime final risk scores.
- `edge_case_summary.md` as a short reference for edge-case classes.
- `risk_score_calibration_report.md` as the human-readable audit trail for the banking SOC calibration pass.

Runtime formula:

1. Look up `base_threat_score_0_10` by MITRE technique or CAPEC ID from the Markdown tables.
2. Apply banking asset criticality to compute `raw_context_risk_0_10`.
3. Apply the Layer 2 independent verification cap.
4. Compute `final_risk_score_0_10`.
5. If final risk is greater than `6.0`, Layer 2 must perform allowed non-disruptive SOC actions and may execute eligible environment-changing containment only when every field in `automation_control.auto_containment_gates` is true: `threat_confirmed`, `l2_verification_performed`, `verification_confirmed`, `verification_supported_or_strong`, `risk_above_floor`, `opa_allow`, `soc_autopilot_on`, `execution_window_open`, `action_scoped_timebound_reversible`, `rollback_available`, `dangerous_now_behavior`, and `verified_target_entity`.

Verification cap: `contradicted` caps final risk at `2.0`; `not_confirmed`, `insufficient`, `not_required`, `sources_unavailable`, `query_timeout`, `stale_context`, and `not_found` cap final risk at `5.5`. Only `confirmed` with `supported` or `strong` verification can exceed `5.5`.

Layer 1 worker count is not part of this formula.
