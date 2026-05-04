# Lumis

> Lumis is a unified clinical intake and patient-communication platform that collapses booking, voice handling, SMS, and care coordination into a single system — operable in three switchable modes (fully autonomous AI, AI with human review, or human-led with AI context).

**Status**

- In production at Evergreen Wellness Clinic (Bay Area), since April 2026
- Built solo in approximately 3 weeks, using Claude Code as the implementation engine
- I drove all product requirements, system design, integrations, and operations
- This repo is an architectural overview only; the source code lives in a separate private repo

## What problem this solves

A typical small clinic operates a fragmented stack: a separate booking portal hosted by one vendor, a phone system (often with a manual answering service after hours), an EHR, ad-hoc SMS sent from staff personal phones, fax for medical records, and voicemail that nobody listens to until the next day. Each tool has its own login. None of them know about the others.

The friction surfaces every time the phone rings. Front-desk staff answers without knowing who is calling, has to ask the patient to identify themselves, looks them up in the EHR, switches to the booking portal to check today's openings, switches back to the EHR to log the visit, then maybe sends a confirmation text from a personal phone. This single interaction touches four systems and takes 4–8 minutes when it should take 60 seconds.

Lumis collapses these into one system. The clinic answers the same phone call, but the staff dashboard already shows the caller's identity, history, upcoming appointments, and one-click actions before the call is even picked up. After hours, an AI voice agent answers the same line and books appointments autonomously. The clinic chooses which mode is active for each hour of the operating day.

## The three operating modes

**Mode A — Fully autonomous AI voice booking.** Patient calls the clinic. Twilio webhook routes the call to a VAPI-orchestrated voice agent named Eva (ElevenLabs voice). Eva answers questions, checks live availability via Lumis APIs, walks the patient through the booking flow, collects details, and confirms. Lumis writes the appointment to the database and fires the SMS confirmation. No human staff in the loop. Used after-hours and on weekends.

**Mode B — AI-collected intake with staff review queue.** The voice agent collects all the same details, but instead of writing the appointment directly, it places the request in an Intake Queue inside Lumis. Clinic staff review and confirm the request during business hours. On confirmation, the SMS confirmation fires automatically.

**Mode C — Staff-assisted with AI context.** The call routes to the staff phone normally, but Lumis renders a Live Call Card on the staff dashboard with the patient's name, phone, past and upcoming appointments, and one-click action buttons (book new, reschedule, cancel, view chart). Staff answer the phone already knowing who they are talking to. The booking modal is two clicks instead of switching between EHR and a separate portal.

The clinic chooses which mode is active for each hour of operation. A typical week might run Mode C during business hours, Mode B for evening overflow, and Mode A overnight.

# For engineers — architecture and implementation

## High-level architecture

```
Patient (call/SMS)
        │
        ▼
   ┌────────┐
   │ Twilio │ ──────────► Lumis Webhook Layer
   └────────┘             (routes by mode + hour)
                                  │
        ┌─────────────────────────┼─────────────────────┐
        ▼                         ▼                     ▼
   VAPI agent              Live Call Card        Lumis Portal UI
   (Mode A & B)            (Mode C — staff       (staff dashboard,
        │                  dashboard overlay)    appointments,
        │                                        intake queue,
        ▼                                        clients, messages)
   ElevenLabs                       │
   (TTS voice)                      │
        │                           │
        └───────────┬───────────────┘
                    ▼
              ┌────────────┐
              │  Supabase  │ ── Resend ──► email confirmations
              │ Postgres + │
              │ Auth + RLS │
              └────────────┘
```

The webhook layer is the dispatch point. It reads the clinic's per-hour mode configuration and either redirects the call into the VAPI agent (modes A and B), forwards to staff phone with a Live Call Card overlay (mode C), or falls back to a configurable IVR menu after-hours when no mode is set.

## Tech stack and rationale

| Component | Choice | Why |
|---|---|---|
| Frontend / Portal | Next.js, TypeScript | Server-side rendering for the staff dashboard; same framework available for the marketing surface |
| Backend / API | Next.js API routes | Single deployment surface; no separate backend service needed at this scale |
| Database + auth | Supabase (Postgres + Auth + Row-Level Security) | Postgres-grade data model; RLS handles per-clinic data isolation natively |
| Telephony | Twilio (voice + SMS) | Industry-standard webhooks; reliable; HIPAA-eligible tier available |
| AI voice orchestration | VAPI | Manages the LLM-driven dialog turn-taking and function calling for booking |
| Voice synthesis | ElevenLabs | Best naturalness for the voice agent persona |
| Transactional email | Resend | Clean DX, modern API, dev-friendly testing |
| Hosting | Vercel | Native Next.js deployment, edge functions, fast iteration |
| Marketing site | Wix | Outside the application surface; clinic owner can edit independently |

## Engineering decisions worth noting

**Mode-switching as configuration, not code branches.** The three operating modes are driven by per-clinic, per-hour configuration in the database. The webhook layer reads the active mode for the current time and routes accordingly. New clinics can be onboarded without code changes; the clinic can change its operating posture (e.g., "tonight we want Mode A from 6 PM to 8 AM") through the portal. This means the system grows new tenants and new operating policies without engineering work.

**Function calling for the voice agent.** Rather than having the voice agent generate prose responses about availability, the VAPI agent calls Lumis API functions (`get_available_slots`, `create_appointment`, `lookup_patient`) and the actual data flows through the agent into the conversation. This gives deterministic data behavior with LLM-managed natural conversation — the agent never hallucinates a Tuesday 3 PM slot that doesn't exist, and every booking is the result of a real database write the system can audit.

**Live Call Card as the killer feature for Mode C.** The hardest UX problem in clinic phone work is "I don't know who's calling until I ask." Lumis solves this by intercepting the inbound call ringing on the staff phone, rendering the caller's full context on the staff dashboard before they answer. By the time the receptionist picks up, the dashboard already shows: patient name, last visit, upcoming appointments, one-click book/reschedule/cancel. Staff stop context-switching between EHR, portal, and notes — the call is handled in one place.

**Audit trail as a first-class table.** Every action in the portal — call answered, appointment created, appointment cancelled, message sent, configuration changed — writes to a security audit trail with actor, timestamp, and before/after state. Healthcare compliance requires this; building it in from day one was easier than retrofitting later, and it has proved invaluable during operations for "what happened on Thursday at 3 PM" questions.

## Built with AI as the implementation engine

The full system was built solo in approximately three weeks. I drove all product requirements, system design, integration architecture, and operational decisions. Claude Code served as the implementation engine — the actual code was largely AI-generated under my direction, with me reviewing, integrating, debugging, and operating the result.

To me, the more interesting story than the system itself: a single product owner with the right architectural taste and the willingness to operate AI tooling effectively can ship production software at a scale that previously required a small team. Lumis is one example. I'd expect a lot more software to look like this in the next few years.

## Contact

[Sankar Neelakandan](https://www.linkedin.com/in/sankarpn/) — sankarpn@gmail.com

For a live demo of the staff portal or to see relevant source excerpts, reach out by email.
