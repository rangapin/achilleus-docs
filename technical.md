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
```

## Database Design

### Schema Overview
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

// Scanner Exception Hierarchy
namespace App\Exceptions;

class ScannerException extends \Exception {}
class NetworkException extends ScannerException {}
class TimeoutException extends ScannerException {}
class RateLimitException extends ScannerException {}
class ValidationException extends ScannerException {}
class SSRFException extends ValidationException {}
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
use App\Exceptions\{NetworkException, TimeoutException, RateLimitException, SSRFException};
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
                // SSRF protection
                NetworkGuard::assertPublicHttpUrl($url);
                
                // Perform actual scan
                $result = $this->performScan($url, $context);
                $result->executionTime = microtime(true) - $startTime;
                $result->retryCount = $attempts - 1;
                
                // Update rate limit
                $this->updateRateLimit($url);
                
                return $result;
                
            } catch (SSRFException $e) {
                Log::warning('SSRF attempt blocked', [
                    'scanner' => $this->name(),
                    'url' => $url,
                    'error' => $e->getMessage()
                ]);
                
                return ModuleResult::error($this->name(), 'Invalid or unsafe URL', $attempts - 1);
                
            } catch (TimeoutException $e) {
                $executionTime = microtime(true) - $startTime;
                
                if ($attempts >= $this->retryAttempts) {
                    Log::error('Scanner timeout after all retries', [
                        'scanner' => $this->name(),
                        'url' => parse_url($url, PHP_URL_HOST),
                        'attempts' => $attempts,
                        'execution_time' => $executionTime
                    ]);
                    
                    return ModuleResult::timeout($this->name(), $executionTime);
                }
                
                $lastError = $e;
                sleep($this->retryDelay * $attempts); // Exponential backoff
                
            } catch (NetworkException $e) {
                if ($attempts >= $this->retryAttempts) {
                    Log::error('Scanner network error after all retries', [
                        'scanner' => $this->name(),
                        'url' => parse_url($url, PHP_URL_HOST),
                        'attempts' => $attempts,
                        'error' => $e->getMessage()
                    ]);
                    
                    return ModuleResult::error($this->name(), $e->getMessage(), $attempts - 1);
                }
                
                $lastError = $e;
                sleep($this->retryDelay * $attempts);
                
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
use App\Exceptions\{NetworkException, TimeoutException};
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
            throw new NetworkException("SSL connection failed: {$errstr}");
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
                throw new NetworkException($errstr ?: 'SSL connection failed');
            }
            
            // Check for timeout
            $elapsed = microtime(true) - $startTime;
            if ($elapsed > $this->timeout) {
                fclose($conn);
                throw new TimeoutException('SSL connection timed out');
            }
            
            // Extract certificate data and metadata
            $contextParams = stream_context_get_params($conn);
            $cert = $contextParams['options']['ssl']['peer_certificate'] ?? null;
            $certChain = $contextParams['options']['ssl']['peer_certificate_chain'] ?? [];
            $meta = stream_get_meta_data($conn);
            
            fclose($conn);
            
            if (!$cert) {
                throw new NetworkException('No SSL certificate found');
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
            $issues[] = 'Certificate chain is empty';
            return $issues;
        }
        
        // Check chain order and validity
        for ($i = 0; $i < count($chain) - 1; $i++) {
            $current = openssl_x509_parse($chain[$i]);
            $next = openssl_x509_parse($chain[$i + 1]);
            
            if (!$current || !$next) {
                $issues[] = 'Invalid certificate in chain';
                continue;
            }
            
            // Check if current cert is issued by next cert
            if (($current['issuer']['CN'] ?? '') !== ($next['subject']['CN'] ?? '')) {
                $issues[] = 'Certificate chain order is incorrect';
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
use App\Exceptions\{NetworkException, TimeoutException};
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
                throw new NetworkException("HTTP request failed: " . $response->status());
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
                throw new TimeoutException('HTTP request timed out');
            }
            throw new NetworkException('HTTP request failed: ' . $e->getMessage());
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
```php
namespace App\Services\Scanners;

use App\Data\ModuleResult;
use App\Exceptions\{NetworkException, TimeoutException};

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
            throw new NetworkException("DNS analysis failed: " . $e->getMessage());
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
    
    private function analyzeDnssec(string $host): array {
        // Simplified DNSSEC check - in production, use proper DNSSEC validation
        // For now, assume DNSSEC is not configured (common case)
        return [
            'score' => 0,
            'issues' => ['DNSSEC not validated'],
            'recommendations' => ['Enable DNSSEC for enhanced DNS security'],
            'strengths' => []
        ];
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
            throw new NetworkException("DNS query failed for {$typeName} records: " . $e->getMessage());
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

// Base exceptions
class AchilleusException extends \Exception {}
class ValidationException extends AchilleusException {}
class SecurityException extends AchilleusException {}
class ScannerException extends AchilleusException {}

// Specific exceptions
class SSRFException extends SecurityException {}
class DomainLimitException extends ValidationException {}
class ScanTimeoutException extends ScannerException {}
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
                } catch (ScanTimeoutException $e) {
                    Log::warning("Scanner timeout", [
                        'scanner' => $scanner->name(),
                        'domain' => $domain->url
                    ]);
                    $results[$scanner->name()] = ModuleResult::timeout($scanner->name());
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
        SSRFException::class => 'The provided URL is not accessible for security reasons.',
        DomainLimitException::class => 'You have reached your 10 domain limit. Please remove a domain to add a new one.',
        ScanTimeoutException::class => 'The scan took too long to complete. Please try again.',
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