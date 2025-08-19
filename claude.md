# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**Achilleus** - Security monitoring SaaS for developers managing multiple websites  
**Target**: Freelancers and small businesses needing affordable security monitoring  
**Price**: $27/month for 10 domains with unlimited scans  
**Stack**: Laravel 12 (with (https://laravel.com/docs/12.x/starter-kits) starter kit) + React 19 + Shadcn/ui + Inertia.js + Laravel Reverb + Laravel Cloud

## Domain Configuration
- **Primary Domain**: https://achilleus.so
- **Support Email**: support@achilleus.so
- **Laravel Cloud URL**: https://achilleus.so
## Email Strategy (MVP)
- Transactional emails via Laravel's mail system
- Critical emails only: Welcome, Payment confirmation, Trial expiry
- No marketing emails or notifications in MVP
- Support handled manually at support@achilleus.so
## Key Resources
- **Laravel 12**: https://laravel.com/docs/12.x
- **Laravel Starter Kits**: https://laravel.com/docs/12.x/starter-kits (we use React kit)
- **React 19**: https://react.dev/
- **Laravel Cloud**: https://cloud.laravel.com/
- **Shadcn/ui**: https://ui.shadcn.com/
- **Laravel Reverb**: https://reverb.laravel.com/
- **Laravel Cashier (Stripe)**: https://laravel.com/docs/12.x/billing
- **Laravel Precognition**: https://laravel.com/docs/12.x/precognition
- **Laravel Socialite**: https://laravel.com/docs/12.x/socialite
- **Laravel Boost**: https://boost.laravel.com/ (AI development assistant)
- **Landing Page**: Salient template from https://tailwindcss.com/plus/templates/salient  

## Documentation Structure

- `/docs/technical.md` - System architecture, scanner implementation, database schema
- `/docs/product.md` - Product specifications, UI mockups, business rules  
- `/docs/execution.md` - Development phases, testing strategy, deployment plan
- `/docs/database.md` - Complete database schema and relationships
- `/docs/design.md` - UI/UX specifications and component library
- `/docs/testing.md` - Testing strategy and patterns

## Laravel Boost Integration

**Laravel Boost** is an MCP server that provides AI assistants with deep project context and Laravel-specific knowledge. When active, AI can inspect your database, run Tinker commands, search Laravel 12 docs, read logs, and understand your actual project structure.

**Installation**: See `/docs/execution.md` Phase 0 for setup instructions.

**Key Capabilities**: Database queries, code execution, route inspection, configuration access, log analysis, test generation following project conventions.

**Security**: Boost only runs in development, respects .env boundaries, and automatically includes SSRF protection in generated code.

## Development Workflow Shortcuts

### Core Commands

- **achilleus-plan**: Break down → Security review → Test cases → SSRF validation
- **achilleus-test**: Tests first (Unit → Feature → E2E → Security validation)
- **achilleus-code**: Laravel conventions → Shadcn/ui → Security-first → Error handling
- **achilleus-verify**: Run all tests in parallel, type checking, and E2E tests

## Security-First Development (Critical)

### S1: Input Validation & SSRF Protection
- **ALWAYS** use Form Requests for validation
- **ALWAYS** validate URLs with NetworkGuard before external requests
- Enforce HTTPS-only domains
- Use PublicUrl validation rule

### S2: Scanner Implementation Pattern
- Extend AbstractScanner for all scanners
- Base class handles SSRF, retries, rate limiting automatically
- Set appropriate timeout and retry attempts
- Jobs dispatched to Laravel Cloud managed workers

### S3: Rate Limiting & Authentication
- Two-tier rate limiting: User level (10 scans/minute) and Scanner level
- Use throttle middleware on scan endpoints
- Always authorize domain access with policies
- Enforce trial expiry with middleware

## Best Practices Checklist

### BP1: Laravel Patterns
- [ ] Use Form Requests for all validation
- [ ] Implement proper authorization policies  
- [ ] Use Eloquent relationships correctly
- [ ] Apply middleware consistently
- [ ] Handle errors gracefully with try/catch

### BP2: React + Shadcn/ui
- [ ] Use TypeScript interfaces consistently
- [ ] Implement React Hook Form + Zod validation
- [ ] Apply Shadcn/ui components (Button, Card, Form, Table, Dialog)
- [ ] Handle loading states and errors
- [ ] Ensure dark theme consistency

### BP3: Database & Models
- [ ] Create migrations before models
- [ ] Define relationships in models
- [ ] Use appropriate data types (JSONB for scan results)
- [ ] Add proper indexes for performance
- [ ] Implement soft deletes where needed

### BP4: Testing Strategy
- [ ] Write tests BEFORE implementation (TDD)
- [ ] Unit tests for scanners and services
- [ ] Feature tests for API endpoints
- [ ] E2E tests with Playwright for critical flows
- [ ] Test security validation and SSRF protection

## Code Quality Standards

### C1: Security Requirements
- HTTPS only domains (no HTTP allowed)
- NetworkGuard validation on all external requests
- Input validation on all user inputs
- Rate limiting on scan operations
- SSRF protection against private IPs and localhost

### C2: Performance Standards
- Dashboard load time < 500ms
- Scan completion < 30 seconds
- API responses < 200ms
- Database queries optimized with proper indexes
- Caching implemented for frequently accessed data

### C3: UI/UX Standards
- Consistent dark theme (#0a0a0b background)
- Mobile-responsive design
- Loading states for all async operations
- Empty states with actionable guidance
- Error messages with clear next steps

### C4: Code Organization
- Follow Laravel directory structure
- Use service classes for business logic
- Keep controllers thin (max 20 lines per method)
- Extract reusable components
- Document complex business logic

## Scanner Development with Boost

Laravel Boost enables AI to:
- Inspect existing scanner implementations
- Query scan results to understand patterns
- Test SSRF protection interactively
- Generate tests based on actual data
- Debug failures by reading logs and database state
- Test scanners with real domains in development

See `/docs/technical.md` for detailed scanner architecture and implementation patterns.

## Scanner Architecture (Core Feature)

### Scanner Types & Weights
- **SSL/TLS Scanner**: 40% weight - Certificate validation, protocol analysis
- **Security Headers Scanner**: 30% weight - HSTS, CSP, security headers
- **DNS/Email Scanner**: 30% weight - SPF, DKIM, DMARC validation

### Implementation Requirements
- All scanners extend AbstractScanner
- Common functionality: rate limiting, SSRF protection, retry logic
- Each scanner implements performScan() method
- Results stored as ModuleResult with score, status, and raw data

### Security Scoring
- Grades: A+ (95-100), A (90-94), B+ (85-89), B (80-84), C (70-79), D (60-69), F (0-59)
- Weighted calculation: Total = (SSL×0.4) + (Headers×0.3) + (DNS×0.3)
- When a scanner fails: Redistribute weights proportionally among working scanners
- Store raw scan data in JSONB for detailed analysis

See `/docs/technical.md` for complete implementation details.

## Database Schema

See **database.md** for complete schema, relationships, indexes, and JSONB structures.

### Core Tables
- **users**: Authentication, billing, OAuth support
- **domains**: User domains with scan history
- **scans**: Security scan records with scoring
- **scan_modules**: Individual scanner results
- **reports**: Generated PDF reports

### Key Relationships
- User hasMany Domains, Scans, Reports
- Domain belongsTo User, hasMany Scans
- Scan belongsTo Domain/User, hasMany ScanModules

## API Design Patterns

### Standard CRUD Controllers
- **GET /domains** - List user domains
- **POST /domains** - Create new domain (with validation)
- **GET /domains/{id}** - Show domain details
- **DELETE /domains/{id}** - Remove domain

### Scan Operations
- **POST /domains/{id}/scan** - Trigger individual scan
- **POST /scans/bulk** - Scan all user domains
- Returns scan ID for tracking progress
- Jobs dispatched to Laravel Cloud workers

## Frontend Patterns

### Page Structure (Inertia.js)
- Use AuthenticatedLayout for protected pages
- Include Head component for page titles
- PageHeader with title and action buttons
- Data passed as props from Laravel controllers

### Form Handling
- React Hook Form for form state management
- Zod for schema validation
- Laravel Precognition for live validation
- Consistent error handling and display

See `/docs/design.md` for complete UI specifications and component library.

## Laravel Package Integration

### Laravel Cashier (Stripe Billing)
- $27/month subscription management
- 14-day trial period
- Webhook handling for payment events
- Subscription status tracking

### Laravel Precognition (Live Validation)
- Real-time form validation
- No duplicate validation rules
- Frontend-backend sync
- Instant user feedback

### Laravel Socialite (OAuth)
- GitHub and Google authentication
- Automatic user creation
- Trial period for OAuth users
- Seamless login flow

See `/docs/technical.md` for implementation patterns and `/docs/execution.md` for setup instructions.

## Testing Approach

See **testing.md** for complete testing strategy, patterns, and examples.

### Testing Stack
- **Unit Tests**: Pest PHP
- **Feature Tests**: Laravel HTTP testing
- **E2E Tests**: Playwright
- **Security Tests**: Custom assertions

### Key Testing Areas
- Scanner functionality with real domains
- SSRF protection validation
- Rate limiting enforcement
- Authorization policies
- Trial period calculations

## Deployment & Production

### Laravel Cloud Configuration
- **URL**: https://cloud.laravel.com/
- Database: PostgreSQL with hibernation (managed)
- Queue: Redis with auto-scaling workers (no manual config needed)
- Storage: S3 for PDF reports
- Monitoring: Built-in application performance monitoring
- Real-time: Laravel Reverb for WebSocket connections
- Workers: Automatic process management and restarts

### Environment Variables
```env
# Core Configuration
APP_ENV=production
APP_DEBUG=false
APP_URL=https://achilleus.so

# Database
DB_CONNECTION=pgsql
DB_DATABASE=achilleus_production

# Queue & Cache (Laravel Cloud managed)
QUEUE_CONNECTION=redis  # Cloud provides Redis
CACHE_DRIVER=redis      # Auto-configured by Cloud

# Stripe (Laravel Cashier)
STRIPE_KEY=pk_live_...
STRIPE_SECRET=sk_live_...
STRIPE_WEBHOOK_SECRET=whsec_...
CASHIER_CURRENCY=usd
CASHIER_CURRENCY_LOCALE=en_US

# OAuth Providers (Laravel Socialite)
GITHUB_CLIENT_ID=...
GITHUB_CLIENT_SECRET=...
GOOGLE_CLIENT_ID=...
GOOGLE_CLIENT_SECRET=...

# Security
BCRYPT_ROUNDS=12
SESSION_SECURE_COOKIE=true
```

## Common Commands

### Development
- `php artisan migrate --seed` - Database setup
- `php artisan queue:work` - Local queue processing
- `npm run dev` - Frontend development server
- `php artisan boost:mcp` - Start AI context server

### Testing
- `php artisan test --parallel` - Run all tests
- `npm run type-check` - TypeScript validation
- `npx playwright test` - E2E tests

### Quality
- `php artisan pint` - PHP code style
- `npm run lint` - JavaScript linting

### Security Testing
- Test SSRF protection with localhost/private IPs
- Verify rate limiting (10 scans/minute)
- Check authorization policies

See `/docs/testing.md` for comprehensive testing commands.

## Critical Security Reminders

1. **NEVER** make HTTP requests without NetworkGuard validation
2. **ALWAYS** validate user input with Form Requests
3. **NEVER** trust external data without sanitization
4. **ALWAYS** use HTTPS-only domain validation
5. **NEVER** log sensitive data (URLs, API keys, personal info)
6. **ALWAYS** implement proper rate limiting
7. **NEVER** expose stack traces in production
8. **ALWAYS** authorize domain access with policies
9. **ALWAYS** enforce trial expiry with EnsureSubscriptionActive middleware
10. **NEVER** allow scanning features without active subscription or trial

## Support & Debugging

### Common Issues
- **SSRF Blocked**: Check NetworkGuard configuration, ensure public URLs only
- **Scan Timeouts**: Verify timeout settings in scanner classes
- **Queue Jobs Failing**: Check failed_jobs table, Laravel Cloud logs
- **Rate Limiting**: Monitor cache keys, adjust limits in config
- **Support Requests**: Direct to support@achilleus.so (manual handling in MVP)

### Logging Strategy
- Log security events (SSRF attempts, auth failures)
- Track scanner errors with context
- Monitor performance metrics
- Use appropriate log levels (error, warning, info)

## Additional Resources

- **Technical Details**: See `/docs/technical.md` for architecture and implementation
- **Database Schema**: See `/docs/database.md` for complete schema
- **Testing Strategy**: See `/docs/testing.md` for TDD approach
- **UI/UX Specs**: See `/docs/design.md` for interface guidelines
- **Development Phases**: See `/docs/execution.md` for timeline and workflow