# Agent L2 - Banking SOC Orchestrator

> **Part of the [Little Boy's Aegis](https://github.com/Little-Boy-s-Aegis) project** -- an AI-driven Security Operations Center (SOC) platform for banking infrastructure.

This folder stores the Layer 2 orchestrator deliverable for Project Little Boy Aegis.
The orchestrator is the SOAR Decision Engine that sits between Layer 1 sensor agents
and the SOC reporting layer, as shown in the system architecture diagram.

Canonical path: `/mnt/d/Hackathon-2026/agent-l2`. This is the current source of
truth for the Layer 2 deliverable. If a product-copy mirror is restored later,
syncing that mirror is a separate release step and is not required by this
runtime contract.

The orchestrator receives Layer 1 findings, verifies them independently against
logs and context, computes final risk, applies policy guardrails, selects
playbooks, records audit events, and reports to SOC channels.

## Main Files

- `layer2_orchestrator_system_prompt.md`: canonical orchestrator system prompt.
- `layer2_orchestrator_output_schema.json`: JSON Schema Draft 2020-12
  validation contract for `littleboy.soc.layer2.orchestrator_decision.v8`.
- `layer2_orchestrator_output_example.json`: non-normative example output only;
  use it as a filled example, not as the validation contract.
- `orchestrator_l2_playbooks.md`: playbook routing, response modes, approval
  rules, banking overrides, predictive defense, and quality gates.
- `l1_l2_system_design.md`: behavioral system design, subagent duties, and
  L1/L2 boundary contract.
- `risk_scoring/`: authoritative Markdown risk scoring tables for the
  orchestrator.
- `mitre_attack_full.json` and `mitre_attack_full.md`: offline MITRE ATT&CK
  knowledge base.

## Runtime Inputs

The orchestrator accepts one Layer 1 finding or an array of Layer 1 findings.
Each finding follows the extended schema `littleboy.soc.layer1.agent_finding.v4` from `../agent-l1/agent_*/layer1_standard_agent_output_schema.json`.

Required fields in every L1 finding:

```json
{
  "schema_version": "littleboy.soc.layer1.agent_finding.v4",
  "timestamp": "<ISO8601 UTC>",
  "agent_id": "agent_a_internal_network_edr | agent_b_ebanking_api_web_ueba | agent_c_atm_iam_adversarial",
  "agent_name": "<human-readable name>",
  "agent_type": "rule_ml_hybrid | contextual_ai | adversarial_ai",
  "threat_detected": true,
  "finding_type": "observed_threat_pattern | observed_suspicious_pattern | anomaly_no_mapping | no_threat | prompt_injection_attempt",
  "capec_id": "CAPEC-### or empty string",
  "mitre_attack_id": "T#### or empty string",
  "raw_evidence": "Masked factual evidence string",
  "safety": {
    "prompt_injection_observed": false,
    "evidence_masked": false
  }
}
```

Optional enrichment fields: `banking_domain_observed`, `entities`, `attack_mapping`, `surfaces_and_context`, `attack_pattern_prediction`, `quality`.

Layer 1 output is a sensor signal, not a final verdict. Any
`attack_pattern_prediction` from Layer 1 is a hypothesis for verification, not a
response decision. The orchestrator must verify the finding with available clean
logs, raw logs, SIEM, EDR, WAF, API gateway, IAM, network, database, asset
inventory, threat intelligence, vector database, and local banking context.

## Decision Flow

1. Validate that each Layer 1 input contains the required fields (`schema_version`, `timestamp`, `agent_id`, `agent_name`, `agent_type`, `threat_detected`, `finding_type`, `capec_id`, `mitre_attack_id`, `raw_evidence`, `safety`). Use optional enrichment fields, including `attack_pattern_prediction`, when present.
2. Normalize and correlate findings by entity, time window, technique, CAPEC,
   evidence terms, target surface, and plausible kill-chain sequence.
3. Decide whether multiple findings describe the same attack. Require at least
   one concrete entity link such as user, account, host, IP, session, request,
   transaction, or object ID. Same time window, same broad domain, or multiple
   workers reporting is not enough by itself. Findings unrelated to the primary
   incident must be listed in `correlation.unrelated_findings[]` with a handling
   recommendation.
4. Independently verify the case from logs and context. A single Layer 1 alert
   can trigger action if the orchestrator verifies real danger.
5. Score final risk with `risk_scoring/attack_vector_risk_scores.md` or
   `risk_scoring/capec_risk_scores.md`, then apply asset criticality and the
   Layer 2 verification cap.
6. If `final_risk_score_0_10 > 5.0`, execute every available non-disruptive SOC
   response floor action: preserve evidence, raise monitoring, add scoped
   watchlists or temporary detections, create hunts, open or update a ticket,
   and notify SOC.
7. Populate `automation_control.auto_containment_gates` and execute
   environment-changing containment only when every gate is `true`.
8. Record audit events, SOC feedback controls, rollback data, notification
   status, and quality limitations in the JSON output.

## Auto-Containment Gate

Environment-changing actions such as block IP, WAF update, quarantine host,
force logout, disable account, limit network, or revoke access may be executed
only when all of these `automation_control.auto_containment_gates` fields are
true:

- `threat_confirmed`
- `l2_verification_performed`
- `verification_confirmed`
- `verification_supported_or_strong`
- `risk_above_floor`
- `opa_allow`
- `soc_autopilot_on`
- `execution_window_open`
- `action_scoped_timebound_reversible`
- `rollback_available`
- `dangerous_now_behavior`
- `verified_target_entity`

If any gate fails, use suggest-only, queued-for-policy-check, or
queued-for-approval status. Do not execute containment automatically.

Every action in `actions[]` must include `priority`, `timestamp`, `target`,
`approval_mode`, `status`, and `playbook_source` so the SOC UI and audit path
can render the decision deterministically.

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
! rg -n "littleboy\\.soc\\.layer2\\.orchestrator_decision\\.v[0-7]" .
! rg -n 'automatic containment.*verification_strength="strong"|verification_strength="strong".*automatic containment' layer2_orchestrator_system_prompt.md
rg -n "final_risk_score_0_10 > 5.0" layer2_orchestrator_system_prompt.md orchestrator_l2_playbooks.md
```

Expected result:

- JSON parses successfully.
- Layer 2 output schema is v8 and documented as the Draft 2020-12 validation
  contract; the example file is documented as example-only.
- No CSV files are required.
- Old Layer 1 confidence, consensus, or BFT scoring terms are not used as
  runtime scoring inputs.
- The risk response floor is triggered at `final_risk_score_0_10 > 5.0`.
- Environment-changing containment requires every
  `automation_control.auto_containment_gates` field to be true.

---

## Related Repositories

All repositories under the [Little Boy's Aegis](https://github.com/Little-Boy-s-Aegis) organization:

| Repository | Description |
|---|---|
| [aegis-bank-deployment](https://github.com/Little-Boy-s-Aegis/aegis-bank-deployment) | Docker Compose orchestration for the full platform |
| [aegis-bank-backend](https://github.com/Little-Boy-s-Aegis/aegis-bank-backend) | Spring Boot banking API |
| [aegis-bank-web-client](https://github.com/Little-Boy-s-Aegis/aegis-bank-web-client) | Next.js web banking portal |
| [aegis-bank-mobile-app](https://github.com/Little-Boy-s-Aegis/aegis-bank-mobile-app) | Flutter mobile banking app |
| [dashboard](https://github.com/Little-Boy-s-Aegis/dashboard) | SOC Dashboard -- Go backend + React frontend |
| [agent-layer-1](https://github.com/Little-Boy-s-Aegis/agent-layer-1) | AI Sensor Agents (Layer 1) |
| [aegis-soar-engine](https://github.com/Little-Boy-s-Aegis/aegis-soar-engine) | SOAR Decision Engine |
| [aegis-staging-sandbox](https://github.com/Little-Boy-s-Aegis/aegis-staging-sandbox) | Staging Sandbox environment |
| [aegis-bank-terraform](https://github.com/Little-Boy-s-Aegis/aegis-bank-terraform) | Terraform IaC for cloud deployment |
