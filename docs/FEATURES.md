# JyotishAI — Full Features Specification

## Overview
JyotishAI is an AI-powered Vedic astrology platform. It generates ClickAstro/AstroVision-quality
reports using Swiss Ephemeris calculations, enhances them with LLM-generated predictions, provides
interactive visualizations, and replaces NotebookLM with a built-in RAG chat system.

---

## FEATURE GROUP 1: Birth Chart Engine

### F1.1 — Birth Data Input
- Name, date of birth, time of birth (HH:MM, AM/PM), place of birth
- Geolocation lookup from place name (Google Places or local DB)
- Timezone auto-detection from coordinates
- Relation tag: Self / Spouse / Parent / Child / Sibling / Other

### F1.2 — Planetary Calculations (via pyswisseph)
- All 9 Vedic planets: Sun, Moon, Mars, Mercury, Jupiter, Venus, Saturn, Rahu, Ketu
- Sidereal longitude with Lahiri (Chitrapaksha) ayanamsha
- Degrees, minutes, seconds precision
- Retrograde status for each planet
- Speed (deg/day) — for combustion/stationary detection

### F1.3 — House System
- Whole Sign house system (Vedic standard)
- Lagna (Ascendant) calculated precisely
- All 12 house cusps
- House lords identified for each house

### F1.4 — Nakshatra Analysis
- Nakshatra for each planet (27 nakshatras)
- Pada (1-4) for each planet
- Nakshatra lord
- Birth star qualities: Gana, Yoni, Pakshibalum, Vriksha

### F1.5 — Divisional Charts
- Navamsa (D9) — marriage and dharma
- Dashamamsha (D10) — career and profession
- Saptamamsha (D7) — children
- Drekana (D3) — siblings

### F1.6 — Special Points
- Arudha Lagna (AL)
- Atmakaraka + Karakamsha
- Yogakaraka identification
- Upapada Lagna (UL)

### F1.7 — Ashtakavarga
- Individual Ashtakavarga for each planet
- Sarvashtakavarga (total score per sign)
- Bindus per house

### F1.8 — Vimshottari Dasha System
- Mahadasha sequence with exact start/end dates
- Antardasha (sub-period) for each Mahadasha
- Pratyantardasha (sub-sub-period)
- Balance at birth (how much of first dasha remains)
- Current active Mahadasha + Antardasha

---

## FEATURE GROUP 2: Interactive Visualizations

### F2.1 — Kundli Chart (D3.js SVG)
- North Indian diamond style (default)
- South Indian grid style (toggle)
- Animated planet placement on load (Framer Motion)
- Click any house → side panel shows house lord, planets, significations
- Click any planet → shows degrees, nakshatra, dignity, aspects
- Hover tooltips with planet abbreviations
- Retrograde planets marked with ℞
- Print/export as SVG or PNG

### F2.2 — 3D Solar System (React Three Fiber)
- Accurate planetary positions at birth time
- Orbital animations at relative speeds
- Stars/nebula background (Three.js ShaderMaterial)
- Click planet → zoom + info card
- Toggle heliocentric / geocentric view
- Toggle: birth time positions ↔ current positions

### F2.3 — Navamsa Chart
- Same D3.js rendering as Lagna chart
- Toggle between Rashi + Navamsa side by side
- Planet dignity highlighting (exalted=gold, debilitated=red, own=green)

### F2.4 — Dasha Timeline (D3.js)
- Horizontal scrollable timeline from birth to age 120
- Mahadasha blocks with color per planet
- Antardasha nested inside each Mahadasha
- Current date marker (pulsing dot)
- Click any period → show active dasha summary + predictions
- Zoom in/out (D3 zoom behavior)

### F2.5 — Yoga Cards (Framer Motion)
- One card per detected yoga
- Strength indicator: Weak / Moderate / Strong / Exceptional
- Benefic (green glow) / Malefic (red glow) / Mixed (amber)
- Card flip animation → back shows classical text source (BPHS reference)
- Filter by category: Raj Yoga / Dhana Yoga / Arishta / Pancha Mahapurusha / etc.

### F2.6 — Transit Wheel
- Outer wheel: current planetary positions (real-time)
- Inner wheel: natal chart
- Aspect lines drawn between transiting and natal planets
- Color coded: trine=green, sextile=blue, square=red, opposition=orange, conjunction=purple
- Slider to scrub forward/backward in time (date picker)
- Auto-refresh every 10 minutes for current positions

### F2.7 — Ashtakavarga Grid
- 8×12 grid display (planet × sign)
- Color intensity based on bindu count (0-8)
- Sarvashtakavarga bar chart below
- Hover: shows which planet contributes bindus

---

## FEATURE GROUP 3: Report Generation

### F3.1 — Report Types
All 9 report types, matching ClickAstro's catalog:

| Report | Sections | Approx Pages |
|--------|----------|--------------|
| In-Depth Horoscope | Personality, Houses (all 12), Planetary effects, Dasha predictions | 50+ |
| Career & Business | Career aptitude, Business potential, Best periods, Challenges | 15-20 |
| Wealth & Fortune | Dhana yogas, Property, Investments, Best periods | 15 |
| Yearly Horoscope | Month-by-month predictions for current year | 15-20 |
| Jupiter Transit | Transit through current + next sign, house effects | 12-15 |
| Saturn Transit | Sade Sati analysis, house transit effects | 15-18 |
| Rahu-Ketu Transit | Nodal axis effects, karmic themes | 10-12 |
| Numerology Report | Birth number, Name number, Destiny number, Lucky periods | 8-10 |
| Gem Recommendation | Shadbala analysis, recommended gems, wearing instructions | 6-8 |

### F3.2 — Report Generation UI
- "Generate" button per report type
- Progress indicator: "Analyzing chart... Generating predictions... Formatting..."
- SSE streaming — text appears word by word like ChatGPT
- Language toggle: English / Hindi (before generation)
- Model selector: Claude / Gemini / GPT-4o

### F3.3 — Report Storage
- Full text stored in Supabase (reports table)
- Timestamp, model used, language stored
- Re-generation possible (keeps history)
- Mark as favorite

### F3.4 — PDF Export
- ClickAstro-quality formatting via ReportLab (Python)
- Military/spiritual color theme (deep navy + gold)
- Birth details header
- Charts embedded as SVG images
- Table of contents
- Page numbers
- Download button → direct PDF

### F3.5 — Web View
- Same content as PDF but interactive HTML
- Collapsible sections
- Internal links (click "Jupiter in 7th" → go to planet detail)
- Share via link (optional, for family)

---

## FEATURE GROUP 4: RAG Chat (NotebookLM Replacement)

### F4.1 — Auto-Indexing
- Every generated report automatically chunked + embedded
- Chunks stored in Supabase with pgvector
- Metadata: report_type, profile_id, date_range mentioned, language
- Re-index on report update

### F4.2 — Chart Context Building
- System prompt auto-built from birth chart:
  - Lagna, Moon sign, Sun sign
  - Active Mahadasha + Antardasha
  - Current transits
  - All detected yogas
  - Shadbala strengths

### F4.3 — Date-Aware RAG
- Parser detects date mentions in query ("Feb 25-28", "next week", "March 2026")
- Retrieves relevant report chunks for those dates
- Calculates planetary positions for requested dates on-the-fly
- Synthesizes: report chunks + real-time calculations + dasha context

### F4.4 — Chat Interface
- SSE streaming responses (same as EAKC)
- Conversation history per profile
- Suggested questions (chips): 
  - "What's favorable this week?"
  - "When is my next career peak period?"
  - "What does my Rahu transit mean?"
  - "Best dates for important decisions this month?"
- Copy response button
- Thumbs up/down feedback

### F4.5 — Source Citations
- NotebookLM-style citations: [Career Report, p.4] [Saturn Transit, p.8]
- Click citation → jumps to that section in the report viewer
- Confidence indicator per answer

---

## FEATURE GROUP 5: Proactive Alerts & Digest

### F5.1 — Daily WhatsApp Digest
Sent every morning (configurable time, default 7:00 AM):

```
🌅 Aaj Ka Din — 25 Feb 2026 (Prabhat)

Active Dasha: Jupiter-Saturn
Moon: Entering Sagittarius at 3:40 PM
Today's Energy: ⭐⭐⭐ (3/5)

⚠️ Watch: Mars aspects natal Sun today — avoid confrontations
✅ Opportunity: Evening favorable for financial decisions
🔮 Tomorrow: Mercury direct — communication improves

Ask me anything: [link to chat]
```

### F5.2 — Weekly Email Digest
Sent every Sunday evening:
- Week-at-a-glance calendar with daily energy ratings (1-5 stars)
- Highlighted dates (avoid / favorable)
- Active dasha summary
- Notable transits this week
- Top 3 things to focus on

### F5.3 — Smart Transit Alerts
Triggered when:
- Any planet transits within 2° of natal planet (orb configurable)
- Dasha period changes (Mahadasha or Antardasha)
- Saturn/Jupiter stations (retrograde/direct)
- Eclipses within 10° of natal placements
- Rahu/Ketu transit over natal Sun/Moon/Lagna

Alert format:
```
⚡ Transit Alert — 3 days from now (Feb 28)
Rahu conjuncts your natal Jupiter (exact)
→ Your career report flags this as "major pivot window"
→ Be open to unexpected opportunities
[View full analysis]
```

### F5.4 — In-App Notification Center
- Bell icon in header with unread count
- Alert history (last 30 days)
- Read/unread state
- Click → opens relevant report section or chart view
- Notification preferences (which alerts, which channels)

### F5.5 — Alert Engine (Background Worker)
- Runs daily at midnight via BullMQ scheduled job
- For each profile: calculates tomorrow's planetary positions
- Runs aspect detection against natal chart (configurable orbs)
- Checks dasha transitions in next 7 days
- Queries report RAG for context on detected events
- Generates alert text via LLM
- Dispatches via WhatsApp + in-app

---

## FEATURE GROUP 6: Family Vault

### F6.1 — Profile Management
- Unlimited family profiles per account
- Profile card: name, photo, relation, current dasha, thumbnail kundli
- Quick-switch between profiles (dropdown in header)
- Profile comparison (side-by-side charts) — future feature

### F6.2 — Per-Profile Data
Each profile has its own:
- Birth chart (calculated and cached)
- Full report history
- RAG chat conversations
- Alert preferences and history
- PDF export history

### F6.3 — Kundli Matching (Future)
- Ashta Koota compatibility score
- Mangalik analysis
- Composite chart overlay
- Compatibility report (as additional report type)

---

## FEATURE GROUP 7: Settings & Preferences

### F7.1 — Calculation Preferences
- Ayanamsha: Lahiri (default) / KP / Raman / Krishnamurti
- House system: Whole Sign (default) / Equal / Placidus / Koch
- Dasha system: Vimshottari (default) / Yogini / Ashtottari

### F7.2 — Notification Preferences
- WhatsApp digest: On/Off, time
- Email digest: On/Off, day
- Transit alerts: On/Off, orb setting (1°-5°)
- Alert types: which transits to alert on

### F7.3 — Display Preferences
- Default chart style: North Indian / South Indian
- Language: English / Hindi
- Theme: Dark (default) / Light
- AI model preference: Claude / Gemini / GPT-4o

### F7.4 — Data Management
- Export all data (JSON)
- Delete profile
- Clear report history
- Re-calculate all charts (after changing ayanamsha)
