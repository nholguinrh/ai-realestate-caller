# 🏠 AI Real Estate Calling System

> An AI-powered outbound calling platform for real estate lead engagement — voice agents, CRM integration, and campaign automation on AWS EC2 (Fedora 41 + Podman).

---

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Tech Stack](#tech-stack)
- [Project Structure](#project-structure)
- [Getting Started](#getting-started)
- [Configuration](#configuration)
- [Services](#services)
- [CRM Integration](#crm-integration)
- [Voice Stack](#voice-stack)
- [Compliance](#compliance)
- [Development Workflow](#development-workflow)
- [Phase Roadmap](#phase-roadmap)
- [Contributing](#contributing)
- [License](#license)

---

## Overview

This system automates outbound real estate calls using AI voice agents. It pulls leads from a CRM, places calls via Twilio or VAPI, synthesizes natural speech with ElevenLabs, and logs all outcomes back to the CRM. The platform is built for reliability, compliance, and scale.

**Key capabilities:**

- Pulls leads from CRM (HubSpot / GoHighLevel / Zoho / Pipedrive)
- Places outbound calls via Twilio or VAPI
- AI voice agent conducts property pitch conversations
- Handles objections, voicemail, wrong numbers, call-backs
- Logs disposition codes and outcomes to CRM
- CRTC-compliant scheduling (Canada) with TCPA awareness (US)
- Time-zone-aware campaign scheduler with DNC list enforcement

---

## Architecture

```
┌─────────────────────────────────────────────────────┐
│                   EC2 (Fedora 41)                   │
│                                                     │
│  ┌─────────────┐    ┌─────────────┐                 │
│  │  API        │    │  Worker     │                 │
│  │  (FastAPI)  │◄──►│  (Queue     │                 │
│  │             │    │   Runner)   │                 │
│  └──────┬──────┘    └──────┬──────┘                 │
│         │                  │                        │
│  ┌──────▼──────────────────▼───────┐                │
│  │          Redis (Queue + State)   │                │
│  └─────────────────────────────────┘                │
└────────────────────┬────────────────────────────────┘
                     │
        ┌────────────▼────────────┐
        │     Twilio  /  VAPI     │  ← Telephony + Call Control
        └────────────┬────────────┘
                     │
        ┌────────────▼────────────┐     ┌──────────────────┐
        │      ElevenLabs         │     │  Whisper/Deepgram │
        │      (TTS Voice)        │     │  (STT — optional) │
        └─────────────────────────┘     └──────────────────┘
                     │
        ┌────────────▼────────────┐
        │         CRM API         │  ← Leads in, outcomes out
        │  (HubSpot / GHL / Zoho) │
        └─────────────────────────┘
```

### Data Flow

```
CRM ──► Lead Queue (Redis) ──► Worker ──► Twilio/VAPI ──► Phone Call
                                  │
                            ElevenLabs TTS
                                  │
                         Call Outcome Logged ──► CRM Update
```

---

## Tech Stack

| Layer | Technology | Notes |
|---|---|---|
| API / Orchestration | Python 3.12 + FastAPI | Async, type-hinted |
| Queue | Redis 7 | Job queue + call state |
| Telephony | Twilio or VAPI | See [Voice Stack](#voice-stack) |
| TTS | ElevenLabs | Natural voice synthesis |
| STT | Whisper / Deepgram | Only if not using VAPI |
| CRM | HubSpot / GoHighLevel / Zoho | Adapter pattern — swappable |
| Infrastructure | AWS EC2 — Fedora 41 | Podman containers |
| Container Runtime | Podman + podman-compose | No Docker daemon |
| CI/CD | GitHub Actions | Phase 2 |

> ⚠️ **Podman only** — never use `docker` or `docker-compose` commands in this project.

---

## Project Structure

```
/
├── api/                          # FastAPI application
│   ├── main.py                   # App entrypoint
│   ├── routes/
│   │   ├── webhooks.py           # Twilio / VAPI webhook handlers
│   │   ├── campaigns.py          # Campaign trigger endpoints
│   │   └── health.py             # Health check
│   ├── services/
│   │   ├── crm/
│   │   │   ├── base.py           # CRMAdapter abstract class + Lead dataclass
│   │   │   ├── hubspot.py        # HubSpot implementation
│   │   │   ├── gohighlevel.py    # GoHighLevel implementation
│   │   │   └── zoho.py           # Zoho implementation
│   │   ├── voice/
│   │   │   ├── vapi.py           # VAPI call client
│   │   │   ├── twilio_client.py  # Twilio call client
│   │   │   └── elevenlabs.py     # ElevenLabs TTS client
│   │   └── scheduler/
│   │       ├── campaign.py       # Campaign scheduler (CRTC-compliant)
│   │       └── dnc.py            # DNC list management
│   └── models/
│       ├── lead.py
│       ├── call.py
│       └── campaign.py
├── worker/                       # Queue consumer
│   ├── main.py
│   ├── tasks/
│   │   ├── place_call.py         # Outbound call task
│   │   └── log_outcome.py        # CRM update task
│   └── Containerfile
├── podman/                       # Container configuration
│   ├── podman-compose.yml
│   └── configs/
│       └── redis.conf
├── scripts/                      # Utility scripts
│   ├── setup_ec2.sh              # EC2 + Podman bootstrap
│   ├── seed_leads.py             # Test lead seeder
│   └── check_dnc.py              # DNC list checker
├── docs/                         # Documentation
│   ├── architecture.md
│   ├── crm-adapters.md
│   ├── voice-stack.md
│   ├── compliance.md
│   └── api-reference.md
├── tests/
│   ├── unit/
│   └── integration/
├── .github/
│   └── workflows/
│       └── ci.yml                # GitHub Actions (Phase 2)
├── .env.example                  # Environment variable template
├── .gitignore
├── podman-compose.yml            # Root-level shortcut
└── README.md
```

---

## Getting Started

### Prerequisites

- Fedora 41 with Podman installed (`sudo dnf install podman podman-compose`)
- Python 3.12+ (for local dev without containers)
- Git
- Accounts: Twilio or VAPI, ElevenLabs, your chosen CRM

### 1. Clone the repository

```bash
git clone git@github.com:<your-org>/ai-realestate-caller.git
cd ai-realestate-caller
```

### 2. Set up environment variables

```bash
cp .env.example .env
# Edit .env with your API keys and config
```

### 3. Start services with Podman

```bash
podman-compose up -d
```

### 4. Verify services are running

```bash
podman-compose ps
curl http://localhost:8000/health
```

---

## Configuration

Copy `.env.example` to `.env` and fill in all values. **Never commit `.env` to version control.**

```env
# ── CRM ──────────────────────────────────────────
CRM_PROVIDER=hubspot                  # hubspot | gohighlevel | zoho | pipedrive
HUBSPOT_API_KEY=your_key_here
# GOHIGHLEVEL_API_KEY=your_key_here
# ZOHO_CLIENT_ID=...
# ZOHO_CLIENT_SECRET=...

# ── Voice Stack ───────────────────────────────────
VOICE_PROVIDER=vapi                   # vapi | twilio
VAPI_API_KEY=your_key_here
VAPI_PHONE_NUMBER_ID=your_phone_id
# TWILIO_ACCOUNT_SID=...
# TWILIO_AUTH_TOKEN=...
# TWILIO_FROM_NUMBER=+1...

# ── ElevenLabs ────────────────────────────────────
ELEVENLABS_API_KEY=your_key_here
ELEVENLABS_VOICE_ID=your_voice_id

# ── Redis ─────────────────────────────────────────
REDIS_URL=redis://redis:6379/0

# ── App ───────────────────────────────────────────
APP_ENV=development                   # development | production
WEBHOOK_BASE_URL=https://your-domain.com
LOG_LEVEL=INFO

# ── Compliance ────────────────────────────────────
JURISDICTION=CA                       # CA (CRTC) | US (TCPA) | BOTH
CALLING_WINDOW_START=09:00
CALLING_WINDOW_END=21:00
```

---

## Services

### API (`api/`)

FastAPI application that exposes:

| Endpoint | Method | Description |
|---|---|---|
| `/health` | GET | Health check |
| `/webhooks/vapi` | POST | VAPI call event handler |
| `/webhooks/twilio/status` | POST | Twilio call status callback |
| `/webhooks/twilio/twiml` | POST | Twilio TwiML call flow |
| `/campaigns/{id}/trigger` | POST | Manually trigger a campaign |
| `/campaigns/{id}/status` | GET | Campaign status + stats |

### Worker (`worker/`)

Consumes jobs from Redis queue:

- `place_call` — validates lead, checks DNC, fires outbound call
- `log_outcome` — receives call result, updates CRM, updates queue state

### Redis

Used for:
- Job queue (call tasks)
- Call state tracking (in-progress, retries)
- DNC list cache

---

## CRM Integration

All CRM connectors implement the `CRMAdapter` interface:

```python
from abc import ABC, abstractmethod
from dataclasses import dataclass
from typing import Optional

@dataclass
class Lead:
    id: str
    name: str
    phone: str
    email: Optional[str]
    property_interest: Optional[str]
    metadata: dict

class CRMAdapter(ABC):

    @abstractmethod
    async def get_leads(self, campaign_id: str) -> list[Lead]:
        """Fetch leads for a campaign."""
        ...

    @abstractmethod
    async def update_call_status(self, lead_id: str, outcome: dict) -> bool:
        """Log call outcome back to the CRM."""
        ...
```

**Available adapters:**

| Adapter | File | Status |
|---|---|---|
| HubSpot | `api/services/crm/hubspot.py` | Planned |
| GoHighLevel | `api/services/crm/gohighlevel.py` | Planned |
| Zoho | `api/services/crm/zoho.py` | Planned |
| Pipedrive | `api/services/crm/pipedrive.py` | Planned |

Switch CRM by changing `CRM_PROVIDER` in `.env` — no code changes required.

---

## Voice Stack

Two supported configurations:

### Option A — VAPI (Recommended for MVP)

VAPI is an all-in-one voice agent platform that handles STT, LLM, TTS, and call control. Lower operational overhead.

```
Outbound call ──► VAPI ──► ElevenLabs voice ──► Conversation ──► Webhook ──► API
```

**Best for:** faster time to market, managed infrastructure.

### Option B — Twilio + ElevenLabs

Twilio manages telephony and TwiML call flow. ElevenLabs generates audio per prompt turn. Optional: Whisper or Deepgram for STT.

```
Outbound call ──► Twilio ──► TwiML flow ──► ElevenLabs TTS ──► Whisper STT ──► Logic ──► API
```

**Best for:** cost control at scale, full ownership of call flow logic.

Switch voice stack by changing `VOICE_PROVIDER` in `.env`.

### Agent Script Design

The AI agent follows this conversation flow:

```
[Intro — <10 seconds]
  → Identify self + company
  → Name the property and price

[Pitch]
  → Key property highlights
  → Single CTA: "Would you like to schedule a viewing?"

[Objection Handling]
  → Not interested → thank and end
  → Call back later → log preferred time
  → Wrong number → mark in CRM
  → Voicemail → leave brief message, log

[Outcome Logging]
  → Disposition code
  → Duration
  → Transcript (if available)
  → CRM update
```

---

## Compliance

> ⚠️ This system places automated outbound calls. Compliance is mandatory.

### Canada — CRTC Rules

- [ ] Call only between **9:00 AM – 9:00 PM** local time
- [ ] Identify the **business name** at the start of every call
- [ ] Provide an **opt-out mechanism** (e.g., "Press 9 to be removed")
- [ ] Maintain an **internal DNC list**, honored within 14 days
- [ ] Do not call numbers on the **National DNCL**
- [ ] Retain call records for a minimum of **24 months**

### United States — TCPA

- **Written consent** is required before auto-dialing cell phones
- Calling hours vary by state — default to 8:00 AM – 9:00 PM local time
- Honor do-not-call requests within 30 days
- Consult legal counsel before launching US campaigns

### Implementation

The `scheduler/campaign.py` service enforces time windows. The `scheduler/dnc.py` service manages the internal DNC list. Both run before any call is placed.

---

## Development Workflow

### Branch Strategy

```
main        ← stable, production-ready
develop     ← integration branch
feature/*   ← new features (branch from develop)
fix/*       ← bug fixes (branch from develop)
```

### Starting a new feature

```bash
git checkout develop
git pull origin develop
git checkout -b feature/your-feature-name
```

### Commit convention

```
feat: add GoHighLevel CRM adapter
fix: handle CRM pagination edge case
chore: update podman-compose Redis version
docs: add compliance section to README
```

### Running tests

```bash
# Unit tests
podman-compose run api python -m pytest tests/unit/

# Integration tests (requires .env with real credentials)
podman-compose run api python -m pytest tests/integration/
```

### Local development with webhooks

For Twilio/VAPI webhooks in local development, use a tunnel:

```bash
# Using ngrok
ngrok http 8000

# Update WEBHOOK_BASE_URL in .env with the ngrok URL
```

---

## Phase Roadmap

| Phase | Focus | Status |
|---|---|---|
| **Phase 0** | Tech stack decisions, GitHub repo, EC2 baseline | 🔄 In Progress |
| **Phase 1** | Podman Compose, CRM connector, webhook endpoints, manual call trigger | ⏳ Planned |
| **Phase 2** | AI voice agent, call flow, ElevenLabs voice, outcome logging | ⏳ Planned |
| **Phase 3** | Campaign scheduler, retry logic, lead queue, dashboard | ⏳ Planned |
| **Phase 4** | Compliance layer, DNC integration, recording storage, analytics | ⏳ Planned |

---

## Contributing

1. Branch from `develop`: `git checkout -b feature/<name>`
2. Follow commit conventions above
3. Ensure all secrets are in `.env` — never in code
4. Run tests before opening a PR
5. Open PR to `develop` — not `main`
6. Tag a reviewer

---

## Reference Links

| Resource | URL |
|---|---|
| VAPI Docs | https://docs.vapi.ai |
| Twilio Voice (Python) | https://www.twilio.com/docs/voice/quickstart/python |
| ElevenLabs API | https://elevenlabs.io/docs/api-reference/getting-started |
| Podman Compose | https://github.com/containers/podman-compose |
| Fedora 41 + Podman | https://docs.fedoraproject.org/en-US/fedora/latest/system-administrators-guide/virtualization/Using_Podman/ |
| CRTC DNCL Rules | https://www.crtc.gc.ca/eng/phone/telemarketing.htm |
| HubSpot CRM API | https://developers.hubspot.com/docs/api/crm/contacts |
| GoHighLevel API | https://highlevel.stoplight.io/docs/integrations |

---

## License

Private — All rights reserved. Not for public distribution.
