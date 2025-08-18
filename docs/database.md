# Achilleus Database Schema

## Schema Overview
```sql
-- Users table with billing and support
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    password VARCHAR(255) NOT NULL,
    email_verified_at TIMESTAMP,
    trial_ends_at TIMESTAMP NOT NULL,
    stripe_customer_id VARCHAR(255),
    stripe_subscription_id VARCHAR(255),
    subscription_status VARCHAR(50) DEFAULT 'trialing',
    notification_preferences JSONB DEFAULT '{}',
    unsubscribe_token VARCHAR(255),
    privacy_preferences JSONB DEFAULT '{}',
    onboarding_progress JSONB DEFAULT '{}',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Domains table
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

-- Scans table
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

-- Scan modules table
CREATE TABLE scan_modules (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    scan_id UUID REFERENCES scans(id) ON DELETE CASCADE,
    module VARCHAR(20) NOT NULL CHECK (module IN ('security_headers', 'ssl_tls', 'dns_email')),
    score INTEGER CHECK (score >= 0 AND score <= 100),
    status VARCHAR(10) NOT NULL CHECK (status IN ('ok', 'warn', 'fail', 'error')),
    raw JSONB,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(scan_id, module)
);

-- Reports table
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
CREATE INDEX idx_users_trial ON users(trial_ends_at, subscription_status);
CREATE INDEX idx_domains_user ON domains(user_id, is_active);
CREATE INDEX idx_domains_last_scan ON domains(last_scan_at DESC);
CREATE INDEX idx_scans_domain ON scans(domain_id, created_at DESC);
CREATE INDEX idx_scans_user ON scans(user_id, created_at DESC);
CREATE INDEX idx_scans_status ON scans(status) WHERE status IN ('pending', 'running');
```

## Support System Tables
```sql
CREATE TABLE support_tickets (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id) ON DELETE SET NULL,
    name VARCHAR(255) NOT NULL,
    email VARCHAR(255) NOT NULL,
    category VARCHAR(20) NOT NULL CHECK (category IN ('billing', 'technical', 'feature', 'other')),
    priority VARCHAR(10) NOT NULL CHECK (priority IN ('low', 'medium', 'high', 'urgent')),
    subject VARCHAR(255) NOT NULL,
    message TEXT NOT NULL,
    status VARCHAR(20) DEFAULT 'open' CHECK (status IN ('open', 'pending', 'resolved', 'closed')),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

## Legal Compliance Tables
```sql
CREATE TABLE legal_documents (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    type VARCHAR(50) NOT NULL CHECK (type IN ('terms_of_service', 'privacy_policy')),
    version VARCHAR(20) NOT NULL,
    content TEXT NOT NULL,
    effective_date DATE NOT NULL,
    is_active BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE user_legal_acceptances (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    document_type VARCHAR(50) NOT NULL,
    document_version VARCHAR(20) NOT NULL,
    accepted_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    ip_address INET,
    user_agent TEXT,
    UNIQUE(user_id, document_type, document_version)
);
```

## Onboarding Tables
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

## Key Relationships
- Users have many Domains (max 10 per trial/subscription)
- Domains have many Scans (historical tracking)
- Scans have many ScanModules (SSL, Headers, DNS results)
- Scans can have many Reports (PDF generation)
- Users have many SupportTickets (customer service)
- Users have many UserAchievements (onboarding gamification)