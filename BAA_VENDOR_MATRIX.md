# BAA-Covered Vendor Matrix

Vendor-by-vendor analysis of which components in the voice AI stack will sign a Business Associate Agreement (BAA), under what plan tier, and recommended substitutions where the standard tier doesn't qualify.

## Why a BAA matters

Under HIPAA (45 CFR § 164.502(e)), any vendor that creates, receives, maintains, or transmits Protected Health Information (PHI) on behalf of a covered entity (a healthcare provider) must sign a **Business Associate Agreement** before any PHI flows.

A BAA binds the vendor to:

- Use PHI only for the purposes the BAA authorizes
- Protect PHI per the HIPAA Security Rule (encryption, access controls, audit logging)
- Report breaches within 60 days of discovery
- Return or destroy PHI at the end of the engagement
- Pass equivalent BAA terms to any of their own subcontractors

**Operating without a BAA when PHI is in scope = HIPAA violation from the first byte transmitted.** Penalties: up to $1.9M per violation category per calendar year, plus state-level penalties (TN: TCA § 47-18-2107), plus reputational damage that often outlasts the fine.

## Vendor matrix

| Vendor | BAA Available? | Required Plan | Notes |
|---|---|---|---|
| **Vapi** | Yes | Enterprise HIPAA tier (contact sales) | Default developer tier is NOT BAA-covered. Confirm scope of BAA with Vapi sales before any production patient calls. |
| **OpenAI** | Yes | Zero Data Retention (ZDR) API agreement executed via enterprise account manager | Standard API tier does NOT have a BAA. Typically requires enterprise commit; pricing varies. |
| **Anthropic (Claude)** | Yes | Direct enterprise agreement OR via AWS Bedrock (Bedrock inherits AWS BAA, which already covers Bedrock as a HIPAA-eligible service) | The Bedrock path is often the simpler procurement story for clients already on AWS. |
| **Twilio** | Yes | Available on all paid plans — request via Customer Success or self-service in console | One of the easiest BAAs to obtain in this stack. |
| **Deepgram** | Yes | Enterprise plan | Self-serve API tier does NOT cover BAA. |
| **AWS** | Yes | All HIPAA-eligible services covered under AWS BAA, no extra cost | Activate via AWS Organizations console. AWS has the broadest pre-covered service catalog in the industry (Lambda, EventBridge, S3, RDS, SQS, KMS, CloudTrail, etc.). |
| **Google Workspace** | Yes | Business Plus, Enterprise, or Education editions; activate via Admin console | **Free Gmail / personal Google Calendar accounts are NOT BAA-eligible** — common mistake for prototype-to-production migrations. |
| **Microsoft Azure** | Yes | All HIPAA-eligible Azure services covered under Microsoft BAA | Useful for Azure Speech (TTS) if Vapi's built-in voice doesn't fit. |
| **Make.com** | Verify with Make.com directly | Status unclear at standard tiers as of May 2026 | **RECOMMENDED SWAP for HIPAA workloads:** AWS Lambda + EventBridge (BAA-covered), or self-hosted n8n on HIPAA-compliant infrastructure (EC2 with proper config, or ECS Fargate). Make.com is fine for non-HIPAA Project #1 (Bright Smile Dental); not recommended here. |
| **ElevenLabs** | Verify with sales | Likely Enterprise tier only | Recommend swapping to Vapi built-in voice (covered under Vapi BAA) or Azure Speech (BAA-covered via Microsoft). |
| **Calendly** | Yes, with HIPAA add-on | Enterprise + HIPAA add-on | Watch for free-tier trap on Standard plans. |
| **Cal.com** | Self-hosted only | Self-hosted on HIPAA-compliant infrastructure | Cloud version not currently BAA-covered. |

## Recommended HIPAA-grade stack for Cedar Park Family Medicine

| Component | Recommended Choice | BAA Path |
|---|---|---|
| Voice orchestration | Vapi (Enterprise HIPAA tier) | Direct BAA with Vapi |
| LLM | OpenAI GPT-4.1 via ZDR API, OR Anthropic Claude via AWS Bedrock | OpenAI direct BAA, or AWS BAA (Bedrock) |
| Phone number / SMS | Twilio | Self-service BAA via console |
| Voice (TTS) | Vapi built-in OR Azure Speech | Vapi BAA, or Azure BAA |
| Transcription (ASR) | Deepgram Enterprise | Direct BAA |
| Workflow glue | AWS Lambda + EventBridge | AWS BAA |
| Secure message queue | AWS SQS (KMS-encrypted) | AWS BAA |
| Calendar | Google Workspace Calendar (Business Plus+) | Workspace BAA |
| Audit logs | AWS CloudTrail → S3 with object lock (6-year retention, WORM) | AWS BAA |
| Notification routing | Twilio SMS to verified clinical team phone numbers only | Twilio BAA |
| Practice EHR integration | Per-EHR; most major EHRs have BAA-covered FHIR APIs | EHR-specific |

## Verification checklist before production

Before patient calls hit production, every item below must be checked off:

- [ ] BAA signed with **every** vendor in the PHI data path (use the matrix above)
- [ ] Each vendor on a HIPAA-eligible plan — verify in writing, not just by web page claims
- [ ] Subcontractor BAAs verified (e.g., if Vapi uses third-party transcription, verify Vapi's BAA passes through, or sign a direct BAA with Deepgram)
- [ ] TLS 1.2+ verified on every API call (no fallback to TLS 1.0/1.1)
- [ ] Encryption at rest verified for any persisted PHI (AES-256 minimum; KMS-managed keys recommended)
- [ ] Audit logging enabled and retained for at least 6 years (HIPAA Security Rule § 164.312(b))
- [ ] Access controls: RBAC, MFA on every administrative interface, least-privilege service accounts with quarterly key rotation
- [ ] Workforce HIPAA training completed and documented (45 CFR § 164.530(b))
- [ ] Annual security risk assessment performed (45 CFR § 164.308(a)(1)(ii)(A))
- [ ] Breach notification protocol established and tested with every BA — including 24-hour internal notification window
- [ ] Patient-rights process in place: access, amendments, accounting of disclosures, confidential communication preferences
- [ ] Backups encrypted and tested; restore procedures documented
- [ ] Penetration test performed on any custom infrastructure before going live

## Migration path from developer-tier to HIPAA-tier

For practices considering a phased rollout:

1. **Phase 0 — Design (this document)**: identify BAA path for every component
2. **Phase 1 — Procurement**: negotiate and execute BAAs with Vapi, OpenAI, Twilio, Deepgram, Google Workspace, AWS. Estimated time: 2–6 weeks (enterprise BAAs take time).
3. **Phase 2 — Infrastructure**: stand up the HIPAA-grade stack in parallel with the dev environment; run synthetic traffic only.
4. **Phase 3 — Workforce + policies**: complete HIPAA training, document policies, run risk assessment, set up audit log dashboards.
5. **Phase 4 — Pilot**: limited deployment with consenting patients; intensive QA on every flow.
6. **Phase 5 — Production cutover**: full deployment; sunset dev-tier accounts within 30 days.

## Sources and disclaimers

This matrix reflects public vendor documentation as of **May 2026** and is provided as engineering guidance for kickstarting BAA procurement.

**Always verify current BAA terms directly with each vendor** before relying on this information in production — vendor policies change frequently, and the burden is on the covered entity to confirm.

This document is **engineering guidance, not legal advice**. Final HIPAA compliance interpretation should be reviewed by qualified counsel before processing real PHI.
