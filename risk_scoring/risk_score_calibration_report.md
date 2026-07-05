# Risk Score Calibration Report

This report records the July 2026 banking SOC calibration pass for Layer 2 base risk tables.

## Summary

- ATT&CK rows: 943; changed: 354; min/max: 2.1/10.0.
- CAPEC rows: 558; changed: 197; min/max: 1.7/10.0.
- L1 worker count, confidence, BFT, and consensus are not scoring inputs.
- L2 still applies asset criticality and independent-verification caps after this base lookup.

## Main Fixes

- Raised public-facing exploit, confirmed execution, IAM/MFA/session abuse, lateral movement, defense impairment, and exfiltration classes that were too low for banking SOC use.
- Capped generic C2 carriers, generic collection/staging, generic XSS, and generic DoS patterns that were too high without confirmed impact.
- Preserved critical scores for ransomware, destructive impact, recovery inhibition, confirmed high-fidelity exfiltration, and financial theft.

## Spot Checks

| ID | Name | Previous | Calibrated | Reason |
| --- | --- | --- | --- | --- |
| T1190 | Exploit Public-Facing Application | 5.0 | 7.5 | Public-facing exploitation is high in banking; impact is refined by asset multiplier and L2 verification. |
| T1059 | Command and Scripting Interpreter | 4.2 | 6.5 | Confirmed execution should not be scored as low/weak medium. Command/scripting execution is elevated when confirmed. |
| T1059.001 | PowerShell | 5.0 | 6.5 | Confirmed execution should not be scored as low/weak medium. Command/scripting execution is elevated when confirmed. |
| T1556 | Modify Authentication Process | 6.2 | 8.0 | Credential or authentication abuse is high-risk in banking. Identity/session/MFA bypass receives a banking-sensitive floor. Authentication process/MFA modification can bypass bank identity controls. |
| T1111 | Multi-Factor Authentication Interception | 6.2 | 7.5 | Credential or authentication abuse is high-risk in banking. Identity/session/MFA bypass receives a banking-sensitive floor. |
| T1550 | Use Alternate Authentication Material | 6.2 | 7.5 | Credential or authentication abuse is high-risk in banking. Identity/session/MFA bypass receives a banking-sensitive floor. |
| T1686 | Disable or Modify System Firewall | 6.2 | 7.5 | Defense impairment should be high once confirmed. Firewall/tool/control tampering weakens bank defenses and receives high floor. |
| T1071.001 | Web Protocols | 10.0 | 8.0 | Web protocol C2 is a common carrier; cap below critical until L2 verifies active command/exfil. |
| T1005 | Data from Local System | 10.0 | 7.5 | Local data access is high; critical requires confirmed sensitive data/exfiltration. |
| T1560 | Archive Collected Data | 10.0 | 7.5 | Archive collected data is staging; critical requires exfiltration or ransomware context. |
| T1486 | Data Encrypted for Impact | 10.0 | 10.0 | Ransomware/data encryption remains critical. |
| T1657 | Financial Theft | 10.0 | 10.0 | Financial theft remains critical for banking. |

| CAPEC ID | Name | Previous | Calibrated | Reason |
| --- | --- | --- | --- | --- |
| CAPEC-98 | Phishing | 4.2 | 6.5 | CAPEC Very High severity receives elevated floor. Very High severity plus High likelihood receives high-risk floor. Phishing is medium-high; credential theft or execution evidence raises final risk. |
| CAPEC-142 | DNS Cache Poisoning | 2.1 | 7.5 | CAPEC High severity should not remain low. CAPEC High severity with meaningful likelihood is elevated. DNS cache poisoning can redirect banking users/services and should be high. |
| CAPEC-18 | XSS Targeting Non-Script Elements | 10.0 | 7.0 | Generic XSS is capped below critical until token theft/account takeover is verified. |
| CAPEC-275 | DNS Rebinding | 10.0 | 7.5 | DNS rebinding is high but not automatically critical without confirmed internal access. |
| CAPEC-126 | Path Traversal | 10.0 | 7.5 | Path traversal is high; critical requires sensitive-file confirmation. |
| CAPEC-150 | Collect Data from Common Resource Locations | 10.0 | 8.5 | Sensitive-data pattern reaches max critical through verified data exposure. |
| CAPEC-204 | Lifting Sensitive Data Embedded in Cache | 10.0 | 8.5 | Sensitive-data pattern reaches max critical through verified data exposure. |
