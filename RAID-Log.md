# RAID Log — IM, Financial Model & GDG OTC (Outline Terms and Conditions) Generation

Nationwide Building Society (Client) — Business Banking / AI CoE
Cross-checked against the customer's own Solution Definition Document (SDD) for alignment.
Items marked *(suggested)* are not yet in the source SDD/OneNote — proposed here, pending sign-off from the document owner (Ashish Ranjan) before being treated as final.

---

## 7.1 Risks

| # | Risk | Description | Owner | Probability | Impact | Mitigation | Managed outside (ref/link) |
|---|---|---|---|---|---|---|---|
| 1 | External site unavailability | Confluence/OneDrive/Teams becomes unavailable, blocking access to source documents or the generation workflow | Platform/IT Service Owner (M365) | Low | Medium — temporary inability to initiate/complete OTC generation | Multiple evidence sources | N/A — covered by M365 service SLAs |
| 2 | AI hallucinations | AI agents may generate content not supported by the IM/Financial Model/GDG evidence | AI CoE / Solution Owner | Medium | High — incorrect OTC content if unchecked | Human approval mandatory | N/A |
| 3 | Data privacy concerns | IM/Financial Model/GDG contain commercially sensitive data; incorrect access or retention could breach data-protection obligations | Data & Analytics / DPO | Low | High | Secure access and retention controls | N/A |
| 4 | Increased usage volumes | Usage beyond the initial ~40-user base could exceed provisioned capacity | Technology Lead (Adrian Sanchez) | Medium | Medium | Autoscaling Azure services | N/A |
| 5 | Inconsistent evidence quality | Source documents vary in format/completeness, reducing extraction accuracy | Business Banking Operations | Medium | Medium | Confidence scoring and validation logic | N/A |
| 6 | Dependency on Teams Guest Account setup and lifecycle management | VM users are onboarded as Client Teams Guest Accounts; delays or gaps in that process block access | Identity & Access Management (Client Entra) | Medium | User onboarding delays and support overhead | Align to existing Client joiner-mover-leaver process; review lifecycle ahead of rollout | N/A — ref existing Client IAM guest-account policy |
| 7 | Business process depends entirely on Microsoft Teams availability | By design the solution has no channel outside Teams | Platform/IT Service Owner (M365) | Low | Process disruption during Teams outages | Accept against existing M365 SLA monitoring; no separate fallback planned (Teams-only by design) | N/A |
| 8 | IM and supporting documents subject to Teams upload limitations | Teams/OneDrive file-size or format limits may block large IM/Financial Model uploads | Business Banking Operations | Medium | Reduced user experience and failed uploads | Confirm Teams/OneDrive limits against typical file sizes pre-rollout; provide guidance | N/A |
| 9 | External guest access restrictions may impact enterprise integrations | Conditional-access policy on guest accounts may be stricter than the VM-user population needs | Identity & Access Management (Client Entra) | Low | Access and support issues | Confirm guest conditional-access policy compatibility before go-live | N/A |
| 10 *(suggested)* | Scope definition (Section 4.4) may be inaccurate | In/out-of-scope table reads as carried over from the sibling "Trading Address Verification" SDD, not rewritten for OTC | Lead Architect (Ashish Ranjan) | Medium | Medium — governance sign-off against wrong scope | Review and correct Section 4.4 with document author before governance approval | N/A |

---

## 7.2 Assumptions

| # | Assumption |
|---|---|
| 1 | Azure AI Foundry is approved for production use |
| 2 | User authentication will leverage existing Entra ID |
| 3 | Evidence retention policies will be defined by Compliance |
| 4 | Initial user base remains approximately 40 users before wider rollout |
| 5 *(suggested)* | VM (guest) users will be provisioned and deprovisioned via Client's existing Teams Guest Account lifecycle process — no new identity workflow required |

---

## 7.3 Issues

| # | Issue | Description | Impact |
|---|---|---|---|
| 1 | Retention policy definition pending | Evidence/document retention duration and deletion rules for OTC source and generated documents have not yet been finalised by Compliance | Medium — interim retention falls back to default M365/OneDrive policy until confirmed |
| 2 | Final NFR approval pending | Hosting-pattern NFR (Section 5.7 — in-house UK vs supplier-hosted) has not been confirmed/selected in the reviewed version | Medium — downstream Azure deployment decisions can't be finalised until resolved |
| 3 *(suggested)* | Section 4.4 scope table inconsistency | In/out-of-scope items appear copied from the sibling TAV SDD rather than written for this initiative | Medium — risk of incorrect scope being signed off |

---

## 7.4 Dependency

| # | Dependency |
|---|---|
| 1 | Application access by Teams Account for VM users |
| 2 | Security approvals |
| 3 | Compliance approval for evidence retention |
| 4 *(suggested)* | NFR hosting-pattern decision (Section 5.7) confirmed before architecture sign-off |
| 5 *(suggested)* | Azure AI Foundry production-use approval (ties to Assumption #1) landing before go-live |

---

*Compiled 2026-11-24. Risks/Assumptions/Issues/Dependencies without a "(suggested)" tag are sourced from the customer's own SDD and OneNote working notes; "(suggested)" items are proposed additions pending the document owner's confirmation.*
