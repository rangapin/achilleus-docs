# Achilleus Design Documentation

## Design Philosophy

**Extend, Don't Rebuild**: Achilleus builds upon the Laravel React starter template structure without modifying the existing layout, navigation, or visual design. The interface uses the exact same sidebar, card grid, spacing, and component styling while filling content areas with security monitoring data.

**Core Principle**: The final project should look identical to the starter template with security-focused content replacing the placeholder data.

---

## Template Structure Analysis

Based on the Laravel React starter template screenshot, the layout consists of:

### Sidebar (256px width)
- **Logo area**: "Laravel Starter Kit" branding at top
- **Navigation menu**: Clean list-style navigation items
- **Profile section**: Bottom-positioned dropdown with avatar, user name, and menu
- **Collapsible toggle**: Hamburger icon in exact same position for sidebar collapse

### Main Content Area
- **Header**: Page title with optional action buttons on the right
- **4-card grid**: Perfectly spaced card layout (2x2 on desktop)
- **Chart area**: Full-width section below cards for data visualization
- **Consistent spacing**: Proper margins and padding throughout

### Visual Design Elements
- **Dark theme**: Neutral dark colors with proper contrast
- **Card styling**: Subtle borders, shadows, and rounded corners
- **Typography**: Clean hierarchy with proper font weights and sizes
- **Component styling**: Consistent button styles, form inputs, and interactive elements

---

## Color System

### Core Palette (From Template)
- **Background**: `#09090b` (neutral-950) - Main background
- **Card Background**: `#171717` (neutral-900) - Elevated surfaces  
- **Border**: `#262626` (neutral-800) - Dividers and card borders
- **Muted Background**: `#262626` (neutral-800) - Subtle backgrounds
- **Hover State**: `#404040` (neutral-700) - Interactive hover effects

### Text Colors (From Template)
- **Primary**: `#ffffff` - Main content and headings
- **Secondary**: `#a0a0a0` - Supporting text and descriptions
- **Muted**: `#6b7280` - Disabled states and placeholders
- **Accent**: `#3b82f6` - Links and primary actions

### Security-Specific Status Colors
- **Success**: `#22c55e` - Scores 80+, positive states, A/B+ grades
- **Warning**: `#f59e0b` - Scores 60-79, caution states, C/D grades  
- **Danger**: `#ef4444` - Scores <60, critical issues, F grade
- **Info**: `#3b82f6` - Informational states, scanning progress

### Grade-Specific Colors
- **A+ Grade**: `#22c55e` (bright green)
- **A Grade**: `#34d399` (green)
- **B+ Grade**: `#fbbf24` (yellow)
- **B Grade**: `#f59e0b` (orange)
- **C Grade**: `#fb923c` (light orange)
- **D Grade**: `#f87171` (light red)
- **F Grade**: `#ef4444` (red)

---

## Typography

### Font Stack (From Template)
```css
font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', 'Inter', 
             'Roboto', 'Oxygen', 'Ubuntu', sans-serif;
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
│ Sidebar (256px)    │  Main Content Area                         │
│ ┌────────────────┐ │                                            │
│ │ Laravel Starter│ │  Dashboard                    [Scan Now]   │
│ │ Kit            │ │                                            │
│ └────────────────┘ │                                            │
│  📊 Dashboard      │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐
│  🌐 Domains        │  │  Score   │ │ Domains  │ │Last Scan │ │ Issues   │
│  📈 Activity       │  │   85     │ │   3/10   │ │ 2hr ago  │ │    0     │
│  📄 Reports        │  │  B+      │ │  Active  │ │example.com│ │  None    │
│                    │  └──────────┘ └──────────┘ └──────────┘ └──────────┘
│  ─────────────     │                                            │
│                    │  Security Score Trend                      │
│  ┌──────────────┐  │  ┌─────────────────────────────────────┐   │
│  │👤 User Name  ▼│  │  │ [7d] [30d] [90d] [1y]              │   │
│  └──────────────┘  │  │                                     │   │
│                    │  │ ████████████████████████████████    │   │
│                    │  │ Chart showing score over time       │   │
│                    │  │                                     │   │
│                    │  └─────────────────────────────────────┘   │
└────────────────────┴─────────────────────────────────────────────┘
```

### Sidebar Navigation Items
Replace template navigation with security-focused items:
- **📊 Dashboard** - Main overview page
- **🌐 Domains** - Domain management and list
- **📈 Activity** - Recent scans and history
- **📄 Reports** - Generated PDF reports

### Dashboard Cards Content
Replace template card content with security metrics:

#### Security Score Card
- **Large number**: Security score (85)
- **Grade indicator**: Letter grade (B+)
- **Color coding**: Green/yellow/red based on score

#### Active Domains Card  
- **Usage count**: Active domains (3/10)
- **Status**: "Active" or similar
- **Progress indicator**: Visual representation of limit usage

#### Last Scan Date Card
- **Timestamp**: "2 hours ago"
- **Domain name**: "example.com"
- **Status**: Completion indicator

#### Critical Issues Card
- **Count**: Critical issues (0)
- **Status**: "None" or issue description
- **Alert level**: Color coding for severity

### Chart Area
Full-width section below cards showing:
- **Time period toggles**: 7d/30d/90d/1y buttons
- **Score trend chart**: Line or area chart
- **Interactive tooltips**: Hover details for data points
- **Responsive design**: Adapts to sidebar collapsed/expanded states

---

## Domain List Page Layout

```
┌─────────────────────────────────────────────────────────────────┐
│ Sidebar │  Domains                      [Add Domain] [Scan All] │
│         │                                                        │
│         │  ┌────────────────────────────────────────────────────┐ │
│         │  │ Domain         Last Scan      Score    Status    │ │
│         │  ├────────────────────────────────────────────────────┤ │
│         │  │ 🔐 example.com  2 hours ago    85 B+    ● Active │ │
│         │  │ 🔐 example.com  1 day ago      92 A     ● Active │ │
│         │  │ 🔐 mysite.io    3 days ago     67 C     ● Active │ │
│         │  │                                                    │ │
│         │  │ Actions: 👁️ View Details  🔍 Scan Now  🗑️ Delete  │ │
│         │  └────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

### Table Structure (Using Template Table Component)
- **Domain column**: SSL icon + domain name
- **Last Scan column**: Relative time (2 hours ago)
- **Score column**: Number + grade with color coding
- **Status column**: Active/inactive with status dot
- **Actions column**: View/scan/delete icon buttons

### Action Buttons (Template Button Styles)
- **Add Domain**: Primary button (top right)
- **Scan All**: Secondary button (top right)
- **Individual actions**: Icon buttons in table rows

---

## Domain Detail Page Layout

```
┌─────────────────────────────────────────────────────────────────┐
│ Sidebar │  example.com                          [New Scan]      │
│         │  Last scanned 2 hours ago                             │
│         │                                                        │
│         │  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐     │
│         │  │ Overall     │ │ SSL/TLS     │ │ Headers     │     │
│         │  │   85/100    │ │   90/100    │ │   75/100    │     │
│         │  │   Grade B+  │ │   Grade A   │ │   Grade C   │     │
│         │  └─────────────┘ └─────────────┘ └─────────────┘     │
│         │                                                        │
│         │  Security Analysis Details                             │
│         │  ┌───────────────────────────────────────────────────┐ │
│         │  │ ✅ SSL/TLS Certificate               
│         │  │    • Valid for 90 more days                       │ │
│         │  │    • TLS 1.3 supported                            │ │
│         │  │_____________________________________________________
│         │  │ ⚠️  Security Headers                 
│         │  │    • Missing CSP header                           │ │
│         │  │    • HSTS configured correctly                    │ │
│         │  _______________________________________________________│    
│         │  │ ✅ DNS & Email Security              
│         │  │    • SPF record valid                             │ │
│         │  │    • DMARC policy: quarantine                     │ │
│         │  └───────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

### Module Results Display
Each scanner module shows:
- **Status icon**: ✅ (good), ⚠️ (warning), ❌ (error)
- **Module name**: SSL/TLS Certificate, Security Headers, etc.
- **Score**: Numerical score out of 100
- **Details list**: Specific findings and recommendations
- **Expandable sections**: Click to show/hide details

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

### Scanning Progress
```
┌─────────────────────────────────────────────────┐
│ Scanning example.com...                         │
│                                                  │
│ [████████████████░░░░░░░░░░░░] 65%              │
│                                                  │
│ Currently checking: Security Headers             │
│                                                  │
│ This usually takes 15-30 seconds                │
└─────────────────────────────────────────────────┘
```

### No Domains State
```
┌─────────────────────────────────────────────────┐
│             🔐                                   │
│                                                  │
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
│ Last: 2 hours ago       │
│ Status: ● Active        │
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

### Mobile Domain Card Component
```tsx
import { Shield, MoreHorizontal } from 'lucide-react'
import { Card, CardContent } from '@/components/ui/card'
import { Badge } from '@/components/ui/badge'
import { Button } from '@/components/ui/button'

interface DomainMobileCardProps {
  domain: Domain
  onViewDetails: (domainId: string) => void
  onScanNow: (domainId: string) => void
  onDelete: (domainId: string) => void
}

export function DomainMobileCard({ domain, onViewDetails, onScanNow, onDelete }: DomainMobileCardProps) {
  return (
    <Card>
      <CardContent className="p-4">
        <div className="flex items-start justify-between mb-3">
          <div className="flex items-center space-x-2">
            <Shield className="h-4 w-4 text-green-500" />
            <span className="font-medium">{domain.url}</span>
          </div>
          <div className="flex items-center space-x-2">
            <span className="text-2xl font-bold">{domain.lastScanScore ?? '-'}</span>
            {domain.grade && (
              <Badge variant="secondary">{domain.grade}</Badge>
            )}
          </div>
        </div>
        
        <div className="space-y-2 mb-4">
          <div className="text-sm text-muted-foreground">
            Last scan: {domain.lastScanAt ? formatRelative(domain.lastScanAt, new Date()) : 'Never'}
          </div>
          <div className="flex items-center space-x-2">
            <div className={`h-2 w-2 rounded-full ${
              domain.isActive ? 'bg-green-500' : 'bg-gray-500'
            }`}></div>
            <span className="text-sm">{domain.isActive ? 'Active' : 'Inactive'}</span>
          </div>
        </div>
        
        <div className="flex space-x-2">
          <Button 
            variant="outline" 
            size="sm" 
            className="flex-1"
            onClick={() => onViewDetails(domain.id)}
          >
            View
          </Button>
          <Button 
            variant="outline" 
            size="sm" 
            className="flex-1"
            onClick={() => onScanNow(domain.id)}
          >
            Scan
          </Button>
          <Button 
            variant="outline" 
            size="sm" 
            className="text-red-600"
            onClick={() => onDelete(domain.id)}
          >
            Delete
          </Button>
        </div>
      </CardContent>
    </Card>
  )
}
```

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

### Dashboard Cards (Using Template Card Component)
```tsx
import { Shield } from 'lucide-react'
import { Card, CardContent } from '@/components/ui/card'

interface SecurityScoreCardProps {
  score: number
  grade: string
  className?: string
}

export function SecurityScoreCard({ score, grade, className }: SecurityScoreCardProps) {
  return (
    <Card className={className}>
      <CardContent className="p-6">
        <div className="flex items-center justify-between">
          <div>
            <p className="text-sm font-medium text-muted-foreground">
              Security Score
            </p>
            <div className="flex items-baseline space-x-2">
              <div className="text-2xl font-bold">{score}</div>
              <div className="text-sm font-medium text-yellow-500">{grade}</div>
            </div>
          </div>
          <Shield className="h-4 w-4 text-muted-foreground" />
        </div>
      </CardContent>
    </Card>
  )
}
```

### Domain Table (Using Template Table Component)
```tsx
import { Shield, MoreHorizontal } from 'lucide-react'
import { formatRelative } from 'date-fns'
import {
  Table,
  TableBody,
  TableCell,
  TableHead,
  TableHeader,
  TableRow,
} from '@/components/ui/table'
import {
  DropdownMenu,
  DropdownMenuContent,
  DropdownMenuItem,
  DropdownMenuTrigger,
} from '@/components/ui/dropdown-menu'
import { Button } from '@/components/ui/button'
import { Badge } from '@/components/ui/badge'

interface Domain {
  id: string
  url: string
  lastScanAt: Date | null
  lastScanScore: number | null
  grade: string | null
  isActive: boolean
}

interface DomainTableProps {
  domains: Domain[]
  onViewDetails: (domainId: string) => void
  onScanNow: (domainId: string) => void
  onDelete: (domainId: string) => void
}

export function DomainTable({ domains, onViewDetails, onScanNow, onDelete }: DomainTableProps) {
  const handleAction = (action: string, domainId: string) => (e: React.MouseEvent) => {
    e.preventDefault()
    switch (action) {
      case 'view':
        onViewDetails(domainId)
        break
      case 'scan':
        onScanNow(domainId)
        break
      case 'delete':
        onDelete(domainId)
        break
    }
  }

  return (
    <Table>
      <TableHeader>
        <TableRow>
          <TableHead>Domain</TableHead>
          <TableHead>Last Scan</TableHead>
          <TableHead>Score</TableHead>
          <TableHead>Status</TableHead>
          <TableHead>Actions</TableHead>
        </TableRow>
      </TableHeader>
      <TableBody>
        {domains.map((domain) => (
          <TableRow key={domain.id}>
            <TableCell className="font-medium">
              <div className="flex items-center space-x-2">
                <Shield className="h-4 w-4 text-green-500" />
                <span>{domain.url}</span>
              </div>
            </TableCell>
            <TableCell>
              {domain.lastScanAt 
                ? formatRelative(domain.lastScanAt, new Date())
                : 'Never'
              }
            </TableCell>
            <TableCell>
              <div className="flex items-center space-x-2">
                <span>{domain.lastScanScore ?? '-'}</span>
                {domain.grade && (
                  <Badge variant="secondary">{domain.grade}</Badge>
                )}
              </div>
            </TableCell>
            <TableCell>
              <div className="flex items-center space-x-2">
                <div className={`h-2 w-2 rounded-full ${
                  domain.isActive ? 'bg-green-500' : 'bg-gray-500'
                }`}></div>
                <span>{domain.isActive ? 'Active' : 'Inactive'}</span>
              </div>
            </TableCell>
            <TableCell>
              <DropdownMenu>
                <DropdownMenuTrigger asChild>
                  <Button variant="ghost" className="h-8 w-8 p-0">
                    <MoreHorizontal className="h-4 w-4" />
                  </Button>
                </DropdownMenuTrigger>
                <DropdownMenuContent align="end">
                  <DropdownMenuItem onClick={handleAction('view', domain.id)}>
                    View Details
                  </DropdownMenuItem>
                  <DropdownMenuItem onClick={handleAction('scan', domain.id)}>
                    Scan Now
                  </DropdownMenuItem>
                  <DropdownMenuItem 
                    onClick={handleAction('delete', domain.id)}
                    className="text-red-600"
                  >
                    Delete
                  </DropdownMenuItem>
                </DropdownMenuContent>
              </DropdownMenu>
            </TableCell>
          </TableRow>
        ))}
      </TableBody>
    </Table>
  )
}
```

### Chart Component (Using Template Chart Patterns)
```tsx
import { useState } from 'react'
import { Card, CardContent, CardHeader, CardTitle } from '@/components/ui/card'
import { Button } from '@/components/ui/button'
import {
  LineChart,
  Line,
  XAxis,
  YAxis,
  CartesianGrid,
  Tooltip,
  ResponsiveContainer,
} from 'recharts'

interface ScoreDataPoint {
  date: string
  score: number
  timestamp: Date
}

interface SecurityScoreChartProps {
  data: ScoreDataPoint[]
  className?: string
}

type TimePeriod = '7d' | '30d' | '90d' | '1y'

export function SecurityScoreChart({ data, className }: SecurityScoreChartProps) {
  const [selectedPeriod, setSelectedPeriod] = useState<TimePeriod>('90d')

  const handlePeriodChange = (period: TimePeriod) => (e: React.MouseEvent<HTMLButtonElement>) => {
    e.preventDefault()
    setSelectedPeriod(period)
  }

  const filteredData = data.filter((point) => {
    const now = new Date()
    const cutoff = new Date()
    
    switch (selectedPeriod) {
      case '7d':
        cutoff.setDate(now.getDate() - 7)
        break
      case '30d':
        cutoff.setDate(now.getDate() - 30)
        break
      case '90d':
        cutoff.setDate(now.getDate() - 90)
        break
      case '1y':
        cutoff.setFullYear(now.getFullYear() - 1)
        break
    }
    
    return point.timestamp >= cutoff
  })

  return (
    <Card className={className}>
      <CardHeader>
        <CardTitle>Security Score Trend</CardTitle>
        <div className="flex space-x-2">
          <Button 
            variant={selectedPeriod === '7d' ? 'default' : 'outline'} 
            size="sm"
            onClick={handlePeriodChange('7d')}
          >
            7d
          </Button>
          <Button 
            variant={selectedPeriod === '30d' ? 'default' : 'outline'} 
            size="sm"
            onClick={handlePeriodChange('30d')}
          >
            30d
          </Button>
          <Button 
            variant={selectedPeriod === '90d' ? 'default' : 'outline'} 
            size="sm"
            onClick={handlePeriodChange('90d')}
          >
            90d
          </Button>
          <Button 
            variant={selectedPeriod === '1y' ? 'default' : 'outline'} 
            size="sm"
            onClick={handlePeriodChange('1y')}
          >
            1y
          </Button>
        </div>
      </CardHeader>
      <CardContent>
        <ResponsiveContainer width="100%" height={300}>
          <LineChart data={filteredData}>
            <CartesianGrid strokeDasharray="3 3" />
            <XAxis dataKey="date" />
            <YAxis domain={[0, 100]} />
            <Tooltip 
              formatter={(value: number) => [`${value}%`, 'Security Score']}
              labelFormatter={(label: string) => `Date: ${label}`}
            />
            <Line 
              type="monotone" 
              dataKey="score" 
              stroke="#22c55e" 
              strokeWidth={2}
              dot={{ fill: '#22c55e', strokeWidth: 2, r: 4 }}
              activeDot={{ r: 6 }}
            />
          </LineChart>
        </ResponsiveContainer>
      </CardContent>
    </Card>
  )
}
```

---

## Accessibility Standards

### WCAG 2.1 AA Compliance
- **Color contrast**: Minimum 4.5:1 ratio for all text
- **Focus indicators**: Visible on all interactive elements
- **Screen readers**: Proper ARIA labels and roles
- **Keyboard navigation**: Full support without mouse

### Screen Reader Support
```tsx
import { Card, CardContent } from '@/components/ui/card'

interface AccessibleScoreDisplayProps {
  score: number
  grade: string
  label?: string
}

export function AccessibleScoreDisplay({ 
  score, 
  grade, 
  label = "Security score" 
}: AccessibleScoreDisplayProps) {
  const ariaLabel = `${label} ${score} out of 100, grade ${grade.replace('+', ' plus')}`
  
  return (
    <Card>
      <CardContent>
        <div aria-label={ariaLabel} role="img">
          <div className="text-2xl font-bold">{score}</div>
          <div className="text-sm">{grade}</div>
        </div>
      </CardContent>
    </Card>
  )
}
```

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
1. **Phase 8**: Dashboard with security metric cards
2. **Phase 9**: Domain list page with table
3. **Phase 9**: Domain detail page with scan results
4. **Phase 10**: Settings and profile integration
5. **Phase 11**: Report generation UI

The goal is seamless integration where users cannot distinguish between template components and security-specific additions.