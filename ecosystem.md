# Laravel Ecosystem for Achilleus

This document outlines all Laravel ecosystem products and packages used in the Achilleus security monitoring SaaS platform.

### Laravel 12
- **Package**: `laravel/laravel`
## First-Party Laravel Packages

### Laravel Cashier
Laravel Cashier Stripe provides an expressive, fluent interface to Stripe's subscription billing services. It handles almost all of the boilerplate subscription billing code you are dreading writing.

https://laravel.com/docs/12.x/billing

<!-- 
composer require laravel/cashier 
-->

### Laravel Pint
Laravel Pint is an opinionated PHP code style fixer for minimalists. Pint is built on top of PHP CS Fixer and makes it simple to ensure that your code style stays clean and consistent.

https://laravel.com/docs/12.x/pint#main-content

<!-- 
composer require laravel/pint --dev 
-->

### Laravel Reverb
Laravel Reverb brings blazing-fast and scalable real-time WebSocket communication directly to your Laravel application, and provides seamless integration with Laravel’s existing suite of event broadcasting tools.

https://laravel.com/docs/12.x/reverb

<!-- 
php artisan install:broadcasting 
-->

### Barry PDF (DomPDF)

https://github.com/barryvdh/laravel-dompdf

<!-- 
composer require barryvdh/laravel-dompdf 
-->

### Shadcn UI
shadcn/ui is a set of beautifully-designed, accessible components and a code distribution platform. Works with your favorite frameworks and AI models. Open Source. Open Code.

https://ui.shadcn.com/docs

### Laravel Cloud

https://cloud.laravel.com/

### Laravel Boost
Laravel Boost accelerates AI-assisted development by providing the essential context and structure that AI needs to generate high-quality, Laravel-specific code.

https://boost.laravel.com/installed

<!-- 
composer require laravel/boost --dev
php artisan boost:install 
-->

### Laravel Precognition
Laravel Precognition allows you to anticipate the outcome of a future HTTP request. One of the primary use cases of Precognition is the ability to provide "live" validation for your frontend JavaScript application without having to duplicate your application's backend validation rules. Precognition pairs especially well with Laravel's Inertia-based starter kits.

https://laravel.com/docs/12.x/precognition#main-content

<!-- 
npm install laravel-precognition-react 
-->