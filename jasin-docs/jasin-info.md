---
description: 
globs: 
alwaysApply: true
---
# Analytics Tracking System

## Overview
The analytics system tracks views and clicks on embedded product cards using a transactional database function. Two API endpoints handle these events: 
- `/api/analytics/track` for tracking views
- `/api/click/[id]` for tracking clicks and redirecting to Amazon

## Important Function Change
As of May 2025, we transitioned from `track_embed_event_with_transaction` to `track_embed_event_with_transaction_v2`.

### Key differences:
- Parameter renaming: `p_referrer` → `p_referer_url` 
- Removed `p_ip` parameter (no longer used)
- Added parameters: 
  - `p_bot_type` (for detailed bot categorization)
  - `p_request_id` (for cross-request tracing)

## Implementation

Both tracking endpoints ([track/route.ts](mdc:src/app/api/analytics/track/route.ts) and [click/[id]/route.ts](mdc:src/app/api/click/[id]/route.ts)) must use `track_embed_event_with_transaction_v2` with this parameter structure:

```typescript
await supabase.rpc('track_embed_event_with_transaction_v2', {
  p_embed_id: embedId,   // UUID of the embed
  p_event_type: 'view',  // 'view' or 'click'
  p_referer_url: referer, // Referring URL
  p_user_agent: userAgent, // Browser user agent
  p_is_bot: botDetected,  // Boolean flag
  p_bot_type: botType,    // String or null
  p_request_id: requestId // Tracking ID
});
```

## Database Tables
Analytics events update multiple tables in a single transaction:
- `embeds` - Increments views/clicks counters
- `embed_views`/`embed_clicks` - Detailed event records
- `embed_stats` - Aggregated statistics per embed
- `daily_stats` - Time-series data for charts

## Troubleshooting
If analytics tracking errors appear in logs with "function not found" messages, verify:
1. The function name is correct (`track_embed_event_with_transaction_v2`)
2. All parameters are provided with correct names
3. Parameter types match expected values (especially UUIDs and booleans)

## Tracking Flow
1. **View Tracking**
   - Embedded cards use Beacon API to send view events to [/api/analytics/track](mdc:src/app/api/analytics/track/route.ts)
   - Direct views of embed pages also trigger view tracking in [/api/embed/[id]](mdc:src/app/api/embed/[id]/route.ts)

2. **Click Tracking**
   - All product clicks route through [/api/click/[id]](mdc:src/app/api/click/[id]/route.ts) 
   - After tracking, users are redirected to Amazon with the appropriate affiliate tag

3. **Database Operations**
   - All tracking operations use the transactional function `track_embed_event_with_transaction_v2`
   - The system updates multiple tables in a single transaction:
     - Increments counters in the `embeds` table
     - Inserts detailed records in `embed_views` or `embed_clicks`
     - Updates aggregated stats in `embed_stats` and `daily_stats`

4. **Bot Detection**
   - All tracking endpoints filter out bot traffic
   - Bot views/clicks are logged but don't increment main counters

5. **Dashboard Display**
   - Analytics are displayed in [AnalyticsClient](mdc:src/app/analytics/AnalyticsClient.tsx)
   - Shows total views, clicks, and CTR across all embeds
   - Lists top-performing products sorted by views

## Tracking Integration
The [embed.js](mdc:public/embed.js) script automatically tracks views when embedded on external sites. The tracking code uses the Beacon API when available, with a fetch fallback for older browsers.

## Important Implementation Note
The source of truth for analytics data should be the `embed_views` and `embed_clicks` tables, which contain detailed records of each view and click event. The counters in the `embeds` table are convenience fields but can sometimes get out of sync with the actual event records.

When building analytics features:
1. Use the `get_user_analytics` function which counts from the detailed event records
2. The [AnalyticsHealthDashboard](mdc:src/app/admin/analytics/AnalyticsHealthDashboard.tsx) shows discrepancies between counters and actual records
3. The `fix_analytics_discrepancies` function can be used to synchronize counters with the actual event records

For accurate analytics, always prioritize data from the event records tables over the counter fields.



# Analytics Tracking System

## Overview
The analytics system tracks views and clicks on embedded product cards. Data is collected when users interact with embedded product cards across the web and displayed in the analytics dashboard.

## Tracking Flow
1. **View Tracking**
   - Embedded cards use Beacon API to send view events to [/api/analytics/track](mdc:src/app/api/analytics/track/route.ts)
   - Direct views of embed pages also trigger view tracking in [/api/embed/[id]](mdc:src/app/api/embed/[id]/route.ts)

2. **Click Tracking**
   - All product clicks route through [/api/click/[id]](mdc:src/app/api/click/[id]/route.ts) 
   - After tracking, users are redirected to Amazon with the appropriate affiliate tag

3. **Database Operations**
   - All tracking operations use the transactional function [track_embed_event_with_transaction](mdc:supabase/migrations/20240805_improve_analytics_transactions.sql)
   - The system updates multiple tables in a single transaction:
     - Increments counters in the `embeds` table
     - Inserts detailed records in `embed_views` or `embed_clicks`
     - Updates aggregated stats in `embed_stats` and `daily_stats`

4. **Bot Detection**
   - All tracking endpoints filter out bot traffic
   - Bot views/clicks are logged but don't increment main counters

5. **Dashboard Display**
   - Analytics are displayed in [AnalyticsClient](mdc:src/app/analytics/AnalyticsClient.tsx)
   - Shows total views, clicks, and CTR across all embeds
   - Lists top-performing products sorted by views

## Database Tables
- `embeds`: Main product embeds with views/clicks counters
- `embed_views`: Detailed view events with referrer data
- `embed_clicks`: Detailed click events with referrer data
- `embed_stats`: Aggregated statistics per embed
- `daily_stats`: Time-series data for charting

## Tracking Integration
The [embed.js](mdc:public/embed.js) script automatically tracks views when embedded on external sites. The tracking code uses the Beacon API when available, with a fetch fallback for older browsers.


# Anti-Abuse System Architecture

## Overview
The anti-abuse system implements a multi-layered defense against credit farming and account abuse through silent backend monitoring with zero UX impact.

## Core Components

### 1. Database Schema
The system uses four main tables in [supabase/migrations/20250115000000_add_anti_abuse_system.sql](mdc:supabase/migrations/20250115000000_add_anti_abuse_system.sql):

- **`device_fingerprints`**: Tracks browser characteristics, IP addresses, and device signatures
- **`email_domain_reputation`**: Scores email domains (0-100) and blocks disposable services
- **`abuse_signals`**: Records suspicious activities with severity levels (low/medium/high/critical)
- **`credit_usage_patterns`**: Monitors credit consumption patterns and domain usage

### 2. Core Library
[src/lib/anti-abuse.ts](mdc:src/lib/anti-abuse.ts) provides the main utilities:

```typescript
// Risk assessment for new signups
assessSignupRisk(email: string, deviceFingerprint: DeviceFingerprint)

// Device fingerprint extraction from requests
extractDeviceFingerprint(request: Request)

// Credit usage pattern analysis
analyzeCreditUsage(userId: string, creditsUsed: number, usageDomains: string[])

// User blocking checks
checkUserBlocked(userId: string)
```

## Integration Points

### Signup Flow Protection
[src/app/api/auth/signup/route.ts](mdc:src/app/api/auth/signup/route.ts) implements:
- Email domain reputation checking
- Device fingerprinting
- Rapid signup pattern detection
- Risk score calculation (blocks at 90+, flags at 60+)

### PAAPI Usage Monitoring
[src/app/api/amazon/paapi/route.ts](mdc:src/app/api/amazon/paapi/route.ts) includes:
- User blocking checks before API calls
- Usage pattern analysis
- Domain tracking for embed usage
- Suspicious activity logging

### Admin Dashboard
[src/app/admin/anti-abuse/page.tsx](mdc:src/app/admin/anti-abuse/page.tsx) provides:
- Real-time abuse statistics
- Recent abuse signals monitoring
- Device fingerprint duplicate detection
- Blocked domain management

## Risk Assessment Logic

### Email Reputation Scoring
- **Blocked domains** (score 0): Automatic signup block
- **Disposable emails**: +40 risk points
- **Low reputation** (<30): +20 risk points
- **Trusted domains** (90+): Gmail, Outlook, ProtonMail, etc.

### Device Fingerprinting
- Combines: User-Agent, Language, Screen Resolution, Timezone, IP
- Detects: Multiple accounts from same device
- Triggers: Medium severity signal when >2 accounts share fingerprint

### Usage Pattern Detection
- **Rapid credit burn**: <1 hour to use 3+ credits (+40 suspicious score)
- **Test domains**: localhost, example.com, 127.0.0.1 (+30 suspicious score)
- **Critical threshold**: 90+ suspicious score triggers blocking

## Security Principles

### Silent Operation
- All protection happens in background
- No UX changes for legitimate users
- Generic error messages to avoid revealing detection methods

### Fail-Safe Design
- System fails open on errors (doesn't block legitimate users)
- Comprehensive logging for debugging
- Graceful degradation when services unavailable

### Privacy Protection
- User IDs logged as partial hashes (first 8 chars + ***)
- Email addresses logged as username@***
- IP addresses stored for pattern detection only

## Implementation Guidelines

### Adding New Abuse Signals
```typescript
await recordAbuseSignal(userId, {
  signal_type: 'your_signal_type',
  severity: 'low' | 'medium' | 'high' | 'critical',
  details: { /* relevant context */ },
  ip_address: deviceFingerprint.ip_address,
  user_agent: deviceFingerprint.user_agent
});
```

### Risk Assessment Integration
```typescript
// Extract device info
const deviceFingerprint = extractDeviceFingerprint(request);

// Perform risk assessment
const riskAssessment = await assessSignupRisk(email, deviceFingerprint);

// Act on results
if (riskAssessment.should_block) {
  // Block with generic error
}
if (riskAssessment.should_flag) {
  // Record for monitoring
}
```

### Usage Pattern Monitoring
```typescript
// Analyze patterns during credit usage
const usageAnalysis = await analyzeCreditUsage(
  userId, 
  creditsUsed, 
  usageDomains
);

// Log high-risk activity
if (usageAnalysis.suspicious_score >= 80) {
  logger.warn('High suspicious score detected', context);
}
```

## Database Functions

### Core Functions Available
- `extract_device_fingerprint()`: Creates SHA256 hash of device characteristics
- `check_email_domain_reputation()`: Returns domain reputation data
- `detect_signup_patterns()`: Analyzes IP and fingerprint patterns
- `analyze_credit_usage_pattern()`: Scores usage behavior
- `record_abuse_signal()`: Logs abuse events

### RLS Security
All anti-abuse tables use Row Level Security with admin-only access:
```sql
CREATE POLICY "Admin access only" ON table_name
  FOR ALL TO authenticated
  USING (false); -- Only service role can access
```

## Monitoring and Alerts

### Key Metrics to Watch
- Critical signals in last 24h
- Blocked domain growth rate
- Device fingerprint reuse patterns
- Rapid signup attempts from same IP

### Admin Dashboard Access
- Restricted to admin emails: `matt@getjasin.com`, `admin@getjasin.com`
- Real-time statistics and signal monitoring
- Device fingerprint duplicate detection
- Blocked domain management

## Phase Implementation

### Phase 1 (Current): Silent Backend Protection
✅ Device fingerprinting
✅ Email domain blocking
✅ Usage pattern monitoring
✅ Admin dashboard

### Phase 2 (Future): Email Verification
- Require email confirmation before credit allocation
- Enhanced domain verification
- Progressive trust building

### Phase 3 (Future): Progressive Delays
- Implement delays for suspicious activity
- Rate limiting based on risk scores
- Enhanced user behavior analysis

## Best Practices

### Error Handling
- Always fail open for legitimate users
- Log errors comprehensively for debugging
- Use generic error messages for security

### Performance
- Async operations where possible
- Efficient database queries with proper indexing
- Minimal impact on user-facing operations

### Privacy
- Hash sensitive data before logging
- Minimize data retention periods
- Clear data retention policies

### Testing
- Test with various device configurations
- Verify legitimate users aren't blocked
- Monitor false positive rates




# 🧱 Architecture Overview (Jasin MVP)

## 🧰 Stack Overview

| Layer       | Tech                              |
|-------------|-----------------------------------|
| Frontend    | Next.js 14 + Tailwind + shadcn/ui |
| UI Logic    | React Server Components + Hooks   |
| Auth        | Supabase Email + Password         |
| Backend     | Next.js API Routes (Vercel)       |
| DB + Cache  | Supabase (Postgres)               |
| Product API | Amazon PAAPI 5.0                  |

---

## 📡 Data Flow

1. User pastes input (Amazon URL or ASIN)
2. App extracts ASIN + affiliate tag (if present)
3. Check Supabase `products` cache
4. If cache miss:
   - Call Amazon PAAPI
   - Normalize and store product data
   - Cache price, availability, and images
5. User previews product card
   - All previews are allowed (even for anonymous users)
6. User clicks "Copy Embed"
   - If **not logged in**, prompt for signup
   - If **logged in**, enforce quota:
     - Free plan: 5 lifetime credits
     - Pro plan: 25 credits/month
7. Create embed record in `embeds` table
   - Must be associated with a valid `user_id`
   - Store theme, timestamp, and product reference
8. Return static HTML embed code (not iframe)
   - Includes affiliate tag, structured markup, and optional "Powered by Jasin" badge

---

## 🔐 Security & Access Control

- PAAPI keys are never exposed to the frontend
- Anonymous users can **preview**, but not save or copy embeds
- Only authenticated users can generate embeds
- Embed limits enforced via API middleware based on:
  - `user_id`
  - `subscription_tier` from Supabase `profiles` table
- Stripe integration updates `subscription_tier` via webhook

---

## 📊 Supabase Tables

- `products`: ASIN-based cache of Amazon API data
- `embeds`: User-created embeds, linked to products + user_id
- `profiles`: Stores email, Stripe customer ID, subscription_tier

---

## 🛠 Deployment

- Hosted on **Vercel**
- API routes handle all secure logic server-side
- Environment variables stored in Vercel for:
  - Supabase keys
  - Amazon PAAPI keys
  - Stripe credentials

---

## ✅ Summary

Jasin securely generates and manages Amazon product embeds through a fast, cache-first API flow. Users preview cards freely, but must authenticate to save or share embeds. All logic is tier-gated, quota-enforced, and optimized for minimal API usage.




# 🧠 Backend API (Jasin MVP)

## 🌐 API Routes

| Route                     | Method | Auth | Purpose                            |
|---------------------------|--------|------|------------------------------------|
| `/api/parse`              | POST   | ✅   | Extract ASIN + enforce quota before lookup |
| `/api/product/:asin`      | GET    | ✅   | Returns cached or fresh product (auth required for embed) |
| `/api/cache/:asin`        | POST   | ✅   | Save result to Supabase `products` |
| `/api/embeds`             | POST   | ✅   | Save embed for logged-in user      |
| `/api/embeds`             | GET    | ✅   | Fetch all embeds for current user  |

---

## 🔐 Amazon PAAPI Integration

- Server-side only (runs on Vercel functions)
- Credentials stored securely in environment variables:
  ```env
  AMZ_ACCESS_KEY=
  AMZ_SECRET_KEY=
  AMZ_ASSOC_TAG=getjasin-20
  ```

- Always use Supabase cache first to avoid unnecessary PAAPI calls.
- Fallback to PAAPI only if:
  - ASIN not cached
  - Cached result is older than TTL window

---

## 🧠 Core Logic Summary

- **`/api/parse`** extracts ASIN and verifies user quota (free or paid)
- **`/api/product/:asin`** checks Supabase cache, falls back to PAAPI
- All returned data is normalized for frontend use
- PAAPI data is saved to `products` table after each lookup

---

## ⚠️ Error Handling

| Code | Meaning                         | Trigger                              |
|------|----------------------------------|--------------------------------------|
| 401  | Unauthenticated                  | Embed actions attempted without login |
| 429  | Rate limit or quota exceeded     | PAAPI or user plan limits hit         |
| 500  | Internal server error            | Unexpected issue in API logic        |

---

## ✅ Notes

- All API routes are rate-limited using Vercel or middleware
- Only logged-in users can generate and save embeds
- Free plan: 5 lifetime credits
- Pro plan: 25 credits per month



# Caching Optimization Strategy

## Overview
Implement Redis caching layer to reduce database load and improve response times. Current caching relies only on database TTL which is insufficient for high-traffic scenarios.

## Current Caching Issues

### 1. Database-Only Caching
**Problem**: All caching happens in the database via TTL checks:
```typescript
// Current pattern in amazon.ts
const hoursSinceUpdate = (now.getTime() - lastUpdated.getTime()) / (1000 * 60 * 60);
if (hoursSinceUpdate < 24) {
  // Return cached data
}
```

**Issues**:
- Every cache check requires database query
- No in-memory caching for hot data
- Dashboard data fetched fresh every time

### 2. No Application-Level Caching
**Problem**: No Redis or in-memory caching layer
**Impact**: Database hit on every request, even for frequently accessed data

### 3. Inefficient Cache Keys
**Problem**: No structured cache invalidation strategy
**Impact**: Stale data and cache pollution

## Redis Caching Strategy

### 1. Implement Redis Client
Add Redis dependency and client setup:

```typescript
// src/lib/redis.ts
import Redis from 'ioredis';

class RedisClient {
  private static instance: Redis;
  
  static getInstance(): Redis {
    if (!RedisClient.instance) {
      RedisClient.instance = new Redis(process.env.REDIS_URL || 'redis://localhost:6379', {
        retryDelayOnFailover: 100,
        maxRetriesPerRequest: 3,
        lazyConnect: true
      });
    }
    return RedisClient.instance;
  }
}

export const redis = RedisClient.getInstance();
```

### 2. Cache Layer Architecture
Implement multi-tier caching:

```typescript
// Cache hierarchy: Redis → Database → PAAPI
async function getCachedData<T>(
  key: string, 
  fetchFn: () => Promise<T>, 
  ttl: number = 300
): Promise<T> {
  
  // 1. Check Redis cache
  const cached = await redis.get(key);
  if (cached) {
    return JSON.parse(cached);
  }
  
  // 2. Fetch fresh data
  const data = await fetchFn();
  
  // 3. Cache in Redis
  await redis.setex(key, ttl, JSON.stringify(data));
  
  return data;
}
```

### 3. Dashboard Data Caching
Cache expensive dashboard queries:

```typescript
// Cache dashboard data for 5 minutes
async function getCachedDashboardData(userId: string) {
  return getCachedData(
    `dashboard:${userId}`,
    () => fetchDashboardDataFromDB(userId),
    300 // 5 minutes
  );
}
```

### 4. Product Data Caching
Multi-level product caching:

```typescript
// src/lib/cache/product-cache.ts
export class ProductCache {
  // Hot cache - 1 hour
  static async getProduct(asin: string): Promise<Product | null> {
    return getCachedData(
      `product:${asin}`,
      () => fetchProductFromDB(asin),
      3600 // 1 hour
    );
  }
  
  // Price cache - 30 minutes (more frequent updates)
  static async getProductPrice(asin: string): Promise<PriceData | null> {
    return getCachedData(
      `price:${asin}`,
      () => fetchPriceFromDB(asin),
      1800 // 30 minutes
    );
  }
  
  // Analytics cache - 10 minutes
  static async getProductAnalytics(embedId: string): Promise<Analytics | null> {
    return getCachedData(
      `analytics:${embedId}`,
      () => fetchAnalyticsFromDB(embedId),
      600 // 10 minutes
    );
  }
}
```

## Cache Invalidation Strategy

### 1. Time-Based Invalidation
Different TTL for different data types:
- **Product data**: 1 hour (relatively stable)
- **Prices**: 30 minutes (changes frequently)
- **Dashboard data**: 5 minutes (user-specific)
- **Analytics**: 10 minutes (updated frequently)

### 2. Event-Based Invalidation
Invalidate cache on data changes:

```typescript
// Invalidate when product is updated
async function updateProduct(asin: string, data: Partial<Product>) {
  await updateProductInDB(asin, data);
  
  // Invalidate related caches
  await redis.del(`product:${asin}`);
  await redis.del(`price:${asin}`);
  
  // Invalidate user dashboards that might show this product
  const userIds = await getUsersWithEmbed(asin);
  await Promise.all(
    userIds.map(userId => redis.del(`dashboard:${userId}`))
  );
}
```

### 3. Cache Warming
Pre-populate cache for popular products:

```typescript
// Warm cache for popular products
async function warmProductCache() {
  const popularAsins = await getPopularProducts(50);
  
  await Promise.all(
    popularAsins.map(async (asin) => {
      const product = await fetchProductFromDB(asin);
      await redis.setex(`product:${asin}`, 3600, JSON.stringify(product));
    })
  );
}
```

## Implementation Steps

### Step 1: Add Redis Infrastructure
- Add Redis to [package.json](mdc:package.json)
- Create Redis client in `src/lib/redis.ts`
- Add Redis URL to environment variables

### Step 2: Implement Cache Utilities
- Create `src/lib/cache/` directory
- Implement generic caching functions
- Add cache key management

### Step 3: Update Product Caching
- Modify [src/lib/amazon.ts](mdc:src/lib/amazon.ts) to use Redis cache
- Add cache invalidation on product updates

### Step 4: Cache Dashboard Data
- Update [src/app/dashboard/page.tsx](mdc:src/app/dashboard/page.tsx) to use cached data
- Cache the [get_complete_dashboard_data](mdc:supabase/migrations/20250103_optimize_dashboard_performance.sql) results

### Step 5: Add Cache Monitoring
```typescript
// Cache hit rate monitoring
class CacheMetrics {
  static hits = 0;
  static misses = 0;
  
  static recordHit() { this.hits++; }
  static recordMiss() { this.misses++; }
  
  static getHitRate() {
    const total = this.hits + this.misses;
    return total > 0 ? (this.hits / total) * 100 : 0;
  }
}
```

## Cache Configuration

### Redis Configuration
```typescript
// Optimized Redis config
const redisConfig = {
  host: process.env.REDIS_HOST,
  port: parseInt(process.env.REDIS_PORT || '6379'),
  password: process.env.REDIS_PASSWORD,
  db: 0,
  keyPrefix: 'jasin:',
  retryDelayOnFailover: 100,
  maxRetriesPerRequest: 3,
  lazyConnect: true,
  keepAlive: 30000,
  connectTimeout: 10000,
  commandTimeout: 5000
};
```

### Cache Keys Structure
```
jasin:product:{asin}           - Product data
jasin:price:{asin}             - Price data  
jasin:dashboard:{userId}       - Dashboard data
jasin:analytics:{embedId}      - Analytics data
jasin:user:{userId}:embeds     - User's embeds list
jasin:popular:products         - Popular products list
```

## Performance Targets
- **Cache hit rate**: > 80%
- **Dashboard load time**: 3-5s → 200-500ms
- **Product lookup**: 2-5s → 100-300ms
- **Database load reduction**: 60-70%

## Files to Modify
- [package.json](mdc:package.json) - Add Redis dependency
- [src/lib/amazon.ts](mdc:src/lib/amazon.ts) - Add product caching
- [src/app/dashboard/page.tsx](mdc:src/app/dashboard/page.tsx) - Add dashboard caching
- Create `src/lib/redis.ts` - Redis client
- Create `src/lib/cache/` - Cache utilities

## Environment Variables
```env
REDIS_URL=redis://localhost:6379
REDIS_HOST=localhost
REDIS_PORT=6379
REDIS_PASSWORD=your_password
```

## Testing Cache Performance
```typescript
// Test cache performance
async function testCachePerformance() {
  console.time('cache-miss');
  await getProduct('B0DGHYDZSD'); // First call - cache miss
  console.timeEnd('cache-miss');
  
  console.time('cache-hit');
  await getProduct('B0DGHYDZSD'); // Second call - cache hit
  console.timeEnd('cache-hit');
  
  console.log(`Cache hit rate: ${CacheMetrics.getHitRate()}%`);
}
```

## Expected Impact
- **Response times**: 70-80% improvement
- **Database load**: 60-70% reduction
- **User experience**: Near-instant dashboard loading
- **Cost savings**: Reduced database and PAAPI usage



# Carousel Implementation Details

## Overview
The product card carousel system consists of two synchronized carousels:
1. Main image carousel - displays the currently selected product image
2. Thumbnail carousel - shows smaller images that can be clicked to change the main view

## Main Carousel
- Implemented in [src/components/ProductCard.tsx](mdc:src/components/ProductCard.tsx) using Embla Carousel
- Initialized with settings: `skipSnaps: false, duration: 20, dragFree: false, loop: false`
- Selected images scale to 1.0 while non-selected images scale to 0.8
- Transitions use CSS `transition-transform duration-500` (500ms)
- Horizontal slide animation along the x-axis

## Thumbnail Carousel
- Uses settings: `containScroll: 'keepSnaps', dragFree: false, duration: 20, loop: false, align: 'start'`
- Shows up to 4 thumbnails at once
- Navigation arrows appear when more than 4 images are available
- Selected thumbnail is highlighted with a 2px ring/outline
- When arrows are clicked, the carousel scrolls by groups of up to 4 thumbnails

## Synchronization
- Clicking a thumbnail updates the main carousel via `emblaMainApi.scrollTo(index)`
- Main carousel changes trigger the `onSelect` handler to synchronize both carousels
- Navigation controls are disabled at carousel boundaries

## Static Implementation
- [public/embed.js](mdc:public/embed.js) loads [public/card/card-carousel.js](mdc:public/card/card-carousel.js)
- The static implementation needs to recreate the React behavior using vanilla JavaScript
- Carousel initialization is triggered via `initializeCarousels()` function
- The `card-carousel.js` file needs to handle:
  - Main carousel horizontal sliding animations
  - Thumbnail selection and highlighting
  - Arrow navigation with proper grouping behavior
  - Synchronization between main and thumbnail carousels
  - Disabling buttons at boundaries

## Animation Behaviors

### Primary Image Transitions
- When changing images, the transition is a horizontal slide (x-axis)
- Current image scales to 1.0, non-selected images scale to 0.8
- Animation duration is 500ms with ease transition
- Sliding direction follows the selection direction (left-to-right or right-to-left)

### Thumbnail Click Behavior
1. Clicked thumbnail receives visual highlight (2px ring)
2. Main carousel smoothly slides to show corresponding image
3. Previous selected image scales down as it slides out
4. New selected image scales up as it slides in
5. If needed, thumbnail carousel adjusts to ensure selected thumbnail is visible

### Thumbnail Arrow Navigation
1. Previous button scrolls thumbnails left by up to 4 items
2. Next button scrolls thumbnails right by up to 4 items
3. Transitions are smooth horizontal slides with 20ms duration
4. Buttons become disabled (grayed out) when reaching carousel limits
5. Scrolling uses calculated steps based on position and total thumbnails

## Implementation Requirements
The static implementation in [public/card/card-carousel.js](mdc:public/card/card-carousel.js) must recreate this behavior without React dependencies, using vanilla JavaScript DOM manipulation and animation.



## Credit pool management:
- Maintain a global PAAPI request counter in Supabase
- Set daily/monthly caps well below your actual limits (e.g., 80% of max)
- Create a system alert when approaching thresholds

## Graceful degradation:
- Cache hit notifications: "Using cached data (no credit used)"
- If PAAPI pool is depleted: "Service under high demand, only cached products available"
- Prioritize paid users when PAAPI quota runs low

## Technical implementation:
- Add credits_used and credits_remaining to user profiles table
- Track global PAAPI usage with timestamps
- Reset credits on billing cycle dates

## Backup strategies:
- Reserve 10-20% of PAAPI quota for paid users only
- Implement multiple PAAPI accounts for redundancy
- Extend cache lifetime during high demand periods

If PAAPI limits are hit, communicate transparently and offer compensation (extra credits next month).



# 🧩 Data Dependency Diagram (Mermaid)

This diagram reflects the updated architecture, where only authenticated users can save or copy embed code. Anonymous users may preview cards, but product data fetch and embed creation are quota-controlled and user-tied.

```mermaid
graph TD
  A[User Pastes Amazon Link] --> B[Extract ASIN + Affiliate Tag]
  B --> C{Check Products Cache}

  C -- Hit --> D[Return Cached Product]
  C -- Miss --> E[Fetch from Amazon PAAPI]
  E --> F[Parse + Normalize Product Data]
  F --> G[Store in Products Table]
  G --> H[Return Product Data]

  D --> I[Render Product Card Preview]
  H --> I

  I --> J{Is User Authenticated?}
  J -- No --> K[Prompt to Sign Up to Copy Embed]
  J -- Yes --> L[Check Plan + Quota]

  L --> M{Within Quota?}
  M -- No --> N[Show Quota Limit Message]
  M -- Yes --> O[Create Embed Record]
  O --> P[Generate Static HTML Embed Code]

  subgraph Product Data
    Q[Basic Info title/brand]
    R[Pricing current/original]
    S[Media primary/gallery]
    T[Specs dimensions/features]
    U[Variants summary]
  end

  subgraph Embed Tracking
    V[View Counter]
    W[Click Counter]
    X[User Association]
    Y[Theme Selection]
  end

  I --> Product Data
  O --> Embed Tracking
```



# Analytics and Data Retention

## Overview
Jasin's analytics system tracks views and clicks on product embeds. This data is collected when users interact with embedded product cards across the web.

## Analytics Collection
- Views are tracked via `/api/embed/[id]` when cards load 
- Clicks are tracked via `/api/click/[id]` when users click through to Amazon
- Both endpoints capture referrer URL and user agent metadata
- Analytics is a Pro-tier feature - see [prd](mdc:prd) for feature tiers

## Data Retention Policy
- Raw analytics data (individual views and clicks) is stored for 90 days
- After 90 days, raw data is automatically cleaned up by a scheduled job
- Aggregated statistics (total counts, daily/weekly/monthly summaries) are preserved long-term

## Implementation Details
- Analytics data is stored in `embed_views` and `embed_clicks` tables
- A scheduled cleanup job (`clean_old_analytics_data()`) runs weekly via pg_cron
- The job was configured in [20240722_add_analytics_retention.sql](mdc:supabase/migrations/20240722_add_analytics_retention.sql)
- Helper functions to manage the cleanup job are in [20240723_add_pg_cron_helper_functions.sql](mdc:supabase/migrations/20240723_add_pg_cron_helper_functions.sql)
- Client-side display of analytics is handled in [AnalyticsClient.tsx](mdc:src/app/analytics/AnalyticsClient.tsx)

## Testing
- To manually run the cleanup function: `SELECT clean_old_analytics_data();`
- To check pg_cron status: `SELECT check_pg_cron_enabled();`
- To verify the scheduled job: `SELECT * FROM cron.job WHERE jobname = 'clean-analytics-data-weekly';`


# Database Optimization Guide

## Overview
Database queries are the primary performance bottleneck. Current issues include missing indexes, multiple round trips, and inefficient query patterns.

## Critical Issues Identified

### Missing Database Indexes
The following indexes are missing and causing slow queries:

```sql
-- High priority indexes to add immediately
CREATE INDEX CONCURRENTLY idx_embeds_user_created ON embeds(user_id, created_at DESC);
CREATE INDEX CONCURRENTLY idx_paapi_calls_user_month ON paapi_calls(user_id, created_at) 
  WHERE created_at >= date_trunc('month', CURRENT_DATE);
CREATE INDEX CONCURRENTLY idx_embed_views_embed_bot ON embed_views(embed_id, is_bot, created_at);
CREATE INDEX CONCURRENTLY idx_embed_clicks_embed_bot ON embed_clicks(embed_id, is_bot, created_at);
CREATE INDEX CONCURRENTLY idx_products_asin_updated ON products(asin, updated_at);
```

### Dashboard Query Consolidation
The [dashboard page](mdc:src/app/dashboard/page.tsx) currently makes multiple separate queries even after the "optimization":

**Current Problem**:
```typescript
// Multiple round trips in dashboard
const { data: dashboardData } = await supabase.rpc('get_complete_dashboard_data', { p_user_id: user.id });
const { count: totalEmbedsCount } = await supabase.from('embeds')...  // Additional query!
const { count: monthlyEmbedsCount } = await supabase.from('embeds')... // Another query!
```

**Solution**: Update the [get_complete_dashboard_data function](mdc:supabase/migrations/20250103_optimize_dashboard_performance.sql) to include ALL data.

### Analytics Query Optimization
Current analytics queries in [get_complete_dashboard_data](mdc:supabase/migrations/20250103_optimize_dashboard_performance.sql) are inefficient:

**Problem**: Multiple LEFT JOINs on large tables
**Solution**: Create materialized view for analytics:

```sql
CREATE MATERIALIZED VIEW mv_user_analytics_fast AS
SELECT 
  e.user_id,
  COUNT(DISTINCT e.id) as total_embeds,
  COUNT(DISTINCT CASE WHEN ev.is_bot = false THEN ev.id END) as total_views,
  COUNT(DISTINCT CASE WHEN ec.is_bot = false THEN ec.id END) as total_clicks,
  COUNT(DISTINCT CASE WHEN ev.is_bot = false AND ev.created_at >= CURRENT_DATE - INTERVAL '7 days' THEN ev.id END) as last_7_days_views,
  COUNT(DISTINCT CASE WHEN ec.is_bot = false AND ec.created_at >= CURRENT_DATE - INTERVAL '7 days' THEN ec.id END) as last_7_days_clicks
FROM embeds e
LEFT JOIN embed_views ev ON e.id = ev.embed_id
LEFT JOIN embed_clicks ec ON e.id = ec.embed_id
GROUP BY e.user_id;

-- Refresh every hour
CREATE UNIQUE INDEX ON mv_user_analytics_fast(user_id);
```

## Implementation Steps

### Step 1: Add Missing Indexes
Create migration file: `supabase/migrations/YYYYMMDD_add_performance_indexes.sql`

### Step 2: Fix Dashboard Function
Update [get_complete_dashboard_data](mdc:supabase/migrations/20250103_optimize_dashboard_performance.sql) to eliminate additional queries in [dashboard page](mdc:src/app/dashboard/page.tsx).

### Step 3: Create Analytics Materialized View
Add materialized view for fast analytics lookups.

### Step 4: Add Query Performance Monitoring
```sql
-- Enable slow query logging
ALTER SYSTEM SET log_min_duration_statement = 1000; -- Log queries > 1s
ALTER SYSTEM SET log_statement = 'mod'; -- Log DDL statements
```

## Query Performance Targets
- Dashboard queries: < 100ms
- Product lookups: < 50ms  
- Analytics queries: < 200ms
- Embed creation: < 100ms

## Files to Modify
- [supabase/migrations/20250103_optimize_dashboard_performance.sql](mdc:supabase/migrations/20250103_optimize_dashboard_performance.sql) - Update function
- [src/app/dashboard/page.tsx](mdc:src/app/dashboard/page.tsx) - Remove duplicate queries
- Create new migration for indexes and materialized views

## Testing
```sql
-- Test query performance
EXPLAIN (ANALYZE, BUFFERS) 
SELECT * FROM get_complete_dashboard_data('user-uuid');

-- Check index usage
SELECT schemaname, tablename, indexname, idx_scan, idx_tup_read, idx_tup_fetch 
FROM pg_stat_user_indexes 
WHERE idx_scan = 0;
```



# Database Architecture and Migration Patterns

## Database Schema Overview

### Core Tables
- **`products`**: Amazon product data cache (ASIN-based)
- **`embeds`**: User-created product cards with analytics
- **`profiles`**: User subscription and preference data
- **`paapi_calls`**: Amazon API usage tracking and rate limiting

### Analytics Tables
- **`embed_views`**: Individual view events with referrer data
- **`embed_clicks`**: Individual click events with referrer data
- **`embed_stats`**: Aggregated statistics per embed
- **`daily_stats`**: Time-series data for charting

### Anti-Abuse Tables
- **`device_fingerprints`**: Browser/device tracking for abuse detection
- **`email_domain_reputation`**: Email domain scoring and blocking
- **`abuse_signals`**: Suspicious activity logging
- **`credit_usage_patterns`**: Credit consumption pattern analysis

## Migration Patterns

### Migration File Structure
```
supabase/migrations/YYYYMMDD_descriptive_name.sql
```

### Standard Migration Template
```sql
-- Migration: descriptive_name
-- Created at: YYYY-MM-DD HH:MM:SS
-- Description: What this migration does

BEGIN;

-- Create tables with proper constraints
CREATE TABLE IF NOT EXISTS public.table_name (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID REFERENCES auth.users(id) ON DELETE CASCADE,
  created_at TIMESTAMP WITH TIME ZONE DEFAULT now(),
  updated_at TIMESTAMP WITH TIME ZONE DEFAULT now()
);

-- Add indexes for performance
CREATE INDEX IF NOT EXISTS idx_table_name_user_id ON public.table_name(user_id);
CREATE INDEX IF NOT EXISTS idx_table_name_created_at ON public.table_name(created_at);

-- Enable Row Level Security
ALTER TABLE public.table_name ENABLE ROW LEVEL SECURITY;

-- Create RLS policies
CREATE POLICY "Users can view own data" ON public.table_name
  FOR SELECT TO authenticated
  USING (user_id = auth.uid());

CREATE POLICY "Users can insert own data" ON public.table_name
  FOR INSERT TO authenticated
  WITH CHECK (user_id = auth.uid());

COMMIT;
```

### Function Creation Pattern
```sql
-- Create or replace functions with proper security
CREATE OR REPLACE FUNCTION function_name(
  p_param1 TYPE,
  p_param2 TYPE DEFAULT NULL
)
RETURNS RETURN_TYPE AS $$
DECLARE
  v_variable TYPE;
BEGIN
  -- Function logic here
  RETURN result;
EXCEPTION
  WHEN OTHERS THEN
    -- Error handling
    RAISE EXCEPTION 'Function failed: %', SQLERRM;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;
```

## Row Level Security (RLS) Patterns

### User Data Access
```sql
-- Standard user data policy
CREATE POLICY "Users can manage own data" ON public.user_table
  FOR ALL TO authenticated
  USING (user_id = auth.uid())
  WITH CHECK (user_id = auth.uid());
```

### Admin-Only Access
```sql
-- Admin-only tables (anti-abuse, system data)
CREATE POLICY "Admin access only" ON public.admin_table
  FOR ALL TO authenticated
  USING (false); -- Only service role can access
```

### Public Read, Authenticated Write
```sql
-- Public embeds with authenticated creation
CREATE POLICY "Public read access" ON public.embeds
  FOR SELECT TO anon, authenticated
  USING (true);

CREATE POLICY "Authenticated users can create" ON public.embeds
  FOR INSERT TO authenticated
  WITH CHECK (user_id = auth.uid());
```

### Quota Enforcement Policies
```sql
-- Credit limit enforcement in RLS
CREATE POLICY "Enforce embed quota" ON public.embeds
FOR INSERT TO authenticated
WITH CHECK (
  -- Free tier: 5 total embeds
  (
    (SELECT subscription_tier FROM profiles WHERE id = auth.uid()) = 'free'
    AND
    (SELECT COUNT(*) FROM embeds WHERE user_id = auth.uid()) < 5
  )
  OR
  -- Pro tier: 50 embeds per month
  (
    (SELECT subscription_tier FROM profiles WHERE id = auth.uid()) = 'pro'
    AND
    (SELECT COUNT(*) FROM embeds 
     WHERE user_id = auth.uid() 
     AND created_at >= date_trunc('month', CURRENT_DATE)) < 50
  )
);
```

## Database Function Patterns

### Analytics Functions
```sql
-- Transactional analytics tracking
CREATE OR REPLACE FUNCTION track_embed_event_with_transaction(
  p_embed_id UUID,
  p_event_type TEXT,
  p_referer_url TEXT DEFAULT NULL,
  p_user_agent TEXT DEFAULT NULL,
  p_is_bot BOOLEAN DEFAULT false,
  p_bot_type TEXT DEFAULT NULL
)
RETURNS VOID AS $$
BEGIN
  -- Update counters in embeds table
  IF p_event_type = 'view' THEN
    UPDATE embeds SET views = views + 1 WHERE id = p_embed_id;
  ELSIF p_event_type = 'click' THEN
    UPDATE embeds SET clicks = clicks + 1 WHERE id = p_embed_id;
  END IF;
  
  -- Insert detailed record
  IF p_event_type = 'view' THEN
    INSERT INTO embed_views (embed_id, referer_url, user_agent, is_bot, bot_type)
    VALUES (p_embed_id, p_referer_url, p_user_agent, p_is_bot, p_bot_type);
  ELSIF p_event_type = 'click' THEN
    INSERT INTO embed_clicks (embed_id, referer_url, user_agent, is_bot, bot_type)
    VALUES (p_embed_id, p_referer_url, p_user_agent, p_is_bot, p_bot_type);
  END IF;
  
  -- Update aggregated stats
  INSERT INTO embed_stats (embed_id, total_views, total_clicks)
  VALUES (p_embed_id, 
    CASE WHEN p_event_type = 'view' THEN 1 ELSE 0 END,
    CASE WHEN p_event_type = 'click' THEN 1 ELSE 0 END
  )
  ON CONFLICT (embed_id) DO UPDATE SET
    total_views = embed_stats.total_views + EXCLUDED.total_views,
    total_clicks = embed_stats.total_clicks + EXCLUDED.total_clicks,
    updated_at = now();
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;
```

### Data Cleanup Functions
```sql
-- Scheduled cleanup with configurable retention
CREATE OR REPLACE FUNCTION clean_old_analytics_data(
  p_retention_days INTEGER DEFAULT 90
)
RETURNS INTEGER AS $$
DECLARE
  v_deleted_count INTEGER;
  v_cutoff_date TIMESTAMP WITH TIME ZONE;
BEGIN
  v_cutoff_date := NOW() - (p_retention_days || ' days')::INTERVAL;
  
  -- Delete old view records
  DELETE FROM embed_views WHERE created_at < v_cutoff_date;
  GET DIAGNOSTICS v_deleted_count = ROW_COUNT;
  
  -- Delete old click records
  DELETE FROM embed_clicks WHERE created_at < v_cutoff_date;
  GET DIAGNOSTICS v_deleted_count = v_deleted_count + ROW_COUNT;
  
  RETURN v_deleted_count;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;
```

## Indexing Strategies

### Performance Indexes
```sql
-- User-based queries
CREATE INDEX IF NOT EXISTS idx_embeds_user_id ON embeds(user_id);
CREATE INDEX IF NOT EXISTS idx_embeds_created_at ON embeds(created_at);

-- Analytics queries
CREATE INDEX IF NOT EXISTS idx_embed_views_embed_id ON embed_views(embed_id);
CREATE INDEX IF NOT EXISTS idx_embed_views_created_at ON embed_views(created_at);

-- Anti-abuse queries
CREATE INDEX IF NOT EXISTS idx_abuse_signals_user_id ON abuse_signals(user_id);
CREATE INDEX IF NOT EXISTS idx_abuse_signals_type ON abuse_signals(signal_type);
CREATE INDEX IF NOT EXISTS idx_abuse_signals_created_at ON abuse_signals(created_at);
```

### Composite Indexes
```sql
-- Multi-column indexes for complex queries
CREATE INDEX IF NOT EXISTS idx_embeds_user_created ON embeds(user_id, created_at);
CREATE INDEX IF NOT EXISTS idx_paapi_calls_user_success ON paapi_calls(user_id, success, created_at);
```

## Scheduled Jobs (pg_cron)

### Job Configuration Pattern
```sql
-- Schedule cleanup jobs
SELECT cron.schedule(
  'clean-analytics-data-weekly',
  '0 0 * * 0', -- Every Sunday at midnight
  'SELECT clean_old_analytics_data(90);'
);

-- Schedule refresh jobs
SELECT cron.schedule(
  'refresh-analytics-summary-hourly',
  '0 * * * *', -- Every hour
  'REFRESH MATERIALIZED VIEW CONCURRENTLY mv_user_analytics_summary;'
);
```

### Job Management Functions
```sql
-- Helper functions for cron job management
CREATE OR REPLACE FUNCTION check_pg_cron_enabled()
RETURNS BOOLEAN AS $$
BEGIN
  RETURN EXISTS (
    SELECT 1 FROM pg_extension WHERE extname = 'pg_cron'
  );
END;
$$ LANGUAGE plpgsql;

CREATE OR REPLACE FUNCTION list_cron_jobs()
RETURNS TABLE(jobname TEXT, schedule TEXT, command TEXT, active BOOLEAN) AS $$
BEGIN
  RETURN QUERY
  SELECT j.jobname, j.schedule, j.command, j.active
  FROM cron.job j
  ORDER BY j.jobname;
END;
$$ LANGUAGE plpgsql;
```

## Data Seeding Patterns

### Reference Data Seeding
```sql
-- Seed known data with conflict handling
INSERT INTO email_domain_reputation (domain, reputation_score, is_disposable, is_blocked) VALUES
  ('gmail.com', 90, false, false),
  ('outlook.com', 90, false, false),
  ('10minutemail.com', 0, true, true),
  ('guerrillamail.com', 0, true, true)
ON CONFLICT (domain) DO UPDATE SET
  reputation_score = EXCLUDED.reputation_score,
  is_disposable = EXCLUDED.is_disposable,
  is_blocked = EXCLUDED.is_blocked,
  updated_at = now();
```

## Client Integration Patterns

### Server-Side Client (Admin Operations)
```typescript
// src/lib/supabase.ts
import { createClient } from '@supabase/supabase-js';

export function getSupabaseAdmin() {
  return createClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.SUPABASE_SERVICE_ROLE_KEY!, // Admin privileges
    {
      auth: {
        autoRefreshToken: false,
        persistSession: false
      }
    }
  );
}
```

### Client-Side with RLS
```typescript
// src/utils/supabase/client.ts
import { createBrowserClient } from '@supabase/ssr';

export function createClient() {
  return createBrowserClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY! // RLS enforced
  );
}
```

### Server Component Client
```typescript
// src/utils/supabase/server.ts
import { createServerClient } from '@supabase/ssr';
import { cookies } from 'next/headers';

export function createClient() {
  const cookieStore = cookies();
  
  return createServerClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
    {
      cookies: {
        get(name: string) {
          return cookieStore.get(name)?.value;
        },
      },
    }
  );
}
```

## Migration Management

### Migration Scripts
- **Create**: `npm run db:create migration_name`
- **Fix Timestamps**: `npm run db:fix-all`
- **Sync Remote**: `npm run db:sync-migrations`
- **Push**: `npm run db:push`

### Migration Best Practices
1. Always use transactions (BEGIN/COMMIT)
2. Use `IF NOT EXISTS` for idempotent operations
3. Enable RLS immediately after table creation
4. Create indexes after table creation
5. Seed reference data with conflict handling
6. Test migrations on development first

### Rollback Strategies
```sql
-- Include rollback instructions in migration comments
-- ROLLBACK: DROP TABLE IF EXISTS new_table;
-- ROLLBACK: ALTER TABLE old_table DROP COLUMN IF EXISTS new_column;
```

## Performance Optimization

### Query Optimization
- Use appropriate indexes for common query patterns
- Leverage RLS for security without performance impact
- Use materialized views for complex aggregations
- Implement proper pagination for large datasets

### Connection Management
- Use connection pooling in production
- Implement proper timeout handling
- Monitor connection usage and limits
- Use read replicas for analytics queries when available



# Database Subscription Schema

## Overview
The `profiles` table stores subscription data that must remain synchronized with Stripe's authoritative data.

## Profiles Table Schema
Key subscription fields in the `profiles` table:
```sql
subscription_tier: 'free' | 'pro'
subscription_id: text (Stripe subscription ID)
subscription_status: text (Stripe status: active, canceled, etc.)
subscription_period_end: timestamp (authoritative end date from Stripe)
subscription_cancel_at_period_end: boolean
subscription_plan_id: text (Stripe price ID)
subscription_plan_interval: text ('month' | 'year')
subscription_amount: integer (amount in cents)
subscription_updated_at: timestamp
stripe_customer_id: text (Stripe customer ID)
```

## Data Integrity Rules

### 1. Authoritative End Dates
- `subscription_period_end` must store Stripe's `ended_at` for cancelled subscriptions
- Never store calculated dates - always use Stripe's provided timestamps
- For active subscriptions, use `current_period_end`

### 2. Status Synchronization
- `subscription_tier` should be 'pro' for active/trialing subscriptions
- `subscription_tier` should be 'free' for canceled/expired subscriptions
- `subscription_status` must match Stripe's subscription status exactly

### 3. Update Patterns
```sql
-- ✅ CORRECT: Store authoritative Stripe data
UPDATE profiles SET
  subscription_period_end = stripe_ended_at_or_current_period_end,
  subscription_status = stripe_subscription_status,
  subscription_updated_at = NOW()
WHERE id = user_id;

-- ❌ NEVER: Calculate or modify Stripe timestamps
UPDATE profiles SET
  subscription_period_end = calculated_date
WHERE id = user_id;
```

## Webhook Data Flow
1. Stripe webhook receives subscription event
2. Extract `ended_at` or `current_period_end` from Stripe data
3. Store in `subscription_period_end` without modification
4. Update `subscription_updated_at` for cache invalidation

## Query Patterns
When checking subscription status:
```sql
-- Check if user has active subscription
SELECT subscription_tier, subscription_period_end, subscription_cancel_at_period_end
FROM profiles 
WHERE id = $1 AND subscription_tier = 'pro';

-- Get subscription end date for UI display
SELECT subscription_period_end
FROM profiles
WHERE id = $1;
```

## Cache Management
- Use `subscription_updated_at` to determine cache freshness
- 24-hour cache threshold for subscription data
- Force refresh when webhook updates occur

## Data Consistency Checks
Periodically verify:
- `subscription_period_end` matches Stripe's authoritative data
- No calculated dates exist in the database
- Cancelled subscriptions have proper end dates stored

## Migration Considerations
When updating existing data:
- Fetch current Stripe subscription data
- Replace any calculated dates with Stripe's `ended_at`
- Ensure all timestamps are in UTC



# Embeddable Card System

## Overview
Jasin's embeddable card system generates Amazon product cards that can be embedded on any website with a simple script tag. The system has two key components:

1. The embed script (`embed.js`) which renders cards on third-party sites
2. The preview system which generates static HTML previews in the dashboard

## Embed Script Implementation

The [embed.js](mdc:public/embed.js) script is completely self-contained and runs without dependencies. When a website includes the script tag:

```html
<script
  async
  src="https://getjasin.com/embed.js"
  data-id="[embed-uuid]"
  data-theme="light"
  data-show-badge="true"
  data-size="medium"
></script>
```

The script:
1. Self-initializes on document load
2. Extracts configuration from its own data attributes
3. Fetches product data from `/api/embed/[id]`
4. Dynamically loads CSS based on theme
5. Renders the HTML card directly into the DOM
6. Tracks views using the Beacon API

## Data Flow

1. User creates an embed in the dashboard, generating a record in the `embeds` table
2. Each embed references a product by ASIN
3. Product data is stored in the `products` table with full details (price, images, specs)
4. When requested by `embed.js` or preview:
   - API combines the embed configuration with complete product data
   - Card is rendered using this combined dataset

## Preview System

The [preview page](mdc:src/app/preview/page.tsx) allows users to see how cards will look before embedding:

1. User enters an embed UUID
2. Same API endpoint (`/api/embed/[id]`) is called as with the real embed
3. `getCardWithData()` function generates a complete HTML preview:
   - Formats titles, prices, and timestamps
   - Renders availability indicators, thumbnails, features and specifications
   - Includes all necessary CSS and structure

## Key Components

- **Base Styling**: [card-base.css](mdc:public/card/card-base.css)
- **Theme Variants**: [card-light.css](mdc:public/card/card-light.css) and [card-dark.css](mdc:public/card/card-dark.css)
- **Formatting Rules**: [productDisplayRules.ts](mdc:src/config/productDisplayRules.ts)
- **Server-side API**: [api/embed/[id]/route.ts](mdc:src/app/api/embed/[id]/route.ts)

## Analytics Tracking

When cards are embedded on external sites:
1. View tracking occurs when cards load via Beacon API
2. Click tracking routes through `/api/click/[id]` before redirecting to Amazon
3. All analytics are stored in the database for the dashboard



# Environment File Handling

## Important: .env.local File Exists

The project **DOES HAVE** a `.env.local` file that contains all necessary environment variables. This file is:
- Present in the workspace root
- Hidden from AI visibility due to security reasons
- Protected by [.gitignore](mdc:.gitignore) 
- Contains all required keys for Supabase, Amazon PAAPI, Stripe, etc.

## Rule for AI Assistant

**NEVER assume the `.env.local` file is missing or doesn't exist.**

When you need environment variable values:

1. ✅ **DO**: Ask the user to provide specific environment variable values
2. ✅ **DO**: Say "Can you share the value of VARIABLE_NAME from your .env.local file?"
3. ✅ **DO**: Request multiple variables at once if needed
4. ❌ **DON'T**: Assume the file doesn't exist
5. ❌ **DON'T**: Suggest creating a new .env.local file
6. ❌ **DON'T**: Say "you need to create an .env.local file"

## Example Requests

Instead of: "You need to create a .env.local file"
Say: "Can you share your NEXT_PUBLIC_SUPABASE_URL and SUPABASE_SERVICE_ROLE_KEY values from your .env.local file?"

Instead of: "The .env.local file is missing"
Say: "I need to see your environment variables. Can you paste the contents of your .env.local file?"

## Security Note

The user can safely share .env.local contents in chat since:
- The file is protected by gitignore
- Chat contents aren't saved to files
- This is the intended way to share sensitive config during debugging



# Frontend Optimization Guide

## Overview
Optimize React components, implement proper loading states, add code splitting, and improve overall frontend performance. Current issues include missing memoization, no loading states, and inefficient re-renders.

## Current Frontend Issues

### 1. Missing Component Memoization
**Problem**: Expensive components re-render unnecessarily
**Impact**: Wasted CPU cycles and slower UI updates

### 2. No Loading States
**Problem**: Users see blank screens during data fetching
**Impact**: Poor user experience and perceived slowness

### 3. No Code Splitting
**Problem**: Large JavaScript bundles slow initial page load
**Impact**: Slower Time to First Byte (TTFB) and First Contentful Paint (FCP)

### 4. Inefficient State Management
**Problem**: Unnecessary re-renders cascade through component tree
**Impact**: Sluggish UI interactions

## Optimization Strategy

### 1. Implement React.memo for Expensive Components
Identify and memoize components that render frequently:

```typescript
// src/components/ProductCard.tsx - Expensive component
import React from 'react';

interface ProductCardProps {
  product: Product;
  theme: string;
  onUpdate?: (id: string) => void;
}

const ProductCard = React.memo<ProductCardProps>(({ product, theme, onUpdate }) => {
  // Expensive rendering logic
  return (
    <div className="product-card">
      {/* Complex product display */}
    </div>
  );
}, (prevProps, nextProps) => {
  // Custom comparison function
  return (
    prevProps.product.asin === nextProps.product.asin &&
    prevProps.theme === nextProps.theme &&
    prevProps.product.updated_at === nextProps.product.updated_at
  );
});

ProductCard.displayName = 'ProductCard';
export default ProductCard;
```

### 2. Add Comprehensive Loading States
Implement skeleton screens and loading indicators:

```typescript
// src/components/DashboardSkeleton.tsx
export function DashboardSkeleton() {
  return (
    <div className="animate-pulse">
      <div className="h-8 bg-gray-200 rounded w-1/4 mb-4"></div>
      <div className="grid gap-4 md:grid-cols-2 lg:grid-cols-3">
        {[...Array(6)].map((_, i) => (
          <div key={i} className="h-48 bg-gray-200 rounded"></div>
        ))}
      </div>
    </div>
  );
}

// src/app/dashboard/page.tsx - Add Suspense boundaries
import { Suspense } from 'react';

export default function Dashboard() {
  return (
    <AuthedLayout>
      <Suspense fallback={<DashboardSkeleton />}>
        <DashboardContent />
      </Suspense>
    </AuthedLayout>
  );
}
```

### 3. Implement Code Splitting
Split large components and routes:

```typescript
// src/app/dashboard/DashboardClient.tsx - Lazy load heavy components
import { lazy, Suspense } from 'react';

const AnalyticsChart = lazy(() => import('@/components/AnalyticsChart'));
const ProductBuilder = lazy(() => import('@/components/ProductBuilder'));

export function DashboardClient() {
  return (
    <div>
      <Suspense fallback={<div>Loading analytics...</div>}>
        <AnalyticsChart />
      </Suspense>
      
      <Suspense fallback={<div>Loading builder...</div>}>
        <ProductBuilder />
      </Suspense>
    </div>
  );
}
```

### 4. Optimize State Management
Use React Query/SWR for server state management:

```typescript
// src/hooks/useDashboardData.ts
import useSWR from 'swr';

interface DashboardData {
  products: Product[];
  analytics: Analytics;
  subscription: Subscription;
}

export function useDashboardData(userId: string) {
  const { data, error, isLoading, mutate } = useSWR<DashboardData>(
    `/api/dashboard/${userId}`,
    fetcher,
    {
      revalidateOnFocus: false,
      revalidateOnReconnect: true,
      dedupingInterval: 60000, // 1 minute
      errorRetryCount: 3
    }
  );

  return {
    data,
    isLoading,
    isError: error,
    refresh: mutate
  };
}
```

### 5. Implement Virtual Scrolling
For large lists (embed lists, analytics tables):

```typescript
// src/components/VirtualizedEmbedList.tsx
import { FixedSizeList as List } from 'react-window';

interface EmbedListProps {
  embeds: Embed[];
  height: number;
}

const EmbedRow = React.memo(({ index, style, data }) => (
  <div style={style}>
    <EmbedCard embed={data[index]} />
  </div>
));

export function VirtualizedEmbedList({ embeds, height }: EmbedListProps) {
  return (
    <List
      height={height}
      itemCount={embeds.length}
      itemSize={120}
      itemData={embeds}
    >
      {EmbedRow}
    </List>
  );
}
```

## Performance Monitoring

### 1. Add Performance Metrics
```typescript
// src/lib/performance.ts
export class PerformanceMonitor {
  static measureComponent(name: string) {
    return function<T extends React.ComponentType<any>>(Component: T): T {
      const MeasuredComponent = React.forwardRef((props, ref) => {
        const renderStart = performance.now();
        
        React.useEffect(() => {
          const renderEnd = performance.now();
          const duration = renderEnd - renderStart;
          
          if (duration > 16) { // Slower than 60fps
            console.warn(`Slow render: ${name} took ${duration.toFixed(2)}ms`);
          }
        });
        
        return <Component {...props} ref={ref} />;
      });
      
      MeasuredComponent.displayName = `Measured(${name})`;
      return MeasuredComponent as T;
    };
  }
}

// Usage
const ProductCard = PerformanceMonitor.measureComponent('ProductCard')(
  React.memo(ProductCardComponent)
);
```

### 2. Bundle Analysis
Add bundle analyzer to identify large dependencies:

```typescript
// next.config.js - Add bundle analyzer
const withBundleAnalyzer = require('@next/bundle-analyzer')({
  enabled: process.env.ANALYZE === 'true',
});

module.exports = withBundleAnalyzer({
  // ... existing config
});
```

### 3. Core Web Vitals Monitoring
```typescript
// src/lib/web-vitals.ts
import { getCLS, getFID, getFCP, getLCP, getTTFB } from 'web-vitals';

function sendToAnalytics(metric: any) {
  // Send to your analytics service
  console.log(metric);
}

export function initWebVitals() {
  getCLS(sendToAnalytics);
  getFID(sendToAnalytics);
  getFCP(sendToAnalytics);
  getLCP(sendToAnalytics);
  getTTFB(sendToAnalytics);
}
```

## Implementation Steps

### Step 1: Add Component Memoization
- Identify expensive components in [src/components/](mdc:src/components)
- Add React.memo to ProductCard, EmbedCard, AnalyticsChart
- Implement custom comparison functions

### Step 2: Implement Loading States
- Create skeleton components for each major section
- Add Suspense boundaries to [dashboard](mdc:src/app/dashboard/page.tsx)
- Implement progressive loading for heavy components

### Step 3: Add Code Splitting
- Lazy load analytics components
- Split admin dashboard components
- Implement route-based code splitting

### Step 4: Optimize Data Fetching
- Replace direct Supabase calls with SWR/React Query
- Implement proper error boundaries
- Add optimistic updates

### Step 5: Add Performance Monitoring
- Implement render time monitoring
- Add bundle analysis
- Monitor Core Web Vitals

## Performance Targets
- **First Contentful Paint (FCP)**: < 1.5s
- **Largest Contentful Paint (LCP)**: < 2.5s
- **Cumulative Layout Shift (CLS)**: < 0.1
- **First Input Delay (FID)**: < 100ms
- **Component render time**: < 16ms (60fps)

## Files to Modify
- [src/app/dashboard/page.tsx](mdc:src/app/dashboard/page.tsx) - Add Suspense
- [src/components/](mdc:src/components) - Add memoization
- [package.json](mdc:package.json) - Add SWR, react-window
- [next.config.js](mdc:next.config.js) - Bundle analyzer

## Dependencies to Add
```json
{
  "swr": "^2.2.4",
  "react-window": "^1.8.8",
  "react-window-infinite-loader": "^1.0.9",
  "@next/bundle-analyzer": "^14.0.2",
  "web-vitals": "^3.5.0"
}
```

## Testing Performance
```typescript
// Performance testing utilities
export function measureRenderTime<T>(
  Component: React.ComponentType<T>,
  props: T
): Promise<number> {
  return new Promise((resolve) => {
    const start = performance.now();
    
    const TestComponent = () => {
      React.useEffect(() => {
        const end = performance.now();
        resolve(end - start);
      });
      
      return <Component {...props} />;
    };
    
    render(<TestComponent />);
  });
}
```

## Expected Impact
- **Initial page load**: 30-50% faster
- **Component render times**: 60-80% improvement
- **User interaction responsiveness**: 2-3x faster
- **Bundle size**: 20-30% reduction through code splitting
- **Memory usage**: 40-50% reduction through proper memoization



# 🧩 Frontend System (Jasin MVP)

---

## 🛠️ Stack

- **Next.js 14**
- **Tailwind CSS**
- **shadcn/ui** – component primitives
- **Radix UI** – modal, popover, dropdown
- **Lucide Icons** – icon system
- **Supabase client SDK** – auth + API integration

---

## 🧱 Routes

| Path          | Purpose                                | Auth Required |
|---------------|----------------------------------------|----------------|
| `/`           | Home: paste Amazon link + preview card | ❌ (public)    |
| `/signup`     | Sign up with email + password          | ❌             |
| `/login`      | Log in with email + password           | ❌             |
| `/dashboard`  | View + manage saved embeds             | ✅             |
| `/card/[id]`  | View single product card               | ❌ (embed preview only) |

---

## 🧩 UI Components

- `InputField` → Accepts Amazon link or ASIN
- `CardPreview` → Displays product card
- `CopyEmbedButton` → Gated by login status
- `EmbedCodeBox` → Shows code, masked for unauthenticated users
- `SignupModal` → Triggered when copying as anonymous user
- `ColorSelector` → Theme selector (Pro features gated)
- `AuthButton` → Auth state (login/logout/account)
- `EmbedList` → Dashboard list of saved cards
- `ErrorNotice` → Handles API, auth, or parsing errors

---

## 🧠 State Management

- **Global context or zustand (optional)**
  - Auth state
  - Product/lookup state
  - Theme selection
- Auth-aware components read from context and re-render on login

---

## 🎨 Design System

- **Tailwind CSS** for layout, spacing, and typography
- **shadcn/ui** for consistent component primitives
- **Radix UI** for dialogs, popovers, and accessibility
- Theme tokens and component variants defined by Tailwind config

---

## ✅ Notes

- Embed copying requires authentication (enforced in `CopyEmbedButton`)
- Free accounts show limit messaging at 5 total embeds
- Pro accounts unlock higher quota + more themes
- Signup flow uses Supabase email/password auth




# Jasin – Paste-to-Preview for Amazon Affiliates

Jasin turns Amazon affiliate links into clean, customizable product cards—no code required. Built for bloggers, content creators, and affiliate marketers who want better-looking embeds without the bloat.

---

## 🚀 What It Does

- Paste any Amazon product URL or ASIN
- Fetches real-time data using Amazon PAAPI
- Renders a responsive, branded product card
- Requires signup to save or copy the embed
- Authenticated users can access saved cards from a dashboard

---

## 💡 How It Works

### 🔍 No Account? You Can Still:
- Paste a product link and preview the card
- Explore different theme styles
- See what your card will look like—but copying the embed is gated

### 🧑‍💻 Create a Free Account to:
- Copy HTML embed code
- Save cards to your dashboard
- Use up to **5 total credits** (lifetime)

### 💳 Upgrade to Pro for:
- **25 new credits per month**
- Save and manage unlimited cards
- Unlock detailed analytics: views, clicks, CTR, top-performing products
- Customize themes and brand colors
- Optionally remove or customize the "Powered by Jasin" badge

---

## 📦 Plan Breakdown

| Plan     | Price                  | Credit Limit        | Save Cards  | Analytics  | Custom Themes   |
|----------|------------------------|--------------------|-------------|------------|-----------------|
| Free     | $0                     | 5 lifetime         | ✅          | ❌          | ❌              |
| Pro      | $15/month or $120/year | 25/month           | ✅          | ✅          | ✅              |

> ⚠️ All usage is tied to your account. Anonymous users cannot copy embeds or save cards.

---

## 🔐 Authentication

- Email + password or magic link login (via Supabase)
- All embed tracking and quota enforcement is account-based
- Stripe handles Pro billing; subscription tier is synced automatically

---

## 🧱 Tech Stack

| Layer       | Tech                            |
|-------------|----------------------------------|
| Frontend    | Next.js, Tailwind, shadcn/ui     |
| Backend     | Next.js API Routes               |
| UI Logic    | React Server Components          |
| Database    | Supabase (PostgreSQL)            |
| Auth        | Supabase Auth                    |
| Billing     | Stripe (webhooks to Supabase)    |
| Product API | Amazon PAAPI 5.0                 |

---

## 🔁 Product Flow

1. User lands on homepage and pastes a product link
2. App extracts ASIN and affiliate tag
3. Supabase cache is checked
4. If cache miss, data is fetched via Amazon PAAPI
5. Card is rendered with product data and previewed on `/card/:id`
6. Copying the embed or saving the card requires account signup
7. Authenticated users see saved cards on their dashboard

---

## ⚙️ Local Development

```bash
npx create-next-app@latest jasin
cd jasin
npx shadcn-ui@latest init
npm install tailwindcss @radix-ui/react-popover lucide-react @supabase/supabase-js
```

Add environment variables to `.env.local`:

```env
SUPABASE_URL=
SUPABASE_ANON_KEY=
AMZ_ACCESS_KEY=
AMZ_SECRET_KEY=
AMZ_ASSOC_TAG=getjasin-20
```

---

## 🛡️ Security Notes

- Amazon API keys are never exposed client-side
- All sensitive API calls happen on server-side via Vercel functions
- Anonymous users are sandboxed—no quota or card saving allowed
- RLS (Row Level Security) and middleware enforce plan limits

---

## ✅ Summary

Jasin makes it stupid-simple to turn Amazon links into branded embeds. Users can preview cards instantly—but saving and copying is gated behind signup, ensuring value and conversion. The free tier gives just enough to try it out, while the Pro plan unlocks what creators really want: speed, customization, and insights.



# Logging System Optimization

## Overview
The current logging system in [src/lib/logger.ts](mdc:src/lib/logger.ts) is causing 50-100ms cumulative overhead due to excessive console output, complex string interpolation, and debug logging enabled by default.

## Current Logging Issues

### 1. Excessive Console Output
**Problem**: Logging is enabled by default in production:
```typescript
// Current inefficient pattern in logger.ts
enabled: process.env.NODE_ENV !== 'production', // Always logs in dev
levels: {
  debug: process.env.NODE_ENV !== 'production', // Debug always on in dev
  info: true, // Info always on everywhere
}
```

**Impact**: Console operations are expensive, especially with complex objects

### 2. Complex String Operations
**Problem**: String interpolation and object serialization on every log call:
```typescript
// Expensive operations even when logging is disabled
console.log(`[INFO] ${module}: ${message}`, data !== undefined ? data : '');
```

**Impact**: CPU cycles wasted on string operations that may not be used

### 3. No Log Levels in Production
**Problem**: No way to selectively enable logging in production for debugging
**Impact**: Difficult to debug production issues

### 4. Synchronous Logging
**Problem**: All logging is synchronous and blocks execution
**Impact**: Adds latency to request processing

## Optimization Strategy

### 1. Implement Lazy Evaluation
Only perform expensive operations when logging is actually enabled:

```typescript
// src/lib/logger-optimized.ts
type LogLevel = 'debug' | 'info' | 'warn' | 'error';

interface LogEntry {
  level: LogLevel;
  module: string;
  message: string;
  data?: any;
  timestamp: number;
}

class OptimizedLogger {
  private static instance: OptimizedLogger;
  private enabled: boolean;
  private levels: Record<LogLevel, boolean>;
  private debugModules: Set<string>;
  
  private constructor() {
    this.enabled = process.env.LOGGER_ENABLED !== 'false';
    this.levels = {
      debug: false, // Disabled by default
      info: process.env.NODE_ENV !== 'production',
      warn: true,
      error: true
    };
    
    // Parse DEBUG env var
    const debugEnv = process.env.DEBUG || '';
    this.debugModules = new Set(debugEnv.split(',').filter(Boolean));
  }
  
  static getInstance(): OptimizedLogger {
    if (!OptimizedLogger.instance) {
      OptimizedLogger.instance = new OptimizedLogger();
    }
    return OptimizedLogger.instance;
  }
  
  // Lazy evaluation - only format when actually logging
  debug(module: string, message: string | (() => string), data?: any | (() => any)) {
    if (!this.shouldLog('debug', module)) return;
    
    const actualMessage = typeof message === 'function' ? message() : message;
    const actualData = typeof data === 'function' ? data() : data;
    
    this.log('debug', module, actualMessage, actualData);
  }
  
  info(module: string, message: string | (() => string), data?: any | (() => any)) {
    if (!this.shouldLog('info', module)) return;
    
    const actualMessage = typeof message === 'function' ? message() : message;
    const actualData = typeof data === 'function' ? data() : data;
    
    this.log('info', module, actualMessage, actualData);
  }
  
  private shouldLog(level: LogLevel, module: string): boolean {
    if (!this.enabled) return false;
    
    // Special case for debug - check module-specific enabling
    if (level === 'debug') {
      return this.debugModules.has(module) || this.debugModules.has('*');
    }
    
    return this.levels[level];
  }
  
  private log(level: LogLevel, module: string, message: string, data?: any) {
    // Use console methods directly for better performance
    const logFn = console[level] || console.log;
    
    if (data !== undefined) {
      logFn(`[${level.toUpperCase()}] ${module}: ${message}`, data);
    } else {
      logFn(`[${level.toUpperCase()}] ${module}: ${message}`);
    }
  }
}

export const logger = OptimizedLogger.getInstance();
```

### 2. Implement Asynchronous Logging
For non-critical logs, use asynchronous processing:

```typescript
// Async logging for non-critical messages
class AsyncLogger {
  private logQueue: LogEntry[] = [];
  private processing = false;
  
  async queueLog(entry: LogEntry) {
    this.logQueue.push(entry);
    
    if (!this.processing) {
      this.processing = true;
      // Process queue in next tick
      setTimeout(() => this.processQueue(), 0);
    }
  }
  
  private processQueue() {
    while (this.logQueue.length > 0) {
      const entry = this.logQueue.shift()!;
      this.writeLog(entry);
    }
    this.processing = false;
  }
  
  private writeLog(entry: LogEntry) {
    const logFn = console[entry.level] || console.log;
    logFn(`[${entry.level.toUpperCase()}] ${entry.module}: ${entry.message}`, entry.data);
  }
}
```

### 3. Conditional Debug Logging
Enable debug logging only for specific modules in production:

```typescript
// Environment-based debug control
export function createModuleLogger(moduleName: string) {
  return {
    debug: (message: string | (() => string), data?: any | (() => any)) => {
      logger.debug(moduleName, message, data);
    },
    info: (message: string | (() => string), data?: any | (() => any)) => {
      logger.info(moduleName, message, data);
    },
    warn: (message: string, data?: any) => {
      logger.warn(moduleName, message, data);
    },
    error: (message: string, error?: any) => {
      logger.error(moduleName, message, error);
    }
  };
}

// Usage in modules
const log = createModuleLogger('amazon');
log.debug(() => `Processing ASIN: ${asin}`, () => ({ asin, userId }));
```

### 4. Performance-Aware Logging
Add timing and performance context:

```typescript
// Performance-aware logging utilities
export class PerformanceLogger {
  private static timers = new Map<string, number>();
  
  static time(label: string) {
    this.timers.set(label, performance.now());
  }
  
  static timeEnd(label: string, module: string) {
    const start = this.timers.get(label);
    if (start) {
      const duration = performance.now() - start;
      this.timers.delete(label);
      
      if (duration > 100) { // Only log slow operations
        logger.warn(module, `Slow operation: ${label} took ${duration.toFixed(2)}ms`);
      } else {
        logger.debug(module, () => `${label} completed in ${duration.toFixed(2)}ms`);
      }
    }
  }
}

// Usage
PerformanceLogger.time('product-fetch');
const product = await getProduct(asin);
PerformanceLogger.timeEnd('product-fetch', 'amazon');
```

## Implementation Steps

### Step 1: Replace Current Logger
- Update [src/lib/logger.ts](mdc:src/lib/logger.ts) with optimized implementation
- Add lazy evaluation for expensive operations

### Step 2: Update All Log Calls
- Convert expensive log calls to lazy functions
- Remove unnecessary debug logging in hot paths

### Step 3: Add Performance Logging
- Implement timing utilities
- Add performance monitoring to critical paths

### Step 4: Configure Production Logging
- Set appropriate log levels for production
- Enable selective debug logging via environment variables

### Step 5: Add Log Aggregation
```typescript
// Optional: Send critical logs to external service
class LogAggregator {
  static async sendToService(level: LogLevel, module: string, message: string, data?: any) {
    if (level === 'error' || level === 'warn') {
      // Send to external logging service
      try {
        await fetch('/api/logs', {
          method: 'POST',
          body: JSON.stringify({ level, module, message, data, timestamp: Date.now() })
        });
      } catch {
        // Fail silently - don't break app due to logging issues
      }
    }
  }
}
```

## Configuration Examples

### Development Environment
```env
DEBUG=amazon,api:*,analytics
LOGGER_ENABLED=true
```

### Production Environment
```env
DEBUG=api:errors,stripe-webhook
LOGGER_ENABLED=true
```

### Performance Testing
```env
DEBUG=performance:*
LOGGER_ENABLED=true
```

## Performance Targets
- **Logging overhead**: < 5ms per request
- **Debug logging**: 0ms when disabled
- **Memory usage**: < 1MB for log buffers
- **CPU impact**: < 1% of total request processing

## Files to Modify
- [src/lib/logger.ts](mdc:src/lib/logger.ts) - Main optimization
- [src/lib/amazon.ts](mdc:src/lib/amazon.ts) - Update log calls
- [src/app/api/amazon/paapi/route.ts](mdc:src/app/api/amazon/paapi/route.ts) - Optimize logging
- [src/middleware.ts](mdc:src/middleware.ts) - Reduce logging overhead

## Migration Strategy
```typescript
// Before (expensive)
logger.debug('amazon', `Processing ASIN: ${asin}`, { asin, userId, timestamp: new Date() });

// After (optimized)
logger.debug('amazon', () => `Processing ASIN: ${asin}`, () => ({ asin, userId, timestamp: new Date() }));

// Or even better - only log when needed
if (process.env.DEBUG?.includes('amazon')) {
  logger.debug('amazon', `Processing ASIN: ${asin}`, { asin, userId });
}
```

## Testing Logging Performance
```typescript
// Benchmark logging performance
function benchmarkLogging() {
  const iterations = 10000;
  
  console.time('logging-disabled');
  for (let i = 0; i < iterations; i++) {
    logger.debug('test', 'Test message', { data: i });
  }
  console.timeEnd('logging-disabled');
  
  // Enable debug logging
  process.env.DEBUG = 'test';
  
  console.time('logging-enabled');
  for (let i = 0; i < iterations; i++) {
    logger.debug('test', 'Test message', { data: i });
  }
  console.timeEnd('logging-enabled');
}
```

## Expected Impact
- **Request processing time**: 50-100ms improvement
- **Memory usage**: 30-50% reduction in log-related allocations
- **CPU usage**: 20-30% reduction in string operations
- **Production debugging**: Better selective logging capabilities
- **Development experience**: Faster development server startup



# Logging System

## Overview

The Jasin application uses a structured logging system to manage debug and application logs in a consistent manner. This replaces the previous approach of direct `console.log` calls with emoji prefixes.

## Core Components

### Logger Utility

The logging system is implemented in [src/lib/logger.ts](mdc:src/lib/logger.ts), which provides consistent logging with environment-aware behavior:

- Debug logs are only shown in development by default
- All logging can be disabled or enabled based on configuration
- Specific log levels can be toggled individually
- Module-specific debug logs can be enabled in production

### Module Structure

Logs are organized by module for better filtering:

- `api:products` - Product API endpoints like [src/app/api/products/[asin]/route.ts](mdc:src/app/api/products/[asin]/route.ts)
- `api:product` - Product fetch API like [src/app/api/product/[asin]/route.ts](mdc:src/app/api/product/[asin]/route.ts)
- `api:parse` - Parse API endpoints
- `auth` - Authentication related logs
- `builder` - Card builder related logs

### Configuration

Logging behavior is controlled via environment variables in [next.config.js](mdc:next.config.js):

```js
env: {
  // Control logging behavior
  // In production, set DEBUG env var to enable specific modules
  // e.g. DEBUG=api:products,api:parse
  DEBUG: process.env.DEBUG || '',
},
```

## Log Levels

The logger provides four levels of logging:

1. `logger.debug()` - Development-only logs (unless enabled via DEBUG)
2. `logger.info()` - Normal informational logs
3. `logger.warn()` - Warning logs
4. `logger.error()` - Error logs

## Controlling Log Output

You can completely disable logging or selectively control which log levels are displayed using the `configure` method:

### Disable All Logging

To completely turn off all logging output in the console:

```javascript
import logger from '@/lib/logger';

// Turn off all logging
logger.configure({
  enabled: false
});
```

### Selectively Enable/Disable Log Levels

To keep only specific log levels (e.g., only show errors and warnings):

```javascript
import logger from '@/lib/logger';

// Selectively disable certain log levels
logger.configure({
  levels: {
    debug: false,  // Hide debug logs
    info: false,   // Hide info logs
    warn: true,    // Keep warnings
    error: true    // Keep errors
  }
});
```

### Enable Specific Modules in Production

To enable debugging for specific modules even in production:

```javascript
import logger from '@/lib/logger';

// Keep debug logs only for specific modules
logger.configure({
  debugModules: ['supabase', 'auth', 'api:products']
});
```

This configuration can be placed in your application bootstrap code or dynamically toggled based on user preferences.

## Migration Tools

A utility script [scripts/migrate-logs.js](mdc:scripts/migrate-logs.js) helps locate and replace direct console.log calls with structured logger calls.

## Documentation

Full documentation is available in [docs/LOGGING.md](mdc:docs/LOGGING.md).



# Middleware Optimization Guide

## Overview
The [middleware](mdc:src/middleware.ts) currently runs on EVERY request and causes 100-300ms overhead. It performs unnecessary authentication checks and database calls for public routes.

## Current Problems

### 1. Overly Broad Matcher
The middleware runs on almost all requests:
```typescript
matcher: ['/((?!_next/static|_next/image|favicon.ico|.*\\.(?:svg|png|jpg|jpeg|gif|webp)$).*)']
```

**Issues**:
- Runs on API routes that don't need auth
- Processes embed/preview requests unnecessarily
- Creates Supabase clients for public routes

### 2. Inefficient Route Checking
Current logic processes all routes then decides what to do:
```typescript
// Current inefficient pattern
const isProtected = PROTECTED_ROUTE_REGEX.test(pathname);
const isAdminRoute = pathname.startsWith(ADMIN_ROUTE_PREFIX);
const isEmbedOrPreview = pathname.startsWith('/api/embed/') || pathname.startsWith('/api/preview/');

// Still calls updateSession for non-protected routes
const { supabase, response } = await updateSession(request);
```

### 3. Unnecessary Database Calls
The `updateSession` function creates database connections even for routes that don't need them.

## Optimization Strategy

### 1. Early Return Pattern
Implement early returns to skip processing for routes that don't need middleware:

```typescript
export async function middleware(request: NextRequest) {
  const { pathname } = request.nextUrl;
  
  // Define routes that need NO middleware processing
  const PUBLIC_ROUTES = [
    '/api/embed/',
    '/api/preview/', 
    '/api/analytics/track',
    '/api/click/',
    '/',
    '/login',
    '/signup'
  ];
  
  // Early return for public routes
  if (PUBLIC_ROUTES.some(route => pathname.startsWith(route))) {
    return NextResponse.next();
  }
  
  // Only process protected routes
  if (pathname.startsWith('/dashboard') || 
      pathname.startsWith('/settings') || 
      pathname.startsWith('/analytics') ||
      pathname.startsWith('/admin')) {
    
    // Only NOW create Supabase client and check auth
    const { supabase, response } = await updateSession(request);
    // ... rest of auth logic
  }
  
  return NextResponse.next();
}
```

### 2. Optimize Security Headers
Move security headers to Next.js config instead of middleware:

**Current**: Headers added in middleware for every request
**Better**: Use [next.config.js](mdc:next.config.js) headers function

```typescript
// In next.config.js - more efficient
async headers() {
  return [
    {
      source: '/api/embed/:path*',
      headers: [
        { key: 'X-Frame-Options', value: 'SAMEORIGIN' },
        { key: 'X-Content-Type-Options', value: 'nosniff' }
      ]
    }
  ];
}
```

### 3. Route-Specific Middleware
Create separate middleware functions for different route types:

```typescript
// Separate functions for different needs
async function handleProtectedRoute(request: NextRequest) {
  // Auth logic only
}

async function handleAdminRoute(request: NextRequest) {
  // Auth + admin check
}

async function handleApiRoute(request: NextRequest) {
  // Minimal processing
}
```

## Implementation Steps

### Step 1: Refactor Middleware Structure
Update [src/middleware.ts](mdc:src/middleware.ts) with early return pattern.

### Step 2: Move Security Headers
Move static security headers to [next.config.js](mdc:next.config.js).

### Step 3: Optimize Matcher
Update matcher to exclude more routes:
```typescript
export const config = {
  matcher: [
    '/dashboard/:path*',
    '/settings/:path*', 
    '/analytics/:path*',
    '/admin/:path*',
    '/api/auth/:path*'
  ]
}
```

### Step 4: Add Performance Monitoring
```typescript
export async function middleware(request: NextRequest) {
  const start = Date.now();
  
  // ... middleware logic
  
  const duration = Date.now() - start;
  if (duration > 100) {
    console.warn(`Slow middleware: ${duration}ms for ${request.nextUrl.pathname}`);
  }
}
```

## Performance Targets
- Public routes: 0ms middleware overhead
- Protected routes: < 50ms middleware processing
- Admin routes: < 100ms middleware processing

## Files to Modify
- [src/middleware.ts](mdc:src/middleware.ts) - Main optimization
- [next.config.js](mdc:next.config.js) - Move security headers
- [src/utils/supabase/middleware.ts](mdc:src/utils/supabase/middleware.ts) - Optimize updateSession

## Testing
```typescript
// Add timing to middleware
const middlewareStart = performance.now();
// ... middleware logic
const middlewareDuration = performance.now() - middlewareStart;
console.log(`Middleware took ${middlewareDuration}ms for ${pathname}`);
```

## Expected Impact
- **Public route performance**: 100-300ms improvement
- **Protected route performance**: 50-100ms improvement  
- **Overall app responsiveness**: 20-30% faster
- **Reduced database connections**: 70% fewer unnecessary connections



# 💸 Upgrade Path: Free vs Paid Plans (Jasin MVP)

---

## 🆓 Free Plan (Account Required)

### 🔒 Limits:
- 5 credits **total** (lifetime)
- Uses default affiliate tag: `getjasin-20`
- 48-hour product cache (ASIN → product info)

### 🎨 Features:
- Card preview + copy (after signup)
- Basic theme only
- Basic view + click counts (per embed)
- Must show "Powered by Jasin" badge
- No analytics
- Cannot save cards anonymously — must sign up first

---

## 💳 Paid Plan (Pro Tier)

### 🔓 Limits:
- 25 credits **per month**
- Supports **custom affiliate tags**
- 48-hour cache (same as free)

### 🧰 Features:
- All free plan features
- Save unlimited cards to dashboard
- Full access to theme library
- **Analytics Dashboard**:
  - Views + Clicks per embed
  - CTR performance
  - Top products by engagement
  - Traffic and referral trends

/* Brand customization features temporarily removed for initial launch
- Brand color customization
*/

---

## 🧠 Supabase + Stripe Integration

### 👤 User Management
- Each user is identified by `auth.uid()`
- Supabase `profiles` table tracks:
  - `subscription_tier` (`free` or `pro`)
  - `stripe_customer_id`
  - `email`, `created_at`

### 💳 Subscription Logic
- Stripe checkout on frontend → webhook to backend
- On successful payment:
  - Webhook updates `profiles.subscription_tier = 'pro'`
- On cancel or fail:
  - Webhook reverts user to `'free'`

### 🔐 RLS Access Control

Use Row Level Security to gate features:

```sql
-- Example: Free users limited to 5 total credits
CREATE POLICY "Limit free embeds" ON embeds
FOR INSERT TO authenticated
WITH CHECK (
  (SELECT subscription_tier FROM profiles WHERE id = auth.uid()) = 'free'
  AND (
    SELECT COUNT(*) FROM embeds
    WHERE user_id = auth.uid()
  ) < 5
);
```

Paid users are not restricted in the same way:
- You can use a similar policy or enforce plan limits in API logic for more flexibility (rolling quota, monthly resets, etc.)

---

## ✅ Summary

- All embed usage is **auth-gated**
- Free users have a clear value ceiling (5 credits, no analytics)
- Paid users unlock full customization, analytics, and branding
- Stripe + Supabase offer seamless integration to scale this model



# Amazon PAAPI Integration Optimization

## Overview
The Amazon PAAPI integration in [src/lib/amazon.ts](mdc:src/lib/amazon.ts) is causing 2-5 second delays for uncached product requests due to inefficient patterns and lack of connection pooling.

## Critical Performance Issues

### 1. SDK Initialization on Every Request
**Problem**: The PAAPI SDK is initialized on every request:
```typescript
// Current inefficient pattern in amazon.ts
async function initializePaapiSdk() {
  if (sdkInitialized) return { success: true };
  // ... initialization logic runs frequently
}
```

**Impact**: 200-500ms per request for SDK setup

### 2. Synchronous Internal API Calls
**Problem**: The [getProduct function](mdc:src/lib/amazon.ts) makes synchronous fetch calls to its own API:
```typescript
// Inefficient internal API call pattern
const response = await fetch(`${baseUrl}/api/paapi`, {
  method: 'POST',
  // ... synchronous call to own API
});
```

**Impact**: Double the latency (client → server → PAAPI → server → client)

### 3. No Connection Pooling
**Problem**: Each PAAPI request creates a new connection
**Impact**: Additional 500-1000ms for connection establishment

### 4. Complex Retry Logic
**Problem**: Multiple retry mechanisms with exponential backoff in [retryDatabaseInsert](mdc:src/lib/amazon.ts)
**Impact**: Can add 2-5 seconds on database errors

## Optimization Strategy

### 1. Implement Singleton PAAPI Client
Create a persistent, reusable PAAPI client:

```typescript
// New optimized pattern
class PaapiClient {
  private static instance: PaapiClient;
  private client: any;
  private initialized: boolean = false;
  
  private constructor() {}
  
  static getInstance(): PaapiClient {
    if (!PaapiClient.instance) {
      PaapiClient.instance = new PaapiClient();
    }
    return PaapiClient.instance;
  }
  
  async initialize() {
    if (this.initialized) return;
    
    // Initialize once and reuse
    const ProductAdvertisingAPIv1 = await import('paapi5-nodejs-sdk');
    this.client = new ProductAdvertisingAPIv1.DefaultApi();
    // ... setup
    this.initialized = true;
  }
  
  async getItems(asins: string[]) {
    await this.initialize();
    return this.client.getItems(/* ... */);
  }
}
```

### 2. Eliminate Internal API Calls
**Current**: [getProduct](mdc:src/lib/amazon.ts) → `/api/paapi` → PAAPI
**Optimized**: [getProduct](mdc:src/lib/amazon.ts) → PAAPI directly

```typescript
// Remove the internal fetch call and call PAAPI directly
export async function getProduct(asin: string, affiliateTag?: string, userId?: string) {
  // Skip the internal API call
  const paapiClient = PaapiClient.getInstance();
  const result = await paapiClient.getItems([asin]);
  // ... process result
}
```

### 3. Implement Request Batching
Batch multiple ASIN requests into single PAAPI calls:

```typescript
class PaapiRequestBatcher {
  private pendingRequests: Map<string, Promise<any>> = new Map();
  private batchTimeout: NodeJS.Timeout | null = null;
  
  async getProduct(asin: string): Promise<any> {
    // Check if request is already pending
    if (this.pendingRequests.has(asin)) {
      return this.pendingRequests.get(asin);
    }
    
    // Create batched request
    const promise = this.createBatchedRequest(asin);
    this.pendingRequests.set(asin, promise);
    
    return promise;
  }
  
  private async createBatchedRequest(asin: string) {
    // Batch multiple requests together
    // Wait 50ms to collect more requests
    await new Promise(resolve => setTimeout(resolve, 50));
    
    const asins = Array.from(this.pendingRequests.keys());
    const results = await PaapiClient.getInstance().getItems(asins);
    
    // Distribute results to pending promises
    // ... implementation
  }
}
```

### 4. Optimize Database Operations
Replace the complex retry logic with simpler error handling:

```typescript
// Simplified database insert
async function storeProductData(productData: any) {
  try {
    const { error } = await getSupabaseClient()
      .from('products')
      .upsert(productData, { onConflict: 'asin' });
    
    if (error) {
      logger.error('amazon', 'Database insert failed:', error.message);
      // Don't retry - just log and continue
    }
  } catch (error) {
    logger.error('amazon', 'Database operation failed:', error);
  }
}
```

### 5. Add Connection Pooling
Configure HTTP agent with connection pooling:

```typescript
import { Agent } from 'https';

const httpsAgent = new Agent({
  keepAlive: true,
  maxSockets: 10,
  maxFreeSockets: 5,
  timeout: 30000
});

// Use in PAAPI client configuration
```

## Implementation Steps

### Step 1: Create Singleton PAAPI Client
- Refactor [src/lib/amazon.ts](mdc:src/lib/amazon.ts) to use singleton pattern
- Remove SDK initialization from every request

### Step 2: Eliminate Internal API Calls  
- Remove the fetch call to `/api/paapi` in [getProduct](mdc:src/lib/amazon.ts)
- Call PAAPI directly from the library

### Step 3: Implement Request Batching
- Add batching logic for multiple ASIN requests
- Optimize for common patterns (dashboard loading multiple products)

### Step 4: Simplify Database Operations
- Remove complex retry logic in [retryDatabaseInsert](mdc:src/lib/amazon.ts)
- Use simple upsert operations

### Step 5: Add Performance Monitoring
```typescript
// Add timing to PAAPI calls
const start = performance.now();
const result = await paapiClient.getItems([asin]);
const duration = performance.now() - start;
logger.info('paapi-performance', `PAAPI call took ${duration}ms for ${asin}`);
```

## Performance Targets
- **Cached product lookup**: < 50ms
- **Uncached product lookup**: < 500ms (down from 2-5s)
- **Batch product lookup**: < 800ms for 5 products
- **SDK initialization**: One-time only

## Files to Modify
- [src/lib/amazon.ts](mdc:src/lib/amazon.ts) - Main optimization
- [src/app/api/amazon/paapi/route.ts](mdc:src/app/api/amazon/paapi/route.ts) - Simplify or remove
- [src/app/api/parse/route.ts](mdc:src/app/api/parse/route.ts) - Update to use optimized calls

## Testing
```typescript
// Performance testing
console.time('product-lookup');
const product = await getProduct('B0DGHYDZSD');
console.timeEnd('product-lookup');

// Batch testing  
console.time('batch-lookup');
const products = await Promise.all([
  getProduct('ASIN1'),
  getProduct('ASIN2'), 
  getProduct('ASIN3')
]);
console.timeEnd('batch-lookup');
```

## Expected Impact
- **Product lookup speed**: 4-10x faster
- **Dashboard loading**: 2-3x faster when loading multiple products
- **API response times**: 60-80% improvement
- **Resource usage**: 50% fewer connections to Amazon



# PAAPI Rate Limiting & Circuit Breaker System

## Overview

This document describes the comprehensive rate limiting and circuit breaker system implemented to protect against Amazon PAAPI quota exhaustion and service failures.

## Problem Statement

The original concern was valid: **"Server-only PAAPI calls + 48h cache strikes a good TOS/rate-limit balance, but you'll still burst if a creator with heavy traffic embeds a card that isn't cached. Add a rolling per-embed fetch throttle or global circuit-breaker tied to the paapi_calls table."**

Without proper rate limiting, a viral embed could trigger hundreds of simultaneous PAAPI calls, quickly exhausting daily quotas and potentially violating Amazon's terms of service.

## Solution Architecture

### 🛡️ Multi-Layer Protection

1. **Global Rate Limiting** - Tracks daily, hourly, and minute-level usage
2. **Circuit Breaker** - Stops requests when service is failing
3. **Per-ASIN Throttling** - Prevents rapid repeated requests for same product
4. **Graceful Degradation** - Falls back to stale cache when limits hit
5. **Admin Monitoring** - Real-time visibility into system health

### 📊 Rate Limits (Conservative)

| Period | Limit | Threshold | Purpose |
|--------|-------|-----------|---------|
| Daily | 8,640 | 85% (7,344) | Amazon's hard limit |
| Hourly | 360 | 80% (288) | Burst protection |
| Minute | 6 | 75% (4.5) | Spike protection |
| Per-ASIN | 30s cooldown | N/A | Duplicate prevention |

### 🔄 Circuit Breaker States

- **CLOSED** - Normal operation, requests allowed
- **OPEN** - Service failing, requests blocked (5 min timeout)
- **HALF_OPEN** - Testing recovery, limited requests

## Implementation Details

### Core Components

#### 1. Rate Limiter (`src/lib/paapi-rate-limiter.ts`)
```typescript
const rateLimiter = getPaapiRateLimiter();
const check = await rateLimiter.checkRateLimit(asin, userId);

if (!check.allowed) {
  // Handle rate limiting
  return 429 response with retry-after
}
```

#### 2. PAAPI Route Integration
- **Before**: Direct PAAPI calls without protection
- **After**: Rate limiting + circuit breaker + graceful fallback

```typescript
// Rate limit check BEFORE PAAPI call
const rateLimitCheck = await rateLimiter.checkRateLimit(asin, userId);
if (!rateLimitCheck.allowed) {
  return 429 with retry info
}

// Record success/failure for circuit breaker
rateLimiter.recordSuccess(); // or recordFailure(error)
```

#### 3. Graceful Degradation
When rate limits are hit:
1. **First**: Try to return stale cached data
2. **Second**: Show user-friendly error with retry time
3. **Never**: Fail silently or lose user requests

```typescript
if (response.status === 429) {
  // Fall back to stale cache if available
  if (cachedProduct) {
    return {
      product: cachedProduct,
      isStale: true,
      rateLimited: true
    };
  }
  // Otherwise throw informative error
}
```

### User Experience

#### Rate Limit Notice Component
```tsx
<RateLimitNotice 
  type="stale_cache"
  reason="High demand - showing cached data"
/>
```

Displays user-friendly messages:
- **Rate Limited**: "High demand, try again in X minutes"
- **Stale Cache**: "Showing cached data, prices may not be current"
- **Service Error**: "Amazon service unavailable, using cached data"

### Admin Monitoring

#### Dashboard Integration
- Circuit breaker state (CLOSED/OPEN/HALF_OPEN)
- Current usage percentages
- Recent failures
- ASIN throttle map size

#### API Endpoint
```
GET /api/admin/rate-limiter-status
```

Returns comprehensive status for debugging and monitoring.

## Database Integration

### Enhanced Logging
All PAAPI calls are logged with additional context:
- `request_type`: 'GetItems', 'GetItems_RateLimited', 'GetItems_ServiceError'
- `cache_hit`: true/false
- `success`: true/false
- `request_details`: Additional metadata

### Usage Tracking
Rate limits are enforced based on real database counts:
```sql
SELECT COUNT(*) FROM paapi_calls 
WHERE cache_hit = false 
  AND success = true 
  AND created_at >= start_of_day;
```

## Configuration

### Environment Variables
No additional environment variables required - uses existing Supabase configuration.

### Tunable Parameters
```typescript
const RATE_LIMITS = {
  DAILY_LIMIT: 8640,        // Amazon's limit
  HOURLY_LIMIT: 360,        // Conservative hourly
  MINUTE_LIMIT: 6,          // Conservative minute
  DAILY_THRESHOLD: 0.85,    // 85% of daily limit
  HOURLY_THRESHOLD: 0.80,   // 80% of hourly limit
  MINUTE_THRESHOLD: 0.75,   // 75% of minute limit
  ASIN_COOLDOWN_MS: 30000,  // 30 seconds between same ASIN
  FAILURE_THRESHOLD: 5,     // Failures before circuit opens
  RECOVERY_TIMEOUT_MS: 300000 // 5 minutes recovery time
};
```

## Benefits

### 🛡️ Protection
- **Quota Safety**: Never exceed Amazon's limits
- **Service Resilience**: Circuit breaker prevents cascade failures
- **Burst Protection**: Minute-level limits prevent spikes

### 📈 Performance
- **Graceful Degradation**: Users get stale data instead of errors
- **Smart Caching**: Prioritizes cache hits during high demand
- **Efficient Throttling**: Per-ASIN cooldowns prevent duplicate requests

### 🔍 Observability
- **Real-time Monitoring**: Admin dashboard shows current state
- **Detailed Logging**: All rate limiting events are logged
- **Usage Analytics**: Track patterns and optimize limits

### 💰 Cost Control
- **Predictable Usage**: Conservative limits prevent surprise overages
- **Efficient API Usage**: Throttling reduces unnecessary calls
- **Cache Optimization**: Extends cache lifetime during high demand

## Testing & Validation

### Load Testing
The system should be tested with:
1. **Burst Traffic**: 100+ simultaneous requests for uncached ASINs
2. **Sustained Load**: Continuous requests over hours
3. **Failure Scenarios**: Simulate Amazon API failures
4. **Recovery Testing**: Verify circuit breaker recovery

### Monitoring Alerts
Set up alerts for:
- Circuit breaker state changes
- Usage approaching thresholds (>80% of limits)
- High failure rates
- Unusual traffic patterns

## Migration & Rollout

### Phase 1: Deploy (✅ Complete)
- Rate limiter implementation
- PAAPI route integration
- Admin monitoring

### Phase 2: Monitor
- Watch usage patterns
- Tune thresholds if needed
- Validate graceful degradation

### Phase 3: Optimize
- Adjust cache TTL based on rate limiting
- Implement priority queuing for Pro users
- Add predictive scaling

## Emergency Procedures

### Circuit Breaker Stuck Open
```bash
# Check status
curl /api/admin/rate-limiter-status

# Manual reset (if implemented)
curl -X POST /api/admin/rate-limiter-status \
  -d '{"action": "reset_circuit_breaker"}'
```

### Quota Exhaustion
1. **Immediate**: Circuit breaker will open automatically
2. **Short-term**: All requests fall back to cached data
3. **Long-term**: Quota resets at midnight UTC

### False Positives
If rate limiting is too aggressive:
1. Check admin dashboard for usage patterns
2. Adjust thresholds in `RATE_LIMITS` configuration
3. Monitor for 24 hours after changes

## Conclusion

This comprehensive rate limiting system addresses the original concern by providing:

✅ **Global circuit breaker** tied to paapi_calls table  
✅ **Rolling per-embed fetch throttle** (per-ASIN cooldowns)  
✅ **Graceful degradation** with stale cache fallback  
✅ **Real-time monitoring** and alerting  
✅ **User-friendly error handling**  

The system protects against quota exhaustion while maintaining excellent user experience through intelligent fallback mechanisms.


# Performance Optimization Plan

## Overview
This rule provides a systematic approach to implementing the performance optimizations identified in the efficiency audit. The current app rating is **6.5/10** with target improvement to **8.5/10**.

## Critical Performance Issues Identified

### 🔴 High Priority (Weeks 1-2)
1. **Database Query Inefficiencies** - Multiple round trips, missing indexes
2. **Middleware Processing Overhead** - Runs on every request unnecessarily  
3. **Amazon PAAPI Integration Bottlenecks** - Synchronous calls, no connection pooling

### 🟡 Medium Priority (Weeks 3-4)
4. **Anti-Abuse System Overhead** - Complex calculations on each request
5. **Logging System Performance Impact** - Excessive console output
6. **Frontend Optimization** - Missing loading states, no memoization

## Implementation Timeline

### Week 1: Database Optimization
- [ ] Add missing database indexes
- [ ] Consolidate dashboard queries into single function
- [ ] Implement materialized views for analytics
- [ ] Add query performance monitoring

### Week 2: Middleware & Caching
- [ ] Optimize middleware to skip unnecessary processing
- [ ] Implement Redis caching layer
- [ ] Add response time monitoring
- [ ] Cache dashboard and product data

### Week 3: PAAPI Integration
- [ ] Implement connection pooling
- [ ] Add request batching for multiple ASINs
- [ ] Optimize SDK initialization
- [ ] Add circuit breaker improvements

### Week 4: Frontend & Monitoring
- [ ] Add React.memo to expensive components
- [ ] Implement proper loading states
- [ ] Add performance metrics collection
- [ ] Set up monitoring dashboards

## Expected Performance Improvements
- **Dashboard load time**: 3-5s → 500ms-1s
- **Product lookup**: 2-5s → 200-500ms  
- **API response times**: 1-3s → 100-300ms
- **Overall app responsiveness**: 2-3x improvement

## Key Files to Modify
- [src/middleware.ts](mdc:src/middleware.ts) - Middleware optimization
- [src/lib/amazon.ts](mdc:src/lib/amazon.ts) - PAAPI integration
- [src/app/dashboard/page.tsx](mdc:src/app/dashboard/page.tsx) - Dashboard queries
- [supabase/migrations/](mdc:supabase/migrations) - Database indexes
- [src/lib/logger.ts](mdc:src/lib/logger.ts) - Logging optimization

## Success Metrics
- Page load times under 1 second
- API response times under 300ms
- Database queries under 100ms
- Zero timeout errors
- Improved user experience scores




# 📄 Product Requirements Document (Jasin MVP)

---

## ✨ Overview

Jasin is a SaaS tool that transforms plain Amazon affiliate links into clean, embeddable product cards. Users paste a link or ASIN and preview a beautiful card that fetches and formats product data via Amazon PAAPI. **Embed copying requires an account**. Saved cards, analytics, and branding features are unlocked through a paid subscription.

---

## 🧠 Key Goals

- Dead-simple UX (paste → preview → embed)
- Card preview available without login
- Embed creation gated behind signup
- Add affiliate tag if user does not provide one
- Efficient PAAPI usage through Supabase caching

---

## 🎯 Core User Flow

1. User visits homepage
2. Pastes Amazon URL or ASIN
3. System extracts:
   - ASIN
   - Affiliate tag (if present)
4. Check Supabase `products` cache
5. If not found or expired, call Amazon PAAPI
6. Normalize and render product card (title, image, price, CTA)
7. User sees preview (no login required)
8. When they click "Copy Embed":
   - Prompt to **sign up or log in**
   - On success:
     - Embed saved to user account
     - Embed code copied
9. Authenticated users can visit `/dashboard` to manage cards

---

## 🔐 Authentication

- **Supabase Email + Password Auth**
- No magic link login (removed)
- Required for copying or saving embeds
- Anonymous users can only preview cards

---

## 💾 Caching & Rate Limits

- Product data cached in `products` table by ASIN
- TTL-based invalidation (~24–48h)
- PAAPI is only called on cache miss
- Daily PAAPI quota: ~8640 requests/account
- Credit limits per plan:
  - Free: 5 lifetime
  - Paid: 25/month

---

## 🧱 Database Tables

- `products` – Cached product data from Amazon API
- `embeds` – User-created product cards (FK: `user_id`)
- `profiles` – Tracks email, subscription tier, Stripe ID

---

## 💡 MVP Constraints

- No JS embed—only HTML snippet
- No affiliate credential input from users (yet)
- No dynamic product comparison blocks
- No card editing after save
- Limited themes in free plan

---

## ✅ Summary

Jasin is optimized for creators who want fast, branded affiliate embeds without writing HTML. It balances low-friction previews with secure and gated embed generation. Paid plans unlock customization and analytics, while usage is optimized to stay under strict Amazon PAAPI limits.



# Amazon Price Update System

## Overview
To comply with Amazon's Terms of Service, product prices must be updated at least every 24 hours. Our system does this efficiently by only fetching and updating price-related information rather than pulling full product data.

## System Components

1. **Database Function**: [update_product_prices()](mdc:supabase/migrations/20240601_add_product_price_update_job.sql) identifies products that haven't been updated within the required timeframe
2. **Queue System**: Updates are queued in the `paapi_calls` table
3. **Cron Job**: Configured in [vercel.json](mdc:vercel.json) to run every 4 hours
4. **API Endpoint**: [update-prices-efficient](mdc:src/app/api/cron/update-prices-efficient/route.ts) processes the queue

## Amazon API Integration

The system uses two specialized functions for efficient price updates:
- [getProductPriceOnly](mdc:src/lib/amazon.ts) - Only requests price-related fields from PAAPI
- [extractPriceDataOnly](mdc:src/lib/amazon.ts) - Extracts just the price information from the API response

## Testing

For manual testing, use the [test-efficient-price-update.js](mdc:test-efficient-price-update.js) script.

## Documentation

For detailed explanation of the system, see [PRICE_UPDATE_SYSTEM.md](mdc:docs/PRICE_UPDATE_SYSTEM.md).



# Security Audit: Preview vs Save Separation

## Overview

This rule documents the security improvements implemented to address the critical concern: **"Make sure 'preview' never leaks embed HTML in dev tools; a quick audit of /card/[id] response headers for CSP + cache-control is worth it."**

## Problem Statement

The original implementation had several security vulnerabilities:

1. **Data Leakage**: The `/api/embed/[id]` endpoint returned complete embed data to anyone, regardless of authentication
2. **Dev Tools Exposure**: Users could inspect network requests and see full product data, affiliate tags, and analytics settings
3. **Missing Security Headers**: No Content Security Policy or cache control headers
4. **No Access Control**: Preview mode had same access as authenticated users

## Solution Implementation

### 1. Strict Access Control in [/api/embed/[id]](mdc:src/app/api/embed/[id]/route.ts)

The endpoint now implements authentication-based access control:

```typescript
// Check authentication status
const { data: { user } } = await supabase.auth.getUser();
const isAuthenticated = !!user;

// Check if user owns this embed (for full access)
const isOwner = isAuthenticated && user.id === embed.user_id;

// For preview mode, check if this is a preview embed
const isPreviewEmbed = embed.is_preview === true;

// Determine access level
const hasFullAccess = isOwner || isPreviewEmbed;
const hasLimitedAccess = !isAuthenticated || previewMode;
```

**Access Levels**:
- **Full Access**: Authenticated embed owners or preview embeds
- **Limited Access**: Unauthenticated users (preview mode only)
- **No Access**: Returns 403 error

### 2. Dedicated Preview API Endpoint

Created [/api/preview/[id]](mdc:src/app/api/preview/[id]/route.ts) that only returns minimal data needed for visual preview:

**Data Restrictions**:
- ❌ No affiliate tags exposed
- ❌ No analytics settings
- ❌ No specifications or detailed descriptions
- ❌ Limited to 3 gallery images and features
- ✅ Only basic product info for visual preview

### 3. Comprehensive Security Headers

Security headers are implemented in [middleware.ts](mdc:src/middleware.ts):

```typescript
// Add strict security headers for embed and preview endpoints
response.headers.set('X-Content-Type-Options', 'nosniff')
response.headers.set('X-Frame-Options', 'SAMEORIGIN')
response.headers.set('Referrer-Policy', 'strict-origin-when-cross-origin')
response.headers.set('X-Robots-Tag', 'noindex, nofollow, nosnippet, noarchive')

// Add CSP to prevent script injection in responses
response.headers.set('Content-Security-Policy', 
  "default-src 'none'; script-src 'self'; style-src 'self' 'unsafe-inline'; img-src 'self' data: https://m.media-amazon.com; connect-src 'self'")

// Vary on authentication headers to ensure proper caching
response.headers.set('Vary', 'Authorization, Cookie')

// For preview endpoints, add even stricter headers
if (request.nextUrl.pathname.startsWith('/api/preview/')) {
  response.headers.set('X-Frame-Options', 'DENY')
  response.headers.set('Referrer-Policy', 'no-referrer')
  response.headers.set('Cache-Control', 'private, no-cache, no-store, must-revalidate')
}
```

**Security Headers Explained**:
- `X-Content-Type-Options: nosniff` - Prevents MIME type sniffing
- `X-Frame-Options: SAMEORIGIN/DENY` - Prevents clickjacking
- `Referrer-Policy: strict-origin-when-cross-origin/no-referrer` - Controls referrer information
- `X-Robots-Tag: noindex, nofollow` - Prevents search engine indexing
- `Content-Security-Policy` - Prevents script injection and data exfiltration
- `Vary: Authorization, Cookie` - Ensures proper cache separation by auth state

### 4. Cache Control Strategy

**Different cache strategies based on access level**:

```typescript
// Determine cache headers based on access level
const cacheHeaders = hasFullAccess 
  ? 'public, s-maxage=600, stale-while-revalidate=1800' // Full data: 10min cache
  : 'public, s-maxage=300, stale-while-revalidate=900'; // Preview: 5min cache
```

**Preview endpoints**: `private, no-cache, no-store, must-revalidate` - No caching at all

### 5. Updated Preview Page

The [preview page](mdc:src/app/preview/page.tsx) now uses the secure preview endpoint:

```typescript
// Fetch data for an existing embed using the secure preview endpoint
const response = await fetch(`/api/preview/${identifier}`)

// For ASIN previews, also use the secure endpoint
const embedResponse = await fetch(`/api/preview/${data.embedId}`)
```

## Security Benefits

### 🛡️ Data Protection
- **No Sensitive Data Leakage**: Affiliate tags, analytics settings, and detailed specs are never exposed in preview mode
- **Limited Data Exposure**: Preview mode only shows minimal data needed for visual representation
- **Authentication-Based Access**: Full data only available to authenticated embed owners

### 🔒 Network Security
- **Strict CSP**: Prevents any script execution or data exfiltration from API responses
- **No-Cache for Previews**: Prevents sensitive data from being cached in browsers or CDNs
- **Proper Vary Headers**: Ensures authenticated and unauthenticated responses are cached separately

### 🚫 Attack Prevention
- **No Clickjacking**: X-Frame-Options prevents embedding in malicious iframes
- **No MIME Sniffing**: Prevents browsers from misinterpreting response content
- **No Search Indexing**: Prevents search engines from indexing sensitive API responses

### 🔍 Dev Tools Protection
- **Limited Network Inspection**: Users can only see minimal preview data in dev tools
- **No Affiliate Tag Exposure**: Affiliate information is completely hidden in preview mode
- **No Analytics Leakage**: Analytics settings and tracking information are not exposed

## Testing Verification

### Manual Testing Steps

1. **Unauthenticated Preview**:
   ```bash
   curl -H "Accept: application/json" https://getjasin.com/api/preview/[embed-id]
   ```
   Should return limited data without affiliate tags or analytics settings.

2. **Authenticated Full Access**:
   ```bash
   curl -H "Accept: application/json" -H "Cookie: [auth-cookies]" https://getjasin.com/api/embed/[embed-id]
   ```
   Should return complete data for owned embeds.

3. **Security Headers Check**:
   ```bash
   curl -I https://getjasin.com/api/preview/[embed-id]
   ```
   Should include all security headers listed above.

### Browser Dev Tools Test

1. Open browser dev tools
2. Navigate to preview page
3. Check Network tab for API calls
4. Verify that only limited data is visible in responses
5. Confirm no affiliate tags or sensitive settings are exposed

## Compliance

This implementation ensures:

✅ **Preview mode never leaks embed HTML** - Separate endpoints with different data exposure levels  
✅ **Proper CSP headers** - Strict Content Security Policy prevents data exfiltration  
✅ **Appropriate cache control** - Different caching strategies for different access levels  
✅ **Authentication-based access control** - Full data only for authenticated owners  
✅ **Dev tools protection** - Limited data exposure in network inspection  

## Future Improvements

1. **Rate Limiting**: Add rate limiting to preview endpoints to prevent abuse
2. **Audit Logging**: Log access attempts to sensitive endpoints
3. **Token-Based Preview**: Consider using temporary tokens for preview access
4. **Watermarking**: Add visual watermarks to preview mode to indicate limited functionality


# Security Patterns and Best Practices

## Core Security Principles

### 1. Fail-Safe Defaults
- Systems should fail securely when errors occur
- Default to denying access rather than allowing
- Graceful degradation without exposing sensitive data

```typescript
// Example: Anti-abuse system fails open for legitimate users
export async function checkUserBlocked(userId: string): Promise<boolean> {
  try {
    // Check for abuse signals
    const { data, error } = await supabase.from('abuse_signals')...
    
    if (error) {
      logger.error('Error checking user status:', error);
      return false; // Fail open - don't block legitimate users
    }
    
    return hasAbuseSignals;
  } catch (error) {
    return false; // Always fail open on exceptions
  }
}
```

### 2. Defense in Depth
Multiple layers of security controls:
- **Authentication**: Supabase Auth with JWT tokens
- **Authorization**: Row Level Security (RLS) policies
- **Input Validation**: Server-side validation on all endpoints
- **Rate Limiting**: Built into anti-abuse system
- **Audit Logging**: Comprehensive activity tracking

### 3. Principle of Least Privilege
- API endpoints only return necessary data
- Database policies restrict access to minimum required
- Admin functions isolated and access-controlled

## Authentication & Authorization Patterns

### API Route Protection
```typescript
export async function GET/POST(request: NextRequest) {
  const supabase = createClient();
  
  // Always verify authentication first
  const { data: { user }, error } = await supabase.auth.getUser();
  
  if (error || !user) {
    return NextResponse.json(
      { error: 'Authentication required' },
      { status: 401 }
    );
  }
  
  // Proceed with authorized operations
}
```

### Row Level Security (RLS)
All sensitive tables use RLS policies:
```sql
-- Example: Anti-abuse tables are admin-only
CREATE POLICY "Admin access only" ON public.abuse_signals
  FOR ALL TO authenticated
  USING (false); -- Only service role can access

-- Example: User data is self-accessible only
CREATE POLICY "Users can view own data" ON public.profiles
  FOR SELECT TO authenticated
  USING (id = auth.uid());
```

### Admin Access Control
```typescript
// Consistent admin check pattern
const adminEmails = ['matt@getjasin.com', 'admin@getjasin.com'];

const { data: profile } = await supabase
  .from('profiles')
  .select('email')
  .eq('id', user.id)
  .single();

if (!profile || !adminEmails.includes(profile.email)) {
  redirect('/dashboard'); // Redirect non-admins
}
```

## Input Validation & Sanitization

### Email Validation
```typescript
const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
if (!emailRegex.test(email)) {
  return NextResponse.json(
    { error: 'Invalid email format' },
    { status: 400 }
  );
}
```

### ASIN Validation
```typescript
if (!asin || typeof asin !== 'string' || asin.length !== 10) {
  return NextResponse.json(
    { error: 'Invalid ASIN format' },
    { status: 400 }
  );
}
```

### SQL Injection Prevention
- Always use parameterized queries
- Supabase client handles escaping automatically
- Never concatenate user input into SQL strings

## Data Protection Patterns

### Sensitive Data Logging
```typescript
// Log partial information for debugging without exposing PII
logger.info('User action', {
  userId: userId.substring(0, 8) + '***',
  email: email.split('@')[0] + '@***',
  ip: deviceFingerprint.ip_address
});
```

### Error Message Security
```typescript
// Generic error messages to avoid information disclosure
if (riskAssessment.should_block) {
  return NextResponse.json(
    { error: 'Unable to create account at this time. Please try again later or contact support.' },
    { status: 429 }
  );
}
```

### Data Minimization
```typescript
// Only return necessary fields
const { data: profile } = await supabase
  .from('profiles')
  .select('subscription_tier, is_admin') // Only what's needed
  .eq('id', user.id)
  .single();
```

## Anti-Abuse Security Patterns

### Device Fingerprinting
```typescript
export function extractDeviceFingerprint(request: Request): DeviceFingerprint {
  const headers = request.headers;
  
  // Extract IP with proxy handling
  const ipHeader = headers.get('x-forwarded-for');
  const ip = ipHeader ? ipHeader.split(',')[0].trim() : 
             headers.get('x-real-ip') || 'unknown';
  
  return {
    user_agent: headers.get('user-agent') || 'unknown',
    ip_address: ip,
    // ... other fingerprint data
  };
}
```

### Risk-Based Blocking
```typescript
// Progressive risk assessment
let riskScore = baseRiskScore;

if (emailReputation.is_blocked) {
  riskScore = 100; // Immediate block
} else if (emailReputation.is_disposable) {
  riskScore += 40; // High risk
} else if (emailReputation.reputation_score < 30) {
  riskScore += 20; // Medium risk
}

const shouldBlock = riskScore >= 90;
const shouldFlag = riskScore >= 60;
```

## Database Security Patterns

### Migration Security
```sql
-- Always use transactions for migrations
BEGIN;

-- Create tables with proper constraints
CREATE TABLE IF NOT EXISTS public.sensitive_table (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID REFERENCES auth.users(id) ON DELETE CASCADE,
  -- ... other fields
);

-- Enable RLS immediately
ALTER TABLE public.sensitive_table ENABLE ROW LEVEL SECURITY;

-- Create restrictive policies
CREATE POLICY "Restricted access" ON public.sensitive_table
  FOR ALL TO authenticated
  USING (false);

COMMIT;
```

### Function Security
```sql
-- Use SECURITY DEFINER for privileged operations
CREATE OR REPLACE FUNCTION check_user_permissions(p_user_id UUID)
RETURNS BOOLEAN AS $$
BEGIN
  -- Function runs with definer's privileges
  -- Implement careful permission checks
  RETURN has_permission;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;
```

## API Security Headers

### CORS Configuration
```typescript
// Restrictive CORS in production
const corsHeaders = {
  'Access-Control-Allow-Origin': process.env.NODE_ENV === 'production' 
    ? 'https://getjasin.com' 
    : '*',
  'Access-Control-Allow-Methods': 'GET, POST, OPTIONS',
  'Access-Control-Allow-Headers': 'Content-Type, Authorization'
};
```

### Security Headers
```typescript
const securityHeaders = {
  'X-Content-Type-Options': 'nosniff',
  'X-Frame-Options': 'DENY',
  'X-XSS-Protection': '1; mode=block',
  'Referrer-Policy': 'strict-origin-when-cross-origin'
};
```

## Monitoring & Alerting

### Security Event Logging
```typescript
// Comprehensive security event logging
logger.warn('Security Event', {
  event_type: 'blocked_signup_attempt',
  risk_score: riskAssessment.risk_score,
  signals: riskAssessment.signals.map(s => s.signal_type),
  timestamp: new Date().toISOString(),
  ip: deviceFingerprint.ip_address
});
```

### Abuse Signal Recording
```typescript
await recordAbuseSignal(userId, {
  signal_type: 'suspicious_activity',
  severity: 'high',
  details: {
    activity_type: 'rapid_credit_usage',
    context: { /* relevant data */ }
  },
  ip_address: request_ip,
  user_agent: request_user_agent
});
```

## Error Handling Security

### Safe Error Responses
```typescript
try {
  // Risky operation
  const result = await riskyOperation();
  return NextResponse.json({ success: true, data: result });
} catch (error) {
  // Log detailed error internally
  logger.error('Operation failed', error);
  
  // Return generic error to client
  return NextResponse.json(
    { error: 'Operation failed. Please try again.' },
    { status: 500 }
  );
}
```

### Graceful Degradation
```typescript
// Fallback to cached data when security checks fail
if (securityCheckFailed) {
  logger.warn('Security check failed, using cached data');
  return cachedSafeData;
}
```

## Testing Security

### Security Test Patterns
- Test with invalid/malicious inputs
- Verify authentication bypasses fail
- Test rate limiting and abuse detection
- Validate error messages don't leak information
- Confirm RLS policies work correctly

### Penetration Testing Checklist
- SQL injection attempts
- XSS payload testing
- Authentication bypass attempts
- Authorization escalation tests
- Rate limiting validation
- Input validation boundary testing

## Compliance Considerations

### Data Privacy
- Minimal data collection
- Clear data retention policies
- User consent for tracking
- Right to deletion support

### Audit Requirements
- Comprehensive activity logging
- Immutable audit trails
- Regular security reviews
- Incident response procedures




# 🗺️ Sitemap and Page Structure (Jasin MVP)

This reflects the current MVP structure including auth-gated embed generation and email/password-based login.

---

## 📍 Page Routes

| Route            | Auth Required | Purpose                                      |
|------------------|----------------|-----------------------------------------------|
| `/`              | ❌             | Landing page with input field for ASIN/URL    |
| `/card/:id`      | ❌             | Product card preview + gated embed copy       |
| `/signup`        | ❌             | Signup form (email + password)                |
| `/login`         | ❌             | Login form (email + password)                 |
| `/dashboard`     | ✅             | Authenticated user’s saved embeds             |

---

## 🧩 Component-to-Page Mapping

| Component        | Used On         | Purpose                                       |
|------------------|------------------|-----------------------------------------------|
| `InputField`     | `/`              | Accept Amazon link or ASIN                    |
| `CardPreview`    | `/card/:id`, `/dashboard` | Render product card                |
| `ColorSelector`  | `/card/:id`      | Select theme variant (Pro access required)    |
| `EmbedCodeBox`   | `/card/:id`      | Embed HTML output (masked until login)        |
| `CopyEmbedButton`| `/card/:id`      | Triggers signup modal or copies HTML          |
| `SignupModal`    | `/card/:id`      | Prompt unauthenticated users to create account|
| `AuthButton`     | All pages        | Login/logout state + CTA                      |
| `EmbedList`      | `/dashboard`     | Display saved embeds for current user         |

---

## ✅ Notes

- All embed copy actions are gated behind authentication
- Anonymous users can preview cards but not save/copy
- Theme customization gated by subscription tier



# Stripe Subscription Webhook Handling

## Overview
Stripe subscription webhooks must use authoritative timestamp fields instead of calculations to ensure accurate subscription end dates.

## Critical Implementation Rule
**ALWAYS use Stripe's `ended_at` field as the primary source for subscription end dates, with `current_period_end` as fallback.**

## Webhook Handler Pattern
When processing subscription events in [src/app/api/stripe/webhook/route.ts](mdc:src/app/api/stripe/webhook/route.ts):

```javascript
// ✅ CORRECT: Use Stripe's authoritative data
let periodEndDate: Date | null = null;
if ((subscription as any).ended_at) {
  // Use Stripe's authoritative end date for cancelled subscriptions
  periodEndDate = new Date((subscription as any).ended_at * 1000);
} else if ((subscription as any).current_period_end) {
  // Normal case: active subscription with current_period_end
  periodEndDate = new Date((subscription as any).current_period_end * 1000);
}

// ❌ NEVER: Calculate end dates from invoices or billing intervals
// This leads to incorrect dates when subscriptions are cancelled immediately
```

## Stripe Field Reference
- `ended_at`: When subscription actually terminates (authoritative for cancelled subscriptions)
- `current_period_end`: End of current billing period (becomes null for immediate cancellations)
- `canceled_at`: When cancellation was requested (not the end date)

## Required Updates for All Handlers
1. `handleSubscriptionUpdated()` - Use ended_at → current_period_end priority
2. `handleSubscriptionCanceled()` - Store ended_at for accurate cancellation dates  
3. `handleCheckoutSessionCompleted()` - Apply same logic for consistency

## API Endpoint Consistency
The subscription API endpoint [src/app/api/stripe/subscription/route.ts](mdc:src/app/api/stripe/subscription/route.ts) must use identical logic when fetching from Stripe to ensure cached data matches webhook data.

## Database Storage
Store the authoritative end date in `subscription_period_end` field in the `profiles` table. This ensures UI components display accurate cancellation information.

## Testing Requirements
Always test webhook logic with these scenarios:
- Active subscription (uses current_period_end)
- Immediately cancelled subscription (uses ended_at)
- Cancelled at period end (uses ended_at)

## Common Pitfall
Never attempt to calculate subscription end dates by fetching invoices and adding billing intervals. Stripe provides the authoritative end date through the `ended_at` field.



# Subscription UI Patterns

## Overview
UI components displaying subscription information must handle both active and cancelled subscription states accurately.

## Settings Page Pattern
The [src/app/settings/SettingsClient.tsx](mdc:src/app/settings/SettingsClient.tsx) demonstrates the correct pattern for displaying subscription status:

## Subscription Status Display
```typescript
// ✅ CORRECT: Show appropriate status badge
<Badge className={`font-bold px-3 py-1 rounded-full text-xs ${
  subscription?.cancel_at_period_end 
    ? 'bg-amber-100 hover:bg-amber-100 text-amber-600' 
    : 'bg-green-100 hover:bg-green-100 text-green-500'
}`}>
  {subscription?.cancel_at_period_end ? 'CANCELLED' : 'ACTIVE'}
</Badge>
```

## Cancellation Notice Pattern
```typescript
// ✅ CORRECT: Show cancellation notice with accurate end date
{isPro && subscription?.cancel_at_period_end && (
  <div className="mt-4 p-3 bg-amber-100 rounded-lg">
    <div className="text-amber-600 text-sm font-medium">
      Your Pro plan will end on {subscription?.current_period_end 
        ? formatDate(subscription.current_period_end)
        : 'the next billing date'}. You'll continue to have access until then.
    </div>
  </div>
)}
```

## Date Formatting
Always format subscription dates consistently:
```typescript
const formatDate = (dateString: string) => {
  const date = new Date(dateString);
  return date.toLocaleDateString('en-US', {
    year: 'numeric',
    month: 'long',
    day: 'numeric',
  });
};
```

## Subscription Data Interface
Use consistent interface for subscription data:
```typescript
interface SubscriptionDetails {
  id: string;
  status: string;
  current_period_end: string;
  cancel_at_period_end: boolean;
  plan: {
    id: string;
    amount: number | null;
    interval: string | null | undefined;
  };
  cancel_url?: string;
  last_updated?: string | null;
}
```

## Button States
Handle subscription management buttons based on cancellation status:
```typescript
<Button onClick={handleManageSubscription}>
  {subscription?.cancel_at_period_end ? 'Reactivate Plan' : 'Manage Plan'}
</Button>
```

## Data Source Priority
1. Use server-side subscription data for initial render
2. Fall back to context subscription data for updates
3. Always prioritize `ended_at` over calculated dates from backend

## Error Handling
- Show "Loading..." when subscription data is not available
- Gracefully handle missing subscription end dates
- Provide fallback text for edge cases



# Supabase Auth Session & Reauthentication

- Access tokens (`jwt_expiry`) are set to 1 hour in [supabase/config.toml](mdc:supabase/config.toml).
- Refresh tokens are enabled and rotated (`enable_refresh_token_rotation = true`).
- No custom `[auth.sessions]` timebox or inactivity_timeout is set, so refresh tokens follow Supabase defaults (typically 1–4 weeks).
- Session refresh and management is handled via the Supabase client in [src/utils/supabase/server.ts](mdc:src/utils/supabase/server.ts), [src/utils/supabase/client.ts](mdc:src/utils/supabase/client.ts), and [src/utils/supabase/middleware.ts](mdc:src/utils/supabase/middleware.ts).
- Users stay logged in for weeks unless they log out, clear cookies, or are inactive for a long time.
- No custom logic overrides this behavior in the codebase.
- To change session duration, adjust `jwt_expiry` or set `[auth.sessions]` options in [supabase/config.toml](mdc:supabase/config.toml).



# Supabase Client Integration

This project uses various Supabase client implementations depending on the context.

## Client Implementations

1. **Server-side Admin Client**: 
   Uses service role key for privileged operations
   [supabase.ts](mdc:src/lib/supabase.ts)

2. **Browser Client**: 
   Uses SSR with cookie handling for client-side operations
   [client.ts](mdc:src/utils/supabase/client.ts)

3. **Server Component Client**:
   Used in Next.js server components with cookie handling
   [server.ts](mdc:src/utils/supabase/server.ts)

4. **Auth Middleware Client**:
   Handles session updates in middleware
   [middleware.ts](mdc:src/utils/supabase/middleware.ts)

## Authentication Flow

The authentication middleware in [middleware.ts](mdc:src/middleware.ts) handles protected routes by:
1. Checking the user's session
2. Redirecting to login if accessing protected routes
3. Checking additional privileges for admin routes

## Row Level Security (RLS)

Tables are protected by Row Level Security:
- `SELECT` operations are permitted for all users on public data
- `INSERT/UPDATE/DELETE` operations usually require authentication
- Some operations require specific user roles or permissions

When querying tables directly, be aware of RLS policies that may restrict access.



# Database Analytics System

## Overview

The analytics system tracks views and clicks on embedded product cards with built-in bot detection. Data flows through transactional functions to ensure consistency and is cleaned up automatically after 90 days.

## Tables & Views

### Core Tables
- `embed_views` - Individual view events with referrer and user agent data
- `embed_clicks` - Individual click events with referrer and user agent data
- `embed_stats` - Aggregated statistics per embed
- `daily_stats` - Time-series data for charting

### Views
- `bot_traffic_summary` - Summarizes bot traffic by type and day
- `mv_user_analytics_summary` - Materialized view with user totals (views, clicks, embeds)

## Data Flow

1. When a card is viewed, `track_embed_event_with_transaction_v2` is called
2. This function:
   - Detects if the request is from a bot
   - Updates counts in `embeds` table
   - Inserts a record in `embed_views`
   - Updates or inserts aggregates in `embed_stats` and `daily_stats`

## Bot Detection

The system identifies and flags bot traffic:
- Adds `is_bot` and `bot_type` flags to view/click records
- Tracks bot views separately from human views
- Provides bot analytics via `get_user_bot_analytics` and `get_all_bot_analytics`

## Data Retention

- Raw event data (views/clicks) is kept for 90 days
- After 90 days, the `clean_old_analytics_data` function removes old records
- Aggregated statistics remain available indefinitely

## Key Functions

```sql
-- Track an embed view or click with full context
CALL track_embed_event_with_transaction_v2(
  p_embed_id, 
  p_event_type, -- 'view' or 'click'
  p_referer_url,
  p_user_agent,
  p_is_bot,
  p_bot_type,
  p_request_id
);

-- Get analytics for a user with optional bot filtering
SELECT * FROM get_user_analytics(
  p_user_id => 'user-uuid',
  p_exclude_bots => true
);
```

The analytics system is optimized for fast writes during tracking and efficient reads for dashboard displays. See [20240805_improve_analytics_transactions.sql](mdc:supabase/migrations/20240805_improve_analytics_transactions.sql) for transaction management.




# Database Functions Overview

## Core Functions

The Supabase database contains numerous functions that power key application features:

### Analytics Functions
- `track_embed_event_with_transaction_v2` - Tracks views and clicks in a single transaction
- `clean_old_analytics_data` - Implements data retention policy (90 day default)
- `get_user_analytics` - Retrieves analytics data for a specific user
- `refresh_analytics_summary` - Updates materialized views for analytics dashboards

### Subscription Management
- `get_user_subscription_data` - Returns complete subscription info including tier, limits, etc.
- `get_user_subscription_status` - Simplified subscription status check

### Product Data Functions
- `update_product_prices` - Updates Amazon product prices (scheduled job)
- `clean_old_products` - Removes outdated product data

### User & Embed Management
- `get_user_embed_counts` - Returns total and monthly embed counts
- `upsert_user_brand_settings` - Manages user brand preferences
- `generate_random_avatar` - Creates randomized avatar options

## Example Usage

These functions can be called from SQL or from server-side code via the Supabase client:

```typescript
const { data, error } = await supabase.rpc('get_user_analytics', { 
  p_user_id: user.id,
  p_exclude_bots: true
})
```

Most functions have RLS policies that enforce authentication and appropriate access controls.



# Supabase Database Schema Overview

This project uses a Supabase PostgreSQL database with the following structure.

## Core Tables

### 📦 Table: `products`

| Column           | Type      | Notes                            |
|------------------|-----------|----------------------------------|
| asin             | text      | Primary key                      |
| title            | text      | Product title                    |
| brand            | text      | Product brand                    |
| manufacturer     | text      | Product manufacturer             |
| current_price    | numeric   | Current selling price            |
| original_price   | numeric   | Original/list price              |
| savings_percent  | numeric   | Discount percentage              |
| currency         | text      | Price currency                   |
| in_stock         | boolean   | Stock availability               |
| prime_eligible   | boolean   | Prime shipping eligibility       |
| amazon_url       | text      | Product URL                      |
| primary_image    | text      | Main product image URL           |
| gallery_images   | jsonb     | Additional product images        |
| features         | jsonb     | Product features list            |
| description      | text      | Product description              |
| raw_data         | jsonb     | Raw API response                 |
| affiliate_tag    | text      | Amazon affiliate tag             |
| updated_at       | timestamp | Last update timestamp            |

### 📦 Table: `embeds`

| Column         | Type      | Notes                                        |
|----------------|-----------|----------------------------------------------|
| id             | uuid      | Primary key                                  |
| user_id        | uuid      | FK to auth.users (required for embeds)       |
| asin           | text      | FK to products.asin                          |
| theme          | text      | Card theme selection                         |
| clicks         | integer   | Click count                                  |
| views          | integer   | View count                                   |
| created_at     | timestamp | Creation timestamp                           |
| updated_at     | timestamp | Last update timestamp                        |
| affiliate_tag  | text      | Override affiliate tag (Pro feature)         |

### 📦 Table: `profiles`

| Column             | Type      | Notes                                 |
|--------------------|-----------|---------------------------------------|
| id                 | uuid      | Matches auth.users.id                 |
| email              | text      | User email                            |
| subscription_tier  | text      | 'free' or 'pro'                       |
| stripe_customer_id | text      | For syncing billing state             |
| created_at         | timestamp | Timestamp when profile was created    |

## Analytics Tables

The system includes tables for tracking views and clicks:
- `embed_views` - Detailed view event records
- `embed_clicks` - Detailed click event records
- `embed_stats` - Aggregated statistics per embed
- `daily_stats` - Time-series data for charting

## Row-Level Security (RLS)

Tables use Row Level Security (RLS) policies:

### `products` (no RLS)
- Accessed via server-side API only

### `embeds` (RLS: ON)
- `SELECT`: allowed for all (public embed display)
- `INSERT`: allowed only if `auth.uid()` is present
- `UPDATE`: only if `user_id = auth.uid()`
- `DELETE`: only if `user_id = auth.uid()`

### `profiles` (RLS: ON)
- Users can only access their own profile data
- Service role can access all profiles for admin functions

## Business Logic

- All embed records are tied to authenticated users
- Quota enforcement uses `embeds.user_id` + `created_at` timestamps
- `profiles.subscription_tier` determines feature access
- Analytics are restricted by subscription tier



# Email Change Implementation

## Overview

This document explains how email address changes are handled in the Jasin application. Instead of using Supabase's default double-confirmation workflow (which requires confirmation from both old and new email addresses), we've implemented a custom server-side approach using Supabase's admin API.

## Previous Implementation

Supabase's default email change flow works as follows:

1. Client calls `supabase.auth.updateUser({ email: newEmail })`
2. Supabase sends confirmation emails to both the old and new email addresses
3. User must click confirmation links in both emails to complete the change
4. After confirmations, the session is updated with the new email

This approach provides security but creates friction in the user experience, requiring users to access both email accounts to complete the change.

## New Implementation

Our new approach streamlines this process:

1. Client sends a request to our custom API endpoint: `/api/auth/update-email`
2. The server-side endpoint:
   - Authenticates the current user
   - Validates the new email format
   - Uses `supabaseAdmin.auth.admin.updateUserById()` to update the email directly
   - Also updates the email in the `profiles` table
3. Upon success, the client refreshes the page to show the updated email

## Benefits

- **Simplified User Experience**: No need to access multiple email accounts
- **Immediate Changes**: Email address is updated instantly
- **Reduced Confusion**: Eliminates issues with email confirmation links expiring or not being received
- **Same Security Level**: Still requires authentication to make changes

## Implementation Details

### 1. Server-Side API Endpoint (`/api/auth/update-email/route.ts`)

The endpoint uses the Supabase admin client with service role key to update user emails:

```typescript
// Uses supabaseAdmin.auth.admin.updateUserById() to bypass double confirmation
const { data, error: updateError } = await supabaseAdmin.auth.admin.updateUserById(
  user.id,
  { email: newEmail }
);
```

### 2. Client-Side Code (`SettingsClient.tsx`)

The client sends a request to our custom endpoint:

```typescript
// Call our custom API endpoint instead of using Supabase client directly
const response = await fetch('/api/auth/update-email', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
  },
  body: JSON.stringify({ newEmail }),
});
```

## Security Considerations

- The admin key is only used server-side and never exposed to clients
- Authentication is still required to make email changes
- Standard validation ensures email format is correct
- The API checks that the user is authenticated and only allows changes to their own account

## Future Improvements

Potential future enhancements:

- Add optional verification email to the new address (post-change)
- Implement additional security checks like password confirmation for sensitive changes
- Add audit logging for email changes


# Supabase CLI MCP Tools

This rule documents how to use the Supabase CLI MCP tools for database operations in our project.

## Migration Management

When working with migrations:

1. **Applying Migrations**: 
   ```
   mcp_supabase_apply_migration({
     project_id: "txjdkveocmpyidzjvzjj",
     name: "migration_name",
     query: "SQL content here"
   })
   ```

2. **Executing SQL**:
   ```
   mcp_supabase_execute_sql({
     project_id: "txjdkveocmpyidzjvzjj",
     query: "SELECT * FROM your_table"
   })
   ```

3. **Listing Tables**:
   ```
   mcp_supabase_list_tables({
     project_id: "txjdkveocmpyidzjvzjj",
     schemas: ["public"]
   })
   ```

## Common Issues

- When applying migrations directly, omit `BEGIN;` and `COMMIT;` statements
- Double dollar signs (`$$`) in migrations may cause syntax errors with the MCP tool
- For complex migrations, break them into smaller parts
- Always check if tables/columns exist before applying migrations

## Project Details

- **Project ID**: `txjdkveocmpyidzjvzjj`
- **URL**: `https://txjdkveocmpyidzjvzjj.supabase.co`

## Migration Workflow

Instead of the standard Supabase CLI workflow:

1. Create the migration SQL file
2. Fix any timestamp issues with `npm run db:fix-all`
3. Apply directly via `mcp_supabase_apply_migration`
4. Verify with `mcp_supabase_execute_sql`

This approach avoids migration history conflicts that can occur with the standard CLI workflow.



# Migration Management

This project uses a structured workflow for managing database migrations with Supabase.

## Migration Scripts

Key scripts for migration management:

1. **Create Migrations**:
   ```bash
   npm run db:create migration_name
   ```
   Implementation: [db-migrate.sh](mdc:scripts/db-migrate.sh)

2. **Fix Future-Dated Migrations**:
   ```bash
   npm run db:fix-all
   ```
   Implementation: [fix-all-future-migrations.sh](mdc:scripts/fix-all-future-migrations.sh)

3. **Rename Migrations**:
   ```bash
   npm run db:rename migration_name
   ```
   Implementation: [rename-migrations.sh](mdc:scripts/rename-migrations.sh)

4. **Sync Remote/Local Migrations**:
   ```bash
   npm run db:sync-migrations
   ```
   Implementation: [sync-migrations.sh](mdc:scripts/sync-migrations.sh)

## Migration Format

Migrations follow this format:
- Filename: `YYYYMMDD_descriptive_name.sql`
- SQL wrapped in transaction blocks (BEGIN/COMMIT)
- Located in `supabase/migrations/` directory

## Migration Workflow

1. Create migration: `npm run db:create description`
2. Edit the SQL in the generated file
3. Fix migration timestamps if needed: `npm run db:fix-all`
4. Apply locally: `supabase db reset`
5. Push to remote: `npm run db:push`

## Local vs Remote Management

For syncing local and remote databases:
- `npm run db:sync` - Sync remote database to local
- `npm run db:sync-migrations` - Create local placeholders for remote migrations




# Secure Password Change Implementation

## Overview

This document explains how password changes are securely handled in the Jasin application. Unlike the previous implementation, our new approach ensures proper validation of the current password before allowing any changes.

## Implementation Details

The password change flow consists of two main parts:

1. **Server-side validation** - A database function and API endpoint that securely verify the current password
2. **Client-side integration** - The UI component that calls the API endpoint

### Database Function for Password Verification

We've created a PostgreSQL function that securely checks if a password matches the user's current password:

```sql
CREATE OR REPLACE FUNCTION verify_user_password(password text)
RETURNS BOOLEAN
SECURITY DEFINER
SET search_path = public
AS $$
BEGIN
  RETURN EXISTS (
    SELECT id 
    FROM auth.users 
    WHERE id = auth.uid() 
    AND encrypted_password = crypt(password::text, auth.users.encrypted_password)
  );
END;
$$ LANGUAGE plpgsql;
```

This function runs with `SECURITY DEFINER` to ensure it has the necessary privileges to check passwords against the auth schema.

### API Endpoint (`/api/auth/update-password`)

The API endpoint handles the password change request securely:

1. Verifies user authentication
2. Validates both the current and new passwords
3. Uses the `verify_user_password` function to check the current password
4. If the current password is correct, uses the admin API to update the password

### Client-Side Integration

The Settings page form sends both the current and new passwords to the API endpoint, handling success and error states appropriately.

## Security Benefits

This implementation provides several security benefits:

1. **Server-side verification** - Password validation happens on the server, preventing client-side bypasses
2. **Current password requirement** - Ensures that only users who know the current password can change it
3. **Admin API usage** - Leverages Supabase's admin API for secure password updates
4. **Detailed error handling** - Provides clear feedback without exposing sensitive information
5. **Input validation** - Ensures password complexity requirements are met

## Flow Diagram

```
┌─────────────┐         ┌──────────────┐         ┌────────────────┐         ┌──────────────┐
│  User       │         │  Frontend    │         │  API Endpoint  │         │  Database    │
│  Interface  │         │  Component   │         │                │         │              │
└──────┬──────┘         └──────┬───────┘         └───────┬────────┘         └──────┬───────┘
       │                       │                         │                         │
       │ Enter current and     │                         │                         │
       │ new passwords         │                         │                         │
       │─────────────────────>│                         │                         │
       │                       │                         │                         │
       │                       │ POST /api/auth/update-  │                         │
       │                       │ password                │                         │
       │                       │────────────────────────>│                         │
       │                       │                         │                         │
       │                       │                         │ Call verify_user_       │
       │                       │                         │ password function       │
       │                       │                         │────────────────────────>│
       │                       │                         │                         │
       │                       │                         │<────────────────────────│
       │                       │                         │ Return boolean result   │
       │                       │                         │                         │
       │                       │                         │ If verified, call       │
       │                       │                         │ supabaseAdmin.auth.     │
       │                       │                         │ admin.updateUserById    │
       │                       │                         │                         │
       │                       │<────────────────────────│                         │
       │                       │ Return success/error    │                         │
       │                       │                         │                         │
       │ Show success/error    │                         │                         │
       │<─────────────────────│                         │                         │
       │                       │                         │                         │
```

## Comparison to Previous Implementation

The previous implementation had several shortcomings:

1. It didn't verify the current password, making it possible for someone with access to a logged-in session to change the password
2. It relied entirely on client-side code for password updates
3. It lacked detailed error handling and messaging

The new implementation addresses all these issues, creating a more secure and user-friendly password change flow.


# PostgreSQL Scheduled Jobs

## Overview

The database uses pg_cron extension to automate maintenance tasks and data updates. These scheduled jobs keep analytics up-to-date and maintain database hygiene.

## Active Jobs

| Job Name | Schedule | Function | Purpose |
|----|----|----|----|
| `clean-analytics-data-weekly` | Weekly (Sunday at midnight) | `clean_old_analytics_data()` | Removes analytics data older than 90 days |
| `refresh-analytics-summary-hourly` | Hourly | `refresh_analytics_summary()` | Updates analytics materialized views |
| `clean-preview-embeds-daily` | Daily (3 AM) | `clean_old_preview_embeds()` | Removes temporary preview embeds |
| `update-product-prices` | Every 4 hours | `update_product_prices(100)` | Updates Amazon product pricing data |

## Job Management

To manage these jobs, you can use the following SQL commands:

```sql
-- View all scheduled jobs
SELECT * FROM cron.job;

-- Disable a job
UPDATE cron.job SET active = false WHERE jobname = 'job-name';

-- Enable a job
UPDATE cron.job SET active = true WHERE jobname = 'job-name';

-- Remove a job
DELETE FROM cron.job WHERE jobname = 'job-name';
```

## Monitoring

Jobs can be monitored via the `cron.job_run_details` table:

```sql
-- View recent job runs
SELECT * FROM cron.job_run_details 
ORDER BY start_time DESC 
LIMIT 10;
```

See [20240723_add_pg_cron_helper_functions.sql](mdc:supabase/migrations/20240723_add_pg_cron_helper_functions.sql) for helper functions to manage cron jobs.




# Product Data Management

## Overview

The system manages Amazon product data through a combination of caching, scheduled updates, and efficient API usage. This ensures up-to-date product information while minimizing Amazon PAAPI calls.

## Data Structure

The `products` table stores complete product information:
- Basic details (title, brand, manufacturer)
- Pricing (current_price, original_price, savings_percent)
- Media (primary_image, gallery_images)
- Specifications and features
- Raw API response data

## Price Update System

To comply with Amazon's terms of service, product prices are updated at least every 24 hours:

1. The `update_product_prices(batch_size)` function identifies products needing updates
2. Updates are processed in batches to manage API rate limits
3. Only price-related fields are requested from PAAPI (not complete product data)
4. A scheduled pg_cron job runs this function every 4 hours

## API Usage Tracking

All PAAPI interactions are logged in the `paapi_calls` table:
- Records user_id, asin, request_type
- Tracks cache hits and misses
- Stores success/failure status
- Enables monitoring of API usage against quotas

## Caching Strategy

The system implements a sophisticated caching strategy:
- Complete product data is cached for ~48 hours
- Price data is refreshed more frequently (every 24 hours)
- Cache hits are prioritized to minimize API calls
- Preview embeds use cached data when possible

## Key Functions

```sql
-- Update prices for a batch of products
SELECT update_product_prices(100); -- Updates 100 products

-- Check API usage statistics
SELECT * FROM get_paapi_usage_stats(7); -- Stats for past 7 days
```

See [20240601_add_product_price_update_job.sql](mdc:supabase/migrations/20240601_add_product_price_update_job.sql) for the implementation of the price update system.




# Remote Supabase Access

This project includes comprehensive tools for connecting to remote Supabase databases.

## Setup and Connection

1. First, authenticate with Supabase CLI:
   ```bash
   supabase login
   ```

2. Set up remote connection using our custom script:
   ```bash
   npm run db:setup-remote txjdkveocmpyidzjvzjj
   ```
   Script: [setup-supabase-remote.sh](mdc:scripts/setup-supabase-remote.sh)

3. Ensure your `.env.local` contains:
   ```
   NEXT_PUBLIC_SUPABASE_URL=https://txjdkveocmpyidzjvzjj.supabase.co
   NEXT_PUBLIC_SUPABASE_ANON_KEY=your-anon-key
   NEXT_PUBLIC_SUPABASE_SERVICE_ROLE_KEY=your-service-role-key
   ```

## Migration Management

- Fix future-dated migrations: [fix-all-future-migrations.sh](mdc:scripts/fix-all-future-migrations.sh)
- Rename migrations: [rename-migrations.sh](mdc:scripts/rename-migrations.sh)
- Sync remote/local migrations: [sync-migrations.sh](mdc:scripts/sync-migrations.sh)

## SQL Access

For direct SQL access, we recommend:
1. Using pgAdmin with connection details in `db-connection.txt`
2. Using the Supabase Dashboard SQL Editor

Our SQL execution script has limited functionality due to RLS restrictions:
[execute-sql.js](mdc:scripts/execute-sql.js)

## Documentation

Full documentation below:

# Remote Supabase Access Guide

This guide explains how to properly connect to your remote Supabase instance for both application development and direct database access.

## Prerequisites

- Supabase CLI installed: `npm install -g supabase`
- Docker Desktop (for local development)
- Access to your Supabase project dashboard

## 1. CLI Authentication

The Supabase CLI uses a separate authentication method from your application:

```bash
# Authenticate Supabase CLI (opens browser window)
supabase login
```

This command will:
- Open a browser window for authentication
- Store an access token in `~/.config/supabase/access-token`
- Allow access to all your Supabase projects

## 2. Project Setup & Connection

### One-Time Setup

Run our setup script to configure your project and store connection details:

```bash
# Replace with your project ID
npm run db:setup-remote txjdkveocmpyidzjvzjj

# If you have a database password, add it
npm run db:setup-remote txjdkveocmpyidzjvzjj your_password
```

This script:
- Links to your remote project
- Updates your `.env.local` file with connection details
- Creates a `db-connection.txt` file with database credentials

### Environment Variables

Ensure your `.env.local` file contains:

```
NEXT_PUBLIC_SUPABASE_URL=https://txjdkveocmpyidzjvzjj.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=your-anon-key
NEXT_PUBLIC_SUPABASE_SERVICE_ROLE_KEY=your-service-role-key
```

These keys can be found in your Supabase dashboard under Project Settings → API.

## 3. Handling Migrations

Our project has scripts to manage migrations between local and remote:

```bash
# Sync migrations between local and remote
npm run db:sync-migrations

# Fix future-dated migrations
npm run db:fix-all

# Rename specific migrations
npm run db:rename migration_name

# Push local migrations to remote
npm run db:push
```

## 4. Direct Database Access

There are two recommended methods for direct SQL access:

### A. Database GUIs

1. Use the connection details in `db-connection.txt`
2. Connect with tools like pgAdmin, DBeaver, or Beekeeper Studio

### B. Supabase Dashboard

1. Log in to the [Supabase Dashboard](https://app.supabase.com)
2. Select your project
3. Go to the SQL Editor

## 5. Common Issues & Solutions

### Migration Mismatch

If you see migration history mismatch errors:

```bash
# First sync migrations to get placeholders for remote migrations
npm run db:sync-migrations

# Then repair the migration history
supabase migration repair --status applied migration_id
```

### Permission Denied

If you see permission denied errors:

1. Ensure you're using the service role key for admin operations
2. Check Row Level Security (RLS) settings in Supabase dashboard
3. For CLI operations, ensure you've linked with your database password

### Connection Issues

If the CLI cannot connect:

1. Check if Supabase CLI is authenticated: `supabase projects list`
2. Verify your project is linked: `supabase status`
3. Try updating the Supabase CLI: `npm update -g supabase`

## 6. Best Practices

- **Never commit service role keys** to your repository
- Use the anon key for client-side operations
- Use service role key only for server-side operations
- Enable RLS for all public tables
- Develop locally, then push to remote
- Use pgAdmin or similar tools for complex database operations 



# Remote Supabase Setup and Access Guide

This guide explains how to set up and access the remote Supabase instance for the Jasin project.

## Overview

The Jasin project uses a remote Supabase instance with the following details:
- **Project ID**: `txjdkveocmpyidzjvzjj`
- **URL**: `https://txjdkveocmpyidzjvzjj.supabase.co`

## Initial Setup

### 1. Authenticate with Supabase CLI

```bash
supabase login
```

This will open a browser window for authentication. Once authenticated, your credentials will be stored locally.

### 2. Link to Remote Project

```bash
npm run db:setup-remote txjdkveocmpyidzjvzjj
```

This script will:
- Link your local project to the remote Supabase instance
- Update your `.env.local` file with the necessary environment variables
- Create a `db-connection.txt` file with database connection details

If you have database password, you can include it:

```bash
npm run db:setup-remote txjdkveocmpyidzjvzjj your_password
```

### 3. Verify Environment Variables

Ensure your `.env.local` file contains:

```
NEXT_PUBLIC_SUPABASE_URL=https://txjdkveocmpyidzjvzjj.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=your-anon-key
NEXT_PUBLIC_SUPABASE_SERVICE_ROLE_KEY=your-service-role-key
```

You can find these keys in the Supabase dashboard under Project Settings → API.

### 4. Sync Migrations

To ensure your local environment has all the migrations:

```bash
npm run db:sync-migrations
```

This will download remote migrations to your local `supabase/migrations` directory.

## Working with Migrations

### Creating Migrations

```bash
npm run db:create migration_name
```

This creates a new SQL migration file in `supabase/migrations`.

### Pushing Migrations to Remote

After creating and testing your migration locally:

```bash
npm run db:push
```

### Fixing Migration Timestamps

If you encounter issues with future-dated migrations:

```bash
npm run db:fix-all
```

## Database Access

### Via Supabase Dashboard

1. Go to [Supabase Dashboard](https://app.supabase.com)
2. Select the Jasin project
3. Use the SQL Editor for queries

### Direct Database Access

Use the connection details in `db-connection.txt` to connect with:

- pgAdmin
- DBeaver
- Beekeeper Studio
- Other PostgreSQL clients

### Using API with Supabase Client

For server-side operations with full admin privileges:

```typescript
import { supabaseAdmin } from '@/lib/supabase';

const { data, error } = await supabaseAdmin
  .from('table_name')
  .select('*');
```

For client-side operations with Row Level Security:

```typescript
import { createClient } from '@/utils/supabase/client';

const supabase = createClient();
const { data, error } = await supabase
  .from('table_name')
  .select('*');
```

## Troubleshooting

### Migration Mismatch

If you see "migration history does not match" errors:

```bash
npm run db:sync-migrations
```

Then try repairing specific migrations if needed:

```bash
supabase migration repair --status applied migration_id
```

### Permission Denied

If you encounter permission issues:
1. Check that you're using the service role key for admin operations
2. Verify Row Level Security (RLS) settings in the Supabase dashboard

### Connection Issues

If you can't connect:
1. Check if Supabase CLI is authenticated: `supabase projects list`
2. Verify your project is linked: `supabase status`
3. Try updating the Supabase CLI: `npm update -g supabase` 


# Row Level Security Policies

## Overview

Supabase uses PostgreSQL's Row Level Security (RLS) to enforce access control at the database level. This ensures that users can only access data they are authorized to see, regardless of the application making the request.

## Key RLS Policies

### `embeds` Table

- `Public read access consolidated`: Everyone can view embed records (for rendering)
- `Users can manage their own embeds`: Authenticated users can only modify their own embeds
- `Enforce embed quota v2`: Limits embed creation based on subscription tier
  - Free tier: 5 embeds total (lifetime)
  - Pro tier: 50 embeds per month

### `products` Table

This table has RLS enabled but forces all access through service_role for caching and security.

### `profiles` Table

- Users can only read/update their own profile data
- Service role has full access for administrative functions

### Analytics Tables

- `embed_stats`: Only accessible by the embed creator
- `daily_stats`: Users can only see statistics for their own accounts
- `embed_views` and `embed_clicks`: No RLS, managed by server-side functions

## Example Policy

```sql
-- Example of quota enforcement policy
CREATE POLICY "Enforce embed quota v2" ON embeds
FOR INSERT TO authenticated
WITH CHECK (
  -- Free tier: 5 total embeds
  (
    (SELECT subscription_tier FROM profiles WHERE id = auth.uid()) = 'free'
    AND
    (SELECT COUNT(*) FROM embeds WHERE user_id = auth.uid()) < 5
  )
  OR
  -- Pro tier: 50 embeds per month
  (
    (SELECT subscription_tier FROM profiles WHERE id = auth.uid()) = 'pro'
    AND
    (SELECT COUNT(*) FROM embeds 
     WHERE user_id = auth.uid() 
     AND created_at >= date_trunc('month', CURRENT_DATE)) < 50
  )
);
```

## Checking Policies

To view existing policies for a table:

```sql
SELECT * FROM pg_policies WHERE tablename = 'embeds';
```

Always test RLS policies thoroughly to ensure they function correctly in all scenarios.





# 🧭 UX Sitemap Diagram (Mermaid)

This diagram reflects the updated authentication model (email + password) and gated embed experience.

```mermaid
graph TD
  A[Home (/)] --> B[User pastes Amazon link]
  B --> C[Parse ASIN + Affiliate Tag]
  C --> D[Check or Save Product in Supabase]
  D --> E[Redirect to /card/:id]
  E --> F[Render Product Card Preview]
  F --> G[Color Selector (Theme)]
  F --> H[Embed Code Output (Gated)]
  F --> I[Signup Modal or Login Prompt]

  I --> J[Signup Page (/signup)]
  I --> J2[Login Page (/login)]
  J --> K[User Submits Email + Password]
  J2 --> K2[User Submits Email + Password]
  K --> L[Redirect to /card/:id]
  K2 --> L2[Redirect to /card/:id]

  L --> M[Session Established]
  L2 --> M2[Session Established]

  M --> N[Embed Copied + Saved]
  M2 --> N2[Embed Copied + Saved]

  N --> O[Redirect or Access Dashboard (/dashboard)]
  O --> P[View Saved Embeds]
  P --> Q[Copy or Delete Saved Cards]
```




# 🔄 UX Flow and Interaction Model (Jasin MVP)

---

## 🧠 Anonymous Flow (Preview Only)

1. User lands on `/`
2. Pastes Amazon URL or ASIN
3. System extracts ASIN and affiliate tag (if present)
4. Checks product cache in Supabase
5. If missing, calls Amazon PAAPI → stores result
6. Redirects to `/card/:id`
7. Loads and renders:
   - Product card preview
   - Theme selector (disabled for free users)
   - Embed code section (blurred or locked)
   - Call-to-action: “Create a free account to copy this card”

---

## 🔐 Authenticated Flow (Email + Password)

1. User clicks “Sign up” or “Login”
2. Enters email + password
3. On success:
   - Auth session starts (stored locally via Supabase)
   - Redirect back to `/card/:id` or `/dashboard`
4. If returning to `/card/:id`:
   - Immediately trigger embed copy + show success toast
   - Embed saved to user account
5. If visiting `/dashboard`:
   - List saved embeds
   - Can copy or delete
   - (Future: support card editing, analytics)

---

## 🧭 UX Strategy

- **Preview-first**: Users can always see their product card before signing up
- **Auth-gated value**: Copying the embed is the incentive to sign up
- **Non-blocking CTA**: Signup is only required when action (embed) is taken
- `/card/:id` becomes the shareable core artifact in the system
- Free plan feels useful but has a clear upgrade path



# Vercel KV Setup and Caching Implementation

## Overview
This project uses Vercel KV for caching to improve performance. The caching system is implemented in [src/lib/kv-cache.ts](mdc:src/lib/kv-cache.ts) and works both locally and in production with graceful fallback.

## Local Development Setup

### 1. Create a Vercel KV Database
1. Go to [Vercel Dashboard](mdc:https:/vercel.com/dashboard)
2. Navigate to Storage → Create Database → KV
3. Name it `jasin-cache` or similar
4. Copy the environment variables

### 2. Add Environment Variables
Add these to your `.env.local` file:

```env
# Vercel KV Configuration
KV_REST_API_URL="https://your-kv-url.kv.vercel-storage.com"
KV_REST_API_TOKEN="your-kv-token"
KV_REST_API_READ_ONLY_TOKEN="your-readonly-token"
```

### 3. Test the Setup
```bash
npm run dev
```

The app will automatically detect if KV is available and use it for caching.

## Implementation Details

### Core Files
- [src/lib/kv-cache.ts](mdc:src/lib/kv-cache.ts) - Main KV cache implementation
- [src/app/dashboard/page.tsx](mdc:src/app/dashboard/page.tsx) - Dashboard caching integration
- [src/lib/amazon.ts](mdc:src/lib/amazon.ts) - Product data caching
- [src/app/api/admin/performance/route.ts](mdc:src/app/api/admin/performance/route.ts) - Cache metrics monitoring

### Cache Behavior
- **KV Available**: Fast caching for dashboard, products, analytics
- **KV Unavailable**: Falls back to database-only mode (still fast!)
- **Graceful Degradation**: App works perfectly without cache

### Cache Keys and TTLs
- `product:{asin}` - Product data (48h TTL)
- `dashboard:{userId}` - Dashboard data (5min TTL)
- `analytics:{embedId}` - Analytics data (10min TTL)
- `user:{userId}:embeds` - User embeds (2min TTL)
- `subscription:{userId}` - Subscription data (24h TTL)

### Performance Benefits
- **Dashboard**: 3-5s → 200-500ms
- **Product Lookup**: 2-5s → 100-300ms
- **Analytics**: 1-2s → 100-200ms

## Production Deployment

### Automatic Setup
When you deploy to Vercel:
1. The KV database is automatically linked
2. Environment variables are set automatically
3. No additional configuration needed

### Manual Setup (if needed)
1. In Vercel Dashboard → Project Settings → Environment Variables
2. Add the KV environment variables
3. Redeploy your application

## Monitoring

Check cache performance at:
```
GET /api/admin/performance
```

Shows:
- Cache hit rate
- KV availability status
- Performance metrics

## Troubleshooting

### KV Not Working Locally
1. Check environment variables in `.env.local`
2. Verify KV database exists in Vercel Dashboard
3. Check network connectivity

### Production Issues
1. Verify KV is linked to your Vercel project
2. Check environment variables in Vercel Dashboard
3. Redeploy if needed

## Cost Considerations

### Vercel KV Pricing
- **Hobby Plan**: 30,000 requests/month free
- **Pro Plan**: 500,000 requests/month included
- Additional usage: $0.50 per 100,000 requests

### Optimization
- Cache TTLs are optimized for cost efficiency
- Only essential data is cached
- Automatic fallback reduces dependency

## Usage Patterns

### Cache Implementation
```typescript
import { cache, CacheKeys, CacheTTL } from '@/lib/kv-cache';

// Get or set pattern
const data = await cache.getOrSet(
  CacheKeys.dashboard(userId),
  () => fetchFromDatabase(),
  CacheTTL.DASHBOARD
);
```

### Cache Invalidation
```typescript
// Delete specific key
await cache.del(CacheKeys.product(asin));

// Pattern invalidation (limited support)
await cache.invalidatePattern('user:*');
```
