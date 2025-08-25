# Achilleus Design Documentation

## Design Philosophy

**Extend, Don't Rebuild**: Achilleus builds upon the Laravel React starter template structure without modifying the existing layout, navigation, or visual design. The interface uses the exact same sidebar, card grid, spacing, and component styling while filling content areas with security monitoring data.

**Core Principle**: The final project should look identical to the starter template with security-focused content replacing the placeholder data.

---

## Template Structure Analysis

Based on the Laravel React starter template screenshot, the layout consists of:

### Sidebar (256px width)
- **Logo area**: "Achilleus" branding at top (using 'Instrument Sans' font)
- **Navigation menu**: Clean list-style navigation items
- **Profile section**: Bottom-positioned dropdown with avatar, user name, and menu
- **Collapsible toggle**: Hamburger icon in exact same position for sidebar collapse

### Main Content Area
- **Header**: Page title with optional action buttons on the right
- **3-card grid**: Perfectly spaced card layout (3 cards in single row on desktop)
- **Chart area**: Full-width interactive bar chart below cards
- **Consistent spacing**: Proper margins and padding throughout

### Visual Design Elements
- **Dark theme**: Neutral dark colors with proper contrast
- **Card styling**: Subtle borders, shadows, and rounded corners
- **Typography**: Clean hierarchy with proper font weights and sizes
- **Component styling**: Consistent button styles, form inputs, and interactive elements

---

## Color System

### Core Palette (Green Theme - HSL Color Space)
Light Mode:
```css
--background: 0 0% 100%;              /* White background */
--foreground: 240 10% 3.9%;           /* Dark text */
--card: 0 0% 100%;                     /* Card background */
--card-foreground: 240 10% 3.9%;       /* Card text */
--popover: 0 0% 100%;                  /* Popover background */
--popover-foreground: 240 10% 3.9%;    /* Popover text */
--primary: 142 70% 45%;                /* Green primary */
--primary-foreground: 0 0% 98%;        /* White on green */
--secondary: 142 30% 95%;              /* Light green */
--secondary-foreground: 142 70% 25%;   /* Dark green text */
--muted: 142 10% 96%;                  /* Very light green */
--muted-foreground: 240 4.8% 45.9%;    /* Muted text */
--accent: 142 50% 90%;                 /* Light green accent */
--accent-foreground: 142 70% 30%;      /* Dark green accent text */
--destructive: 0 84.2% 60.2%;          /* Red for errors */
--destructive-foreground: 0 0% 98%;    /* White on red */
--border: 142 20% 90%;                 /* Light green borders */
--input: 142 20% 90%;                  /* Light green inputs */
--ring: 142 70% 45%;                   /* Green focus ring */
--radius: 0.5rem;                      /* Border radius */
```

Dark Mode:
```css
--background: 240 10% 3.9%;            /* Dark background */
--foreground: 0 0% 98%;                /* Light text */
--card: 240 10% 3.9%;                  /* Dark card */
--card-foreground: 0 0% 98%;           /* Light card text */
--popover: 240 10% 3.9%;               /* Dark popover */
--popover-foreground: 0 0% 98%;        /* Light popover text */
--primary: 142 70% 50%;                /* Bright green primary */
--primary-foreground: 142 10% 10%;     /* Dark text on green */
--secondary: 142 30% 15%;              /* Dark green secondary */
--secondary-foreground: 142 60% 70%;   /* Light green text */
--muted: 142 20% 15%;                  /* Dark muted green */
--muted-foreground: 240 5% 64.9%;      /* Muted light text */
--accent: 142 40% 20%;                 /* Dark green accent */
--accent-foreground: 142 60% 70%;      /* Light green accent text */
--destructive: 0 63% 31%;              /* Dark red for errors */
--destructive-foreground: 0 0% 98%;    /* White on dark red */
--border: 142 30% 18%;                 /* Dark green borders */
--input: 142 30% 18%;                  /* Dark green inputs */
--ring: 142 70% 50%;                   /* Bright green focus ring */
```

### Chart Colors (Green Theme)
```css
--chart-1: 142 70% 45%;  /* Primary green */
--chart-2: 142 50% 55%;  /* Medium green */
--chart-3: 142 30% 65%;  /* Light green */
--chart-4: 142 60% 35%;  /* Dark green */
--chart-5: 142 40% 75%;  /* Very light green */
```

### Security-Specific Status Colors
- **Success/A+**: `hsl(142 70% 45%)` - Primary green for excellent scores (95+)
- **Good/A-B+**: `hsl(142 50% 55%)` - Medium green for good scores (80-94)
- **Warning/B-C**: `hsl(47 80% 50%)` - Yellow for average scores (70-79)
- **Caution/D**: `hsl(25 80% 50%)` - Orange for poor scores (60-69)
- **Danger/F**: `hsl(0 84% 60%)` - Red for failing scores (<60)
- **Info**: `hsl(142 40% 60%)` - Light green for informational states

### Grade-Specific Colors (Aligned with Green Theme)
- **A+ Grade**: `hsl(142 70% 45%)` - Primary green
- **A Grade**: `hsl(142 60% 50%)` - Strong green
- **B+ Grade**: `hsl(142 40% 55%)` - Medium green
- **B Grade**: `hsl(90 50% 50%)` - Yellow-green
- **C Grade**: `hsl(47 80% 50%)` - Yellow
- **D Grade**: `hsl(25 80% 50%)` - Orange
- **F Grade**: `hsl(0 84% 60%)` - Red

---

## Typography

### Font Stack (From Template)
```css
font-family: 'Instrument Sans', ui-sans-serif, system-ui, sans-serif,
             'Apple Color Emoji', 'Segoe UI Emoji', 'Segoe UI Symbol',
             'Noto Color Emoji';
```

### Font Sizes (From Template)
- **H1**: 2.5rem (40px) - Dashboard page title
- **H2**: 2rem (32px) - Section headers
- **H3**: 1.5rem (24px) - Card titles
- **H4**: 1.125rem (18px) - Subsections
- **Body**: 0.875rem (14px) - Default text
- **Small**: 0.75rem (12px) - Metadata and labels
- **Tiny**: 0.625rem (10px) - Status badges

### Font Weights (From Template)
- **Regular**: 400 - Body text
- **Medium**: 500 - Emphasized text
- **Semibold**: 600 - Card titles, headers
- **Bold**: 700 - Important metrics and numbers

---

## Dashboard Layout (Exact Template Structure)

```
┌─────────────────────────────────────────────────────────────────┐
│ Sidebar (256px)    │ Collapse Icon Main Content Area                         
│ ┌────────────────┐ │                                            │
│ │   Achilleus    │ │  Dashboard                    [Scan All]  │
│ └────────────────┘ │                                            │
│  Dashboard         │                                            │
│  Domains           │  ┌────────────┐ ┌────────────┐ ┌────────────┐
│  Activity          │  │Security    │ │Your        │ │Critical    │
│  Reports           │  │Score       │ │Domains     │ │Issues      │
│                    │  │    85      │ │   3/10     │ │     0      │
│                    │  │  Grade B+  │ │  3 used    │ │   None     │
│                    │  └────────────┘ └────────────┘ └────────────┘
│                                                               │
│                    │                                            │
│                     │  Security Score Trend - Last 30 Days      │
│                    │  ┌─────────────────────────────────────┐   │
│  ┌──────────────┐  │  │  [All Domains] [Domain 1] [Domain 2]│   │
│  │👤 User Name  ▼│  │  │                                     │   │
│  └──────────────┘  │  │  ███ ███ ███ ███ ███ ███ ███ ███   │   │
│                    │  │  Interactive bar chart with toggle  │   │
│                    │  │  Total Score: 2,550                 │   │
│                    │  └─────────────────────────────────────┘   │
└────────────────────┴─────────────────────────────────────────────┘
```

### Sidebar Navigation Items
Replace template navigation with security-focused items:
- **Dashboard** - Main overview page
- **Domains** - Domain management and list
- **Activity** - Recent scans and history
- **Reports** - Generated PDF reports

### Dashboard Cards Content (3 Cards Layout)
Replace template card content with security metrics:

#### Security Score Card
- **Large number**: Average security score (85)
- **Grade indicator**: Letter grade (B+)
- **Color coding**: Green/yellow/red based on score
- **Trend indicator**: ↑ or ↓ from last period

#### Your Domains Card  
- **Usage count**: Total domains (3/10)
- **Limit indicator**: "3 of 10 domains used"
- **Progress bar**: Visual representation of limit usage
- **Quick stats**: Last scan time across all domains

#### Critical Issues Card
- **Count**: Critical issues across all domains (0)
- **Status**: "None" or count with severity
- **Alert level**: Color coding (green for 0, red for critical)
- **Breakdown**: SSL: 0, Headers: 0, DNS: 0

### Interactive Bar Chart (shadcn/ui)
Full-width interactive bar chart below cards:
- **Chart Type**: Bar chart with toggle between datasets
- **Toggle Options**: "All Domains" vs individual domain selection
- **Time Period**: Last 30 days of security scores
- **Interactive Header**: 
  - Left side: Chart title and description
  - Right side: Toggle buttons showing totals
- **Visual Features**:
  - Vertical bars for each day's score
  - Color coding: Green for good scores, yellow for medium, red for poor
  - Hover tooltips with exact score and date
  - Animated transitions when switching datasets
- **Implementation**: Using shadcn/ui ChartContainer with Recharts
- **Height**: 250px fixed height
- **Responsive**: Adjusts margins and tick gaps for mobile

---

## Domain List Page Layout

```
┌─────────────────────────────────────────────────────────────────┐
│ Sidebar │  Domains                      [Add Domain] [Scan All] │
│         │                                                        │
│         │  ┌────────────────────────────────────────────────────┐ │
│         │  │ Domain         Last Scan       Score      Actions  │ │
│         │  ├────────────────────────────────────────────────────┤ │
│         │  │ example.com    2 hours ago    85 B+   View|Scan|Delete │ │
│         │  │ myapp.com      1 day ago      92 A    View|Scan|Delete │ │
│         │  │ mysite.io      3 days ago     67 C    View|Scan|Delete │ │
│         │  │                                                    │ │
│         │  └────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

### Table Structure (Using Template Table Component)
- **Domain column**: Domain name (no icons)
- **Last Scan column**: Relative time (2 hours ago) or "Never"
- **Score column**: Number + grade with color coding
- **Actions column**: Text links or buttons (View | Scan | Delete)

### Action Buttons (Template Button Styles)
- **Add Domain**: Primary button (top right)
- **Scan All**: Secondary button (top right)
- **Individual actions**: Icon buttons in table rows

### Inline Scanning State (Shadcn Progress + Reverb)
When a scan is triggered from the domain list, the row shows inline progress:

```tsx
// Domain row during scanning
<TableRow>
  <TableCell>example.com</TableCell>
  <TableCell>
    <div className="space-y-2">
      <Progress value={scanProgress} className="h-2 w-32" />
      <span className="text-xs text-muted-foreground">Scanning...</span>
    </div>
  </TableCell>
  <TableCell>-</TableCell>
  <TableCell>
    <Button variant="ghost" size="sm" disabled>
      Scanning...
    </Button>
  </TableCell>
</TableRow>

// Reverb subscription for inline progress
useReverbChannel(`scan.${domain.id}`, {
  onProgress: (data) => updateRowProgress(domain.id, data.percentage)
})
```

---

## Domain Detail Page Layout

```
┌─────────────────────────────────────────────────────────────────┐
│ Sidebar │  example.com                          [New Scan]      │
│         │                                                        │
│         │  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐     │
│         │  │ Security    │ │ SSL Grade   │ │ Last Scan   │     │
│         │  │ Score       │ │             │ │             │     │
│         │  │   85/100    │ │     A       │ │  2 hrs ago  │     │
│         │  │   Grade B+  │ │  Strong SSL │ │   Success   │     │
│         │  └─────────────┘ └─────────────┘ └─────────────┘     │
│         │                                                        │
│         │  Scanner Results                                       │
│         │  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐     │
│         │  │ SSL/TLS     │ │ Security    │ │ DNS/Email   │     │
│         │  │ Scanner     │ │ Headers     │ │ Security    │     │
│         │  │   45/50     │ │   15/20     │ │   25/30     │     │
│         │  │   Grade A   │ │   Grade C   │ │   Grade B   │     │
│         │  └─────────────┘ └─────────────┘ └─────────────┘     │
│         │                                                        │
│         │  Detailed Analysis                                     │
│         │  ┌───────────────────────────────────────────────────┐ │
│         │  │ ✅ SSL/TLS Certificate               
│         │  │    • Valid for 90 more days                       │ │
│         │  │    • TLS 1.3 supported                            │ │
│         │  │    • Strong cipher suites                         │ │
│         │  │_____________________________________________________
│         │  │ ⚠️  Security Headers                 
│         │  │    • Missing CSP header (-5 points)               │ │
│         │  │    • HSTS configured correctly                    │ │
│         │  │    • X-Frame-Options: DENY                        │ │
│         │  _______________________________________________________│    
│         │  │ ✅ DNS & Email Security              
│         │  │    • SPF record valid                             │ │
│         │  │    • DMARC policy: quarantine                     │ │
│         │  │    • DKIM configured                              │ │
│         │  └───────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

### Card Structure Breakdown

#### Top Row - Overview Cards
1. **Security Score Card**: Overall composite score (0-100) with grade
2. **SSL Grade Card**: Industry-standard SSL grade (A+ to F)
3. **Last Scan Card**: Time since last scan with status

#### Bottom Row - Scanner Results Cards
1. **SSL/TLS Scanner Card**: Score out of 50 points with SSL grade
2. **Security Headers Card**: Score out of 20 points with grade
3. **DNS/Email Security Card**: Score out of 30 points with grade

#### Detailed Analysis Section
- **Expandable sections** for each scanner with specific findings
- **Color-coded status indicators** (✅ pass, ⚠️ warning, ❌ fail)
- **Point deductions** shown for specific issues
- **Actionable recommendations** for improvements

---

## Activity Page Layout

```
┌─────────────────────────────────────────────────────────────────┐
│ Sidebar │  Activity                                             │
│         │  Scan history across all domains                      │
│         │                                                        │
│         │  ┌────────────────────────────────────────────────────┐ │
│         │  │ Domain Filter: [All Domains ▼]                    │ │
│         │  └────────────────────────────────────────────────────┘ │
│         │                                                        │
│         │  ┌────────────────────────────────────────────────────┐ │
│         │  │ Date          Domain         Score      Actions      │ │
│         │  ├────────────────────────────────────────────────────┤ │
│         │  │ 2 hours ago   example.com    85 B+   Generate Report │ │
│         │  │ 5 hours ago   mysite.io      92 A    Generate Report │ │
│         │  │ 1 day ago     example.com    83 B+   Generate Report │ │
│         │  │ 2 days ago    another.com    78 C    Generate Report │ │
│         │  │ 3 days ago    example.com    80 B    Generate Report │ │
│         │  │                                                    │ │
│         │  │ Showing 1-20 of 145 scans                         │ │
│         │  │ [Previous] [1] 2 3 4 ... 8 [Next]                 │ │
│         │  └────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

### Activity Page Features
- **Domain Filter Dropdown**: Filter by specific domain or "All Domains"
- **Scan History Table**: 
  - Date/time (relative format)
  - Domain name with link to domain detail
  - Score with grade badge
  - Generate Report button for each scan
- **Pagination**: 20 items per page
- **Sorting**: Default by date (newest first)
- **Mobile**: Cards instead of table on <768px

---

## Reports Page Layout

```
┌─────────────────────────────────────────────────────────────────┐
│ Sidebar │  Reports                                               │
│         │  Security reports                           │
│         │                                                        │
│         │  ┌────────────────────────────────────────────────────┐ │
│         │  │ Domain Filter: [All Domains ▼]                    │ │
│         │  └────────────────────────────────────────────────────┘ │
│         │                                                        │
│         │  ┌────────────────────────────────────────────────────┐ │
│         │  │ Date          Domain         Score     Actions      │ │
│         │  ├────────────────────────────────────────────────────┤ │
│         │  │ 1 hour ago    example.com    85 B+   Download PDF │ │
│         │  │ 3 hours ago   mysite.io      92 A    Download PDF │ │
│         │  │ 1 day ago     example.com    83 B+   Download PDF │ │
│         │  │ 3 days ago    another.com    78 C    Download PDF │ │
│         │  │ 1 week ago    example.com    80 B    Download PDF │ │
│         │  │                                                    │ │
│         │  │ Showing 1-20 of 52 reports                        │ │
│         │  │ [Previous] [1] 2 3 [Next]                         │ │
│         │  └────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

### Reports Page Features
- **Domain Filter Dropdown**: Filter by specific domain or "All Domains"
- **Reports Table**:
  - Generation date/time (relative format)
  - Domain name
  - Score at time of report
  - Download button (generates 1-hour S3 signed URL)
- **Pagination**: 20 items per page
- **Sorting**: Default by date (newest first)
- **File Size**: Shown in hover tooltip
- **Mobile**: Cards instead of table on <768px

---

## Modal Dialogs (Using Template Dialog Component)

### Add Domain Modal
```
┌─────────────────────────────────────────────────┐
│ Add New Domain                              [✕] │
├─────────────────────────────────────────────────┤
│                                                  │
│ Domain Name *                                    │
│ ┌──────────────────────────────────────────┐    │
│ │ example.com                   → https:// │    │
│ └──────────────────────────────────────────┘    │
│ Just enter the domain name (e.g., example.com)  │
│                                                  │
│ Email Configuration                              │
│ ● Domain sends email                             │
│ ○ Domain doesn't send email                      │
│                                                  │
│ DKIM Selector (optional)                        │
│ ┌──────────────────────────────────────────┐    │
│ │ google                                    │    │
│ └──────────────────────────────────────────┘    │
│ Common selectors: google, k1, k2, mandrill      │
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
│ Are you sure you want to delete example.com?    │
│ All scan history will be permanently removed.   │
│                                                  │
│              [Cancel]  [Delete]                  │
└─────────────────────────────────────────────────┘
```

---

## Loading States and Empty States

### Scanning Progress (Shadcn Progress + Laravel Reverb)

#### UI Implementation
```tsx
import { Progress } from "@/components/ui/progress"
import { Card, CardContent, CardDescription, CardHeader, CardTitle } from "@/components/ui/card"
import { useReverbChannel } from "@/hooks/use-reverb"

// Real-time progress updates via Laravel Reverb WebSocket
const [progress, setProgress] = useState(0)
const [currentModule, setCurrentModule] = useState("")

// Subscribe to Reverb channel for scan progress
useReverbChannel(`scan.${scanId}`, {
  onProgress: (data) => {
    setProgress(data.percentage)
    setCurrentModule(data.module)
  },
  onComplete: (data) => {
    setProgress(100)
    // Handle completion
  }
})

// UI Component
<Card className="w-full max-w-md mx-auto">
  <CardHeader>
    <CardTitle>Scanning {domain}</CardTitle>
    <CardDescription>This usually takes 15-30 seconds</CardDescription>
  </CardHeader>
  <CardContent className="space-y-4">
    <Progress value={progress} className="w-full" />
    <div className="flex justify-between text-sm text-muted-foreground">
      <span>Currently checking: {currentModule}</span>
      <span>{progress}%</span>
    </div>
  </CardContent>
</Card>
```

#### Visual States
- **0-33%**: SSL/TLS Certificate scan (progress bar in default color)
- **34-66%**: Security Headers scan (smooth transition)
- **67-100%**: DNS/Email Security scan (completion animation)

#### Progress Component Variants
```tsx
// Different styles based on scan status
<Progress 
  value={progress} 
  className={cn(
    "w-full transition-all",
    progress === 100 && "bg-green-500/20"
  )}
/>

// Indeterminate state for initial connection
<Progress className="w-full" /> // Shows pulsing animation when no value

// Error state
<Progress value={progress} className="w-full bg-red-500/20" />
```

#### Reverb WebSocket Integration
```php
// Backend broadcasts progress via Reverb
broadcast(new ScanProgressEvent($scan))
  ->toChannel("scan.{$scan->id}")
  ->withData([
    'percentage' => 65,
    'module' => 'Security Headers',
    'status' => 'in_progress',
    'estimated_time_remaining' => 10 // seconds
  ]);
```

#### Real-time Updates Flow
1. **Scan Initiated**: Progress component appears with indeterminate state
2. **Connection Established**: Reverb WebSocket connects to scan channel
3. **Module Progress**: Updates every 2-3 seconds with current module
4. **Completion**: Progress reaches 100%, success animation plays
5. **Results Ready**: Progress component replaced with results summary

### No Domains State
```
┌─────────────────────────────────────────────────┐
│       No domains added yet                      │
│                                                  │
│  Add your first domain to start monitoring      │
│  your website's security posture                │
│                                                  │
│  Just enter the domain name like "example.com"  │
│                                                  │
│        [Add Your First Domain]                  │
└─────────────────────────────────────────────────┘
```

---

## Toast Notification System

### Notification Positioning
- **Position**: Top-right corner of viewport
- **Stacking**: Multiple toasts stack vertically
- **Animation**: Slide in from right, fade out
- **Duration**: 5 seconds (10 for errors)

### Notification Types

#### Success Toast
- **Background**: Green with subtle opacity
- **Icon**: Checkmark
- **Message**: "Domain added successfully"

#### Error Toast  
- **Background**: Red with subtle opacity
- **Icon**: X mark
- **Message**: "Scan failed - please try again"

#### Info Toast
- **Background**: Blue with subtle opacity
- **Icon**: Info circle
- **Message**: "Scan completed for example.com"

---

## Responsive Design

### Desktop (>1024px)
- **Sidebar**: 256px width, collapsible
- **Cards**: 2x2 grid layout
- **Table**: Full table with all columns
- **Chart**: Full width with all controls

### Tablet (640px-1024px)
- **Sidebar**: Collapsible, overlay when open
- **Cards**: 2x2 grid, slightly smaller
- **Table**: Condensed columns
- **Chart**: Responsive with scroll

### Mobile (<640px)
- **Sidebar**: Hidden, hamburger menu
- **Cards**: Single column stack
- **Table**: Card-based layout
- **Chart**: Simplified view with touch controls

### Mobile Card Layout (Below 768px)
```
┌─────────────────────────┐
│ example.com     B+  85  │
│ Last scan: 2 hours ago  │
│ [View] [Scan] [Delete]  │
└─────────────────────────┘
```

### Responsive Breakpoints
- **Desktop (>1024px)**: Full table layout with all columns visible
- **Tablet (768px-1024px)**: Condensed table with abbreviated columns
- **Mobile (<768px)**: Card-based layout replacing table entirely

### Implementation Pattern
```tsx
const isMobile = useWindowSize().width < 768

return (
  <div className="w-full">
    {isMobile ? (
      <div className="grid gap-4">
        {domains.map(domain => <DomainMobileCard key={domain.id} domain={domain} />)}
      </div>
    ) : (
      <DomainTable domains={domains} />
    )}
  </div>
)
```

### Mobile Components
- **Domain Cards**: shadcn/ui Card component
  - Shows domain name (no icons)
  - Score and grade prominently displayed
  - Last scan time
  - Action buttons (View | Scan | Delete)
- **Responsive Breakpoints**:
  - Desktop (>1024px): Full tables
  - Tablet (768-1024px): Condensed tables
  - Mobile (<768px): Card layouts

---

## Interactive Elements

### Hover States (From Template)
- **Cards**: Slight shadow increase, border color change
- **Buttons**: Background color shift
- **Table rows**: Subtle background highlight
- **Navigation items**: Background color change

### Focus States (From Template)
- **Interactive elements**: 2px blue outline with 2px offset
- **Form inputs**: Border color change to blue
- **Buttons**: Outline ring around element

### Active States (From Template)
- **Buttons**: Slightly scaled down (0.98 transform)
- **Navigation**: Active item highlighted
- **Tab controls**: Underline or background change

---

## Component Implementation

### Dashboard Cards
- **Component**: shadcn/ui Card component
- **Layout**: 3-card grid (responsive)
- **Content Structure**:
  - Title (small, muted text)
  - Main metric (large, bold)
  - Supporting text (grade/status)
- **Cards**:
  1. Security Score: Number + Grade (color-coded)
  2. Your Domains: Usage count with progress bar
  3. Critical Issues: Count with severity indicator

### Domain Table
- **Component**: shadcn/ui Data Table (TanStack Table)
- **Reference**: https://ui.shadcn.com/docs/components/data-table
- **Columns**:
  - Domain (sortable)
  - Last Scan (relative time or "Never", sortable)
  - Security Score (number + grade badge, sortable)
  - Actions (text buttons: View | Scan | Delete)
- **Features**:
  - Column sorting (domain name, date, score)
  - Pagination (10/20/50 rows per page)
  - Filtering by domain name (search input)
  - Row selection for bulk actions (future: bulk scan)
  - Column visibility toggle
- **Mobile**: Converts to card layout below 768px
- **Empty State**: "No domains added yet" with Add Domain button

### Activity Page
- **Component**: shadcn/ui Data Table (TanStack Table)
- **Reference**: https://ui.shadcn.com/docs/components/data-table
- **Columns**:
  - Date (sortable, default sort desc)
  - Domain (sortable, filterable)
  - Score + Grade (sortable)
  - Actions (Generate Report button)
- **Features**:
  - Global filter (search across all columns)
  - Domain-specific filter (dropdown)
  - Pagination (20 items default)
  - Export to CSV (future enhancement)
- **Mobile**: Cards instead of table

### Reports Page
- **Component**: shadcn/ui Data Table (TanStack Table)
- **Reference**: https://ui.shadcn.com/docs/components/data-table
- **Columns**:
  - Generated Date (sortable, default sort desc)
  - Domain (sortable, filterable)
  - Score + Grade (sortable)
  - File Size (sortable)
  - Actions (Download button)
- **Features**:
  - Domain filter (dropdown or faceted filter)
  - Date range filter (future)
  - Pagination (20 items default)
  - Download generates 1-hour S3 signed URL
  - Bulk download (with row selection)
- **Mobile**: Cards instead of table

### Dashboard Chart
- **Component**: shadcn/ui interactive bar chart (with toggle buttons)
- **Data**: Last 30 days of security scores (0-100 range)
- **Toggle Options**: "All Domains" average vs individual domain scores
- **Colors**: Green theme (hsl(142 70% 45%) primary)
- **Height**: 250px fixed
- **Features**:
  - Toggle buttons show average score (not total)
  - Hover tooltips with exact score and date
  - Responsive layout with proper mobile sizing
  - Month/day labels on X-axis

---

## Accessibility Standards

### WCAG 2.1 AA Compliance
- **Color contrast**: Minimum 4.5:1 ratio for all text
- **Focus indicators**: Visible on all interactive elements
- **Screen readers**: Proper ARIA labels and roles
- **Keyboard navigation**: Full support without mouse

### Screen Reader Support
- Proper ARIA labels for all interactive elements
- Role attributes for semantic meaning
- Score announcements include context (e.g., "85 out of 100, grade B plus")
- Form validation messages announced immediately

### Keyboard Shortcuts
- **Tab navigation**: Logical tab order through interface
- **Enter/Space**: Activate buttons and controls
- **Arrow keys**: Navigate through data tables and lists
- **Escape**: Close modals and dropdowns

---

## Implementation Rules

### ✅ Always Do
- Use exact template component structure and styling
- Maintain original spacing, colors, and typography
- Follow template's responsive breakpoint behavior
- Keep sidebar navigation and profile section unchanged
- Use template's existing icon system and button styles

### ❌ Never Do
- Modify the sidebar layout, width, or profile section
- Change card grid dimensions, spacing, or positioning
- Override template's CSS classes or styling system
- Add custom navigation components outside template structure
- Change the collapsible sidebar behavior or toggle position

### Implementation Priority
1. **Phase 8**: Dashboard with security metric cards (using green theme)
2. **Phase 9**: Domain list page with table (green accents)
3. **Phase 9**: Domain detail page with scan results (green status indicators)
4. **Phase 10**: Settings and profile integration (green theme buttons)
5. **Phase 11**: Report generation UI (green success states)

### CSS Implementation
```css
/* Add to global CSS file */
@layer base {
  :root {
    /* Green theme variables as defined above */
  }
  
  .dark {
    /* Dark mode green theme variables as defined above */
  }
}

/* Security score colors using CSS variables */
.score-excellent { color: hsl(var(--primary)); }
.score-good { color: hsl(142 50% 55%); }
.score-warning { color: hsl(47 80% 50%); }
.score-danger { color: hsl(var(--destructive)); }
```

The goal is seamless integration with the green theme where all UI elements naturally fit the security monitoring context.