<p align="center">
  <img src="screenshots/banner.png" alt="CallGenius Banner" width="100%"/>
</p>

<h1 align="center">📞 CallGenius</h1>
<p align="center">
  <b>AI-Powered Telecaller Training Platform for Indian SMBs</b><br/>
  Train your sales team without a human trainer — voice-first, scenario-driven, bilingual
</p>

<p align="center">
  <img src="https://img.shields.io/badge/Platform-Electron%20Desktop-blue?style=flat&logo=electron"/>
  <img src="https://img.shields.io/badge/Frontend-React%20%2B%20TypeScript-cyan?style=flat&logo=react"/>
  <img src="https://img.shields.io/badge/STT-AssemblyAI%20%7C%20Web%20Speech-orange?style=flat"/>
  <img src="https://img.shields.io/badge/TTS-Google%20Cloud%20%7C%20Browser-green?style=flat"/>
  <img src="https://img.shields.io/badge/AI-Groq%20%7C%20Gemini%20%7C%20OpenRouter-purple?style=flat"/>
  <img src="https://img.shields.io/badge/Language-English%20%2B%20Tamil-red?style=flat"/>
  <img src="https://img.shields.io/badge/Status-Stage%201%20Complete-brightgreen?style=flat"/>
</p>

> ⚠️ **This is a commercial product.** Source code is proprietary and not included in this repository. This page serves as a technical showcase.

---

## 🧩 The Problem

Small businesses in India — insurance agencies, real estate firms, BPOs, loan DSAs — rely heavily on telecallers. But attrition is brutal. A new hire joins, a senior employee spends 2–3 weeks training them on real calls, and then they quit. The cycle repeats endlessly.

The cost is not just salary. It is the senior employee's time — spent training someone who may not stay — and the revenue lost from poorly handled calls in the meantime.

**CallGenius lets a telecaller practice real call scenarios alone, as many times as needed, with an AI that responds like a real customer — in voice, in real time.**

---

## 🎯 What It Does

CallGenius simulates the customer side of a sales or support call. The trainee speaks into their microphone. The AI listens, understands context, and responds in voice — handling objections, asking questions, and reacting exactly as a real customer would, based on a scripted persona.

Managers generate training scenarios from their own real call transcripts. The AI learns from those transcripts and creates reusable persona banks — so every trainee practices against scenarios that actually happen in that business.

### Core Capabilities

| Feature | Description |
|---|---|
| **Live Trainer** | Real-time voice conversation with AI customer persona |
| **AI Coach** | Post-call evaluation — score, strengths, specific improvement suggestions grounded in the real script |
| **Persona Bank Engine** | Auto-generate training scenarios from real call transcripts using Gemini |
| **Scenario Types** | Open-ended (insurance, real estate) + scripted linear (semiconductor sourcing, Raise ON) |
| **Auditor** | Upload recorded real calls for AI-powered diarized analysis |
| **Call List CRM** | Assign customers to telecallers, tag call outcomes, auto-trigger calendar reminders |
| **Reports** | Session scores, tag breakdown, calls-per-telecaller — clean and simple |
| **Bilingual** | English + Tamil — TTS switches automatically based on language |

---

## 🏗️ Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                     Electron Shell                           │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐  │
│  │              React + TypeScript Frontend               │  │
│  │                                                        │  │
│  │  Trainer · Auditor · Persona Bank · Call List · Reports│  │
│  └────────────────────────┬───────────────────────────────┘  │
└───────────────────────────│──────────────────────────────────┘
                            │ HTTP / WebSocket
┌───────────────────────────▼──────────────────────────────────┐
│                    FastAPI Backend                            │
│                                                              │
│  ┌──────────────┐  ┌─────────────┐  ┌────────────────────┐  │
│  │  Persona     │  │  STT Layer  │  │   TTS Layer        │  │
│  │  Bank Engine │  │ AssemblyAI  │  │ Google Cloud TTS   │  │
│  │  (pools.py + │  │ (streaming) │  │ (Tamil: Chirp3-HD) │  │
│  │  matcher.py) │  │ Web Speech  │  │ Browser SpeechSynth│  │
│  └──────────────┘  │ (English/   │  │ (English)          │  │
│                    │  fallback)  │  └────────────────────┘  │
│  ┌──────────────┐  └─────────────┘                          │
│  │  AI Cascade  │  ┌─────────────┐  ┌────────────────────┐  │
│  │  Groq →      │  │  Scenario   │  │  Google Workspace  │  │
│  │  OpenRouter →│  │  Extractor  │  │  Calendar + Gmail  │  │
│  │  Gemini →    │  │  (Gemini)   │  │  (tag automation)  │  │
│  │  Ollama      │  └─────────────┘  └────────────────────┘  │
│  └──────────────┘                                           │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │              SQLite Database (via SQLAlchemy)         │   │
│  │  Sessions · Evaluations · Personas · CallList ·       │   │
│  │  Telecallers · CallTagEvents · GoogleTokens           │   │
│  └──────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────┘
                            │
┌───────────────────────────▼──────────────────────────────────┐
│                  Firebase (Licensing)                         │
│   OTP Auth · Machine Fingerprint (SHA-256) · Per-seat limits │
└──────────────────────────────────────────────────────────────┘
```

---

## 🔬 Key Engineering Decisions & Challenges

### 1. Dual STT Architecture — AssemblyAI + Web Speech API
Running a single STT provider for all languages failed in practice. AssemblyAI's real-time streaming (`whisper-rt`) was tested for Tamil but proved unreliable — undocumented behavior on silence handling and hallucination artifacts (a known Whisper artifact: repeating "thank you for watching" on silence). **Solution:** Tamil and Tanglish are forced to Chrome's Web Speech API. English uses AssemblyAI streaming with a custom 4.5-second debounce accumulator — because AssemblyAI's own `end_of_turn` timing parameters did not behave as documented.

**Lesson learned:** Never trust third-party real-time API documented parameters at face value. Verify with actual simulation before and after shipping.

### 2. AI Cascade with Hard Exclusions
Four AI providers are wired in cascade: Groq → OpenRouter → Gemini → Ollama. But not all providers are equal for all tasks:
- **Groq** is reserved for real-time conversation (low latency). Hard-excluded from evaluation/coaching — quota must be preserved.
- **Gemini** `gemini-3.5-flash-lite` is the soft preference for conversation. `gemini-2.0-flash` is hard-excluded — zero quota.
- **Evaluation tasks** use a 120s timeout (raised from 60s after repeatedly hitting the limit on large structured report generation).
- **Ollama** runs locally on a 4GB laptop as the final fallback — no internet required.

### 3. Persona Bank — From Real Transcripts to Live Training Scenarios
The most complex feature: managers paste 10–50 real call transcripts, Gemini extracts representative scenarios (objections, questions, responses, key phrases) in a structured JSON format, and these become live training personas — without any code changes to the existing conversation engine.

The extraction prompt took significant iteration to fix a specific failure mode: the model consistently dropped the final closing/sign-off exchange, treating it as low-value filler. Explicit instruction was added requiring full conversation capture including confirmations and sign-offs.

### 4. TTS Echo Problem — Mic Starting Before Playback Ends
A critical bug: the microphone started listening immediately after *triggering* TTS playback — not after it *finished*. Without earphones, the speaker audio leaked into the mic and was transcribed as trainee speech, creating an AI-talks-to-itself loop. Fixed by wiring `onStart`/`onEnd` callbacks to all TTS call sites, with the microphone only activating via the `onEnd` callback.

### 5. Cross-Client Evaluation Contamination
AI Coach was generating "Better Response" suggestions using the wrong company's system prompt. A Raise ON (semiconductor sourcing) call was being evaluated with MasterBrains (real estate) context — producing suggestions with the wrong company name and wrong script content. Fixed independently in both evaluation code paths (live post-call and the audit queue path), using the persona's own scenario script as primary evaluation context.

### 6. Google Calendar 403 — Token vs. API Enablement
When Calendar event creation returned 403, the first assumption was a stale/cached OAuth token. A full disconnect-and-reconnect cycle confirmed the token was fine. The actual cause: the Google Calendar API was simply not enabled on the project. Fixed by capturing Google's actual error response body (instead of discarding it via `raise_for_status()`) — which immediately surfaced the real reason.

### 7. Machine Fingerprint Licensing
Preventing license sharing required a SHA-256 hash of hardware identifiers sent as a heartbeat to Firebase Cloud Functions. Licenses are device-locked. Per-seat limits enforced per Firebase account.

---

## 📱 Screenshots

| Live Trainer | AI Coach Evaluation | Persona Bank | Call List CRM |
|---|---|---|---|
| ![Trainer](screenshots/trainer.png) | ![Coach](screenshots/coach.png) | ![PersonaBank](screenshots/persona_bank.png) | ![CallList](screenshots/call_list.png) |

> 📹 **Demo Video:** [Watch on YouTube](https://youtube.com/your-link-here)

---

## 🛠️ Tech Stack

| Layer | Technology |
|---|---|
| Desktop Shell | Electron |
| Frontend | React + TypeScript + Vite |
| Backend | FastAPI (Python) + SQLAlchemy + SQLite |
| STT (English) | AssemblyAI Streaming (WebSocket, PCM16) |
| STT (Tamil) | Web Speech API (Chrome built-in) |
| TTS (Tamil) | Google Cloud TTS — Chirp3-HD (ta-IN-Aoede) |
| TTS (English) | Browser SpeechSynthesis API |
| AI — Conversation | Groq (llama-3.1-8b-instant) → OpenRouter → Gemini → Ollama |
| AI — Evaluation | Gemini (gemini-3.5-flash-lite) + OpenRouter fallback |
| Scenario Extraction | Gemini (batch, 180s timeout) |
| Licensing | Firebase OTP + Machine Fingerprint (SHA-256) |
| Calendar / Email | Google Workspace REST API (OAuth2) |

---

## 🎯 Target Markets

| Segment | Use Case |
|---|---|
| Insurance agencies | Policy pitch + objection handling training |
| Real estate firms | Site visit call + follow-up call training |
| Semiconductor sourcing (Raise ON) | Linear scripted BOM inquiry + negotiation scenarios |
| Loan DSAs | Lead qualification + rejection handling |
| BPOs | Onboarding new agents without senior trainer involvement |

---

## 🗺️ Roadmap

- [x] Live voice trainer — real-time AI customer conversation
- [x] AI Coach — post-call evaluation grounded in real script content
- [x] Persona Bank Engine — auto-generate scenarios from real transcripts
- [x] Auditor — batch upload + diarized analysis of real calls
- [x] Raise ON scenario engine — deterministic scripted linear flows
- [x] AssemblyAI streaming STT — English real-time with debounce accumulator
- [x] Google Cloud TTS — Tamil Chirp3-HD with transliteration display
- [x] Call List CRM — per-telecaller assignment, tagging, calendar automation
- [x] Multi-telecaller management — roles, rename, historical log integrity
- [x] Bulk customer import — Excel paste + comma-separated text
- [x] Firebase licensing — OTP + machine fingerprint + per-seat limits
- [x] Reports dashboard — session scores, tag breakdown, telecaller stats
- [ ] SIP/VoIP bridge — train on a real phone call
- [ ] Live call copilot — real-time suggestion during actual customer calls
- [ ] CRM sync — Zoho / Salesforce integration

---

## 👤 Author

**Arun Stanely Prakash G** — Founder, SISAI Edutech
[![LinkedIn](https://img.shields.io/badge/LinkedIn-arunstanely-blue?style=flat&logo=linkedin)](https://linkedin.com/in/arunstanely)
[![Website](https://img.shields.io/badge/Website-sisai--edutech.com-green?style=flat&logo=googlechrome)](https://sisai-edutech.com/)
[![Email](https://img.shields.io/badge/Email-arun__mem@yahoo.com-red?style=flat&logo=gmail)](mailto:arun_mem@yahoo.com)


> For demo access or licensing inquiries, connect on LinkedIn.
