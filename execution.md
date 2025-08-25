# Achilleus Development Plan

**Timeline**: 30 days across 13 phases  
**Test Target**: 280 tests total  

---

## Phase 1: Project Setup, Database & Trial System
**Days 1-4 | 15 tests**

Create PostgreSQL schema with UUID primary keys. Build models for users, domains, scans, scan_modules, and reports. Add trial fields to User model and implement 14-day trial activation on registration. Generate unsubscribe tokens for new users. Create middleware for trial and subscription validation. Enforce 10-domain limit and implement domain normalization (remove www, protocols, trailing slashes, paths).

**Tests**: Model relationships (5), Trial activation (4), Domain limit (3), URL normalization (2), Middleware (1)

---

## Phase 2: Security Infrastructure
**Day 5 | 25 tests**

Implement NetworkGuard for SSRF protection. Create AbstractScanner base class with retry logic. Setup exception hierarchy. Implement rate limiting per technical.md specifications (configure rate limiters in bootstrap/app.php).

**Tests**: SSRF validation (10), retry logic (5), rate limiting (5), exceptions (5)

---

## Phase 3: SSL/TLS Scanner
**Days 6-7 | 30 tests**

Build SSL/TLS certificate analyzer with 50% scoring weight. Check certificate expiry, protocol versions, cipher suites, and chain validation. Implement platform detection (Cloudflare, GitHub Pages, Vercel) for context-aware scoring adjustments.

**Tests**: Certificate validation (10), protocol detection (6), cipher analysis (6), chain validation (5), platform detection (3)

---

## Phase 4: Security Headers Scanner
**Days 8-9 | 30 tests**

Create security headers scanner with 20% weight. Analyze HSTS, CSP, X-Frame-Options, and other security headers.

**Tests**: Header parsing (10), CSP analysis (7), HSTS validation (7), scoring algorithm (6)

---

## Phase 5: DNS/Email Scanner
**Days 10-11 | 30 tests**

Implement DNS/email security scanner with 30% weight. Validate SPF, DKIM, DMARC, DNSSEC, and CAA records.

**Tests**: SPF/DKIM/DMARC parsing (15), DNSSEC validation (7), domain type detection (4), fallback mechanisms (4)

---

## Phase 6: Scan Orchestration
**Days 12-14 | 30 tests**

Build ScanOrchestrator service for coordinated scanner execution. Configure Redis queue connection for job processing. Install and configure Laravel Reverb for WebSocket real-time updates (php artisan install:broadcasting). Implement weighted scoring with automatic redistribution on scanner failure.

**Tests**: Orchestration logic (10), job processing (6), WebSocket events (6), scoring accuracy (5), error handling (3)

---

## Phase 7: Dashboard UI
**Days 15-16 | 20 tests**

Create dashboard with 3-card metrics layout (Security Score, Active Domains, Critical Issues). Add interactive security score chart with domain toggle. Implement real-time scan progress updates.

**Tests**: Dashboard cards (7), chart visualization (7), real-time updates (6)

---

## Phase 8: Domains, Activity & Reports Pages
**Days 17-19 | 20 tests**

Build domains page with list, detail view, and add/edit functionality. Create Activity page for scan history with report generation. Add Reports page for PDF downloads. All pages include domain filtering.

**Tests**: Domains pages (10), Activity page (5), Reports page (5)

---

## Phase 9: Settings & Profile
**Days 20-21 | 15 tests**

Implement user profile management and notification preferences. Create settings pages for Profile, Notifications, and Security tabs following the starter template structure. Build UI for toggling notification types: certificate_expiry (30/7/1 day warnings), trial_expiry (7/3/1 day warnings), and scan_completion notifications.

**Tests**: Profile updates (5), notification preferences (5), settings UI (5)

---

## Phase 10: Reports & Notifications
**Days 22-23 | 20 tests**

Generate PDF reports with DomPDF and S3 storage. Configure Resend mail driver for email delivery. Create scheduled commands: CheckCertificateExpiry and CheckTrialExpiry. Configure Laravel scheduler in cron (* * * * * php artisan schedule:run). Setup email notifications for certificate and trial expiry warnings with unsubscribe links.

**Tests**: PDF generation (8), email delivery (4), S3 storage (4), scheduling (4)

---

## Phase 11: Payment Integration
**Days 24-25 | 25 tests**

Integrate Stripe for $27/month subscriptions using Laravel Cashier. Build subscription management UI in settings. Handle Stripe webhook events: payment_failed (retry logic), subscription.deleted (downgrade to trial expired), subscription.updated (sync status). Add subscription tab to settings following starter template structure.

**Tests**: Payment flow (10), subscription UI (8), webhook handling (7)

---

## Phase 12: Landing Page Customization
**Days 26-27 | 15 tests**

Customize the starter kit's Welcome page into marketing site with hero section, feature showcase, and pricing page. Update styling to match Achilleus branding. Optimize for SEO and conversions.

**Tests**: Page rendering (7), SEO tags (4), responsive design (4)

---

## Phase 13: Production Deployment
**Days 28-30 | 5 tests**

Deploy to Laravel Cloud with optimized configuration. Configure queue workers with supervisor for scan jobs. Deploy Reverb WebSocket server for real-time updates. Setup cron for Laravel scheduler (* * * * * php artisan schedule:run). Configure environment variables per claude.md specifications. Setup monitoring, backups, and verify all production systems meet performance targets (dashboard < 200ms, scans < 30s).

**Tests**: Deployment validation (3), performance benchmarks (2)

---