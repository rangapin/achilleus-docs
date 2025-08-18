# Achilleus Deployment Guide

## Laravel Cloud Setup

### Prerequisites
- Laravel Cloud account
- GitHub repository
- Stripe account (production keys)
- Domain name for production

### Initial Configuration
```bash
# Connect to Laravel Cloud
laravel cloud:configure

# Link GitHub repository
laravel cloud:github --repository="your-org/achilleus"

# Set environment variables
laravel cloud:env --set APP_ENV=production
laravel cloud:env --set APP_URL=https://achilleus.com
laravel cloud:env --set STRIPE_KEY=pk_live_...
laravel cloud:env --set STRIPE_SECRET=sk_live_...
```

### Database Configuration
```bash
# Create PostgreSQL database
laravel cloud:database:create --type=postgresql --hibernation

# Run migrations
laravel cloud:artisan migrate --force
```

### Queue Configuration
```bash
# Enable queue workers
laravel cloud:queue:create --workers=3 --autoscale

# Set queue connection
laravel cloud:env --set QUEUE_CONNECTION=sqs
```

### Storage Configuration
```bash
# Create S3 bucket for PDF reports
laravel cloud:storage:create --bucket=achilleus-reports

# Configure filesystem
laravel cloud:env --set FILESYSTEM_DISK=s3
```

### CDN Setup
Laravel Cloud automatically configures CDN for static assets.

## Environment Variables

### Required Production Variables
```env
APP_ENV=production
APP_KEY=base64:...
APP_URL=https://achilleus.com

DB_CONNECTION=pgsql
DB_HOST=automatically_configured
DB_DATABASE=automatically_configured
DB_USERNAME=automatically_configured
DB_PASSWORD=automatically_configured

REDIS_HOST=automatically_configured
REDIS_PASSWORD=automatically_configured

QUEUE_CONNECTION=sqs

MAIL_MAILER=ses
MAIL_FROM_ADDRESS=noreply@achilleus.com
MAIL_FROM_NAME="Achilleus"

STRIPE_KEY=pk_live_...
STRIPE_SECRET=sk_live_...
STRIPE_WEBHOOK_SECRET=whsec_...

AWS_ACCESS_KEY_ID=automatically_configured
AWS_SECRET_ACCESS_KEY=automatically_configured
AWS_DEFAULT_REGION=us-east-1
AWS_BUCKET=achilleus-reports
```

## Deployment Pipeline

### Automatic Deployment
Laravel Cloud deploys automatically on push to `main` branch:

1. Code pushed to GitHub
2. Laravel Cloud webhook triggered  
3. Tests run automatically
4. If tests pass, deploy to production
5. Database migrations run
6. Cache cleared
7. Application ready

### Manual Deployment
```bash
# Trigger manual deployment
laravel cloud:deploy --branch=main

# Check deployment status
laravel cloud:status

# View logs
laravel cloud:logs --tail
```

## Monitoring & Maintenance

### Health Checks
Laravel Cloud provides built-in monitoring:
- Application uptime
- Database connectivity
- Queue worker health
- Response time metrics

### Scaling Configuration
```bash
# Configure auto-scaling
laravel cloud:scale --min-instances=1 --max-instances=10

# Queue worker scaling
laravel cloud:queue:scale --min-workers=1 --max-workers=5
```

### Backup Strategy
```bash
# Database backups (automatic)
laravel cloud:backup:schedule --frequency=daily

# Manual backup
laravel cloud:backup:create --type=database
```

## SSL/Domain Setup

### Custom Domain
```bash
# Add custom domain
laravel cloud:domain:add achilleus.com

# Configure DNS records (provided by Laravel Cloud)
# A record: @ -> Laravel Cloud IP
# CNAME record: www -> Laravel Cloud domain
```

### SSL Certificate
SSL certificates are automatically provisioned and renewed by Laravel Cloud.

## Performance Optimization

### Caching Strategy
```bash
# Configure Redis caching
laravel cloud:env --set CACHE_DRIVER=redis
laravel cloud:env --set SESSION_DRIVER=redis

# Enable route caching
laravel cloud:artisan route:cache

# Enable config caching  
laravel cloud:artisan config:cache
```

### Database Optimization
- Connection pooling (automatic)
- Read replicas for reporting (if needed)
- Query optimization with indexes

## Cost Optimization

### Hibernation
Laravel Cloud automatically hibernates idle applications:
- Database hibernation after 30 minutes of inactivity
- Application hibernation after 15 minutes
- Wake time: ~2-3 seconds

### Resource Monitoring
```bash
# View usage metrics
laravel cloud:metrics

# Set spending alerts
laravel cloud:billing:alert --amount=100
```

## Troubleshooting

### Common Issues
```bash
# Check application logs
laravel cloud:logs --tail

# Debug database connectivity
laravel cloud:artisan tinker
>>> DB::connection()->getPdo()

# Queue worker debugging
laravel cloud:queue:status
laravel cloud:queue:logs

# Clear all caches
laravel cloud:artisan optimize:clear
```

### Emergency Procedures
```bash
# Rollback deployment
laravel cloud:rollback

# Scale down to save costs
laravel cloud:scale --min-instances=0

# Emergency maintenance mode
laravel cloud:artisan down --secret=emergency-token
```