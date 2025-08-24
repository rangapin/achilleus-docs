# Laravel Ecosystem for Achilleus

This document outlines all Laravel ecosystem products and packages used in the Achilleus security monitoring SaaS platform.

---

## Core Framework

### Laravel 12
- **Package**: `laravel/laravel`
- **Purpose**: Core web application framework
- **Usage**: Foundation for entire application

---

## First-Party Laravel Packages

### Laravel Socialite
- **Package**: `laravel/socialite`
- **Purpose**: OAuth authentication
- **Usage**: Google OAuth integration

### Laravel Cashier
- **Package**: `laravel/cashier`
- **Purpose**: Subscription billing management
- **Usage**: $27/month Stripe subscriptions, 14-day trials

### Laravel Pint
- **Package**: `laravel/pint`
- **Purpose**: Code style fixer
- **Usage**: Maintain consistent PHP code formatting

### Laravel Reverb
- **Package**: `laravel/reverb`
- **Purpose**: WebSocket server
- **Usage**: Real-time scan status updates

### Laravel Starter Kits (React)
- **Package**: `laravel/breeze` with React
- **Purpose**: Authentication scaffolding
- **Usage**: User registration, login, React + Inertia setup

---

## Third-Party Packages

### Barry PDF (DomPDF)
- **Package**: `barryvdh/laravel-dompdf`
- **Purpose**: PDF generation
- **Usage**: Security scan report generation

### Shadcn UI
- **Package**: `shadcn/ui`
- **Purpose**: React component library
- **Usage**: Consistent UI components

### Lucide Icons
- **Package**: `lucide-react`
- **Purpose**: Icon library
- **Usage**: UI icons throughout application

---

## Hosting & Infrastructure

### Laravel Cloud
- **Service**: Official Laravel hosting platform
- **Purpose**: Production deployment
- **Features**: 
  - Managed PostgreSQL database
  - Redis for queues and caching
  - Automatic SSL certificates
  - S3 integration

---

## Additional Tools

### Laravel Boost
- **Purpose**: AI Performance optimization
- **Usage**: Application performance enhancements

### Laravel Precognition