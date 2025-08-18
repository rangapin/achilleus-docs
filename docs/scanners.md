# Achilleus Scanner Implementation

## Scanner Interface
```php
namespace App\Contracts\Scanners;

interface Scanner {
    public function scan(string $url, array $context = []): ModuleResult;
    public function name(): string;
    public function getWeight(): int;
    public function getTimeout(): int;
    public function getRetryAttempts(): int;
}
```

## AbstractScanner Base Class
All scanners extend AbstractScanner for consistency:
- Automatic SSRF protection via NetworkGuard
- Rate limiting per target host
- Retry logic with exponential backoff
- Consistent error handling and logging
- Execution time tracking

## Scanner Modules

### 1. SSL/TLS Scanner (40% weight)
**File**: `app/Services/Scanners/SslTlsScanner.php`

**Checks**:
- Certificate validation and expiration
- Protocol version analysis (TLS 1.2+ required)
- Cipher suite strength evaluation
- Perfect Forward Secrecy verification
- Certificate chain validation
- Hostname verification (CN + SAN)

**Scoring**:
- Valid certificate: 40 points
- TLS 1.3: +10 bonus points
- Strong ciphers: 30 points
- Perfect Forward Secrecy: 20 points

### 2. Security Headers Scanner (30% weight)
**File**: `app/Services/Scanners/SecurityHeadersScanner.php`

**Headers Analyzed**:
- HSTS (35 points): max-age, includeSubDomains, preload
- CSP (35 points): presence and strength
- X-Content-Type-Options (10 points): nosniff
- X-Frame-Options (10 points): DENY/SAMEORIGIN
- Referrer-Policy (10 points): strict policies

**HTTP Client Configuration**:
- HTTPS-only requests
- Limited redirects (max 3)
- SSL verification enabled
- Custom User-Agent

### 3. DNS/Email Scanner (30% weight)
**File**: `app/Services/Scanners/DnsEmailScanner.php`

**DNS Records Checked**:
- SPF records: syntax and policy strength
- DKIM verification: common selectors + custom
- DMARC policy: none < quarantine < reject
- DNSSEC validation
- CAA records for certificate authority authorization

**Email Security Scoring**:
- SPF configured: 40 points
- DKIM present: 30 points  
- DMARC policy: 30 points
- DNSSEC bonus: +10 points

## Error Handling

### Exception Hierarchy
```php
class ScannerException extends Exception {}
class NetworkException extends ScannerException {}
class TimeoutException extends ScannerException {}
class RateLimitException extends ScannerException {}
class ValidationException extends ScannerException {}
class SSRFException extends ValidationException {}
```

### Rate Limiting by Scanner
- SSL/TLS: 5 requests per minute (conservative for SSL handshakes)
- Security Headers: 8 requests per minute (HTTP requests)
- DNS/Email: 15 requests per minute (DNS queries are lighter)

## ModuleResult Format
```php
class ModuleResult {
    public string $module;
    public int $score;        // 0-100
    public string $status;    // ok, warn, fail, error
    public array $raw;        // Detailed findings
}
```

## Scoring Engine
```php
class ScanScoringEngine {
    const WEIGHTS = [
        'ssl_tls' => 40,
        'security_headers' => 30,
        'dns_email' => 30
    ];
    
    public function calculateTotalScore(array $moduleResults): int;
    public function assignGrade(int $score): string;
}
```

## Grade Mapping
- A+ (95-100): Excellent security
- A (90-94): Very good security  
- B+ (85-89): Good security with minor issues
- B (80-84): Acceptable security
- C (70-79): Moderate security gaps
- D (60-69): Significant security issues
- F (0-59): Poor security, immediate action needed

## Testing Strategy
- Unit tests for each scanner with known domains
- Integration tests with badssl.com variants
- Mocked external dependencies for CI/CD
- Real domain tests in staging environment