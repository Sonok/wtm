### Big picture

This architecture cleanly separates **user identity and personalization** from the **global event catalog**.  
The two are only combined at request time inside the backend API.

Think of it as:

- **Events DB** → *What exists in the world*
- **User DB** → *Who this person is*
- **FastAPI** → *How to match the two*

This separation is intentional and important.

```
               ┌──────────────────────────────┐
               │          Frontend            │
               │     React (search/feed)      │
               └──────────────┬───────────────┘
                              │ HTTPS
                              ▼
               ┌──────────────────────────────┐
               │     Backend API (FastAPI)     │
               │  - auth/session              │
               │  - recommendation service     │
               │  - search endpoints           │
               └───────┬───────────┬──────────┘
                       │           │
         user/profile  │           │  events/query
                       │           │
                       ▼           ▼
     ┌──────────────────────┐   ┌─────────────────────────┐
     │       User DB        │   │        Events DB         │
     │ (private, sensitive) │   │ (Eventbrite-derived)     │
     │ Postgres (users)     │   │ Postgres + PostGIS +     │
     │                      │   │ pgvector (recommended)   │
     └──────────┬───────────┘   └───────────┬─────────────┘
                │                           │
                │ interactions/logs         │ ETL writes
                ▼                           ▼
     ┌──────────────────────┐   ┌─────────────────────────┐
     │ Interactions Store   │   │  ETL / Ingest Pipeline   │
     │ (can be same PG)     │   │  pulls Eventbrite JSON   │
     │ views/likes/rsvps    │   │  normalizes + embeds     │
     └──────────────────────┘   └─────────────────────────┘
```

```

## Explanation & design rationale

### Why two databases?

**User DB**
- Contains sensitive data (email, password hash, preferences)
- Changes frequently (settings, interactions, embeddings)
- Should be tightly access-controlled

**Events DB**
- Public or semi-public data
- High read volume, lower write volume
- Optimized for search, GIS queries, and vector similarity
- Can be rebuilt or re-ingested without touching user data

This makes the system safer, easier to scale, and easier to reason about.

### Role of the ETL / ingest pipeline

The ETL pipeline is responsible for:
- Pulling raw Eventbrite JSON
- Normalizing inconsistent fields
- Enriching data (tags, popularity, derived fields)
- Generating vector embeddings for semantic search
- Writing clean, query-ready rows into the Events DB

Once data is ingested, **the backend never talks to Eventbrite directly** — it only queries your own database.

### Why PostGIS + pgvector together?

These two dimensions are the core of event discovery:

- **PostGIS** answers: *Is this event physically relevant to the user?*
- **pgvector** answers: *Is this event conceptually relevant to the user?*

Combining them allows:
- “Nearby but boring” events to be filtered out
- “Interesting but far away” events to be deprioritized
- Truly relevant events to rise to the top

### Why interactions are separate

The interactions table acts as a **safe bridge**:
- No event needs to know who the user is
- No user data leaks into the events catalog
- Behavior can be weighted, decayed, or replayed later

This also enables:
- Re-ranking without re-ingesting events
- Offline model training
- A/B testing recommendation logic

### Why the join happens in the API (not the DB)

Keeping the join logic in FastAPI gives you:
- Flexibility to change ranking logic quickly
- Ability to mix SQL, vector math, and business rules
- Easier experimentation without schema changes

Your API becomes the **recommendation engine**, not just a CRUD layer.

### Scalability path (later)

This design naturally evolves into:
- Read replicas for Events DB
- Caching popular geo queries
- Async embedding updates
- Dedicated recommendation workers
- External vector stores if needed (but not required early)

You are starting with a **production-grade architecture** that can grow without rewrites


## What’s inside each DB

### User DB (private)

```

users

* id (uuid)
* email
* password_hash
* home_lat, home_lon  (or home_city_id)
* personality_vector  (pgvector or stored elsewhere)
* preferences_json
* created_at

user_settings (optional)

* user_id
* distance_miles
* categories_allowed
* price_pref
* time_of_day_pref

```

### Events DB (catalog)

```

events

* id (uuid)                 (your internal id)
* source_eventbrite_id      (string)
* title
* description
* category
* start_time, end_time
* venue_name
* venue_lat, venue_lon
* geom (PostGIS geography point)
* city, country
* tags_json
* event_vector (pgvector embedding)
* popularity_score (derived)
* updated_at

```

### Interactions (links the two worlds safely)

```

user_event_interactions

* user_id
* event_id
* action   (view|like|hide|rsvp_click|share)
* weight   (for ranking)
* ts

```

## How a recommendation request works (the “join” happens in your API)

```

User opens feed
│
▼
FastAPI loads user profile (location + personality_vector) from User DB
│
▼
FastAPI queries Events DB:

1. GIS filter: events within radius of user location
2. Time filter: upcoming window
3. Vector rank: similarity(user_vector, event_vector)
4. Business rules: not hidden/seen too much, category prefs, etc.
   │
   ▼
   FastAPI returns ranked list to frontend
```
