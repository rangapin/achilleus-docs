# Achilleus Database Architecture

## Overview
PostgreSQL 15 database managed by Laravel Cloud with automatic hibernation and scaling. All tables use UUID primary keys for better distribution in clustered environments.

**Performance Target**: Dashboard loads in <200ms (upgraded from 500ms for competitive advantage)

## Phase 2 Development (Days 3-4)
- **Test Allocation**: 25 tests covering models, relationships, business rules
- **Migration Order**: users → domains → scans → scan_modules → reports → indexes
- **Key Validations**: 10 domain limit, HTTPS-only URLs, trial period calculations

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
    provider VARCHAR(20), -- 'google', or NULL for email
    provider_id VARCHAR(255), -- OAuth provider's user ID
    
    -- Subscription fields (Laravel Cashier)
    trial_ends_at TIMESTAMP NOT NULL DEFAULT (CURRENT_TIMESTAMP + INTERVAL '14 days'),
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
    UNIQUE(user_id, url),
    -- Business rule: 10 domain limit enforced
    CONSTRAINT check_domain_limit CHECK ((SELECT COUNT(*) FROM domains WHERE user_id = domains.user_id) <= 10)
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

-- Error logging table for monitoring and analytics
CREATE TABLE error_logs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    type VARCHAR(50) NOT NULL, -- 'scanner_error', 'client_error', 'application_error'
    severity VARCHAR(20) NOT NULL CHECK (severity IN ('debug', 'info', 'warning', 'error', 'critical')),
    message TEXT NOT NULL,
    context JSONB,
    
    user_id UUID,
    url TEXT,
    ip INET,
    user_agent TEXT,
    
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE SET NULL
);

-- Indexes for error analytics
CREATE INDEX idx_error_logs_type_severity ON error_logs(type, severity);
CREATE INDEX idx_error_logs_created_at ON error_logs(created_at);
CREATE INDEX idx_error_logs_user_id ON error_logs(user_id) WHERE user_id IS NOT NULL;
```

### scan_modules
Individual scanner results with raw data storage.

```sql
CREATE TABLE scan_modules (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    scan_id UUID REFERENCES scans(id) ON DELETE CASCADE,
    module VARCHAR(20) NOT NULL CHECK (module IN ('ssl_tls', 'security_headers', 'dns_email')),
    score INTEGER CHECK (score >= 0 AND score <= 100),
    status VARCHAR(10) NOT NULL CHECK (status IN ('ok', 'warn', 'fail', 'error')),
    confidence VARCHAR(10) DEFAULT 'high' CHECK (confidence IN ('high', 'medium', 'low')),
    platform VARCHAR(50), -- detected platform (cloudflare, github-pages, etc)
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

## Performance Indexes

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
CREATE INDEX idx_scan_modules_raw_gin ON scan_modules USING GIN(raw);

-- Report queries
CREATE INDEX idx_reports_user ON reports(user_id, generated_at DESC);
CREATE INDEX idx_reports_scan ON reports(scan_id);
```

## Model Relationships

```php
// User Model
public function domains(): HasMany
{
    return $this->hasMany(Domain::class);
}

public function scans(): HasMany
{
    return $this->hasMany(Scan::class);
}

// Domain Model
public function user(): BelongsTo
{
    return $this->belongsTo(User::class);
}

public function scans(): HasMany
{
    return $this->hasMany(Scan::class)->latest();
}

public function latestScan(): HasOne
{
    return $this->hasOne(Scan::class)->latestOfMany();
}

// Scan Model
public function domain(): BelongsTo
{
    return $this->belongsTo(Domain::class);
}

public function modules(): HasMany
{
    return $this->hasMany(ScanModule::class);
}

public function report(): HasOne
{
    return $this->hasOne(Report::class);
}
```

## Performance Optimization for <200ms

### Dashboard Query (Target: <100ms)
```sql
-- Single optimized query for dashboard metrics
SELECT 
    AVG(last_scan_score)::INTEGER as avg_score,
    COUNT(*) as total_domains,
    COUNT(CASE WHEN last_scan_score < 60 THEN 1 END) as critical_issues
FROM domains 
WHERE user_id = ? AND is_active = true;
```

### Caching Strategy
```php
// Dashboard metrics (5 minute cache)
Cache::remember("dashboard.{$userId}", 300, function() {
    return [
        'avg_score' => $user->domains()->avg('last_scan_score'),
        'domain_count' => $user->domains()->count(),
        'last_scan' => $user->scans()->latest()->first()
    ];
});
```

### Laravel Cloud Configuration
```env
DB_POOL_SIZE=10
DB_STATEMENT_TIMEOUT=30000
CACHE_DRIVER=redis
QUEUE_CONNECTION=redis
```

## Data Retention & Migration Strategy

### Retention Policies
- **Scans**: 90 days (configurable)
- **Reports**: 1 year
- **User data**: Until account deletion

### Deployment Commands
```bash
# Production deployment
php artisan migrate --force
php artisan config:cache
php artisan route:cache
```