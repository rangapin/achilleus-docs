# Achilleus Database Schema

## Overview
PostgreSQL 15 database managed by Laravel Cloud with automatic hibernation and scaling.

## Core Tables

### users
Primary user table with authentication, billing, and OAuth support.

```sql
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    password VARCHAR(255), -- Nullable for OAuth users
    email_verified_at TIMESTAMP,
    
    -- OAuth fields (Laravel Socialite)
    provider VARCHAR(20), -- 'github', 'google', or NULL for email
    provider_id VARCHAR(255), -- OAuth provider's user ID
    
    -- Subscription fields (Laravel Cashier)
    trial_ends_at TIMESTAMP NOT NULL,
    stripe_customer_id VARCHAR(255),
    stripe_subscription_id VARCHAR(255),
    stripe_price_id VARCHAR(255), -- For tracking plan type
    subscription_status VARCHAR(50) DEFAULT 'trialing',
    subscription_ends_at TIMESTAMP, -- For cancelled subscriptions
    
    
    notification_preferences JSONB DEFAULT '{}',
    unsubscribe_token VARCHAR(255),
    onboarding_progress JSONB DEFAULT '{}',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### domains
User domains for security monitoring.

```sql
CREATE TABLE domains (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    url VARCHAR(255) NOT NULL,
    display_name VARCHAR(255) NOT NULL,
    email_mode VARCHAR(10) DEFAULT 'expected' CHECK (email_mode IN ('expected', 'none')),
    dkim_selector VARCHAR(50),
    last_scan_at TIMESTAMP,
    last_scan_score INTEGER CHECK (last_scan_score >= 0 AND last_scan_score <= 100),
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(user_id, url)
);
```

### scans
Security scan records with scoring.

```sql
CREATE TABLE scans (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    domain_id UUID REFERENCES domains(id) ON DELETE CASCADE,
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    status VARCHAR(20) NOT NULL CHECK (status IN ('pending', 'running', 'completed', 'failed')),
    total_score INTEGER CHECK (total_score >= 0 AND total_score <= 100),
    grade VARCHAR(3),
    scoring_version VARCHAR(10),
    weights_used JSONB,
    started_at TIMESTAMP,
    completed_at TIMESTAMP,
    error_message TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### scan_modules
Individual scanner results with raw data storage.

```sql
CREATE TABLE scan_modules (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    scan_id UUID REFERENCES scans(id) ON DELETE CASCADE,
    module VARCHAR(20) NOT NULL CHECK (module IN ('security_headers', 'ssl_tls', 'dns_email')),
    score INTEGER CHECK (score >= 0 AND score <= 100),
    status VARCHAR(10) NOT NULL CHECK (status IN ('ok', 'warn', 'fail', 'error')),
    raw JSONB NOT NULL DEFAULT '{}',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(scan_id, module)
);
```

### reports
Generated PDF security reports.

```sql
CREATE TABLE reports (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    scan_id UUID REFERENCES scans(id) ON DELETE CASCADE,
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    filename VARCHAR(255) NOT NULL,
    s3_path VARCHAR(500) NOT NULL,
    file_size INTEGER,
    generated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

## Laravel Package Tables

### subscription_items
Laravel Cashier subscription items for metered billing.

```sql
CREATE TABLE subscription_items (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    subscription_id VARCHAR(255) NOT NULL,
    stripe_id VARCHAR(255) UNIQUE NOT NULL,
    stripe_product VARCHAR(255) NOT NULL,
    stripe_price VARCHAR(255) NOT NULL,
    quantity INTEGER DEFAULT 1,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### features
Additional user preferences.

```sql
CREATE TABLE features (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    scope VARCHAR(255) NOT NULL, -- 'User:123' or 'global'
    value TEXT NOT NULL, -- Serialized value
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(name, scope)
);
```

## Legal & Compliance Tables

### user_legal_acceptances
Terms acceptance tracking for compliance.

```sql
CREATE TABLE user_legal_acceptances (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    terms_accepted_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    ip_address INET,
    user_agent TEXT,
    UNIQUE(user_id)
);
```

### user_achievements
Gamification and onboarding progress tracking.

```sql
CREATE TABLE user_achievements (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    achievement_id VARCHAR(50) NOT NULL,
    unlocked_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    metadata JSONB DEFAULT '{}',
    UNIQUE(user_id, achievement_id)
);
```

## Indexes

Performance-optimized indexes for common queries.

```sql
-- User queries
CREATE INDEX idx_users_trial ON users(trial_ends_at, subscription_status);
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_provider ON users(provider, provider_id) WHERE provider IS NOT NULL;

-- Domain queries
CREATE INDEX idx_domains_user ON domains(user_id, is_active);
CREATE INDEX idx_domains_last_scan ON domains(last_scan_at DESC);
CREATE INDEX idx_domains_score ON domains(last_scan_score) WHERE last_scan_score IS NOT NULL;

-- Scan queries
CREATE INDEX idx_scans_domain ON scans(domain_id, created_at DESC);
CREATE INDEX idx_scans_user ON scans(user_id, created_at DESC);
CREATE INDEX idx_scans_status ON scans(status) WHERE status IN ('pending', 'running');
CREATE INDEX idx_scans_completed ON scans(completed_at DESC) WHERE status = 'completed';

-- Module queries
CREATE INDEX idx_scan_modules_scan ON scan_modules(scan_id);
CREATE INDEX idx_scan_modules_type ON scan_modules(module);

-- Report queries
CREATE INDEX idx_reports_user ON reports(user_id, generated_at DESC);
CREATE INDEX idx_reports_scan ON reports(scan_id);
```

## Model Relationships

### Eloquent Relationships

```
User Model:
- hasMany: domains, scans, reports
- hasOne: subscription (via Cashier)

Domain Model:
- belongsTo: user
- hasMany: scans
- hasOne: latestScan

Scan Model:
- belongsTo: domain, user
- hasMany: modules
- hasOne: report

ScanModule Model:
- belongsTo: scan

Report Model:
- belongsTo: scan, user
```

## JSONB Data Structures

### scan_modules.raw Structure

```json
{
  "ssl_tls": {
    "certificate": {
      "issuer": "Let's Encrypt",
      "subject": "example.com",
      "validFrom": "2024-01-01T00:00:00Z",
      "validTo": "2024-12-31T23:59:59Z",
      "daysRemaining": 90,
      "san": ["example.com", "www.example.com"]
    },
    "protocol": {
      "version": "TLSv1.3",
      "cipher": "TLS_AES_256_GCM_SHA384",
      "keyExchange": "ECDHE",
      "keySize": 256
    },
    "vulnerabilities": [],
    "grade": "A+"
  },
  
  "security_headers": {
    "headers": {
      "strict-transport-security": "max-age=31536000; includeSubDomains",
      "content-security-policy": "default-src 'self'",
      "x-frame-options": "DENY",
      "x-content-type-options": "nosniff",
      "referrer-policy": "strict-origin-when-cross-origin"
    },
    "missing": [],
    "warnings": []
  },
  
  "dns_email": {
    "spf": {
      "record": "v=spf1 include:_spf.google.com ~all",
      "valid": true
    },
    "dkim": {
      "selector": "google",
      "valid": true,
      "publicKey": "..."
    },
    "dmarc": {
      "record": "v=DMARC1; p=reject; rua=mailto:dmarc@example.com",
      "policy": "reject",
      "valid": true
    },
    "dnssec": true,
    "caa": {
      "records": ["0 issue \"letsencrypt.org\""],
      "valid": true
    }
  }
}
```

## Data Retention Policies

- **Scans**: 90 days (configurable)
- **Reports**: 1 year
- **Audit logs**: 2 years
- **User data**: Until account deletion
- **Failed jobs**: 7 days

## Migration Notes

1. Run Laravel's default migrations first
2. Run Cashier migrations: `php artisan migrate:cashier`
3. Run application migrations in order
5. Seed test data only in development

## Performance Considerations

- Use UUID for better distribution in clustered environments
- JSONB for flexible scanner data without schema changes
- Partial indexes for active record queries
- Consider partitioning scans table by created_at for large datasets
- Enable pg_stat_statements for query optimization

## Related Documentation

- **Technical Architecture**: See `/docs/technical.md` for system design and scanner implementation
- **Product Specifications**: See `/docs/product.md` for business requirements
- **UI/UX Design**: See `/docs/design.md` for interface specifications
- **Testing Strategy**: See `/docs/testing.md` for database testing patterns
- **Development Workflow**: See `/docs/execution.md` for migration and setup instructions