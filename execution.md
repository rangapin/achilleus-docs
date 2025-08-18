# Achilleus Execution Roadmap - Complete MVP Build

## Overview
This document provides the complete task breakdown to build Achilleus MVP from scratch using numbered subtasks. Each subtask is designed to be completed independently while building toward the production-ready security monitoring SaaS.

## Development Principles
- **Backend-First Architecture**: Complete API functionality before frontend
- **Laravel Starter Kit Foundation**: React/Inertia starter with custom UI replacing Breeze
- **Auto-scaling Infrastructure**: Leverage Laravel Cloud's queue clusters and serverless database
- **Test-Driven Development**: Pest framework for comprehensive testing
- **Feature Flag Management**: Laravel Pennant for controlled rollout

## Phase 1: Foundation & Authentication (Backend)

### Subtask 1.1: Laravel Project Setup
```bash
# Initialize Laravel 12 with React starter kit
laravel new achilleus --react --inertia
```
- Configure Laravel 12 with React starter kit and Inertia 2
- Set up PostgreSQL database connection for JSONB support
- Install core dependencies: Pennant, Pulse, ActivityLog, DomPDF, Stripe
- Configure environment variables and basic application settings
- **Reference**: technical.md - Technology Stack section

### Subtask 1.2: Database Schema Design
- Create users table with trial_ends_at, stripe_customer_id, subscription_status
- Design domains table with email_mode, dkim_selector, last_scan tracking
- Build scans table with UUID primary keys and JSONB scan results
- Create scan_modules table for individual scanner results  
- Implement scoring_algorithms table for version control
- **Reference**: technical.md - Database Architecture section

### Subtask 1.3: Authentication Backend (Breeze)
- Install Laravel Breeze authentication scaffolding (backend only)
- Implement email verification requirement with MustVerifyEmail
- Configure password reset and change functionality
- Set up automatic 14-day trial creation on registration
- Add session management with "remember me" functionality

### Subtask 1.4: User Model & Policies
- Enhance User model with Stripe integration methods
- Implement trial management (canScan, isOnTrial, trialDaysLeft)
- Create subscription status tracking and billing methods
- Build authorization policies for domain access and scan permissions
- **Reference**: technical.md - Authorization Implementation section

## Phase 2: Core Security Scanner Backend

### Subtask 2.1: Scanner Architecture
- Create Scanner interface contract for consistent implementation
- Build ModuleResult data structure for scan findings
- Implement NetworkGuard for SSRF protection with IP validation
- Set up scanner configuration files (scanner.php, scoring.php)
- **Reference**: claude.md - Scanner Implementation section

### Subtask 2.2: SSL/TLS Scanner (40% weight)
- Implement certificate validation and expiration checking
- Build cipher strength analysis and protocol version detection
- Add Perfect Forward Secrecy (PFS) verification
- Create scoring algorithm with deductions for weak configurations
- **Reference**: technical.md - Scanner Implementations section

### Subtask 2.3: Security Headers Scanner (30% weight)  
- Build HSTS header analysis with max-age validation
- Implement CSP header parsing and unsafe-inline detection
- Add X-Frame-Options, X-Content-Type-Options, Referrer-Policy checks
- Create weighted scoring with 35 points each for HSTS/CSP
- **Reference**: product.md - Core Value Drivers section

### Subtask 2.4: DNS/Email Scanner (30% weight)
- Implement SPF record validation and permissive policy detection
- Build DKIM verification with custom selector support
- Add DMARC policy checking (none/quarantine/reject scoring)
- Include DNSSEC and CAA record validation
- Support email_mode: 'expected' vs 'none' for non-email domains

### Subtask 2.5: Scoring Engine
- Create SecurityScorer with weighted calculation (40/30/30 split)
- Implement grade assignment (A+ to F) based on total scores
- Build version tracking for scoring algorithm changes
- Add caps and parameter support for future adjustments
- **Reference**: technical.md - Scanner Configuration section

## Phase 3: Queue Processing & Jobs

### Subtask 3.1: Laravel Cloud Queue Setup
- Configure auto-scaling Queue clusters (developer preview)
- Remove manual Horizon configuration dependencies
- Set up Redis connection for queue management
- Implement job retry logic with exponential backoff
- **Reference**: claude.md - Laravel Cloud Queue Architecture section

### Subtask 3.2: Scan Processing Job
- Create RunDomainScan job implementing ShouldQueue
- Build scan orchestration calling all three scanners
- Implement error handling and status tracking
- Add result aggregation and score calculation
- Store structured findings in JSONB format for performance

### Subtask 3.3: HTTP Service Provider
- Create custom HTTP macro with security defaults
- Implement safe redirect following with SSRF protection
- Add proper User-Agent and timeout configurations
- Build connection pooling for scanner efficiency
- **Reference**: technical.md - HTTP Service Provider section

## Phase 4: Domain Management Backend

### Subtask 4.1: Domain Model & Validation
- Create Domain model with proper relationships
- Implement real-time URL validation with helpful feedback
- Add domain limit enforcement (10 domains per user)
- Build active/inactive status management
- Track last_scan_at and last_scan_score for dashboard

### Subtask 4.2: Domain CRUD Controllers
- Build DomainController with complete CRUD operations
- Implement domain creation with duplicate prevention
- Add bulk operations support for "Scan All Domains"
- Create domain editing with email configuration options
- **Reference**: technical.md - API Architecture section

### Subtask 4.3: Scan Triggering System
- Create DomainScanController for manual scan initiation
- Implement scan authorization and rate limiting
- Build scan status tracking and progress indicators
- Add scan history retrieval and filtering

## Phase 5: Stripe Integration (No Cashier)

### Subtask 5.1: Stripe Service Architecture
- Implement direct Stripe API integration without Cashier
- Create customer creation and management methods
- Build subscription lifecycle management
- Add payment method handling and storage
- **Reference**: technical.md - Payment Integration section

### Subtask 5.2: Billing Controllers
- Create BillingController for subscription overview
- Implement subscription creation and cancellation
- Build payment method management interface
- Add invoice history and download functionality

### Subtask 5.3: Webhook Processing
- Set up Stripe webhook endpoint for payment events
- Implement subscription status synchronization
- Handle failed payments and dunning management
- Add trial conversion tracking

## Phase 6: Reporting System Backend

### Subtask 6.1: PDF Generation Engine
- Configure DomPDF with custom Achilleus templates
- Create professional report layouts with branding
- Implement scan result formatting for client consumption
- Build visual security score representations

### Subtask 6.2: Report Management
- Create Report model with scan relationship tracking
- Implement report generation and storage in S3
- Build report history and download functionality
- Add bulk report generation capabilities

## Phase 7: Frontend Implementation (React/Inertia)

### Subtask 7.1: Authentication UI (Replace Breeze)
- Create custom Login component with Achilleus branding
- Build Registration form with trial messaging
- Implement ForgotPassword flow with email verification
- Replace all Breeze views with shadcn/ui components
- **Reference**: claude.md - Authentication Flow section

### Subtask 7.2: Dashboard Implementation
- Build 4-card metrics display (Score, Domains, Last Scan, Issues)
- Implement security trends chart with Recharts integration
- Create trial banner with days remaining countdown
- Add real-time updates with Inertia partial reloads

### Subtask 7.3: Domain Management UI
- Create domain list with bulk action capabilities
- Build domain creation form with real-time validation
- Implement domain editing with email configuration
- Add scan triggering buttons and status indicators

### Subtask 7.4: Scan Results Display
- Build comprehensive scan results page with visual scores
- Implement SSL/TLS detailed analysis display
- Create security headers breakdown with recommendations
- Add DNS/email security status with actionable insights

### Subtask 7.5: Billing Interface
- Create subscription overview with upgrade/downgrade options
- Implement payment method management with Stripe Elements
- Build invoice history with download links
- Add trial conversion flow with seamless payment

### Subtask 7.6: Reports Interface
- Build report generation interface with scan selection
- Implement report history with filtering and search
- Create PDF preview and download functionality
- Add bulk report generation with progress tracking

## Phase 8: Laravel Cloud Deployment

### Subtask 8.1: Cloud Configuration
- Set up Laravel Cloud project with GitHub integration
- Configure auto-scaling PostgreSQL database with hibernation
- Enable auto-scaling Redis cache and S3 storage
- Set up Queue clusters for scan processing
- **Reference**: technical.md - Laravel Cloud Deployment section

### Subtask 8.2: Environment Configuration
- Configure production environment variables
- Set up Stripe webhook endpoints and secrets
- Enable CloudFront CDN for asset delivery
- Configure auto-scaling and hibernation settings

### Subtask 8.3: Domain & SSL Setup
- Add custom domain with automatic SSL provisioning
- Configure DNS settings for production deployment
- Set up email delivery for authentication and notifications
- Test auto-scaling behavior under load

## Phase 9: Testing & Quality Assurance

### Subtask 9.1: Backend Testing Suite
- Write Pest tests for all scanner implementations
- Test queue processing and auto-scaling behavior
- Create integration tests for Stripe webhook processing
- Add authorization and security policy tests

### Subtask 9.2: Frontend Testing
- Test all React components with proper TypeScript coverage
- Verify mobile responsiveness across device sizes
- Test Inertia.js navigation and partial reloads
- Validate cross-browser compatibility

### Subtask 9.3: End-to-End Testing
- Test complete user registration and trial flow
- Verify domain addition and scan processing
- Test payment processing and subscription management
- Validate PDF report generation and download

## Phase 10: Production Launch Preparation

### Subtask 10.1: Performance Optimization
- Optimize database queries with proper indexing
- Configure Redis caching for dashboard metrics
- Set up Laravel Pulse for performance monitoring
- Test hibernation and auto-scaling cost optimization

### Subtask 10.2: Security Hardening
- Implement rate limiting for scan requests and API calls
- Configure CSRF protection and XSS prevention
- Add input validation for all user data entry points
- Test SSRF protection with malicious URL attempts

### Subtask 10.3: Launch Readiness
- Configure error tracking and logging
- Set up monitoring dashboards for queue cluster metrics
- Prepare customer support documentation
- Create onboarding email sequences for trial users

## Task Completion Tracking

Each subtask should be tracked with:
- **Status**: Not Started / In Progress / Complete / Blocked
- **Estimated Hours**: For project planning and resource allocation
- **Dependencies**: Which other subtasks must be completed first
- **Testing Requirements**: Specific tests that must pass
- **Documentation Updates**: Which docs need updating after completion

## Integration Points

### Backend → Frontend Integration
- Complete API endpoints before building corresponding UI components
- Test all authentication flows before implementing frontend auth
- Ensure scan processing works before building result displays

### Local → Cloud Integration  
- Test all functionality locally before deploying to Laravel Cloud
- Verify queue processing works with local Redis before enabling auto-scaling
- Confirm Stripe integration in test mode before production deployment

This execution roadmap provides a clear path from empty Laravel project to production-ready Achilleus MVP, with each subtask building systematically toward the complete solution.