# Achilleus Product Documentation

## Executive Summary

Achilleus is a security monitoring SaaS that provides comprehensive website security scanning at $27/month for up to 10 domains. Built for developers and freelancers managing multiple websites, it delivers enterprise-grade security analysis at 90% less cost than competitors.

## Market Position

### Target Market
- **Primary**: 127,000+ freelance developers managing 5-20 websites
- **Secondary**: Small businesses with multiple web properties
- **Tertiary**: Security-conscious developers needing documentation

### Value Proposition
**"Enterprise security monitoring at freelancer pricing"**
- 10 domains for $27/month (competitors: $250+/month)
- Unlimited security scans (no usage limits)
- Professional PDF reports included
- Single comprehensive scan covering all security vectors

### Competitive Advantage
- **97% cost savings** vs per-site pricing models
- **No client management complexity** - simple, focused tool
- **8/10 security coverage** - optimal balance of completeness and reliability
- **<15 second scans** - fast enough for real-time use

## Product Features

### Core Functionality

#### 1. Security Scanning (One Comprehensive Scan)
**SSL/TLS Analysis (40% weight)**
- Certificate validation and expiration monitoring
- Protocol version checking (TLS 1.2+)
- Cipher suite strength analysis
- Perfect Forward Secrecy verification

**Security Headers (30% weight)**
- HSTS configuration analysis
- Content Security Policy evaluation
- X-Frame-Options verification
- X-Content-Type-Options checking
- Referrer-Policy assessment

**DNS/Email Security (30% weight)**
- SPF record validation
- DKIM verification (with custom selectors)
- DMARC policy analysis
- DNSSEC status checking
- CAA record verification

#### 2. Domain Management
- Add up to 10 domains per account
- HTTPS-only (security-first approach)
- Email configuration options (expected/none)
- Custom DKIM selector support
- Domain validation with helpful feedback

#### 3. Dashboard & Monitoring
**Four Key Metrics**
- Security Score (0-100 with letter grade)
- Active Domains (X of 10 available)
- Last Scan (time and domain)
- Critical Issues (domains scoring <60)

**Security Trends**
- 7, 30, 90-day, and 1-year views
- Visual score progression
- Historical comparison

#### 4. Professional Reporting
**PDF Reports Include**
- Executive summary with overall score
- Module-by-module breakdown
- Prioritized recommendations
- Technical details for developers
- Achilleus branding

#### 5. Activity Tracking
- Complete scan history
- Filter by domain, date, score
- One-click report generation
- Score trend analysis

#### 6. Support System (Simplified MVP)
- Support email displayed in footer and profile
- Basic FAQ on landing page
- Contact email: support@achilleus.so

#### 7. Legal Pages (MVP Requirements)
- Terms of Service acceptance on signup (required)
- Privacy Policy link (static page)

### User Experience

#### Navigation Structure
```
Dashboard (default view)
├── Domains (manage websites)
├── Activity (scan history)  
└── Reports (PDF downloads)

Profile Menu:
├── Profile Settings
├── Subscription (billing management)
└── Sign Out
```

## Technology Stack

- **Backend**: Laravel 12 with PHP 8.3+
- **Frontend**: React 19 with TypeScript
- **UI Library**: Shadcn/ui
- **Infrastructure**: Laravel Cloud
- **Real-time**: Laravel Reverb
- **Payments**: Laravel Cashier (Stripe)

## User Interface

**Complete UI/UX specifications have been moved to design.md**

Key aspects:
- Dark theme design for security focus
- Dashboard-centric navigation
- Real-time updates via WebSockets
- Mobile-responsive layouts
- Comprehensive empty states

## Data Requirements

**See `/docs/design.md` for complete UI component specifications**

### Dashboard Metrics
```
- **Security Score**: Average of all domain scores
- **Active Domains**: Count of active domains (max 10)
- **Last Scan**: Most recent scan timestamp and domain
- **Critical Issues**: Domains scoring below 60

### Security Score Trends
- Time periods: 7, 30, 90 days, 1 year
- Data aggregated by day/week/month
- Color-coded based on score ranges

## Business Logic

### Subscription Management
- **Trial Period**: 14 days from registration
- **Price**: $27/month for Solo Plan
- **Limits**: 10 domains, unlimited scans
- **Billing**: Handled via Laravel Cashier
- **Cancellation**: Immediate with access until period end

### Domain Management Rules
- **HTTPS Only**: No HTTP domains allowed
- **URL Normalization**: Remove www, trailing slashes
- **Duplicate Prevention**: One entry per unique domain
- **Email Mode**: Expected (with SPF/DKIM) or None
- **Deletion**: Cascades to remove all scan history

### Scanning Logic
- **Rate Limiting**: 10 scans/minute per user
- **Scan Duration**: 15-30 seconds typical
- **Retry Logic**: 3 attempts with exponential backoff
- **Failure Handling**: Redistribute weights among working scanners
- **Queue Processing**: Laravel Cloud managed workers

### Data Access Patterns

All UI components pull from database:
- Dashboard metrics from aggregated domain scores
- Activity history from scans table
- Reports from S3 with signed URLs
- Real-time updates via Laravel Reverb

See **database.md** for schema details.

### Trial Management
- Display banner when trial_ends_at > now()
- Show days remaining prominently
- Block scan features after trial expiry
- Automatic transition to subscription or locked state

### Error Handling Strategy

- Empty states with clear CTAs
- Graceful degradation for failed scanners
- User-friendly error messages
- Support email for critical issues
- Automatic retry for transient failures

## Related Documentation

- **Technical Implementation**: See `/docs/technical.md`
- **UI/UX Specifications**: See `/docs/design.md`
- **Database Schema**: See `/docs/database.md`
- **Testing Strategy**: See `/docs/testing.md`
- **Development Timeline**: See `/docs/execution.md`

#### Loading States
- Skeleton loading for data tables
- Spinner for action buttons
- Progress bars for scan progress

This design system ensures consistency across all pages while maintaining the professional security tool aesthetic.

#### Trial Experience
- 14-day free trial
- No credit card required
- Full feature access
- Clear upgrade path
- Trial countdown banner

#### Design Principles
- Dark theme (#0a0a0b background)
- Mobile-responsive
- Accessibility (WCAG AA)
- Fast page loads (<500ms)
- Clear visual hierarchy

## Pricing Strategy

### Solo Plan - $27/month
**Includes:**
- 10 domain slots
- Unlimited scans
- Professional PDF reports
- Full scan history
- Email support

**Why This Price:**
- Above basic tools ($5-15) - signals quality
- Below enterprise ($50+) - remains accessible
- Sweet spot for freelancers
- Sustainable unit economics

**Technical Foundation:**
- Built with Laravel 12 starter kit (React + TypeScript): https://laravel.com/docs/12.x/starter-kits
- Landing page uses Salient template: https://tailwindcss.com/plus/templates/salient

### Trial Strategy
- **14 days**: Optimal for technical evaluation
- **No restrictions**: Full features during trial
- **No card required**: Reduces friction
- **Clear value**: Immediate scan results

## Business Model

### Revenue Model
- **SaaS subscription**: Monthly recurring revenue
- **Single tier**: Simplicity over complexity
- **Billing**: Laravel Cashier with Stripe integration
- **Auto-renewal**: Reduced churn

### Unit Economics (per customer)
- **Revenue**: $27/month
- **Infrastructure**: ~$3/month (with hibernation)
- **Payment processing**: ~$1.50/month
- **Gross margin**: ~80%

### Growth Strategy
1. **Land**: Acquire developers with free trial
2. **Activate**: First scan within 24 hours
3. **Convert**: Trial to paid at day 10-12
4. **Retain**: Consistent value delivery

## Market Analysis

### Total Addressable Market
- **Freelance developers**: 127,000 × $324/year = $41M
- **Small businesses**: 50,000 × $324/year = $16M
- **Total TAM**: $57M annually

### Serviceable Addressable Market
- **Target**: 1% market share in year 1
- **SAM**: $570,000 annual revenue
- **Goal**: 1,760 customers @ $27/month

### Competition Landscape

#### Direct Competitors
| Service | Price/Domain | 10 Domains | Limitations |
|---------|-------------|------------|-------------|
| SiteLock | $24.99 | $250/month | No headers analysis |
| Sucuri | $28.00 | $280/month | WordPress focus |
| MalCare | $24.92 | $249/month | Basic SSL only |
| **Achilleus** | **$2.70** | **$27/month** | **None** |

#### Why We Win
- **Price**: 90% cheaper for multi-domain users
- **Simplicity**: One scan type, clear scoring
- **Speed**: 15-second scans vs 2-5 minutes
- **Focus**: Security only, no feature bloat

## User Journey Documentation

### New User Journey (Trial to Paid)

#### 1. Discovery & Signup (0-5 minutes)
**Entry Points:**
- Landing page via search/referral
- Product Hunt or community recommendation
- Comparison article

**User Actions:**
1. Lands on homepage, sees "$27 for 10 domains" value prop
2. Clicks "Start Free Trial" (no credit card required)
3. Fills signup form (name, email, password)
4. Accepts Terms of Service (checkbox required)
5. Receives welcome email with trial details

**System Response:**
- Creates user account with 14-day trial
- Sets trial_ends_at timestamp
- Sends welcome email
- Redirects to empty dashboard

#### 2. First Domain Addition (5-10 minutes)
**User Actions:**
1. Sees empty dashboard with "Add Your First Domain" CTA
2. Clicks "Add Domain" button
3. Enters HTTPS URL (e.g., https://mysite.com)
4. Selects email configuration (expected/none)
5. Clicks "Add & Scan"

**System Response:**
- Validates URL (HTTPS-only, public IP)
- Creates domain record
- Triggers automatic first scan
- Shows scan progress in real-time

#### 3. First Scan Results (10-15 minutes)
**User Actions:**
1. Watches scan progress (15-30 seconds)
2. Views security score and grade (A-F)
3. Explores module breakdowns (SSL, Headers, DNS)
4. Reviews recommendations list

**System Response:**
- Displays overall score with color coding
- Shows detailed module results
- Provides actionable recommendations
- Updates dashboard metrics

#### 4. Exploration Phase (Days 1-7)
**User Actions:**
- Adds more domains (up to 10)
- Runs additional scans
- Generates first PDF report
- Explores scan history

**Key Moments:**
- Day 3: In-app trial progress indicator
- Day 5: Dashboard shows feature exploration
- Day 7: Trial halfway banner emphasis

#### 5. Conversion Decision (Days 8-14)
**User Actions:**
- Evaluates value from scan results
- Compares with alternatives ($27 vs $250+)
- Decides to upgrade or abandon

**Conversion Triggers:**
- Day 10: Prominent upgrade prompt in dashboard
- Day 12: Trial banner shows urgency
- Day 14: Trial expiration with clear upgrade path

#### 6. Paid Subscription (Post-Trial)
**User Actions:**
1. Clicks "Upgrade Now"
2. Enters payment information (Stripe)
3. Confirms $27/month subscription
4. Continues using full features

**System Response:**
- Processes payment via Stripe
- Updates subscription_status to 'active'
- Removes trial banner
- Sends payment confirmation

### Returning User Journey (Daily/Weekly Use)

#### Quick Scan Workflow (2-3 minutes)
1. **Login** → Dashboard
2. **Select Domain** → Click scan icon
3. **View Results** → Check score changes
4. **Generate Report** → Download PDF

#### Bulk Operations (5-10 minutes)
1. **Login** → Dashboard
2. **Click "Scan All"** → Queue all domains
3. **Monitor Progress** → Watch scan queue
4. **Review Results** → Check for issues
5. **Export Data** → Generate reports

### Edge Case User Journeys

#### Failed Payment Recovery
1. Payment fails → Dashboard notification
2. Grace period (3 days) → Limited access
3. Update payment method → Full access restored
4. Continued failure → Account suspended

#### Domain Limit Reached
1. Attempts to add 11th domain
2. Sees "Domain limit reached" message
3. Options presented:
   - Remove existing domain
   - Upgrade plan (future feature)
   - Contact support

#### Scanner Failure Handling
1. Scan fails (timeout/error)
2. User sees error message with reason
3. Retry option provided
4. Support contact if persistent

### Trial Expiry & Subscription Enforcement

#### Trial Period Lifecycle
1. **Registration**: 14-day trial starts automatically
2. **Day 1-11**: Full access to all features
3. **Day 12-14**: Warning banner appears (3 days remaining)
4. **Day 14**: Critical warning (trial ends today)
5. **Day 15+**: Trial expired, features restricted

#### Trial Expiry User Flow
1. **User with Expired Trial Attempts Scan**:
   - Click "Scan Now" button
   - Redirected to billing page
   - Error message: "Your trial has expired. Please subscribe to continue."
   - Subscribe button prominently displayed
   - Trial usage statistics shown

2. **User with Expired Trial Views Dashboard**:
   - Dashboard accessible (read-only)
   - All action buttons disabled
   - Red banner: "Trial Expired - Subscribe Now"
   - Can view existing domains and past scans
   - Cannot add domains or run new scans

3. **User with Expired Trial Attempts Report Generation**:
   - Report button disabled
   - Tooltip: "Subscription required"
   - Clicking redirects to billing page
   - Previous reports still downloadable

#### Grace Period for Payment Failures
1. **Initial Payment Failure**:
   - 3-day grace period activated
   - Yellow warning banner appears
   - Email notification sent
   - Full access maintained

2. **During Grace Period**:
   - Daily reminder emails
   - Warning banner on all pages
   - "Update Payment Method" CTA
   - Countdown timer shown

3. **Grace Period Expired**:
   - Access restricted like expired trial
   - Red banner: "Subscription Suspended"
   - All scanning features disabled
   - Existing data remains accessible

#### Subscription States & Access Levels

| State | Dashboard | View Domains | Add Domains | Run Scans | Generate Reports | Billing |
|-------|-----------|--------------|-------------|-----------|------------------|---------|
| Trial Active | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Trial Expired | ✅ | ✅ | ❌ | ❌ | ❌ | ✅ |
| Active Subscription | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Grace Period | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Suspended | ✅ | ✅ | ❌ | ❌ | ❌ | ✅ |
| Cancelled (Active) | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Cancelled (Expired) | ✅ | ✅ | ❌ | ❌ | ❌ | ✅ |

#### Reactivation Flow
1. **Expired Trial User Subscribes**:
   - Immediate access restoration
   - All features unlocked
   - Trial data preserved
   - Welcome back email sent

2. **Suspended User Updates Payment**:
   - Payment processed immediately
   - Access restored within 1 minute
   - Grace period counter reset
   - Confirmation email sent

## Product Roadmap

### Current MVP (Launched)
✅ Three-module security scanning
✅ 10 domain management
✅ Professional PDF reports
✅ Stripe billing integration
✅ 14-day free trial

### Phase 2 (Months 2-3)
- Email notifications for critical issues
- Scan scheduling (daily/weekly/monthly)
- API access for integrations
- Enhanced PDF customization

### Phase 3 (Months 4-6)
- Additional security modules
- Bulk domain import
- CSV/JSON exports
- Webhook notifications

### Future Vision (Year 2)
- Team accounts (multi-user)
- White-label options
- Advanced analytics
- Compliance reporting

## Success Metrics

### Key Performance Indicators
- **Trial conversion rate**: Target 25%
- **Monthly churn**: Target <5%
- **Customer acquisition cost**: <$50
- **Lifetime value**: >$500
- **Monthly recurring revenue**: $8,100 (300 customers)

### Product Metrics
- **Scan completion rate**: >95%
- **Time to first scan**: <5 minutes
- **Support tickets**: <5% of users/month
- **Uptime**: 99.9%

### User Satisfaction
- **NPS score**: Target >50
- **Feature requests**: Track and prioritize
- **Churn reasons**: Monthly analysis
- **Review ratings**: 4.5+ stars

## Risk Analysis

### Technical Risks
- **Scanner accuracy**: Continuous testing and validation
- **False positives**: Conservative scoring approach
- **Scaling issues**: Laravel Cloud auto-scaling
- **Security breaches**: SSRF protection, input validation

### Business Risks
- **Low conversion**: Optimize onboarding flow
- **High churn**: Focus on consistent value
- **Competition**: Maintain price advantage
- **Support burden**: Comprehensive documentation

### Mitigation Strategies
- Regular security audits
- Automated testing pipeline
- Customer feedback loops
- Proactive monitoring

## Go-to-Market Strategy

### Positioning
**"The security monitoring tool that doesn't cost more than your hosting"**

### Launch Channels
1. **Product Hunt**: Technical audience
2. **Developer communities**: Reddit, HN, Dev.to
3. **Content marketing**: Security best practices
4. **Comparison pages**: vs competitors

### Messaging Framework
- **Problem**: Security tools charge per-site, becoming unaffordable
- **Solution**: Flat-rate pricing for 10 domains
- **Benefit**: Enterprise security at freelancer prices
- **Proof**: Live demo, free trial

### Customer Acquisition
1. **Organic search**: "affordable security monitoring"
2. **Comparison content**: Alternative to [competitor]
3. **Developer tools directories**: Free listings
4. **Word of mouth**: Referral incentives

## Business Rules & Constraints

### Domain Limits
```php
const MAX_DOMAINS = 10;
const MAX_SCANS_PER_MINUTE = 10; // Rate limiting
```

### Trial Configuration
```php
const TRIAL_DAYS = 14;
const TRIAL_FEATURES = 'full'; // No restrictions during trial
```

### Subscription Details
```php
const PLAN_NAME = 'Solo Plan';
const PLAN_PRICE = 27; // USD per month
const SCAN_LIMIT = 'unlimited';
```

### Domain Validation Rules
- Must be HTTPS URL (HTTP rejected)
- Private IPs blocked (192.168.x, 10.x, etc.)
- Localhost blocked
- URL paths ignored (normalized to domain)
- Duplicate domains prevented per user

## Legal Pages (Simplified for MVP)

### Terms of Service
Basic terms page (`/terms`) includes:
- Service description and limitations
- Acceptable use policy
- Payment terms and refund policy
- Liability limitations
- Intellectual property rights
- Termination conditions

### Privacy Policy
Basic privacy page (`/privacy`) includes:
- What data we collect (email, domains, scan results)
- How we use it (to provide the service)
- We don't sell data
- Contact: support@achilleus.so

## What This Project Does NOT Include

❌ **Client Management** - No client portals or multi-tenancy
❌ **Automation** - No automated/scheduled scans in MVP
❌ **Multiple Scan Types** - Single comprehensive scan only
❌ **Uptime Monitoring** - Security focus only
❌ **White-labeling** - Achilleus branding only
❌ **Team Features** - Single user per account
❌ **API Access** - Web interface only in MVP
❌ **Laravel Cashier** - Direct Stripe API integration
❌ **Horizon** - Laravel Cloud handles queues
❌ **Complex Support System** - Simple email only for MVP
❌ **Legal Compliance Features** - Basic terms and privacy pages only
❌ **Email System** - Contact support via support@achilleus.so only

## Conclusion

Achilleus addresses a clear market need: affordable security monitoring for developers managing multiple websites. By focusing on essential security checks, maintaining simplicity, and offering revolutionary pricing, Achilleus can capture significant market share in the underserved freelancer and small business segments.

The product is designed to be:
- **Affordable**: $27/month for 10 domains
- **Reliable**: 8/10 security coverage with low false positives
- **Fast**: 15-second scans
- **Simple**: One scan type, clear scoring
- **Professional**: PDF reports for documentation

This combination creates a compelling value proposition that competitors cannot match without destroying their revenue models.

## Deployment Strategy

**Laravel Cloud Only**: This project is designed specifically for Laravel Cloud deployment (https://cloud.laravel.com/). No self-hosting, Docker, or VPS deployment options are provided in the MVP. Laravel Cloud provides automatic scaling, SSL certificates, database management, and global CDN out of the box.