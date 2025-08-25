# CLAUDE.md

This file provides guidance to Claude and AI assistants when working with the Achilleus repository.

## Project Overview

**Achilleus** - Security monitoring SaaS for developers managing multiple websites  
**Target**: Freelancers and small businesses needing affordable security monitoring  
**Price**: $27/month for 10 domains with unlimited scans  
**Timeline**: 30 day development plan across 13 phases with 280 comprehensive tests

## Domain Configuration
- **Primary Domain**: https://achilleus.so
- **Support Email**: support@achilleus.so
- **Laravel Cloud URL**: https://achilleus.so

## Documentation Structure
- `/docs/execution.md` - 13-phase roadmap (30 days) with test targets
- `/docs/database.md` - Complete schema with PostgreSQL 15
- `/docs/technical.md` - Scanner implementations and architecture
- `/docs/testing.md` - Testing strategy with 280 test target
- `/docs/product.md` - Product specifications and user journeys
- `/docs/design.md` - UI/UX specifications with mockups

## Development Commands

### Achilleus Commands (Custom CLI)

#### `achilleus plan "Phase Name"`
Generates a comprehensive TODO list for the specified phase from execution.md. 
- **Output**: Displays a detailed task list with checkboxes
- **Behavior**: Only shows the plan, does NOT proceed with implementation
- **Example Output**:
  ```
  Phase 3: SSL/TLS Scanner Plan
  □ Create app/Services/Scanners/SslTlsScanner.php
  □ Implement certificate parsing with openssl_x509_parse()
  □ Add protocol version detection (TLS 1.3/1.2)
  □ Calculate expiry penalties (-25 for 1 day, -10 for 7 days)
  □ Implement chain validation
  □ Add platform detection for context-aware scoring
  ```

#### `achilleus test "Phase Name"`
Generates tests for the CURRENT phase only based on execution.md specifications.
- **Output**: Creates test files with the EXACT number of tests specified
- **Behavior**: 
  - Never creates tests for future phases
  - Follows the test count precisely (e.g., Phase 3 = exactly 30 tests)
  - Mocks all external services per testing.md guidelines
- **Example**: Phase 3 creates exactly 30 tests:
  - Certificate validation (10 tests)
  - Protocol detection (6 tests)
  - Cipher analysis (6 tests)
  - Chain validation (5 tests)
  - Platform detection (3 tests)

#### `achilleus code "Phase Name"`
Implements MINIMAL working code for the phase using Laravel best practices.
- **Output**: Creates necessary files with clean, working implementation
- **Behavior**:
  - Follows Laravel conventions and patterns
  - No over-engineering or future features
  - Uses existing Laravel features (no reinventing)
- **Example**: For Phase 3, creates:
  - SslTlsScanner.php with scan() method
  - Required support classes
  - No UI, no extras, just core functionality

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

### Dual Scoring System

#### Overall Security Score
- **SSL/TLS Scanner**: 50% weight (critical security)
- **Security Headers Scanner**: 20% weight (varies by implementation)
- **DNS/Email Scanner**: 30% weight (important but often optional)
- **Overall Grade**: A+ (95-100), A (90-94), B+ (85-89), B (80-84), C (70-79), D (60-69), F (0-59)
- **Score Redistribution**: When scanner fails, redistribute weights proportionally
- **Confidence Levels**: High/Medium/Low confidence affects final scoring
- **Platform Detection**: Adjusts expectations for Cloudflare, GitHub Pages, etc.

#### SSL Grade (Separate from Overall Score)
- **Industry Standard**: A+ to F rating based on SSL/TLS configuration
- **A+ Requirements**: TLS 1.3, HSTS preload, strong ciphers, PFS, valid cert (30+ days)
- **Independent**: SSL grade is calculated separately from the numerical score
- **Display**: Shown prominently alongside overall security score
- **Color Coding**: Green (A+/A), Yellow (B+/B), Orange (C), Red (D/F)

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
- **API Calls**: Rate limited per endpoint (see API Rate Limiting section)

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
- **Dashboard Cards**: 3 cards showing Security Score, Active Domains, Critical Issues
- **Domain Detail Cards**: 
  - Top row: Security Score, SSL Grade, Last Scan Date
  - Bottom row: Individual scanner results with scores
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

### Test Distribution (280 Total)
- **Phase 1-2**: Foundation (40 tests)
- **Phase 3-5**: Core Scanners (90 tests)
- **Phase 6-9**: Application Logic (85 tests)
- **Phase 10-13**: Advanced Features (65 tests)

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

# WorkOS removed - using Laravel's built-in authentication only

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
- **Regions**: US East (primary), auto-scaling globally
- **PHP Version**: 8.3
- **Node Version**: 20 LTS

### Laravel Cloud Configuration File
```yaml
# .laravel.cloud.yaml
id: achilleus-production
name: 'Achilleus Security'
environments:
  - name: production
    domain: achilleus.so
    database: achilleus-db
    warm: 2
    scale:
      cpus: 0.5
      memory: 1024
    build:
      - 'composer install --no-dev'
      - 'npm ci && npm run build'
      - 'php artisan config:cache'
      - 'php artisan route:cache'
      - 'php artisan view:cache'
      - 'php artisan event:cache'
      - 'php artisan icons:cache'
    deploy:
      - 'php artisan migrate --force'
      - 'php artisan queue:restart'
      - 'php artisan reverb:restart'
```

### Environment Variables (Laravel Cloud)
```env
# Laravel Cloud Managed
LARAVEL_CLOUD=true
APP_ENV=production
APP_DEBUG=false
APP_URL=https://achilleus.so

# Database (automatically injected)
DB_CONNECTION=pgsql
DB_HOST=${LARAVEL_CLOUD_DB_HOST}
DB_PORT=${LARAVEL_CLOUD_DB_PORT}
DB_DATABASE=${LARAVEL_CLOUD_DB_NAME}
DB_USERNAME=${LARAVEL_CLOUD_DB_USER}
DB_PASSWORD=${LARAVEL_CLOUD_DB_PASSWORD}

# Redis (automatically injected)
REDIS_HOST=${LARAVEL_CLOUD_REDIS_HOST}
REDIS_PASSWORD=${LARAVEL_CLOUD_REDIS_PASSWORD}
REDIS_PORT=${LARAVEL_CLOUD_REDIS_PORT}

# Queue Configuration
QUEUE_CONNECTION=redis
LARAVEL_CLOUD_QUEUE_CONNECTION=redis
LARAVEL_CLOUD_QUEUE_WORKER_MAX_JOBS=1000
LARAVEL_CLOUD_QUEUE_WORKER_MAX_TIME=3600
LARAVEL_CLOUD_QUEUE_WORKER_MEMORY=512

# Cache
CACHE_DRIVER=redis
SESSION_DRIVER=redis

# Laravel Reverb (WebSockets)
BROADCAST_DRIVER=reverb
REVERB_APP_ID=${LARAVEL_CLOUD_REVERB_APP_ID}
REVERB_APP_KEY=${LARAVEL_CLOUD_REVERB_APP_KEY}
REVERB_APP_SECRET=${LARAVEL_CLOUD_REVERB_APP_SECRET}
REVERB_HOST=achilleus.so
REVERB_PORT=443
REVERB_SCHEME=https

# Storage
FILESYSTEM_DISK=s3
AWS_ACCESS_KEY_ID=${LARAVEL_CLOUD_AWS_ACCESS_KEY_ID}
AWS_SECRET_ACCESS_KEY=${LARAVEL_CLOUD_AWS_SECRET_ACCESS_KEY}
AWS_DEFAULT_REGION=us-east-1
AWS_BUCKET=achilleus-reports
```

### Queue Worker Configuration
```php
// config/queue.php
'connections' => [
    'redis' => [
        'driver' => 'redis',
        'connection' => 'default',
        'queue' => env('REDIS_QUEUE', 'default'),
        'retry_after' => 90,
        'block_for' => null,
        'after_commit' => false,
    ],
],

// Laravel Cloud queue priorities
'queue_priorities' => [
    'high' => 100,    // Critical scans
    'default' => 50,  // Normal operations
    'low' => 10,      // Reports, emails
],
```

### Health Check Configuration
```php
// routes/web.php
Route::get('/health', function () {
    return response()->json([
        'status' => 'healthy',
        'timestamp' => now()->toIso8601String(),
        'checks' => [
            'database' => DB::connection()->getPdo() ? 'connected' : 'error',
            'redis' => Redis::ping() ? 'connected' : 'error',
            'storage' => Storage::disk('s3')->exists('health-check') ? 'connected' : 'error',
        ],
    ]);
})->name('health');
```

### Auto-scaling Configuration
```php
// config/cloud.php
return [
    'scaling' => [
        'min_instances' => env('LARAVEL_CLOUD_MIN_INSTANCES', 2),
        'max_instances' => env('LARAVEL_CLOUD_MAX_INSTANCES', 10),
        'target_cpu' => env('LARAVEL_CLOUD_TARGET_CPU', 70),
        'target_memory' => env('LARAVEL_CLOUD_TARGET_MEMORY', 80),
        'scale_down_cooldown' => 300, // 5 minutes
        'scale_up_cooldown' => 60,    // 1 minute
    ],
];
```

### Deployment Commands
```bash
# Initial setup
laravel-cloud environment:create production
laravel-cloud database:create achilleus-db --type=postgresql --version=15
laravel-cloud redis:create achilleus-cache

# Deploy
git push production main

# Manual commands if needed
laravel-cloud run production "php artisan migrate --force"
laravel-cloud run production "php artisan queue:restart"
laravel-cloud run production "php artisan cache:clear"

# Monitoring
laravel-cloud logs production --tail
laravel-cloud metrics production
laravel-cloud database:shell achilleus-db
```

### Laravel Cloud CLI Commands
```bash
# Environment management
laravel-cloud env:push production .env.production
laravel-cloud env:get production
laravel-cloud env:set production KEY=value

# Scaling
laravel-cloud scale production --instances=5
laravel-cloud scale production --cpus=1 --memory=2048

# Maintenance
laravel-cloud down production
laravel-cloud up production

# Backups
laravel-cloud database:backup achilleus-db
laravel-cloud database:restore achilleus-db --backup=backup-id
```

## Key Implementation Files

### Phase 1-2: Foundation
- `app/Support/NetworkGuard.php` - SSRF protection
- `app/Services/Scanners/AbstractScanner.php` - Base scanner class
- Database migrations and models

### Phase 3-5: Core Scanners
- `app/Services/Scanners/SslTlsScanner.php` - SSL/TLS implementation
- `app/Services/Scanners/SecurityHeadersScanner.php` - Headers implementation
- `app/Services/Scanners/DnsEmailScanner.php` - DNS/Email implementation

### Phase 6-9: Application Logic
- `app/Services/ScanOrchestrator.php` - Manages scan execution
- `app/Jobs/RunDomainScan.php` - Async scan job
- `resources/js/Pages/Dashboard.tsx` - Dashboard with security metrics
- `resources/js/Pages/Domains/` - Domain management components

### Phase 10-13: Advanced Features
- `app/Services/ReportGenerator.php` - PDF generation
- `app/Jobs/GenerateReport.php` - Async report job
- Payment integration via Laravel Cashier
- Email notifications for expiry warnings

## 13-Phase Development Roadmap

### Foundation (Days 1-5)
1. **Project Setup, Database & Trial System** - Laravel + React starter kit with auth, PostgreSQL schema, models, and trial system
2. **Security Infrastructure** - NetworkGuard + AbstractScanner

### Core Scanners (Days 6-11)
3. **SSL/TLS Scanner** - Certificate analysis (50% weight)
4. **Security Headers Scanner** - HTTP headers analysis (20% weight)
5. **DNS/Email Scanner** - SPF/DKIM/DMARC/DNSSEC (30% weight)

### Application Logic (Days 12-20)
6. **Scan Orchestration** - Job queue + WebSocket updates
7. **Core UI** - Dashboard + domain list extending starter template
8. **Domain Detail** - Scan results + configuration
9. **Settings + Profile** - User management + activity

### Advanced Features (Days 21-30)
10. **Report Generation** - PDF creation + S3 storage + Email Notifications
11. **Payment Integration** - Stripe + $27/month subscriptions + subscription settings tab
12. **Landing Page** - Marketing site + SEO
13. **Production Polish** - Laravel Cloud deployment + monitoring

## Common Pitfalls to Avoid

### File Management & Template Usage
- ❌ Never create duplicate files
- ❌ Always check if file exists before creating new ones
- ❌ Use TSX files for React components (never JSX)
- ❌ Don't rebuild what the starter template already provides
- ❌ Don't rearrange the existing template structure
- ✅ Extend existing components and layouts
- ✅ Follow Laravel React Starter Kit conventions
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
- ❌ Don't exceed 280 tests
- ❌ Never use real Stripe in tests

### Performance
- ❌ Don't make synchronous external calls
- ❌ Avoid N+1 queries (use eager loading)
- ❌ Don't skip caching for expensive operations
- ❌ Never process scans synchronously

## Success Criteria

### MVP Completeness
- ✅ All 13 phases completed
- ✅ 280 tests passing
- ✅ All features functional
- ✅ Deployed to Laravel Cloud
- ✅ Processing real payments

### Quality Metrics
- ✅ 80% code coverage on business logic
- ✅ Dashboard loads < 500ms
- ✅ Zero critical security issues
- ✅ All user journeys working