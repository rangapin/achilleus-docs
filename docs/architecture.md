# Achilleus System Architecture

## Technology Stack
- **Backend**: Laravel 12 with PHP 8.3+
- **Frontend**: React 19 with TypeScript and Inertia.js
- **UI Components**: Shadcn/ui with dark theme support
- **Database**: Serverless PostgreSQL 15 (Laravel Cloud managed)
- **Cache**: Redis-compatible key-value store (auto-scaling)
- **Queue**: Laravel Cloud Queue Workers (auto-scaling)
- **Storage**: S3-compatible object storage for PDF reports
- **CDN**: Laravel Cloud Edge Network with global CDN
- **Hosting**: Laravel Cloud (serverless with hibernation)

## Application Architecture
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
                │   Services   │                              │    Queue     │
                │  (Business   │                              │  Workers     │
                │   Logic)     │                              │ (Auto-scale) │
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

## Service Layer Pattern
- **Controllers**: Thin, handle HTTP concerns only
- **Services**: Business logic and domain operations
- **Jobs**: Asynchronous processing via queues
- **Policies**: Authorization logic
- **Form Requests**: Input validation

## Security Architecture
- **SSRF Protection**: NetworkGuard validates all external URLs
- **Input Validation**: Form requests with custom rules
- **Rate Limiting**: Per-user and per-endpoint throttling
- **HTTPS Only**: All external requests use HTTPS
- **Private IP Blocking**: Prevents internal network access

## Scanner Architecture
- **AbstractScanner**: Base class with common functionality
- **ModuleResult**: Standardized result format
- **Weighted Scoring**: SSL (40%), Headers (30%), DNS (30%)
- **Error Handling**: Graceful degradation with specific exceptions

## Deployment Architecture (Laravel Cloud)
- **Serverless Functions**: Auto-scaling based on load
- **Queue Workers**: Auto-scaling background job processing
- **Database Hibernation**: Automatic sleep/wake for cost efficiency
- **CDN Edge Caching**: Global content delivery
- **Health Monitoring**: Built-in uptime and performance monitoring