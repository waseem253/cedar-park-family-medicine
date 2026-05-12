# Cedar Park Family Medicine — HIPAA-Grade AI Receptionist (Portfolio Demo)

A HIPAA-compliant AI voice receptionist for a fictional family medicine practice in Nashville, Tennessee. Demonstrates the engineering, architecture, and documentation work required to ship voice AI into a regulated healthcare environment — *not* just the conversational layer.

This is the second of three portfolio projects targeting the **medical / HIPAA voice AI cluster** ($75–150/hr on Upwork, premium tier).

- 🌐 **Live demo site:** https://cedar-park-family-medicine.vercel.app
- 🗣️ **Talk to Sage:** open the demo site and click "Talk to Sage" (web-only — see [`ARCHITECTURE.md`](./ARCHITECTURE.md#why-no-phone-number) for why no phone number)
- 🎥 **Loom walkthrough:** https://www.loom.com/share/123b6387572949bc99a171db7619a1e2
- 📄 **Case study (1-page PDF):** [`case-study.pdf`](./case-study.pdf)

> ⚠️ **Status:** This is a **design demonstration**. The live demo runs on developer-tier vendor accounts and is explicitly NOT a HIPAA-compliant production deployment. Production deployment requires BAA-tier accounts at every vendor — see [`BAA_VENDOR_MATRIX.md`](./BAA_VENDOR_MATRIX.md) for the full path.

## What it does

Sage, the AI receptionist for Cedar Park Family Medicine, handles five conversation flows:

| Flow | Behavior |
|---|---|
| **New patient intake** | Collects demographics + insurance + chief complaint, schedules new-patient visit |
| **Existing patient booking** | Verifies identity (name + DOB) before any booking action, schedules follow-up |
| **Insurance verification** | Captures plan details, hands off to billing team for eligibility check |
| **Prescription refill request** | Identity verification → captures medication + pharmacy → routes to clinical team |
| **Urgent triage** | Distinguishes "needs same-day visit" from "dial 911 now" |

Every flow enforces HIPAA-grade identity verification, refuses medical advice, and routes anything ambiguous to a human.

## Differentiators vs. a generic voice AI demo

This repo includes two documents that are the actual deliverables for HIPAA clients:

1. **[`BAA_VENDOR_MATRIX.md`](./BAA_VENDOR_MATRIX.md)** — every vendor in the stack mapped to whether they sign a Business Associate Agreement, the required plan, and recommended substitutions where the default tier doesn't qualify
2. **[`HIPAA_COMPLIANCE_AUDIT.md`](./HIPAA_COMPLIANCE_AUDIT.md)** — design audit covering PHI inventory, encryption, access controls, audit logging, retention, patient rights, and breach protocol

These two docs are what hiring clients in regulated healthcare actually need from a voice AI engineer.

## Recommended HIPAA-grade stack

| Component | Choice | BAA Path |
|---|---|---|
| Voice orchestration | Vapi (Enterprise HIPAA tier) | Direct BAA with Vapi |
| LLM | OpenAI GPT-4.1 (ZDR API) or Anthropic Claude via Bedrock | OpenAI direct BAA, or AWS BAA |
| Phone / SMS | Twilio | Self-service BAA |
| Transcription | Deepgram Enterprise | Direct BAA |
| TTS | Vapi built-in or Azure Speech | Inherited / Azure BAA |
| Workflow glue | AWS Lambda + EventBridge | AWS BAA |
| Calendar | Google Workspace Calendar (Business Plus+) | Workspace BAA |
| Audit logs | AWS CloudTrail + S3 (object lock, 6yr retention) | AWS BAA |

Full matrix and verification checklist in [`BAA_VENDOR_MATRIX.md`](./BAA_VENDOR_MATRIX.md).

## Repo structure

```
cedar-park-family-medicine/
├── README.md                          # This file
├── BAA_VENDOR_MATRIX.md               # Vendor-by-vendor BAA analysis + recommended stack
├── HIPAA_COMPLIANCE_AUDIT.md          # PHI handling, encryption, audit logging, retention, breach protocol
├── ARCHITECTURE.md                    # Sequence diagrams + component breakdown + PHI flow map
├── case-study.md                      # 1-page case study for Upwork portfolio
├── case-study.pdf                     # PDF rendering of the case study
├── index.html                         # Demo landing page with embedded Vapi web Talk button
├── prompts/
│   └── sage-system-prompt.md          # Sage's full v1 system prompt (5 flows + HIPAA rules)
└── make-scenarios/                    # Make.com scenario blueprints (production swaps Make.com for AWS Lambda — see BAA matrix)
```

## About the author

Built by **Waseem Iftikhar** — AI Voice Engineer with 7 years of production Python backend experience including HIPAA-compliant US healthcare practice management at PRG (AWS + FastAPI microservices, full compliance audit).

I specialize in production-grade voice AI for regulated industries: HIPAA-grade medical receptionists, multi-tenant clinic platforms, and outbound qualification agents.

**Hire me on Upwork:** https://www.upwork.com/freelancers/~0166938f759e168a91

## License

Portfolio demonstration. Code free for learning. "Cedar Park Family Medicine" is a fictional practice — no business or trademark rights granted.

This repository is engineering guidance, not legal advice. Final HIPAA compliance interpretation should be reviewed by qualified counsel before any production deployment.
