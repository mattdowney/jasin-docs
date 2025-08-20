# Jasin

**Transform Amazon affiliate links into beautiful, embeddable product cards**

Jasin is a sophisticated, enterprise-grade SaaS platform that converts Amazon product URLs into clean, customizable product cards with comprehensive analytics, advanced security, anti-abuse protection, and automated maintenance systems.

## 🚀 Features

### Core Functionality
- **Amazon Product Integration** - Fetch real-time product data via Amazon PAAPI 5.0
- **Embeddable Cards** - Generate static HTML cards (not iframes) for any website
- **Theme Customization** - Light/dark themes with dynamic color extraction from product images
- **Real-time Analytics** - Track views, clicks, and conversion rates with bot detection
- **Subscription Management** - Freemium model with Stripe integration and webhook automation

### Advanced Security & Anti-Abuse
- **Device Fingerprinting** - SHA256-based browser/device tracking for abuse detection
- **Email Domain Reputation** - Scoring system (0-100) with disposable email blocking
- **Risk-Based Signup Protection** - Multi-factor risk assessment with automatic blocking
- **Credit Usage Pattern Analysis** - Detects suspicious consumption patterns
- **Silent Operation** - All protection happens in background with zero UX impact
- **Admin Monitoring Dashboard** - Real-time abuse signal tracking and management

### Enterprise Analytics
- **Bot Traffic Detection** - Advanced filtering with detailed bot categorization
- **90-Day Data Retention** - Automated cleanup with configurable retention policies
- **Materialized Views** - Optimized analytics performance with hourly refresh
- **Analytics Health Monitoring** - Dashboard showing data integrity and discrepancies
- **Audit Logging** - Comprehensive activity tracking for security and debugging

### PAAPI Rate Limiting & Circuit Breaker
- **Multi-Layer Protection** - Daily (8,640), hourly (360), and minute (6) rate limits
- **Circuit Breaker Pattern** - CLOSED/OPEN/HALF_OPEN states with automatic recovery
- **Per-ASIN Throttling** - 30-second cooldowns to prevent duplicate requests
- **Graceful Degradation** - Falls back to stale cache when limits are hit
- **Real-time Monitoring** - Admin dashboard with current usage and circuit breaker status

### Automated Operations
- **Price Monitoring** - Updates every 4 hours with efficient PAAPI usage
- **Color Extraction** - Dynamic theming based on product imagery (every 6 hours)
- **Carousel Support** - Multi-image product galleries with smooth transitions
- **Preview System** - Temporary embeds with automatic cleanup
- **Database Maintenance** - Automated cleanup, refresh, and optimization jobs

## 🏗️ Architecture

### Tech Stack
- **Frontend**: Next.js 14 (App Router), TypeScript, Tailwind CSS, shadcn/ui
- **Backend**: Next.js API Routes, Supabase (PostgreSQL), Row Level Security
- **Authentication**: Supabase Auth with email/password and secure session management
- **Payments**: Stripe with comprehensive webhook integration
- **External APIs**: Amazon Product Advertising API 5.0 with rate limiting
- **Deployment**: Vercel with automated cron jobs and edge functions
- **Security**: CSP headers, bot detection, device fingerprinting, abuse prevention

### Database Schema

#### Core Tables
- `products` - Cached Amazon product data (ASIN-indexed) with price history
- `embeds` - User-created product cards with themes, settings, and analytics
- `profiles` - User accounts with subscription tiers and usage tracking
- `embed_views`/`embed_clicks` - Detailed analytics events with bot detection
- `embed_stats`/`daily_stats` - Aggregated analytics with time-series data

#### Anti-Abuse Tables
- `device_fingerprints` - Browser/device tracking with SHA256 hashing
- `email_domain_reputation` - Domain scoring (0-100) and blocking lists
- `abuse_signals` - Suspicious activity logging with severity levels
- `credit_usage_patterns` - Usage pattern analysis and anomaly detection

#### System Tables
- `paapi_calls` - API usage tracking with rate limiting enforcement
- `color_extraction_queue` - Automated color processing queue
- `admin_logs` - Comprehensive audit trail for administrative actions

## 🔐 Authentication & Plans

### Free Tier
- 10 lifetime embed credits
- Basic themes only
- View/click analytics (no bot filtering)
- Default affiliate tags
- Standard support

### Pro Tier ($15/month or $120/year - Special Launch Pricing)
- 25 credits per month
- Advanced analytics dashboard with bot detection
- Custom themes and dynamic color extraction
- Custom affiliate tags
- Detailed traffic analysis and referrer data
- Priority support
- Analytics export capabilities

### Admin Features
- Comprehensive admin dashboard at `/admin`
- Anti-abuse monitoring and management
- Analytics health and integrity monitoring
- User management and subscription oversight
- System performance and rate limiting status
- Real-time abuse signal tracking

## 🚀 Getting Started

### Prerequisites
- Node.js 18+ and npm
- Supabase account and project
- Amazon Product Advertising API credentials
- Stripe account (for payments)
- Basic understanding of PostgreSQL and Row Level Security

### Installation

1. **Clone and install dependencies**
   ```bash
   git clone https://github.com/yourusername/jasin.git
   cd jasin
   npm install
   ```

2. **Environment setup**
   ```bash
   cp .env.example .env.local
   ```
   
   Required environment variables:
   ```env
   # Supabase
   NEXT_PUBLIC_SUPABASE_URL=your_supabase_url
   NEXT_PUBLIC_SUPABASE_ANON_KEY=your_anon_key
   SUPABASE_SERVICE_ROLE_KEY=your_service_role_key
   
   # Amazon PAAPI
   AMZ_ACCESS_KEY=your_access_key
   AMZ_SECRET_KEY=your_secret_key
   AMZ_ASSOC_TAG=your_affiliate_tag
   AMZ_PARTNER_TYPE=Associate
   
   # Stripe
   STRIPE_SECRET_KEY=your_stripe_secret
   STRIPE_WEBHOOK_SECRET=your_webhook_secret
   NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY=your_publishable_key
   STRIPE_PRO_MONTHLY_PRICE_ID=your_monthly_price_id
   STRIPE_PRO_YEARLY_PRICE_ID=your_yearly_price_id
   
   # Upstash Redis
   UPSTASH_REDIS_REST_URL=your_redis_url
   UPSTASH_REDIS_REST_TOKEN=your_redis_token
   
   # PostHog Analytics
   NEXT_PUBLIC_POSTHOG_KEY=your_posthog_project_key
   NEXT_PUBLIC_POSTHOG_HOST=https://us.i.posthog.com
   
   # Email Service
   RESEND_API_KEY=your_resend_api_key
   
   # Application URLs
   NEXT_PUBLIC_BASE_URL=http://localhost:3000
   NEXT_PUBLIC_APP_URL=https://getjasin.com
   
   # Security & Operations
   CRON_SECRET=your_cron_secret
   ADMIN_API_TOKEN=your_admin_token
   
   # Optional: CMS Integration
   WP_GRAPHQL_ENDPOINT=your_wordpress_graphql_endpoint
   
   # Optional: Slack Notifications
   SLACK_WEBHOOK_URL=your_slack_webhook_url
   
   # Optional: Mailerlite Integration
   MAILERLITE_API_KEY=your_mailerlite_key
   
   # Optional Debug
   DEBUG=api:*  # Enable debug logging
   ```

3. **Database setup**
   ```bash
   # Connect to remote Supabase (recommended)
   npm run db:setup-remote your_project_id
   
   # Sync migrations (includes anti-abuse system)
   npm run db:sync-migrations
   
   # Verify setup
   npm run db:status
   ```

4. **Start development server**
   ```bash
   npm run dev
   ```

## 📦 Database Management

### Migration Commands
```bash
# Create new migration
npm run db:create migration_name

# Push migrations to remote
npm run db:push

# Sync remote migrations locally
npm run db:sync-migrations

# Fix migration timestamps
npm run db:fix-all

# Verify migration status
npm run db:status
```

### Remote Supabase Access
This project uses **remote Supabase exclusively** - no local Docker setup required.

- **Dashboard**: Access via [Supabase Dashboard](https://app.supabase.com)
- **Direct DB Access**: Use connection details in `db-connection.txt`
- **SQL Queries**: Use dashboard SQL editor or pgAdmin/DBeaver
- **Admin Functions**: Access via `/admin` dashboard (requires admin email)

### Key Database Functions
```sql
-- Anti-abuse risk assessment
SELECT assess_signup_risk('email@domain.com', fingerprint_data);

-- Analytics with bot filtering
SELECT * FROM get_user_analytics('user-id', true);

-- Track embed events (v2 function)
CALL track_embed_event_with_transaction_v2(
  'embed-id', 'view', 'referer', 'user-agent', false, null, 'request-id'
);

-- Check rate limiting status
SELECT * FROM check_paapi_rate_limits();
```

## 🔧 Development Scripts

### Core Commands
```bash
npm run dev              # Start development server
npm run build            # Build for production
npm run start            # Start production server
npm run lint             # Run ESLint
npm run type-check       # TypeScript validation
```

### Database Operations
```bash
npm run db:status        # Check migration status
npm run db:sql           # Execute SQL queries
npm run db:reset         # Reset local database
npm run price:apply      # Apply price update migrations
npm run price:verify     # Verify price update jobs
```

### Stripe Integration
```bash
npm run dev:stripe       # Dev server with Stripe webhook listener
npm run stripe:listen    # Listen for Stripe webhooks
```

### Admin Operations
```bash
npm run admin:abuse      # Check anti-abuse statistics
npm run admin:analytics  # Verify analytics health
npm run admin:users      # User management operations
```

## 📊 Analytics System

### Advanced Tracking Implementation
- **View Tracking**: Automatic via embed.js Beacon API with fallback to fetch
- **Click Tracking**: Server-side through `/api/click/[id]` with redirect
- **Bot Detection**: Advanced filtering with 15+ bot categories and patterns
- **Data Retention**: 90-day policy with automated cleanup via pg_cron
- **Real-time Processing**: Transactional updates with consistency guarantees

### Analytics Functions (v2)
```sql
-- Get comprehensive user analytics
SELECT * FROM get_user_analytics('user-id', exclude_bots := true);

-- Track embed events with full context
CALL track_embed_event_with_transaction_v2(
  p_embed_id := 'embed-uuid',
  p_event_type := 'view',
  p_referer_url := 'https://example.com',
  p_user_agent := 'Mozilla/5.0...',
  p_is_bot := false,
  p_bot_type := null,
  p_request_id := 'req-12345'
);

-- Get bot analytics
SELECT * FROM get_user_bot_analytics('user-id');

-- Analytics health check
SELECT * FROM check_analytics_health();
```

### Analytics Dashboard Features
- **Real-time Statistics**: Views, clicks, CTR with bot filtering
- **Traffic Sources**: Detailed referrer analysis and geographic data
- **Performance Metrics**: Top-performing products and conversion tracking
- **Bot Traffic Analysis**: Separate bot analytics with categorization
- **Data Export**: CSV/JSON export for external analysis
- **Health Monitoring**: Data integrity checks and discrepancy detection

## 🛡️ Anti-Abuse System

### Risk Assessment Engine
```typescript
// Comprehensive signup risk evaluation
const riskAssessment = await assessSignupRisk(email, deviceFingerprint);

// Risk factors include:
// - Email domain reputation (0-100 score)
// - Device fingerprint duplicates
// - IP address patterns
// - Signup velocity from same source
// - Disposable email detection
```

### Device Fingerprinting
- **Browser Characteristics**: User-Agent, language, screen resolution, timezone
- **Network Information**: IP address with proxy detection
- **Behavioral Patterns**: Signup timing, usage patterns, domain preferences
- **SHA256 Hashing**: Secure fingerprint storage with privacy protection

### Email Domain Protection
- **Reputation Scoring**: 90+ for trusted domains (Gmail, Outlook, ProtonMail)
- **Disposable Email Blocking**: Automatic detection and blocking
- **Domain Abuse Tracking**: Reputation degradation based on abuse reports
- **Whitelist Management**: Admin-controlled trusted domain lists

### Abuse Signal Detection
```sql
-- Record suspicious activity
INSERT INTO abuse_signals (user_id, signal_type, severity, details)
VALUES ('user-id', 'rapid_credit_usage', 'high', '{"credits_used": 5, "time_span": "10 minutes"}');

-- Severity levels: low, medium, high, critical
-- Automatic blocking at 90+ risk score
-- Flagging at 60+ risk score for monitoring
```

### Admin Anti-Abuse Dashboard
- **Real-time Monitoring**: Live abuse signal feed with severity indicators
- **Device Fingerprint Analysis**: Duplicate detection and user correlation
- **Blocked Domain Management**: Add/remove domains from reputation system
- **Risk Score Visualization**: User risk distribution and trending
- **Incident Response**: Tools for investigating and responding to abuse

## 🔄 PAAPI Rate Limiting & Circuit Breaker

### Multi-Layer Rate Limiting
```typescript
// Conservative rate limits (85% of Amazon's actual limits)
const RATE_LIMITS = {
  DAILY_LIMIT: 8640,        // Amazon's daily limit
  HOURLY_LIMIT: 360,        // Conservative hourly (8640/24)
  MINUTE_LIMIT: 6,          // Conservative minute limit
  ASIN_COOLDOWN_MS: 30000,  // 30 seconds between same ASIN
};
```

### Circuit Breaker Implementation
- **CLOSED State**: Normal operation, all requests allowed
- **OPEN State**: Service failing, requests blocked for 5 minutes
- **HALF_OPEN State**: Testing recovery with limited requests
- **Failure Threshold**: 5 consecutive failures trigger circuit opening
- **Automatic Recovery**: Self-healing with exponential backoff

### Graceful Degradation
```typescript
// Fallback strategy when rate limited
if (rateLimited) {
  // 1. Try stale cached data
  if (cachedProduct && isStale) {
    return { product: cachedProduct, isStale: true };
  }
  // 2. Return user-friendly error with retry time
  return { error: 'High demand, try again in X minutes' };
}
```

### Rate Limiter Admin Dashboard
- **Current Usage**: Real-time daily/hourly/minute usage percentages
- **Circuit Breaker Status**: Current state and failure count
- **ASIN Throttle Map**: Active cooldowns and request patterns
- **Historical Data**: Usage trends and limit approach warnings

## 🎨 Embed System

### Static HTML Generation
Jasin generates self-contained HTML embeds (not iframes) with:
- **Responsive Design**: Mobile-first with breakpoint optimization
- **Theme-Aware Styling**: Dynamic CSS loading based on theme selection
- **Affiliate Link Integration**: Automatic affiliate tag injection and tracking
- **Analytics Tracking**: Built-in view/click tracking with bot detection
- **Carousel Support**: Smooth image transitions with thumbnail navigation
- **Error Handling**: Graceful fallbacks and user-friendly error messages

### Embed Script (38KB)
The `embed.js` script provides:
- **Dynamic Card Rendering**: Real-time product data fetching and display
- **Theme Loading**: Conditional CSS loading for light/dark themes
- **View Tracking**: Beacon API with fetch fallback for analytics
- **Error Handling**: Comprehensive error catching and user feedback
- **Debug Mode**: Detailed logging for development and troubleshooting
- **Performance Optimization**: Lazy loading and efficient DOM manipulation

### Advanced Embed Features
```html
<script
  async
  src="https://getjasin.com/embed.js"
  data-id="embed-uuid"
  data-theme="light"
  data-show-badge="true"

  data-debug="false">
</script>
```

### Carousel Implementation
- **Main Image Carousel**: Horizontal sliding with scale transitions (1.0 selected, 0.8 others)
- **Thumbnail Navigation**: Up to 4 visible thumbnails with arrow navigation
- **Synchronization**: Clicking thumbnails updates main carousel smoothly
- **Animation**: 500ms transitions with CSS transform optimization
- **Accessibility**: Keyboard navigation and screen reader support

## 🔄 Automated Operations

### Cron Jobs (Vercel)
```json
// vercel.json cron configuration
{
  "crons": [
    {
      "path": "/api/cron/update-prices-efficient",
      "schedule": "0 */4 * * *"  // Every 4 hours
    },
    {
      "path": "/api/admin/update-product-colors",
      "schedule": "0 */6 * * *"  // Every 6 hours
    }
  ]
}
```

### Database Maintenance (pg_cron)
```sql
-- Weekly analytics cleanup (90+ day retention)
SELECT cron.schedule('clean-analytics-data-weekly', '0 0 * * 0', 
  'SELECT clean_old_analytics_data(90);');

-- Hourly materialized view refresh
SELECT cron.schedule('refresh-analytics-summary-hourly', '0 * * * *',
  'REFRESH MATERIALIZED VIEW CONCURRENTLY mv_user_analytics_summary;');

-- Daily preview embed cleanup
SELECT cron.schedule('clean-preview-embeds-daily', '0 3 * * *',
  'SELECT clean_old_preview_embeds();');
```

### Price Update System
- **Efficient PAAPI Usage**: Only requests price-related fields, not full product data
- **Batch Processing**: Configurable batch sizes (default: 25 products)
- **Queue Management**: Database-driven queue with retry logic
- **Change Detection**: Only updates when prices actually change
- **Compliance**: Ensures 24-hour price update requirement per Amazon TOS

### Color Extraction Pipeline
- **Dynamic Theme Generation**: Extracts dominant colors from product images
- **Queue-Based Processing**: Handles large volumes with concurrency limits
- **Error Handling**: Graceful fallbacks for failed extractions
- **Cache Optimization**: Stores extracted colors for theme generation

## 🛡️ Security

### Comprehensive Security Implementation
- **Row Level Security (RLS)** on all user data with granular policies
- **Content Security Policy (CSP)** headers preventing XSS and data exfiltration
- **API rate limiting** with circuit breaker pattern and quota enforcement
- **Webhook signature verification** for Stripe with replay attack prevention
- **Bot detection** and filtering with 15+ detection patterns
- **Device fingerprinting** with privacy-preserving SHA256 hashing

### Access Control Matrix
| User Type | Preview Cards | Create Embeds | Analytics | Admin Dashboard | Anti-Abuse |
|-----------|---------------|---------------|-----------|-----------------|------------|
| Anonymous | ✅ | ❌ | ❌ | ❌ | ❌ |
| Free User | ✅ | ✅ (5 total) | Basic | ❌ | ❌ |
| Pro User | ✅ | ✅ (25/month) | Advanced | ❌ | ❌ |
| Admin | ✅ | ✅ (Unlimited) | Full | ✅ | ✅ |

### Security Headers
```typescript
// Comprehensive security headers
const securityHeaders = {
  'X-Content-Type-Options': 'nosniff',
  'X-Frame-Options': 'SAMEORIGIN',
  'X-XSS-Protection': '1; mode=block',
  'Referrer-Policy': 'strict-origin-when-cross-origin',
  'Content-Security-Policy': "default-src 'self'; script-src 'self'; ...",
  'Vary': 'Authorization, Cookie'
};
```

### Preview vs. Authenticated Access
- **Preview Mode**: Limited data exposure, no sensitive information
- **Authenticated Mode**: Full data access for embed owners
- **Cache Separation**: Different caching strategies based on auth state
- **Data Minimization**: Only necessary fields exposed in each mode

## 📝 Logging & Debugging

### Structured Logging System
```bash
# Enable specific modules
DEBUG=api:products,api:parse,anti-abuse npm run dev

# Enable all debug logs
DEBUG=* npm run dev

# Production logging
DEBUG=api:errors,stripe-webhook,analytics npm run start
```

### Log Categories
- `api:*` - API route debugging with request/response details
- `amazon` - PAAPI integration with rate limiting status
- `stripe-webhook` - Payment processing and subscription updates
- `analytics` - Tracking events and bot detection
- `anti-abuse` - Risk assessment and abuse signal logging
- `rate-limiter` - Circuit breaker state and usage monitoring

### Debug Features
- **Request Tracing**: Unique request IDs for cross-service correlation
- **Performance Monitoring**: Response times and database query analysis
- **Error Aggregation**: Structured error logging with context
- **Admin Dashboards**: Real-time system health and performance metrics

## 🚀 Deployment

### Vercel Configuration
```json
// vercel.json
{
  "functions": {
    "src/app/api/**/*.ts": {
      "maxDuration": 30
    }
  },
  "crons": [
    {
      "path": "/api/cron/update-prices-efficient",
      "schedule": "0 */4 * * *"
    }
  ]
}
```

### Environment Variables (Production)
```env
# Core Services
NEXT_PUBLIC_SUPABASE_URL=https://your-project.supabase.co
SUPABASE_SERVICE_ROLE_KEY=your-service-role-key
STRIPE_SECRET_KEY=sk_live_...
AMZ_ACCESS_KEY=your-paapi-access-key
AMZ_SECRET_KEY=your-paapi-secret-key
AMZ_ASSOC_TAG=your-affiliate-tag

# Upstash Redis
UPSTASH_REDIS_REST_URL=your-redis-url
UPSTASH_REDIS_REST_TOKEN=your-redis-token

# PostHog Analytics
NEXT_PUBLIC_POSTHOG_KEY=your-posthog-key
NEXT_PUBLIC_POSTHOG_HOST=https://us.i.posthog.com

# Email Service
RESEND_API_KEY=your-resend-key

# Security & Operations
CRON_SECRET=secure-random-string
ADMIN_API_TOKEN=admin-access-token
STRIPE_WEBHOOK_SECRET=stripe-webhook-secret

# Application URLs
NEXT_PUBLIC_APP_URL=your-domain.com
```

### Production Checklist
- [ ] Environment variables configured in Vercel
- [ ] Supabase RLS policies enabled and tested
- [ ] Stripe webhooks configured with proper endpoints
- [ ] Amazon PAAPI credentials validated and rate limits configured
- [ ] Cron job secrets set and endpoints secured
- [ ] Domain DNS configured with SSL certificates
- [ ] Admin email addresses configured for dashboard access
- [ ] Anti-abuse system enabled and thresholds configured
- [ ] Analytics retention policies configured
- [ ] Rate limiting thresholds set appropriately for production traffic

### Monitoring & Alerts
- **Vercel Analytics**: Performance monitoring and error tracking
- **Supabase Monitoring**: Database performance and query analysis
- **Custom Dashboards**: Admin interfaces for system health
- **Error Reporting**: Structured logging with alert thresholds
- **Rate Limit Monitoring**: Circuit breaker state and usage alerts

## 📚 Documentation

### System Architecture & Design
- [**Architecture Overview**](./.cursor/rules/architecture.mdc) - System design, data flow, and tech stack
- [**Database Patterns**](./.cursor/rules/database-patterns.mdc) - Schema design, migration patterns, and RLS policies
- [**Security Patterns**](./.cursor/rules/security-patterns.mdc) - Comprehensive security implementation
- [**Security Audit**](./.cursor/rules/security-audit.mdc) - Security assessment and best practices

### Core Features & Implementation
- [**Anti-Abuse System**](./.cursor/rules/anti-abuse-system.mdc) - Fraud prevention and risk assessment
- [**PAAPI Rate Limiting**](./.cursor/rules/paapi-rate-limiting.mdc) - Circuit breaker and rate limiting system
- [**Analytics Tracking System**](./.cursor/rules/analytics-tracking-system.mdc) - Event tracking and bot detection
- [**Embeddable Card System**](./.cursor/rules/embeddable-card-system.mdc) - Static HTML card generation
- [**Carousel Implementation**](./.cursor/rules/carousel-implementation.mdc) - Multi-image gallery system

### Supabase Integration
- [**Remote Setup**](./.cursor/rules/supabase-remote-setup.mdc) - Remote Supabase configuration
- [**Remote Access**](./.cursor/rules/supabase-remote-access.mdc) - Database connection and access patterns
- [**Migration Management**](./.cursor/rules/supabase-migration-management.mdc) - Migration workflow and best practices
- [**Database Schema**](./.cursor/rules/supabase-database-schema.mdc) - Complete schema documentation
- [**Row Level Security**](./.cursor/rules/supabase-row-level-security.mdc) - RLS policies and access control
- [**Database Functions**](./.cursor/rules/supabase-database-functions.mdc) - Custom PostgreSQL functions
- [**Analytics System**](./.cursor/rules/supabase-database-analytics-system.mdc) - Database-level analytics
- [**Product Data Management**](./.cursor/rules/supabase-product-data-management.mdc) - Product caching and updates
- [**pg_cron Jobs**](./.cursor/rules/supabase-pg-cron-jobs.mdc) - Automated database maintenance
- [**Client Integration**](./.cursor/rules/supabase-client-integration.mdc) - Frontend integration patterns

### Authentication & User Management
- [**Auth Session Management**](./.cursor/rules/supabase-auth-session.mdc) - Session handling and security
- [**Password Change Flow**](./.cursor/rules/supabase-password-change.mdc) - Secure password updates
- [**Email Change Flow**](./.cursor/rules/supabase-email-change.mdc) - Email update process
- [**MCP Tools**](./.cursor/rules/supabase-mcp-tools.mdc) - Management and control plane tools

### Subscription & Monetization
- [**Subscription UI Patterns**](./.cursor/rules/subscription-ui-patterns.mdc) - Frontend subscription components
- [**Stripe Webhooks**](./.cursor/rules/stripe-subscription-webhooks.mdc) - Payment processing integration
- [**Database Schema**](./.cursor/rules/database-subscription-schema.mdc) - Subscription data modeling
- [**Monetization Strategy**](./.cursor/rules/monetization.mdc) - Business model and pricing
- [**Credits System**](./.cursor/rules/credits.mdc) - Credit allocation and tracking

### Development & Operations
- [**Frontend Patterns**](./.cursor/rules/frontend.mdc) - React/Next.js development patterns
- [**Backend Patterns**](./.cursor/rules/backend.mdc) - API development and server-side logic
- [**Logging System**](./.cursor/rules/logging.mdc) - Structured logging and debugging
- [**Environment File Handling**](./.cursor/rules/env-file-handling.mdc) - Configuration management
- [**Price Update System**](./.cursor/rules/price-update-system.mdc) - Automated price monitoring

### Product & UX Documentation
- [**Product Requirements**](./.cursor/rules/prd.mdc) - Product specification and requirements
- [**Project Overview**](./.cursor/rules/jasin.mdc) - High-level project description
- [**UX Flow**](./.cursor/rules/ux-flow.mdc) - User experience and interaction flows
- [**UX Flow Diagram**](./.cursor/rules/ux-flow-diagram.mdc) - Visual user journey mapping
- [**Sitemap**](./.cursor/rules/sitemap.mdc) - Site structure and navigation
- [**Data Flow Diagram**](./.cursor/rules/data-flow-diagram.mdc) - System data flow visualization

### Analytics & Data Management
- [**Analytics Overview**](./.cursor/rules/analytics.mdc) - Analytics strategy and implementation
- [**Data Retention**](./.cursor/rules/data-retention.mdc) - Data lifecycle and cleanup policies

### External API Documentation
- [**Amazon PAAPI 5.0**](./docs/paapi5.0/README.md) - Amazon Product Advertising API integration
- [**PAAPI SDK Examples**](./docs/paapi5.0/) - Sample implementations and test scripts

## 🤝 Contributing

### Development Process
1. Fork the repository
2. Create a feature branch (`git checkout -b feature/amazing-feature`)
3. Make your changes with comprehensive tests
4. Ensure all security checks pass
5. Update documentation as needed
6. Submit a pull request with detailed description

### Code Standards
- **TypeScript**: Strict mode with comprehensive type coverage
- **ESLint**: Enforced code style and best practices
- **Security**: All new features must include security considerations
- **Testing**: Unit tests for business logic, integration tests for APIs
- **Documentation**: All new features require documentation updates

### Security Considerations
- All database changes must include RLS policy updates
- New API endpoints require rate limiting and authentication
- Anti-abuse system integration for user-facing features
- Comprehensive logging for audit and debugging purposes

## 📄 License

This project is proprietary software. All rights reserved.

## 🆘 Support

### Documentation & Resources
- **System Documentation**: Check the `/docs` directory for comprehensive guides
- **API Reference**: Complete API documentation with examples
- **Admin Dashboard**: Real-time system monitoring and management tools
- **Debug Tools**: Built-in debugging and performance monitoring

### Getting Help
- **Issues**: Create GitHub issues for bugs with detailed reproduction steps
- **Security**: Report security issues privately to security@getjasin.com
- **General Support**: Contact support@getjasin.com for assistance
- **Enterprise**: Contact enterprise@getjasin.com for custom solutions

### System Status
- **Health Dashboard**: `/admin` for real-time system status
- **Rate Limiting**: Monitor PAAPI usage and circuit breaker status
- **Analytics Health**: Verify data integrity and processing status
- **Anti-Abuse**: Monitor security events and risk assessments

---

**Jasin** - Enterprise-grade Amazon product cards with comprehensive security, analytics, and abuse prevention for the modern web. 