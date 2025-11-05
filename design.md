# Content Calendar Management System Design
yes,
## Question 1: How would you design a system that manages the user content that populates our calendar each month?

---

## Problem Statement

Design a database schema and system to track which clients receive which pieces of content on which dates, based on their subscription plan and billing cycle.

**Context:**
- Clients subscribe to receive X amount of content per billing cycle (monthly/quarterly/annual)
- Content (posts/videos) is allocated to their calendar and published on scheduled dates
- System must track allocations across different billing cycles
- Focus: Database design for allocation tracking, not content creation

---

## Core Assumptions

1. **Content Quality & Selection:** Content is pre-curated and matches client needs. Selection algorithm (details out of scope) considers user preferences, template variety to prevent repetition, content type mix, and analytics data to ensure month-over-month freshness
2. **Content Creation:** Out of scope - content already exists in library
3. **Credit-Based Quota:** Clients receive a quota (credit balance) per billing cycle and request content on-demand, using one credit per piece of content; distributing the same content to multiple platforms does not consume additional credits; fallback content (used when OpenAI unavailable) does not count toward quota
4. **Content Reuse:** Content may be allocated to multiple clients; unique constraint prevents same client receiving duplicate content within a billing cycle
5. **Quota Scope:** Quota applies to unique content pieces allocated by the system; distributing content across multiple platforms counts as one allocation; client's manual posts (if supported) do not count toward quota
6. **One-Time Publication:** Content is allocated and published once; reposting old content can be a possibility in the future
7. **Automated Publishing:** Posts publish automatically on scheduled dates
8. **Minimal User Modification:** This design focuses on allocation tracking; editing/rejection/enhancement workflows are possible future enhancements

---

## System Architecture

### High-Level Flow

```
Client Pays Subscription
    ↓
Client receives quota (credit balance) for billing cycle
    ↓
Client requests content from library (uses 1 credit per request)
    ↓
System allocates content and schedules on client-specified date
    ↓
Automated publishing on scheduled dates
    ↓
On next billing date: reset quota, start new cycle
```

### Example: Monthly Subscription (30 posts quota)

```
Jan 1: Client pays → Quota: 0/30 used
    ↓
Jan 5: Client requests content for Jan 10 → Quota: 1/30 used
Jan 8: Client requests content for Jan 15 → Quota: 2/30 used
Jan 12: Client requests content for Jan 20 → Quota: 3/30 used
... (continues throughout month)
Jan 28: Client requests 30th piece → Quota: 30/30 used (limit reached)
    ↓
Each day: cron job publishes posts where scheduled_date = today
    ↓
Feb 1: Next billing → Quota resets to 0/30, cycle repeats
```

---

## Database Schema

### Table 1: content_library

**Purpose:** Minimal reference to content pieces (creation details out of scope)

```sql
CREATE TABLE content_library (
    id UUID PRIMARY KEY,
    content_type VARCHAR(20) NOT NULL,     -- 'static_post' or 'video'
    template_id VARCHAR(50),               -- 'carousel', 'quote', 'infographic', 'story', etc.
    visual_style VARCHAR(50),              -- 'modern', 'minimalist', 'bold', 'playful', etc.
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_content_type ON content_library(content_type);
CREATE INDEX idx_template ON content_library(template_id);
```

**Notes:**
- Intentionally minimal - don't need caption, media_url, etc.
- Just enough to have something to reference in allocations
- Content creation system is separate concern
- Contains both AI-generated custom content and pre-curated fallback content pool (used when OpenAI unavailable)
- Template and visual style metadata enable variety tracking to prevent repetitive content across billing cycles
- Content selection algorithm (out of scope) uses these fields to ensure diverse allocation: template rotation, content type mix, alignment with user preferences and analytics

---

### Table 2: client_subscriptions

**Purpose:** Track subscription plans, billing cycles, and payment status

```sql
CREATE TABLE client_subscriptions (
    id UUID PRIMARY KEY,
    client_id UUID NOT NULL,

    -- Plan details
    billing_cycle VARCHAR(20) NOT NULL,    -- 'monthly', 'quarterly', 'annual'
    content_quota_per_cycle INTEGER NOT NULL,
    content_used_this_cycle INTEGER DEFAULT 0,

    -- Current billing cycle
    current_cycle_start DATE NOT NULL,
    current_cycle_end DATE NOT NULL,
    next_billing_date DATE NOT NULL,

    -- Status tracking
    subscription_status VARCHAR(20) NOT NULL,  -- 'active', 'suspended', 'cancelled'
    payment_status VARCHAR(20) NOT NULL,       -- 'paid', 'pending', 'failed'
    grace_period_end_date DATE,                -- When to suspend if payment still failing

    -- Timestamps
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_client_subscription ON client_subscriptions(client_id);
CREATE INDEX idx_next_billing ON client_subscriptions(next_billing_date)
    WHERE subscription_status = 'active';
```

**Key Fields:**
- `billing_cycle`: Determines how often content is allocated
- `content_quota_per_cycle`: How many pieces client gets per cycle (e.g., 30, 90, 365)
- `content_used_this_cycle`: Tracks allocations made this cycle (resets on new billing cycle)
- `current_cycle_start/end`: Boundaries for current allocation period
- `payment_status`: Used for payment failure handling
- `grace_period_end_date`: Deadline for payment before suspending content (typically 3 days)

---

### Table 3: content_allocations ⭐ **THE KEY TABLE**

**Purpose:** Track which content was allocated to which client on which date

```sql
CREATE TABLE content_allocations (
    id UUID PRIMARY KEY,
    client_id UUID NOT NULL,
    content_id UUID NOT NULL REFERENCES content_library(id),

    -- Scheduling
    scheduled_date DATE NOT NULL,
    scheduled_time TIME NOT NULL DEFAULT '09:00:00',

    -- Billing tracking
    billing_cycle_id VARCHAR(50) NOT NULL,  -- e.g., 'client-123-2025-01'
    allocation_batch_id UUID,               -- Groups content allocated together
    allocated_at TIMESTAMP DEFAULT NOW(),

    -- Publishing status
    status VARCHAR(20) NOT NULL DEFAULT 'scheduled',  -- 'scheduled', 'published', 'suspended', 'failed'
    published_at TIMESTAMP,
    is_fallback BOOLEAN DEFAULT false,              -- True if using placeholder content (OpenAI failure)

    -- Platform targeting
    platform VARCHAR(20),                           -- 'instagram', 'facebook', 'twitter', 'linkedin'
    platform_account_id VARCHAR(100),               -- Client's linked account for that platform

    -- Metadata
    created_at TIMESTAMP DEFAULT NOW()
);

-- Fast calendar queries
CREATE INDEX idx_client_calendar ON content_allocations(client_id, scheduled_date)
    WHERE status IN ('scheduled', 'published');

-- Track allocations by billing cycle
CREATE INDEX idx_billing_cycle ON content_allocations(client_id, billing_cycle_id);

-- Ensure content isn't double-allocated to same client in same cycle
CREATE UNIQUE INDEX idx_unique_allocation
    ON content_allocations(client_id, content_id, billing_cycle_id);
```

**Key Fields:**
- `content_id`: **WHAT** content was allocated
- `client_id`: **WHO** received it
- `scheduled_date`: **WHEN** it should publish
- `billing_cycle_id`: **WHICH** billing cycle it belongs to (enables tracking/analytics)
- `allocation_batch_id`: Groups multiple simultaneous requests (if supported, e.g., "give me 5 posts for next week") and supports rollback if request fails midway
- `is_fallback`: Tracks placeholder allocations when OpenAI unavailable; only non-fallback content counts toward quota
- `platform`: **WHERE** to publish (Instagram, Facebook, Twitter, LinkedIn)
- `platform_account_id`: Client's linked social media account for that platform

**Note:** Multiple allocations of the same `content_id` to different platforms count as one piece of content for quota purposes (one credit per content piece, not per platform distribution).

**This table directly answers the question:**
> "How we create the database rows necessary to keep track of what clients are getting what pieces of content on what dates"

---

## How the System Works

### 1. Client Subscribes

```sql
INSERT INTO client_subscriptions (
    client_id,
    billing_cycle,
    content_quota_per_cycle,
    content_used_this_cycle,
    current_cycle_start,
    current_cycle_end,
    subscription_status,
    payment_status
) VALUES (
    'client-123',
    'monthly',
    30,
    0,  -- Starts with 0 used, 30 available
    '2025-01-01',
    '2025-01-31',
    'active',
    'paid'
);
```

### 2. Client Requests Content

When client requests content for a specific date:
1. System checks quota availability
2. Selects content from library (selection algorithm considers user preferences, template variety, content type mix, and analytics to ensure diverse, fresh allocations)
3. Creates allocation record
4. Increments quota usage counter

```sql
-- Use transaction with row lock to prevent quota race conditions
BEGIN;

-- Check quota with lock
SELECT content_used_this_cycle, content_quota_per_cycle
FROM client_subscriptions
WHERE client_id = 'client-123'
FOR UPDATE;  -- Lock row to prevent concurrent quota checks
-- Returns: used=5, quota=30 → OK, has 25 remaining

-- Allocate content (may create multiple rows if distributing to multiple platforms)
INSERT INTO content_allocations (client_id, content_id, scheduled_date, billing_cycle_id, platform)
VALUES ('client-123', 'content-042', '2025-01-15', 'client-123-2025-01', 'instagram');

-- Increment usage (once per content piece, not per platform)
UPDATE client_subscriptions
SET content_used_this_cycle = content_used_this_cycle + 1
WHERE client_id = 'client-123';
-- Now: used=6, quota=30

COMMIT;
```

### 3. Load Calendar for Client

```sql
-- Get all scheduled/published content for January 2025
SELECT
    a.id,
    a.scheduled_date,
    a.scheduled_time,
    a.status,
    c.content_type
FROM content_allocations a
JOIN content_library c ON a.content_id = c.id
WHERE a.client_id = 'client-123'
  AND a.scheduled_date BETWEEN '2025-01-01' AND '2025-01-31'
  AND a.status IN ('scheduled', 'published')
ORDER BY a.scheduled_date;
```

### 4. Publish Content (Cron Job)

```sql
-- Daily cron job at scheduled times
UPDATE content_allocations
SET status = 'published',
    published_at = NOW()
WHERE status = 'scheduled'
  AND scheduled_date = CURRENT_DATE
  AND scheduled_time <= CURRENT_TIME;
```

### 5. Next Billing Cycle

```sql
-- On Feb 1, reset cycle and allocate next month's content
UPDATE client_subscriptions
SET current_cycle_start = '2025-02-01',
    current_cycle_end = '2025-02-28',
    content_used_this_cycle = 0;  -- Reset usage counter

-- Generate new rows with billing_cycle_id = 'client-123-2025-02'
-- Dates: 2025-02-01 through 2025-02-28
```

> **Automation:** See [triggers.md](./triggers.md) for details on cron jobs, webhooks, and API triggers that create/update these database rows

---

## Key Design Decisions

### Decision 1: Separate Allocation Table

**Why not just store content_id in a single table?**
- ✅ Allocation is a distinct event (who got what when)
- ✅ Enables tracking across billing cycles
- ✅ Separates "content exists" from "content assigned to client"
- ✅ Allows content to be reused across multiple clients

### Decision 2: billing_cycle_id Field

**Why track which cycle allocations belong to?**
- ✅ Analytics: "How much content did client get in Q1?"
- ✅ Payment failures: Suspend content from future cycles
- ✅ Prevents double-allocation in same cycle
- ✅ Clear boundaries for quota enforcement

**Format:** `{client_id}-{year}-{cycle_number}`
- Monthly: `client-123-2025-01`, `client-123-2025-02`
- Quarterly: `client-123-2025-Q1`, `client-123-2025-Q2`
- Annual: `client-123-2025`

**Note:** Billing cycles are staggered per client based on subscription start date (not all clients reset on same day), preventing thundering herd at quota reset

### Decision 3: Minimal Content Library

**Why not include caption, media_url, etc.?**
- ✅ Question explicitly says "don't worry about how we create content"
- ✅ Focus on allocation tracking, not content details
- ✅ Content creation is separate system concern
- ✅ Keeps schema focused and simple

### Decision 4: Credit-Based Quota System

**When does allocation happen?**
- ✅ On-demand when client requests content (not upfront)
- ✅ Each request uses 1 credit from quota
- ✅ Quota tracked with `content_used_this_cycle` counter
- ✅ Flexible, supports rejection/replacement, user control

---

## Example Data Flow

### Scenario: Client with Monthly Plan (30 posts quota)

**Step 1: Payment succeeds Jan 1**
```sql
client_subscriptions:
    client_id: client-123
    billing_cycle: 'monthly'
    content_quota_per_cycle: 30
    content_used_this_cycle: 0  -- Fresh quota, nothing used yet
    cycle_start: 2025-01-01
    cycle_end: 2025-01-31
```

**Step 2: Client requests content throughout January**
```sql
-- Jan 5: Client requests first post
content_allocations:
    Row 1: client-123, content-042, 2025-01-10, status='scheduled'

client_subscriptions:
    content_used_this_cycle: 1  -- 1/30 used

-- Jan 8: Client requests second post
content_allocations:
    Row 2: client-123, content-107, 2025-01-15, status='scheduled'

client_subscriptions:
    content_used_this_cycle: 2  -- 2/30 used

-- ... (continues throughout month)

-- Jan 28: Client requests 30th post
content_allocations:
    Row 30: client-123, content-255, 2025-01-31, status='scheduled'

client_subscriptions:
    content_used_this_cycle: 30  -- 30/30 used (quota exhausted)

-- Jan 29: Client tries to request 31st post
-- System rejects: "Quota exceeded (30/30 used)"
```

**Step 3: Publishing throughout January**
```
Jan 10 09:00 → Post 1 publishes (status → 'published')
Jan 15 09:00 → Post 2 publishes
Jan 31 09:00 → Post 30 publishes
```

**Step 4: Next billing cycle Feb 1**
```sql
-- Reset quota for new cycle
UPDATE client_subscriptions
SET current_cycle_start = '2025-02-01',
    current_cycle_end = '2025-02-28',
    content_used_this_cycle = 0;  -- Reset to 0/30

-- Client can now request 30 new pieces for February
```

---

## What This Design Enables

✅ **Track allocations:** Query "what content did client X get in January?"
```sql
SELECT COUNT(*) FROM content_allocations
WHERE client_id = 'client-123' AND billing_cycle_id = 'client-123-2025-01';
```

✅ **Load calendar:** Show all posts for date range
```sql
SELECT * FROM content_allocations
WHERE client_id = 'client-123'
  AND scheduled_date BETWEEN '2025-01-01' AND '2025-01-31';
```

✅ **Analytics:** Content usage per client
```sql
SELECT client_id, COUNT(*) as total_allocated, COUNT(published_at) as total_published
FROM content_allocations
GROUP BY client_id;
```

✅ **Billing cycle management:** Separate allocations by cycle
```sql
SELECT billing_cycle_id, COUNT(*) FROM content_allocations
WHERE client_id = 'client-123'
GROUP BY billing_cycle_id;
```

**UX Features Enabled:**
✅ **Quota visibility:** Real-time credit tracking (15/30 remaining) via `content_used_this_cycle`
✅ **Post status:** See if content is scheduled/published/failed via `status` field
✅ **Bulk requests:** Allocate multiple pieces at once via `allocation_batch_id`
✅ **No duplicates:** Unique constraint prevents same content twice per cycle
✅ **Usage history:** Track content usage across billing cycles via `billing_cycle_id`
✅ **Retry failures:** Identify and republish failed posts via `status = 'failed'`
✅ **Flexible upgrades:** Update `content_quota_per_cycle` mid-cycle as business grows

---

## Intentionally Out of Scope (Future Features)

The following features are explicitly excluded to keep design focused:

❌ **Content creation/editing UI** - Question states "don't worry about how we create content"
❌ **User-generated content** - This is content-as-a-service model
❌ **Content rejection/replacement** - Assuming high content quality
❌ **Approval workflows** - Automated publishing model
❌ **Content personalization** - Handled during onboarding/selection phase
❌ **Subscription plan normalization** - Plans stored inline in `client_subscriptions`; future enhancement could normalize to `subscription_plans` table for easier bulk plan updates
❌ **Quota refunds for failed publishes** - Failed publications marked for retry but do not refund quota; design prioritizes simplicity over edge-case optimization

**Rationale:** Focus on core allocation tracking and billing cycle management as requested.

---

## Trade-offs

| Decision | Pro | Con | Mitigation |
|----------|-----|-----|------------|
| Separate allocation table | Clean separation, enables analytics | Slightly more complex queries | Well-indexed, minimal joins |
| Minimal content table | Focused, simple schema | Can't query content details | Add fields as needed in V2 |
| Credit-based quota | Flexible, user control, supports rejection | Requires quota enforcement logic | Simple counter increment/check |
| Incremental allocation | User chooses dates, can pace requests | No instant calendar preview | Can suggest dates via algorithm |
| billing_cycle_id as string | Human-readable, flexible | Not normalized | Sufficient for tracking needs |

---

## Additional Resources

- **[generate-content-dates.md](./generate-content-dates.md)** - Question 2: Pseudocode for date distribution across billing cycles
- **[triggers.md](./triggers.md)** - Automation details (cron jobs, webhooks, API triggers)
- **[Entity Relationship Diagram](./assets/erd.png)** - Visual diagram of table relationships