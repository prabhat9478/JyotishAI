# CLAUDE.md — JyotishAI

## What This Is
A cutting-edge, AI-powered Vedic astrology platform that generates ClickAstro/AstroVision-quality
horoscope reports for personal and family use. Built as a portfolio-grade full-stack project.

## Project Rules
- Never hardcode API keys — always use env vars
- All calculations via pyswisseph (same engine as AstroVision/LifeSign)
- Ayanamsha: Lahiri (Chitrapaksha) — matches ClickAstro exactly
- House system: Whole Sign (Vedic standard)
- Dasha system: Vimshottari
- LLM predictions in both English and Hindi (user toggleable)
- SSE streaming for report generation (same pattern as EAKC)
- All reports must be exportable as PDF (ReportLab, same as RakshaSutra)
- Family profiles stored in Supabase with RLS
- No commercial features needed — private/family use + portfolio

## Repo Structure
```
jyotish-ai/
├── web/                    # Next.js 15 App (frontend + BFF API routes)
│   ├── src/
│   │   ├── app/
│   │   │   ├── (auth)/     # Login, signup
│   │   │   ├── (main)/
│   │   │   │   ├── dashboard/      # Family profiles list
│   │   │   │   ├── chart/[id]/     # Interactive birth chart view
│   │   │   │   ├── reports/[id]/   # Report viewer
│   │   │   │   ├── transits/       # Real-time transit tracker
│   │   │   │   └── chat/[id]/      # AI chart chat
│   │   │   └── api/v1/
│   │   │       ├── profiles/       # CRUD family profiles
│   │   │       ├── calculate/      # Calls astro-engine
│   │   │       ├── reports/        # Generate/stream reports
│   │   │       ├── transits/       # Current planetary positions
│   │   │       └── chat/           # RAG chat over birth chart
│   │   ├── components/
│   │   │   ├── ui/                 # shadcn/ui primitives
│   │   │   ├── kundli/             # Birth chart SVG renderer
│   │   │   ├── solar-system/       # Three.js/R3F 3D visualization
│   │   │   ├── dasha-timeline/     # Interactive dasha timeline
│   │   │   ├── yoga-cards/         # Detected yoga cards
│   │   │   ├── transit-wheel/      # Transit overlay on natal chart
│   │   │   └── report-viewer/      # Streaming report display
│   │   └── lib/
│   │       ├── astro-client.ts     # HTTP client to astro-engine
│   │       ├── report-prompts/     # LLM prompt templates per report type
│   │       ├── pdf-generator/      # ReportLab PDF via Python or jsPDF
│   │       └── supabase/           # DB client + types
├── astro-engine/           # FastAPI Python microservice
│   ├── main.py
│   ├── routers/
│   │   ├── chart.py        # Birth chart calculation endpoint
│   │   ├── transits.py     # Current + future transits
│   │   ├── dasha.py        # Dasha period calculations
│   │   ├── yogas.py        # Yoga detection engine
│   │   └── pdf.py          # PDF report generation
│   ├── core/
│   │   ├── calculator.py   # pyswisseph wrapper
│   │   ├── houses.py       # House lord + placement analysis
│   │   ├── nakshatra.py    # Nakshatra + pada calculations
│   │   ├── dasha.py        # Vimshottari dasha engine
│   │   ├── yoga_rules.py   # 100+ yoga detection rules
│   │   └── ashtakavarga.py # Ashtakavarga calculation
│   └── schemas/
│       ├── birth_data.py
│       └── chart_data.py
├── workers/                # BullMQ report generation workers (Node.js)
│   └── report-worker.ts    # Async PDF generation jobs
├── supabase/
│   └── migrations/
└── docker-compose.yml
```

## Tech Stack

### Frontend (web/)
- **Framework:** Next.js 15 (App Router) + React 19 + TypeScript
- **UI:** Shadcn/UI + Tailwind CSS + Framer Motion
- **3D:** React Three Fiber (@react-three/fiber) + @react-three/drei
- **Charts/SVG:** D3.js (kundli chart rendering, dasha timeline)
- **State:** Zustand (simpler than Redux for this scope)
- **Real-time:** SSE streaming for report generation
- **Auth:** Supabase Auth (email + Google OAuth)

### Backend (astro-engine/)
- **Framework:** FastAPI (Python 3.11)
- **Astro Calculations:** pyswisseph (Swiss Ephemeris)
- **PDF:** ReportLab
- **Server:** Uvicorn

### Data
- **Database:** Supabase PostgreSQL
- **Tables:** profiles, charts, reports, chat_sessions, chat_messages
- **RLS:** Profile-level isolation (each user sees only their family)

### AI/LLM
- **Gateway:** OpenRouter (same as EAKC)
- **Model:** claude-sonnet-4-5 / gemini-2.0-flash (togglable)
- **Pattern:** Streaming SSE (same as EAKC chat)
- **Prompts:** Structured per report type (Career, Wealth, Yearly, etc.)

### Queue
- **BullMQ + Redis** (same as EAKC) — for async PDF report generation

### Deployment
- **Platform:** Dokploy on Hostinger (same as EAKC + StackArchitect)
- **Docker:** Multi-container (web + astro-engine + redis)

## Report Types to Build
1. In-Depth Horoscope (full life analysis, 50+ pages)
2. Career & Business Horoscope
3. Wealth & Fortune Horoscope
4. Yearly Horoscope
5. Jupiter Transit Predictions
6. Saturn Transit Predictions
7. Rahu-Ketu Transit Predictions
8. Numerology Report
9. Gem Recommendation Report

## Calculation Accuracy Targets
- Planetary positions: ±1 arcminute (Swiss Ephemeris standard)
- Ayanamsha: Lahiri (23°46'46" for Prabhat's birth date) ✓
- Dasha balance at birth: Matches ClickAstro output exactly
- Yoga detection: Top 50 classical yogas from BPHS

## Key Design Decisions
- **No LifeSign API** — build our own engine, full control
- **AI over pre-written text** — LLM predictions are more personalized
- **Interactive charts** — Not static images like ClickAstro
- **Offline-capable** — Local Docker, no cloud dependency for calculations
- **Hindi + English** — LLM handles bilingual output natively
- **NotebookLM replacement** — Reports auto-indexed into pgvector RAG, chat interface built in
- **Proactive alerts** — Daily digest via WhatsApp/email, smart transit alerts
- **Date-range queries** — Ask "what's happening Feb 25-28?" and get transit + report synthesis

## Core Feature Set

### Report Generation
- 9 report types (Career, Wealth, Yearly, Transits, Numerology, Gems, In-Depth)
- Streaming SSE (watch it write in real-time)
- PDF export (ClickAstro-quality via ReportLab)
- Interactive web view (same content, beautiful visualizations)
- English + Hindi toggle

### Interactive Visualizations
- Kundli chart (D3.js SVG — North Indian + South Indian styles)
- 3D Solar System at birth time (React Three Fiber)
- Dasha timeline (D3.js — scrollable, interactive)
- Yoga cards (Framer Motion — animated, strength-coded)
- Transit wheel (current planets overlaid on natal chart)

### RAG Chat (NotebookLM Replacement)
- All generated reports auto-chunked + embedded into pgvector
- Ask date-specific questions: "What does my chart say for Feb 25-28?"
- System combines: report chunks + real-time transit calculation + active dasha
- SSE streaming responses (same as EAKC chat)
- Conversation history per profile

### Proactive Alerts & Digest
- **Daily WhatsApp digest** (morning) — active dasha, Moon sign, key transits, watch-outs
- **Weekly email digest** — week overview with highlighted dates
- **Smart alerts** — significant transits flagged 3-7 days in advance
  - Rahu/Ketu over natal planets
  - Saturn/Jupiter station (retrograde/direct)
  - Dasha period changes
  - Eclipses near natal placements
- **In-app notifications** — bell icon with unread alerts

### Family Vault
- Unlimited family profiles
- Switch between profiles instantly
- All features (reports, chat, alerts) per profile
- Relation tagging (self, spouse, parent, child, sibling)

## Environment Variables
See web/.env.local — never commit this file. It is gitignored.

## Supabase Project
- Account: prabhat9478
- URL: https://mzzqsjdcqhfpjhtlrejg.supabase.co
- Anon key is already in web/.env.local
- Get service role key from: Supabase Dashboard → Project Settings → API → service_role

## Infrastructure
- Database: Supabase PostgreSQL (cloud, free tier)
- Redis: Upstash (cloud, free tier) — add REDIS_URL to .env.local once created
- astro-engine: run locally with `cd astro-engine && python main.py` (no Docker needed)
- Frontend: Next.js dev server `cd web && npm run dev`
- Deployment: Dokploy on adaptivesmartsystems.cc
