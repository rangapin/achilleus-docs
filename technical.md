# Achilleus Technical Documentation

## System Architecture

### Technology Stack
- **Backend**: Laravel 12 with PHP 8.3+ (https://laravel.com/docs/12.x)
- **Frontend**: React 19 with TypeScript and Inertia.js (https://react.dev/)
- **UI Components**: Shadcn/ui with dark theme support (https://ui.shadcn.com/)
- **Database**: Serverless PostgreSQL 15 (Laravel Cloud managed)
- **Cache**: Redis-compatible key-value store (auto-scaling)
- **Queue**: Laravel Cloud Queue Workers (auto-scaling, no manual configuration)
- **Storage**: S3-compatible object storage for PDF reports
- **CDN**: Laravel Cloud Edge Network with global CDN
- **Hosting**: Laravel Cloud only (https://cloud.laravel.com/)
- **Real-time**: Laravel Reverb for WebSocket connections (https://reverb.laravel.com/)

### Laravel Ecosystem Packages
- **Laravel Cashier**: Stripe subscription billing for $27/month plans
- **Laravel Pennant**: Feature flags for gradual rollouts and A/B testing
- **Laravel Precognition**: Live form validation without duplicating rules
- **Laravel Socialite**: OAuth authentication (GitHub, Google)
- **Laravel Boost**: AI-powered development assistant with MCP server

### Application Architecture
```
┌─────────────────┐     ┌──────────────┐     ┌─────────────┐
│   React 19      │────▶│  Inertia.js  │────▶│  Laravel 12 │
│  + Shadcn/ui    │     │   (SSR)      │     │ Controllers │
│  + TypeScript   │     │              │     │             │
└─────────────────┘     └──────────────┘     └─────────────┘
                                                     │
                        ┌────────────────────────────┴────────────────┐
                        ▼                                             ▼
                ┌──────────────┐                              ┌──────────────┐
                │   Services   │                              │ Laravel Cloud│
                │  (Business   │                              │ Queue Workers│
                │   Logic)     │                              │ (Managed)    │
                └──────────────┘                              └──────────────┘
                        │                                             │
                        ▼                                             ▼
        ┌──────────────────────────────┐                     ┌──────────────┐
        │    Laravel Cloud Services    │                     │   External   │
        │  • Serverless PostgreSQL    │                     │   Scanners   │
        │  • Redis Cache              │                     │   (HTTPS)    │
        │  • S3 Storage              │                     │              │
        │  • Edge CDN                │                     │              │
        └──────────────────────────────┘                     └──────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                    Laravel Boost MCP Server                      │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐       │
│  │  Tinker  │  │ Database │  │   Docs   │  │  Routes  │       │
│  │ Executor │  │  Query   │  │  Search  │  │ Inspector│       │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘       │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐       │
│  │   Logs   │  │  Schema  │  │  Config  │  │   Tests  │       │
│  │  Reader  │  │ Inspector│  │  Reader  │  │ Generator│       │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘       │
└─────────────────────────────────────────────────────────────────┘
         ▲                                             ▲
         │                                             │
    ┌─────────┐                                 ┌─────────┐
    │ Claude  │                                 │ Cursor  │
    │  Code   │                                 │   IDE   │
    └─────────┘                                 └─────────┘
```

## Database Design

### Schema Overview
```sql
-- Users table with billing and OAuth support
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
    
    -- Feature flags (Laravel Pennant)
    features JSONB DEFAULT '{}', -- Stores active feature flags
    
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
    raw JSONB NOT NULL DEFAULT '{}',
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

-- Indexes for performance
CREATE INDEX idx_users_trial ON users(trial_ends_at, subscription_status);
CREATE INDEX idx_domains_user ON domains(user_id, is_active);
CREATE INDEX idx_domains_last_scan ON domains(last_scan_at DESC);
CREATE INDEX idx_scans_domain ON scans(domain_id, created_at DESC);
CREATE INDEX idx_scans_user ON scans(user_id, created_at DESC);
CREATE INDEX idx_scans_status ON scans(status) WHERE status IN ('pending', 'running');

-- Support is handled via contact email (security@achilleus.so)
-- No support tables needed for MVP

-- Simplified legal (MVP: Terms acceptance on signup only)
CREATE TABLE user_legal_acceptances (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    terms_accepted_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    ip_address INET,
    user_agent TEXT,
    UNIQUE(user_id)
);

-- User achievements for onboarding
CREATE TABLE user_achievements (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    achievement_id VARCHAR(50) NOT NULL,
    unlocked_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    metadata JSONB DEFAULT '{}',
    UNIQUE(user_id, achievement_id)
);

-- Laravel Cashier subscription items (for metered billing if needed)
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

-- Laravel Pennant feature states
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

## Laravel Boost Integration

### AI-Powered Development Architecture

Laravel Boost provides an MCP (Model Context Protocol) server that enables AI assistants to deeply understand and interact with the Achilleus codebase. This integration dramatically improves development speed and code quality.

### Boost Tools Available for Achilleus

```php
// 1. Database Inspection - AI can query actual data
boost:query "SELECT * FROM scans WHERE total_score < 60 ORDER BY created_at DESC"
boost:schema domains  // Shows complete table structure

// 2. Code Execution - Test scanner logic in real-time
boost:tinker "
    $scanner = new SslTlsScanner();
    $result = $scanner->scan('https://github.com');
    return $result->score;
"

// 3. Route Analysis - Understand application structure
boost:routes --method=POST --name=scan
boost:routes --middleware=auth

// 4. Configuration Access - Get actual settings
boost:config app.env
boost:config services.stripe

// 5. Log Analysis - Debug issues
boost:logs --grep="NetworkGuard" --tail=100
boost:logs --level=error

// 6. Documentation Search - Laravel 12 specific
boost:docs "Cashier subscription"
boost:docs "Pennant feature flags"
```

### AI Guidelines for Achilleus

Boost automatically generates guidelines for:
- **Security Requirements**: SSRF protection, HTTPS-only domains
- **Scanner Patterns**: AbstractScanner implementation, retry logic
- **Testing Standards**: TDD approach, Pest PHP conventions
- **Laravel 12 Features**: Latest APIs and best practices
- **Achilleus Conventions**: Domain normalization, scoring algorithms

### Development Workflow with Boost

```bash
# Initial Setup
composer require laravel/boost --dev
php artisan boost:install

# Development Session
php artisan boost:mcp  # Start MCP server

# AI can now:
# - Generate scanners that follow AbstractScanner pattern
# - Create tests using actual database fixtures
# - Debug scan failures by inspecting logs and data
# - Optimize queries by analyzing actual performance
# - Generate UI components matching existing patterns
```

### Security Considerations with Boost

- Boost only runs in development environment
- Never exposes production credentials
- Respects .env and .gitignore boundaries
- AI-generated code automatically includes:
  - NetworkGuard validation for external requests
  - Form Request validation for user inputs
  - Authorization policies for resource access
  - Rate limiting for scan operations

### Boost-Enhanced Scanner Development

When developing scanners with Boost active:

```php
// AI can see actual scanner implementations
boost:tinker "File::get(app_path('Services/Scanners/SslTlsScanner.php'))"

// AI can test SSRF protection
boost:tinker "
    try {
        NetworkGuard::assertPublicHttpUrl('http://169.254.169.254');
    } catch (SecurityException \$e) {
        return \$e->getMessage();
    }
"

// AI can analyze scan performance
boost:query "
    SELECT module, AVG(execution_time) as avg_time, COUNT(*) as runs
    FROM scan_modules 
    WHERE created_at > NOW() - INTERVAL '7 days'
    GROUP BY module
"

// AI can generate test data
boost:tinker "
    \$domain = Domain::factory()->create(['user_id' => 1]);
    \$scan = Scan::factory()->create(['domain_id' => \$domain->id]);
    return \$scan->toArray();
"
```

## Scanner Implementation

### Enhanced Scanner Interface
```php
namespace App\Contracts\Scanners;

use App\Data\ModuleResult;
use App\Exceptions\ScannerException;

interface Scanner {
    public function scan(string $url, array $context = []): ModuleResult;
    public function name(): string;
    public function getWeight(): int;
    public function getTimeout(): int;
    public function getRetryAttempts(): int;
}

// Simplified Scanner Exception Hierarchy
namespace App\Exceptions;

class ScannerException extends \Exception {}

// Core exception types (3 total)
class RetryableException extends ScannerException {
    // For temporary issues that should be retried
    // Examples: timeouts, DNS failures, connection reset, network congestion
    // These errors may resolve themselves on retry
}

class ConfigurationException extends ScannerException {
    // For domain misconfigurations that require user action
    // Examples: expired certificate, missing DNS records, invalid SSL, wrong hostname
    // These errors should NOT be retried as they won't resolve without user intervention
}

class SecurityException extends ScannerException {
    // For security policy violations and access restrictions
    // Examples: SSRF attempts, WAF blocking, rate limiting, CORS rejection, authentication required
    // These errors should NOT be retried as they indicate policy violations
}
```

### Enhanced Module Result Structure
```php
namespace App\Data;

class ModuleResult {
    public function __construct(
        public string $module,
        public int $score,
        public string $status, // ok|warn|fail|error|timeout|rate_limited
        public array $raw = [],
        public ?string $error = null,
        public float $executionTime = 0.0,
        public int $retryCount = 0
    ) {}
    
    public static function error(string $module, string $error, int $retryCount = 0): self {
        return new self(
            module: $module,
            score: 0,
            status: 'error',
            raw: ['error' => $error],
            error: $error,
            retryCount: $retryCount
        );
    }
    
    public static function timeout(string $module, float $executionTime): self {
        return new self(
            module: $module,
            score: 0,
            status: 'timeout',
            raw: ['timeout' => true],
            error: 'Request timed out',
            executionTime: $executionTime
        );
    }
    
    public static function rateLimited(string $module): self {
        return new self(
            module: $module,
            score: 0,
            status: 'rate_limited',
            raw: ['rate_limited' => true],
            error: 'Rate limit exceeded'
        );
    }
}
```

### Abstract Base Scanner
```php
namespace App\Services\Scanners;

use App\Contracts\Scanners\Scanner;
use App\Data\ModuleResult;
use App\Exceptions\{ScannerException, RetryableException, ConfigurationException, SecurityException};
use App\Support\{NetworkGuard, RateLimiter};
use Illuminate\Support\Facades\Log;
use Illuminate\Support\Facades\Cache;

abstract class AbstractScanner implements Scanner {
    protected int $timeout = 30;
    protected int $retryAttempts = 3;
    protected int $retryDelay = 1; // seconds
    
    public function getTimeout(): int {
        return $this->timeout;
    }
    
    public function getRetryAttempts(): int {
        return $this->retryAttempts;
    }
    
    public function scan(string $url, array $context = []): ModuleResult {
        $startTime = microtime(true);
        $attempts = 0;
        $lastError = null;
        
        // Rate limiting check
        if (!$this->checkRateLimit($url)) {
            Log::warning('Rate limit exceeded for scanner', [
                'scanner' => $this->name(),
                'url' => parse_url($url, PHP_URL_HOST)
            ]);
            return ModuleResult::rateLimited($this->name());
        }
        
        while ($attempts < $this->retryAttempts) {
            $attempts++;
            
            try {
                // SSRF protection - throws SecurityException on violation
                try {
                    NetworkGuard::assertPublicHttpUrl($url);
                } catch (\Exception $e) {
                    throw new SecurityException('SSRF attempt blocked: ' . $e->getMessage());
                }
                
                // Perform actual scan
                $result = $this->performScan($url, $context);
                $result->executionTime = microtime(true) - $startTime;
                $result->retryCount = $attempts - 1;
                
                // Update rate limit
                $this->updateRateLimit($url);
                
                return $result;
                
            } catch (SecurityException $e) {
                // Security violations shouldn't be retried (includes SSRF, rate limiting, WAF blocks)
                Log::warning('Security policy violation', [
                    'scanner' => $this->name(),
                    'url' => parse_url($url, PHP_URL_HOST),
                    'type' => class_basename($e),
                    'error' => $e->getMessage()
                ]);
                
                return ModuleResult::error($this->name(), $e->getMessage(), $attempts - 1);
                
            } catch (ConfigurationException $e) {
                // Configuration errors shouldn't be retried (expired certs, DNS issues, etc)
                Log::warning('Domain configuration issue detected', [
                    'scanner' => $this->name(),
                    'url' => parse_url($url, PHP_URL_HOST),
                    'error' => $e->getMessage()
                ]);
                
                return ModuleResult::error(
                    $this->name(),
                    'Configuration issue: ' . $e->getMessage(),
                    $attempts - 1
                );
                
            } catch (RetryableException $e) {
                // Temporary issues that may resolve on retry (timeouts, DNS failures, network issues)
                if ($attempts >= $this->retryAttempts) {
                    Log::warning('Retryable error persisted after all attempts', [
                        'scanner' => $this->name(),
                        'url' => parse_url($url, PHP_URL_HOST),
                        'attempts' => $attempts,
                        'error' => $e->getMessage()
                    ]);
                    
                    return ModuleResult::error(
                        $this->name(),
                        $e->getMessage(),
                        $attempts - 1
                    );
                }
                
                $lastError = $e;
                sleep($this->retryDelay * $attempts); // Exponential backoff
                
            } catch (\Exception $e) {
                Log::error('Scanner unexpected error', [
                    'scanner' => $this->name(),
                    'url' => parse_url($url, PHP_URL_HOST),
                    'error' => $e->getMessage(),
                    'trace' => $e->getTraceAsString()
                ]);
                
                return ModuleResult::error($this->name(), 'Scan failed: ' . $e->getMessage(), $attempts - 1);
            }
        }
        
        return ModuleResult::error($this->name(), $lastError?->getMessage() ?? 'Unknown error', $this->retryAttempts);
    }
    
    abstract protected function performScan(string $url, array $context = []): ModuleResult;
    
    /**
     * Exception usage guide for scanner implementations:
     * 
     * @throws RetryableException - Timeouts, DNS failures, temporary network issues (will be retried)
     * @throws ConfigurationException - Invalid SSL, expired certs, DNS misconfig (won't be retried)
     * @throws SecurityException - SSRF, WAF blocks, rate limits, access denied (won't be retried)
     */
    
    protected function checkRateLimit(string $url): bool {
        $host = parse_url($url, PHP_URL_HOST);
        $key = "scanner_rate_limit:{$this->name()}:{$host}";
        
        // Laravel Cloud Redis handles this automatically
        $requests = Cache::get($key, 0);
        return $requests < $this->getRateLimitPerMinute();
    }
    
    protected function updateRateLimit(string $url): void {
        $host = parse_url($url, PHP_URL_HOST);
        $key = "scanner_rate_limit:{$this->name()}:{$host}";
        
        // Laravel Cloud Redis handles expiration
        Cache::increment($key, 1);
        Cache::expire($key, 60);
    }
    
    protected function getRateLimitPerMinute(): int {
        return 10; // Default: 10 requests per minute per host
    }
}
```

### Enhanced SSL/TLS Scanner
```php
namespace App\Services\Scanners;

use App\Data\ModuleResult;
use App\Exceptions\{RetryableException, ConfigurationException, SecurityException};
use Illuminate\Support\Facades\Log;

class SslTlsScanner extends AbstractScanner {
    protected int $timeout = 30;
    protected int $retryAttempts = 3;
    
    public function name(): string {
        return 'ssl_tls';
    }
    
    public function getWeight(): int {
        return 40; // 40% of total score
    }
    
    protected function getRateLimitPerMinute(): int {
        return 5; // More conservative for SSL checks
    }
    
    protected function performScan(string $url, array $context = []): ModuleResult {
        $host = parse_url($url, PHP_URL_HOST);
        $port = 443;
        
        // Enhanced SSL context with comprehensive settings
        $sslContext = stream_context_create([
            'ssl' => [
                'capture_peer_cert' => true,
                'capture_peer_cert_chain' => true, // Capture full chain
                'verify_peer' => true,
                'verify_peer_name' => true,
                'allow_self_signed' => false,
                'SNI_enabled' => true,
                'disable_compression' => true, // Prevent CRIME attacks
                'peer_name' => $host,
                'crypto_method' => STREAM_CRYPTO_METHOD_TLSv1_2_CLIENT | STREAM_CRYPTO_METHOD_TLSv1_3_CLIENT
            ],
            'http' => [
                'timeout' => $this->timeout,
                'user_agent' => 'Achilleus Security Scanner/1.0'
            ]
        ]);
        
        // Set timeout using alarm for stream_socket_client
        $oldHandler = set_error_handler(function($errno, $errstr) {
            // Categorize error based on message
            if (str_contains(strtolower($errstr), 'certificate') || str_contains(strtolower($errstr), 'verify')) {
                throw new ConfigurationException("SSL certificate issue: {$errstr}");
            } elseif (str_contains(strtolower($errstr), 'refused') || str_contains(strtolower($errstr), 'denied')) {
                throw new SecurityException("Connection blocked: {$errstr}");
            } else {
                // Timeouts, DNS failures, and other network issues are retryable
                throw new RetryableException("SSL connection failed: {$errstr}");
            }
        });
        
        try {
            $startTime = microtime(true);
            $conn = stream_socket_client(
                "ssl://{$host}:{$port}",
                $errno,
                $errstr,
                $this->timeout,
                STREAM_CLIENT_CONNECT,
                $sslContext
            );
            
            if (!$conn) {
                throw new RetryableException($errstr ?: 'SSL connection failed');
            }
            
            // Check for timeout
            $elapsed = microtime(true) - $startTime;
            if ($elapsed > $this->timeout) {
                fclose($conn);
                throw new RetryableException('SSL connection timed out');
            }
            
            // Extract certificate data and metadata
            $contextParams = stream_context_get_params($conn);
            $cert = $contextParams['options']['ssl']['peer_certificate'] ?? null;
            $certChain = $contextParams['options']['ssl']['peer_certificate_chain'] ?? [];
            $meta = stream_get_meta_data($conn);
            
            fclose($conn);
            
            if (!$cert) {
                throw new ConfigurationException('No SSL certificate found - domain may not support HTTPS');
            }
            
            // Enhanced certificate analysis
            $certData = openssl_x509_parse($cert);
            $chainData = array_map('openssl_x509_parse', $certChain);
            
            // Calculate comprehensive score
            $analysis = $this->performEnhancedAnalysis($certData, $chainData, $meta, $host);
            
            return new ModuleResult(
                module: $this->name(),
                score: $analysis['score'],
                status: $this->determineStatus($analysis['score']),
                raw: $analysis['details']
            );
            
        } finally {
            set_error_handler($oldHandler);
        }
    }
    
    private function performEnhancedAnalysis(array $cert, array $chain, array $meta, string $host): array {
        $score = 100;
        $details = [
            'certificate' => $cert,
            'chain' => $chain,
            'protocol' => $meta['crypto']['protocol'] ?? 'unknown',
            'cipher' => $meta['crypto']['cipher_name'] ?? 'unknown',
            'issues' => [],
            'recommendations' => []
        ];
        
        // Certificate expiration check
        if (isset($cert['validTo_time_t'])) {
            $expiryTime = $cert['validTo_time_t'];
            $now = time();
            $daysUntilExpiry = ($expiryTime - $now) / 86400;
            
            if ($expiryTime <= $now) {
                $score = 0; // Expired certificate is critical failure
                $details['issues'][] = 'Certificate has expired';
                return ['score' => $score, 'details' => $details];
            } elseif ($daysUntilExpiry <= 7) {
                $score -= 15;
                $details['issues'][] = "Certificate expires in {$daysUntilExpiry} days";
                $details['recommendations'][] = 'Renew SSL certificate immediately';
            } elseif ($daysUntilExpiry <= 30) {
                $score -= 10;
                $details['issues'][] = "Certificate expires in {$daysUntilExpiry} days";
                $details['recommendations'][] = 'Plan SSL certificate renewal';
            }
            
            $details['days_until_expiry'] = (int)$daysUntilExpiry;
        }
        
        // Hostname verification
        if (!$this->verifyHostname($cert, $host)) {
            $score -= 20;
            $details['issues'][] = 'Certificate hostname mismatch';
            $details['recommendations'][] = 'Ensure certificate matches domain name';
        }
        
        // Protocol version analysis
        $protocol = $meta['crypto']['protocol'] ?? '';
        if (preg_match('/TLSv1\.(0|1)/', $protocol)) {
            $score -= 25;
            $details['issues'][] = "Outdated TLS version: {$protocol}";
            $details['recommendations'][] = 'Upgrade to TLS 1.2 or higher';
        } elseif ($protocol === 'TLSv1.3') {
            $score = min(100, $score + 5); // TLS 1.3 bonus
            $details['strengths'][] = 'Using latest TLS 1.3 protocol';
        }
        
        // Cipher analysis
        $cipher = $meta['crypto']['cipher_name'] ?? '';
        if (preg_match('/(RC4|3DES|NULL|DES|EXPORT)/', $cipher)) {
            $score -= 20;
            $details['issues'][] = "Weak cipher: {$cipher}";
            $details['recommendations'][] = 'Configure stronger cipher suites';
        }
        
        // Perfect Forward Secrecy check
        if (!str_contains($cipher, 'DHE') && !str_contains($cipher, 'ECDHE')) {
            $score -= 10;
            $details['issues'][] = 'No Perfect Forward Secrecy';
            $details['recommendations'][] = 'Enable ECDHE cipher suites';
        } else {
            $details['strengths'][] = 'Perfect Forward Secrecy enabled';
        }
        
        // Key strength analysis
        if (isset($cert['publicKey'])) {
            $keyDetails = openssl_pkey_get_details(openssl_pkey_get_public($cert['publicKey']));
            if ($keyDetails) {
                $keySize = $keyDetails['bits'] ?? 0;
                $keyType = $keyDetails['type'] ?? 0;
                
                if ($keyType === OPENSSL_KEYTYPE_RSA && $keySize < 2048) {
                    $score -= 15;
                    $details['issues'][] = "Weak RSA key size: {$keySize} bits";
                    $details['recommendations'][] = 'Use at least 2048-bit RSA keys';
                } elseif ($keySize >= 4096) {
                    $details['strengths'][] = "Strong key size: {$keySize} bits";
                }
                
                $details['key_size'] = $keySize;
                $details['key_type'] = $this->getKeyTypeName($keyType);
            }
        }
        
        // Certificate chain validation
        $chainIssues = $this->validateCertificateChain($chain);
        if (!empty($chainIssues)) {
            $score -= 10;
            $details['issues'] = array_merge($details['issues'], $chainIssues);
            $details['recommendations'][] = 'Fix certificate chain configuration';
        }
        
        // Signature algorithm check
        if (isset($cert['signatureTypeSN'])) {
            $sigAlg = strtolower($cert['signatureTypeSN']);
            if (str_contains($sigAlg, 'sha1')) {
                $score -= 15;
                $details['issues'][] = 'Using weak SHA-1 signature algorithm';
                $details['recommendations'][] = 'Upgrade to SHA-256 or higher';
            }
        }
        
        return [
            'score' => max(0, $score),
            'details' => $details
        ];
    }
    
    private function verifyHostname(array $cert, string $host): bool {
        // Check CN
        $subject = $cert['subject'] ?? [];
        $cn = $subject['CN'] ?? '';
        
        if ($this->matchesHostname($cn, $host)) {
            return true;
        }
        
        // Check SAN (Subject Alternative Names)
        $extensions = $cert['extensions'] ?? [];
        $san = $extensions['subjectAltName'] ?? '';
        
        if ($san) {
            $sanEntries = explode(', ', $san);
            foreach ($sanEntries as $entry) {
                if (strpos($entry, 'DNS:') === 0) {
                    $dnsName = substr($entry, 4);
                    if ($this->matchesHostname($dnsName, $host)) {
                        return true;
                    }
                }
            }
        }
        
        return false;
    }
    
    private function matchesHostname(string $certName, string $host): bool {
        if ($certName === $host) {
            return true;
        }
        
        // Wildcard matching
        if (str_starts_with($certName, '*.')) {
            $pattern = substr($certName, 2);
            $hostParts = explode('.', $host);
            if (count($hostParts) > 1) {
                $hostWithoutSubdomain = implode('.', array_slice($hostParts, 1));
                return $hostWithoutSubdomain === $pattern;
            }
        }
        
        return false;
    }
    
    private function validateCertificateChain(array $chain): array {
        $issues = [];
        
        if (empty($chain)) {
            return ['Certificate chain is empty'];
        }
        
        // Verify each certificate against its issuer
        for ($i = 0; $i < count($chain) - 1; $i++) {
            $cert = $chain[$i];
            $issuer = $chain[$i + 1];
            
            // Use OpenSSL's proper verification
            $result = openssl_x509_verify($cert, $issuer);
            if ($result !== 1) {
                $issues[] = 'Certificate chain cryptographic verification failed';
                break;
            }
        }
        
        return $issues;
    }
    
    private function getKeyTypeName(int $keyType): string {
        return match($keyType) {
            OPENSSL_KEYTYPE_RSA => 'RSA',
            OPENSSL_KEYTYPE_EC => 'ECDSA',
            OPENSSL_KEYTYPE_DSA => 'DSA',
            default => 'Unknown'
        };
    }
    
    private function determineStatus(int $score): string {
        return match(true) {
            $score >= 90 => 'ok',
            $score >= 70 => 'warn',
            $score > 0 => 'fail',
            default => 'error'
        };
    }
}
```

### Enhanced Security Headers Scanner
```php
namespace App\Services\Scanners;

use App\Data\ModuleResult;
use App\Exceptions\{RetryableException, ConfigurationException, SecurityException};
use Illuminate\Support\Facades\Http;
use Illuminate\Http\Client\RequestException;

class SecurityHeadersScanner extends AbstractScanner {
    protected int $timeout = 20;
    protected int $retryAttempts = 2;
    
    public function name(): string {
        return 'security_headers';
    }
    
    public function getWeight(): int {
        return 30; // 30% of total score
    }
    
    protected function getRateLimitPerMinute(): int {
        return 8; // Conservative HTTP requests
    }
    
    protected function performScan(string $url, array $context = []): ModuleResult {
        try {
            // Configure HTTP client with timeout and security headers
            $response = Http::timeout($this->timeout)
                ->withUserAgent('Achilleus Security Scanner/1.0')
                ->withOptions([
                    'verify' => true, // Verify SSL certificates
                    'allow_redirects' => [
                        'max' => 3,
                        'strict' => true,
                        'referer' => false,
                        'protocols' => ['https'] // HTTPS only
                    ]
                ])
                ->get($url);
                
            if ($response->failed()) {
                $status = $response->status();
                if ($status === 403 || $status === 401) {
                    throw new SecurityException("Access denied: HTTP {$status}");
                } elseif ($status === 502 || $status === 503 || $status === 504) {
                    throw new RetryableException("Server temporarily unavailable: HTTP {$status}");
                } elseif ($status >= 400 && $status < 500) {
                    throw new ConfigurationException("Client error: HTTP {$status}");
                } else {
                    throw new RetryableException("HTTP request failed: {$status}");
                }
            }
            
            // Extract and normalize headers
            $headers = collect($response->headers())
                ->mapWithKeys(function ($value, $key) {
                    return [strtolower($key) => is_array($value) ? ($value[0] ?? null) : $value];
                })
                ->filter(); // Remove empty values
            
            // Perform comprehensive header analysis
            $analysis = $this->performHeaderAnalysis($headers->toArray(), $url);
            
            return new ModuleResult(
                module: $this->name(),
                score: $analysis['score'],
                status: $this->determineStatus($analysis['score']),
                raw: $analysis['details']
            );
            
        } catch (RequestException $e) {
            if ($e->getCode() === CURLE_OPERATION_TIMEDOUT) {
                throw new RetryableException('HTTP request timed out');
            }
            throw new RetryableException('HTTP request failed: ' . $e->getMessage());
        }
    }
    
    private function performHeaderAnalysis(array $headers, string $url): array {
        $score = 100;
        $details = [
            'url' => $url,
            'headers_found' => [],
            'headers_missing' => [],
            'issues' => [],
            'recommendations' => [],
            'strengths' => []
        ];
        
        // HSTS Analysis (35 points)
        $hstsAnalysis = $this->analyzeHSTS($headers);
        $score -= (35 - $hstsAnalysis['score']);
        $details = array_merge_recursive($details, $hstsAnalysis['details']);
        
        // Content Security Policy (35 points)  
        $cspAnalysis = $this->analyzeCSP($headers);
        $score -= (35 - $cspAnalysis['score']);
        $details = array_merge_recursive($details, $cspAnalysis['details']);
        
        // X-Content-Type-Options (10 points)
        $xContentAnalysis = $this->analyzeXContentTypeOptions($headers);
        $score -= (10 - $xContentAnalysis['score']);
        $details = array_merge_recursive($details, $xContentAnalysis['details']);
        
        // X-Frame-Options (10 points)
        $xFrameAnalysis = $this->analyzeXFrameOptions($headers);
        $score -= (10 - $xFrameAnalysis['score']);
        $details = array_merge_recursive($details, $xFrameAnalysis['details']);
        
        // Referrer-Policy (10 points)
        $referrerAnalysis = $this->analyzeReferrerPolicy($headers);
        $score -= (10 - $referrerAnalysis['score']);
        $details = array_merge_recursive($details, $referrerAnalysis['details']);
        
        // Additional security headers (bonus/penalty)
        $additionalAnalysis = $this->analyzeAdditionalHeaders($headers);
        $score += $additionalAnalysis['score_adjustment'];
        $details = array_merge_recursive($details, $additionalAnalysis['details']);
        
        return [
            'score' => max(0, min(100, $score)),
            'details' => $details
        ];
    }
    
    private function analyzeHSTS(array $headers): array {
        $hsts = $headers['strict-transport-security'] ?? null;
        $analysis = ['score' => 0, 'details' => []];
        
        if (!$hsts) {
            $analysis['details']['headers_missing'][] = 'Strict-Transport-Security';
            $analysis['details']['issues'][] = 'HSTS header missing - site vulnerable to protocol downgrade attacks';
            $analysis['details']['recommendations'][] = 'Add HSTS header with max-age of at least 180 days';
            return $analysis;
        }
        
        $analysis['details']['headers_found']['strict-transport-security'] = $hsts;
        $analysis['score'] = 35; // Base score for having HSTS
        
        // Parse HSTS directives
        $directives = $this->parseHeaderDirectives($hsts);
        
        // Check max-age
        $maxAge = (int)($directives['max-age'] ?? 0);
        if ($maxAge === 0) {
            $analysis['score'] -= 15;
            $analysis['details']['issues'][] = 'HSTS max-age is 0 or missing';
        } elseif ($maxAge < 15552000) { // 180 days
            $analysis['score'] -= 10;
            $analysis['details']['issues'][] = "HSTS max-age is too short: {$maxAge} seconds";
            $analysis['details']['recommendations'][] = 'Increase HSTS max-age to at least 15552000 (180 days)';
        } else {
            $analysis['details']['strengths'][] = "HSTS configured with adequate max-age: {$maxAge} seconds";
        }
        
        // Check includeSubDomains
        if (!isset($directives['includesubdomains'])) {
            $analysis['score'] -= 5;
            $analysis['details']['recommendations'][] = 'Add includeSubDomains to HSTS header';
        } else {
            $analysis['details']['strengths'][] = 'HSTS includes subdomains';
        }
        
        // Check preload
        if (!isset($directives['preload'])) {
            $analysis['score'] -= 5;
            $analysis['details']['recommendations'][] = 'Consider adding preload directive to HSTS';
        } else {
            $analysis['details']['strengths'][] = 'HSTS configured for preload';
        }
        
        return $analysis;
    }
    
    private function analyzeCSP(array $headers): array {
        $csp = $headers['content-security-policy'] ?? null;
        $analysis = ['score' => 0, 'details' => []];
        
        if (!$csp) {
            $analysis['details']['headers_missing'][] = 'Content-Security-Policy';
            $analysis['details']['issues'][] = 'CSP header missing - site vulnerable to XSS attacks';
            $analysis['details']['recommendations'][] = 'Implement Content Security Policy header';
            return $analysis;
        }
        
        $analysis['details']['headers_found']['content-security-policy'] = $csp;
        $analysis['score'] = 35; // Base score for having CSP
        
        // Parse CSP directives
        $directives = $this->parseCSPDirectives($csp);
        
        // Check for unsafe directives
        $unsafePatterns = ['unsafe-inline', 'unsafe-eval', '*', 'data:'];
        $unsafeFound = 0;
        
        foreach ($directives as $directive => $sources) {
            foreach ($sources as $source) {
                foreach ($unsafePatterns as $pattern) {
                    if (str_contains(strtolower($source), $pattern)) {
                        $unsafeFound++;
                        break 2; // Break out of both loops
                    }
                }
            }
        }
        
        if ($unsafeFound > 0) {
            $penalty = min(15, $unsafeFound * 5); // Max 15 point penalty
            $analysis['score'] -= $penalty;
            $analysis['details']['issues'][] = 'CSP contains unsafe directives (unsafe-inline, unsafe-eval, or wildcard)';
            $analysis['details']['recommendations'][] = 'Remove unsafe CSP directives and use nonces or hashes instead';
        } else {
            $analysis['details']['strengths'][] = 'CSP configured without unsafe directives';
        }
        
        // Check for important directives
        $importantDirectives = ['default-src', 'script-src', 'object-src', 'frame-ancestors'];
        $missingImportant = [];
        
        foreach ($importantDirectives as $directive) {
            if (!isset($directives[$directive])) {
                $missingImportant[] = $directive;
            }
        }
        
        if (!empty($missingImportant)) {
            $analysis['details']['recommendations'][] = 'Consider adding CSP directives: ' . implode(', ', $missingImportant);
        }
        
        return $analysis;
    }
    
    private function analyzeXContentTypeOptions(array $headers): array {
        $header = $headers['x-content-type-options'] ?? null;
        $analysis = ['score' => 0, 'details' => []];
        
        if ($header === 'nosniff') {
            $analysis['score'] = 10;
            $analysis['details']['headers_found']['x-content-type-options'] = $header;
            $analysis['details']['strengths'][] = 'X-Content-Type-Options configured correctly';
        } else {
            $analysis['details']['headers_missing'][] = 'X-Content-Type-Options';
            $analysis['details']['issues'][] = 'X-Content-Type-Options header missing or incorrect';
            $analysis['details']['recommendations'][] = 'Add "X-Content-Type-Options: nosniff" header';
        }
        
        return $analysis;
    }
    
    private function analyzeXFrameOptions(array $headers): array {
        $header = $headers['x-frame-options'] ?? null;
        $analysis = ['score' => 0, 'details' => []];
        
        if (in_array(strtoupper($header ?? ''), ['DENY', 'SAMEORIGIN'])) {
            $analysis['score'] = 10;
            $analysis['details']['headers_found']['x-frame-options'] = $header;
            $analysis['details']['strengths'][] = 'X-Frame-Options configured correctly';
        } else {
            $analysis['details']['headers_missing'][] = 'X-Frame-Options';
            $analysis['details']['issues'][] = 'X-Frame-Options header missing or set to ALLOWALL';
            $analysis['details']['recommendations'][] = 'Set X-Frame-Options to DENY or SAMEORIGIN';
        }
        
        return $analysis;
    }
    
    private function analyzeReferrerPolicy(array $headers): array {
        $header = $headers['referrer-policy'] ?? null;
        $analysis = ['score' => 0, 'details' => []];
        
        $secureValues = [
            'no-referrer',
            'strict-origin',
            'strict-origin-when-cross-origin',
            'same-origin'
        ];
        
        if (in_array($header, $secureValues)) {
            $analysis['score'] = 10;
            $analysis['details']['headers_found']['referrer-policy'] = $header;
            $analysis['details']['strengths'][] = 'Referrer-Policy configured securely';
        } else {
            $analysis['details']['headers_missing'][] = 'Referrer-Policy';
            $analysis['details']['issues'][] = 'Referrer-Policy header missing or insecure';
            $analysis['details']['recommendations'][] = 'Set Referrer-Policy to strict-origin-when-cross-origin';
        }
        
        return $analysis;
    }
    
    private function analyzeAdditionalHeaders(array $headers): array {
        $analysis = ['score_adjustment' => 0, 'details' => []];
        
        // Permissions-Policy (bonus)
        if (isset($headers['permissions-policy'])) {
            $analysis['score_adjustment'] += 2;
            $analysis['details']['headers_found']['permissions-policy'] = $headers['permissions-policy'];
            $analysis['details']['strengths'][] = 'Permissions-Policy header configured';
        }
        
        // X-XSS-Protection (legacy, but still good)
        if (isset($headers['x-xss-protection'])) {
            $value = $headers['x-xss-protection'];
            if ($value === '1; mode=block') {
                $analysis['score_adjustment'] += 1;
                $analysis['details']['strengths'][] = 'X-XSS-Protection configured correctly';
            }
        }
        
        // Server header disclosure (penalty)
        if (isset($headers['server'])) {
            $server = strtolower($headers['server']);
            if (preg_match('/(apache|nginx|iis)\/[\d.]+/', $server)) {
                $analysis['score_adjustment'] -= 2;
                $analysis['details']['issues'][] = 'Server version disclosed in Server header';
                $analysis['details']['recommendations'][] = 'Hide server version information';
            }
        }
        
        return $analysis;
    }
    
    private function parseHeaderDirectives(string $header): array {
        $directives = [];
        $parts = explode(';', $header);
        
        foreach ($parts as $part) {
            $part = trim($part);
            if (strpos($part, '=') !== false) {
                list($key, $value) = explode('=', $part, 2);
                $directives[strtolower(trim($key))] = trim($value);
            } else {
                $directives[strtolower($part)] = true;
            }
        }
        
        return $directives;
    }
    
    private function parseCSPDirectives(string $csp): array {
        $directives = [];
        $parts = explode(';', $csp);
        
        foreach ($parts as $part) {
            $part = trim($part);
            $pieces = explode(' ', $part);
            if (!empty($pieces)) {
                $directive = strtolower(array_shift($pieces));
                $directives[$directive] = $pieces;
            }
        }
        
        return $directives;
    }
    
    private function determineStatus(int $score): string {
        return match(true) {
            $score >= 90 => 'ok',
            $score >= 70 => 'warn',
            $score > 0 => 'fail',
            default => 'error'
        };
    }
}
```

### Enhanced DNS/Email Scanner

#### DNSSEC Implementation Requirements

The DNS scanner includes comprehensive DNSSEC validation to ensure domain integrity:

**Key Components:**
1. **DS Records (Delegation Signer)**: Links parent and child zones in DNSSEC chain
2. **DNSKEY Records**: Contains public keys used to verify DNS signatures
3. **RRSIG Records**: Digital signatures for DNS resource records
4. **Chain of Trust Validation**: Verifies the complete DNSSEC validation path

**Supported Algorithms:**
- **Strong (10 points)**: ECDSA (13-14), EdDSA (15-16)
- **Good (10 points)**: RSASHA256 (8)
- **Weak (7 points)**: RSASHA1 (7) - penalized for using deprecated algorithm

**Validation Process:**
1. Query DS records at parent zone
2. Retrieve DNSKEY records from authoritative nameserver
3. Check for RRSIG signatures on DNS responses
4. Validate the complete chain of trust
5. Assess cryptographic algorithm strength

**Dependencies:**
- Requires `dig` command available on system
- Falls back gracefully if DNS tools unavailable
- Caches results for 5 minutes to reduce query load

```php
namespace App\Services\Scanners;

use App\Data\ModuleResult;
use App\Exceptions\{RetryableException, ConfigurationException, SecurityException};

class DnsEmailScanner extends AbstractScanner {
    protected int $timeout = 15;
    protected int $retryAttempts = 2;
    
    public function name(): string {
        return 'dns_email';
    }
    
    public function getWeight(): int {
        return 30; // 30% of total score
    }
    
    protected function getRateLimitPerMinute(): int {
        return 15; // DNS queries are generally lighter
    }
    
    protected function performScan(string $url, array $context = []): ModuleResult {
        $host = parse_url($url, PHP_URL_HOST);
        $emailMode = $context['email_mode'] ?? 'expected';
        $dkimSelector = $context['dkim_selector'] ?? 'default';
        
        $analysis = $this->performDnsAnalysis($host, $emailMode, $dkimSelector);
        
        return new ModuleResult(
            module: $this->name(),
            score: $analysis['score'],
            status: $this->determineStatus($analysis['score']),
            raw: $analysis['details']
        );
    }
    
    private function performDnsAnalysis(string $host, string $emailMode, string $dkimSelector): array {
        $score = 100;
        $details = [
            'host' => $host,
            'email_mode' => $emailMode,
            'dns_records' => [],
            'email_security' => [],
            'issues' => [],
            'recommendations' => [],
            'strengths' => []
        ];
        
        try {
            // Basic DNS connectivity check
            $aRecords = $this->queryDnsRecords($host, DNS_A + DNS_AAAA, 'A/AAAA');
            if (empty($aRecords)) {
                $score -= 10;
                $details['issues'][] = 'Domain has no A or AAAA records';
                $details['recommendations'][] = 'Configure DNS A/AAAA records for the domain';
            } else {
                $details['dns_records']['a_records'] = array_map(function($record) {
                    return $record['ip'] ?? $record['ipv6'] ?? 'unknown';
                }, $aRecords);
                $details['strengths'][] = 'Domain resolves correctly';
            }
            
            // Email security analysis (if expected)
            if ($emailMode === 'expected') {
                $emailAnalysis = $this->analyzeEmailSecurity($host, $dkimSelector);
                $score -= (30 - $emailAnalysis['score']); // Max 30 points for email
                $details['email_security'] = $emailAnalysis['details'];
                $details['issues'] = array_merge($details['issues'], $emailAnalysis['issues']);
                $details['recommendations'] = array_merge($details['recommendations'], $emailAnalysis['recommendations']);
                $details['strengths'] = array_merge($details['strengths'], $emailAnalysis['strengths']);
            } else {
                $details['email_security']['status'] = 'skipped';
                $details['strengths'][] = 'Email security analysis skipped as requested';
            }
            
            // DNSSEC analysis
            $dnssecAnalysis = $this->analyzeDnssec($host);
            $score -= (10 - $dnssecAnalysis['score']);
            $details['dns_records']['dnssec'] = $dnssecAnalysis['details'] ?? [];
            if ($dnssecAnalysis['score'] > 0) {
                $details['strengths'] = array_merge($details['strengths'], $dnssecAnalysis['strengths']);
            } else {
                $details['issues'] = array_merge($details['issues'], $dnssecAnalysis['issues']);
                $details['recommendations'] = array_merge($details['recommendations'], $dnssecAnalysis['recommendations']);
            }
            
            // CAA record analysis
            $caaAnalysis = $this->analyzeCaaRecords($host);
            $score -= (5 - $caaAnalysis['score']);
            $details['dns_records']['caa'] = $caaAnalysis['details'];
            if ($caaAnalysis['score'] > 0) {
                $details['strengths'] = array_merge($details['strengths'], $caaAnalysis['strengths']);
            } else {
                $details['recommendations'] = array_merge($details['recommendations'], $caaAnalysis['recommendations']);
            }
            
        } catch (\Exception $e) {
            // DNS failures are usually temporary and retryable
            throw new RetryableException("DNS analysis failed: " . $e->getMessage());
        }
        
        return [
            'score' => max(0, $score),
            'details' => $details
        ];
    }
    
    private function analyzeEmailSecurity(string $host, string $dkimSelector): array {
        $score = 30; // Full points if all email security is perfect
        $details = [];
        $issues = [];
        $recommendations = [];
        $strengths = [];
        
        // MX Records (5 points)
        $mxRecords = $this->queryDnsRecords($host, DNS_MX, 'MX');
        if (empty($mxRecords)) {
            $score -= 5;
            $issues[] = 'No MX records found';
            $recommendations[] = 'Configure MX records for email delivery';
            $details['mx_records'] = [];
        } else {
            $details['mx_records'] = array_map(function($record) {
                return [
                    'priority' => $record['pri'] ?? 0,
                    'target' => $record['target'] ?? 'unknown'
                ];
            }, $mxRecords);
            $strengths[] = 'MX records configured';
        }
        
        // SPF Analysis (8 points)
        $spfAnalysis = $this->analyzeSPF($host);
        $score -= (8 - $spfAnalysis['score']);
        $details['spf'] = $spfAnalysis['details'];
        $issues = array_merge($issues, $spfAnalysis['issues']);
        $recommendations = array_merge($recommendations, $spfAnalysis['recommendations']);
        $strengths = array_merge($strengths, $spfAnalysis['strengths']);
        
        // DKIM Analysis (7 points)
        $dkimAnalysis = $this->analyzeDKIM($host, $dkimSelector);
        $score -= (7 - $dkimAnalysis['score']);
        $details['dkim'] = $dkimAnalysis['details'];
        $issues = array_merge($issues, $dkimAnalysis['issues']);
        $recommendations = array_merge($recommendations, $dkimAnalysis['recommendations']);
        $strengths = array_merge($strengths, $dkimAnalysis['strengths']);
        
        // DMARC Analysis (10 points)
        $dmarcAnalysis = $this->analyzeDMARC($host);
        $score -= (10 - $dmarcAnalysis['score']);
        $details['dmarc'] = $dmarcAnalysis['details'];
        $issues = array_merge($issues, $dmarcAnalysis['issues']);
        $recommendations = array_merge($recommendations, $dmarcAnalysis['recommendations']);
        $strengths = array_merge($strengths, $dmarcAnalysis['strengths']);
        
        return [
            'score' => max(0, $score),
            'details' => $details,
            'issues' => $issues,
            'recommendations' => $recommendations,
            'strengths' => $strengths
        ];
    }
    
    private function analyzeSPF(string $host): array {
        $txtRecords = $this->queryDnsRecords($host, DNS_TXT, 'TXT');
        $spfRecord = null;
        
        foreach ($txtRecords as $record) {
            $txt = $record['txt'] ?? '';
            if (str_starts_with($txt, 'v=spf1')) {
                $spfRecord = $txt;
                break;
            }
        }
        
        if (!$spfRecord) {
            return [
                'score' => 0,
                'details' => ['present' => false],
                'issues' => ['SPF record not found'],
                'recommendations' => ['Add SPF record to prevent email spoofing'],
                'strengths' => []
            ];
        }
        
        $analysis = [
            'score' => 8,
            'details' => ['present' => true, 'record' => $spfRecord],
            'issues' => [],
            'recommendations' => [],
            'strengths' => ['SPF record configured']
        ];
        
        // Check for overly permissive SPF
        if (str_contains($spfRecord, '+all')) {
            $analysis['score'] -= 3;
            $analysis['issues'][] = 'SPF record allows any server (+all)';
            $analysis['recommendations'][] = 'Tighten SPF policy to specific servers';
        } elseif (str_contains($spfRecord, '?all')) {
            $analysis['score'] -= 1;
            $analysis['issues'][] = 'SPF record is neutral (?all)';
            $analysis['recommendations'][] = 'Consider using ~all or -all for stronger protection';
        } elseif (str_contains($spfRecord, '-all')) {
            $analysis['strengths'][] = 'SPF configured with strict policy (-all)';
        }
        
        return $analysis;
    }
    
    private function analyzeDKIM(string $host, string $selector): array {
        $dkimHost = "{$selector}._domainkey.{$host}";
        $dkimRecords = $this->queryDnsRecords($dkimHost, DNS_TXT, 'DKIM');
        
        if (empty($dkimRecords)) {
            // Try common selectors
            $commonSelectors = ['google', 'default', 'selector1', 'selector2', 'k1'];
            foreach ($commonSelectors as $commonSelector) {
                $testHost = "{$commonSelector}._domainkey.{$host}";
                $testRecords = $this->queryDnsRecords($testHost, DNS_TXT, 'DKIM');
                if (!empty($testRecords)) {
                    $dkimRecords = $testRecords;
                    break;
                }
            }
        }
        
        if (empty($dkimRecords)) {
            return [
                'score' => 0,
                'details' => ['present' => false, 'selector' => $selector],
                'issues' => ['DKIM record not found'],
                'recommendations' => ['Configure DKIM signing for email authentication'],
                'strengths' => []
            ];
        }
        
        $dkimRecord = $dkimRecords[0]['txt'] ?? '';
        $analysis = [
            'score' => 7,
            'details' => ['present' => true, 'selector' => $selector, 'record' => $dkimRecord],
            'issues' => [],
            'recommendations' => [],
            'strengths' => ['DKIM record configured']
        ];
        
        // Parse DKIM record
        $dkimParts = $this->parseDkimRecord($dkimRecord);
        
        // Check key algorithm
        if (isset($dkimParts['k']) && $dkimParts['k'] === 'rsa') {
            $analysis['strengths'][] = 'Using RSA key algorithm';
        }
        
        // Check public key length (if parseable)
        if (isset($dkimParts['p']) && !empty($dkimParts['p'])) {
            $keyData = base64_decode($dkimParts['p']);
            if (strlen($keyData) < 256) { // Rough check for key strength
                $analysis['score'] -= 2;
                $analysis['issues'][] = 'DKIM key may be too short';
                $analysis['recommendations'][] = 'Use at least 2048-bit DKIM keys';
            }
        }
        
        return $analysis;
    }
    
    private function analyzeDMARC(string $host): array {
        $dmarcHost = "_dmarc.{$host}";
        $dmarcRecords = $this->queryDnsRecords($dmarcHost, DNS_TXT, 'DMARC');
        
        if (empty($dmarcRecords)) {
            return [
                'score' => 0,
                'details' => ['present' => false],
                'issues' => ['DMARC record not found'],
                'recommendations' => ['Implement DMARC policy for email security'],
                'strengths' => []
            ];
        }
        
        $dmarcRecord = $dmarcRecords[0]['txt'] ?? '';
        $analysis = [
            'score' => 10,
            'details' => ['present' => true, 'record' => $dmarcRecord],
            'issues' => [],
            'recommendations' => [],
            'strengths' => ['DMARC record configured']
        ];
        
        // Parse DMARC policy
        $dmarcPolicy = $this->parseDmarcRecord($dmarcRecord);
        $policy = $dmarcPolicy['p'] ?? 'none';
        
        switch ($policy) {
            case 'reject':
                $analysis['strengths'][] = 'DMARC configured with strict reject policy';
                break;
            case 'quarantine':
                $analysis['score'] -= 2;
                $analysis['recommendations'][] = 'Consider upgrading DMARC policy to reject';
                break;
            case 'none':
                $analysis['score'] -= 5;
                $analysis['issues'][] = 'DMARC policy is set to none (monitoring only)';
                $analysis['recommendations'][] = 'Upgrade DMARC policy to quarantine or reject';
                break;
        }
        
        // Check reporting addresses
        if (isset($dmarcPolicy['rua']) || isset($dmarcPolicy['ruf'])) {
            $analysis['strengths'][] = 'DMARC reporting configured';
        } else {
            $analysis['recommendations'][] = 'Configure DMARC reporting addresses (rua/ruf)';
        }
        
        return $analysis;
    }
    
    /**
     * Analyze DNSSEC configuration for the domain
     * 
     * Scoring Criteria (10 points total):
     * - 10 points: Full DNSSEC chain validated (DS, DNSKEY, RRSIG all present and valid)
     * - 7 points: DNSSEC enabled with valid signatures but weak algorithm
     * - 5 points: DS records present but signatures missing/invalid
     * - 3 points: DNSKEY records present but no DS at parent
     * - 0 points: No DNSSEC configuration detected
     */
    private function analyzeDnssec(string $host): array {
        try {
            $score = 0;
            $issues = [];
            $recommendations = [];
            $strengths = [];
            $details = [];
            
            // Step 1: Check for DS records at parent zone (delegation signer)
            $dsCommand = sprintf("dig +short DS %s", escapeshellarg($host));
            $dsOutput = shell_exec($dsCommand);
            $hasDS = !empty(trim($dsOutput ?? ''));
            
            // Step 2: Check for DNSKEY records (zone signing keys)
            $dnskeyCommand = sprintf("dig +short DNSKEY %s", escapeshellarg($host));
            $dnskeyOutput = shell_exec($dnskeyCommand);
            $hasDNSKEY = !empty(trim($dnskeyOutput ?? ''));
            
            // Step 3: Check for RRSIG records (resource record signatures)
            $rrsigCommand = sprintf("dig +dnssec +short A %s", escapeshellarg($host));
            $rrsigOutput = shell_exec($rrsigCommand);
            $hasRRSIG = strpos($rrsigOutput ?? '', 'RRSIG') !== false;
            
            // Step 4: Validate the chain of trust
            $validationCommand = sprintf("dig +dnssec +short %s | grep -E '^flags.*ad'", escapeshellarg($host));
            $validationOutput = shell_exec($validationCommand);
            $isValidated = !empty(trim($validationOutput ?? ''));
            
            // Step 5: Check algorithm strength (if DNSKEY exists)
            $algorithmStrength = 'unknown';
            if ($hasDNSKEY && preg_match('/\s+(7|8|13|14|15|16)\s+/', $dnskeyOutput, $matches)) {
                // 7=RSASHA1-NSEC3-SHA1 (weak), 8=RSASHA256 (good), 
                // 13=ECDSAP256SHA256 (strong), 14=ECDSAP384SHA384 (strong),
                // 15=ED25519 (strong), 16=ED448 (strong)
                $algorithm = (int)$matches[1];
                if ($algorithm === 7) {
                    $algorithmStrength = 'weak';
                    $issues[] = 'Using weak RSASHA1 algorithm';
                    $recommendations[] = 'Upgrade to RSASHA256 or ECDSA algorithms';
                } elseif ($algorithm === 8) {
                    $algorithmStrength = 'good';
                    $details['algorithm'] = 'RSASHA256';
                } elseif (in_array($algorithm, [13, 14, 15, 16])) {
                    $algorithmStrength = 'strong';
                    $details['algorithm'] = 'ECDSA/EdDSA';
                    $strengths[] = 'Using modern cryptographic algorithms';
                }
            }
            
            // Calculate score based on DNSSEC status
            if ($hasDS && $hasDNSKEY && $hasRRSIG) {
                if ($isValidated) {
                    // Full DNSSEC validation
                    if ($algorithmStrength === 'weak') {
                        $score = 7;
                        $strengths[] = 'DNSSEC fully configured and validated';
                    } elseif ($algorithmStrength === 'strong') {
                        $score = 10;
                        $strengths[] = 'DNSSEC fully configured with strong algorithms';
                    } else {
                        $score = 10;
                        $strengths[] = 'DNSSEC fully configured and validated';
                    }
                    $details['status'] = 'validated';
                    $details['chain_of_trust'] = 'complete';
                } else {
                    // DNSSEC present but validation failed
                    $score = 5;
                    $issues[] = 'DNSSEC signatures present but validation failed';
                    $recommendations[] = 'Check DNSSEC configuration for errors';
                    $details['status'] = 'validation_failed';
                }
            } elseif ($hasDS && !$hasDNSKEY) {
                // DS records but no DNSKEY
                $score = 3;
                $issues[] = 'DS records present but DNSKEY missing';
                $recommendations[] = 'Complete DNSSEC setup by adding DNSKEY records';
                $details['status'] = 'incomplete';
            } elseif (!$hasDS && $hasDNSKEY) {
                // DNSKEY but no DS records
                $score = 3;
                $issues[] = 'DNSKEY records present but not secured at parent (no DS)';
                $recommendations[] = 'Submit DS records to parent zone/registrar';
                $details['status'] = 'unsigned_delegation';
            } else {
                // No DNSSEC
                $score = 0;
                $issues[] = 'DNSSEC not configured';
                $recommendations[] = 'Enable DNSSEC to protect against DNS spoofing and cache poisoning';
                $details['status'] = 'not_configured';
            }
            
            // Add detailed information
            $details['has_ds_records'] = $hasDS;
            $details['has_dnskey_records'] = $hasDNSKEY;
            $details['has_rrsig_records'] = $hasRRSIG;
            $details['validation_passed'] = $isValidated;
            
            return [
                'score' => $score,
                'issues' => $issues,
                'recommendations' => $recommendations,
                'strengths' => $strengths,
                'details' => $details
            ];
            
        } catch (\Exception $e) {
            Log::warning('DNSSEC validation failed', [
                'host' => $host,
                'error' => $e->getMessage()
            ]);
            
            return [
                'score' => 0,
                'issues' => ['DNSSEC validation could not be performed'],
                'recommendations' => ['Unable to verify DNSSEC status - check DNS configuration'],
                'strengths' => [],
                'details' => ['error' => $e->getMessage()]
            ];
        }
    }
    
    private function analyzeCaaRecords(string $host): array {
        $caaRecords = $this->queryDnsRecords($host, DNS_CAA, 'CAA');
        
        if (empty($caaRecords)) {
            return [
                'score' => 0,
                'details' => ['present' => false],
                'recommendations' => ['Add CAA records to control certificate issuance'],
                'strengths' => []
            ];
        }
        
        return [
            'score' => 5,
            'details' => [
                'present' => true,
                'records' => array_map(function($record) {
                    return $record['value'] ?? 'unknown';
                }, $caaRecords)
            ],
            'recommendations' => [],
            'strengths' => ['CAA records configured']
        ];
    }
    
    private function queryDnsRecords(string $host, int $type, string $typeName): array {
        $records = [];
        
        // Set timeout for DNS queries
        $oldTimeout = ini_get('default_socket_timeout');
        ini_set('default_socket_timeout', $this->timeout);
        
        try {
            $result = @dns_get_record($host, $type);
            if ($result !== false) {
                $records = $result;
            }
        } catch (\Exception $e) {
            // DNS query failures are typically retryable
            throw new RetryableException("DNS query failed for {$typeName} records: " . $e->getMessage());
        } finally {
            ini_set('default_socket_timeout', $oldTimeout);
        }
        
        return $records;
    }
    
    private function parseDkimRecord(string $record): array {
        $parts = [];
        $pairs = explode(';', $record);
        
        foreach ($pairs as $pair) {
            $pair = trim($pair);
            if (strpos($pair, '=') !== false) {
                list($key, $value) = explode('=', $pair, 2);
                $parts[trim($key)] = trim($value);
            }
        }
        
        return $parts;
    }
    
    private function parseDmarcRecord(string $record): array {
        $parts = [];
        $pairs = explode(';', $record);
        
        foreach ($pairs as $pair) {
            $pair = trim($pair);
            if (strpos($pair, '=') !== false) {
                list($key, $value) = explode('=', $pair, 2);
                $parts[trim($key)] = trim($value);
            }
        }
        
        return $parts;
    }
    
    private function determineStatus(int $score): string {
        return match(true) {
            $score >= 90 => 'ok',
            $score >= 70 => 'warn',
            $score > 0 => 'fail',
            default => 'error'
        };
    }
}
```
```

## Security Implementation

### SSRF Protection (NetworkGuard)
```php
<?php
namespace App\Support;

final class NetworkGuard {
    private const BLOCKED_IPS = [
        '127.0.0.1', '::1', '0.0.0.0', '169.254.169.254'
    ];
    
    private const BLOCKED_CIDRS = [
        '10.0.0.0/8', '172.16.0.0/12', '192.168.0.0/16', 'fc00::/7'
    ];
    
    public static function assertPublicHttpUrl(string $url): void {
        $parsed = parse_url($url);
        
        if (!in_array($parsed['scheme'] ?? '', ['http', 'https'])) {
            throw new \InvalidArgumentException('Invalid URL scheme');
        }
        
        $host = $parsed['host'] ?? '';
        if (!$host) {
            throw new \InvalidArgumentException('Missing host');
        }
        
        // Resolve all IPs for the host
        $ips = self::resolveAllIps($host);
        
        foreach ($ips as $ip) {
            if (self::isBlockedIp($ip)) {
                throw new \RuntimeException('Private/reserved IP not allowed');
            }
        }
    }
    
    private static function resolveAllIps(string $host): array {
        $records = @dns_get_record($host, DNS_A + DNS_AAAA) ?: [];
        $ips = [];
        
        foreach ($records as $record) {
            if (!empty($record['ip'])) $ips[] = $record['ip'];
            if (!empty($record['ipv6'])) $ips[] = $record['ipv6'];
        }
        
        return array_unique($ips);
    }
    
    private static function isBlockedIp(string $ip): bool {
        // Check against blocked IPs
        if (in_array($ip, self::BLOCKED_IPS)) {
            return true;
        }
        
        // Check against private ranges
        if (!filter_var($ip, FILTER_VALIDATE_IP, FILTER_FLAG_NO_PRIV_RANGE | FILTER_FLAG_NO_RES_RANGE)) {
            return true;
        }
        
        // Check against CIDR blocks
        foreach (self::BLOCKED_CIDRS as $cidr) {
            if (self::ipInCidr($ip, $cidr)) {
                return true;
            }
        }
        
        return false;
    }
    
    private static function ipInCidr(string $ip, string $cidr): bool {
        list($subnet, $bits) = explode('/', $cidr);
        if (filter_var($ip, FILTER_VALIDATE_IP, FILTER_FLAG_IPV4)) {
            $ip = ip2long($ip);
            $subnet = ip2long($subnet);
            $mask = -1 << (32 - $bits);
            $subnet &= $mask;
            return ($ip & $mask) == $subnet;
        }
        // IPv6 handling would go here if needed
        return false;
    }
}
```

## Job Processing (Laravel Cloud Managed)

### Scan Job Implementation
```php
namespace App\Jobs;

use App\Models\Scan;
use App\Services\Scanners\SecurityHeadersScanner;
use App\Services\Scanners\SslTlsScanner;
use App\Services\Scanners\DnsEmailScanner;
use App\Services\Scoring\SecurityScorer;
use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Queue\SerializesModels;

class RunDomainScan implements ShouldQueue {
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;
    
    public $tries = 3;
    public $timeout = 60;
    
    public function __construct(
        private Scan $scan
    ) {}
    
    public function handle(
        SecurityHeadersScanner $headersScanner,
        SslTlsScanner $sslScanner,
        DnsEmailScanner $dnsScanner,
        SecurityScorer $scorer
    ): void {
        $domain = $this->scan->domain;
        
        // Update status to running
        $this->scan->update([
            'status' => 'running',
            'started_at' => now()
        ]);
        
        try {
            // Run scanners
            $context = [
                'email_mode' => $domain->email_mode,
                'dkim_selector' => $domain->dkim_selector
            ];
            
            $results = [
                'security_headers' => $headersScanner->scan($domain->url, $context),
                'ssl_tls' => $sslScanner->scan($domain->url, $context),
                'dns_email' => $dnsScanner->scan($domain->url, $context)
            ];
            
            // Store module results
            foreach ($results as $module => $result) {
                $this->scan->modules()->create([
                    'module' => $module,
                    'score' => $result->score,
                    'status' => $result->status,
                    'raw' => $result->raw
                ]);
            }
            
            // Calculate total score
            $scores = collect($results)->pluck('score', 'module')->toArray();
            $scoring = $scorer->calculate($scores);
            
            // Update scan with results
            $this->scan->update([
                'status' => 'completed',
                'total_score' => $scoring['total'],
                'grade' => $scoring['grade'],
                'weights_used' => $scoring['weights_used'],
                'completed_at' => now()
            ]);
            
            // Update domain last scan info
            $domain->update([
                'last_scan_at' => now(),
                'last_scan_score' => $scoring['total']
            ]);
            
        } catch (\Exception $e) {
            $this->scan->update([
                'status' => 'failed',
                'error_message' => $e->getMessage(),
                'completed_at' => now()
            ]);
            
            throw $e;
        }
    }
    
    public function failed(\Throwable $exception): void {
        $this->scan->update([
            'status' => 'failed',
            'error_message' => $exception->getMessage()
        ]);
    }
}
```

## Scoring Engine

### Security Scorer Implementation
```php
namespace App\Services\Scoring;

class SecurityScorer {
    private array $config;
    
    public function __construct() {
        $this->config = config('scoring');
    }
    
    public function calculate(array $subscores): array {
        $weights = $this->config['weights'];
        
        // Get only the modules we have scores for
        $availableModules = array_intersect_key($weights, $subscores);
        
        // When scanner fails: Redistribute weights proportionally
        $totalWeight = array_sum($availableModules);
        $normalizedWeights = [];
        
        foreach ($availableModules as $module => $weight) {
            $normalizedWeights[$module] = $weight / $totalWeight;
        }
        
        // Calculate weighted score
        $totalScore = 0;
        foreach ($subscores as $module => $score) {
            $totalScore += ($normalizedWeights[$module] ?? 0) * $score;
        }
        
        $totalScore = (int) round(max(0, min(100, $totalScore)));
        
        return [
            'total' => $totalScore,
            'grade' => $this->calculateGrade($totalScore),
            'weights_used' => $normalizedWeights,
            'version' => $this->config['version']
        ];
    }
    
    private function calculateGrade(int $score): string {
        foreach ($this->config['grades'] as $gradeConfig) {
            if ($score >= $gradeConfig['min']) {
                return $gradeConfig['grade'];
            }
        }
        
        return 'F';
    }
}
```

## Stripe Integration

### Direct Stripe Service
```php
namespace App\Services;

use App\Models\User;
use Stripe\StripeClient;

class StripeService {
    private StripeClient $stripe;
    
    public function __construct() {
        $this->stripe = new StripeClient(config('stripe.secret'));
    }
    
    public function createCustomer(User $user): \Stripe\Customer {
        return $this->stripe->customers->create([
            'email' => $user->email,
            'metadata' => ['user_id' => $user->id]
        ]);
    }
    
    public function createSubscription(User $user, string $paymentMethodId): \Stripe\Subscription {
        // Attach payment method to customer
        $this->stripe->paymentMethods->attach($paymentMethodId, [
            'customer' => $user->stripe_customer_id
        ]);
        
        // Set as default payment method
        $this->stripe->customers->update($user->stripe_customer_id, [
            'invoice_settings' => ['default_payment_method' => $paymentMethodId]
        ]);
        
        // Create subscription
        return $this->stripe->subscriptions->create([
            'customer' => $user->stripe_customer_id,
            'items' => [['price' => config('stripe.price_id')]],
            'trial_end' => $user->trial_ends_at->timestamp,
            'expand' => ['latest_invoice.payment_intent']
        ]);
    }
    
    public function cancelSubscription(string $subscriptionId): \Stripe\Subscription {
        return $this->stripe->subscriptions->cancel($subscriptionId);
    }
}
```

## Laravel Package Implementations

### Laravel Cashier (Subscription Management)

```php
namespace App\Services;

use Laravel\Cashier\Cashier;

class BillingService {
    public function createSubscription(User $user, string $paymentMethodId) {
        // Create $27/month subscription with 14-day trial
        return $user->newSubscription('default', config('cashier.price_id'))
            ->trialDays(14)
            ->create($paymentMethodId, [
                'email' => $user->email,
                'metadata' => [
                    'user_id' => $user->id,
                    'plan' => 'professional',
                    'domains_limit' => 10
                ]
            ]);
    }
    
    public function swapPlan(User $user, string $newPriceId) {
        return $user->subscription('default')->swap($newPriceId);
    }
    
    public function resumeSubscription(User $user) {
        return $user->subscription('default')->resume();
    }
    
    public function cancelAtPeriodEnd(User $user) {
        return $user->subscription('default')->cancel();
    }
    
    public function handleWebhook(array $payload) {
        // Webhook handler for Stripe events
        switch($payload['type']) {
            case 'customer.subscription.deleted':
                $this->handleSubscriptionCancelled($payload);
                break;
            case 'customer.subscription.updated':
                $this->handleSubscriptionUpdated($payload);
                break;
            case 'invoice.payment_failed':
                $this->handlePaymentFailed($payload);
                break;
        }
    }
}
```

### Laravel Pennant (Feature Flags)

```php
namespace App\Providers;

use Laravel\Pennant\Feature;
use App\Models\User;

class AppServiceProvider extends ServiceProvider {
    public function boot() {
        // Define feature flags
        Feature::define('enhanced-scanners', function (User $user) {
            // Enable for paid users or beta testers
            return $user->subscribed('default') || 
                   in_array($user->email, config('features.beta_testers'));
        });
        
        Feature::define('ai-insights', function (User $user) {
            // Gradual rollout: 20% of users
            return crc32($user->id) % 100 < 20;
        });
        
        Feature::define('bulk-operations', function (User $user) {
            // Enable for users with 5+ domains
            return $user->domains()->count() >= 5;
        });
        
        Feature::define('api-access', function (User $user) {
            // API access for subscribed users only
            return $user->subscribed('default') && !$user->onTrial();
        });
    }
}

// Usage in controllers
class ScanController extends Controller {
    public function store(Domain $domain) {
        $scanners = [
            new SslTlsScanner(),
            new SecurityHeadersScanner(),
            new DnsEmailScanner(),
        ];
        
        // Add enhanced scanners if feature is active
        if (Feature::active('enhanced-scanners')) {
            $scanners[] = new VulnerabilityScanner();
            $scanners[] = new PerformanceScanner();
        }
        
        // Include AI insights if enabled
        if (Feature::active('ai-insights')) {
            $scanners[] = new AiInsightsScanner();
        }
        
        return $this->runScanners($domain, $scanners);
    }
}
```

### Laravel Precognition (Live Validation)

```php
// Backend: Controller with Precognition support
namespace App\Http\Controllers;

use App\Http\Requests\StoreDomainRequest;
use Illuminate\Foundation\Http\Middleware\HandlePrecognitiveRequests;

class DomainController extends Controller {
    public function __construct() {
        $this->middleware(HandlePrecognitiveRequests::class)->only(['store', 'update']);
    }
    
    public function store(StoreDomainRequest $request) {
        // Validation rules are in StoreDomainRequest
        // Precognition reuses these for live validation
        
        $domain = $request->user()->domains()->create([
            'url' => $request->validated()['url'],
            'email_mode' => $request->validated()['email_mode'],
            'dkim_selector' => $request->validated()['dkim_selector'] ?? null,
        ]);
        
        return redirect()->route('domains.show', $domain);
    }
}

// Form Request with complex validation
class StoreDomainRequest extends FormRequest {
    public function rules(): array {
        return [
            'url' => [
                'required',
                'url',
                'starts_with:https://',
                new PublicUrl(),
                new UniqueDomainPerUser($this->user()),
                function ($attribute, $value, $fail) {
                    // Custom validation: Check domain is reachable
                    if (!$this->isDomainReachable($value)) {
                        $fail('The domain is not reachable.');
                    }
                }
            ],
            'email_mode' => ['required', 'in:expected,none'],
            'dkim_selector' => ['nullable', 'string', 'max:50', 'regex:/^[a-zA-Z0-9_-]+$/'],
        ];
    }
    
    private function isDomainReachable(string $url): bool {
        try {
            $headers = @get_headers($url);
            return $headers && strpos($headers[0], '200') !== false;
        } catch (\Exception $e) {
            return false;
        }
    }
}
```

```typescript
// Frontend: React component with live validation
import { useForm } from 'laravel-precognition-react-inertia';

export function AddDomainForm() {
    const form = useForm('post', '/domains', {
        url: '',
        email_mode: 'expected',
        dkim_selector: '',
    });
    
    const submit = (e: React.FormEvent) => {
        e.preventDefault();
        form.submit({
            preserveScroll: true,
            onSuccess: () => form.reset(),
        });
    };
    
    return (
        <form onSubmit={submit}>
            <div>
                <Label htmlFor="url">Domain URL</Label>
                <Input
                    id="url"
                    type="url"
                    value={form.data.url}
                    onChange={(e) => form.setData('url', e.target.value)}
                    onBlur={() => form.validate('url')}
                    className={form.errors.url ? 'border-red-500' : ''}
                />
                {form.errors.url && (
                    <p className="text-red-500 text-sm mt-1">{form.errors.url}</p>
                )}
            </div>
            
            <div>
                <Label htmlFor="email_mode">Email Configuration</Label>
                <Select
                    value={form.data.email_mode}
                    onValueChange={(value) => form.setData('email_mode', value)}
                    onBlur={() => form.validate('email_mode')}
                >
                    <SelectItem value="expected">Expected (SPF/DKIM/DMARC)</SelectItem>
                    <SelectItem value="none">No Email</SelectItem>
                </Select>
            </div>
            
            <Button type="submit" disabled={form.processing}>
                {form.processing ? 'Adding...' : 'Add Domain'}
            </Button>
        </form>
    );
}
```

### Laravel Socialite (OAuth Authentication)

```php
namespace App\Http\Controllers\Auth;

use Laravel\Socialite\Facades\Socialite;
use App\Models\User;

class SocialAuthController extends Controller {
    protected array $providers = ['github', 'google'];
    
    public function redirect(string $provider) {
        if (!in_array($provider, $this->providers)) {
            abort(404);
        }
        
        return Socialite::driver($provider)
            ->scopes($this->getScopes($provider))
            ->redirect();
    }
    
    public function callback(string $provider) {
        if (!in_array($provider, $this->providers)) {
            abort(404);
        }
        
        try {
            $socialUser = Socialite::driver($provider)->user();
        } catch (\Exception $e) {
            return redirect('/login')->with('error', 'Authentication failed');
        }
        
        $user = $this->findOrCreateUser($socialUser, $provider);
        
        Auth::login($user, true);
        
        // Check if new user needs onboarding
        if ($user->wasRecentlyCreated) {
            return redirect()->route('onboarding.start');
        }
        
        return redirect()->intended('/dashboard');
    }
    
    protected function findOrCreateUser($socialUser, string $provider): User {
        // Try to find existing user by provider
        $user = User::where('provider', $provider)
            ->where('provider_id', $socialUser->getId())
            ->first();
        
        if ($user) {
            return $user;
        }
        
        // Try to find by email (link accounts)
        $user = User::where('email', $socialUser->getEmail())->first();
        
        if ($user) {
            // Link OAuth to existing account
            $user->update([
                'provider' => $provider,
                'provider_id' => $socialUser->getId(),
            ]);
            return $user;
        }
        
        // Create new user
        return User::create([
            'name' => $socialUser->getName() ?? $socialUser->getNickname(),
            'email' => $socialUser->getEmail(),
            'provider' => $provider,
            'provider_id' => $socialUser->getId(),
            'email_verified_at' => now(),
            'trial_ends_at' => now()->addDays(14),
            'subscription_status' => 'trialing',
            'onboarding_progress' => json_encode(['step' => 'welcome']),
        ]);
    }
    
    protected function getScopes(string $provider): array {
        return match($provider) {
            'github' => ['read:user', 'user:email'],
            'google' => ['openid', 'profile', 'email'],
            default => []
        };
    }
}
```

## Trial & Subscription Enforcement

### Subscription Middleware

```php
namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Auth;

class EnsureSubscriptionActive {
    /**
     * Routes that are accessible during trial
     */
    protected array $trialAllowedRoutes = [
        'dashboard.index',
        'profile.edit',
        'profile.update',
        'billing.index',
        'billing.subscribe',
        'stripe.checkout',
        'stripe.success',
        'domains.index', // View domains only
    ];
    
    /**
     * Routes that require active subscription or trial
     */
    protected array $protectedRoutes = [
        'domains.store',
        'domains.scan',
        'scans.*',
        'reports.*',
    ];
    
    public function handle(Request $request, Closure $next) {
        $user = Auth::user();
        
        if (!$user) {
            return redirect()->route('login');
        }
        
        // Check subscription status
        $subscriptionStatus = $this->getSubscriptionStatus($user);
        
        // Store status in request for use in views
        $request->attributes->set('subscription_status', $subscriptionStatus);
        
        // Handle based on status
        switch ($subscriptionStatus) {
            case 'active':
                return $next($request);
                
            case 'trial':
                // Allow all features during trial
                if ($user->trial_ends_at->isFuture()) {
                    return $next($request);
                }
                // Trial expired, fall through to expired case
                
            case 'expired':
            case 'cancelled':
                // Check if accessing protected route
                if ($this->isProtectedRoute($request)) {
                    return redirect()
                        ->route('billing.index')
                        ->with('error', 'Your trial has expired. Please subscribe to continue using Achilleus.');
                }
                // Allow access to billing and basic pages
                return $next($request);
                
            case 'grace_period':
                // Payment failed but within grace period
                if ($user->subscription_grace_ends_at->isFuture()) {
                    $request->session()->flash('warning', 'Payment failed. Please update your payment method.');
                    return $next($request);
                }
                // Grace period expired, restrict access
                if ($this->isProtectedRoute($request)) {
                    return redirect()
                        ->route('billing.index')
                        ->with('error', 'Your subscription has been suspended due to payment failure.');
                }
                return $next($request);
                
            default:
                // No subscription, redirect to billing
                if ($this->isProtectedRoute($request)) {
                    return redirect()
                        ->route('billing.index')
                        ->with('info', 'Please subscribe to access this feature.');
                }
                return $next($request);
        }
    }
    
    protected function getSubscriptionStatus($user): string {
        // Active subscription
        if ($user->subscription_status === 'active' && 
            $user->subscription_ends_at === null) {
            return 'active';
        }
        
        // Cancelled but still active until end date
        if ($user->subscription_status === 'cancelled' && 
            $user->subscription_ends_at && 
            $user->subscription_ends_at->isFuture()) {
            return 'active';
        }
        
        // In trial period
        if ($user->trial_ends_at && $user->trial_ends_at->isFuture()) {
            return 'trial';
        }
        
        // Grace period for failed payments
        if ($user->subscription_grace_ends_at && 
            $user->subscription_grace_ends_at->isFuture()) {
            return 'grace_period';
        }
        
        // Trial expired
        if ($user->trial_ends_at && $user->trial_ends_at->isPast()) {
            return 'expired';
        }
        
        // Subscription cancelled and expired
        if ($user->subscription_status === 'cancelled' && 
            $user->subscription_ends_at && 
            $user->subscription_ends_at->isPast()) {
            return 'cancelled';
        }
        
        return 'none';
    }
    
    protected function isProtectedRoute(Request $request): bool {
        $routeName = $request->route()->getName();
        
        foreach ($this->protectedRoutes as $pattern) {
            if (fnmatch($pattern, $routeName)) {
                return true;
            }
        }
        
        return false;
    }
}
```

### API Subscription Enforcement

```php
namespace App\Http\Middleware;

class EnsureApiSubscriptionActive {
    public function handle(Request $request, Closure $next) {
        $user = $request->user();
        
        // Check if user has active subscription or valid trial
        if (!$this->hasActiveSubscription($user)) {
            return response()->json([
                'error' => 'Subscription required',
                'message' => 'Your trial has expired. Please subscribe to continue.',
                'subscription_url' => route('billing.index')
            ], 402); // Payment Required
        }
        
        // Add subscription info to response headers
        $response = $next($request);
        
        if ($user->trial_ends_at) {
            $daysRemaining = now()->diffInDays($user->trial_ends_at, false);
            $response->header('X-Trial-Days-Remaining', max(0, $daysRemaining));
        }
        
        $response->header('X-Subscription-Status', $user->subscription_status);
        
        return $response;
    }
    
    protected function hasActiveSubscription($user): bool {
        // Active subscription
        if ($user->subscription_status === 'active') {
            return true;
        }
        
        // Valid trial
        if ($user->trial_ends_at && $user->trial_ends_at->isFuture()) {
            return true;
        }
        
        // Grace period
        if ($user->subscription_grace_ends_at && 
            $user->subscription_grace_ends_at->isFuture()) {
            return true;
        }
        
        return false;
    }
}
```

### Route Protection Configuration

```php
// routes/web.php
Route::middleware(['auth', 'verified'])->group(function () {
    // Always accessible routes
    Route::get('/dashboard', [DashboardController::class, 'index'])->name('dashboard.index');
    Route::get('/billing', [BillingController::class, 'index'])->name('billing.index');
    Route::post('/billing/subscribe', [BillingController::class, 'subscribe'])->name('billing.subscribe');
    
    // Protected routes - require active subscription
    Route::middleware(['subscription'])->group(function () {
        // Domain management
        Route::post('/domains', [DomainController::class, 'store'])->name('domains.store');
        Route::post('/domains/{domain}/scan', [DomainScanController::class, 'store'])->name('domains.scan');
        Route::delete('/domains/{domain}', [DomainController::class, 'destroy'])->name('domains.destroy');
        
        // Scans
        Route::get('/scans/{scan}', [ScanController::class, 'show'])->name('scans.show');
        
        // Reports
        Route::post('/reports', [ReportController::class, 'store'])->name('reports.store');
        Route::get('/reports/{report}/download', [ReportController::class, 'download'])->name('reports.download');
    });
});

// routes/api.php
Route::middleware(['auth:sanctum'])->group(function () {
    Route::middleware(['api.subscription'])->group(function () {
        Route::apiResource('domains', Api\DomainController::class);
        Route::post('domains/{domain}/scan', [Api\DomainScanController::class, 'store']);
        Route::apiResource('scans', Api\ScanController::class)->only(['index', 'show']);
        Route::apiResource('reports', Api\ReportController::class);
    });
});
```

### Trial Expiry Banner Component

```php
// resources/views/components/trial-banner.blade.php
@props(['user'])

@php
    $daysRemaining = $user->trial_ends_at ? now()->diffInDays($user->trial_ends_at, false) : null;
    $isExpired = $daysRemaining !== null && $daysRemaining <= 0;
    $isWarning = $daysRemaining !== null && $daysRemaining <= 3 && $daysRemaining > 0;
@endphp

@if($user->trial_ends_at && !$user->subscription_status)
    <div class="relative">
        @if($isExpired)
            <div class="bg-red-900/20 border border-red-900/50 text-red-400 px-4 py-3 rounded-lg mb-4">
                <div class="flex items-center justify-between">
                    <div class="flex items-center">
                        <svg class="w-5 h-5 mr-2" fill="currentColor" viewBox="0 0 20 20">
                            <path fill-rule="evenodd" d="M10 18a8 8 0 100-16 8 8 0 000 16zM8.707 7.293a1 1 0 00-1.414 1.414L8.586 10l-1.293 1.293a1 1 0 101.414 1.414L10 11.414l1.293 1.293a1 1 0 001.414-1.414L11.414 10l1.293-1.293a1 1 0 00-1.414-1.414L10 8.586 8.707 7.293z" clip-rule="evenodd"/>
                        </svg>
                        <span class="font-medium">Your trial has expired</span>
                        <span class="ml-2">Subscribe now to continue monitoring your domains</span>
                    </div>
                    <a href="{{ route('billing.index') }}" class="bg-green-600 hover:bg-green-700 text-white font-bold py-2 px-4 rounded transition">
                        Subscribe Now
                    </a>
                </div>
            </div>
        @elseif($isWarning)
            <div class="bg-yellow-900/20 border border-yellow-900/50 text-yellow-400 px-4 py-3 rounded-lg mb-4">
                <div class="flex items-center justify-between">
                    <div class="flex items-center">
                        <svg class="w-5 h-5 mr-2" fill="currentColor" viewBox="0 0 20 20">
                            <path fill-rule="evenodd" d="M8.257 3.099c.765-1.36 2.722-1.36 3.486 0l5.58 9.92c.75 1.334-.213 2.98-1.742 2.98H4.42c-1.53 0-2.493-1.646-1.743-2.98l5.58-9.92zM11 13a1 1 0 11-2 0 1 1 0 012 0zm-1-8a1 1 0 00-1 1v3a1 1 0 002 0V6a1 1 0 00-1-1z" clip-rule="evenodd"/>
                        </svg>
                        <span class="font-medium">{{ $daysRemaining }} {{ Str::plural('day', $daysRemaining) }} left in trial</span>
                        <span class="ml-2">Subscribe now to ensure uninterrupted service</span>
                    </div>
                    <a href="{{ route('billing.index') }}" class="bg-green-600 hover:bg-green-700 text-white font-bold py-2 px-4 rounded transition">
                        Subscribe Now
                    </a>
                </div>
            </div>
        @else
            <div class="bg-blue-900/20 border border-blue-900/50 text-blue-400 px-4 py-3 rounded-lg mb-4">
                <div class="flex items-center justify-between">
                    <div class="flex items-center">
                        <svg class="w-5 h-5 mr-2" fill="currentColor" viewBox="0 0 20 20">
                            <path fill-rule="evenodd" d="M18 10a8 8 0 11-16 0 8 8 0 0116 0zm-7-4a1 1 0 11-2 0 1 1 0 012 0zM9 9a1 1 0 000 2v3a1 1 0 001 1h1a1 1 0 100-2v-3a1 1 0 00-1-1H9z" clip-rule="evenodd"/>
                        </svg>
                        <span>{{ $daysRemaining }} {{ Str::plural('day', $daysRemaining) }} remaining in your free trial</span>
                    </div>
                    <a href="{{ route('billing.index') }}" class="text-blue-400 hover:text-blue-300 underline">
                        View Plans
                    </a>
                </div>
            </div>
        @endif
    </div>
@endif
```

### Database Schema for Subscription Tracking

```sql
-- Add subscription tracking columns to users table
ALTER TABLE users ADD COLUMN subscription_status VARCHAR(20) DEFAULT 'trial';
ALTER TABLE users ADD COLUMN subscription_ends_at TIMESTAMP NULL;
ALTER TABLE users ADD COLUMN subscription_grace_ends_at TIMESTAMP NULL;
ALTER TABLE users ADD COLUMN stripe_subscription_id VARCHAR(255) NULL;
ALTER TABLE users ADD COLUMN last_payment_failed_at TIMESTAMP NULL;
ALTER TABLE users ADD COLUMN payment_failure_count INT DEFAULT 0;

-- Index for efficient subscription queries
CREATE INDEX idx_users_subscription ON users(subscription_status, trial_ends_at, subscription_ends_at);
```

## Laravel Cloud Configuration

### Laravel Cloud Deployment (Cloud Only - No Self-Hosting)
**Laravel Cloud Platform**: https://cloud.laravel.com/
- No configuration files needed - managed through UI
- Automatic SSL/TLS certificate provisioning
- Push-to-deploy with GitHub integration
- Auto-hibernation for cost optimization
- Global edge network included
- Laravel Reverb for WebSocket connections

### Deployment Process
```bash
# 1. Connect GitHub repository to Laravel Cloud (https://cloud.laravel.com/)
# 2. Laravel Cloud auto-detects Laravel 12 project
# 3. Configure environment variables via UI:
#    - STRIPE_KEY, STRIPE_SECRET, STRIPE_WEBHOOK_SECRET
#    - APP_ENV=production, APP_DEBUG=false
#    - REVERB_APP_ID, REVERB_APP_KEY, REVERB_APP_SECRET (for real-time)
# 4. Laravel Cloud automatically provisions:
#    - Serverless PostgreSQL database
#    - Redis-compatible cache
#    - S3-compatible storage
#    - Queue workers with auto-scaling (Laravel Cloud managed)
#    - Laravel Reverb WebSocket server
# 5. Deploy with one click
```

### Auto-Managed Services
```yaml
# These are automatically configured by Laravel Cloud:
database:
  type: serverless-postgres
  auto_hibernation: true
  connection_pooling: enabled
  
cache:
  type: redis-compatible
  auto_scaling: true
  
storage:
  type: s3-compatible
  automatic_backups: true
  
# Laravel Cloud handles queue configuration automatically
  type: managed_workers
  auto_scaling: 0-10_workers
  cost_optimization: enabled

network:
  type: edge_network
  ssl_certificates: automatic
  ddos_protection: included
  global_cdn: enabled

scaling:
  min_replicas: 1
  max_replicas: 10
  hibernation_after: 30_minutes
  cost_optimization: automatic
```

## Performance Best Practices

### Query Optimization (Application Level)
```php
// Eager load relationships to prevent N+1 queries
$domains = auth()->user()
    ->domains()
    ->with(['lastScan.modules'])
    ->where('is_active', true)
    ->paginate(10);

// Use database indexes (defined in migrations)
// Laravel Cloud handles query optimization and connection pooling
```

### Caching (Handled by Laravel Cloud)
Laravel Cloud provides Redis caching automatically. Simple usage:
```php
// Dashboard metrics with 15-minute cache
Cache::remember("dashboard:{$user->id}", 900, fn() => $dashboardData);
```

## Error Handling Standards

### Error Handling Architecture

#### Exception Hierarchy
```php
namespace App\Exceptions;

// Application-level exceptions (separate from scanner exceptions)
class AchilleusException extends \Exception {}
class ValidationException extends AchilleusException {}
class DomainLimitException extends ValidationException {}
class PaymentFailedException extends AchilleusException {}
class SubscriptionExpiredException extends AchilleusException {}
```

#### Global Error Handler
```php
namespace App\Exceptions;

class Handler extends ExceptionHandler {
    public function render($request, Throwable $e) {
        // API responses
        if ($request->expectsJson()) {
            return $this->handleApiException($e);
        }
        
        // Inertia responses
        if ($e instanceof ValidationException) {
            return back()->withErrors($e->errors());
        }
        
        // Security exceptions - log but don't expose details
        if ($e instanceof SecurityException) {
            Log::warning('Security exception', [
                'type' => get_class($e),
                'message' => $e->getMessage(),
                'user' => auth()->id(),
                'ip' => request()->ip()
            ]);
            
            return inertia('Error', [
                'status' => 403,
                'message' => 'Security validation failed'
            ]);
        }
        
        // Scanner exceptions - user-friendly messages
        if ($e instanceof ScannerException) {
            return inertia('Error', [
                'status' => 500,
                'message' => 'Scan failed. Please try again.',
                'retry' => true
            ]);
        }
        
        return parent::render($request, $e);
    }
    
    private function handleApiException(Throwable $e): JsonResponse {
        $status = 500;
        $message = 'Server error';
        
        if ($e instanceof ValidationException) {
            $status = 422;
            $message = $e->getMessage();
        } elseif ($e instanceof SecurityException) {
            $status = 403;
            $message = 'Forbidden';
        } elseif ($e instanceof ScannerException) {
            $status = 503;
            $message = 'Service temporarily unavailable';
        }
        
        return response()->json([
            'error' => $message,
            'type' => class_basename($e)
        ], $status);
    }
}
```

#### Controller Error Handling
```php
class DomainController extends Controller {
    public function store(StoreDomainRequest $request) {
        try {
            // Check domain limit
            if (auth()->user()->domains()->count() >= 10) {
                throw new DomainLimitException('Domain limit reached');
            }
            
            $domain = auth()->user()->domains()->create($request->validated());
            
            // Trigger scan
            RunDomainScan::dispatch($domain);
            
            return redirect()->route('domains.show', $domain)
                ->with('success', 'Domain added successfully');
                
        } catch (DomainLimitException $e) {
            return back()->with('error', 'You have reached the 10 domain limit');
        } catch (\Exception $e) {
            Log::error('Domain creation failed', [
                'error' => $e->getMessage(),
                'user' => auth()->id()
            ]);
            
            return back()->with('error', 'Failed to add domain. Please try again.');
        }
    }
}
```

#### Service Layer Error Handling
```php
class ScanOrchestrator {
    public function executeScan(Domain $domain): Scan {
        $scan = $domain->scans()->create(['status' => 'running']);
        
        try {
            $results = [];
            
            // Run scanners with individual error handling
            foreach ($this->scanners as $scanner) {
                try {
                    $results[$scanner->name()] = $scanner->scan($domain->url);
                } catch (RetryableException $e) {
                    Log::warning("Scanner retryable error", [
                        'scanner' => $scanner->name(),
                        'domain' => $domain->url,
                        'error' => $e->getMessage()
                    ]);
                    $results[$scanner->name()] = ModuleResult::error($scanner->name(), $e->getMessage());
                } catch (\Exception $e) {
                    Log::error("Scanner failed", [
                        'scanner' => $scanner->name(),
                        'error' => $e->getMessage()
                    ]);
                    $results[$scanner->name()] = ModuleResult::error($scanner->name());
                }
            }
            
            // Calculate score even with failed modules
            $scoring = $this->scorer->calculate($results);
            
            $scan->update([
                'status' => 'completed',
                'total_score' => $scoring['total'],
                'grade' => $scoring['grade']
            ]);
            
            return $scan;
            
        } catch (\Exception $e) {
            $scan->update([
                'status' => 'failed',
                'error_message' => $e->getMessage()
            ]);
            
            throw new ScannerException('Scan orchestration failed', 0, $e);
        }
    }
}
```

#### Frontend Error Handling (React)
```typescript
// Error boundary component
export function ErrorBoundary({ children }: { children: React.ReactNode }) {
    return (
        <ErrorBoundaryComponent
            fallback={<ErrorFallback />}
            onError={(error, errorInfo) => {
                console.error('Error caught by boundary:', error, errorInfo);
            }}
        >
            {children}
        </ErrorBoundaryComponent>
    );
}

// Form error handling
const form = useForm({
    onError: (errors) => {
        if (errors.domain_limit) {
            toast.error('You have reached the 10 domain limit');
        } else if (errors.url) {
            toast.error('Invalid domain URL');
        } else {
            toast.error('An error occurred. Please try again.');
        }
    }
});

// API error handling
const handleScan = async (domainId: string) => {
    try {
        await axios.post(`/domains/${domainId}/scan`);
        toast.success('Scan started');
    } catch (error) {
        if (error.response?.status === 429) {
            toast.error('Rate limit exceeded. Please wait.');
        } else if (error.response?.status === 403) {
            toast.error('Access denied');
        } else {
            toast.error('Scan failed. Please try again.');
        }
    }
};
```

#### User-Friendly Error Messages
```php
// Error message mapping
class ErrorMessages {
    private static $messages = [
        // Scanner exceptions (simplified to 3 types)
        RetryableException::class => 'A temporary issue occurred. Please try again.',
        ConfigurationException::class => 'There is a configuration issue with this domain. Please check your setup.',
        SecurityException::class => 'Access to this resource was blocked for security reasons.',
        
        // Application exceptions
        DomainLimitException::class => 'You have reached your 10 domain limit. Please remove a domain to add a new one.',
        PaymentFailedException::class => 'Payment processing failed. Please check your payment method.',
        SubscriptionExpiredException::class => 'Your subscription has expired. Please renew to continue.'
    ];
    
    public static function get(Throwable $e): string {
        $class = get_class($e);
        return self::$messages[$class] ?? 'An unexpected error occurred. Please try again.';
    }
}
```

#### Logging Standards
```php
// Structured logging for errors
Log::channel('errors')->error('Critical error', [
    'exception' => get_class($e),
    'message' => $e->getMessage(),
    'trace' => $e->getTraceAsString(),
    'user_id' => auth()->id(),
    'url' => request()->fullUrl(),
    'ip' => request()->ip(),
    'user_agent' => request()->userAgent()
]);

// Security event logging
Log::channel('security')->warning('Security violation', [
    'type' => 'ssrf_attempt',
    'url' => $url,
    'user_id' => auth()->id(),
    'timestamp' => now()
]);

// Performance issue logging
Log::channel('performance')->warning('Slow operation', [
    'operation' => 'scan',
    'duration' => $duration,
    'threshold' => 30,
    'domain' => $domain->url
]);
```

## GDPR Compliance Implementation

### Database Schema for GDPR Compliance

#### User Consent Tracking
```sql
-- User consent records table
CREATE TABLE user_consents (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    consent_type VARCHAR(50) NOT NULL, -- 'privacy_policy', 'terms_of_service', 'marketing_emails', 'cookies'
    consented BOOLEAN NOT NULL DEFAULT false,
    ip_address INET,
    user_agent TEXT,
    consented_at TIMESTAMP NOT NULL,
    withdrawn_at TIMESTAMP NULL,
    version VARCHAR(20) NOT NULL, -- Version of document consented to
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_user_consent (user_id, consent_type),
    INDEX idx_consent_date (consented_at)
);

-- Data processing activities log
CREATE TABLE data_processing_logs (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    activity_type VARCHAR(100) NOT NULL, -- 'export', 'deletion', 'rectification', 'access'
    status VARCHAR(20) NOT NULL, -- 'pending', 'processing', 'completed', 'failed'
    requested_at TIMESTAMP NOT NULL,
    completed_at TIMESTAMP NULL,
    requested_by VARCHAR(255), -- Could be user or admin
    ip_address INET,
    details JSONB, -- Additional metadata about the request
    file_path TEXT, -- For export requests
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_user_activity (user_id, activity_type),
    INDEX idx_activity_status (status)
);

-- Audit trail for all data changes
CREATE TABLE gdpr_audit_logs (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL,
    table_name VARCHAR(100) NOT NULL,
    record_id BIGINT NOT NULL,
    action VARCHAR(20) NOT NULL, -- 'create', 'update', 'delete', 'access'
    old_data JSONB,
    new_data JSONB,
    changed_by VARCHAR(255),
    reason TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_user_audit (user_id),
    INDEX idx_table_record (table_name, record_id)
);
```

### GDPR Service Implementation

```php
namespace App\Services;

use App\Models\User;
use App\Models\UserConsent;
use App\Models\DataProcessingLog;
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Storage;

class GdprService {
    /**
     * Record user consent
     */
    public function recordConsent(User $user, string $type, bool $consented, string $version): UserConsent {
        return UserConsent::create([
            'user_id' => $user->id,
            'consent_type' => $type,
            'consented' => $consented,
            'ip_address' => request()->ip(),
            'user_agent' => request()->userAgent(),
            'consented_at' => now(),
            'version' => $version
        ]);
    }
    
    /**
     * Withdraw consent
     */
    public function withdrawConsent(User $user, string $type): void {
        UserConsent::where('user_id', $user->id)
            ->where('consent_type', $type)
            ->whereNull('withdrawn_at')
            ->update(['withdrawn_at' => now()]);
    }
    
    /**
     * Export all user data (GDPR Article 20 - Data Portability)
     */
    public function exportUserData(User $user): string {
        $log = DataProcessingLog::create([
            'user_id' => $user->id,
            'activity_type' => 'export',
            'status' => 'processing',
            'requested_at' => now(),
            'requested_by' => $user->email,
            'ip_address' => request()->ip()
        ]);
        
        try {
            $data = [
                'user' => $user->toArray(),
                'domains' => $user->domains()->with('scans.scanModules')->get()->toArray(),
                'scans' => $user->scans()->with('scanModules')->get()->toArray(),
                'reports' => $user->reports()->get()->toArray(),
                'consents' => $user->consents()->get()->toArray(),
                'audit_logs' => DB::table('gdpr_audit_logs')
                    ->where('user_id', $user->id)
                    ->get()
                    ->toArray()
            ];
            
            $filename = "user-data-export-{$user->id}-" . now()->format('Y-m-d-His') . '.json';
            $path = "gdpr-exports/{$filename}";
            
            Storage::disk('s3')->put($path, json_encode($data, JSON_PRETTY_PRINT));
            
            $log->update([
                'status' => 'completed',
                'completed_at' => now(),
                'file_path' => $path
            ]);
            
            return Storage::disk('s3')->temporaryUrl($path, now()->addHours(24));
            
        } catch (\Exception $e) {
            $log->update([
                'status' => 'failed',
                'details' => ['error' => $e->getMessage()]
            ]);
            throw $e;
        }
    }
    
    /**
     * Delete all user data (GDPR Article 17 - Right to Erasure)
     */
    public function deleteUserData(User $user): void {
        $log = DataProcessingLog::create([
            'user_id' => $user->id,
            'activity_type' => 'deletion',
            'status' => 'processing',
            'requested_at' => now(),
            'requested_by' => $user->email,
            'ip_address' => request()->ip()
        ]);
        
        DB::transaction(function () use ($user, $log) {
            try {
                // Delete all related data
                $user->domains()->delete();
                $user->scans()->delete();
                $user->reports()->delete();
                $user->consents()->delete();
                
                // Anonymize audit logs instead of deleting
                DB::table('gdpr_audit_logs')
                    ->where('user_id', $user->id)
                    ->update([
                        'user_id' => 0,
                        'changed_by' => 'ANONYMIZED',
                        'old_data' => DB::raw("'{}'::jsonb"),
                        'new_data' => DB::raw("'{}'::jsonb")
                    ]);
                
                // Delete the user account
                $user->delete();
                
                $log->update([
                    'status' => 'completed',
                    'completed_at' => now()
                ]);
                
            } catch (\Exception $e) {
                $log->update([
                    'status' => 'failed',
                    'details' => ['error' => $e->getMessage()]
                ]);
                throw $e;
            }
        });
    }
    
    /**
     * Check if user has given consent
     */
    public function hasConsent(User $user, string $type): bool {
        return UserConsent::where('user_id', $user->id)
            ->where('consent_type', $type)
            ->where('consented', true)
            ->whereNull('withdrawn_at')
            ->exists();
    }
}
```

### Data Retention Policies

```php
namespace App\Console\Commands;

use Illuminate\Console\Command;
use App\Models\Scan;
use App\Models\Report;
use App\Models\GdprAuditLog;

class CleanupOldData extends Command {
    protected $signature = 'gdpr:cleanup';
    protected $description = 'Clean up old data according to GDPR retention policies';
    
    public function handle(): void {
        // Delete scan data older than 90 days
        Scan::where('created_at', '<', now()->subDays(90))
            ->each(function ($scan) {
                $scan->scanModules()->delete();
                $scan->delete();
            });
        
        // Delete reports older than 90 days
        Report::where('created_at', '<', now()->subDays(90))
            ->each(function ($report) {
                Storage::disk('s3')->delete($report->file_path);
                $report->delete();
            });
        
        // Anonymize audit logs older than 2 years
        GdprAuditLog::where('created_at', '<', now()->subYears(2))
            ->update([
                'user_id' => 0,
                'changed_by' => 'ANONYMIZED',
                'old_data' => '{}',
                'new_data' => '{}'
            ]);
        
        $this->info('GDPR cleanup completed');
    }
}

// Schedule in app/Console/Kernel.php
protected function schedule(Schedule $schedule) {
    $schedule->command('gdpr:cleanup')->daily()->at('02:00');
}
```

### Cookie Consent Implementation

```php
namespace App\Http\Middleware;

class CookieConsent {
    public function handle($request, $next) {
        $response = $next($request);
        
        // Check if user has accepted cookies
        if (!$request->hasCookie('cookie_consent')) {
            // Inject cookie banner JavaScript
            $response->header('X-Cookie-Consent-Required', 'true');
        }
        
        return $response;
    }
}
```

### GDPR API Endpoints

```php
namespace App\Http\Controllers\Api;

use App\Services\GdprService;

class GdprController extends Controller {
    protected GdprService $gdprService;
    
    public function __construct(GdprService $gdprService) {
        $this->gdprService = $gdprService;
    }
    
    /**
     * Record user consent
     * POST /api/gdpr/consent
     */
    public function recordConsent(Request $request): JsonResponse {
        $validated = $request->validate([
            'consent_type' => 'required|in:privacy_policy,terms_of_service,marketing_emails,cookies',
            'consented' => 'required|boolean',
            'version' => 'required|string'
        ]);
        
        $consent = $this->gdprService->recordConsent(
            auth()->user(),
            $validated['consent_type'],
            $validated['consented'],
            $validated['version']
        );
        
        return response()->json(['consent' => $consent]);
    }
    
    /**
     * Export user data
     * POST /api/gdpr/export
     */
    public function exportData(): JsonResponse {
        $url = $this->gdprService->exportUserData(auth()->user());
        
        // Send email with download link
        Mail::to(auth()->user())->send(new DataExportReady($url));
        
        return response()->json([
            'message' => 'Your data export is being prepared. You will receive an email with the download link.'
        ]);
    }
    
    /**
     * Delete user account and all data
     * DELETE /api/gdpr/delete-account
     */
    public function deleteAccount(Request $request): JsonResponse {
        $request->validate([
            'password' => 'required|current_password',
            'confirmation' => 'required|in:DELETE'
        ]);
        
        $this->gdprService->deleteUserData(auth()->user());
        
        return response()->json([
            'message' => 'Your account and all associated data has been permanently deleted.'
        ]);
    }
    
    /**
     * Get privacy settings
     * GET /api/gdpr/privacy-settings
     */
    public function getPrivacySettings(): JsonResponse {
        $user = auth()->user();
        
        return response()->json([
            'consents' => [
                'privacy_policy' => $this->gdprService->hasConsent($user, 'privacy_policy'),
                'terms_of_service' => $this->gdprService->hasConsent($user, 'terms_of_service'),
                'marketing_emails' => $this->gdprService->hasConsent($user, 'marketing_emails'),
                'cookies' => $this->gdprService->hasConsent($user, 'cookies')
            ],
            'data_retention' => [
                'scan_data' => '90 days',
                'reports' => '90 days',
                'audit_logs' => '2 years (anonymized)',
                'account_data' => 'Until deletion requested'
            ]
        ]);
    }
}
```

### Privacy Policy Requirements

The privacy policy must include:
1. **Data Controller Information**: Company name, address, contact details
2. **Data Processing Purposes**: Security scanning, performance monitoring, reporting
3. **Legal Basis**: Legitimate interest (security), contract fulfillment (service provision)
4. **Data Categories**: Domain URLs, scan results, email addresses, IP addresses
5. **Data Retention**: 90 days for scan data, 2 years for anonymized logs
6. **Third Parties**: Stripe (payments), AWS (infrastructure), no data selling
7. **User Rights**: Access, rectification, erasure, portability, restriction, objection
8. **Data Transfers**: EU/US data transfers under standard contractual clauses
9. **Security Measures**: Encryption at rest and in transit, access controls
10. **Contact Information**: Data Protection Officer email (privacy@achilleus.so)

## Security Measures

### Input Validation
```php
namespace App\Http\Requests;

class StoreDomainRequest extends FormRequest {
    public function rules(): array {
        return [
            'url' => [
                'required',
                'url',
                'starts_with:https://',
                new PublicUrl(),
                new UniqueDomainForUser(auth()->id())
            ],
            'email_mode' => ['required', 'in:expected,none'],
            'dkim_selector' => ['nullable', 'string', 'max:50', 'alpha_dash']
        ];
    }
}
```

### Rate Limiting (Two-Tier System)
```php
// User-level rate limiting in RouteServiceProvider
RateLimiter::for('scans', function (Request $request) {
    return Limit::perMinute(10)->by($request->user()->id);
});

RateLimiter::for('api', function (Request $request) {
    return Limit::perMinute(60)->by($request->user()->id);
});

RateLimiter::for('reports', function (Request $request) {
    return Limit::perDay(10)->by($request->user()->id);
});

// Applied to routes
Route::post('/domains/{domain}/scan', [DomainScanController::class, 'store'])
    ->middleware('throttle:scans');

Route::post('/reports', [ReportController::class, 'store'])
    ->middleware('throttle:reports');
```

**Scanner-level rate limiting (per host):**
- SSL/TLS Scanner: 5 requests/minute per host
- Security Headers Scanner: 8 requests/minute per host  
- DNS/Email Scanner: 15 requests/minute per host
- Default (AbstractScanner): 10 requests/minute per host

## Performance Targets

- Scan completion: <15 seconds
- Dashboard load: <500ms
- PDF generation: <5 seconds
- Queue processing: <30 second latency
- Database queries: <50ms
- Page bundle size: <200KB

## PDF Report Generation Details

### Report Structure
```php
// Page 1: Executive Summary
- Overall score and grade
- Score breakdown by module
- Critical issues count
- Scan date and domain

// Page 2: Detailed Findings
- SSL/TLS analysis details
- Security headers table
- DNS/Email configuration status

// Page 3: Recommendations
- Prioritized fixes (critical first)
- How-to instructions
- Resource links
```

### DomPDF Configuration
```php
// Configuration
- A4 portrait orientation
- Achilleus branding
- Store in S3 after generation
- Signed URLs for download
```

## Development Tools & Packages

### Required Packages
```bash
# Core packages
composer require barryvdh/laravel-dompdf stripe/stripe-php

# Development packages
composer require --dev laravel/pint pestphp/pest

# Optional packages (useful but not required for MVP)
composer require --dev laravel/pennant laravel/pulse
```

### Testing Tools
```bash
# E2E testing and debugging
npm install --save-dev @playwright/test playwright
```

### Laravel Cloud Managed Services
- ✅ **Database**: PostgreSQL 15 (auto-managed)
- ✅ **Cache**: Redis (auto-scaling)
- ✅ **Queue**: Workers (auto-scaling)
- ✅ **Storage**: S3 (auto-configured)
- ❌ **Horizon**: Not needed (Laravel Cloud handles queues)
- ❌ **Manual Redis setup**: Not needed (auto-provided)

### UI Theme Configuration with Shadcn/ui

**Component System:**
- **Base**: Shadcn/ui component library with React 19
- **Theming**: CSS variables with built-in dark mode support
- **Customization**: Tailwind CSS with custom color palette

**Color Palette (Shadcn/ui Compatible):**
```css
:root {
  --background: 0 0% 3.9%; /* #0a0a0b */
  --foreground: 0 0% 98%; /* #fafafa */
  --card: 0 0% 3.9%; /* #0a0a0b */
  --card-foreground: 0 0% 98%;
  --primary: 0 0% 9%; /* #171717 */
  --primary-foreground: 0 0% 98%;
  --secondary: 0 0% 14.9%; /* #262626 */
  --secondary-foreground: 0 0% 98%;
  --muted: 0 0% 14.9%; /* #262626 */
  --muted-foreground: 0 0% 63.9%; /* #a3a3a3 */
  --accent: 0 0% 14.9%; /* #262626 */
  --accent-foreground: 0 0% 98%;
  --destructive: 0 84.2% 60.2%; /* #ef4444 */
  --destructive-foreground: 0 0% 98%;
  --border: 0 0% 14.9%; /* #262626 */
  --input: 0 0% 14.9%; /* #262626 */
  --ring: 0 0% 83.1%; /* #d4d4d4 */
}
```

**Typography & Components:**
- **Font**: Inter via Tailwind CSS
- **Components**: Button, Card, Form, Input, Select, Badge, etc.
- **Forms**: React Hook Form + Zod validation integration
- **Icons**: Lucide React icons
- **Responsive**: Mobile-first design with Tailwind breakpoints

## UX Enhancement Components

### Trial Conversion System
Context-aware trial banner that adapts messaging based on user progress:
- High urgency: Trial expires in <3 days with no scans
- Medium urgency: 80%+ feature exploration
- Low urgency: Progress-based messaging with available slots

**Implementation**: TrialBanner component with dynamic messaging based on TrialProgress interface

### Real-Time Scan Progress System
WebSocket-based live progress tracking for scan operations:
- 4 scan steps: SSL/TLS, Security Headers, DNS/Email, Score Calculation
- Progress events: ScanStepStarted, ScanStepProgress, ScanStepCompleted, ScanCompleted, ScanFailed
- Visual progress bars with status indicators

**Implementation**: ScanProgress component using Laravel Echo WebSocket events
```

### Smart Empty States System
Context-aware empty states that adapt based on user trial status and progress. Provides clear actionable steps with value propositions for different page types (domains, scans, reports, dashboard).

### Security Score Explanation System
Educational tooltip component that explains security scores (A+ to F grades), module weights (SSL/TLS 40%, Headers 30%, DNS/Email 30%), and assessment factors. Uses Shadcn/ui Tooltip with detailed explanations.

### Quick Actions System
Context-aware action buttons throughout the app (dashboard, domains, activity, reports). Primary and secondary actions adapt based on user data and page context with loading states and disabled conditions.

### Backend Support for UX Enhancements
WebSocket events for real-time scan progress tracking, user progress analytics for trial conversion, and automated email sequences based on user behavior patterns.
## Summary

This technical documentation provides a comprehensive architecture for building Achilleus - a security monitoring SaaS platform with Laravel 12, React 19, and robust scanner implementations focusing on essential security analysis while maintaining production-ready code standards.