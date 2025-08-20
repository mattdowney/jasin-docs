# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Jasin is an enterprise-grade SaaS platform that transforms Amazon affiliate links into beautiful, embeddable product cards. Built with Next.js 14 (App Router), TypeScript, Supabase, and Stripe, featuring advanced security, analytics, and automated operations.

## Essential Commands

### Development
```bash
npm run dev              # Start development server
npm run dev:stripe       # Dev server + Stripe webhook listener
npm run build            # Production build
npm run lint             # ESLint validation
```

### Database Operations (Remote Supabase Only)
```bash
npm run db:status        # Check migration status
npm run db:push          # Push migrations to remote
npm run db:sync-migrations # Sync remote migrations locally
npm run db:sql           # Execute SQL queries
npm run db:setup-remote  # Connect to remote Supabase project
```

### Specialized Operations
```bash
npm run price:apply      # Apply price update jobs
npm run price:verify     # Verify price update system
```

**IMPORTANT**: This project uses **remote Supabase exclusively** - no local Docker setup. All database operations work against the remote instance.

## Architecture Overview

### Tech Stack
- **Frontend**: Next.js 14 with App Router, TypeScript, Tailwind CSS, shadcn/ui
- **Backend**: Next.js API Routes (50+ endpoints), Supabase PostgreSQL
- **External APIs**: Amazon PAAPI 5.0, Stripe, PostHog, Upstash Redis
- **Deployment**: Vercel with cron jobs and edge functions

### Key Application Structure
```
src/app/
├── admin/              # Admin dashboard (/admin)
├── dashboard/          # User management (/dashboard)  
├── builder/           # Embed creation (/builder)
├── auth/              # Authentication flows
└── api/               # 50+ API endpoints with rate limiting
```

### Core Database Architecture
- **User Data**: profiles, embeds, analytics (all with RLS policies)
- **Product Cache**: products table with ASIN indexing and price history
- **Anti-Abuse**: device_fingerprints, abuse_signals, email_domain_reputation
- **Analytics**: embed_views, embed_clicks with bot detection and 90-day retention

## Critical System Features

### Amazon PAAPI Rate Limiting & Circuit Breaker
- **Multi-layer limits**: Daily (8,640), hourly (360), minute (6) with per-ASIN throttling
- **Circuit breaker states**: CLOSED/OPEN/HALF_OPEN with automatic recovery
- **Graceful degradation**: Falls back to stale cache when rate limited
- Admin dashboard shows real-time usage and circuit breaker status

### Anti-Abuse System  
- **Device fingerprinting**: SHA256-based browser/device tracking
- **Email domain reputation**: 0-100 scoring with disposable email blocking
- **Risk assessment**: Multi-factor signup evaluation with automatic blocking at 90+ score
- **Abuse signals**: Comprehensive logging with severity levels (low/medium/high/critical)

### Analytics with Bot Detection
- **Advanced bot filtering**: 15+ detection categories with pattern matching
- **90-day retention**: Automated cleanup via pg_cron jobs
- **Materialized views**: Hourly refresh for performance optimization
- **v2 functions**: `track_embed_event_with_transaction_v2()` for event tracking

### Security Implementation
- **Row Level Security**: All user data tables have granular RLS policies
- **Comprehensive CSP headers**: Prevents XSS and data exfiltration
- **Webhook verification**: Stripe webhook signatures with replay attack prevention
- **API rate limiting**: Per-endpoint limits with quota enforcement

## Key Development Patterns

### Database Access Patterns
```typescript
// Always use service role for admin operations
const { data } = await supabase
  .from('profiles')
  .select('*')
  .eq('user_id', userId)
  .single();
```

### Error Handling & Logging
```typescript
// Use structured logging with debug categories
const debug = createDebug('api:products');
debug('Processing ASIN request: %s', asin);
```

### Rate Limiting Integration
```typescript
// Check rate limits before PAAPI calls
const rateLimitStatus = await checkRateLimit();
if (rateLimitStatus.blocked) {
  return fallbackToCache(asin);
}
```

## Automated Operations

### Vercel Cron Jobs
- **Price updates**: Every 4 hours (`/api/cron/update-prices-efficient`)  
- **Color extraction**: Every 6 hours (`/api/admin/update-product-colors`)

### Database Maintenance (pg_cron)
- **Analytics cleanup**: Weekly 90+ day data retention
- **Materialized view refresh**: Hourly performance optimization
- **Preview embed cleanup**: Daily temporary data removal

## Environment Variables

### Required for Development
```env
# Supabase (Remote Only)
NEXT_PUBLIC_SUPABASE_URL=your_supabase_url
SUPABASE_SERVICE_ROLE_KEY=your_service_role_key

# Amazon PAAPI 5.0
AMZ_ACCESS_KEY=your_access_key
AMZ_SECRET_KEY=your_secret_key
AMZ_ASSOC_TAG=your_affiliate_tag

# Stripe Integration  
STRIPE_SECRET_KEY=your_stripe_secret
STRIPE_WEBHOOK_SECRET=your_webhook_secret

# Security & Operations
CRON_SECRET=your_cron_secret
ADMIN_API_TOKEN=your_admin_token
```

## Security Considerations

### Development Guidelines
- **Never bypass RLS**: All database queries must respect Row Level Security policies
- **Rate limit integration**: New API endpoints must implement rate limiting
- **Anti-abuse checks**: User-facing features require abuse prevention integration
- **Comprehensive logging**: All operations must include structured debug logging
- **Webhook validation**: External webhooks require signature verification

### Data Access Patterns
- **Preview mode**: Limited data exposure for anonymous users
- **Authenticated mode**: Full data access for embed owners only
- **Admin access**: Requires admin email verification and token authentication
- **Cache separation**: Different strategies based on authentication state

## Performance Optimization

### Database Optimization
- Use materialized views for heavy analytics queries
- Implement proper indexing on ASIN, user_id, and timestamp columns
- Leverage pg_cron for automated maintenance and cleanup

### PAAPI Efficiency
- Only request necessary fields (price updates use minimal field sets)  
- Implement proper caching with 30-second per-ASIN cooldowns
- Use circuit breaker pattern for fault tolerance

### Frontend Performance
- Static HTML embed generation (not iframes)
- Lazy loading with efficient DOM manipulation
- Conditional CSS loading based on theme selection

## Testing & Validation

### Before Deployment
1. **Database migrations**: Verify with `npm run db:status`
2. **Rate limiting**: Test circuit breaker functionality
3. **Anti-abuse**: Validate risk assessment engine
4. **Analytics**: Confirm bot detection and retention policies
5. **Security headers**: Verify CSP and authentication policies

## Documentation References

Comprehensive documentation exists in `.claude/reference/` covering:
- `architecture.md` - System design and data flow
- `database-schema.md` - Complete database schema and patterns
- `data-flow.md` - System data flow documentation
- `prd.md` - Product requirements and specifications  
- `ux-flow.md` - User experience and interaction flows

This is a production-ready, enterprise-grade application with sophisticated security, comprehensive analytics, and automated operations requiring careful attention to rate limiting, anti-abuse integration, and security best practices.