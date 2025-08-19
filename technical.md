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
    │ Claude  │                                 │ Visual Studio  │
    │  Code   │                                 │   IDE   │
    └─────────┘                                 └─────────┘
```

## Database Design

**Complete database schema has been moved to `/docs/database.md`**

### Schema Overview
- PostgreSQL 15 with UUID primary keys
- JSONB for flexible scanner data storage
- Comprehensive indexes for performance
- Laravel package tables (Cashier)
- Full foreign key constraints

### Core Tables
- **users**: Authentication, billing, OAuth support
- **domains**: User domains with scan history
- **scans**: Security scan records with scoring
- **scan_modules**: Individual scanner results
- **reports**: Generated PDF reports

See `/docs/database.md` for complete schema, indexes, and relationships.

## Laravel Boost Integration

### AI-Powered Development Architecture

Laravel Boost provides an MCP server that enables AI assistants to deeply understand and interact with the Achilleus codebase. This integration dramatically improves development speed and code quality.

### Boost Capabilities

**Database Operations**
- Query actual database data
- Inspect schema and relationships
- Analyze query performance
- Generate test fixtures

**Code Execution**
- Run Tinker commands in project context
- Test scanner implementations
- Validate SSRF protection
- Debug with real data

**Development Tools**
- Route inspection and analysis
- Configuration access
- Log analysis and debugging
- Laravel 12 documentation search

### Security with Boost

- Development environment only
- Respects .env boundaries
- No production credential exposure
- Auto-includes security validations in generated code

### Scanner Development with Boost

Boost enables AI to:
- Inspect existing scanner patterns
- Test with real domains (badssl.com)
- Analyze scan performance metrics
- Generate comprehensive test cases
- Debug failures with actual logs

See `/docs/execution.md` Phase 0 for installation and setup instructions.

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

### SSRF Protection
- NetworkGuard validates all external URLs
- Blocks private IPs, localhost, cloud metadata endpoints
- Enforces HTTPS-only for domain scanning
- Throws SecurityException on violations

### Rate Limiting
- User-level: 10 scans/minute via middleware
- Scanner-level: Per-host limits in AbstractScanner
- Cache-based tracking with automatic expiry

## Job Processing

### Queue Management
- Laravel Cloud managed workers with auto-scaling
- Separate queues: default, scans, reports
- Automatic retry with exponential backoff
- Failed job handling with notifications

### Scan Job Workflow
1. Create scan record with 'pending' status
2. Dispatch RunDomainScan job to queue
3. Update status to 'running'
4. Execute scanners sequentially
5. Calculate weighted scores
6. Store results in scan_modules
7. Update domain's last_scan_score
8. Mark scan 'completed' or 'failed'

## Scoring Engine

### Score Calculation
- Weighted average: SSL/TLS (40%), Headers (30%), DNS/Email (30%)
- Dynamic weight redistribution when scanners fail
- Grade assignment: A+ (95-100), A (90-94), B+ (85-89), B (80-84), C (70-79), D (60-69), F (0-59)
- Results stored in JSONB for detailed analysis

## Payment Integration

### Laravel Cashier Configuration
- $27/month subscription with 14-day trial
- Stripe webhook handling for payment events
- Grace period for failed payments (30 days)
- Subscription status tracking in users table

## Laravel Ecosystem Integration

### Package Usage
- **Cashier**: Subscription and billing management
- **Socialite**: GitHub and Google OAuth authentication
- **Precognition**: Live form validation
- **Reverb**: WebSocket connections for real-time updates

See `/docs/execution.md` for installation and configuration.

## Subscription Management

### Trial & Billing Enforcement
- Middleware checks trial/subscription status
- Protected routes require active subscription
- Trial banner shows days remaining
- Automatic lock after trial expiry

## Laravel Cloud Configuration

### Managed Infrastructure
- Auto-scaling application servers
- PostgreSQL with automatic backups
- Redis cache and queue management
- S3-compatible storage for reports
- Global CDN for assets
- Automatic SSL certificates

### Deployment
- Git-based deployments
- Zero-downtime updates
- Automatic migrations
- Environment variable management

## Performance Optimization

### Key Strategies
- Database query optimization with indexes
- Caching for frequently accessed data
- Queue-based asynchronous processing
- CDN for static assets
- Connection pooling for database

### Monitoring
- Laravel Cloud APM
- Custom performance metrics
- Slow query logging
- Resource usage tracking

## Error Handling

### Exception Hierarchy
- **RetryableException**: Temporary issues (retry automatically)
- **ConfigurationException**: Domain issues (user action required)
- **SecurityException**: Policy violations (no retry)

### Logging Strategy
- Structured logging with context
- Different log levels for severity
- Security event tracking
- Performance metric logging

## API & Service Layer

### Service Classes
- **ScanOrchestrator**: Coordinates scanner execution
- **ScoringEngine**: Calculates weighted scores and grades
- **ReportGenerator**: Creates PDF security reports
- **NetworkGuard**: SSRF protection for all external requests

### API Endpoints
- RESTful design following Laravel conventions
- Resource-based routing
- Policy-based authorization
- Rate limiting via throttle middleware

## Summary

This technical documentation covers the core architecture for building Achilleus:
- System architecture with Laravel Cloud infrastructure
- Complete scanner implementations (SSL/TLS, Security Headers, DNS/Email)
- Scoring engine with weighted calculations
- Security measures including SSRF protection
- Job processing and queue management
- API and service layer design

For UI/UX specifications, see `/docs/design.md`
For database schema, see `/docs/database.md`
For testing strategy, see `/docs/testing.md`
For development workflow, see `/docs/execution.md`
