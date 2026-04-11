# AI Calling Platform Decision: Synthflow → Retell AI

**Change:** Migrating from Synthflow to Retell AI

---

## Summary

After evaluating four AI calling platforms against our real estate outbound use case — and specifically accounting for how real estate leads behave on calls — we are migrating from Synthflow to Retell AI. The primary driver is Synthflow's documented failure to handle interruptions and off-script moments, which are extremely common in real estate lead calls.

---

## Platforms Evaluated

| Platform | Developer skill required | Pricing | FUB integration |
|---|---|---|---|
| Synthflow | None (true no-code) | ~$0.08/min, all-in | Via Zapier |
| Retell AI | Low (one-time setup) | $0.07–$0.12/min PAYG | Via Zapier / API |
| Bland AI | High (developer-first) | $0.09/min + hidden fees | API only |
| Vapi | High (developer-first) | $0.05/min + $0.05–0.20 add-ons | API only |

---

## Scoring: Full Criteria

Scores are out of 10. The **interruption handling** and **edge case recovery** categories were added after initial evaluation revealed a critical gap for real estate use.

| Criterion | Synthflow | Retell AI | Bland AI | Vapi |
|---|---|---|---|---|
| No-code ease | 9.5 | 5.2 | 3.0 | 2.0 |
| CRM integration | 8.5 | 6.5 | 5.0 | 4.5 |
| Voice quality | 9.0 | 8.7 | 8.0 | 7.0 |
| Latency | 9.2 | 8.2 | 7.5 | 6.2 |
| **Interruption handling** | **3.5** | **9.0** | 7.0 | 6.8 |
| **Edge case recovery** | **4.0** | **8.5** | 6.5 | 5.5 |
| **Weighted total** | **7.3** | **7.7** | **6.2** | **5.3** |

> Interruption handling and edge case recovery are weighted 2× given real estate lead behaviour.

---

## Why Interruptions Matter in Real Estate

Real estate leads are among the most interrupt-heavy callers in any outbound use case:

- They answer calls while driving, distracted, or mid-task
- They frequently cut in with "wait, is this a real person?"
- They redirect mid-sentence: "actually, I'm looking to sell, not buy"
- They ask off-script questions about pricing, neighbourhoods, or timelines
- They challenge the agent: "how did you get my number?"

An AI agent that cannot handle these moments gracefully will lose the lead — or worse, damage trust with the team's brand before a human agent ever gets involved.

---

## Why Synthflow Was Deprioritised

Synthflow performed well in initial testing across voice quality, ease of setup, and CRM integration. It remains the easiest platform to deploy without developer involvement.

However, production testing and user reports identified a critical weakness for outbound real estate calls:

- **Inconsistent barge-in handling:** Interruptions are not consistently processed, leading to overlapping speech and reduced naturalness during live caller interactions.
- **Off-script failure mode:** When tested, Synthflow agents quickly lost track when someone went off-script and defaulted back to a canned line — described by reviewers as feeling "more like a fancy IVR than an agent."
- **Inability to interrupt mid-sentence:** Longer pauses and the inability to interrupt the voice agent mid-sentence make it clear to leads they are speaking to a bot.

These are not fringe failure cases for real estate — they are the normal experience of calling a lead who did not book their own appointment.

---

## Why Retell AI Was Selected

Retell AI has made turn-taking and barge-in handling a core infrastructure investment:

- **Advanced turn-taking model:** Retell's model enhancements significantly reduce false interruptions while maintaining responsive barge-in capabilities, better distinguishing between natural speech pauses and actual conversation turns.
- **Responds mid-sentence:** Retell supports interruption handling, allowing agents to respond mid-sentence, keeping conversations feeling natural.
- **LLM-native conversation:** Rather than strict decision trees, Retell uses LLM-driven conversation logic, which recovers more gracefully from unexpected input.
- **Tested for outbound:** Retell's outbound mode was rated stronger in head-to-head testing, especially with branded caller ID and verified numbers that improve answer rates.

---

## Trade-offs Accepted

| Trade-off | Impact | Mitigation |
|---|---|---|
| Requires one-time developer setup | Medium — not self-service | Hire Upwork freelancer ($150–300) to wire Retell → Zapier → FUB |
| Pay-as-you-go pricing can vary | Low — manageable at our volume | Monitor monthly; costs only vary with real call volume |
| No native FUB integration | Low — same as Synthflow | Zapier bridge works identically |
| Less intuitive UI for non-technical users | Low — setup is one-time | Ongoing script edits can be done in the Retell dashboard |

---

## Recommended Architecture: Retell AI + Follow Up Boss via Zapier

```
New lead enters FUB
        │
        ▼
FUB triggers Zapier (on new contact or tag "AI Call")
        │
        ▼
Zapier sends lead data to Retell AI
(phone, name, property interest, lead source)
        │
        ▼
Retell AI agent makes outbound call
(qualify: timeline, budget, location, financing)
        │
        ▼
Call summary + outcome sent back via Zapier webhook
        │
        ▼
Zapier logs note in FUB + updates stage + adds tag
("Hot", "Not Ready", "Booked", "Wrong Number")
        │
        ▼
Hot leads trigger FUB action plan or agent alert
```

---

## Bland AI and Vapi: Why They Were Ruled Out

Both platforms require significant developer resources to deploy and maintain — making them unsuitable for a team without in-house engineering.

**Bland AI** is enterprise-grade and can handle extremely high call volumes, but setting up custom CRM integration and call logic requires developer time, and the pricing structure has hidden fees for voice cloning, transcription, and advanced features that add up quickly.

**Vapi** is a developer-first infrastructure tool. The base price of $0.05/min is misleading — once STT, TTS, LLM, and telephony are added, costs can jump 3–6×. It also requires engineering effort to configure even basic call flows, and does not offer the conversational recovery quality needed for real estate.

---

## Compliance Note

AI outbound calling requires prior express written consent under TCPA (US) and CASL (Canada). Before deploying any outbound AI calling campaign:

- Ensure lead capture forms include consent language for automated/AI contact
- Scrub against federal and provincial/state do-not-call lists
- Include a clear disclosure that the caller may be an AI agent
- Provide an easy opt-out path ("press 2 or say stop to be removed")

---

## Next Steps

1. Engage Upwork freelancer for Retell + Zapier + FUB integration setup
2. Draft Retell call script for real estate lead qualification (handling interruptions, objections, off-script moments)
3. Run 50-call pilot before full rollout
4. Monitor call transcripts for edge cases in first two weeks and refine prompts
5. Set up FUB tag logic for call outcomes (Hot / Warm / Not Ready / DNC)
