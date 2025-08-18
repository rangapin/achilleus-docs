# CLAUDE.md

This file provides essential guidance to Claude Code when working with this repository.

## Project Overview

**Achilleus** - Security monitoring SaaS for developers managing multiple websites
**Target**: Freelancers and small businesses needing affordable security monitoring
**Price**: $27/month for 10 domains with unlimited scans
**Stack**: Laravel 12 + React 19 + Shadcn/ui + Inertia.js + Playwright MCP

## Documentation Structure

Complete project documentation is available in `/docs/`:
- `architecture.md` - System overview, technology stack, deployment architecture
- `database.md` - Database schema, relationships, indexes
- `scanners.md` - Scanner implementation, scoring engine, error handling
- `security.md` - SSRF protection, validation, authentication patterns
- `api.md` - API endpoints, request/response formats, error handling
- `deployment.md` - Laravel Cloud setup, environment configuration, monitoring
- `product.md` - Business requirements, UI specifications, user flows
- `execution.md` - Development phases, testing strategy, timeline

## Core Development Principles

### Security-First Approach (Critical)
- **SSRF Protection**: Use `NetworkGuard::assertPublicHttpUrl($url)` for all external requests
- **Input Validation**: Always use Form Requests with custom validation rules
- **HTTPS Only**: Block HTTP URLs, only allow HTTPS domains
- **Rate Limiting**: Implement per-user and per-host rate limiting
- **Error Sanitization**: Never expose internal details in production

### Architecture Patterns  
- **AbstractScanner Pattern**: All scanners extend AbstractScanner base class
- **Service Layer**: Controllers thin, business logic in services
- **Queue Jobs**: Asynchronous processing with retry logic and failure handling
- **Direct Stripe Integration**: No Laravel Cashier, use Stripe API directly
- **Weighted Scoring**: SSL (40%), Headers (30%), DNS (30%)

### Key Technologies & Components
- **Frontend**: React 19 + TypeScript + Inertia.js for SPA experience
- **UI Library**: Shadcn/ui components with dark theme (`bg-black` / `#0a0a0b`)
- **Backend**: Laravel 12 with service layer pattern
- **Testing**: Playwright MCP for E2E, Pest for backend testing
- **Queue**: Laravel Cloud auto-scaling queue workers
- **Billing**: Direct Stripe API integration (no Laravel Cashier)

## Critical Business Rules
- **Domain Limit**: 10 domains per user (enforced in validation)
- **Pricing**: $27/month Solo Plan with 14-day trial
- **Security Modules**: SSL/TLS (40%), Security Headers (30%), DNS/Email (30%)
- **HTTPS Only**: No HTTP URLs allowed, only HTTPS domains
- **Trial**: 14 days, no credit card required, full feature access

```bash
# Testing Commands
php artisan pest --parallel        # Run backend tests
npm run type-check                 # TypeScript validation  
npx playwright test               # E2E testing with MCP

# Development Commands
php artisan serve                 # Start Laravel server
npm run dev                      # Start Vite dev server
php artisan tinker               # Laravel REPL for testing

# Code Quality
php artisan pint                 # Code formatting
npm run lint                     # Frontend linting
```

## What NOT to Include
- **Laravel Cashier** (use direct Stripe API)
- **Client management** (single user per account)
- **Multiple scan types** (one comprehensive scan only)
- **Uptime monitoring** (security focus only)  
- **Team features** (individual accounts only)
- **HTTP domains** (HTTPS only for security)

## Quick Reference Links
- **Architecture**: `docs/architecture.md` - System design and technology stack
- **Database**: `docs/database.md` - Schema, relationships, migrations
- **Scanners**: `docs/scanners.md` - Implementation details and scoring  
- **Security**: `docs/security.md` - SSRF protection and validation rules
- **API**: `docs/api.md` - Endpoints and response formats
- **Deployment**: `docs/deployment.md` - Laravel Cloud configuration