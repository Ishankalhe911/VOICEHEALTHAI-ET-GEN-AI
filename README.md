# 🎙️ VERNAI – VoiceHealth AI

**Voice-First, Multi-Agent Healthcare Guidance for India's Underserved**

---

## 🚨 Problem

A large portion of India's population relies on **regional languages and voice interaction**, yet most digital healthcare tools are:

- Text-heavy and English-first
- Difficult for low-literacy users
- Disconnected from local disease outbreaks and seasonal health trends

This leads to misunderstanding of basic symptoms, delayed preventive action, and lack of awareness about available healthcare schemes.

---

## 💡 Solution

**VoiceHealth AI (VERNAI)** is a voice-first, multi-agent healthcare assistant that:

- 🗣️ Accepts voice input in **Hindi and Tamil** (scalable to more languages)
- 🛡️ Enforces strict **safety guardrails** — no diagnosis, no prescriptions
- 📍 Delivers **region-aware, live advisories** using real-time data from IDSP, MoHFW, IMD, and trusted news sources
- 🏥 Dynamically maps the user's location to local state health insurance schemes (e.g., CMCHIS in Tamil Nadu) or defaults to Ayushman Bharat
- 🔊 Responds in the **user's own language** via high-quality Indic TTS
- ⚡ End-to-end response in **under 10 seconds**

---

## 👤 Example Use Case

**Priya (58, rural Maharashtra)**

She speaks in Hindi: *"Mujhe bahut garmi lag rahi hai aur chakkar aa rahe hain"*
*(Translation: "I'm feeling very hot and dizzy")*

The system:
1. Transcribes her voice using Groq Whisper Large V3
2. Classifies it as safe (not an emergency, not a diagnosis request)
3. Searches IDSP, MoHFW, and IMD for live heatwave advisories in Pune
4. Generates a clear, empathetic Hindi advisory using Gemini 2.5 Pro
5. Speaks the response back to her in Hindi via Sarvam AI TTS (bulbul:v3)

**Output:** *"पुणे में IMD ने हीटवेव अलर्ट जारी किया है। खूब पानी पिएं, ORS लें, और धूप में न निकलें। डॉक्टर से मिलें यदि स्थिति बिगड़े।"*

No diagnosis. No jargon. Clear voice guidance in under 10 seconds.

---

## ⚙️ Multi-Agent Architecture

VERNAI is built as a pipeline of 5 specialised agents, each with a single responsibility:

```
User Voice Input
      │
      ▼
┌──────────────────────────────────────────────────────┐
│  Agent 1: Intake Agent                               │
│  Audio-to-Text: Groq Whisper Large V3                │
│  Translation & Cleanup: Groq Llama 3.3 70b           │
└────────────────────┬─────────────────────────────────┘
                     │
                     ▼
┌──────────────────────────────────────────────────────┐
│  Agent 2: Guardrail Agent                            │
│  Safety Classification: Groq Llama 3.3 70b           │
│  Category A → Block (Emergency / Diagnosis request)  │
│  Category B → Pass (Safe informational query)        │
└────────────────────┬─────────────────────────────────┘
                     │ (only Category B passes)
                     ▼
┌──────────────────────────────────────────────────────┐
│  Agent 3: Research Agent                             │
│  Live Health Data: Serper.dev (Google Search API)    │
│  5-Level Priority Fallback Chain                     │
│  Custom Relevance Scoring Engine                     │
└────────────────────┬─────────────────────────────────┘
                     │
                     ▼
┌──────────────────────────────────────────────────────┐
│  Agent 4: Synthesis Agent                            │
│  Primary:  Gemini 2.5 Pro                            │
│  Fallback: Groq Llama 3.3 70b (auto, no downtime)   │
└────────────────────┬─────────────────────────────────┘
                     │
                     ▼
┌──────────────────────────────────────────────────────┐
│  Agent 5: TTS Agent                                  │
│  Sarvam AI bulbul:v3 (Hindi: Priya, Tamil: Rahul)   │
└────────────────────┬─────────────────────────────────┘
                     │
                     ▼
              Voice Response
```

---

## 🔬 Research Agent — Deep Dive

The Research Agent is the most technically complex component. It solves the core challenge: **getting fresh, relevant, official health data in real time without hallucination.**

### 5-Level Priority Fallback Chain

All 3 API calls fire **simultaneously** using `asyncio.gather()` — max latency equals the slowest single call (~3-5s), not the sum.

| Priority | Source | Trust Level |
| :--- | :--- | :--- |
| **P1** | IDSP Official (`site:idsp.mohfw.gov.in`) | Highest — government disease surveillance |
| **P2** | Broad MoHFW / NCDC / NDMA / IMD search | High — official health + disaster + weather |
| **P4** | Trusted news (NDTV, Times of India, The Hindu) | Medium — recent but not official |
| **P5** | Static safe fallback message | Last resort — never crashes |

> P3 (direct IDSP PDF scrape) dropped for MVP — PDFs cannot be parsed within a 10s response window. Planned for v2 with PyMuPDF pipeline.

### Relevance Scoring Engine

A custom heuristic scorer evaluates every result set — a result only scores a point if it matches **both** a symptom keyword AND the location. This prevents a result about Nipah virus in Kerala from scoring against a heatwave query in Pune.

### Priority Override Mechanism

If P4 (news) scores **2+ more relevance hits** than P2 (MoHFW), P4 wins — even though P2 has higher trust. This ensures the most symptom-specific data is always used.

```
Example: User says "feeling very hot, dizziness" in Pune
→ P2 returns 10 MoHFW results  (score: 1 — mostly generic disease data)
→ P4 returns 5 news results    (score: 4 — heatwave + IMD + Pune specific)
→ OVERRIDE: P4 selected, logged clearly in console
```

---

## 🛡️ Safety & Guardrails

The Guardrail Agent runs on every request before any search or LLM synthesis:

| Input Type | Action |
| :--- | :--- |
| Emergency symptoms (chest pain, unconsciousness) | Instant hardcoded response → "Call 112 immediately" |
| Direct diagnosis request ("Do I have dengue?") | Blocked → "I cannot diagnose. See a doctor." |
| Safe informational query | Passes through to Research + Synthesis |

- **Fail-open design:** if Guardrail itself crashes, defaults to Category B (safe) — never blocks a real user
- **`response_format: json_object`** forces valid JSON from Groq — zero parse failures
- **Hardcoded emergency response** — never touches an LLM for Category A, zero hallucination risk

---

## ⚡ Latency Architecture

Every design decision was made with sub-10s response in mind:

| Agent | Approach | Latency |
| :--- | :--- | :--- |
| **Intake** | Groq Whisper Large V3 (fastest Whisper endpoint) | ~1-2s |
| **Guardrail** | Groq Llama 3.3 70b (`max_tokens: 150`) | ~0.3s |
| **Research** | P1 + P2 + P4 fire in parallel via `asyncio.gather()` | ~3-5s |
| **Synthesis** | Gemini 2.5 Pro (`max_tokens: 500`) | ~2-3s |
| **TTS** | Sarvam AI bulbul:v3 | ~1-2s |
| **Total** | Agents partially overlap in execution | **~7-9s** |

---

## 🧰 Tech Stack

| Layer | Technology |
| :--- | :--- |
| Backend | FastAPI (Python) |
| ASR | Groq Whisper Large V3 |
| Translation | Groq Llama 3.3 70b Versatile |
| Safety Guardrail | Groq Llama 3.3 70b Versatile |
| Health Data Search | Serper.dev (Google Search API, `tbs=qdr:m3`) |
| Advisory Generation | Gemini 2.5 Pro → Groq Llama 3.3 70b (auto fallback) |
| TTS | Sarvam AI bulbul:v3 (Hindi: Priya, Tamil: Rahul) |
| Frontend | Vanilla HTML / CSS / JS (single file, no framework) |

---

## 📂 Project Structure

```
voicehealth-ai/
├── agents/
│   ├── intake_agent.py         # Groq Whisper + Llama 3.3 70b translation
│   ├── guardrail_agent.py      # Safety classification (Groq Llama 3.3 70b)
│   ├── research_agent.py       # 5-level fallback + relevance scoring engine
│   ├── synthesis_agent.py      # Gemini 2.5 Pro + Groq auto fallback
│   └── tts_agent.py            # Sarvam AI bulbul:v3 TTS
├── api/
│   ├── main.py                 # FastAPI app entry point
│   └── routes/
│       ├── audio.py            # POST /audio/advise (full pipeline)
│       └── health.py           # GET /health (status check)
├── core/
│   ├── config.py               # API keys via pydantic-settings + .env
│   └── utils.py                # Language detection, shared helpers
├── prompts/
│   ├── guardrail_prompt.txt    # Classification rules for Guardrail agent
│   └── synthesis_prompt.txt    # Advisory generation instructions
├── frontend/
│   └── index.html              # Single-file voice UI (no framework)
├── tests/
│   └── test_research_agent.py  # Unit + integration tests
├── .env                        # API keys (gitignored)
├── requirements.txt
└── README.md
```

---

## 🚀 Running Locally

**1. Install dependencies**
```bash
pip install -r requirements.txt
```

**2. Set up your `.env` file**
```
GROQ_API_KEY=your_groq_key
GEMINI_API_KEY=your_gemini_key
SERPER_API_KEY=your_serper_key
SARVAM_API_KEY=your_sarvam_key
```

**3. Start the backend**
```bash
uvicorn api.main:app --reload
```

**4. Open the frontend**
```
Open frontend/index.html in your browser
```

**5. Test the API directly**
```
http://127.0.0.1:8000/docs
```

---

## 📊 Data Sources

| Source | Type | Used For |
| :--- | :--- | :--- |
| IDSP (`idsp.mohfw.gov.in`) | Government disease surveillance | P1 search |
| MoHFW / NCDC / NCVBDC | Official health advisories | P2 search |
| NDMA / IMD | Disaster + weather alerts (heatwave, floods) | P2 search |
| NDTV / Times of India / The Hindu | Health news | P4 search |

All data filtered to **last 90 days** using Serper's `tbs=qdr:m3` parameter.
No personal health data is collected or stored.

---

## 🔮 Future Scope

- Add Marathi, Bengali, Telugu support
- Offline / low-connectivity mode
- WhatsApp integration via Twilio
- P3 IDSP PDF parsing pipeline (PyMuPDF + LLM summariser)
- Voice biomarker-based early risk detection

---

## ⚠️ Disclaimer

VoiceHealth AI provides **general health awareness information only**.
It does **not** offer medical diagnosis, treatment advice, or financial recommendations.
All responses are grounded in live public health data from official government sources.
Designed to **support awareness** — not replace healthcare professionals.

---

## 🏁 Summary

VoiceHealth AI is not just an AI tool —
it's a **bridge between healthcare knowledge and the people who are currently excluded from it**.

Built for ET GenAI Hackathon · Problem Statement 5
