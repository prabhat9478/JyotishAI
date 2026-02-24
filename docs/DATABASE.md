# JyotishAI — Database Schema

## Stack
- PostgreSQL 15 via Supabase (hosted, ap-south-1)
- pgvector extension for RAG embeddings
- Row Level Security (RLS) on all user tables
- Supabase Auth for user management

---

## Tables

### profiles
Family member birth chart profiles.
```sql
CREATE TABLE profiles (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES auth.users NOT NULL,
    name TEXT NOT NULL,
    relation TEXT CHECK (relation IN ('self','spouse','parent','child','sibling','other')),
    birth_date DATE NOT NULL,
    birth_time TIME NOT NULL,
    birth_place TEXT NOT NULL,
    latitude DECIMAL(9,6) NOT NULL,
    longitude DECIMAL(9,6) NOT NULL,
    timezone TEXT NOT NULL,
    chart_data JSONB,           -- Full calculated chart (cached from astro-engine)
    chart_calculated_at TIMESTAMPTZ,
    avatar_url TEXT,
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- RLS
ALTER TABLE profiles ENABLE ROW LEVEL SECURITY;
CREATE POLICY "Users see own profiles" ON profiles
    FOR ALL USING (auth.uid() = user_id);
```

### reports
Generated horoscope reports.
```sql
CREATE TABLE reports (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    profile_id UUID REFERENCES profiles ON DELETE CASCADE NOT NULL,
    report_type TEXT NOT NULL CHECK (report_type IN (
        'in_depth', 'career', 'wealth', 'yearly',
        'transit_jupiter', 'transit_saturn', 'transit_rahu_ketu',
        'numerology', 'gem_recommendation'
    )),
    language TEXT NOT NULL DEFAULT 'en' CHECK (language IN ('en', 'hi')),
    content TEXT,               -- Full report markdown
    summary TEXT,               -- 2-3 line summary
    model_used TEXT,            -- LLM model that generated
    pdf_url TEXT,               -- Supabase Storage download URL
    pdf_generated_at TIMESTAMPTZ,
    is_favorite BOOLEAN DEFAULT false,
    year INTEGER,               -- For yearly reports
    generation_status TEXT DEFAULT 'pending' CHECK (generation_status IN (
        'pending', 'generating', 'complete', 'failed'
    )),
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- RLS via profile ownership
ALTER TABLE reports ENABLE ROW LEVEL SECURITY;
CREATE POLICY "Users see own reports" ON reports
    FOR ALL USING (
        profile_id IN (SELECT id FROM profiles WHERE user_id = auth.uid())
    );
```

### report_chunks
Chunked + embedded report content for RAG.
```sql
CREATE EXTENSION IF NOT EXISTS vector;

CREATE TABLE report_chunks (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    report_id UUID REFERENCES reports ON DELETE CASCADE NOT NULL,
    profile_id UUID REFERENCES profiles ON DELETE CASCADE NOT NULL,
    chunk_index INTEGER NOT NULL,
    content TEXT NOT NULL,
    embedding vector(1536),     -- text-embedding-3-small (OpenAI via OpenRouter)
    metadata JSONB,             -- {report_type, date_mentions, section_title, page}
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX ON report_chunks USING hnsw (embedding vector_cosine_ops);
CREATE INDEX ON report_chunks (profile_id);
CREATE INDEX ON report_chunks USING gin (metadata);

-- RLS
ALTER TABLE report_chunks ENABLE ROW LEVEL SECURITY;
CREATE POLICY "Users see own chunks" ON report_chunks
    FOR ALL USING (
        profile_id IN (SELECT id FROM profiles WHERE user_id = auth.uid())
    );
```

### chat_sessions
RAG chat conversation sessions.
```sql
CREATE TABLE chat_sessions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    profile_id UUID REFERENCES profiles ON DELETE CASCADE NOT NULL,
    title TEXT,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

ALTER TABLE chat_sessions ENABLE ROW LEVEL SECURITY;
CREATE POLICY "Users see own sessions" ON chat_sessions
    FOR ALL USING (
        profile_id IN (SELECT id FROM profiles WHERE user_id = auth.uid())
    );
```

### chat_messages
Individual messages in RAG chat.
```sql
CREATE TABLE chat_messages (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    session_id UUID REFERENCES chat_sessions ON DELETE CASCADE NOT NULL,
    role TEXT NOT NULL CHECK (role IN ('user', 'assistant')),
    content TEXT NOT NULL,
    sources JSONB,              -- [{report_id, chunk_id, report_type, excerpt}]
    model_used TEXT,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

ALTER TABLE chat_messages ENABLE ROW LEVEL SECURITY;
CREATE POLICY "Users see own messages" ON chat_messages
    FOR ALL USING (
        session_id IN (
            SELECT id FROM chat_sessions WHERE profile_id IN (
                SELECT id FROM profiles WHERE user_id = auth.uid()
            )
        )
    );
```

### transit_alerts
Generated and dispatched transit alerts.
```sql
CREATE TABLE transit_alerts (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    profile_id UUID REFERENCES profiles ON DELETE CASCADE NOT NULL,
    alert_type TEXT NOT NULL CHECK (alert_type IN (
        'planet_transit', 'dasha_change', 'station',
        'eclipse', 'nodal_transit', 'digest'
    )),
    title TEXT NOT NULL,
    content TEXT NOT NULL,      -- Full alert text (LLM generated)
    trigger_date DATE NOT NULL, -- Date the event occurs
    planet TEXT,                -- Planet involved
    natal_planet TEXT,          -- Natal planet being aspected/transited
    orb DECIMAL(4,2),           -- Exact orb at trigger_date
    dispatched_whatsapp BOOLEAN DEFAULT false,
    dispatched_email BOOLEAN DEFAULT false,
    is_read BOOLEAN DEFAULT false,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

ALTER TABLE transit_alerts ENABLE ROW LEVEL SECURITY;
CREATE POLICY "Users see own alerts" ON transit_alerts
    FOR ALL USING (
        profile_id IN (SELECT id FROM profiles WHERE user_id = auth.uid())
    );
```

### user_preferences
Per-user settings and preferences.
```sql
CREATE TABLE user_preferences (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES auth.users UNIQUE NOT NULL,
    ayanamsha TEXT DEFAULT 'lahiri',
    house_system TEXT DEFAULT 'whole_sign',
    dasha_system TEXT DEFAULT 'vimshottari',
    default_language TEXT DEFAULT 'en',
    chart_style TEXT DEFAULT 'north_indian',
    preferred_model TEXT DEFAULT 'anthropic/claude-sonnet-4-5',
    whatsapp_number TEXT,
    whatsapp_digest_enabled BOOLEAN DEFAULT true,
    whatsapp_digest_time TIME DEFAULT '07:00:00',
    email_digest_enabled BOOLEAN DEFAULT true,
    email_digest_day TEXT DEFAULT 'sunday',
    alert_orb DECIMAL(3,1) DEFAULT 2.0,
    alert_enabled BOOLEAN DEFAULT true,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

ALTER TABLE user_preferences ENABLE ROW LEVEL SECURITY;
CREATE POLICY "Users see own prefs" ON user_preferences
    FOR ALL USING (auth.uid() = user_id);
```

---

## Chart Data JSON Structure (profiles.chart_data)

```json
{
  "calculated_at": "2026-02-24T09:00:00Z",
  "ayanamsha": "lahiri",
  "ayanamsha_value": 23.779,
  "julian_day": 2461096.5,
  "lagna": {
    "sign": "Libra",
    "sign_num": 6,
    "degrees": 12.45,
    "lord": "Venus"
  },
  "planets": {
    "Sun": {
      "sign": "Aquarius", "sign_num": 10, "degrees": 305.23,
      "house": 5, "nakshatra": "Dhanistha", "pada": 2,
      "retrograde": false, "combust": false, "lord": "Saturn"
    },
    "Moon": { ... },
    "Mars": { ... },
    "Mercury": { ... },
    "Jupiter": { ... },
    "Venus": { ... },
    "Saturn": { ... },
    "Rahu": { ... },
    "Ketu": { ... }
  },
  "houses": {
    "1": { "sign": "Libra", "lord": "Venus", "planets": [] },
    "2": { "sign": "Scorpio", "lord": "Mars", "planets": ["Jupiter"] },
    ...
  },
  "dashas": {
    "balance_at_birth": {
      "planet": "Sun", "years": 1, "months": 9, "days": 22
    },
    "sequence": [
      { "planet": "Sun", "start": "1992-04-28", "end": "1998-04-28" },
      { "planet": "Moon", "start": "1998-04-28", "end": "2008-04-28" },
      ...
    ],
    "current": {
      "mahadasha": "Jupiter",
      "antardasha": "Saturn",
      "mahadasha_start": "2020-06-15",
      "mahadasha_end": "2036-06-15",
      "antardasha_start": "2024-01-12",
      "antardasha_end": "2026-07-06"
    }
  },
  "yogas": [
    {
      "name": "Gaja Kesari Yoga",
      "type": "raj",
      "strength": "strong",
      "description": "Moon and Jupiter in mutual kendras",
      "planets": ["Moon", "Jupiter"],
      "effect": "Wisdom, fame, leadership qualities"
    }
  ],
  "ashtakavarga": {
    "Sun": [4,3,5,2,4,3,6,2,4,3,5,4],
    "Moon": [...],
    ...
    "sarvashtakavarga": [28,24,32,...]
  },
  "numerology": {
    "birth_number": 9,
    "name_number": 11,
    "destiny_number": 7
  }
}
```

---

## Hybrid Search Function
```sql
CREATE OR REPLACE FUNCTION search_report_chunks(
    p_profile_id UUID,
    p_query_embedding vector(1536),
    p_query_text TEXT,
    p_limit INT DEFAULT 10
)
RETURNS TABLE (
    id UUID,
    content TEXT,
    metadata JSONB,
    report_id UUID,
    similarity FLOAT,
    ts_rank FLOAT,
    combined_score FLOAT
)
LANGUAGE plpgsql AS $$
BEGIN
    RETURN QUERY
    SELECT
        rc.id,
        rc.content,
        rc.metadata,
        rc.report_id,
        1 - (rc.embedding <=> p_query_embedding) AS similarity,
        ts_rank(to_tsvector('english', rc.content),
                plainto_tsquery('english', p_query_text)) AS ts_rank,
        (0.7 * (1 - (rc.embedding <=> p_query_embedding))) +
        (0.3 * ts_rank(to_tsvector('english', rc.content),
                plainto_tsquery('english', p_query_text))) AS combined_score
    FROM report_chunks rc
    WHERE rc.profile_id = p_profile_id
    ORDER BY combined_score DESC
    LIMIT p_limit;
END;
$$;
```
