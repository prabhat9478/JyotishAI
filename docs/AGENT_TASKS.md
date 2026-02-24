# JyotishAI — Agent Task Breakdown

## Team Structure

Three parallel agent teams working on independent domains.
Each team has a clearly scoped task with no cross-dependencies at start.

---

## TEAM 1: astro-engine (Python FastAPI)

### Scope
Build the complete Python calculation microservice at `astro-engine/`.
This is the calculation backbone — no UI, no frontend dependency.

### Task
```
Build the complete astro-engine FastAPI microservice.

Project root: C:\Prabhat\Projects\JyotishAI\astro-engine\

REFERENCE DOCS:
- C:\Prabhat\Projects\JyotishAI\CLAUDE.md
- C:\Prabhat\Projects\JyotishAI\docs\ARCHITECTURE.md
- C:\Prabhat\Projects\JyotishAI\docs\FEATURES.md

CREATE these files:

1. requirements.txt
   fastapi>=0.115, uvicorn[standard], pyswisseph, pydantic, python-dateutil,
   httpx, reportlab, python-multipart

2. main.py
   FastAPI app with CORS, routers registered

3. routers/chart.py
   POST /chart — accepts BirthData, returns full ChartData JSON
   GET /transits — current planetary positions (no birth data needed)
   POST /transits/natal — current transits vs natal chart (aspects)

4. routers/dasha.py
   POST /dasha — Vimshottari dasha from birth data
   GET /dasha/current — current mahadasha+antardasha for a profile

5. routers/yogas.py
   POST /yogas — detect yogas from ChartData
   Implement at minimum 30 classical yogas:
   - Gaja Kesari, Raj Yoga variants (5,9 lords + kendra lords), 
     Pancha Mahapurusha (Ruchaka/Bhadra/Hamsa/Malavya/Sasa),
     Viparita Raja Yoga (6,8,12 lords in 6,8,12),
     Dhana Yogas (2nd+11th lord combinations),
     Neecha Bhanga Raja Yoga, Budha Aditya,
     Chandra Mangala, Amala, Adhi, Kahala, Chamara, Saraswati

6. routers/pdf.py
   POST /pdf/report — generate full styled PDF for a report
   Uses ReportLab with deep navy + gold color theme

7. core/calculator.py
   Swiss Ephemeris wrapper using pyswisseph:
   - set_ephe_path to bundled ephemeris files
   - set_sid_mode(SIDM_LAHIRI)
   - calc_planetary_positions(jd) → dict of all planets
   - calc_lagna(jd, lat, lon) → ascendant degrees
   - calc_houses(jd, lat, lon) → 12 house cusps (Whole Sign)
   - is_retrograde(planet, jd) → bool
   - calc_julian_day(date, time, utc_offset) → float

8. core/nakshatra.py
   - get_nakshatra(longitude) → {name, pada, lord, number}
   - NAKSHATRA_LIST: all 27 with lords, degrees
   - NAKSHATRA_LORDS dasha order mapping

9. core/dasha.py
   - DASHA_YEARS: {Sun:6, Moon:10, Mars:7, Rahu:18, Jupiter:16, Saturn:19, Mercury:17, Ketu:7, Venus:20}
   - calc_dasha_balance(moon_longitude, birth_datetime) → remaining years
   - get_dasha_sequence(birth_datetime, balance) → list of {planet, start, end}
   - get_antardasha(mahadasha_planet, start, end) → list
   - get_current_dasha(sequence, today) → {mahadasha, antardasha, pratyantardasha}

10. core/yoga_rules.py
    YogaDetector class with detect_all(chart_data) → list of Yoga objects

11. core/ashtakavarga.py
    - calc_ashtakavarga(chart_data) → {planet: [12 values], sarva: [12 values]}

12. schemas/birth_data.py
    Pydantic models: BirthData, ChartData, Planet, House, Dasha, Yoga

13. Download Swiss Ephemeris data files:
    Place in astro-engine/ephe/ directory
    Use pyswisseph's bundled data or download from astro.com

TEST: Create tests/test_calculator.py
Verify Prabhat's chart:
- DOB: 18 Feb 1994, 23:07 IST, Raipur (81.38E, 21.14N)
- Expected: Lagna=Libra, Moon=Taurus/Krittika Pada 3, Sun=Aquarius
- Verify against ClickAstro output in C:\Prabhat\Personal docs\2026 Horoscope\

When done, run: openclaw system event --text "Done: astro-engine complete — all calculation endpoints working" --mode now
```

---

## TEAM 2: Next.js Web App Scaffold

### Scope
Build the Next.js 15 project structure, Supabase integration, auth,
database migrations, and core layout. No complex UI yet — foundation only.

### Task
```
Build the Next.js 15 web application scaffold.

Project root: C:\Prabhat\Projects\JyotishAI\web\

REFERENCE DOCS:
- C:\Prabhat\Projects\JyotishAI\CLAUDE.md
- C:\Prabhat\Projects\JyotishAI\docs\ARCHITECTURE.md
- C:\Prabhat\Projects\JyotishAI\docs\DATABASE.md
- C:\Prabhat\Projects\JyotishAI\docs\FEATURES.md

REFERENCE EXISTING PROJECT (same patterns): 
- C:\Prabhat\Projects\EAKC-MVP (for Supabase SSR, BullMQ, SSE patterns)

CREATE:

1. Next.js 15 app with: npx create-next-app@latest web --typescript --tailwind --app
   Then install: shadcn/ui, framer-motion, zustand, d3, @react-three/fiber,
   @react-three/drei, three, lucide-react, next-themes, @supabase/supabase-js,
   @supabase/ssr, ioredis, bullmq, zod, react-hook-form

2. src/app layout:
   (auth)/login/page.tsx
   (auth)/signup/page.tsx
   (main)/layout.tsx — sidebar + header
   (main)/dashboard/page.tsx — family profiles grid
   (main)/profile/[id]/page.tsx — profile detail + chart
   (main)/profile/[id]/reports/page.tsx — report list
   (main)/profile/[id]/reports/[reportId]/page.tsx — report viewer
   (main)/profile/[id]/chat/page.tsx — RAG chat
   (main)/transits/page.tsx — current transits (no profile)
   (main)/settings/page.tsx — preferences
   api/v1/profiles/route.ts — CRUD
   api/v1/calculate/route.ts — calls astro-engine
   api/v1/reports/generate/route.ts — SSE streaming generation
   api/v1/reports/[id]/pdf/route.ts — PDF download
   api/v1/chat/route.ts — SSE RAG chat
   api/v1/transits/route.ts — current transit positions
   api/v1/alerts/route.ts — alert CRUD

3. Supabase migrations in supabase/migrations/:
   Copy ALL SQL from C:\Prabhat\Projects\JyotishAI\docs\DATABASE.md
   Create numbered migration files:
   001_profiles.sql
   002_reports.sql
   003_report_chunks.sql (with pgvector)
   004_chat.sql
   005_alerts.sql
   006_preferences.sql
   007_hybrid_search_function.sql

4. src/lib/supabase/:
   client.ts — browser client
   server.ts — server client (SSR)
   middleware.ts — session refresh
   types.ts — Database type definitions (manually typed from schema)

5. src/lib/astro-client.ts
   Type-safe HTTP client to astro-engine:
   calculateChart(birthData) → ChartData
   getCurrentTransits() → TransitData
   getTransitsVsNatal(natal, current) → AspectData[]
   generatePDF(reportId, content) → Buffer

6. src/lib/report-generator.ts
   generateReport(profileId, reportType, language, model) — SSE streaming
   Uses OpenRouter API with structured prompts from report-prompts/

7. src/lib/report-prompts/ — one file per report type:
   career.ts, wealth.ts, yearly.ts, in_depth.ts,
   transit_jupiter.ts, transit_saturn.ts, transit_rahu_ketu.ts,
   numerology.ts, gem_recommendation.ts
   Each prompt takes ChartData and returns a structured prompt string
   Include: planetary positions, dashas, yogas, house analysis

8. src/lib/rag/:
   chunker.ts — split report text into chunks (500 tokens, 50 overlap)
   embedder.ts — embed via OpenRouter text-embedding-3-small
   retriever.ts — pgvector hybrid search using DB function
   chat.ts — RAG chat with date-aware query handling

9. src/lib/workers/:
   report-worker.ts — BullMQ worker for PDF generation
   alert-worker.ts — BullMQ scheduled job for daily alerts
   queue.ts — queue setup

10. src/components/layout/:
    Sidebar.tsx — profile switcher + nav links
    Header.tsx — breadcrumb + theme toggle + notifications bell
    ThemeProvider.tsx

11. .env.local.example:
    NEXT_PUBLIC_SUPABASE_URL=
    NEXT_PUBLIC_SUPABASE_ANON_KEY=
    SUPABASE_SERVICE_ROLE_KEY=
    OPENROUTER_API_KEY=
    ASTRO_ENGINE_URL=http://localhost:8000
    REDIS_URL=redis://localhost:6379
    NEXT_PUBLIC_APP_URL=http://localhost:3000

12. middleware.ts — Supabase session refresh + auth protection

When done, run: openclaw system event --text "Done: Next.js scaffold complete — all routes, migrations, lib files created" --mode now
```

---

## TEAM 3: UI Components & Visualizations

### Scope
Build the core visual components: Kundli chart, Dasha timeline, Yoga cards, Transit wheel.
These are independent React components that can be built in parallel.

### Task
```
Build the core UI visualization components for JyotishAI.

Project root: C:\Prabhat\Projects\JyotishAI\web\src\components\

REFERENCE DOCS:
- C:\Prabhat\Projects\JyotishAI\CLAUDE.md
- C:\Prabhat\Projects\JyotishAI\docs\FEATURES.md (Feature Group 2)

DESIGN SYSTEM:
- Color theme: Celestial Dark
  - Background: #0a0a1a (deep space)
  - Surface: #0f1729 (dark navy)
  - Border: #1e2d4a
  - Accent gold: #c9a227
  - Accent purple: #7c3aed
  - Text: #e2e8f0
  - Muted: #64748b
- Tailwind + Shadcn/ui + Framer Motion
- All components must be fully TypeScript typed

BUILD THESE COMPONENTS:

1. kundli/KundliChart.tsx
   - Props: chartData: ChartData, style: 'north_indian' | 'south_indian'
   - North Indian: diamond/square layout with 12 houses
   - South Indian: 4x3 grid layout
   - Uses D3.js or pure SVG
   - Render planet abbreviations in correct houses
   - Retrograde planets shown with ℞ superscript
   - Click house → dispatch onHouseClick(houseNum) event
   - Click planet → dispatch onPlanetClick(planet) event
   - Animated entrance (planets fly into position)
   - Show lagna marker (arrow or L)

2. kundli/PlanetInfoPanel.tsx
   - Shown when planet is clicked
   - Shows: sign, degrees, nakshatra, pada, lord, dignity, house, retrograde status
   - Aspect list (which natal planets it aspects)
   - Slide-in animation from right

3. dasha/DashaTimeline.tsx
   - Props: dashas: DashaSequence, currentDate: Date
   - D3.js horizontal timeline
   - Mahadasha blocks colored by planet (Sun=orange, Moon=white, Mars=red, etc.)
   - Antardasha sub-blocks inside each Mahadasha (thinner, same color family)
   - Current date marker: pulsing gold vertical line
   - Zoom: D3 brush/zoom to see antardasha detail
   - Click period → onPeriodClick({planet, start, end, type})
   - Today's label + "Current: Jupiter-Saturn (until Jul 2026)"

4. yoga/YogaCard.tsx
   - Props: yoga: Yoga (name, type, strength, description, effect)
   - Card with front (name + strength badge + type) and back (description + classical source)
   - Framer Motion flip on hover/click
   - Strength badge: Weak (gray) / Moderate (blue) / Strong (purple) / Exceptional (gold)
   - Type colors: Raj (gold) / Dhana (green) / Arishta (red) / Pancha Mahapurusha (purple)
   - Glow effect matching strength on card border

5. yoga/YogaGrid.tsx
   - Props: yogas: Yoga[]
   - Masonry or responsive grid of YogaCard components
   - Filter tabs: All / Raj / Dhana / Arishta / Pancha Mahapurusha
   - Sort: by strength (desc default)
   - Empty state with cosmic illustration

6. transit/TransitWheel.tsx
   - Outer ring: current transiting planets
   - Inner ring: natal chart planets
   - Aspect lines between transiting + natal (color by aspect type)
   - Legend: aspect colors
   - Date picker to change transit date
   - Animation when date changes (planets move to new positions)
   - Props: natalChart: ChartData, transitDate: Date

7. solar-system/SolarSystem3D.tsx
   - React Three Fiber scene
   - Sun at center (emissive gold sphere)
   - 9 planets as spheres with colored materials
   - Orbital rings (dashed circle lines)
   - Planet labels (html overlays via @react-three/drei Html)
   - Animated: planets slowly orbit
   - Click planet → onPlanetSelect(planet)
   - OrbitControls for pan/zoom/rotate
   - Starfield background (Points geometry)
   - Toggle: birth positions / current positions

8. chat/ChatInterface.tsx
   - Full-height chat UI
   - Message bubbles (user right, assistant left)
   - SSE streaming: assistant messages appear word by word
   - Source citations: [Career Report] chips at end of message
   - Suggested question chips below input
   - Loading state: pulsing dots
   - Copy button on assistant messages
   - Profile context shown at top: "Chatting about: Prabhat Tiwari"

9. reports/ReportViewer.tsx
   - Props: reportContent: string (markdown), reportType, language
   - Renders markdown with react-markdown
   - Collapsible sections (H2 headings = sections)
   - Language badge (EN/HI)
   - Download PDF button
   - Share button (copy link)
   - Generation metadata footer (model, date, time taken)

10. notifications/AlertBell.tsx
    - Bell icon with unread count badge
    - Dropdown panel with recent alerts
    - Alert cards: title, date, read/unread state
    - Mark all read button
    - "View all" link

11. dashboard/ProfileCard.tsx
    - Family member card with mini kundli thumbnail
    - Name, relation badge, current dasha pill
    - "View Chart" and "Generate Report" quick actions
    - Framer Motion hover lift effect
    - Glassmorphism card style

12. shared/StarField.tsx
    - Animated CSS star field for page backgrounds
    - Pure CSS or canvas-based
    - Configurable density + speed
    - Used on login page + dashboard background

When done, run: openclaw system event --text "Done: UI components complete — kundli, dasha, yoga, transit, 3D, chat all built" --mode now
```

---

## Dependencies Between Teams

```
Team 1 (astro-engine) ─── independent ───────────────────► Done
Team 2 (Next.js scaffold) ─── independent ───────────────► Done
Team 3 (UI Components) ── needs Team 2 file structure ──► Done

After all 3:
Team 4 (Integration) — wire everything together
  - Connect API routes to astro-engine
  - Wire UI components into pages
  - Test full flow: enter birth data → chart → reports → chat
```
