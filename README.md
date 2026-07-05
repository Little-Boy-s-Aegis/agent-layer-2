# Agent L2 - Banking SOC Orchestrator

This folder stores the Layer 2 orchestrator deliverable for Project Little Boy Aegis.
The orchestrator is the SOAR Decision Engine that sits between Layer 1 sensor agents
and the SOC reporting layer, as shown in the system architecture diagram.

The orchestrator receives Layer 1 findings, verifies them independently against
logs and context, computes final risk, applies policy guardrails, selects
playbooks, records audit events, and reports to SOC channels.

## Main Files

- `layer2_orchestrator_system_prompt.md`: canonical orchestrator system prompt.
- `layer2_orchestrator_output_schema.json`: required JSON output shape.
- `orchestrator_l2_playbooks.md`: playbook routing, response modes, approval
  rules, banking overrides, predictive defense, and quality gates.
- `risk_scoring/`: authoritative Markdown risk scoring tables for the
  orchestrator.
- `mitre_attack_full.json` and `mitre_attack_full.md`: offline MITRE ATT&CK
  knowledge base.

## Runtime Inputs

The orchestrator accepts one Layer 1 finding or an array of Layer 1 findings.
Each finding follows the extended schema `littleboy.soc.layer1.agent_finding.v4` from `../agent-l1/layer1_standard_agent_output_schema.json`.

Required fields in every L1 finding:

```json
{
  "schema_version": "littleboy.soc.layer1.agent_finding.v4",
  "timestamp": "<ISO8601 UTC>",
  "agent_id": "agent_a_internal_network_edr | agent_b_ebanking_api_web_ueba | agent_c_atm_iam_adversarial",
  "agent_name": "<human-readable name>",
  "agent_type": "rule_ml_hybrid | contextual_ai | adversarial_ai",
  "threat_detected": true,
  "finding_type": "confirmed_threat | suspected_threat | anomaly_no_mapping | no_threat | prompt_injection_attempt",
  "capec_id": "CAPEC-### or empty string",
  "mitre_attack_id": "T#### or empty string",
  "raw_evidence": "Masked factual evidence string",
  "safety": {
    "prompt_injection_observed": false,
    "evidence_masked": false
  }
}
```

Optional enrichment fields: `banking_domain_observed`, `entities`, `attack_mapping`, `surfaces_and_context`, `quality`.

Layer 1 output is a sensor signal, not a final verdict. The orchestrator must
verify the finding with available clean logs, raw logs, SIEM, EDR, WAF, API
gateway, IAM, network, database, asset inventory, threat intelligence, vector
database, and local banking context.

## Decision Flow

1. Validate that each Layer 1 input contains the required fields (`schema_version`, `timestamp`, `agent_id`, `agent_name`, `agent_type`, `threat_detected`, `finding_type`, `capec_id`, `mitre_attack_id`, `raw_evidence`, `safety`). Use optional enrichment fields when present.
2. Normalize and correlate findings by entity, time window, technique, CAPEC,
   evidence terms, target surface, and plausible kill-chain sequence.
3. Decide whether multiple findings describe the same attack. Do not use worker
   count as a score, multiplier, cap, or action gate.
4. Independently verify the case from logs and context. A single Layer 1 alert
   can trigger action if the orchestrator verifies real danger.
5. Score final risk with `risk_scoring/attack_vector_risk_scores.md` or
   `risk_scoring/capec_risk_scores.md`, then apply asset criticality and the
   Layer 2 verification cap.
6. If `final_risk_score_0_10 > 6.0`, execute every available non-disruptive SOC
   response floor action: preserve evidence, raise monitoring, add scoped
   watchlists or temporary detections, create hunts, open or update a ticket,
   and notify SOC.
7. Execute environment-changing containment only when every auto-containment
   gate passes.
8. Record audit events, SOC feedback controls, rollback data, notification
   status, and quality limitations in the JSON output.

## Auto-Containment Gate

Environment-changing actions such as block IP, WAF update, quarantine host,
force logout, disable account, limit network, or revoke access may be executed
only when all of these conditions are true:

- Threat is confirmed by the orchestrator.
- Independent verification was performed.
- `verification_state` is `confirmed`.
- `verification_strength` is `supported` or `strong`.
- `final_risk_score_0_10 > 6.0`.
- OPA returns `allow`.
- SOC Autopilot is explicitly ON.
- Current time is inside the execution window.
- The action is low or medium impact, scoped, time-bound, reversible, and not
  manual-only.
- Rollback support exists.
- The confirmed behavior is dangerous now.

If any gate fails, use suggest-only, queued-for-policy-check, or
queued-for-approval status. Do not execute containment automatically.

## Never Automate

The orchestrator must never automatically perform these actions:

- SWIFT isolation
- Core Banking isolation
- HSM isolation
- ATM switch isolation
- critical VLAN isolation
- disaster recovery activation
- broad service isolation
- mass credential reset
- customer notification
- regulator notification
- law enforcement notification
- destructive cleanup
- data deletion

These actions are manual-only even when final risk is high.

## Validation

Use these checks after editing:

```bash
python3 -m json.tool layer2_orchestrator_output_schema.json >/dev/null
find . -name '*.csv' -print
rg -n "verification_modifier|structural_agreement|base_confident" .
rg -n "only when `verification_strength=\\\"strong\\\"`" layer2_orchestrator_system_prompt.md
rg -n "final_risk_score_0_10 > 6.0" layer2_orchestrator_system_prompt.md orchestrator_l2_playbooks.md
```

Expected result:

- JSON parses successfully.
- No CSV files are required.
- Old Layer 1 confidence, consensus, or BFT scoring terms are not used as
  runtime scoring inputs.
- The risk response floor is triggered at `final_risk_score_0_10 > 6.0`.
- Environment-changing containment requires the full auto-containment gate.
