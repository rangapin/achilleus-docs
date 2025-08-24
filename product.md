# Achilleus Product Documentation

## Executive Summary

**Product Name**: Achilleus  
**Tagline**: Security monitoring for developers managing multiple websites  
**Target Market**: Freelancers, agencies, and small businesses  
**Price Point**: $27/month for 10 domains with unlimited scans  
**Core Value**: Affordable, automated security monitoring without enterprise complexity  

---

## Problem Statement

### The Problem
Developers and small businesses managing multiple websites face significant security challenges:
- **No Visibility**: Security vulnerabilities remain hidden until exploited
- **Enterprise Tools**: Existing solutions are complex and expensive ($200+/month)
- **Manual Checks**: Time-consuming to verify SSL, headers, DNS across domains
- **Client Trust**: Hard to demonstrate security compliance to clients
- **Alert Fatigue**: Too many notifications from complex tools

### Our Solution
Achilleus provides simple, automated security monitoring that:
- Scans all domains in under 30 seconds
- Presents results in a clear, actionable format
- Costs 85% less than enterprise alternatives
- Requires zero configuration or security expertise
- Generates professional reports for clients

---

## Target Audience

### Primary Persona: Freelance Developer
- **Demographics**: 25-40 years old, technical background
- **Manages**: 5-15 client websites
- **Pain Points**: 
  - Manually checking SSL expiry dates
  - Explaining security to non-technical clients
  - Justifying security maintenance fees
- **Goals**: 
  - Automate security monitoring
  - Professional client reporting
  - Affordable monthly cost

### Secondary Persona: Small Agency Owner
- **Demographics**: 30-45 years old, business-focused
- **Manages**: 20-50 client websites
- **Pain Points**:
  - Team needs security visibility
  - Compliance requirements from clients
  - Cost of enterprise tools
- **Goals**:
  - Centralized security dashboard
  - Delegate monitoring to team
  - White-label reports for clients

### Tertiary Persona: Small Business Owner
- **Demographics**: 35-55 years old, non-technical
- **Manages**: 1-3 business websites
- **Pain Points**:
  - Doesn't understand security
  - Worried about breaches
  - Can't afford security consultants
- **Goals**:
  - Peace of mind
  - Simple explanations
  - Automated protection

---

## Core Features

### 1. Domain Management
**Purpose**: Central hub for all monitored websites

**Functionality**:
- Add domains with HTTPS validation
- Configure email security settings (SPF/DKIM/DMARC)
- Set DKIM selector for accurate scanning
- Mark domains as non-email sending
- View last scan status at a glance

**User Flow**:
1. User clicks "Add Domain"
2. Enters clean domain name (e.g., "example.com") - system shows "→ https://example.com" preview
3. Selects email configuration
4. Optional: Enters DKIM selector
5. Domain added and immediately scanned
6. Results appear in dashboard

**Business Rules**:
- Maximum 10 domains per account
- Clean domain input (e.g., "example.com")
- Automatic HTTPS prepending for security scanning
- No duplicate domains per user
- URL normalization (remove www, protocol, trailing slash, paths)

### 2. Security Scanning
**Purpose**: Automated vulnerability detection

**Three Scanner Modules**:

#### SSL/TLS Scanner (40% weight)
- Certificate expiry monitoring
- Protocol version checking (TLS 1.2+)
- Cipher strength analysis
- Certificate chain validation
- Hostname verification

#### Security Headers Scanner (30% weight)
- HSTS (Strict Transport Security)
- CSP (Content Security Policy)
- X-Frame-Options
- X-Content-Type-Options
- Referrer-Policy

#### DNS/Email Scanner (30% weight)
- SPF record validation
- DKIM key verification
- DMARC policy checking
- DNSSEC status
- CAA record detection

**Scoring System**:
- 0-100 point scale
- Letter grades: A+ (95+), A (90-94), B+ (85-89), B (80-84), C (70-79), D (60-69), F (0-59)
- Weighted average across modules
- Automatic weight redistribution if scanner fails

### 3. Dashboard
**Purpose**: Quick security overview

**Components**:
- **Overall Score Card**: Average across all domains
- **Domains Card**: Active/total count
- **Last Scan Card**: Most recent scan with domain name
- **Issues Card**: Count of critical problems
- **Trend Chart**: Score history over time (7d/30d/90d/1y)

**Real-time Updates**:
- WebSocket connection for live scan progress
- Instant score updates
- Toast notifications for completions

### 4. Report Generation
**Purpose**: Professional PDF reports for clients

**Report Contents**:
- Executive summary with overall grade
- Module-by-module breakdown
- Specific issues found
- Actionable recommendations
- Scan timestamp and domain details

**Delivery**:
- One-click PDF generation
- Secure S3 storage
- Shareable download links (1-hour expiry)
- Email delivery option

### 5. User Management
**Purpose**: Account and subscription handling

**Authentication Options**:
- Email/password registration
- Google OAuth login
- Password reset flow

**Trial System**:
- 14-day free trial for all new users
- Full feature access during trial
- Automatic conversion to paid

### 6. Email Notifications
**Purpose**: Proactive security monitoring alerts

**Notification Types**:
- **Certificate Expiry Warnings**: 30, 7, and 1 day before SSL certificate expiration
- **Trial Expiry Warnings**: 7, 3, and 1 day before trial period ends
- **Scan Completion**: Summary email when domain scans finish

**Features**:
- Responsive email templates
- One-click unsubscribe functionality
- Notification preferences management
- Queued delivery for reliability

### 7. Billing Management
**Purpose**: Simple subscription handling

**Subscription Model**:
- $27/month flat rate
- 10 domains included
- Unlimited scans
- All features included

**Payment Features**:
- Stripe integration
- Credit card management
- Invoice history
- Subscription cancellation
- Payment method updates

---

## User Journeys

### Journey 1: First-Time User Setup
1. **Discover**: User finds Achilleus via search/recommendation
2. **Sign Up**: Creates account with email or OAuth
3. **Trial Starts**: 14-day countdown begins
4. **Add First Domain**: Types "mywebsite.com" (no https:// needed)
5. **Domain Preview**: System shows "→ https://mywebsite.com" 
6. **Initial Scan**: Automatic scan shows security status
7. **Explore Results**: Views detailed findings
8. **Add More Domains**: Adds client websites using simple domain names
9. **Generate Report**: Creates first PDF report
10. **Convert to Paid**: Subscribes before trial ends

### Journey 2: Daily Monitoring Workflow
1. **Morning Check**: Opens dashboard
2. **Review Scores**: Checks overall security status
3. **Investigate Issues**: Clicks through to problems
4. **Fix Problems**: Takes action based on recommendations
5. **Rescan**: Triggers new scan to verify fixes
6. **Report Success**: Generates report for client

### Journey 3: Client Reporting Process
1. **Monthly Review**: End of month arrives
2. **Scan All**: Triggers scans for all domains
3. **Generate Reports**: Creates PDFs for each client
4. **Download Reports**: Gets secure links
5. **Send to Clients**: Emails reports with explanations
6. **Justify Fees**: Uses reports to show value

### Journey 4: SSL Certificate Expiry
1. **Warning Email**: Receives 30-day expiry notice
2. **Dashboard Alert**: Sees warning in dashboard
3. **View Details**: Checks certificate information
4. **Renew Certificate**: Updates SSL on server
5. **Rescan**: Verifies renewal successful
6. **Clear Alert**: Warning removed from dashboard

---

## Business Model

### Pricing Strategy
- **Single Tier**: $27/month (simple, no confusion)
- **No Setup Fees**: Immediate value
- **No Per-Scan Costs**: Unlimited usage
- **Annual Option**: Consider 20% discount for yearly

### Revenue Projections
- **Target**: 1,000 customers in Year 1
- **MRR Goal**: $27,000/month by Month 12
- **Churn Target**: <5% monthly
- **LTV**: $540 (20-month average retention)

### Cost Structure
- **Infrastructure**: ~$500/month (Laravel Cloud)
- **Payment Processing**: 2.9% + $0.30 (Stripe)
- **Email Service**: ~$100/month (Resend)
- **Domain**: $15/year
- **Break-even**: ~25 customers

---

## Competitive Analysis

### Direct Competitors

#### Sucuri ($199/month)
- **Strengths**: CDN included, malware scanning
- **Weaknesses**: Expensive, complex, enterprise-focused
- **Our Advantage**: 85% cheaper, simpler interface

#### Detectify ($170/month)
- **Strengths**: Penetration testing, vulnerability scanning
- **Weaknesses**: Requires security knowledge, expensive
- **Our Advantage**: No expertise required, affordable

#### SSL Labs (Free)
- **Strengths**: Free, detailed SSL analysis
- **Weaknesses**: SSL only, no monitoring, manual checks
- **Our Advantage**: Complete security, automated monitoring

### Indirect Competitors
- Uptime monitoring tools (Pingdom, UptimeRobot)
- Security plugins (Wordfence, Sucuri plugin)
- Manual security audits

### Competitive Advantages
1. **Price Point**: 85% cheaper than alternatives
2. **Simplicity**: No security expertise required
3. **Speed**: 30-second comprehensive scans
4. **Focus**: Built specifically for developers
5. **Reports**: Professional PDFs for clients

---

## Success Metrics

### User Acquisition
- **Sign-ups**: 100/month target
- **Trial-to-Paid**: 15% conversion target
- **Referral Rate**: 20% from existing users
- **CAC**: <$50 per customer

### User Engagement
- **Domains per User**: Average 5-7
- **Scans per Month**: Average 20 per user
- **Reports Generated**: 3 per user per month
- **Dashboard Visits**: Weekly active use

### Business Health
- **MRR Growth**: 20% month-over-month
- **Churn Rate**: <5% monthly
- **NPS Score**: >50
- **Support Tickets**: <2 per customer per year

---

## Feature Roadmap

### Phase 1: MVP (Days 1-21)
✅ Core scanning (SSL, Headers, DNS)  
✅ Dashboard with metrics  
✅ Domain management  
✅ PDF reports  
✅ Stripe payments  
✅ OAuth login  

### Phase 2: Enhancement (Months 2-3)
- Email notifications for issues
- Scheduled automatic scans
- Bulk domain import
- White-label reports
- API access

### Phase 3: Growth (Months 4-6)
- WordPress plugin
- Uptime monitoring
- Performance metrics
- Team accounts
- Slack integration

### Phase 4: Scale (Months 7-12)
- Mobile app
- Advanced vulnerability scanning
- Compliance reports (GDPR, PCI)
- Partner program
- Enterprise tier

---

## Marketing Strategy

### Positioning Statement
"Achilleus is the security monitoring tool that helps developers protect multiple websites without the complexity or cost of enterprise solutions."

### Key Messages
1. **Simple**: "Security monitoring in plain English"
2. **Affordable**: "Enterprise features at freelancer prices"
3. **Fast**: "Complete security scan in 30 seconds"
4. **Professional**: "Client-ready reports in one click"

### Distribution Channels
1. **Content Marketing**: Security guides and tutorials
2. **Developer Communities**: Reddit, Dev.to, Hacker News
3. **Freelance Platforms**: Upwork, Fiverr profiles
4. **Partner Program**: Referral commissions
5. **SEO**: Target "website security monitoring" keywords

### Launch Strategy
1. **Soft Launch**: 50 beta users for feedback
2. **Product Hunt**: Launch when stable
3. **Developer Forums**: Share in relevant communities
4. **Case Studies**: Document customer success
5. **Influencer Outreach**: Developer YouTube channels

---

## Risk Mitigation

### Technical Risks
- **Scanner Failures**: Implement retry logic and fallbacks
- **False Positives**: Regular scanner calibration
- **Performance Issues**: Queue system for scans
- **Data Loss**: Regular backups to S3

### Business Risks
- **Low Conversion**: A/B test onboarding flow
- **High Churn**: Exit surveys and improvements
- **Competition**: Focus on simplicity differentiator
- **Support Burden**: Comprehensive documentation

### Security Risks
- **SSRF Attacks**: NetworkGuard validation
- **Data Breaches**: Encryption at rest and transit
- **Payment Fraud**: Stripe Radar protection
- **DDoS**: CloudFlare protection

---

## Support Strategy

### Self-Service
- Comprehensive documentation
- Video tutorials
- FAQ section
- In-app tooltips

### Assisted Support
- Email support (24-hour response)
- Priority support for paid users
- Community forum
- Monthly webinars

### Success Resources
- Getting started guide
- Security best practices
- Client communication templates
- Report interpretation guide

---

## Legal Considerations

### Terms of Service
- Scanning own domains only
- No guarantee of finding all vulnerabilities
- Limitation of liability
- Data retention policy

### Privacy Policy
- GDPR compliant
- Data minimization
- User data ownership
- Deletion rights

### Compliance
- PCI DSS for payments
- SOC 2 consideration for enterprise
- GDPR for EU customers
- CCPA for California users

---

## Success Definition

### Year 1 Goals
- 1,000 paying customers
- $27,000 MRR
- <5% monthly churn
- 50+ NPS score
- Break-even by Month 6

### Long-term Vision
Become the default security monitoring tool for developers and agencies worldwide, making website security accessible and affordable for everyone.

---

## Conclusion

Achilleus addresses a clear market need for simple, affordable security monitoring. By focusing on developers and small businesses, we can capture a underserved market segment and build a sustainable SaaS business with strong unit economics and growth potential.