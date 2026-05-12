# HIPAA Compliance Audit — Cedar Park Family Medicine AI Receptionist

This document describes how the Cedar Park Family Medicine voice AI system handles Protected Health Information (PHI) in line with the HIPAA Privacy Rule (45 CFR § 164.500 et seq.) and Security Rule (45 CFR § 164.302 et seq.).

> **Status:** Design audit. This describes the **recommended production architecture**. The public demo at [demo URL — to be added Day 3] runs on developer-tier vendor accounts and is explicitly **NOT** a HIPAA-compliant deployment. It exists to demonstrate the design, not to process real PHI.

## 1. Scope

This audit covers:

- The AI voice receptionist agent (Sage) at Cedar Park Family Medicine
- The phone, voice orchestration, LLM, transcription, calendar, SMS, and workflow components
- All PHI created, received, maintained, or transmitted by the system during patient calls

Out of scope:

- The practice's clinical EHR system (separately audited)
- Provider-to-provider communication
- Paper records

## 2. PHI inventory

The system may collect, process, or transmit the following PHI categories during a call:

| Category | Examples | Where it lives |
|---|---|---|
| **Identifiers** | Name, date of birth, phone number, address | Call transcript (encrypted), calendar event, billing queue |
| **Health information** | Chief complaint (high level), medication names, refill requests, appointment reason | Call transcript, secure messaging queue to clinical team |
| **Payment information** | Insurance provider name, member ID | Call transcript, billing workflow queue |
| **Audio** | Full call recording, speaker-diarized | Vapi storage (BAA-covered), encrypted at rest |

The system does **NOT** collect or store:

- Social Security Numbers
- Full credit card numbers
- Detailed medical history beyond chief complaint
- Lab results, imaging, diagnoses
- Mental-health-specific PHI subject to 42 CFR Part 2 (handled in separate dedicated system if practice expands into mental health)

## 3. Encryption

### In transit
- All API calls between components use **TLS 1.2 or higher** (TLS 1.3 preferred). TLS 1.0/1.1 explicitly disabled.
- Telephony audio is encrypted via **SRTP** (Twilio default) and Vapi's WebRTC stream
- Webhook calls between Vapi and workflow infrastructure use HTTPS with HMAC-signed payloads for authenticity

### At rest
- **Call audio and transcripts:** encrypted in Vapi storage (AES-256, Vapi-managed keys); BAA scope verified
- **Calendar entries:** encrypted in Google Workspace (AES-256, Workspace-managed keys)
- **Workflow execution logs:** encrypted in AWS CloudWatch with KMS customer-managed keys
- **Secure messaging queue** (to clinical/billing teams): AWS SQS with server-side encryption using KMS customer-managed keys
- **Audit log archive:** S3 with object lock (WORM mode), server-side encryption with KMS

## 4. Access controls

- **Multi-factor authentication required** on every administrative interface (Vapi dashboard, AWS console, Google Workspace admin, Twilio console, EHR portal)
- **Role-based access control (RBAC)** — administrative staff get read-only access to call transcripts; only the practice manager and Privacy Officer have delete authority
- **No public exposure of webhook endpoints** — webhooks authenticated via HMAC-signed payloads with rotating shared secrets
- **Service accounts** for system-to-system calls, with credentials rotated every 90 days and stored in AWS Secrets Manager (KMS-encrypted)
- **Principle of least privilege** — every IAM policy reviewed quarterly

## 5. Audit logging

Per HIPAA Security Rule § 164.312(b), all PHI access is logged and retained for **6 years**:

- **Every call:** caller phone (one-way hashed for analytics, raw stored in restricted-access log), agent ID, duration, transcript ID, action taken (booked / messaged team / transferred)
- **Every transcript access by staff:** who, when, transcript ID, reason
- **Every administrative action:** who, what, when, source IP, change diff
- **Logs stored in AWS S3 with object lock enabled** (immutable, WORM) and lifecycle policy retaining 6 years before transition to Glacier Deep Archive and final deletion at year 7

## 6. Retention and deletion

| Data type | Retention | Disposition at end |
|---|---|---|
| Call audio | 90 days (rolling automatic deletion) | Deleted; audit log of deletion retained |
| Call transcript | 6 years (HIPAA minimum; TN medical-records statute aligns) | Deleted; deletion logged |
| Calendar entries | Per practice retention policy (typically 7 years for adult patient records, longer for minors) | Deleted per policy |
| Workflow execution logs | 6 years | Deleted; deletion logged |
| Audit logs | 6 years active + 1 year cold storage | Final deletion at year 7 |
| Backups | Rolling 90 days | Encrypted; expire automatically |

## 7. Patient rights

Per 45 CFR § 164.520 et seq., patients have the right to:

- **Access** their own PHI in the system — request via the practice manager, fulfilled within **30 days**
- **Request amendments** — fulfilled within **60 days**
- **Receive an accounting of disclosures** — provided within **60 days** of request
- **Request restrictions on use** — practice may decline if it would impair care; restrictions on payment-related disclosures must be honored
- **Request confidential communications** — e.g., "do not leave voicemails," "text only," "use email" — honored on a per-patient basis and reflected in Sage's behavior

## 8. Breach notification protocol

Per 45 CFR § 164.404 (covered entity) and § 164.410 (business associate), in the event of a breach affecting PHI:

1. **Discovery → internal notification:** within **24 hours** to the practice manager and HIPAA Privacy Officer
2. **Vendor notification:** within **24 hours** to any affected Business Associate
3. **Patient notification:** within **60 days** via first-class mail or per patient's confidential communication preference
4. **HHS notification:**
   - If breach affects **≥ 500 individuals:** within **60 days** of discovery
   - Otherwise: within **60 days of the end of the calendar year**
5. **Media notification:** if breach affects ≥ 500 residents of a state or jurisdiction
6. **State notification:** Tennessee Identity Theft Deterrence Act (TCA § 47-18-2107) — within **45 days**

Breach response runbook is maintained separately and reviewed annually.

## 9. Workforce training

Per 45 CFR § 164.530(b), all staff with access to PHI complete HIPAA training:

- **Initial training** upon hire (within 30 days)
- **Annual refresher** training (within 12 months of last completion)
- **Targeted training** following any policy change or breach incident
- Training records retained for **6 years** per § 164.530(j)

## 10. Business Associate Agreements

Every vendor with PHI access has an executed BAA on file. See [`BAA_VENDOR_MATRIX.md`](./BAA_VENDOR_MATRIX.md) for the full vendor list, BAA status, and required plan tiers.

BAAs are reviewed annually for currency and re-signed when vendors materially update their terms.

## 11. Risk assessment

Per HIPAA Security Rule § 164.308(a)(1)(ii)(A), a written risk assessment is conducted **annually** and after any major architectural change. The most recent assessment is on file with the practice manager.

Key identified risks and mitigations specific to AI voice systems:

| Risk | Mitigation |
|---|---|
| LLM hallucination producing false medical claims | Sage is prompted to never provide medical advice or diagnose; out-of-scope queries routed to clinical staff; system prompt is locked and changes require Privacy Officer review. |
| Identity-verification bypass | Sage is prompted to NEVER discuss patient-specific information without name + DOB verification; tool calls blocked at the orchestration layer if verification flag is not set. |
| Call recording exposure | 90-day automatic deletion, encryption at rest, access logged, BAA with Vapi. |
| Tool-call exfiltration (LLM tricked into emitting PHI to an unauthorized endpoint) | Only allowlisted webhook endpoints accepted; all tool calls signed and logged; outbound network egress restricted at infrastructure level. |
| Voicemail PHI leak | Sage system prompt explicitly forbids PHI in voicemails; voicemail messages reviewed by Privacy Officer during periodic audits. |
| Prompt injection by malicious caller | Sage refuses to take instructions from callers; meta-instructions embedded in caller speech are ignored; security review of any prompt updates. |

## 12. Disclaimer

This document describes the recommended production architecture for HIPAA compliance and serves as a starting framework for procurement and engineering work.

This is **engineering guidance, not legal advice**. Final compliance interpretation should be reviewed by qualified HIPAA counsel before any production deployment.

Last reviewed: 2026-05-11. Next scheduled review: 2026-11-11 (6 months) or upon any material architectural change.
