# Achilleus Design System

## Design Philosophy

Achilleus uses a dark, professional design that conveys security and trust while remaining approachable for developers. The interface prioritizes clarity, speed, and actionable insights.

## Color Palette

### Primary Colors
- **Background**: #000000 (pure black) or #0a0a0b (near black)
- **Card Background**: #1a1a1a (dark gray)
- **Border**: #2a2a2a (subtle gray)
- **Surface**: #0f0f0f (elevated elements)

### Text Colors
- **Primary**: #ffffff (white)
- **Secondary**: #a0a0a0 (light gray)
- **Muted**: #6b7280 (dim gray)
- **Disabled**: #4b5563 (dark gray)

### Status Colors
- **Success/Green**: #22c55e (scores 80+)
- **Warning/Orange**: #f59e0b (scores 60-79)
- **Danger/Red**: #ef4444 (scores <60, critical issues)
- **Info/Blue**: #3b82f6 (trial banner, CTAs)
- **Purple**: #8b5cf6 (premium features)

### Grade Colors
- **A+ Grade**: #22c55e (bright green)
- **A Grade**: #34d399 (green)
- **B+ Grade**: #fbbf24 (yellow)
- **B Grade**: #f59e0b (orange)
- **C Grade**: #fb923c (light orange)
- **D Grade**: #f87171 (light red)
- **F Grade**: #ef4444 (red)

## Typography

### Font Stack
```css
font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', 'Inter', 
             'Roboto', 'Oxygen', 'Ubuntu', sans-serif;
```

### Font Sizes
- **H1**: 2.5rem (40px) - Page titles
- **H2**: 2rem (32px) - Section headers
- **H3**: 1.5rem (24px) - Card titles
- **H4**: 1.125rem (18px) - Subsections
- **Body**: 0.875rem (14px) - Default text
- **Small**: 0.75rem (12px) - Metadata, timestamps
- **Tiny**: 0.625rem (10px) - Labels, badges

### Font Weights
- **Regular**: 400 - Body text
- **Medium**: 500 - Emphasized text
- **Semibold**: 600 - Headers, buttons
- **Bold**: 700 - Important metrics

## Component Library (Shadcn/ui)

### Core Components

#### Button Variants
- **Primary**: Blue background, white text (main CTAs)
- **Secondary**: Dark border, white text (secondary actions)
- **Danger**: Red background, white text (destructive actions)
- **Success**: Green background, white text (positive actions)
- **Ghost**: Transparent, hover effect (tertiary actions)

#### Card Component
- Background: #1a1a1a
- Border: 1px solid #2a2a2a
- Border radius: 0.5rem
- Padding: 1.5rem
- Shadow: subtle drop shadow

#### Form Elements
- Input background: #0f0f0f
- Border: 1px solid #2a2a2a
- Focus border: #3b82f6
- Error border: #ef4444
- Border radius: 0.375rem
- Height: 2.5rem (40px)

#### Badges
- Small rounded rectangles
- Padding: 0.25rem 0.5rem
- Font size: 0.75rem
- Color-coded by status

## Page Layouts

### Dashboard Layout

```
┌─────────────────────────────────────────────────────────────────┐
│ Navigation Bar                                    Profile ▼ │
├─────────────────────────────────────────────────────────────────┤
│ Trial Banner (conditional)                       [Upgrade] │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Dashboard                                                      │
│  Your security monitoring overview                             │
│                                                                 │
│  ┌──────────────────┐  ┌──────────────────┐                   │
│  │ Security Score   │  │ Active Domains   │                   │
│  │    85/100        │  │      7/10        │                   │
│  │    Grade: B+     │  │   3 available    │                   │
│  └──────────────────┘  └──────────────────┘                   │
│                                                                 │
│  ┌──────────────────┐  ┌──────────────────┐                   │
│  │ Last Scan        │  │ Critical Issues  │                   │
│  │  2 hours ago     │  │       0          │                   │
│  │  github.com      │  │    All good      │                   │
│  └──────────────────┘  └──────────────────┘                   │
│                                                                 │
│  Security Score Trends                          [7d ▼]         │
│  ┌──────────────────────────────────────────────────┐         │
│  │ [Chart Area]                                      │         │
│  └──────────────────────────────────────────────────┘         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Domains Page Layout

```
┌─────────────────────────────────────────────────────────────────┐
│ Domains                                   [Scan All] [Add Domain]│
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│ ┌───────────────────────────────────────────────────────────┐ │
│ │ Domain         Last Scan      Score    Status    Actions   │ │
│ ├───────────────────────────────────────────────────────────┤ │
│ │ 🔐 github.com   2 hours ago    85 B+    ● Active   👁 🔍 🗑  │ │
│ │ 🔐 example.com  1 day ago      92 A     ● Active   👁 🔍 🗑  │ │
│ │ 🔐 mysite.io    3 days ago     67 C     ● Active   👁 🔍 🗑  │ │
│ └───────────────────────────────────────────────────────────┘ │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Domain Detail Layout

```
┌─────────────────────────────────────────────────────────────────┐
│ ← Back to Domains                                   [New Scan]  │
├─────────────────────────────────────────────────────────────────┤
│ github.com                                                      │
│ Last scanned 2 hours ago                                        │
│                                                                 │
│ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐               │
│ │ Overall     │ │ SSL/TLS     │ │ Coverage    │               │
│ │   85/100    │ │   Grade A   │ │   100%      │               │
│ │   Grade B+  │ │  90 days    │ │  Complete   │               │
│ └─────────────┘ └─────────────┘ └─────────────┘               │
│                                                                 │
│ Security Modules                                                │
│ ┌───────────────────────────────────────────────────────────┐ │
│ │ ✅ SSL/TLS Certificate         Score: 95/100              │ │
│ │    • Valid for 90 more days                               │ │
│ │    • TLS 1.3 supported                                    │ │
│ │    • Strong cipher suites                                 │ │
│ ├───────────────────────────────────────────────────────────┤ │
│ │ ⚠️  Security Headers           Score: 75/100              │ │
│ │    • Missing CSP header                                   │ │
│ │    • HSTS configured correctly                            │ │
│ │    • X-Frame-Options: DENY                                │ │
│ ├───────────────────────────────────────────────────────────┤ │
│ │ ✅ DNS & Email Security        Score: 85/100              │ │
│ │    • SPF record valid                                     │ │
│ │    • DMARC policy: reject                                 │ │
│ │    • DNSSEC enabled                                       │ │
│ └───────────────────────────────────────────────────────────┘ │
│                                                                 │
│ Recommended Actions                                             │
│ ┌───────────────────────────────────────────────────────────┐ │
│ │ 🔴 High Priority                                          │ │
│ │    • Add Content-Security-Policy header                   │ │
│ │                                                            │ │
│ │ 🟡 Medium Priority                                        │ │
│ │    • Upgrade cipher suites for better performance         │ │
│ └───────────────────────────────────────────────────────────┘ │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## Modal Dialogs

### Add Domain Modal

```
┌─────────────────────────────────────────────────┐
│ Add New Domain                              [✕] │
├─────────────────────────────────────────────────┤
│                                                  │
│ Domain URL *                                     │
│ ┌──────────────────────────────────────────┐    │
│ │ https://                                  │    │
│ └──────────────────────────────────────────┘    │
│ Enter the full HTTPS URL of your domain         │
│                                                  │
│ Email Configuration                              │
│ ┌──────────────────────────────────────────┐    │
│ │ ● Domain sends email                     │    │
│ │ ○ Domain doesn't send email              │    │
│ └──────────────────────────────────────────┘    │
│                                                  │
│ DKIM Selector (optional)                        │
│ ┌──────────────────────────────────────────┐    │
│ │ google                                    │    │
│ └──────────────────────────────────────────┘    │
│                                                  │
│         [Cancel]  [Add & Scan Domain]            │
└─────────────────────────────────────────────────┘
```

### Delete Confirmation Modal

```
┌─────────────────────────────────────────────────┐
│ Delete Domain?                              [✕] │
├─────────────────────────────────────────────────┤
│                                                  │
│ ⚠️  This action cannot be undone                 │
│                                                  │
│ Are you sure you want to delete github.com?     │
│ All scan history will be permanently removed.   │
│                                                  │
│              [Cancel]  [Delete]                  │
└─────────────────────────────────────────────────┘
```

## Empty States

### No Domains
```
┌─────────────────────────────────────────────────┐
│              🔐                                  │
│                                                  │
│         No domains yet                          │
│                                                  │
│   Add your first domain to start monitoring     │
│   your website's security posture               │
│                                                  │
│         [Add Your First Domain]                 │
└─────────────────────────────────────────────────┘
```

### No Scan Activity
```
┌─────────────────────────────────────────────────┐
│              📊                                  │
│                                                  │
│       No scan activity yet                      │
│                                                  │
│   Run your first security scan to see           │
│   detailed results and recommendations          │
│                                                  │
│           [Run First Scan]                      │
└─────────────────────────────────────────────────┘
```

## Loading States

### Scanning Animation
```
┌─────────────────────────────────────────────────┐
│ Scanning github.com...                          │
│                                                  │
│ [████████░░░░░░░░░░░░░░░░░░] 33%               │
│                                                  │
│ Currently checking: SSL/TLS Certificate          │
│                                                  │
│ This usually takes 15-30 seconds                │
└─────────────────────────────────────────────────┘
```

### Skeleton Loaders
- Use for tables while data loads
- Gray animated bars (#2a2a2a to #3a3a3a)
- Match the height of actual content

## Interactive Elements

### Hover States
- Buttons: Lighten by 10%
- Cards: Subtle border highlight (#3a3a3a)
- Table rows: Background #1a1a1a
- Links: Underline on hover

### Focus States
- Blue outline (#3b82f6)
- 2px width
- 2px offset
- Rounded corners matching element

### Active States
- Buttons: Darken by 10%
- Scale: 0.98
- Transition: 150ms ease

## Navigation

### Primary Navigation
```
┌─────────────────────────────────────────────────────────┐
│ 🔐 Achilleus    Dashboard  Domains  Activity  Reports  │
└─────────────────────────────────────────────────────────┘
```

### Profile Dropdown
```
┌──────────────────┐
│ Profile Settings │
│ Subscription     │
│ ─────────────    │
│ Sign Out         │
└──────────────────┘
```

## Trial Banner

Displays when user.trial_ends_at > now():

```
┌─────────────────────────────────────────────────────────────────┐
│ 🕒 Solo Plan Trial - 14 days remaining - 3/10 domains         │
│                                          [Upgrade Now] [✕]     │
└─────────────────────────────────────────────────────────────────┘
```

## Responsive Design

### Breakpoints
- Mobile: < 640px
- Tablet: 640px - 1024px
- Desktop: > 1024px

### Mobile Adaptations
- Stack dashboard cards vertically
- Hamburger menu for navigation
- Simplified tables (key data only)
- Full-width buttons
- Bottom sheet modals

### Touch Targets
- Minimum 44x44px
- 8px spacing between targets
- Larger buttons on mobile

## Charts & Visualizations

### Security Score Chart
- Bar chart for daily/weekly/monthly trends
- Color-coded bars based on score
- Smooth animations on data change
- Tooltip on hover with exact values

### Score Gauge
```
     A+
   ╱    ╲
  │  85  │
   ╲    ╱
     B+
```

## Icon System

### Status Icons
- ✅ Success/Complete
- ⚠️ Warning/Attention
- ❌ Error/Failed
- 🔄 In Progress
- 🔐 Security/Lock
- 📊 Report/Chart
- 👁 View
- 🔍 Scan
- 🗑 Delete
- ➕ Add
- 📧 Email

### Grade Badges
- Rounded rectangles
- Bold text
- Color-coded backgrounds
- Min width: 48px

## Animation & Transitions

### Page Transitions
- Fade in: 200ms ease-out
- Slide up: 300ms ease-out
- No jarring movements

### Loading Spinners
- Rotating circle
- 1s rotation duration
- Border color matches brand

### Progress Bars
- Smooth fill animation
- Striped pattern for active state
- Color changes based on value

## Accessibility

### WCAG 2.1 AA Compliance
- Color contrast ratios > 4.5:1
- Focus indicators on all interactive elements
- Keyboard navigation support
- Screen reader labels
- Alt text for icons

### Keyboard Shortcuts
- `/` - Focus search
- `n` - New domain
- `s` - Start scan
- `?` - Show help
- `Esc` - Close modals

## Error Handling

### Error Messages
- Red text (#ef4444)
- Clear, actionable language
- Suggest next steps
- Include support contact for critical errors

### Validation Messages
- Inline below form fields
- Real-time validation feedback
- Success checkmarks when valid

## Print Styles

### PDF Reports
- White background
- Black text
- Company logo in header
- Page numbers
- Clean, professional layout
- A4 format

## Landing Page (Salient Template)

Uses customized Salient template with:
- Hero section with security focus
- Feature grid highlighting benefits
- Pricing section ($27/month emphasis)
- Trust badges and security certifications
- Dark theme matching app design
- Smooth scroll animations