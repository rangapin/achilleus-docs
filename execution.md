# Achilleus Execution Plan

## Project Overview
Build a security monitoring SaaS from scratch using Laravel 12 with React starter kit. The application provides automated security scanning for up to 10 domains at $27/month, targeting developers and freelancers who need affordable security monitoring.

## Development Phases

**🔴 TDD APPROACH: Write tests first, then implement features. Each subtask specifies what to test and what to defer.**

### Phase 0: Repository Setup
**Duration**: Initial setup

**Deliverables**:
- Laravel 12 project with React 19 starter kit
- Required packages: laravel-dompdf, stripe-php, pest, playwright
- Shadcn/ui components initialized with dark theme
- Salient marketing template added to resources/marketing/
- Git repository configured and pushed to origin

**Acceptance Criteria**:
- `php artisan serve` works
- `npm run dev` compiles assets
- Shadcn/ui components available
- Dark theme configured

### Phase 1: Foundation Setup (Day 1-2)
**Duration**: 2 days

**Deliverables**:
- Database migrations for all core tables (users, domains, scans, scan_modules, reports)
- Authentication system with 14-day trial setup
- Shadcn/ui styling applied to auth pages
- Laravel Cloud deployment preparation

**Acceptance Criteria**:
- `php artisan migrate` creates all tables
- User registration creates trial_ends_at field
- Auth pages use dark theme consistently
- Environment configured for Laravel Cloud

**Testing Focus**:
- Database schema validation
- Authentication flow testing
- Trial setup verification

### Phase 2: Scanner Implementation (Day 3-5)
**Duration**: 3 days

**Deliverables**:
- Scanner interface and AbstractScanner base class
- NetworkGuard SSRF protection system
- SSL/TLS Scanner (40% weight) - certificate validation, protocol checking
- Security Headers Scanner (30% weight) - HSTS, CSP, security headers analysis
- DNS/Email Scanner (30% weight) - SPF, DKIM, DMARC verification
- Weighted scoring engine with A+ to F grades

**Acceptance Criteria**:
- All scanners extend AbstractScanner
- SSRF protection blocks private IPs
- Scoring matches specification (SSL 40%, Headers 30%, DNS 30%)
- Grade calculation works correctly

**Testing Focus**:
- Unit tests for each scanner with badssl.com variants
- SSRF protection validation
- Scoring engine accuracy

### Phase 3: Queue & Job Processing (Day 6-7)
**Duration**: 2 days

**Deliverables**:
- Laravel Cloud queue configuration
- RunDomainScan job with retry logic and failure handling
- Scan orchestration workflow (pending → running → completed)
- Secure HTTP client with SSRF protection
- Status tracking and result storage

**Acceptance Criteria**:
- Jobs process in background queues
- Scan results stored in JSONB format
- Domain last_scan fields updated
- Failed jobs handled gracefully

**Testing Focus**:
- Queue job processing
- Scan workflow integration
- HTTP client security

### Phase 4: Domain Management (Day 8-9)
**Duration**: 2 days

**Deliverables**:
- Domain model with relationships
- CRUD controllers with authorization policies
- URL validation and normalization
- 10 domain limit enforcement
- Duplicate prevention system

**Acceptance Criteria**:
- Users can add max 10 domains
- HTTPS-only validation works
- Domain ownership enforced
- URL normalization consistent

**Testing Focus**:
- Domain validation rules
- Authorization policies
- Limit enforcement

**UI Components**:
- Domains page with Shadcn/ui Table, Dialog, Form components
- Add domain modal with validation
- Domain list with actions (view, scan, delete)
- Domain counter in navigation

**Shadcn Components**: Dialog, Button, Form, Input, Table, Badge, Avatar, Tabs

**Testing Focus**:
- Playwright MCP for UI interactions
- Form validation
- Table functionality

### Phase 5: Dashboard & UI (Day 10-12)
**Duration**: 3 days

**Deliverables**:
- Dashboard with 4 metric cards (Security Score, Active Domains, Last Scan, Critical Issues)
- Trial banner with countdown and upgrade CTA
- Security trends chart with time period selection
- Navigation sidebar (Dashboard, Domains, Activity, Reports)
- Activity page with scan history and filtering
- Scan results display with module breakdowns

**Acceptance Criteria**:
- Dashboard calculations accurate (average scores, domain counts)
- Solo Plan trial banner shows correct domain limit (10)
- Dark theme consistent across all pages
- Mobile responsive design

**Testing Focus**:
- Dashboard data aggregation
- UI component functionality with Playwright MCP
- Responsive design

### Phase 6: Billing Integration (Day 13-15)
**Duration**: 3 days

**Deliverables**:
- StripeService class with direct API integration (no Cashier)
- Billing controllers (subscribe, cancel, update payment method)
- Stripe webhook handler for subscription events
- Billing UI pages (subscription overview, payment methods, invoices)
- Trial to paid conversion flow

**Acceptance Criteria**:
- Users can subscribe for $27/month Solo Plan
- Webhooks handle subscription status changes
- Payment method updates work
- Subscription cancellation at period end

**Testing Focus**:
- Stripe integration with test cards
- Webhook processing
- Subscription lifecycle

### Phase 7: Reporting System (Day 16-17)
**Duration**: 2 days

**Deliverables**:
- PDF report generation with DomPDF
- Professional report templates with Achilleus branding
- Report storage and management (S3 integration)
- Reports UI with download functionality
- Report history tracking

**Acceptance Criteria**:
- PDF reports generate in under 5 seconds
- Reports contain executive summary and detailed findings
- Download links expire appropriately
- Report history displays correctly

**Testing Focus**:
- PDF generation quality
- S3 storage integration
- Report UI functionality

### Phase 8: Landing Page & Integration Testing (Day 18-19)
**Duration**: 2 days

**Deliverables**:
- Marketing landing page using Salient template
- Pricing section with $27/month plan
- Dashboard screenshots and testimonials
- SEO optimization (meta tags, schema)
- Complete integration testing with real external services
- End-to-end user flows with Playwright MCP
- Performance and load testing

**Acceptance Criteria**:
- Landing page loads under 2 seconds
- Signup flow connects to app registration
- All scanners work with real domains (badssl.com, etc.)
- Stripe checkout flow completes successfully
- Dashboard loads under 500ms
- Scans complete under 15 seconds

**Testing Focus**:
- Real domain integration testing
- Stripe payment flow testing
- Performance benchmarks
- Cross-browser compatibility

### Phase 9: Support & Engagement Systems (Day 17-18)
**Duration**: 2 days

**Deliverables**:
- Email communication system (welcome series, trial reminders, notifications)
- Support infrastructure (contact form, FAQ system, feedback widget)
- Legal compliance pages (Terms of Service, Privacy Policy, cookie consent)
- Guided onboarding flow with achievement system
- Email preference management and unsubscribe handling

**Acceptance Criteria**:
- Welcome emails send on registration, first domain, first scan
- Trial reminders trigger on days 7, 10, 12, 14
- Support tickets route correctly by category
- Legal documents track user acceptance
- Onboarding achievements unlock properly

**Testing Focus**:
- Email delivery and templates
- Support workflow
- Legal compliance
- User engagement metrics

### Phase 10: Laravel Cloud Deployment (Day 19)
**Duration**: 1 day

**Deliverables**:
- Laravel Cloud project configuration
- Production environment setup
- Database and services configuration (PostgreSQL, Redis, S3)
- Queue workers and auto-scaling enabled
- Domain and SSL certificate setup

**Acceptance Criteria**:
- Application deploys successfully
- Database hibernation works
- Queue workers auto-scale
- CDN serves static assets
- All features work in production

**Testing Focus**:
- Production deployment
- Service integration
- Performance monitoring

### Phase 11: Polish & Launch (Day 20)
**Duration**: 1 day

**Deliverables**:
- Performance optimization (database queries, caching, bundling)
- Security hardening (rate limiting, CSRF, headers)
- Final UI polish and consistency
- Documentation (user guide, API docs, deployment guide)
- Launch preparation

**Acceptance Criteria**:
- Dark theme consistent across all pages
- Mobile responsiveness verified
- Accessibility compliance (WCAG AA)
- Performance targets met
- Security audit passed

**Testing Focus**:
- Final UI consistency
- Performance benchmarks
- Security validation

## Success Criteria

### MVP Requirements
✅ User registration with 14-day trial
✅ Add up to 10 domains
✅ Run security scans
✅ View scan results with scores
✅ Generate PDF reports
✅ Subscribe for $27/month
✅ Cancel subscription
✅ Email communication system (welcome, trial, notifications)
✅ Support infrastructure (contact form, FAQ, feedback)
✅ Legal compliance (TOS, Privacy Policy, GDPR)
✅ Guided onboarding flow (tour, achievements)

### Performance Targets
- Scan completion: <15 seconds
- Dashboard load: <500ms
- PDF generation: <5 seconds
- 95% uptime

### Quality Metrics
- Test coverage: >80%
- Zero critical bugs
- Mobile responsive
- WCAG AA compliant

## Resource Requirements

### Development Tools
- Laravel 12
- React 19
- PostgreSQL 15
- Redis
- Stripe account
- Laravel Cloud account

### External Services
- Stripe ($27/month plan)
- Laravel Cloud (serverless)
- GitHub repository
- Domain for production

## Risk Mitigation

### Technical Risks
- **Scanner failures**: Implement robust error handling
- **SSRF attacks**: NetworkGuard validation
- **Payment failures**: Stripe webhook retry logic
- **Queue delays**: Laravel Cloud auto-scaling

### Business Risks
- **Low conversion**: 14-day trial optimization
- **High churn**: Focus on value delivery
- **Support burden**: Comprehensive documentation

## Timeline Summary

**Total Duration**: 22 days

- **Week 1**: Foundation, Scanners, Queue
- **Week 2**: Domain Management, Dashboard, Billing
- **Week 3**: Reporting, Testing, Support & Engagement Systems
- **Final Days**: Deployment, Polish & Launch

## Next Steps

1. Initialize Laravel project with React starter kit
2. Set up development environment
3. Create database migrations
4. Begin scanner implementation
5. Follow phase progression

This execution plan provides a clear path from empty directory to production-ready Achilleus MVP in 21 days.