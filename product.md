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

#### 6. Email Communication System
- Welcome email sequence for trial activation
- Automated trial reminders (days 7, 10, 12, 14)
- Scan completion notifications with results summary
- Billing and payment confirmation emails
- Preference management and unsubscribe options

#### 7. Support Infrastructure
- Multi-channel contact form with intelligent routing
- Comprehensive FAQ system with search functionality
- In-app feedback widget for user insights
- Knowledge base integration for self-service

#### 8. Guided Onboarding Flow
- Interactive welcome sequence with personalization
- Real-time first scan progress with educational content
- Achievement system with milestone celebrations
- Progressive feature discovery to reduce overwhelm
- Onboarding analytics for conversion optimization

#### 9. Legal & Compliance
- GDPR-compliant Terms of Service and Privacy Policy
- Cookie consent management with user preferences
- Data export and account deletion functionality
- Legal document versioning and acceptance tracking

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

## Design System & UI Specifications

### Color Scheme
- **Primary Background**: Black (#000000) or very dark gray (#0a0a0b)
- **Card Backgrounds**: Dark gray (#1a1a1a)
- **Text Primary**: White (#ffffff)
- **Text Secondary**: Light gray (#a0a0a0)
- **Accent Colors**:
  - Success/Green: #22c55e (scores 80+)
  - Warning/Orange: #f59e0b (scores 60-79)
  - Danger/Red: #ef4444 (scores <60, critical issues)
  - Info/Blue: #3b82f6 (trial banner, action buttons)

### Typography
- **Font Family**: System font stack (Inter or similar)
- **Heading Sizes**: 
  - H1: 2.5rem (40px) - Page titles
  - H2: 2rem (32px) - Section headers
  - H3: 1.5rem (24px) - Card titles
- **Body Text**: 0.875rem (14px)
- **Small Text**: 0.75rem (12px) - Subtitles, metadata

## Data-Driven UI Components

### Dashboard Cards (Dynamic Data)
```
Card Layout: 2x2 Grid

┌─────────────────────────┐ ┌─────────────────────────┐
│ SECURITY SCORE          │ │ ACTIVE DOMAINS          │
│                         │ │                         │
│ {avg_score}/100         │ │ {domain_count}          │
│ {grade_badge}           │ │ {count}/{max} available │
│ {vs_industry_text}      │ │                         │
└─────────────────────────┘ └─────────────────────────┘

┌─────────────────────────┐ ┌─────────────────────────┐
│ LAST SCAN               │ │ CRITICAL ISSUES         │
│                         │ │                         │
│ {time_ago}              │ │ {critical_count}        │
│ {domain_name}           │ │ {issues_text}           │
│ {date_range}            │ │                         │
└─────────────────────────┘ └─────────────────────────┘
```

**Data Sources:**
- `{avg_score}`: ROUND(AVG(domains.last_scan_score)) WHERE user_id = auth_user
- `{grade_badge}`: Calculated from avg_score using grade mapping
- `{vs_industry_text}`: "Above/Below industry average" (70 = industry baseline)
- `{domain_count}`: COUNT(domains) WHERE user_id = auth_user AND is_active = true
- `{count}/{max}`: domain_count + "/" + plan_config.max_domains
- `{time_ago}`: Latest scans.completed_at formatted as relative time
- `{domain_name}`: domain.display_name from latest scan
- `{critical_count}`: COUNT(domains) WHERE last_scan_score < 60
- `{issues_text}`: "Require attention" if > 0, else "All good"

### Security Score Trends Chart (Dynamic)
```
Chart Component
┌─────────────────────────────────────────────────────────────┐
│ Security Score Trends              [Time Period Dropdown ▼] │
│                                                             │
│ 100 │                                                       │
│  80 │ {bars_rendered_from_historical_scan_data}             │
│  60 │                                                       │
│  40 │                                                       │
│  20 │                                                       │
│   0 └─┬─────┬─────┬─────┬─────┬─────┬─────┬──               │
│      {time_periods_based_on_selection}                      │
└─────────────────────────────────────────────────────────────┘
```

**Data Source:** 
```sql
SELECT DATE_TRUNC('day/week/month', created_at) as period,
       AVG(total_score) as avg_score
FROM scans 
WHERE user_id = auth_user 
  AND created_at >= {selected_range_start}
GROUP BY period 
ORDER BY period
```

### Page Layouts (Data-Driven)

#### 1. Profile Settings Page
```
Profile Settings
┌─────────────────────────────────────────────────────────────────────────┐
│ ┌─────────────────┐  ┌─────────────────────────┐  ┌─────────────────────┐ │
│ │   {initials}    │  │ Full Name  Email Address│  │  Account Status     │ │
│ │ [Change Avatar] │  │ {input}    {input}      │  │ Plan: {plan_badge}  │ │
│ └─────────────────┘  │ [Save Changes]          │  │ Status: {status}    │ │
│                      └─────────────────────────┘  │ Member: {date}      │ │
│ ┌──────────────────────────────────────────────┐  └─────────────────────┘ │
│ │            Change Password                   │  ┌─────────────────────┐ │
│ │ Current: {input}                             │  │    Quick Stats      │ │
│ │ New: {input}  Confirm: {input}               │  │ Domains: {count}    │ │
│ │ [Update Password]                            │  │ Scans: {count}      │ │
│ └──────────────────────────────────────────────┘  │ Reports: {count}    │ │
│                                                   └─────────────────────┘ │
│ ┌─────────────────────────────────────────────────────────────────────────┤
│ │                      Danger Zone                                       │ │
│ │ Once you delete your account, there is no going back                   │ │
│ │ [Delete Account]                                                       │ │
│ └─────────────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────┘
```

#### 2. Domains Page
```
Domains                                          [Scan Now] [Add Domain]
┌─────────────────────────────────────────────────────────────────────────┐
│ Domain Name           │ Last Scan Date    │ Security Score │ Actions    │
├─────────────────────────────────────────────────────────────────────────┤
│ {domain.url}          │ {scan.date}       │ {score}{grade} │ 👁🔍🗑    │
│ {domain.display}      │ {time_ago}        │ {color_coded}  │            │
│                       │                   │                │            │
│ [Empty State: No domains yet - Add your first domain]                   │
└─────────────────────────────────────────────────────────────────────────┘
```

#### 3. Activity Page
```
Activity                                             [Generate Report]
┌─────────────────────────────────────────────────────────────────────────┐
│ [All Domains ▼] [Last 30 days ▼]                                       │
│                                                                         │
│ Date & Time       │ Domain Name    │ Scan Type │ Score     │ Actions    │
├─────────────────────────────────────────────────────────────────────────┤
│ {scan.created_at} │ {domain.url}   │ Full Scan │ {score}   │ 👁📊       │
│ {time_ago}        │ {display_name} │           │ {grade}   │            │
│                   │                │           │           │            │
│ [Empty State: No scan activity - Run your first scan]                  │
└─────────────────────────────────────────────────────────────────────────┘
```

#### 4. Reports Page
```
Reports                                          [Download Selected]
┌─────────────────────────────────────────────────────────────────────────┐
│                               📄                                       │
│                                                                         │
│                         No reports yet                                 │
│                                                                         │
│           Generate your first security report to get started           │
│                                                                         │
│                    [Generate Your First Report]                        │
│                                                                         │
│ [Future State with Data]                                                │
│ Report Name        │ Date Generated │ Type │ Status │ Actions          │
│ {report.filename}  │ {created_at}   │ PDF  │ Ready  │ ⬇ 👁 🗑          │
└─────────────────────────────────────────────────────────────────────────┘
```

#### 5. Subscription/Billing Page
```
Billing & Subscription
┌─────────────────────────────────────────────────────────────────────────┐
│ ┌───────────────────────────┐ ┌─────────────────────────────────────────┐ │
│ │      Current Plan         │ │         Usage This Month                │ │
│ │ {plan.name} {trial_badge} │ │ Domains: {used}/{max} {progress_bar}    │ │
│ │ ${plan.price}/month       │ │ Scans: {used}/{limit} {progress_bar}    │ │
│ │ {max_domains} domains     │ │                                         │ │
│ │ [Change] [Cancel]         │ │         Payment Method                  │ │
│ └───────────────────────────┘ │ 💳 **** **** **** {last4}               │ │
│                               │ Expires {month}/{year}                  │ │
│ ┌─────────────────────────────────────────────────────────────────────┐ │ │
│ │                    Billing History                                 │ │ │
│ │ {plan.name} - {month} {year}  [{status}] ${amount}  [⬇]           │ │ │
│ │ {invoice_date}                                                     │ │ │
│ └─────────────────────────────────────────────────────────────────────┘ │ │
│                               │         Need Help?                      │ │
│                               │ [📧 Contact Support]                    │ │
│                               └─────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────┘
```

#### 6. Domain Detail Page
```
{domain.url}                                             [🔄 New Scan]
┌─────────────────────────────────────────────────────────────────────────┐
│ ┌─────────────────┐ ┌─────────────────┐ ┌─────────────────────────────┐ │
│ │ Overall Score   │ │ SSL/TLS Grade   │ │     Scan Coverage           │ │
│ │ {total_score}   │ │ {ssl_grade}     │ │ {completion_icon}           │ │
│ │ /100            │ │ {days_left}     │ │ {coverage_text}             │ │
│ │ {status_badge}  │ │ {protocol_info} │ │ {scan_type}                 │ │
│ └─────────────────┘ └─────────────────┘ └─────────────────────────────┘ │
│                                                                         │
│ ┌─────────────────────────────┐ ┌─────────────────────────────────────┐ │
│ │      Security Headers       │ │          DNS & Network              │ │
│ │ {configured}/{total} config │ │ {dnssec_badge}                      │ │
│ │ • HSTS {status_badge}       │ │ 📧 Email Security                   │ │
│ │ • CSP {status_badge}        │ │ {spf_badge} {dmarc_badge}           │ │
│ │ • X-Frame {status_badge}    │ │ 🔌 Open Ports                       │ │
│ │ • X-Content {status_badge}  │ │ {port_count} detected               │ │
│ │ • Referrer {status_badge}   │ │                                     │ │
│ └─────────────────────────────┘ └─────────────────────────────────────┘ │
│                                                                         │
│ Recommended Actions                                                     │
│ ┌─────────────────────────────────────────────────────────────────────┐ │
│ │ • {priority_badge} {recommendation_text}                           │ │
│ │ • {priority_badge} {recommendation_text}                           │ │
│ │ • {priority_badge} {recommendation_text}                           │ │
│ └─────────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────┘
```

### Complete Data Source Mapping

**Every UI element maps to real database data:**

```php
// Dashboard Data
$dashboardData = [
    'avg_score' => DB::table('domains')
        ->where('user_id', auth()->id())
        ->whereNotNull('last_scan_score')
        ->avg('last_scan_score'),
    'domain_count' => auth()->user()->domains()->where('is_active', true)->count(),
    'max_domains' => config('plans.solo.max_domains'), // 10
    'critical_count' => auth()->user()->domains()->where('last_scan_score', '<', 60)->count(),
    'last_scan' => auth()->user()->scans()->latest()->with('domain')->first(),
    'trial_days_remaining' => auth()->user()->trial_ends_at->diffInDays(now()),
];

// Profile Data
$profileData = [
    'initials' => substr(auth()->user()->name, 0, 1) . substr(explode(' ', auth()->user()->name)[1] ?? '', 0, 1),
    'plan_name' => auth()->user()->subscription_status === 'trialing' ? 'Solo Plan' : 'Active',
    'domain_count' => auth()->user()->domains()->count(),
    'monthly_scans' => auth()->user()->scans()->whereMonth('created_at', now()->month)->count(),
    'reports_count' => auth()->user()->reports()->count(),
];

// Domain Detail Data
$domainData = [
    'total_score' => $domain->latest_scan->total_score ?? null,
    'ssl_grade' => $domain->latest_scan->modules()->where('module', 'ssl_tls')->first()->raw['grade'] ?? 'Unknown',
    'headers_configured' => $domain->latest_scan->modules()->where('module', 'security_headers')->first()->score ?? 0,
    'dnssec_status' => $domain->latest_scan->modules()->where('module', 'dns_email')->first()->raw['dnssec'] ?? false,
];
```

### Trial Banner (Conditional Display)
```
Displays only when: user.trial_ends_at > now()
┌─────────────────────────────────────────────────────────────────────────────────┐
│ 🕒 {plan.name} Trial  {days_remaining} days remaining  {domain_count} of {max_domains} domains  [Upgrade Now] [✕] │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### Empty States & Error Handling

**Every page handles missing data appropriately:**

```php
// Dashboard Empty States
if ($domainCount === 0) {
    return view('dashboard')->with('emptyState', [
        'title' => 'No domains yet',
        'message' => 'Add your first domain to start monitoring',
        'action' => 'Add Domain',
        'route' => route('domains.create')
    ]);
}

// Activity Empty State
if ($scanCount === 0) {
    return view('activity.index')->with('emptyState', [
        'title' => 'No scan activity yet',
        'message' => 'Run your first scan to see results here',
        'action' => 'Scan Now',
        'route' => route('scans.create')
    ]);
}

// Reports Empty State
if ($reportCount === 0) {
    return view('reports.index')->with('emptyState', [
        'title' => 'No reports yet',
        'message' => 'Generate your first security report to get started',
        'action' => 'Generate Report',
        'route' => route('reports.create')
    ]);
}
```

### Domains Page Layout

#### Header Section
- Page title and description
- "Add Domain" and "Scan Now" buttons (top right)
- Tab filters: "Your Domains" / "All Clients" (note: no clients in MVP)

#### Domains Table
```
┌──────────────────────────────────────────────────────────────────┐
│ CLIENT NAME | DOMAIN NAME           | LAST SCAN DATE | SCORE | ACTIONS │
├──────────────────────────────────────────────────────────────────┤
│ Richard     | https://hackerscope.ai| Aug 12, 02:33PM|   85  | 👁 🔍 🗑  │
└──────────────────────────────────────────────────────────────────┘
```
- Client name (note: remove for MVP)
- Full domain URL with favicon
- Last scan timestamp
- Score with color coding
- Action buttons: View, Scan, Delete

### Activity Page Layout

#### Filters Section
- "All Domains" dropdown
- Date range picker (Last 30 days)
- "Generate Report" button (top right)

#### Activity Table
- Date & Time column
- Domain name with client info
- Scan type badge (Full Scan)
- Security score with color
- Actions (View, Generate)

### Reports Page Layout
- Empty state: "No reports yet"
- "Generate Your First Report" CTA button
- Future: Table with report name, date, type, status, actions

### Domain Detail Page Layout

#### Header Section
- Domain URL as title
- "New Scan" button
- Breadcrumb navigation

#### Score Overview Cards
- Overall Score (large number with grade)
- SSL/TLS Grade (A+, with days left)
- Scan Coverage indicator

#### Security Modules Grid

##### Security Headers Module
- 5/5 configured indicator
- List of headers with status badges:
  - HSTS ✓ Set
  - CSP ✓ Set  
  - X-Frame-Options ✓ Set
  - X-Content-Type ✓ Set
  - Referrer-Policy ✓ Set

##### DNS & Network Module
- DNSSEC Enabled badge
- Email Security indicators (SPF, DMARC badges)
- Open Ports: "0 detected"

#### Recommended Actions Section
- List of prioritized security improvements
- How-to guidance for each recommendation
- Color-coded by severity

### Common UI Patterns

#### Buttons
- **Primary**: Blue background (#3b82f6), white text
- **Secondary**: Dark border, white text
- **Danger**: Red background (#ef4444), white text
- **Success**: Green background (#22c55e), white text

#### Form Fields
- Dark background with light border
- Focus state with blue outline
- Error states with red border

#### Status Badges
- **Success**: Green background, white text
- **Warning**: Orange background, white text
- **Error**: Red background, white text
- **Info**: Blue background, white text

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

### Trial Strategy
- **14 days**: Optimal for technical evaluation
- **No restrictions**: Full features during trial
- **No card required**: Reduces friction
- **Clear value**: Immediate scan results

## Business Model

### Revenue Model
- **SaaS subscription**: Monthly recurring revenue
- **Single tier**: Simplicity over complexity
- **Direct billing**: Stripe integration (no Cashier)
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

## Conclusion

Achilleus addresses a clear market need: affordable security monitoring for developers managing multiple websites. By focusing on essential security checks, maintaining simplicity, and offering revolutionary pricing, Achilleus can capture significant market share in the underserved freelancer and small business segments.

The product is designed to be:
- **Affordable**: $27/month for 10 domains
- **Reliable**: 8/10 security coverage with low false positives
- **Fast**: 15-second scans
- **Simple**: One scan type, clear scoring
- **Professional**: PDF reports for documentation

This combination creates a compelling value proposition that competitors cannot match without destroying their revenue models.