# Achilleus Execution Plan

## Project Overview
Build a security monitoring SaaS from scratch using Laravel 12 with React 19 starter kit. The application provides automated security scanning for up to 10 domains at $27/month, targeting developers and freelancers who need affordable security monitoring.

**Key Technologies**:
- Laravel 12: https://laravel.com/docs/12.x
- React 19: https://react.dev/
- Laravel Starter Kits: https://laravel.com/docs/12.x/starter-kits
- Laravel Cloud: https://cloud.laravel.com/
- Shadcn/ui: https://ui.shadcn.com/
- Laravel Reverb: https://reverb.laravel.com/

**Target Timeline**: 21 days from empty repository to production MVP
**Development Approach**: Test-Driven Development (TDD) with security-first implementation
**Starter Kit**: Laravel 12 with React starter kit from https://laravel.com/docs/12.x/starter-kits

## Development Philosophy

### Core Principles
- **Security First**: Every feature must prioritize security over convenience
- **Test-Driven Development**: Write tests before implementation
- **Progressive Enhancement**: Build core functionality first, polish later
- **Database-Driven UI**: All UI components must be backed by real data
- **SSRF Protection**: All external requests must be validated

## Development Workflow

For EVERY subtask, follow this workflow:

```bash
# 1. PLAN - Understand requirements
achilleus-plan [phase].[subtask]   # Review requirements, check technical.md

# 2. TEST - Write tests first (TDD)
achilleus-test [phase].[subtask]   # Write comprehensive tests

# 3. CODE - Implement functionality
achilleus-code [phase].[subtask]   # Build based on tests

# 4. VERIFY - Ensure quality
achilleus-verify [phase].[subtask] # Run tests, check performance
```

---

## Phase 0: Repository Setup (Day 0 - Pre-Development)

### ✅ DO
- Use Laravel 12 with React 19 starter kit
- Install all required packages before starting development
- Configure dark theme from the beginning
- Set up proper git workflow with meaningful commits
- Initialize testing infrastructure early

### ❌ DON'T
- Skip package version verification
- Use outdated Laravel or React versions  
- Forget to configure dark theme initially
- Start coding without proper project structure

### Subtasks

#### 0.1 Initial Laravel Project Setup
- Create new directory `achilleus`
- Install Laravel 12 with React starter kit
- Verify React 19, Inertia.js, and Tailwind CSS are included
- Confirm TypeScript configuration is present
- Test default authentication pages work

#### 0.2 Package Installation  
- Install PDF generation (barryvdh/laravel-dompdf)
- Install Stripe integration (stripe/stripe-php)
- Install development tools (laravel/pint, pestphp/pest)
- Install Playwright for E2E testing
- Run playwright setup

#### 0.3 Shadcn/ui Configuration
- Initialize Shadcn/ui with dark theme
- Install core components (button, card, form, input, select, badge, alert)
- Install additional components (dialog, table, tabs, progress, tooltip)
- Verify dark theme classes in tailwind.config.js
- Test component rendering

#### 0.4 Project Structure Setup
- Create marketing directory for landing page
- Download and configure Salient template
- Create scanner directory structure (app/Contracts/Scanners, app/Data, app/Support)
- Set up testing directories (Unit, Feature, E2E)
- Configure .env.example with all required variables

#### 0.5 Version Control Setup
- Verify git repository is initialized
- Create comprehensive .gitignore
- Make initial commit with all setup
- Add remote repository
- Push to main branch

---

## Phase 1: Foundation & Security (Days 1-2)

### ✅ DO
- Create all database migrations before models
- Implement SSRF protection before any external calls
- Test with real data, not mocks
- Use proper Laravel conventions
- Follow TDD strictly

### ❌ DON'T
- Skip writing tests first
- Make external HTTP requests without NetworkGuard
- Create models without migrations
- Hardcode any configuration values
- Skip validation on user inputs

### What to Test
- Database schema constraints and relationships
- SSRF protection with localhost, private IPs (192.168.x, 10.x), cloud metadata (169.254.169.254)
- Trial period calculation (exactly 14 days from registration)
- Model relationships and scopes
- Configuration values and defaults

### What NOT to Test
- Laravel framework functionality
- Third-party package internals
- Database engine behavior
- Simple getters/setters without logic

### Subtasks

#### 1.1 Database Schema
- Users table with trial_ends_at, stripe_customer_id, subscription_status
- Domains table with url, email_mode, last_scan_score, is_active
- Scans table with status, total_score, grade, timestamps
- Scan_modules table with module type, score, status, raw JSONB data
- Reports table with filename, file_path, file_size

#### 1.2 Models & Relationships
- User model with domains(), scans(), reports() relationships
- Domain model with user(), scans(), latestScan() relationships
- Scan model with domain(), user(), modules() relationships
- ScanModule model with scan() relationship
- Model factories for all models

#### 1.3 Security Infrastructure (NetworkGuard)
- NetworkGuard class for SSRF protection
- Block localhost (127.0.0.1, ::1)
- Block private IPs (192.168.0.0/16, 10.0.0.0/8, 172.16.0.0/12)
- Block cloud metadata endpoints (169.254.169.254)
- Custom exception classes (SSRFException, TimeoutException, RateLimitException)

#### 1.4 Authentication Enhancement
- Registration sets trial_ends_at to 14 days from now
- Trial banner component for authenticated users
- Shadcn/ui components in auth pages
- Subscription status tracking
- Trial expiry calculations

#### 1.5 Configuration Setup
- Scoring config with weights (SSL: 40%, Headers: 30%, DNS: 30%)
- Grade thresholds (A+: 95+, A: 90+, B+: 85+, B: 80+, C: 70+, D: 60+, F: <60)
- Domain limits (max 10 per user)
- Rate limits (10 scans/minute per user)
- Environment variables for all services

---

## Phase 2: Scanner Implementation (Days 3-5)

### ✅ DO
- Test with real external services (badssl.com, github.com)
- Implement proper timeout and retry mechanisms
- Follow the AbstractScanner pattern for all scanners
- Use SSRF protection on every external request
- Test both success and failure scenarios

### ❌ DON'T
- Mock external services for integration tests
- Skip timeout configuration
- Hardcode scoring weights
- Make synchronous external requests
- Ignore SSL certificate validation

### What to Test
- Scanner behavior with expired certificates (expired.badssl.com)
- Scanner behavior with wrong hostname (wrong.host.badssl.com)
- Header detection with well-configured sites (github.com)
- DNS resolution with real domains
- Timeout and retry mechanisms
- Rate limiting enforcement

### What NOT to Test
- External service availability
- DNS server response times
- Third-party SSL libraries
- Network connectivity

### Subtasks

#### 2.1 Scanner Architecture
- Scanner interface with name(), getWeight(), scan() methods
- ModuleResult data class with score, status, raw data
- AbstractScanner base class with retry logic
- Rate limiting per scanner per host
- Scanner service provider for dependency injection

#### 2.2 SSL/TLS Scanner (40% weight)
- Certificate expiration checking (0 points if expired)
- Hostname verification (CN and SAN)
- Protocol version detection (TLS 1.3 bonus, TLS 1.0/1.1 penalty)
- Cipher suite strength analysis
- Key size validation (minimum 2048-bit RSA)

#### 2.3 Security Headers Scanner (30% weight)
- HSTS detection and max-age validation
- CSP presence and unsafe directive detection
- X-Frame-Options verification
- X-Content-Type-Options checking
- Referrer-Policy assessment

#### 2.4 DNS/Email Scanner (30% weight)
- SPF record validation
- DKIM verification with custom selectors
- DMARC policy analysis
- MX record checking (if email_mode='expected')
- CAA record verification

#### 2.5 Scoring Engine
- Weighted score calculation
- Grade assignment based on thresholds
- Weight redistribution when modules fail
- Score caching mechanism
- Historical score tracking

---

## Phase 3: Scan Processing & Jobs (Days 6-7)

### ✅ DO
- Implement async job processing with Laravel Cloud workers
- Add comprehensive logging for debugging
- Handle failures gracefully with retry logic
- Update scan status throughout the process
- Use Laravel Cloud's auto-scaling queue workers

### ❌ DON'T
- Process scans synchronously
- Skip job failure handling
- Forget status updates
- Skip timeout configuration
- Manually manage queue infrastructure

### What to Test
- Job dispatch and processing logic
- Retry logic with failures
- Scan status transitions (pending → running → completed/failed)
- Job timeout handling
- Scanner orchestration

### What NOT to Test
- Laravel Cloud queue infrastructure
- Worker auto-scaling
- Redis connection

### Subtasks

#### 3.1 Scan Job Implementation (RunDomainScan)
- Job class with proper timeout (120 seconds)
- Retry configuration (3 attempts, exponential backoff)
- Scan status updates (pending → running → completed/failed)
- Scanner execution orchestration
- Error message capture and logging
- Domain last_scan_score update

#### 3.2 Scan Orchestrator Service
- Scanner dependency injection
- Sequential scanner execution with error isolation
- Individual scanner failure handling
- Score aggregation and grade calculation
- Module result storage in JSONB
- Scan completion notifications preparation

#### 3.3 HTTP Client Service
- Secure HTTP client with NetworkGuard integration
- Timeout configuration (30 seconds default)
- Retry logic (2 attempts with exponential backoff)
- User agent setting for scanner identification
- HTTPS-only redirect following
- Response caching for duplicate requests

#### 3.4 Rate Limiting Implementation
- User-level: 10 scans/minute via Laravel middleware
- Scanner-level: Per-host limits in AbstractScanner
- SSL Scanner: 5 requests/minute per host
- Headers Scanner: 8 requests/minute per host
- DNS Scanner: 15 requests/minute per host
- Rate limit exceeded handling with proper errors

#### 3.5 Background Processing Setup
- Configure Laravel Cloud queue workers in deployment
- Set appropriate memory and timeout limits
- Failed job handling and notifications
- Job monitoring and health checks
- Queue priority for different job types

---

## Phase 4: Complete Backend API (Days 8-9)

### ✅ DO
- Validate all URLs are HTTPS only
- Enforce 10 domain limit strictly
- Normalize URLs for deduplication
- Use proper authorization policies
- Implement domain ownership validation

### ❌ DON'T
- Allow HTTP domains
- Skip domain ownership validation
- Allow duplicate domains
- Trust user input without validation
- Bypass domain limits

### What to Test
- Domain limit enforcement (max 10)
- URL normalization (remove www, trailing slash)
- HTTPS-only validation
- Authorization policies
- Duplicate prevention

### What NOT to Test
- DNS resolution of domains
- SSL certificate validity
- Domain registrar APIs

### Subtasks

#### 4.1 Domain Management API
- Domain model with URL normalization (lowercase, remove protocol/www/trailing slash)
- Domain scopes (active, forUser) and computed attributes
- StoreDomainRequest with HTTPS-only and SSRF validation
- DomainController with full CRUD operations
- Domain policy for authorization

#### 4.2 Scan Management API
- DomainScanController for triggering scans
- Single domain scan endpoint
- Bulk scan all domains endpoint
- Scan status and history endpoints
- Real-time scan progress via Reverb preparation

#### 4.3 Dashboard API
- DashboardController with metrics endpoints
- Average security score calculation
- Active domains count and critical issues
- Recent scan activity endpoint
- Trend data for last 7/30/90 days

#### 4.4 Reports API
- ReportController for PDF generation
- Report generation job dispatch
- Report download with signed URLs
- Report history and management
- Generation limits (10/day) enforcement

#### 4.5 User & Billing API
- Profile update endpoints
- Subscription status endpoint
- Trial days remaining calculation
- Payment method management preparation
- Billing history endpoint preparation

---

## Phase 5: Complete Frontend UI (Days 10-13)

### ✅ DO
- Build ALL pages to create a complete MVP
- Connect every UI component to real backend APIs
- Implement all user flows end-to-end
- Use Shadcn/ui consistently throughout
- Ensure every feature is usable and testable

### ❌ DON'T
- Leave any pages as "coming soon"
- Use placeholder/fake data anywhere
- Skip any critical user flows
- Leave broken navigation links
- Deploy without complete functionality

### What to Test
- Dashboard data aggregation accuracy
- Empty state rendering
- Navigation flow
- Responsive breakpoints
- Data caching

### What NOT to Test
- CSS styling details
- Animation timing
- Browser rendering
- Shadcn/ui internals

### Subtasks

#### 5.1 Core Layout & Navigation
- AuthenticatedLayout with dark theme
- Sidebar navigation (Dashboard, Domains, Activity, Reports, Billing)
- User menu dropdown (Profile, Billing, Sign Out)
- Mobile responsive hamburger menu
- Trial/subscription status banner

#### 5.2 Dashboard Page (Complete)
- Four metric cards with real data (Score, Domains, Last Scan, Critical Issues)
- Security trends chart using Recharts (7/30/90 day views)
- Recent activity feed with scan results
- Quick actions (Add Domain, Run All Scans, Generate Report)
- Empty state for new users with CTA

#### 5.3 Domains Management (Complete)
- Domains/Index with DataTable component
- Add Domain dialog with React Hook Form + Zod
- Domain detail modal with scan history
- Scan trigger button with loading states
- Bulk actions (scan all, delete selected)
- Real-time scan progress with Reverb

#### 5.4 Activity & Scan Results (Complete)
- Activity/Index with filterable scan history
- Scan detail page with module breakdowns
- Security grade display with color coding
- Issue list grouped by severity
- Recommendations with actionable steps
- Export to CSV functionality

#### 5.5 Reports Page (Complete)
- Reports/Index with generation history
- Generate Report dialog with options
- Download buttons with loading states
- Report preview (if feasible)
- Generation limit indicator (X/10 today)

#### 5.6 Billing Page (Complete)
- Current plan display (Trial/Active/Cancelled)
- Days remaining in trial with upgrade CTA
- Subscription management buttons
- Payment method section (placeholder for Stripe)
- Billing history table
- Cancel subscription flow

#### 5.7 Profile & Settings (Complete)
- Profile edit form (name, email, timezone)
- Password change form
- Email notification preferences
- Account deletion option
- API keys section (future enhancement)

#### 5.8 Empty States & Error Handling
- Empty states for all lists with helpful CTAs
- Loading skeletons for all async operations
- Error boundaries with user-friendly messages
- 404 page with navigation back
- Offline state handling

---

## Phase 6: Billing & Stripe Integration (Days 14-15)

### ✅ DO
- Use Stripe's direct API (no Cashier)
- Verify webhook signatures
- Handle all subscription states
- Test with Stripe test mode
- Implement idempotency

### ❌ DON'T
- Store payment data locally
- Skip webhook verification
- Ignore subscription states
- Use production keys in development
- Allow access after expiry

### What to Test
- Subscription creation flow
- Payment method updates
- Webhook signature verification
- Trial to paid conversion
- Subscription cancellation

### What NOT to Test
- Stripe API internals
- Payment processing
- Bank transfers
- Credit card validation

### Subtasks

#### 6.1 Stripe Service (Direct API)
- Customer creation method
- Subscription creation with trial
- Payment method attachment
- Subscription cancellation
- Invoice retrieval

#### 6.2 Subscription Controllers
- SubscriptionController for management
- CheckoutController for payment flow
- Trial to paid conversion handling
- Payment method update endpoint
- Cancellation endpoint

#### 6.3 Webhook Processing
- StripeWebhookController
- Signature verification
- Event handlers (subscription.created, invoice.payment_succeeded, etc.)
- Idempotency handling
- Webhook event logging

#### 6.4 Billing UI
- Billing/Index page
- Current plan display
- Payment method management
- Billing history
- Cancellation flow

#### 6.5 Access Control
- SubscriptionMiddleware
- User helper methods (hasActiveSubscription, isInTrial)
- Domain limit gates
- Grace period handling
- Trial expiry warnings

---

## Phase 7: Testing & Polish (Days 16-18)

### ✅ DO
- Test the complete user journey end-to-end
- Verify all features work as expected
- Polish UI/UX rough edges
- Fix any bugs discovered
- Optimize performance bottlenecks

### ❌ DON'T
- Add new features at this stage
- Skip testing edge cases
- Ignore error scenarios
- Deploy with known bugs
- Skip performance testing

### What to Test
- PDF generation with various data states
- File storage and retrieval
- Access control validation
- Generation limits (10/day)
- Async job processing

### What NOT to Test
- PDF rendering accuracy
- Font rendering
- File system operations
- S3 upload mechanics

### Subtasks

#### 7.1 End-to-End Testing
- Complete user signup and trial flow
- Add and scan multiple domains
- View scan results and history
- Generate and download reports
- Test billing upgrade flow
- Verify subscription cancellation

#### 7.2 PDF Report Implementation
- Professional report templates with Achilleus branding
- ReportGeneratorService with DomPDF
- Async GenerateReportJob with status updates
- Secure storage with signed URLs
- Report management UI completion

#### 7.3 Performance Optimization
- Dashboard query optimization (N+1 prevention)
- Frontend bundle size reduction
- Image optimization and lazy loading
- API response time improvements
- Cache implementation for frequent queries

#### 7.4 UI/UX Polish
- Consistent spacing and typography
- Smooth transitions and animations
- Form validation feedback improvements
- Loading state refinements
- Dark theme consistency check

#### 7.5 Bug Fixes & Edge Cases
- Fix issues discovered during testing
- Handle network timeout scenarios
- Improve error messages
- Test with various data states
- Browser compatibility testing

---

## Phase 8: Production Deployment (Days 19-20)

### ✅ DO
- Deploy to Laravel Cloud production
- Configure all environment variables
- Set up monitoring and alerts
- Verify all features in production
- Create backup procedures

### ❌ DON'T
- Deploy without testing locally
- Skip SSL verification
- Forget webhook configurations
- Ignore error tracking setup
- Deploy without backups

### What to Test
- Complete user journey (signup → scan → report → billing)
- Performance against targets
- Security vulnerabilities
- Load handling
- Error recovery

### What NOT to Test
- Third-party service uptime
- Network latency
- Browser performance
- External API limits

### Subtasks

#### 8.1 Laravel Cloud Setup
- Create project at cloud.laravel.com
- Configure PostgreSQL database with hibernation
- Set up Redis for cache and queues
- Configure queue workers with auto-scaling
- Set up S3 storage for PDF reports

#### 8.2 Environment Configuration
- Set all production environment variables
- Configure Stripe production keys
- Set up Stripe webhook endpoints
- Configure domain and SSL
- Set up Laravel Reverb for WebSockets

#### 8.3 Deployment & Verification
- Deploy application code
- Run database migrations
- Verify all features work in production
- Test payment flow with real card
- Confirm email delivery (support@achilleus.so)

#### 8.4 Monitoring & Alerts
- Configure application monitoring
- Set up error tracking (Sentry/Bugsnag)
- Configure uptime monitoring
- Set up performance alerts
- Database backup verification

#### 8.5 Security Hardening
- Verify HTTPS enforcement
- Test security headers
- Confirm rate limiting works
- Validate CORS settings
- Run security scan on production

---

## Phase 9: Landing Page & Launch (Day 21)

### ✅ DO
- Create compelling landing page
- Add legal pages (Terms, Privacy)
- Implement SEO optimization
- Set up analytics
- Prepare for launch

### ❌ DON'T
- Launch without legal pages
- Skip SEO basics
- Forget analytics setup
- Launch without testing signup flow
- Skip social meta tags

### What to Test
- Landing page loading
- Error page display
- Health check endpoints
- Backup procedures
- Legal flow

### What NOT to Test
- Marketing copy effectiveness
- Design aesthetics
- CDN performance
- Laravel Cloud internals

### Subtasks

#### 9.1 Landing Page Implementation
- Integrate Salient template from Tailwind
- Hero section with clear value proposition
- Feature showcase with benefits
- Pricing section with $27/month CTA
- Testimonials section (placeholder)
- Footer with legal links

#### 9.2 Legal Pages
- Terms of Service page
- Privacy Policy page
- Cookie Policy page
- GDPR compliance statement
- Terms acceptance checkbox on signup

#### 9.3 SEO & Analytics
- Meta tags for all pages
- Open Graph tags for social sharing
- Sitemap generation
- robots.txt configuration
- Google Analytics or Plausible setup

#### 9.4 Error Pages
- Custom 404 page with navigation
- 403 page with upgrade CTA
- 500 page with support email
- 503 maintenance mode page
- Consistent branding on all error pages

#### 9.5 Final Launch Checklist
- Test complete signup to payment flow
- Verify all email notifications
- Check all external links
- Validate SSL certificate
- Submit to search engines

---

## Success Criteria

### MVP Checklist
- [ ] User registration with 14-day trial
- [ ] 10 domain management (HTTPS only)
- [ ] Three-module security scanning
- [ ] Security scoring (A+ to F grades)
- [ ] PDF report generation
- [ ] Stripe subscription ($27/month)
- [ ] Dashboard with real metrics
- [ ] Activity tracking
- [ ] Support contact (security@achilleus.so)
- [ ] Legal compliance

### Performance Targets
- Dashboard load: <500ms
- Scan completion: <30 seconds
- PDF generation: <10 seconds
- API response: <200ms
- System uptime: >99.9%

### Quality Metrics
- Test coverage: >85%
- Zero critical vulnerabilities
- Mobile responsive
- WCAG 2.1 AA compliant
- SEO optimized landing

---

## Deployment Checklist

### Pre-Deployment
- [ ] All tests passing
- [ ] Security audit complete
- [ ] Performance tested
- [ ] Legal documents ready
- [ ] Monitoring configured
- [ ] Environment variables set
- [ ] Database migrations tested

### Laravel Cloud Deployment
- [ ] Project created at https://cloud.laravel.com/
- [ ] Database provisioned
- [ ] Redis configured
- [ ] Queue workers set
- [ ] Storage configured
- [ ] SSL automatic
- [ ] Reverb configured
- [ ] Stripe webhooks set

### Post-Deployment
- [ ] Health checks passing
- [ ] User flows tested
- [ ] Payment processing verified
- [ ] Monitoring active
- [ ] Backups verified
- [ ] Performance baseline set
- [ ] Security scan complete

---

This execution plan provides a clear roadmap for building Achilleus MVP in 21 days. Each subtask describes WHAT to build, and the workflow (plan → test → code → verify) is applied TO each subtask during implementation.