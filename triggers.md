# System Triggers & Automation

Overview of automated processes that create and update database rows.

---

## Summary Table

| Trigger Type | What It Does | When It Runs | Database Impact |
|--------------|--------------|--------------|-----------------|
| **Cron: Publish Posts** | Publishes scheduled content | Every 5-15 minutes | Updates `content_allocations.status` |
| **Cron: Reset Quotas** | Resets billing cycle quotas | Daily at midnight | Resets `content_used_this_cycle` to 0 |
| **Cron: Check Grace Periods** | Suspends accounts with failed payments | Daily at midnight | Updates `subscription_status` |
| **Webhook: Payment Success** | Activates/renews subscription | Stripe event | Creates/updates `client_subscriptions` |
| **Webhook: Payment Failure** | Sets grace period | Stripe event | Updates `payment_status`, `grace_period_end_date` |
| **API: Request Content** | Allocates content to client | User action | Creates `content_allocations`, increments quota |
| **API: Retry Failed Post** | Resets failed post to scheduled | User action | Updates `content_allocations.status` |

---

## 1. Cron Jobs (Time-Based)

### Cron A: Publish Scheduled Posts
**Frequency:** Every 5-15 minutes

**Purpose:** Publish posts that have reached their scheduled date/time

**SQL:**
```sql
UPDATE content_allocations
SET status = 'published',
    published_at = NOW()
WHERE status = 'scheduled'
  AND scheduled_date = CURRENT_DATE
  AND scheduled_time <= CURRENT_TIME;
```

**Behavior:**
- Updates `status` from 'scheduled' to 'published'
- Sets `published_at` timestamp
- Publishes content via platform APIs
- Failed publishes marked as 'failed' for retry

---

### Cron B: Reset Billing Cycle Quotas
**Frequency:** Daily at 00:00 UTC

**Purpose:** Reset quota counters for clients whose billing cycle has ended

**Logic:**
```javascript
const expiredCycles = await db.query(`
    SELECT * FROM client_subscriptions
    WHERE current_cycle_end = CURRENT_DATE
      AND subscription_status = 'active'
`);

for (const sub of expiredCycles) {
    const newCycleStart = addDays(sub.current_cycle_end, 1);
    const newCycleEnd = calculateCycleEnd(newCycleStart, sub.billing_cycle);

    await db.query(`
        UPDATE client_subscriptions
        SET current_cycle_start = $1,
            current_cycle_end = $2,
            content_used_this_cycle = 0,
            next_billing_date = $1
        WHERE id = $3
    `, [newCycleStart, newCycleEnd, sub.id]);
}
```

**Behavior:**
- Resets `content_used_this_cycle` to 0
- Calculates new cycle boundaries based on `billing_cycle` type
- Updates `current_cycle_start`, `current_cycle_end`, `next_billing_date`
- Example: Jan 31 cycle end → Feb 1 new cycle start

---

### Cron C: Check Grace Periods & Suspend Accounts
**Frequency:** Daily at 00:00 UTC

**Purpose:** Suspend accounts where grace period expired without payment

**SQL:**
```sql
-- Find accounts with expired grace periods (grace period extends through end of day)
UPDATE client_subscriptions
SET subscription_status = 'suspended'
WHERE grace_period_end_date < CURRENT_DATE
  AND payment_status = 'failed'
  AND subscription_status = 'active';

-- Suspend all future scheduled posts for suspended accounts
UPDATE content_allocations a
SET status = 'suspended'
FROM client_subscriptions s
WHERE a.client_id = s.client_id
  AND s.subscription_status = 'suspended'
  AND a.status = 'scheduled'
  AND a.scheduled_date > CURRENT_DATE;
```

**Behavior:**
- Enforces 3-day grace period expiration
- Sets `subscription_status = 'suspended'` for accounts with `payment_status = 'failed'`
- Suspends future scheduled posts (`scheduled_date > CURRENT_DATE`)
- Already-published posts remain unaffected
- Existing scheduled posts during grace period continue publishing

---

## 2. Webhooks (Event-Based)

### Webhook A: Stripe Payment Success
**Trigger:** `payment_intent.succeeded` or `invoice.payment_succeeded`

**Purpose:** Create/update subscription when payment succeeds

**Logic:**
```javascript
stripe.webhooks.on('payment_intent.succeeded', async (event) => {
    const { customer, amount, metadata } = event.data.object;

    // Check if subscription exists
    const existing = await getSubscription(metadata.client_id);

    if (existing) {
        // Renew existing subscription
        await db.query(`
            UPDATE client_subscriptions
            SET payment_status = 'paid',
                subscription_status = 'active',
                grace_period_end_date = NULL,
                next_billing_date = calculateNextBilling(billing_cycle)
            WHERE client_id = $1
        `, [metadata.client_id]);
    } else {
        // Create new subscription
        await db.query(`
            INSERT INTO client_subscriptions (
                client_id, billing_cycle, content_quota_per_cycle,
                content_used_this_cycle, payment_status, subscription_status
            ) VALUES ($1, $2, $3, 0, 'paid', 'active')
        `, [metadata.client_id, metadata.billing_cycle, metadata.quota]);
    }
});
```

**Behavior:**
- Creates new subscription records
- Renews existing subscriptions
- Reactivates suspended accounts on late payment success
- Clears grace period (`grace_period_end_date = NULL`)
- Sets `payment_status = 'paid'` and `subscription_status = 'active'`

---

### Webhook B: Stripe Payment Failure
**Trigger:** `payment_intent.payment_failed` or `invoice.payment_failed`

**Purpose:** Set grace period and initiate retry logic

**Logic:**
```javascript
stripe.webhooks.on('invoice.payment_failed', async (event) => {
    const { customer, attempt_count } = event.data.object;

    await db.query(`
        UPDATE client_subscriptions
        SET payment_status = 'failed',
            grace_period_end_date = CURRENT_DATE + INTERVAL '3 days'
        WHERE client_id = $1
    `, [customer]);
});
```

**Behavior:**
- Sets 3-day grace period (`grace_period_end_date`)
- Updates `payment_status = 'failed'`
- Keeps `subscription_status = 'active'` (existing posts continue publishing)
- Blocks new content requests via API `payment_status` check
- Stripe automatic retries run independently (see Payment Edge Cases)

---

## 3. User-Triggered Actions (API Calls)

### API A: Request Content Allocation
**Endpoint:** `POST /api/v1/content/request`

**Trigger:** User clicks "Request Content" in UI

**Request:**
```json
{
    "scheduled_date": "2025-01-15",
    "scheduled_time": "09:00:00"
}
```

**Logic:**
```javascript
async function requestContent(clientId, scheduledDate) {
    const sub = await getSubscription(clientId);

    // 1. Check payment status (blocks requests during grace period)
    if (sub.payment_status === 'failed') {
        throw new Error('Payment failed. Update payment method to request content.');
    }

    // 2. Check quota
    if (sub.content_used_this_cycle >= sub.content_quota_per_cycle) {
        throw new Error('Quota exceeded (30/30 used)');
    }

    // 3. Validate date is within billing cycle
    if (scheduledDate < sub.current_cycle_start ||
        scheduledDate > sub.current_cycle_end) {
        throw new Error('Date outside billing cycle');
    }

    // 4. Select content from library
    const content = await selectContentForClient(clientId);

    // 5. Create allocation (no batch ID for single request)
    await db.query(`
        INSERT INTO content_allocations (
            client_id, content_id, scheduled_date, scheduled_time,
            billing_cycle_id, allocation_batch_id, status
        ) VALUES ($1, $2, $3, $4, $5, $6, 'scheduled')
    `, [clientId, content.id, scheduledDate, scheduledTime,
        getCurrentCycleId(clientId), null, 'scheduled']);

    // 6. Increment quota usage
    await db.query(`
        UPDATE client_subscriptions
        SET content_used_this_cycle = content_used_this_cycle + 1
        WHERE client_id = $1
    `, [clientId]);

    return { success: true, remaining: sub.content_quota_per_cycle - sub.content_used_this_cycle - 1 };
}
```

**Database Impact:**
- Creates 1 row in `content_allocations`
- Increments `content_used_this_cycle` by 1

---

### API B: Smart Bulk Allocation
**Endpoint:** `POST /api/v1/content/auto-schedule`

**Trigger:** User requests automatic scheduling with smart date distribution

**Request:**
```json
{
    "count": 5,
    "period": "weekly",           // 'daily', 'weekly', or 'biweekly'
    "start_date": "2025-01-15",
    "end_date": "2025-01-22"
}
```

**Logic:**
```javascript
async function autoScheduleContent(clientId, params) {
    const sub = await getSubscription(clientId);

    // Check payment status + quota
    if (sub.payment_status === 'failed') throw new Error('Payment failed.');
    if (sub.content_used_this_cycle + params.count > sub.content_quota_per_cycle) {
        throw new Error('Not enough quota');
    }

    // Smart date generation based on period
    const dates = generateSmartDates(params);  // Evenly distributes based on period

    // Single batch_id for all allocations
    const batchId = generateUUID();

    for (const date of dates) {
        const content = await selectContentForClient(clientId);
        await createAllocation(clientId, content.id, date, batchId);
    }

    await incrementQuota(clientId, params.count);
}
```

**Database Impact:**
- Creates N rows in `content_allocations` with same `allocation_batch_id`
- Enables smart scheduling patterns (daily, weekly spread)
- Groups all allocations for atomicity/rollback

---

## Trigger Dependencies

```
Payment Success (Webhook)
    ↓
Subscription Created/Updated
    ↓
User Requests Content (API)
    ↓
Allocation Created + Quota Incremented
    ↓
Publish Cron Runs
    ↓
Post Published (or Failed)
    ↓
(If failed) User Retries (API)
    ↓
Next Billing Cycle
    ↓
Reset Quota Cron Runs
    ↓
Quota Reset to 0

--- Failure Path ---
Payment Fails (Webhook)
    ↓
Grace Period Set (3 days)
    ↓
Payment Retries (Stripe automatic)
    ↓
If still fails after 3 days:
    ↓
Grace Period Cron Runs
    ↓
Account Suspended
```

---

## Payment Edge Cases

### Billing Cycle Persistence
The system maintains billing cycle dates on the original schedule regardless of payment timing. When payment fails and succeeds days later, the next billing date remains unchanged to prevent billing date drift and simplify accounting.

**Behavior:**
- Billing cycle dates (`current_cycle_start`, `current_cycle_end`, `next_billing_date`) remain constant
- Payment success during grace period does not shift future billing dates
- Next billing cycle begins on originally scheduled date

**Example:**
```
Billing date: Jan 1
Payment fails: Jan 1 → Grace period active
Payment succeeds: Jan 3 → Subscription reactivated
Next billing: Feb 1 (unchanged from original schedule)
```

---

### Quota Retention During Grace Period
Quota counters persist through payment failures and reactivation. The `content_used_this_cycle` value remains frozen during the grace period and resumes at the same level upon reactivation.

**During Payment Failure:**
- `content_used_this_cycle` remains frozen (not incremented or reset)
- New content requests blocked via `payment_status='failed'` check in API
- Existing scheduled posts continue publishing (does not affect quota tracking)

**Upon Reactivation:**
- Quota counter resumes at frozen value
- Client retains remaining quota until next billing cycle
- Ensures client receives full value for paid billing period

**Example:**
```
Jan 1-10: Client allocates 15/30 quota
Jan 11: Payment fails (quota frozen at 15/30)
Jan 11-13: Grace period (no new allocations, existing posts publish)
Jan 13: Payment succeeds (quota remains 15/30, client has 15 remaining)
Feb 1: Quota resets to 0/30 for new billing cycle
```

---

### Suspended Content Reactivation
Suspended content allocations automatically revert to scheduled status when payment succeeds after suspension. The payment success webhook includes logic to restore content for reactivated accounts.

**Reactivation Logic:**
```javascript
if (previous_status === 'suspended' && new_status === 'active') {
    await db.query(`
        UPDATE content_allocations
        SET status = 'scheduled'
        WHERE client_id = $1
          AND status = 'suspended'
          AND scheduled_date >= CURRENT_DATE
    `, [client_id]);
}
```

**Behavior:**
- Only future content restored (`scheduled_date >= CURRENT_DATE`)
- Past suspended content remains suspended
- Restored content resumes normal publishing schedule

---

### Grace Period and Stripe Retry Coordination
The system implements two independent but complementary mechanisms for payment recovery: a 3-day grace period for user intervention and Stripe's automatic retry schedule for payment recovery.

**Stripe Automatic Retries** (default configuration):
- Attempt 1: Immediate (on initial failure)
- Attempt 2: 3 days after failure
- Attempt 3: 5 days after failure
- Attempt 4: 7 days after failure (final)

**3-Day Grace Period:**
- Prevents immediate account suspension
- Allows existing scheduled content to publish
- Provides time for user to update payment method

**Timeline:**
```
Day 1: Payment fails
    → Stripe retry attempt fails
    → System sets 3-day grace period
    → Existing posts continue publishing

Day 2-3: Grace period active
    → User can update payment method
    → Existing posts publish normally

Day 4: Grace period expires
    → Stripe attempts second retry
    → System suspends account if payment still failed
    → Future posts suspended

Day 5-7: Stripe continues automatic retries
    → Late payment success reactivates suspended account
    → Manual intervention required after 4 failed attempts
```

The grace period provides immediate UX benefit (no instant suspension), while Stripe retries handle automatic payment recovery in the background.

---

## Implementation Notes

### Cron Schedule
```
*/15 * * * *  - Publish posts (every 15 minutes)
0 0 * * *     - Reset quotas (daily midnight)
0 0 * * *     - Check grace periods (daily midnight)
```

### Webhook Security
- Verify Stripe webhook signatures
- Use idempotency keys to prevent duplicate processing
- Log all webhook events for debugging

### Error Handling
- Publish failures: Set status='failed', allow retry
- Payment webhook failures: Log and alert, manual intervention
- Quota enforcement: Validate before allocation, use transactions

### Performance Considerations
- Publish cron: Batch updates, index on (status, scheduled_date, scheduled_time)
- Reset quota cron: Only update subscriptions where current_cycle_end = today
- Webhook processing: Async, queue-based to handle spikes

---

## External Service Reliability

### OpenAI API Failure Handling

The system relies on OpenAI API for content generation. When OpenAI experiences downtime or API failures, the system uses retry logic and fallback strategies to maintain availability.

---

### Retry Logic

**Basic Retry with Exponential Backoff:**
```javascript
async function generateWithRetry(promptData, maxRetries = 3) {
    let delay = 1000;  // Start with 1 second

    for (let attempt = 1; attempt <= maxRetries; attempt++) {
        try {
            return await openai.createCompletion(promptData);
        } catch (error) {
            if (attempt === maxRetries || !isRetryable(error)) {
                throw error;  // Give up, move to fallback
            }
            await sleep(delay);
            delay *= 2;  // Double delay each retry (1s, 2s, 4s)
        }
    }
}

function isRetryable(error) {
    // Retry timeouts and server errors, not client errors
    return error.code === 'ETIMEDOUT' ||
           error.status >= 500 ||
           error.status === 429;  // Rate limit
}
```

---

### Fallback Strategies

When OpenAI API fails after retries, the system uses three fallback mechanisms:

**1. Pre-Generated Content Pool**
- Maintain buffer of pre-generated content in `content_library`
- System maintains minimum pool buffer (100-200 pieces) to prevent exhaustion
- Allocate from existing pool when OpenAI unavailable
- System prioritizes fresh generation but falls back to pool when needed

**2. Template-Based Content**
```javascript
async function generateContent(clientId, contentType) {
    try {
        return await generateWithRetry({ client: clientId, type: contentType });
    } catch (error) {
        // Fallback: Use template engine
        return fillTemplate({
            template: getTemplateForType(contentType),
            variables: getClientPreferences(clientId)
        });
    }
}
```

**3. Graceful Degradation**
- User requests don't fail when OpenAI down
- System allocates available content from library immediately
- No blocking or service interruption for users

---

### Quota Handling for Fallback Content

Fallback content does not count against client quota. Only successfully generated custom content consumes quota credits.

**Rationale:**
- Clients subscribe for AI-generated custom content
- Fallback content is emergency mode due to system limitations
- Fair business practice: Don't charge for system failures

**Implementation:**
```javascript
async function allocateContent(clientId, scheduledDate) {
    try {
        // Attempt custom generation
        const customContent = await generateWithRetry(clientId);
        await createAllocation(clientId, customContent.id, scheduledDate, {
            is_fallback: false
        });
        await incrementQuota(clientId, 1);  // Count quota
    } catch (error) {
        // OpenAI unavailable: use pre-existing fallback content
        const fallbackContent = await getFromPool(clientId);
        await createAllocation(clientId, fallbackContent.id, scheduledDate, {
            is_fallback: true
        });
        // Quota NOT incremented - fallback doesn't count
    }
}
```

**Schema Field:**
- `content_allocations.is_fallback` (BOOLEAN): Tracks fallback allocations that don't count toward quota

**Note:** Content creation and custom generation retry logic are out of scope for this design.

---

### User Experience During Downtime

When OpenAI is unavailable, the system maintains seamless operation using pre-existing content from the library pool.

**Behavior:**
- Content allocation requests succeed immediately
- Users receive content from existing library pool
- No blocking, errors, or service degradation
- System transparently indicates fallback content usage

**API Response:**
```javascript
{
    "content_allocated": true,
    "content_id": "pool-content-123",
    "is_fallback": true,  // Using pre-existing pool content
    "message": "Content allocated from library."
}
```

This approach ensures zero user-facing downtime when external services experience issues.
