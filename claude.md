# CLAUDE.md

This file provides guidance to Claude and AI assistants when working with the Achilleus repository.

## Project Overview

**Achilleus** - Security monitoring SaaS for developers managing multiple websites  
**Target**: Freelancers and small businesses needing affordable security monitoring  
**Price**: $27/month for 10 domains with unlimited scans  
**Timeline**: 28-30 day development plan across 16 phases with 365 comprehensive tests

## Domain Configuration
- **Primary Domain**: https://achilleus.so
- **Support Email**: support@achilleus.so
- **Laravel Cloud URL**: https://achilleus.so

## Documentation Structure
- `/docs/execution.md` - 15-phase roadmap (25 days) with test targets
- `/docs/database.md` - Complete schema with PostgreSQL 15
- `/docs/technical.md` - Scanner implementations and architecture
- `/docs/testing.md` - Testing strategy with 365 test target
- `/docs/product.md` - Product specifications and user journeys
- `/docs/design.md` - UI/UX specifications with mockups

## Development Commands

### Achilleus Commands (Custom CLI)
```bash
# Generate implementation plan for a phase
achilleus plan "Phase Name"

# Generate tests for the phase
achilleus test "Phase Name"

# Generate code implementation for the phase
achilleus code "Phase Name"
```

### Standard Laravel Commands
```bash
# Create models, controllers, etc.
php artisan make:model Domain -mfsc --all

# Run tests by phase
php artisan test --filter="Phase1Test"
php artisan test --parallel
php artisan test --coverage --min=80
```

## Critical Security Requirements

### SSRF Protection (NetworkGuard)
- **ALWAYS** validate URLs before external requests
- Block private IPs (192.168.x, 10.x, 172.16.x, 169.254.169.254)
- Enforce HTTPS-only for domains
- Throw SecurityException on violations

### Scanner Architecture
- All scanners extend AbstractScanner
- Implement retry logic (3 attempts)
- Set timeout to 30 seconds
- Handle three exception types:
  - RetryableException (temporary issues)
  - ConfigurationException (user must fix)
  - SecurityException (policy violations)

### Scanner Weights & Scoring
- **SSL/TLS Scanner**: 40% weight
- **Security Headers Scanner**: 30% weight
- **DNS/Email Scanner**: 30% weight
- **Grade Calculation**: A+ (95-100), A (90-94), B+ (85-89), B (80-84), C (70-79), D (60-69), F (0-59)
- **Score Redistribution**: When scanner fails, redistribute weights proportionally

## Business Rules

### Domain Management
- **Limit**: Maximum 10 domains per user (strict enforcement)
- **Input UX**: Clean domain names only (e.g., "example.com")
- **Processing**: Automatic HTTPS prepending for scanning
- **Normalization**: Remove protocol, www, trailing slash, paths
- **Validation**: Real-time feedback with helpful error messages
- **Duplicate Prevention**: No duplicate domains per user

### Trial & Subscription
- **Trial Period**: 14 days from registration (OAuth users included)
- **Trial Enforcement**: Block features after expiry (return 402)
- **Subscription**: $27/month via Laravel Cashier
- **Grace Period**: 3 days for failed payments
- **Access Levels**: Different for trial, active, expired states

### User Limits
- **Domains**: 10 maximum
- **Scans**: 10 per minute rate limit
- **Reports**: 10 per day generation limit
- **API Calls**: Rate limited per endpoint

## UI/UX Standards

### Design Philosophy
- **Extend, Don't Rebuild**: Work with Laravel React starter template, don't rearrange
- **Use Built-In Components**: Leverage existing Shadcn UI components from starter kit
- **Minimal Template Modifications**: Final project should look like the starter template essentially
- **Reference Documentation**: Always review component docs before implementation

### Starter Template Structure
- **Layout**: Use existing sidebar layout (don't rebuild navigation)
- **Theme**: Built-in dark theme from starter kit (preserve existing dark mode functionality)
- **Components**: Extend existing Shadcn components in `resources/js/`
- **Authentication**: Keep existing auth flow, minimal modifications only
- **Sidebar Navigation**: Remove "Repository" section, keep "Documentation", add "Subscription" tab in settings

### Component References
- **Shadcn Components**: https://ui.shadcn.com/docs/components
- **Charts**: https://ui.shadcn.com/charts/bar#charts (for dashboard metrics)
- **Blocks**: https://ui.shadcn.com/blocks (for layout patterns)
- **Installation**: Use `npx shadcn@latest add [component]` for additional components

### Key Components (Using Starter Kit)
- **Dashboard Cards**: Use existing card components with security metrics
- **Forms**: Use built-in form components for domain management
- **Tables**: Use existing table components for domain lists
- **Modals**: Use built-in dialog components
- **Settings Pages**: Use existing settings layout style for subscription tab (consistent with Profile/Password/Appearance)

### Performance Targets
- **Dashboard Load**: < 200ms (aggressive optimization required)
- **Scan Completion**: < 30 seconds
- **API Response**: < 200ms (< 100ms for cached data)
- **Test Suite**: < 30 seconds with parallel execution
- **Database Queries**: < 50ms (individual queries)
- **Static Assets**: < 100ms (with CDN)

## Testing Strategy

### Test Distribution (365 Total)
- **Phase 1-3**: Foundation (75 tests)
- **Phase 4-6**: Core Scanners (85 tests)
- **Phase 7-10**: Application Logic (110 tests)
- **Phase 11-15**: Advanced Features (120 tests)

### Mocking Requirements
- **Always Mock**: Stripe, OAuth, External HTTP, Email, Queue Jobs
- **Never Test**: Laravel framework, external APIs, database queries
- **Focus On**: Business logic, security rules, user workflows

## Environment Variables (Production)
```env
APP_URL=https://achilleus.so
DB_CONNECTION=pgsql
QUEUE_CONNECTION=redis
CACHE_DRIVER=redis

STRIPE_KEY=pk_live_...
STRIPE_SECRET=sk_live_...
STRIPE_WEBHOOK_SECRET=whsec_...
CASHIER_CURRENCY=usd

WORKOS_CLIENT_ID=...
WORKOS_API_KEY=...
WORKOS_REDIRECT_URL=https://achilleus.so/auth/workos/callback

MAIL_MAILER=resend
RESEND_API_KEY=re_...
```

## Laravel Cloud Deployment

### Infrastructure
- **Hosting**: Laravel Cloud only (no VPS/Docker)
- **Database**: Managed PostgreSQL 15
- **Queue**: Managed Redis workers
- **Storage**: S3 for PDF reports
- **SSL**: Automatic certificates

### Deployment Commands
```bash
git push production main
php artisan migrate --force
php artisan config:cache
php artisan route:cache
```

## Key Implementation Files

### Phase 1-3: Foundation
- `app/Support/NetworkGuard.php` - SSRF protection
- `app/Services/Scanners/AbstractScanner.php` - Base scanner class
- Database migrations and models

### Phase 4-6: Core Scanners
- `app/Services/Scanners/SslTlsScanner.php` - SSL/TLS implementation
- `app/Services/Scanners/SecurityHeadersScanner.php` - Headers implementation
- `app/Services/Scanners/DnsEmailScanner.php` - DNS/Email implementation

### Phase 7-10: Application Logic
- `app/Services/ScanOrchestrator.php` - Manages scan execution
- `app/Jobs/RunDomainScan.php` - Async scan job
- `resources/js/Pages/Dashboard.tsx` - Dashboard with security metrics
- `resources/js/Pages/Domains/` - Domain management components

### Phase 11-15: Advanced Features
- `app/Services/ReportGenerator.php` - PDF generation
- `app/Jobs/GenerateReport.php` - Async report job
- Payment integration via Laravel Cashier
- WorkOS authentication enhancement

## 16-Phase Development Roadmap

### Foundation (Days 1-6)
1. **Project Setup & Authentication** - Laravel + React starter kit + WorkOS
2. **Database Schema & Models** - Complete data layer
3. **Security Infrastructure** - NetworkGuard + AbstractScanner

### Core Scanners (Days 7-12)
4. **SSL/TLS Scanner** - Certificate analysis (40% weight)
5. **Security Headers Scanner** - HTTP headers analysis (30% weight)
6. **DNS/Email Scanner** - SPF/DKIM/DMARC/DNSSEC (30% weight)

### Application Logic (Days 13-20)
7. **Scan Orchestration** - Job queue + WebSocket updates
8. **Core UI** - Dashboard + domain list extending starter template
9. **Domain Detail** - Scan results + configuration
10. **Settings + Profile** - User management + activity
10.5. **Email Notifications** - Certificate/trial expiry warnings

### Advanced Features (Days 21-30)
11. **Report Generation** - PDF creation + S3 storage
12. **WorkOS Authentication** - Enhanced auth features (Passkeys, Magic Auth)
13. **Payment Integration** - Stripe + $27/month subscriptions + subscription settings tab
14. **Landing Page** - Marketing site + SEO
15. **Production Polish** - Laravel Cloud deployment + monitoring

## Common Pitfalls to Avoid

### File Management & Template Usage
- ❌ Never create duplicate files
- ❌ Always check if file exists before creating new ones
- ❌ Use TSX files for React components (never JSX)
- ❌ Don't rebuild what the starter template already provides
- ❌ Don't rearrange the existing template structure
- ✅ Extend existing components and layouts
- ✅ Follow Laravel Starter Kit conventions
- ✅ Review Shadcn documentation before implementation

### TypeScript & React Requirements
- ✅ All React components MUST use .tsx extension
- ✅ Define interfaces for all component props
- ✅ Use proper TypeScript event handler types:
  - `React.FormEvent<HTMLFormElement>` for form submissions
  - `React.ChangeEvent<HTMLInputElement>` for input changes
  - `React.MouseEvent<HTMLButtonElement>` for button clicks
  - `React.KeyboardEvent<HTMLElement>` for keyboard events
- ✅ Import types from React: `import { useState } from 'react'`
- ✅ Use generic type parameters: `useState<FormData>(initialState)`
- ✅ Export components as named functions with proper interfaces

### Security
- ❌ Never skip NetworkGuard validation
- ❌ Don't allow HTTP domains (auto-prepend HTTPS for scanning)
- ❌ Never log sensitive data (API keys, passwords)
- ❌ Don't trust user input without validation
- ✅ Use clean domain input with real-time validation
- ✅ Normalize domains before storage (remove www, protocol, paths)
- ✅ Provide helpful error messages for common user mistakes

### Testing
- ❌ Don't test external APIs (mock them)
- ❌ Avoid testing Laravel framework features
- ❌ Don't exceed 365 tests
- ❌ Never use real Stripe/OAuth in tests

### Performance
- ❌ Don't make synchronous external calls
- ❌ Avoid N+1 queries (use eager loading)
- ❌ Don't skip caching for expensive operations
- ❌ Never process scans synchronously

## Success Criteria

### MVP Completeness
- ✅ All 15 phases completed
- ✅ 365 tests passing
- ✅ All features functional
- ✅ Deployed to Laravel Cloud
- ✅ Processing real payments

### Quality Metrics
- ✅ 80% code coverage on business logic
- ✅ Dashboard loads < 500ms
- ✅ Zero critical security issues
- ✅ All user journeys working