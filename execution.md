# Achilleus Execution Plan

## Project Overview
Build a security monitoring SaaS from scratch using Laravel 12 with React starter kit. The application provides automated security scanning for up to 10 domains at $27/month, targeting developers and freelancers who need affordable security monitoring.

## Development Phases

**🔴 TDD APPROACH: Write tests first, then implement features. Each subtask specifies what to test and what to defer.**

### Phase 0: Repository Setup (You Handle)
```bash
# Initial repository creation
git init achilleus
cd achilleus

# Laravel 12 project with React 19 starter kit
laravel new . --stack=react --git --no-interaction
# This automatically includes:
# - React 19 with TypeScript
# - Inertia.js for server-side routing
# - Shadcn/ui component library
# - Tailwind CSS with dark theme support

# Install additional packages
composer require barryvdh/laravel-dompdf stripe/stripe-php
composer require --dev laravel/pint pestphp/pest

# Install Playwright MCP for enhanced testing
npm install --save-dev @playwright/test
npx playwright install
npm install --save-dev @microsoft/playwright-mcp

# Initialize Shadcn/ui components
npx shadcn@latest init
# Select dark theme and configure for existing project

# Add commonly needed components
npx shadcn@latest add button card form input select badge alert

# Add Salient template to resources/marketing/
mkdir resources/marketing
# Copy Salient template files here

# Initial commit
git add .
git commit -m "Initial Achilleus with Laravel 12 + React 19 + Shadcn/ui + Playwright MCP"
git remote add origin [repository-url]
git push -u origin main
```

### Phase 1: Foundation Setup (Day 1-2)

#### 1.1 Project Verification
**📝 Verify Phase 0 setup - no additional tests needed**

**Verify:**
- Laravel 12 with React starter installed
- Required packages installed
- Salient template in resources/marketing/
- Git repository configured

**Don't Test:**
- Package functionality (tested in later phases)
- Template customization (Phase 9.5)

#### 1.2 Database Setup
Create migrations for core tables:
- Users (with trial_ends_at, stripe fields)
- Domains (with email_mode, DKIM selector)
- Scans (with status, scoring)
- Scan_modules (with raw JSON results)
- Reports (for PDF tracking)

#### 1.3 Authentication Configuration
**📝 Note: Laravel 12 React starter includes authentication out-of-the-box**

- Verify Laravel 12 React starter auth pages work correctly
- Customize existing auth forms with Shadcn/ui components
- Update registration to add 14-day trial (trial_ends_at field)
- Configure email verification (already included)
- Test authentication flow with new Shadcn/ui styling
- Ensure dark theme consistency across auth pages

#### 1.4 Laravel Cloud Preparation
- Initialize git repository
- Configure .env.example
- Set up GitHub repository
- Prepare for Laravel Cloud deployment

### Phase 2: Scanner Implementation (Day 3-5)

#### 2.1 Scanner Architecture
Create core scanner structure:
```
app/Contracts/Scanners/Scanner.php
app/Data/ModuleResult.php
app/Support/NetworkGuard.php
```

#### 2.2 SSL/TLS Scanner (40% weight)
Implement certificate validation:
- Check expiration and validity
- Detect weak protocols (TLS < 1.2)
- Identify weak ciphers
- Verify Perfect Forward Secrecy
- Add TLS 1.3 bonus scoring

#### 2.3 Security Headers Scanner (30% weight)
Analyze HTTP headers:
- HSTS (35 points)
- CSP (35 points)
- X-Content-Type-Options (10 points)
- X-Frame-Options (10 points)
- Referrer-Policy (10 points)

#### 2.4 DNS/Email Scanner (30% weight)
Check DNS configuration:
- SPF records and validation
- DKIM verification
- DMARC policies
- DNSSEC status
- CAA records

#### 2.5 Scoring Engine
Build weighted scoring system:
- Calculate module scores
- Apply weights (40/30/30)
- Assign grades (A+ to F)
- Version tracking

### Phase 3: Queue & Job Processing (Day 6-7)

#### 3.1 Queue Configuration
Set up Laravel Cloud queues:
- Configure Redis connection
- Create RunDomainScan job
- Implement retry logic
- Add failure handling

#### 3.2 Scan Orchestration
Build scan workflow:
- Status tracking (pending → running → completed)
- Execute scanners in sequence
- Store results in JSONB
- Update domain last_scan fields

#### 3.3 HTTP Service Configuration
Create safe HTTP client:
- SSRF protection via NetworkGuard
- Timeout configuration
- User-Agent headers
- Redirect following

### Phase 4: Domain Management (Day 8-9)

#### 4.1 Domain Model & Validation
Create domain functionality:
- Domain model with relationships
- URL validation and normalization
- Duplicate prevention
- 10 domain limit enforcement

#### 4.2 Domain Controllers
Build CRUD operations:
- DomainController (index, store, update, destroy)
- DomainScanController (trigger scans)
- Form request validation
- Authorization policies

#### 4.3 Domain UI Components with Shadcn/ui
**🧪 TDD: Test domain management interface using Playwright MCP**

**Test First (Enhanced with Playwright MCP):**
```typescript
// Using Playwright MCP for structured testing
test('domains page shows table with correct columns', async ({ page }) => {
  // Playwright MCP provides accessibility tree navigation
  await page.goto('/domains');
  const table = await page.getByRole('table');
  await expect(table).toContainText(['Domain Name', 'Last Scan', 'Score', 'Actions']);
});

test('user can add domain via shadcn dialog', async ({ page }) => {
  await page.goto('/domains');
  await page.getByRole('button', { name: 'Add Domain' }).click();
  // Test Shadcn/ui Dialog component
  await expect(page.getByRole('dialog')).toBeVisible();
});

test('form validation with shadcn components', async ({ page }) => {
  // Test Shadcn/ui Form with React Hook Form validation
  await page.fill('[data-testid="domain-url"]', 'invalid-url');
  await page.click('[data-testid="submit"]');
  await expect(page.getByText('Invalid URL format')).toBeVisible();
});
```

**Then Implement with Shadcn/ui:**
- **Domains Page Layout:**
  - Header with Shadcn/ui Button components ("Add Domain", "Scan Now")
  - Shadcn/ui Tabs component ("Your Domains" visible, "All Clients" hidden)
  - Shadcn/ui Table component with responsive design
- **Table Columns using Shadcn/ui:**
  - Domain Name with Shadcn/ui Avatar (favicon) + Link
  - Last Scan Date with Shadcn/ui Badge for time formatting
  - Security Score with color-coded Shadcn/ui Badge
  - Actions with Shadcn/ui Button variants (ghost icons)
- **Add Domain Modal:**
  - Shadcn/ui Dialog component
  - Shadcn/ui Form with React Hook Form + Zod validation
  - Shadcn/ui Input with proper HTTPS validation styling
- **Domain Counter:**
  - Shadcn/ui Badge in sidebar navigation

**Shadcn Components to Use:**
- `Dialog`, `Button`, `Form`, `Input`, `Table`, `Badge`, `Avatar`, `Tabs`

**Don't Test Yet:**
- Client management features (not in MVP)
- Bulk domain operations (future phases)
- Advanced domain settings (future phases)

### Phase 5: Dashboard & UI (Day 10-12)

#### 5.1 Dashboard Implementation with Shadcn/ui
**🧪 TDD: Test dashboard data aggregation and UI using Playwright MCP**

**Test First (Enhanced with Playwright MCP):**
```php
// Backend tests remain the same
test('dashboard calculates average security score')
test('dashboard counts active domains correctly')
test('dashboard shows last scan instead of uptime')
test('dashboard counts critical issues')
test('dashboard handles users with no scans')
test('security trends chart displays correctly')
```

**Playwright MCP Tests:**
```typescript
test('dashboard shows 4 metric cards with shadcn styling', async ({ page }) => {
  await page.goto('/dashboard');
  const cards = await page.locator('[data-testid="dashboard-card"]');
  await expect(cards).toHaveCount(4);
  // Test dark theme styling
  await expect(cards.first()).toHaveClass(/bg-card/);
});

test('trial banner uses shadcn alert component', async ({ page }) => {
  await page.goto('/dashboard');
  const banner = await page.getByRole('alert'); // Shadcn Alert component
  await expect(banner).toContainText('Solo Plan Trial');
  await expect(banner).toContainText('days remaining');
});

test('security trends chart with shadcn select', async ({ page }) => {
  const select = await page.getByRole('combobox', { name: 'Time Period' });
  await select.click();
  await expect(page.getByRole('option', { name: '7 Days' })).toBeVisible();
});
```

**Then Implement with Shadcn/ui:**
- **4-Card Dashboard Grid:**
  - Shadcn/ui Card components with dark theme
  - Security Score Card: Large number with Shadcn/ui Badge for grade
  - Active Domains Card: Counter with progress indicator
  - **Last Scan Card** (replaces uptime): Time ago with domain name
  - Critical Issues Card: Alert styling for urgent items
- **Trial Banner:**
  - Shadcn/ui Alert component with info variant
  - "Solo Plan Trial" (corrected from Team Plan)
  - "X of 10 domains" (corrected from 25)
  - Shadcn/ui Button for "Upgrade Now"
  - Dismissible with close icon
- **Security Trends Chart:**
  - Chart.js or Recharts with dark theme
  - Shadcn/ui Select for time period dropdown
  - Responsive design with Shadcn/ui Card container
- **Actions:**
  - Shadcn/ui Button for "Scan All Domains"

**Shadcn Components to Use:**
- `Card`, `CardContent`, `CardHeader`, `Alert`, `Badge`, `Button`, `Select`

**Corrections Made:**
- Changed "Team Plan" to "Solo Plan" for consistency
- Changed "25 domains" to "10 domains" per product specs
- Replaced "Uptime" with "Last Scan" card as uptime is not in MVP

**Don't Test Yet:**
- Real-time updates (future phases)
- Advanced analytics (future phases)
- Performance with large datasets (Phase 8)

#### 5.2 Navigation Structure
Create layout:
- Sidebar: Dashboard, Domains, Activity, Reports
- User menu: Profile, Billing, Sign out
- Trial banner component
- Dark theme (#0a0a0b)

#### 5.3 Activity Page
Scan history interface:
- Activity list with filtering
- Date range selection
- Score display
- Generate report buttons

#### 5.4 Scan Results Display
Results presentation:
- Overall score and grade
- Module breakdowns
- Issue explanations
- Visual indicators

### Phase 6: Billing Integration (Day 13-15)

#### 6.1 Stripe Service
Direct API integration:
```php
class StripeService {
    - createCustomer()
    - createSubscription()
    - updatePaymentMethod()
    - cancelSubscription()
}
```

#### 6.2 Billing Controllers
Payment endpoints:
- Subscribe endpoint
- Cancel endpoint
- Update payment method
- Webhook handler

#### 6.3 Billing UI
React billing pages:
- Subscription overview
- Payment method management
- Invoice history
- Upgrade from trial

#### 6.4 Webhook Processing
Handle Stripe events:
- subscription.created
- subscription.updated
- subscription.deleted
- payment_failed

### Phase 7: Reporting System (Day 16-17)

#### 7.1 PDF Generation
Configure DomPDF:
- Executive summary template
- Detailed findings layout
- Recommendations section
- Achilleus branding

#### 7.2 Report Management
Build report features:
- Report model and storage
- S3 integration
- Signed URL generation
- Download tracking

#### 7.3 Report UI
Create interface:
- Reports list page
- Generate report button
- Download functionality
- History filtering

### Phase 8: Landing Page & Integration Testing (Day 18-19)

#### 8.1 Landing Page Implementation
**📝 Marketing site creation - high priority for launch**

**Test First:**
```playwright
test('landing page loads under 2 seconds')
test('signup CTA redirects to app registration')
test('pricing section displays correctly')
test('dashboard screenshots display properly')
test('mobile navigation works correctly')
```

**Then Implement:**
- Install and customize Salient template
- Create marketing landing page at `/`
- Add dashboard screenshots and testimonials
- Implement pricing section with $27/month plan
- Connect signup flow to app registration
- Mobile-responsive design
- SEO optimization (meta tags, schema)

**Don't Test Yet:**
- A/B testing variations (future phases)
- Advanced analytics (future phases)
- Complex marketing funnels (future phases)

#### 8.2 Real Integration Tests with Playwright MCP
**🧪 Test with real external services using enhanced browser automation**

**Integration Tests (Enhanced with Playwright MCP):**
```php
// Backend integration tests
test('ssl scanner works with real certificate')
test('headers scanner fetches real headers')
test('dns scanner resolves real records')
test('full scan pipeline completes')
test('stripe webhook processes real events')
```

**Playwright MCP Integration Tests:**
```typescript
// Using Playwright MCP for deterministic browser testing
test('end-to-end scan flow with real domain', async ({ page }) => {
  // Playwright MCP provides structured page interaction
  await page.goto('/domains');
  await page.getByRole('button', { name: 'Add Domain' }).click();
  
  // Test with badssl.com for SSL scenarios
  await page.fill('[data-testid="domain-url"]', 'https://badssl.com');
  await page.getByRole('button', { name: 'Add & Scan' }).click();
  
  // Wait for scan completion using accessibility tree
  await page.waitForSelector('[data-testid="scan-completed"]');
  
  // Verify results display correctly
  const scoreCard = await page.getByTestId('security-score');
  await expect(scoreCard).toBeVisible();
});

test('stripe checkout flow integration', async ({ page }) => {
  // Test Stripe integration with MCP browser automation
  await page.goto('/billing');
  await page.getByRole('button', { name: 'Upgrade Now' }).click();
  
  // Playwright MCP can interact with Stripe's iframe safely
  const stripeFrame = await page.frameLocator('iframe[src*="stripe"]');
  await stripeFrame.fill('[placeholder="Card number"]', '4242424242424242');
  // Continue with test card flow...
});
```

**Enhanced Test Infrastructure:**
- **Test Domains:** badssl.com, httpbin.org for various SSL/HTTP scenarios
- **DNS Testing:** Use controlled domains with known DNS configurations
- **Stripe Testing:** Stripe test mode with Playwright MCP automation
- **Network Monitoring:** Playwright MCP can track network requests
- **Screenshot Comparison:** Visual regression testing capabilities

#### 8.3 End-to-End User Flows with Playwright MCP
**🧪 Enhanced E2E testing with structured browser automation**

**Complete User Flows (Playwright MCP Enhanced):**
```typescript
test('new user signup to first scan flow', async ({ page }) => {
  // Playwright MCP provides fast, lightweight automation
  await page.goto('/');
  await page.getByRole('link', { name: 'Sign Up' }).click();
  
  // Registration with Shadcn/ui forms
  await page.fill('[name="email"]', 'test@example.com');
  await page.fill('[name="password"]', 'password123');
  await page.getByRole('button', { name: 'Create Account' }).click();
  
  // Verify trial setup
  await expect(page.getByText('14-day trial started')).toBeVisible();
  
  // Add first domain
  await page.getByRole('button', { name: 'Add Your First Domain' }).click();
  await page.fill('[name="url"]', 'https://example.com');
  await page.getByRole('button', { name: 'Add & Scan' }).click();
  
  // Wait for scan completion using MCP structured data
  await page.waitForSelector('[data-scan-status="completed"]');
  await expect(page.getByTestId('security-score')).toBeVisible();
});

test('trial to subscription upgrade flow', async ({ page }) => {
  // Test complete billing flow with Stripe integration
  await page.goto('/dashboard');
  await page.getByRole('button', { name: 'Upgrade Now' }).click();
  
  // Playwright MCP can safely interact with Stripe Elements
  await page.waitForSelector('iframe[src*="stripe"]');
  const stripe = page.frameLocator('iframe[src*="stripe"]');
  await stripe.fill('[placeholder="Card number"]', '4242424242424242');
  await stripe.fill('[placeholder="MM / YY"]', '12/28');
  await stripe.fill('[placeholder="CVC"]', '123');
  
  await page.getByRole('button', { name: 'Subscribe' }).click();
  await expect(page.getByText('Subscription activated')).toBeVisible();
});
```

**Enhanced Testing Capabilities:**
- **Structured Automation:** Uses accessibility tree instead of pixel-based
- **Network Monitoring:** Track API calls and responses
- **Cross-browser Testing:** Chrome, Firefox, Safari, Mobile
- **Performance Testing:** Lighthouse integration
- **Visual Regression:** Screenshot comparison
- **Accessibility Testing:** WCAG compliance checks

#### 8.4 Performance & Load Testing
**🧪 Test system under realistic load**

**Performance Tests:**
```php
test('dashboard loads under 500ms')
test('scan completes under 15 seconds')
test('pdf generates under 5 seconds')
test('concurrent scans handle properly')
```

**Load Testing:**
- 50 concurrent users
- 100 scans per minute
- Memory usage monitoring
- Queue latency testing

### Phase 9: Support & Engagement Systems (Day 17-18)

#### 9.1 Email Communication System
Build automated email sequences:
- Welcome email series (registration, first domain, first scan)
- Trial reminder emails (days 7, 10, 12, 14)
- Scan completion notifications with contextual content
- Billing confirmations and payment notifications
- Email preference management and unsubscribe handling

#### 9.2 Support Infrastructure
Create comprehensive support system:
- Contact form with intelligent routing (billing, technical, feature, other)
- FAQ system with search and voting functionality
- In-app feedback widget with contextual triggers
- Support ticket management and auto-response system
- Knowledge base integration

#### 9.3 Legal & Compliance Pages
Implement GDPR-compliant legal framework:
- Terms of Service with domain ownership requirements
- Privacy Policy with transparent data handling practices
- Cookie consent banner with preference management
- Legal document versioning and acceptance tracking
- Data export and account deletion functionality

#### 9.4 Guided Onboarding Flow
Build user activation and engagement system:
- Interactive welcome sequence with personalization
- Real-time scan progress with educational content
- Achievement system with milestone celebrations
- Progressive feature disclosure based on user behavior
- Onboarding analytics and conversion tracking

### Phase 10: Laravel Cloud Deployment (Day 19)

#### 10.1 Cloud Configuration
Set up Laravel Cloud:
- Create project
- Connect GitHub
- Configure environment
- Enable queue clusters

#### 10.2 Database & Services
Configure services:
- PostgreSQL with hibernation
- Redis auto-scaling
- S3 for reports
- CloudFront CDN

#### 10.3 Deploy & Test
Launch application:
- Deploy to production
- Test all features
- Monitor performance
- Check auto-scaling

### Phase 11: Polish & Launch (Day 20)

#### 11.1 Performance Optimization
- Database query optimization
- Redis caching implementation
- Frontend bundle optimization
- Image optimization

#### 11.2 Security Hardening
- Rate limiting configuration
- CSRF verification
- Input validation review
- Security headers

#### 11.3 Final UI Polish
**🧪 Final testing and UI consistency**

**Test:**
- All pages use black theme consistently
- Trial banner appears on all pages
- Navigation highlights work correctly
- Color coding matches design system
- Mobile responsiveness
- Accessibility compliance

**Implement:**
- Final design system consistency
- Polish animations and transitions
- Optimize bundle size
- Browser compatibility testing

#### 11.4 Documentation
- User documentation
- API documentation
- Deployment guide
- Support documentation

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