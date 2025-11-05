# Exam Questions - Answers

---

## Question 2: How do you handle multiple billing cycle lengths? (Quarterly / monthly)

### How We Handle Different Billing Cycles

The `client_subscriptions` table stores `billing_cycle` (monthly/quarterly/annual) and dynamically calculates cycle boundaries. Each cycle type has its own quota and date range. A daily cron job checks for cycle expiration and resets `content_used_this_cycle` to 0 when a new cycle begins. See [design.md](./design.md) for schema details.

---

### Pseudocode Function

**Purpose:** Generate evenly-distributed dates for content scheduling across a billing cycle.

```javascript
function generateContentDates(billingDate, billingCycle, numPieces) {
    // Input validation
    if (numPieces <= 0) {
        return [];
    }

    // 1. Calculate cycle end date
    let cycleEndDate;

    if (billingCycle === 'monthly') {
        cycleEndDate = addMonths(billingDate, 1).subtract(1, 'day');
    }
    else if (billingCycle === 'quarterly') {
        cycleEndDate = addMonths(billingDate, 3).subtract(1, 'day');
    }
    else if (billingCycle === 'annual') {
        cycleEndDate = addYears(billingDate, 1).subtract(1, 'day');
    }

    // 2. Calculate interval between posts
    const totalDays = daysBetween(billingDate, cycleEndDate);
    const interval = Math.floor(totalDays / numPieces);

    // 3. Generate evenly-spaced dates
    const dates = [];
    let currentDate = billingDate;

    for (let i = 0; i < numPieces; i++) {
        // Ensure we don't exceed cycle end
        if (currentDate > cycleEndDate) {
            currentDate = cycleEndDate;
        }

        dates.push(currentDate);
        currentDate = addDays(currentDate, interval);
    }

    return dates;
}
```

---

### Examples

**Monthly (30 posts):**
```javascript
generateContentDates('2025-01-01', 'monthly', 30)
// Returns: ['2025-01-01', '2025-01-02', '2025-01-03', ..., '2025-01-31']
// Interval: 31 days / 30 posts ≈ 1 post/day
```

**Quarterly (90 posts):**
```javascript
generateContentDates('2025-01-01', 'quarterly', 90)
// Returns: ['2025-01-01', '2025-01-02', ..., '2025-03-31']
// Interval: 90 days / 90 posts ≈ 1 post/day across 3 months
```

**Annual (365 posts):**
```javascript
generateContentDates('2025-01-01', 'annual', 365)
// Returns: ['2025-01-01', '2025-01-02', ..., '2025-12-31']
// Interval: 365 days / 365 posts ≈ 1 post/day for entire year
```

---

### Enhanced Version with User Preferences

**Purpose:** Same as above, but respects user posting preferences (days, times, analytics).

```javascript
function generateContentDates(billingDate, billingCycle, numPieces, options = {}) {
    // Input validation
    if (numPieces <= 0) {
        return [];
    }

    // User preferences (from client_preferences table or analytics)
    const {
        preferredDays = null,      // ['Monday', 'Wednesday', 'Friday']
        avoidWeekends = false,     // Skip Sat/Sun
        timeOfDay = '09:00',       // Best performing time from analytics
        analyticsData = null       // { 'Monday': 0.85, 'Tuesday': 0.72, ... } (engagement scores)
    } = options;

    // Calculate cycle end
    const cycleEndDate = calculateCycleEnd(billingDate, billingCycle);
    const totalDays = daysBetween(billingDate, cycleEndDate);
    const interval = Math.floor(totalDays / numPieces);

    const dates = [];
    let currentDate = billingDate;

    for (let i = 0; i < numPieces; i++) {
        // Ensure we don't exceed cycle end
        if (currentDate > cycleEndDate) {
            currentDate = cycleEndDate;
        }

        // Skip to preferred day if specified
        if (preferredDays && !preferredDays.includes(getDayName(currentDate))) {
            currentDate = getNextDayInList(currentDate, preferredDays);
        }

        // Skip weekends if configured
        while (avoidWeekends && isWeekend(currentDate)) {
            currentDate = addDays(currentDate, 1);
        }

        // Use analytics to pick best day within next interval
        if (analyticsData) {
            currentDate = findBestDayInRange(currentDate, interval, analyticsData);
        }

        dates.push({
            date: currentDate,
            time: timeOfDay  // Apply time preference
        });

        currentDate = addDays(currentDate, interval);
    }

    return dates;
}
```

**Example with Preferences:**
```javascript
// Client preferences: only post Mon/Wed/Fri at 6 PM (best engagement time)
generateContentDates('2025-01-01', 'monthly', 12, {
    preferredDays: ['Monday', 'Wednesday', 'Friday'],
    timeOfDay: '18:00',
    avoidWeekends: true
})

// Returns:
[
  { date: '2025-01-03', time: '18:00' },  // Friday
  { date: '2025-01-06', time: '18:00' },  // Monday
  { date: '2025-01-08', time: '18:00' },  // Wednesday
  { date: '2025-01-10', time: '18:00' },  // Friday
  ...
]
// Only posts on Mon/Wed/Fri, all at 6 PM
```

**Where Preferences Come From (Possibly):**
- Stored in `client_preferences` table (preferred days, times)
- Derived from `analytics_data` table (best performing days/times from past posts)
- Can be overridden per API request

---

### Usage in System

This function powers the Smart Bulk Allocation API (see [triggers.md](./triggers.md) - API B), allowing users to request multiple posts with automatic date distribution based on their billing cycle.
