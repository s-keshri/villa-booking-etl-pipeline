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
4. The guest instantly receives a **branded HTML confirmation email** with full booking details
5. The data is immediately queryable via SQL for revenue, occupancy, and guest analytics

Every piece of this is deployed and running right now. You can make a booking and watch it appear in the database — and in your inbox.

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
│  5. Send confirmation email via Resend   │
└──────────┬─────────────────┬─────────────┘
           │ asyncpg (SQL)   │ Resend API
           ▼                 ▼
┌──────────────────┐  ┌─────────────────────┐
│    DATABASE      │  │   GUEST'S INBOX     │
│ Supabase · PG15  │  │  HTML confirmation  │
│                  │  │  email with booking │
│ properties       │  │  reference, dates,  │
│ inventory        │  │  property details   │
│ guests           │  │  & total amount     │
│ bookings         │  └─────────────────────┘
└──────────┬───────┘
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

When a guest clicks **Confirm Booking**, a single API call runs this sequence:

```
Extract   →  Guest inputs: dates, guest count, name, email, phone

Transform →  Validate availability (inventory table)
             Enforce guest count limits
             Generate booking reference (BK-YYYYMMDD-XXXX)
             Snapshot price at time of booking

Load      →  INSERT guest record (upsert on email)        ┐
             INSERT booking row with generated columns     │ Single atomic
             UPDATE inventory → mark dates unavailable     │ transaction
             COMMIT — or ROLLBACK if anything fails        ┘

Notify    →  Send branded HTML confirmation email via Resend
             (fires after commit — failure never affects booking)
```

This atomic transaction pattern means no partial writes, no double bookings, no orphaned records — even under concurrent requests.

---

## Confirmation Email

Every confirmed booking triggers an automated HTML email to the guest containing:

- AureoStays branding with dark/amber theme
- Unique booking reference in a highlighted box
- Full booking details — property, location, dates, duration, guests, special requests
- Total amount in Indian Rupees
- Check-in instructions note

Built with **Resend** — fires automatically after every successful booking transaction.

> Note: Currently using Resend's sandbox mode which delivers to the registered address only. Production deployment would use a verified custom domain to send to all guests.

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

**Non-blocking email** — The confirmation email fires after the transaction commits. If Resend is down, the booking still succeeds — the guest's data is safe. Email is a notification layer, not part of the core transaction.

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
| Email | Resend | Resend (3,000/month free) |
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
7. Check your inbox — a confirmation email arrives within seconds

---

## What's Next

- Verify a custom domain on Resend to send confirmation emails to all guests
- Deploy Metabase to Render for a publicly shareable analytics dashboard
- Add a `/dashboard` page to the frontend showing live booking analytics
- Migrate API from Railway to Render when trial ends (free forever)
- Add more properties and real villa photos

---

*Built by Shivam Keshri · Data Engineering Portfolio · April 2026*
