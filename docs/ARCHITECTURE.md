# JyotishAI — Architecture Document

## Overview

JyotishAI is a full-stack Vedic astrology platform that generates AstroVision-quality reports
using the same underlying calculation engine (Swiss Ephemeris) enhanced with modern AI for
prediction generation. Built for personal/family use and portfolio showcase.

---

## System Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                          Browser / Client                            │
│                                                                       │
│  ┌──────────────────────────────────────────────────────────┐       │
│  │  Next.js 15 (App Router)                                  │       │
│  │  ┌──────────────┐  ┌───────────────┐  ┌───────────────┐  │       │
│  │  │ Kundli Chart │  │ 3D Solar Sys  │  │ Dasha Timeline│  │       │
│  │  │ (D3.js SVG)  │  │ (R3F/Three.js)│  │ (D3.js)       │  │       │
│  │  └──────────────┘  └───────────────┘  └───────────────┘  │       │
│  │  ┌──────────────┐  ┌───────────────┐  ┌───────────────┐  │       │
│  │  │ Report Viewer│  │  Yoga Cards   │  │  AI Chat      │  │       │
│  │  │ (SSE stream) │  │  (Framer Mot) │  │  (SSE stream) │  │       │
│  │  └──────────────┘  └───────────────┘  └───────────────┘  │       │
│  └──────────────────────────────────────────────────────────┘       │
└─────────────────────────────────────────────────────────────────────┘
                               │ HTTPS
┌─────────────────────────────────────────────────────────────────────┐
│                     Next.js API Routes (/api/v1)                     │
│                                                                       │
│  /profiles     /calculate     /reports      /chat     /transits      │
└─────────────────────────────────────────────────────────────────────┘
          │                │                   │
          ▼                ▼                   ▼
┌─────────────┐   ┌──────────────────┐   ┌──────────────────┐
│  Supabase   │   │  astro-engine    │   │    OpenRouter    │
│  PostgreSQL │   │  (FastAPI)       │   │    (LLM API)     │
│             │   │                  │   │                  │
│  profiles   │   │  pyswisseph      │   │  claude-sonnet   │
│  charts     │   │  yoga_rules      │   │  gemini-flash    │
│  reports    │   │  dasha engine    │   │  (streaming SSE) │
│  chat_msgs  │   │  ReportLab PDF   │   │                  │
└─────────────┘   └──────────────────┘   └──────────────────┘
          │                                      │
          ▼                                      ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      BullMQ + Redis                                   │
│              (Async PDF report generation queue)                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Data Flow: Birth Chart Calculation

```
User inputs birth data (name, date, time, place)
    │
    ▼
Next.js API: POST /api/v1/calculate
    │
    ▼
astro-engine: POST /chart
    │
    ├── pyswisseph.calc_ut() → planetary longitudes
    ├── Ayanamsha correction (Lahiri)
    ├── House cusps (Whole Sign)
    ├── Nakshatra + Pada for each planet
    ├── Vimshottari Dasha balance at birth
    ├── Yoga detection (yoga_rules.py — 100+ rules)
    └── Ashtakavarga scores
    │
    ▼
ChartData JSON stored in Supabase (profiles.chart_data)
    │
    ▼
Next.js renders:
    ├── Kundli SVG (D3.js — traditional North/South Indian style)
    ├── 3D Solar System (React Three Fiber)
    └── Dasha Timeline (D3.js interactive)
```

---

## Data Flow: Report Generation

```
User requests report (e.g., Career Horoscope)
    │
    ▼
Next.js API: POST /api/v1/reports/generate
    │
    ├── Fetch chart_data from Supabase
    ├── Build structured prompt (report-prompts/career.ts)
    │   └── Includes: planetary positions, houses, dashas, yogas,
    │                 transits, nakshatra, divisional charts
    ├── Stream to OpenRouter (SSE)
    │   └── Model: claude-sonnet / gemini-flash (user choice)
    │
    ▼
Client receives streaming chunks → renders progressively
    │
    ▼
On completion:
    ├── Save full report text to Supabase (reports table)
    └── Enqueue PDF generation job (BullMQ)
        └── Worker: calls astro-engine /pdf endpoint
            └── ReportLab generates styled PDF
                └── Stored in Supabase Storage → download link
```

---

## Data Flow: AI Chart Chat

```
User asks: "What does my Jupiter dasha mean for career?"
    │
    ▼
POST /api/v1/chat/completions (SSE)
    │
    ├── Retrieve chart_data (context)
    ├── Retrieve conversation history (last 10 messages)
    ├── Build system prompt:
    │   "You are a Vedic astrology expert. Here is the birth chart:
    │    Lagna: Libra, Jupiter in 7th house, Active dasha: Jupiter-Saturn..."
    │
    ▼
OpenRouter streaming → SSE to client
    │
    ▼
Store message in chat_messages table
```

---

## Calculation Engine Details (astro-engine)

### Planetary Positions
```python
import swisseph as swe

swe.set_ephe_path('/path/to/ephe')
swe.set_sid_mode(swe.SIDM_LAHIRI)  # Chitrapaksha ayanamsha

jd = swe.julday(year, month, day, hour_ut)
for planet in [swe.SUN, swe.MOON, swe.MARS, ...]:
    pos, _ = swe.calc_ut(jd, planet, swe.FLG_SIDEREAL)
    longitude = pos[0]  # Sidereal longitude
```

### House Calculation (Whole Sign)
```python
# Whole Sign: Lagna determines house 1
# Each house = one complete sign
lagna_sign = int(ascendant / 30)  # 0=Aries, 1=Taurus, ...
houses = [(lagna_sign + i) % 12 for i in range(12)]
```

### Vimshottari Dasha
```python
DASHA_YEARS = {
    'Ketu': 7, 'Venus': 20, 'Sun': 6, 'Moon': 10,
    'Mars': 7, 'Rahu': 18, 'Jupiter': 16, 'Saturn': 19, 'Mercury': 17
}
DASHA_ORDER = ['Ketu', 'Venus', 'Sun', 'Moon', 'Mars',
               'Rahu', 'Jupiter', 'Saturn', 'Mercury']

# Balance at birth = remaining nakshatra traversal
nakshatra_elapsed_fraction = (moon_longitude % (360/27)) / (360/27)
```

### Yoga Detection (sample rules)
```python
def detect_gaja_kesari(chart):
    """Moon and Jupiter in mutual kendras (1,4,7,10 from each other)"""
    moon_house = chart.get_house('Moon')
    jupiter_house = chart.get_house('Jupiter')
    diff = abs(moon_house - jupiter_house)
    return diff in [0, 3, 6, 9]

def detect_raj_yoga(chart):
    """Lords of trine (1,5,9) and kendra (1,4,7,10) conjunct/exchange"""
    ...
```

---

## Database Schema

```sql
-- Family profiles
CREATE TABLE profiles (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES auth.users,
    name TEXT NOT NULL,
    relation TEXT,  -- self, spouse, parent, child, sibling
    birth_date DATE NOT NULL,
    birth_time TIME NOT NULL,
    birth_place TEXT NOT NULL,
    latitude DECIMAL(9,6),
    longitude DECIMAL(9,6),
    timezone TEXT,
    chart_data JSONB,  -- Full calculated chart (cached)
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Generated reports
CREATE TABLE reports (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    profile_id UUID REFERENCES profiles,
    report_type TEXT NOT NULL,  -- career, yearly, wealth, transit_jupiter, etc.
    language TEXT DEFAULT 'en',  -- en, hi
    content TEXT,               -- Full report text (markdown)
    pdf_url TEXT,               -- Supabase Storage URL
    model_used TEXT,            -- LLM model that generated it
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- AI Chat sessions
CREATE TABLE chat_sessions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    profile_id UUID REFERENCES profiles,
    title TEXT,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE chat_messages (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    session_id UUID REFERENCES chat_sessions,
    role TEXT CHECK (role IN ('user', 'assistant')),
    content TEXT,
    created_at TIMESTAMPTZ DEFAULT NOW()
);
```

---

## UI/UX Design Direction

### Theme: **Celestial Dark**
- Deep space background (#0a0a1a) with subtle star field animation
- Gold/amber accents (#c9a227) for planetary highlights
- Gradient overlays (indigo → purple) for section transitions
- Glassmorphism cards for profile/report cards

### Key Visual Components

**1. Interactive Kundli Chart (D3.js)**
- North Indian style diamond chart (default)
- South Indian style grid chart (toggle)
- Animated planet placement on load
- Click planet → show detailed info panel
- Hover → show degrees, nakshatra, dignity

**2. 3D Solar System (React Three Fiber)**
- Actual planetary positions at birth time
- Orbiting animation with correct speeds
- Click planet → zoom in with info card
- Toggle between heliocentric/geocentric view

**3. Dasha Timeline (D3.js)**
- Horizontal scrollable timeline
- Major dasha → sub-periods nested
- Current position highlighted
- Hover → show predictions for that period

**4. Yoga Cards (Framer Motion)**
- Card flip animation to reveal yoga details
- Strength indicator (weak/medium/strong)
- Benefic/malefic color coding
- Expandable to show classical text source

---

## Development Phases

### Phase 1 — Core Engine (Week 1-2)
- [ ] astro-engine FastAPI setup
- [ ] pyswisseph integration + Lahiri ayanamsha
- [ ] Planetary positions endpoint
- [ ] House/lagna calculation
- [ ] Nakshatra + pada
- [ ] Basic Vimshottari dasha

### Phase 2 — Web Foundation (Week 2-3)
- [ ] Next.js 15 project scaffold
- [ ] Supabase setup (auth + DB)
- [ ] Profile CRUD (family members)
- [ ] Static kundli chart (D3.js SVG)
- [ ] Dasha timeline component

### Phase 3 — AI Reports (Week 3-4)
- [ ] OpenRouter integration (streaming SSE)
- [ ] Report prompt templates (all 9 types)
- [ ] Report viewer component (streaming)
- [ ] Language toggle (EN/HI)
- [ ] PDF generation (ReportLab via astro-engine)

### Phase 4 — Advanced Features (Week 4-6)
- [ ] Yoga detection engine (50+ yogas)
- [ ] Yoga cards UI
- [ ] 3D Solar System (React Three Fiber)
- [ ] Transit tracker (real-time positions)
- [ ] AI Chart Chat (RAG over birth chart)

### Phase 5 — Polish (Week 6)
- [ ] Celestial dark theme refinement
- [ ] Framer Motion animations throughout
- [ ] Mobile responsive
- [ ] Docker setup + Dokploy deployment

---

## Environment Variables

```env
# Supabase
NEXT_PUBLIC_SUPABASE_URL=
NEXT_PUBLIC_SUPABASE_ANON_KEY=
SUPABASE_SERVICE_ROLE_KEY=

# OpenRouter
OPENROUTER_API_KEY=

# Astro Engine
ASTRO_ENGINE_URL=http://astro-engine:8000

# Redis
REDIS_URL=redis://redis:6379
```
