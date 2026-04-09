# 🏠 AI Real Estate Calling System — No-Code MVP

> An AI-powered outbound calling platform for real estate — built entirely with SaaS tools, no servers or code required. Pulls leads from Follow Up Boss, calls them via a Synthflow AI voice agent, and logs outcomes back automatically.

[![CRM: Follow Up Boss](https://img.shields.io/badge/CRM-Follow%20Up%20Boss-green.svg)](https://followupboss.com)
[![Voice: Synthflow](https://img.shields.io/badge/Voice-Synthflow-brightgreen.svg)](https://synthflow.ai)
[![Automation: Zapier](https://img.shields.io/badge/Automation-Zapier-orange.svg)](https://zapier.com)
[![Compliance: CRTC / TCPA](https://img.shields.io/badge/Compliance-CRTC%20%2F%20TCPA-blue.svg)](#compliance)
[![No Code](https://img.shields.io/badge/No%20Code-100%25-purple.svg)](#the-three-tool-stack)

---

## Table of Contents

- [Overview](#overview)
- [The Three-Tool Stack](#the-three-tool-stack)
- [How It Works](#how-it-works)
- [Integration Flow](#integration-flow)
- [Zapier Setup](#zapier-setup)
- [Synthflow Setup](#synthflow-setup)
- [Follow Up Boss Setup](#follow-up-boss-setup)
- [Build Phases](#build-phases)
- [Compliance](#compliance)
- [DNC — Do Not Call](#dnc--do-not-call)
- [Costs](#costs)
- [Scaling Beyond MVP](#scaling-beyond-mvp)

---

## Overview

This is **Version 2** of the AI Real Estate Calling System — a no-code MVP.

```
Follow Up Boss  →  Zapier  →  Synthflow  →  Lead's phone
      ↑                               |
      └───────── Zapier ──────────────┘
             (writes outcome back)
```

**No EC2. No Podman. No Python. No GitHub deploys.**
The entire system runs from three browser tabs.

---

## The Three-Tool Stack

| Tool | Role | Replaces |
|---|---|---|
| **Follow Up Boss** | CRM — leads in, outcomes out | — |
| **Synthflow** | AI voice agent + campaign manager + dashboard | EC2 + FastAPI + Redis + voice platform code |
| **Zapier** | Connects FUB to Synthflow, enforces DNC and time rules | Worker queue + scheduler + webhook server |

### Why Synthflow for MVP

- True no-code setup — agent live in under an hour
- Built-in campaign manager with retry and voicemail detection
- ElevenLabs voices bundled — no separate TTS account needed
- Call transcripts and outcome dashboard included
- 200+ integrations including Zapier and GoHighLevel
- 14-day free trial — test before committing
- Real estate explicitly listed as a supported use case

---

## How It Works

```mermaid
flowchart TD
    A([Lead enters Follow Up Boss\nor changes stage]) --> B

    subgraph ZAPIER ["Zapier — Trigger and Filter"]
        B[Trigger: FUB new lead\nor stage change] --> C
        C{DNC tag\npresent?} -->|Yes| STOP([Stop - do not call])
        C -->|No| D
        D{9am to 9pm\nlocal time?} -->|No| WAIT([Delay until\ncalling window])
        D -->|Yes| E
        E[Format payload\nname + phone + property]
    end

    subgraph SYNTHFLOW ["Synthflow — Voice Agent"]
        E --> F[Lead added to campaign]
        F --> G[AI agent dials lead]
        G --> H[Live conversation\npitch + objections + CTA]
        H --> I{Outcome}
        I -->|Interested| J[Webhook fires to Zapier]
        I -->|No answer| K[Auto-retry after delay]
        I -->|Not interested| L[Webhook fires to Zapier]
        I -->|Voicemail| M[Leave voicemail\nthen schedule retry]
        K --> G
        M --> K
    end

    subgraph WRITEBACK ["Zapier — Write Back to FUB"]
        J --> N[Stage to Hot Lead\nCreate agent task\nPost transcript note]
        L --> O[Add DNC tag\nStage to Dead\nPost note]
    end

    style ZAPIER fill:#FFF8E1,stroke:#F9A825,color:#5D3E00
    style SYNTHFLOW fill:#E8F5E9,stroke:#2E7D32,color:#1B5E20
    style WRITEBACK fill:#E3F2FD,stroke:#1565C0,color:#0D47A1
    style STOP fill:#FFEBEE,stroke:#C62828,color:#B71C1C
    style WAIT fill:#FFF3E0,stroke:#E65100,color:#BF360C
```

---

## Integration Flow

### Outbound — FUB to Synthflow

```mermaid
flowchart LR
    Z1[Trigger\nFUB - Updated Person\nstage = New Inquiry] -->
    Z2[Filter\nDNC tag NOT present] -->
    Z3[Filter\n9am to 9pm\nlead timezone] -->
    Z4[Formatter\nMap FUB fields to\nSynthflow format] -->
    Z5[Synthflow\nAdd lead to\nactive campaign]

    style Z1 fill:#E3F2FD,stroke:#1565C0,color:#0D47A1
    style Z2 fill:#E8F5E9,stroke:#2E7D32,color:#1B5E20
    style Z3 fill:#E8F5E9,stroke:#2E7D32,color:#1B5E20
    style Z4 fill:#FFF8E1,stroke:#F9A825,color:#5D3E00
    style Z5 fill:#F3E5F5,stroke:#7B1FA2,color:#4A0072
```

### Return — Synthflow outcome to FUB

```mermaid
flowchart LR
    R1[Trigger\nSynthflow webhook\nfires on call end] -->
    R2[Router\nbranch on\noutcome code]
    R2 --> R3A[Interested\nUpdate stage\nCreate task\nPost transcript]
    R2 --> R3B[No answer\nUpdate stage\nPost note\nSchedule retry]
    R2 --> R3C[Not interested\nAdd DNC tag\nStage to Dead\nPost note]

    style R1 fill:#E3F2FD,stroke:#1565C0,color:#0D47A1
    style R2 fill:#FFF8E1,stroke:#F9A825,color:#5D3E00
    style R3A fill:#E8F5E9,stroke:#2E7D32,color:#1B5E20
    style R3B fill:#FFF3E0,stroke:#E65100,color:#BF360C
    style R3C fill:#FFEBEE,stroke:#C62828,color:#B71C1C
```

---

## Zapier Setup

### Zap 1 — Outbound trigger

| Step | Type | Configuration |
|---|---|---|
| 1 | **Trigger** | Follow Up Boss — New or Updated Person |
| 2 | **Filter** | Only continue if tag `DNC` is NOT present |
| 3 | **Filter** | Only continue if current time is between 09:00 and 21:00 in lead's timezone |
| 4 | **Formatter** | Map FUB fields: `name`, `phone`, `stage`, `assigned agent` |
| 5 | **Action** | Synthflow — Add Contact to Campaign |

### Zap 2 — Post-call return

| Step | Type | Configuration |
|---|---|---|
| 1 | **Trigger** | Webhooks by Zapier — Catch Hook (Synthflow fires this on call end) |
| 2 | **Router** | Branch on `disposition` field from Synthflow payload |
| 3a | **Branch: Interested** | FUB Update Person (stage → Hot Lead) + Create Note + Create Task |
| 3b | **Branch: No Answer** | FUB Update Person (stage → Contacted) + Create Note |
| 3c | **Branch: Not Interested** | FUB Update Person (add tag DNC, stage → Dead) + Create Note |

> **Tip:** Synthflow's webhook payload includes `transcript`, `duration_seconds`, `outcome`, and `recording_url`. Map all four into the FUB note so agents have full context on every call.

---

## Synthflow Setup

### 1. Create your AI agent

In the Synthflow dashboard:

- **Voice** — choose from the ElevenLabs library or clone your own in one click
- **Language** — English, French, and Spanish supported out of the box
- **Prompt** — paste the template below and fill in your property details

### Agent prompt template

```
You are [Agent Name], a friendly real estate assistant calling on behalf of [Company Name].

You are calling about [Property Name] located at [Address], listed at [Price].

Your goal is to gauge the lead's interest and offer to schedule a viewing.

Guidelines:
- Keep the call under 2 minutes
- Introduce yourself and the company within the first 10 seconds
- State the property address and price clearly and early
- Ask one clear question: "Would you be interested in scheduling a viewing this week?"
- If they say not interested: thank them warmly and end the call politely
- If they say call back later: ask for a preferred time and confirm it
- If they ask a question you cannot answer: offer to have an agent call them back
- Never fabricate property details you were not given
- Always identify yourself as an AI assistant if asked directly
```

### 2. Configure your campaign

In Synthflow Campaigns:

| Setting | Value |
|---|---|
| **Call window** | 9:00am – 9:00pm (matches CRTC / TCPA requirement) |
| **Retry attempts** | 2–3 retries, spaced 4 hours apart |
| **Voicemail detection** | Enabled — drop a short pre-recorded message |
| **Webhook on completion** | Paste the Zapier Catch Hook URL from Zap 2 |
| **Concurrent calls** | Start at 3–5, increase once call quality is confirmed |

### 3. Connect to Zapier

In Synthflow → Integrations → Zapier:
- Enable the integration
- Copy your Synthflow API key
- Paste into the Zapier Synthflow app connection

---

## Follow Up Boss Setup

### Stages to create

| Stage | Trigger | Meaning |
|---|---|---|
| `New Inquiry` | Lead enters FUB | Triggers outbound Zap 1 |
| `Contacted` | Zap 2 — no answer | Call placed, in retry queue |
| `Hot Lead` | Zap 2 — interested | Agent follow-up required today |
| `Dead` | Zap 2 — not interested | No further automated contact |

### Tags to create

| Tag | Set by | Meaning |
|---|---|---|
| `DNC` | Agent manually or Zap 2 automatically | Never call this lead again |
| `AI Called` | Zap 2 on every call completion | Audit trail — AI has contacted this person |

### How to manually mark a lead DNC

1. Open the lead profile in FUB
2. Click **Add Tag** on the left panel
3. Type `DNC` and save

Any lead with the `DNC` tag is caught by the Zapier filter in Zap 1 and never receives an automated call — regardless of which campaign or stage they are in.

---

## Build Phases

```mermaid
gantt
    title AI Real Estate Calling System — No-Code MVP (Synthflow)
    dateFormat  YYYY-MM-DD

    section Phase 0 — Setup
    Create Synthflow account             :done, p0a, 2026-04-13, 1d
    Create Zapier account                :done, p0b, 2026-04-13, 1d
    Get FUB API key from settings        :done, p0c, 2026-04-13, 1d

    section Phase 1 — Connect the Tools
    Connect FUB to Zapier                :p1a, 2026-04-20, 2d
    Connect Synthflow to Zapier          :p1b, after p1a, 2d
    Test FUB trigger fires correctly     :p1c, after p1b, 1d

    section Phase 2 — Build the Voice Agent
    Write agent script and prompt        :p2a, after p1c, 3d
    Configure voice and language in SF   :p2b, after p2a, 1d
    Set up campaign in Synthflow         :p2c, after p2b, 2d
    Test call end to end                 :p2d, after p2c, 2d

    section Phase 3 — Zapier Automation
    Zap 1 - FUB trigger to SF campaign   :p3a, after p2d, 2d
    Add DNC filter step to Zap 1         :p3b, after p3a, 1d
    Add time window filter to Zap 1      :p3c, after p3b, 1d
    Zap 2 - SF outcome to FUB writeback  :p3d, after p3c, 2d
    Test full loop with real lead         :p3e, after p3d, 2d

    section Phase 4 — Compliance and Launch
    Scrub lead list against DNC registry :p4a, after p3e, 2d
    Tag existing DNC leads in FUB        :p4b, after p4a, 1d
    Review CRTC time window settings     :p4c, after p4b, 1d
    Soft launch - small batch test       :p4d, after p4c, 3d
    Full campaign launch                 :p4e, after p4d, 1d
```

---

## Compliance

> Automated outbound calls are regulated. These rules are **legal requirements**, not suggestions.

### CRTC (Canada)

- [ ] Only call between **9am and 9pm local time** — set in Synthflow campaign settings
- [ ] Agent must **identify the business name** within the first 10 seconds of the call
- [ ] Provide an opt-out option: *"Press 9 to be removed from our list"* — configure in Synthflow
- [ ] Honour DNC requests **within 14 days** — automated via Zap 2 tagging
- [ ] Do not call numbers on the **National DNCL** — scrub before every campaign launch
- [ ] Retain call records for **minimum 24 months** — Synthflow stores transcripts; export monthly

### TCPA (United States)

- [ ] **Prior written consent required** for auto-dialed calls to cell phones
- [ ] Property inquiry = implied consent for **90 days** from inquiry date
- [ ] Prior completed transaction (sale / purchase) = **18 months**
- [ ] Honour the **National Do Not Call Registry**

---

## DNC — Do Not Call

A lead must be marked DNC in three situations:

| Situation | How the tag gets added |
|---|---|
| Lead says "remove me" or "don't call" during the AI call | Zap 2 adds `DNC` tag automatically via Synthflow webhook |
| Lead phones in and asks to be removed | Agent adds `DNC` tag manually in FUB |
| Number appears on National DNC Registry pre-scrub | Tag added during pre-campaign scrub before launch |

Once `DNC` is on a profile, **Zap 1 blocks that lead permanently** — no campaign will ever call them again.

To scrub your lead list before launch, use [CallAction Verify](https://callaction.co) — it checks FUB phone numbers against the National DNC Registry and flags matches directly in your account.

---

## Costs

### Monthly estimate at MVP scale (~1,000 calls/month)

| Tool | Plan | Approx. cost |
|---|---|---|
| **Synthflow** | Pro — 2,000 minutes included | ~$375/month |
| **Zapier** | Starter — 750 tasks/month | ~$20/month |
| **Follow Up Boss** | Your existing plan | — |
| **CallAction Verify** | DNC scrub — one-time per campaign | ~$30–50/scrub |
| **Total** | | **~$425–445/month** |

> At 1,000 calls averaging 2 minutes each = 2,000 minutes. Synthflow Pro covers this exactly. If calls average under 1.5 minutes you will stay comfortably within the cap.

### Cost per call

```
$375 Synthflow / 1,000 calls = $0.375 per call attempt
```

Compare to a human ISA at $15–25/hour making 20–30 calls/hour = **$0.50–$1.25 per call attempt** before salary overhead.

---

## Reference Links

| Resource | URL |
|---|---|
| Synthflow Dashboard | https://app.synthflow.ai |
| Synthflow Docs | https://docs.synthflow.ai |
| Synthflow + Zapier | https://zapier.com/apps/synthflow/integrations |
| Follow Up Boss API | https://docs.followupboss.com |
| Follow Up Boss + Zapier | https://zapier.com/apps/follow-up-boss/integrations |
| CallAction DNC Verify | https://callaction.co/verify |
| CRTC DNCL Rules | https://www.crtc.gc.ca/eng/phone/telemarketing.htm |
| National DNC Registry (US) | https://www.donotcall.gov |

---

<p align="center">
  <sub>AI Real Estate Calling System · No-Code MVP · Version 2 · April 2026</sub>
</p>
