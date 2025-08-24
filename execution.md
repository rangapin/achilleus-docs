# Achilleus Development Plan - 16-Phase Approach

**Target**: Security monitoring SaaS for developers  
**Timeline**: 28-30 days  
**Test Goal**: 365 tests total  
**Architecture**: Laravel 12 + React + Inertia.js + WorkOS Auth on Laravel Cloud

---

## Phase 1: Project Setup & Authentication
**Duration**: Day 1  
**Tests**: 20 tests

### Goals
- Laravel 12 installation with React starter kit
- WorkOS authentication integration (Google OAuth, Passkeys, Magic Auth)
- User registration/login system with trial activation
- Authentication middleware setup

### Implementation Hints
- **Starter Kit**: Laravel Breeze with React/Inertia preset
- **WorkOS Integration**: Use `workos/workos-php` package
- **Trial Logic**: Set `trial_ends_at = now()->addDays(14)` on user creation
- **Middleware**: Create `EnsureTrialValid` middleware for protected routes

### Key Deliverables
- Working Laravel + Inertia + React foundation
- WorkOS authentication with Google OAuth, Passkeys, and Magic Auth
- User registration with automatic 14-day trial activation
- Secure login/logout functionality
- Basic dashboard skeleton extending starter template

### Key Files to Create
- `app/Http/Middleware/EnsureTrialValid.php`
- `app/Services/WorkOSService.php`
- `resources/js/Pages/Auth/` (Login/Register components)

### Test Focus
- WorkOS authentication flows
- Trial period activation on registration
- Login/logout flows across auth methods
- Middleware protection

---

## Phase 2: Database Schema & Models
**Duration**: Days 2-3  
**Tests**: 30 tests  
**Reference**: `/docs/database.md`

### Goals
- Complete PostgreSQL schema with UUID primary keys
- Eloquent models with relationships
- Business rule enforcement at model level
- Factory and seeder creation

### Implementation Hints
- **UUIDs**: Use `$keyType = 'string'` and `$incrementing = false` in models
- **Domain Limit**: Add `static::creating()` hook in User model to check count
- **Soft Deletes**: Use `SoftDeletes` trait for domains/scans (data retention)
- **Indexes**: Add composite indexes for user_id + created_at queries

### Key Deliverables
- Users, domains, scans, scan_modules, reports tables
- Model relationships (User → Domain → Scan → ScanModule)
- Business rules: 10 domain limit, HTTPS validation, trial logic
- Comprehensive factories for testing

### Key Files to Create
- `database/migrations/` (UUID migrations)
- `app/Models/` (User, Domain, Scan, ScanModule, Report)
- `database/factories/` (comprehensive test data)

### Test Focus
- Model relationships and constraints
- Business rule validation (10 domain limit, trial logic)
- Factory data generation
- Database performance queries (<50ms target)

---

## Phase 3: Security Infrastructure & NetworkGuard
**Duration**: Days 4-5  
**Tests**: 25 tests  
**Reference**: `/docs/technical.md` → "NetworkGuard" & "AbstractScanner"

### Goals
- SSRF protection via NetworkGuard
- AbstractScanner base class with retry logic
- Exception hierarchy for scanner errors
- Rate limiting framework

### Implementation Hints
- **SSRF Protection**: Use `filter_var($ip, FILTER_VALIDATE_IP, FILTER_FLAG_NO_PRIV_RANGE)`
- **Rate Limiting**: Laravel's `Cache::throttle()` with Redis backend
- **Retry Logic**: Progressive backoff (1s, 2s, 4s) in AbstractScanner
- **Exception Types**: Extend base `ScannerException` for categorization

### Key Deliverables
- NetworkGuard SSRF validation (blocks private IPs, localhost, metadata endpoints)
- AbstractScanner with timeout/retry/error handling
- RetryableException, ConfigurationException, SecurityException classes
- Per-host rate limiting system

### Key Files to Create
- `app/Support/NetworkGuard.php`
- `app/Services/Scanners/AbstractScanner.php`
- `app/Exceptions/Scanner/` (exception hierarchy)
- `app/Contracts/Scanners/Scanner.php` (interface)

### Test Focus
- SSRF protection against various attack vectors
- Scanner retry logic and timeout handling
- Exception categorization accuracy
- Rate limiting enforcement

---

## Phase 4: SSL/TLS Scanner
**Duration**: Days 6-7  
**Tests**: 30 tests  
**Reference**: `/docs/technical.md` → "SSL/TLS Scanner"

### Goals
- Complete SSL/TLS security analysis (40% scoring weight)
- Certificate validation and expiry monitoring
- Protocol and cipher analysis
- Certificate chain verification

### Implementation Hints
- **Certificate Analysis**: Use `openssl_x509_parse()` for certificate details
- **Protocol Testing**: `stream_socket_client()` with specific `crypto_method` flags
- **Chain Validation**: Verify issuer trust chain to known root CAs
- **Scoring Logic**: Expiry penalty (-30 expired, -15 <7 days, -5 <30 days)
- **SAN Parsing**: Extract Subject Alternative Names from certificate extensions

### Key Deliverables
- Certificate expiration warnings (30/7/1 days)
- Hostname verification with SAN support
- TLS protocol analysis (prefer 1.3, deprecate 1.0/1.1)
- Cipher strength and Perfect Forward Secrecy validation
- Full certificate chain cryptographic verification

### Key Files to Create
- `app/Services/Scanners/SslTlsScanner.php`
- `app/Data/SslCertificateInfo.php` (certificate data structure)
- `app/Support/CertificateAnalyzer.php` (certificate parsing logic)

### Test Focus
- Certificate validation edge cases
- Protocol version scoring accuracy
- Cipher suite security analysis
- Chain validation with various CA structures

---

## Phase 5: Security Headers Scanner
**Duration**: Days 8-9  
**Tests**: 25 tests  
**Reference**: `/docs/technical.md` → "Security Headers Scanner"

### Goals
- HTTP security header analysis (30% scoring weight)
- HSTS, CSP, X-Frame-Options, X-Content-Type-Options validation
- Advanced directive parsing and scoring
- Information disclosure detection

### Implementation Hints
- **Header Extraction**: Parse `$http_response_header` from `file_get_contents()`
- **CSP Parsing**: Split by semicolons, then spaces for directive analysis
- **HSTS Validation**: Regex `/max-age=(\d+)/` to extract duration
- **Unsafe CSP Detection**: Check for 'unsafe-inline', 'unsafe-eval', '*' in policies

### Key Deliverables
- HSTS analysis with max-age, includeSubDomains, preload validation
- CSP parsing with unsafe directive detection
- Security header presence/absence scoring
- Server information disclosure warnings

### Key Files to Create
- `app/Services/Scanners/SecurityHeadersScanner.php`
- `app/Support/HeaderAnalyzer.php` (header parsing utilities)
- `app/Data/SecurityHeaders.php` (header data structure)

### Test Focus
- Header parsing accuracy across various formats
- CSP directive security analysis
- Scoring algorithm correctness
- Edge cases in header configurations

---

## Phase 6: DNS/Email Scanner
**Duration**: Days 10-11  
**Tests**: 30 tests  
**Reference**: `/docs/technical.md` → "DNS/Email Scanner"

### Goals
- DNS security and email authentication (30% scoring weight)
- SPF, DKIM, DMARC validation
- DNSSEC trust chain verification
- CAA record analysis

### Implementation Hints
- **DNS Queries**: Use `dns_get_record()` with specific types (TXT, CAA, DNSKEY)
- **SPF Parsing**: Look for `v=spf1` prefix, check for `-all` vs `~all` vs `+all`
- **DKIM Selectors**: Try common ones: 'default', 'google', 'k1', 'k2', 'mandrill'
- **DMARC Policy**: Parse `_dmarc.domain.com` TXT records for policy extraction
- **DNSSEC**: Extended timeout (35s) to prevent false negatives

### Key Deliverables
- SPF policy analysis with selector validation
- DKIM key discovery using common selectors
- DMARC policy evaluation with reporting analysis
- Comprehensive DNSSEC validation with fallback methods
- Email/non-email domain handling

### Key Files to Create
- `app/Services/Scanners/DnsEmailScanner.php`
- `app/Support/DnsAnalyzer.php` (DNS query utilities)
- `app/Data/EmailSecurityInfo.php` (email auth data structure)

### Test Focus
- SPF/DKIM/DMARC parsing accuracy
- DNSSEC validation reliability (prevent false negatives)
- Multiple validation method fallbacks
- Domain configuration flexibility

---

## Phase 7: Scan Orchestration & Job Queue
**Duration**: Days 12-14  
**Tests**: 30 tests  
**Reference**: `/docs/technical.md` → "ScanOrchestrator" & Jobs  
**WebSocket Reference**: https://laravel.com/docs/12.x/reverb

### Goals
- Coordinated scanner execution with weighted scoring
- Asynchronous job processing
- Real-time progress updates via WebSockets
- Error aggregation and grade calculation

### Implementation Hints
- **WebSocket Events**: `scan.started`, `scan.progress`, `scan.completed`, `scan.failed`
- **Progress Payload**: `{domain_id, scan_id, progress_percent, current_module, eta_seconds}`
- **Job Retry Logic**: 3 attempts with exponential backoff (1s, 2s, 4s)
- **Weight Redistribution**: Proportional reallocation when scanners fail
- **Reverb Broadcasting**: Use `broadcast(new ScanProgressEvent($scan))`

### Key Deliverables
- ScanOrchestrator service with 40/30/30 weight distribution
- RunDomainScan job with timeout/retry logic
- Laravel Reverb WebSocket integration for progress
- Grade calculation (A+ to F) based on weighted scores
- Weight redistribution when scanners fail

### Key Files to Create
- `app/Services/ScanOrchestrator.php`
- `app/Jobs/RunDomainScan.php`
- `app/Events/ScanProgressEvent.php` (WebSocket broadcast event)
- `resources/js/hooks/useScanProgress.ts` (React WebSocket hook)

### Test Focus
- Weighted scoring accuracy
- Job processing reliability
- WebSocket message delivery
- Error handling in scanner failures

---

## Phase 8: Core UI (Dashboard + Domain List)
**Duration**: Days 15-17  
**Tests**: 30 tests  
**Reference**: `/docs/design.md` + shadcn/ui documentation

### Goals
- Essential interface extending Laravel React starter template (preserve existing layout/navigation)
- Dashboard with security metrics cards using existing components
- Domain list with responsive design
- Real-time updates integration
- Remove 'Repository' from sidebar, keep 'Documentation'

### Implementation Hints
- **Template Extension**: Keep existing `resources/js/Layouts/AuthenticatedLayout.tsx`
- **Card Components**: Use `@/components/ui/card` for dashboard metrics
- **Charts**: Install `recharts` for trend visualization with Shadcn chart components
- **Real-time**: Use `useScanProgress` hook from Phase 7 for live updates
- **Responsive Tables**: Shadcn Table component with mobile card fallback at <768px

### Key Deliverables
- Dashboard cards: Overall Score, Domain Count, Last Scan, Issues (using existing card components)
- Security score trend charts (7d/30d/90d/1y) with shadcn/ui charts
- Responsive domain list (table on desktop, cards on mobile)
- Real-time scan progress indicators
- Preserve existing dark theme functionality with security-specific color coding
- Sidebar navigation: Remove 'Repository', keep 'Documentation'

### Key Files to Create
- `resources/js/Pages/Dashboard.tsx` (main dashboard)
- `resources/js/Pages/Domains/Index.tsx` (domain list)
- `resources/js/Components/SecurityScoreCard.tsx`
- `resources/js/Components/DomainTable.tsx`
- `resources/js/Components/SecurityScoreChart.tsx`

### Test Focus
- Component rendering and data display
- Real-time update functionality
- Responsive design breakpoints
- Chart data visualization accuracy

---

## Phase 9: Domain Detail + Scan Results
**Duration**: Days 18-19  
**Tests**: 25 tests  
**Reference**: `/docs/design.md` → "Domain Detail Layout"

### Goals
- Comprehensive scan result presentation
- Domain configuration management
- Historical data visualization
- Actionable security recommendations

### Implementation Hints
- **Module Cards**: Use Shadcn Card for each scanner module (SSL, Headers, DNS)
- **Status Icons**: Lucide icons - CheckCircle (✅), AlertTriangle (⚠️), XCircle (❌)
- **Expandable Sections**: Shadcn Collapsible for detailed findings
- **Configuration Forms**: Shadcn Form components with validation
- **Historical Charts**: Mini trend charts for score history per module

### Key Deliverables
- Modular scan results display per scanner
- SSL/TLS certificate details with expiration tracking
- Security headers explanation in plain language
- DNS/email authentication status with guidance
- Historical score trends and drill-down analysis
- Domain-specific settings (email mode, DKIM selector)

### Key Files to Create
- `resources/js/Pages/Domains/Show.tsx` (domain detail page)
- `resources/js/Components/ScanModuleCard.tsx`
- `resources/js/Components/DomainSettings.tsx`
- `resources/js/Components/ScanHistoryChart.tsx`

### Test Focus
- Complex data presentation accuracy
- Historical data visualization
- Domain configuration validation
- User interaction flows

---

## Phase 10: Settings + Profile + Activity
**Duration**: Day 20  
**Tests**: 25 tests

### Goals
- User account management
- Notification preferences
- Activity audit trail
- Security settings

### Implementation Hints
- **Settings Layout**: Extend existing template settings structure
- **Tab Navigation**: Use Shadcn Tabs component for settings sections
- **Activity Log**: Table with pagination showing user actions
- **Form Validation**: Laravel Precognition for real-time validation
- **Data Export**: Generate CSV/JSON export of user data

### Key Deliverables
- Profile management with email verification
- Granular notification preferences
- Activity history with audit trail
- Account security settings
- Data export functionality

### Key Files to Create
- `resources/js/Pages/Settings/Profile.tsx`
- `resources/js/Pages/Settings/Notifications.tsx`
- `resources/js/Pages/Settings/Activity.tsx`
- `app/Http/Controllers/Settings/` (settings controllers)

### Test Focus
- Profile update validation
- Notification preference persistence
- Activity logging accuracy
- Security measure effectiveness

---

## Phase 10.5: Email Notifications
**Duration**: Days 20.5-21  
**Tests**: 15 tests  
**Reference**: https://laravel.com/docs/12.x/mail

### Goals
- Certificate expiry warnings (30/7/1 days before expiration)
- Trial expiry warnings (7/3/1 days before trial ends)
- Scan completion notifications
- Unsubscribe handling

### Implementation Hints
- **Mail Driver**: Configure Resend in `config/mail.php`
- **Notification Classes**: Use Laravel Notifications for structured emails
- **Queued Mail**: Queue all emails via `ShouldQueue` interface
- **Templates**: Blade templates in `resources/views/emails/`
- **Unsubscribe**: UUID-based unsubscribe tokens in user table
- **Scheduling**: Laravel Scheduler for daily expiry checks

### Key Deliverables
- Certificate expiry email notifications (30/7/1 day warnings)
- Trial expiry email notifications (7/3/1 day warnings)
- Scan completion summary emails
- Email templates with responsive design
- Unsubscribe functionality
- Notification preferences management

### Key Files to Create
- `app/Notifications/CertificateExpiringNotification.php`
- `app/Notifications/TrialExpiringNotification.php`
- `app/Notifications/ScanCompletedNotification.php`
- `resources/views/emails/` (email templates)
- `app/Console/Commands/SendExpiryWarnings.php`

### Test Focus
- Email delivery accuracy
- Template rendering with correct data
- Unsubscribe token generation and validation
- Notification preferences respect

---

## Phase 11: Report Generation & PDF Creation
**Duration**: Days 22-23  
**Tests**: 25 tests  
**Reference**: `/docs/technical.md` → "Report Generator"

### Goals
- Professional PDF security reports
- S3 storage with signed URLs
- Automated report generation
- Client-ready formatting

### Implementation Hints
- **PDF Library**: Use `barryvdh/laravel-dompdf` for PDF generation
- **Templates**: Blade views in `resources/views/reports/` with CSS styling
- **S3 Integration**: Laravel Filesystem with signed URL generation
- **Queue Jobs**: `GenerateReport` job for async PDF creation
- **Branding**: Company logo, colors, and professional layout

### Key Deliverables
- PDF reports using DomPDF with executive summaries
- S3 storage with 1-hour signed download URLs
- Report generation job queue
- Professional formatting with domain branding
- Email delivery integration

### Key Files to Create
- `app/Services/ReportGenerator.php`
- `app/Jobs/GenerateReport.php`
- `resources/views/reports/security.blade.php`
- `app/Http/Controllers/ReportController.php`

### Test Focus
- PDF generation reliability
- S3 storage and URL signing
- Report content accuracy
- Queue job processing

---

## Phase 12: WorkOS Authentication Enhancement
**Duration**: Day 24  
**Tests**: 15 tests  
**Reference**: Laravel WorkOS documentation

### Goals
- Enhanced WorkOS features (Passkeys, Magic Auth)
- Social account profile synchronization
- Trial period validation for all auth methods

### Implementation Hints
- **Passkeys**: Use WorkOS WebAuthn API for passwordless authentication
- **Magic Auth**: WorkOS Magic Link implementation for email-based login
- **Profile Sync**: Update user data from WorkOS profile on each login
- **Avatar Storage**: S3 storage for profile pictures from social providers
- **Multi-Auth**: Support switching between auth methods in user settings

### Key Deliverables
- Enhanced authentication options (Passkeys, Magic Auth) beyond basic OAuth
- Profile picture and data synchronization from WorkOS
- Trial period consistency across all authentication methods
- User preference management for auth methods

### Key Files to Create
- `app/Services/WorkOS/PasskeyService.php`
- `app/Services/WorkOS/MagicAuthService.php`
- `app/Http/Controllers/Auth/WorkOSController.php`
- `resources/js/Components/AuthMethodSelector.tsx`

### Test Focus
- Multiple authentication method flows
- Profile data synchronization accuracy
- Trial period activation consistency
- Data synchronization accuracy

---

## Phase 13: Payment Integration & Subscription Management
**Duration**: Days 25-26  
**Tests**: 30 tests  
**Reference**: Laravel Cashier documentation

### Goals
- Stripe subscription management
- $27/month pricing with 14-day trial
- Payment method management
- Subscription lifecycle handling
- Subscription settings UI in existing settings layout

### Implementation Hints
- **Cashier Setup**: Install `laravel/cashier` and run migrations
- **Stripe Products**: Create $27/month price in Stripe dashboard
- **Billable Model**: Add `Billable` trait to User model
- **Webhooks**: Handle subscription updates, payment failures
- **Settings Tab**: Add "Subscription" tab to existing settings layout
- **Payment Forms**: Use Stripe Elements for secure card collection
- **Invoice Portal**: Stripe Customer Portal for self-service

### Key Deliverables
- Laravel Cashier integration with Stripe
- $27/month subscription plans
- Subscription settings tab (below Appearance in settings sidebar)
- Payment method addition/update/removal
- Subscription cancellation and reactivation
- Invoice history and downloads
- Trial-to-paid conversion flow
- Settings UI matching existing Profile/Password/Appearance styling

### Key Files to Create
- `app/Http/Controllers/SubscriptionController.php`
- `resources/js/Pages/Settings/Subscription.tsx`
- `resources/js/Components/PaymentMethodForm.tsx`
- `app/Http/Controllers/WebhookController.php`

### Test Focus
- Payment processing accuracy
- Subscription state management
- Trial conversion logic
- Settings UI component integration
- Error handling for failed payments

---

## Phase 14: Landing Page & Marketing Site
**Duration**: Days 27-28  
**Tests**: 20 tests

### Goals
- Public marketing website
- Feature presentation
- Pricing information
- SEO optimization

### Implementation Hints
- **Public Routes**: Create guest-accessible routes for marketing pages
- **SEO Components**: React Helmet for dynamic meta tags
- **Hero Section**: Compelling headline with security monitoring value proposition
- **Feature Cards**: Benefits of SSL monitoring, security headers, DNS checks
- **Social Proof**: Testimonials, security badges, trust indicators
- **CTA Optimization**: Multiple sign-up points, trial emphasis

### Key Deliverables
- Landing page with value proposition
- Feature showcase with screenshots
- Pricing page with plan comparison
- About and contact pages
- SEO meta tags and structured data
- Call-to-action optimization

### Key Files to Create
- `resources/js/Pages/Welcome.tsx` (landing page)
- `resources/js/Pages/Pricing.tsx`
- `resources/js/Pages/Features.tsx`
- `resources/js/Components/HeroSection.tsx`
- `resources/js/Components/FeatureCard.tsx`

### Test Focus
- Page load performance
- SEO tag accuracy
- Conversion funnel testing
- Mobile responsiveness

---

## Phase 15: Production Polish & Deployment
**Duration**: Days 29-30  
**Tests**: 25 tests

### Goals
- Production readiness
- Laravel Cloud deployment
- Performance optimization
- Monitoring and alerting

### Implementation Hints
- **Laravel Cloud**: Push to production branch for auto-deployment
- **Environment**: Production .env with Stripe live keys, proper URLs
- **Infrastructure**: Laravel Cloud automatically provides:
  - Managed Redis for caching and queues
  - Auto-scaling queue workers (no configuration needed)
  - Built-in error tracking and monitoring
  - Database backups and disaster recovery
  - CDN for static assets
  - SSL certificates (automatic)
- **DNS Setup**: Point achilleus.so to Laravel Cloud load balancer
- **Reference**: https://cloud.laravel.com/ for deployment details

### Key Deliverables
- Laravel Cloud production deployment
- Performance optimization (caching, database indexes)
- Error monitoring and alerting setup
- Backup and disaster recovery procedures
- Security hardening and SSL configuration
- Domain setup and DNS configuration

### Key Files to Create
- `.env.production` (production environment)
- `deploy.yml` (deployment configuration)
- `app/Console/Commands/CacheWarmup.php`
- Production monitoring dashboard

### Test Focus
- Production environment validation
- Performance benchmarks
- Error monitoring accuracy
- Security configuration verification

---