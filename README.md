# AureoStays — Villa Booking Platform with Live ETL Pipeline

> A full-stack data engineering project demonstrating a production-grade ETL pipeline where every guest interaction flows into a structured PostgreSQL database, queryable in real time.

**Live Site** → [villa-frontend.vercel.app](https://villa-frontend.vercel.app)  
**Live API** → [villa-api-production-f7b1.up.railway.app/docs](https://villa-api-production-f7b1.up.railway.app/docs)  
**Backend Code** → [github.com/s-keshri/villa-api](https://github.com/s-keshri/villa-api)  
**Frontend Code** → [github.com/s-keshri/villa-frontend](https://github.com/s-keshri/villa-frontend)

---

## What This Is

Most data engineering portfolios show pipelines built on CSV files and Jupyter notebooks. This project demonstrates something real — a live booking platform where:

1. A guest opens the website on their phone or laptop
2. They browse 3 villas, pick dates, enter their details, and confirm a booking
3. That interaction creates clean, structured records across 4 PostgreSQL tables — atomically, with no possibility of double-booking
4. The data is immediately queryable via SQL for revenue, occupancy, and guest analytics

Every piece of this is deployed and running right now. You can make a booking and watch it appear in the database.

---

## Architecture

```
┌──────────────────────────────────────────┐
│           GUEST (Browser / Phone)        │
│       villa-frontend.vercel.app          │
│    Next.js 15 · Tailwind · TypeScript    │
└──────────────┬───────────────────────────┘
               │ HTTPS POST /bookings/
               ▼
┌──────────────────────────────────────────┐
│              BACKEND API                 │
│  villa-api-production-f7b1.up.railway.app│
│         FastAPI · Python 3.13            │
│                                          │
│  1. Validate availability                │
│  2. Upsert guest record                  │
│  3. Write booking (atomic transaction)   │
│  4. Block inventory dates                │
└──────────────┬───────────────────────────┘
               │ asyncpg (raw SQL)
               ▼
┌──────────────────────────────────────────┐
│              DATABASE                    │
│         Supabase · PostgreSQL 15         │
│                                          │
│  properties · inventory · guests ·       │
│  bookings                                │
└──────────────┬───────────────────────────┘
               │ SQL queries
               ▼
┌──────────────────────────────────────────┐
│              ANALYTICS                   │
│               Metabase                   │
│                                          │
│  Revenue · Occupancy · Guest analytics   │
└──────────────────────────────────────────┘
```

---

## The ETL Pipeline

When a guest clicks **Confirm Booking**, a single API call runs this sequence inside one database transaction:

```
Extract   →  Guest inputs: dates, guest count, name, email, phone
Transform →  Validate availability, enforce guest limits,
             generate booking reference, snapshot price
Load      →  INSERT guest record (upsert)
             INSERT booking row
             UPDATE inventory (mark dates unavailable)
             COMMIT — or ROLLBACK if anything fails
```

This atomic transaction pattern means no partial writes, no double bookings, no orphaned records — even under concurrent requests.

---

## Database Design

Four tables. Every design decision has a reason.

| Table | Purpose | Key Design Decision |
|---|---|---|
| `properties` | One row per villa | Amenities stored as `TEXT[]` array |
| `inventory` | One row per property per date | Enables O(1) availability checks |
| `guests` | One row per unique email | `UNIQUE(email)` — repeat guests reuse record |
| `bookings` | Every confirmed booking | Generated columns for nights, total, guests |

**Generated columns** — Postgres computes `num_nights`, `total_guests`, and `total_amount` automatically. Analytics queries never need to recalculate them.

**Price snapshotting** — `price_per_night` is stored at booking time. Revenue reports always reflect what guests actually paid, even if prices change later.

**Separate inventory table** — A naive design would query bookings to check availability. The inventory table makes availability a simple `WHERE is_available = TRUE` lookup and enables maintenance blocks without touching bookings.

---

## Analytics Queries (Metabase)

```sql
-- Revenue per property
SELECT
    p.name,
    COUNT(b.id) AS total_bookings,
    SUM(b.total_amount) AS total_revenue,
    ROUND(AVG(b.total_amount), 2) AS avg_booking_value
FROM bookings b
JOIN properties p ON b.property_id = p.id
WHERE b.status = 'confirmed'
GROUP BY p.name
ORDER BY total_revenue DESC;

-- Occupancy rate (booked days out of 90)
SELECT
    p.name,
    COUNT(*) AS total_days,
    SUM(CASE WHEN i.is_available = FALSE THEN 1 ELSE 0 END) AS booked_days,
    ROUND(
        SUM(CASE WHEN i.is_available = FALSE THEN 1 ELSE 0 END) * 100.0 / COUNT(*), 1
    ) AS occupancy_pct
FROM inventory i
JOIN properties p ON i.property_id = p.id
GROUP BY p.name
ORDER BY occupancy_pct DESC;
```

---

## Tech Stack

| Layer | Technology | Hosting |
|---|---|---|
| Frontend | Next.js 15, Tailwind CSS, TypeScript | Vercel (free) |
| Backend | FastAPI, Python 3.13, Pydantic v2 | Railway |
| Database | PostgreSQL 15, asyncpg | Supabase (free) |
| Analytics | Metabase | Local / Render |

---

## Repositories

| Repo | Description |
|---|---|
| [villa-api](https://github.com/s-keshri/villa-api) | FastAPI backend — ETL logic, database schema, all endpoints |
| [villa-frontend](https://github.com/s-keshri/villa-frontend) | Next.js frontend — listing page, detail page, booking widget |

---

## Try It Yourself

1. Open [villa-frontend.vercel.app](https://villa-frontend.vercel.app)
2. Pick any villa and click on it
3. Select check-in and check-out dates
4. Add guests and fill in your details
5. Click Confirm Booking
6. Your booking reference (e.g. `BK-20260420-0012`) is now live in the database

---

*Built by Shivam Keshri · Data Engineering Portfolio · April 2026*
