# Paperclip UI & Design System Investigation

> Deep-dive research report on the Linear-inspired UI layer, design tokens, components, animations, and visual quality.
> Generated: 2026-03-17

---

## Executive Summary

Paperclip's UI is a **highly polished, Linear-inspired project management interface** built on React 19, Vite 6, TailwindCSS v4, and Radix UI primitives. It features 100+ components, a modern OKLCH color system, full dark mode, real-time websocket updates, drag-and-drop Kanban, a command palette, keyboard shortcuts, and careful attention to animation, accessibility, and mobile responsiveness.

---

## 1. Overall UI Architecture

### Framework & Build Setup

| Layer | Technology |
|-------|-----------|
| **Framework** | React 19.0.0 with TypeScript 5.7.3 |
| **Build Tool** | Vite 6.1.0 with React plugin + TailwindCSS plugin |
| **Routing** | Custom React Router v7.1.5 abstraction (`@/lib/router`) |
| **Server State** | TanStack React Query v5.90.21 |
| **UI State** | React Context (8+ providers) |
| **Package Manager** | pnpm workspace monorepo |

### Application Structure

The UI lives at `ui/src/` with this organization:

- **`/pages`** — 31 main page components (Dashboard, Issues, Projects, Agents, Goals, Approvals, etc.)
- **`/components`** — 80+ UI components organized into:
  - **`/components/ui/`** — 21 base UI components (Button, Badge, Dialog, Card, Command, etc.)
  - **Domain-specific components** — StatusIcon, PriorityIcon, KanbanBoard, IssueRow, CommandPalette, etc.
- **`/context`** — 8 React Context providers for global state
- **`/hooks`** — Custom React hooks for business logic
- **`/lib`** — Utilities including design tokens (`status-colors.ts`), routing, query keys
- **`/adapters`** — AI model/provider adapters
- **`/api`** — API layer abstraction
- **`/plugins`** — Plugin system infrastructure

### Entry Point: Nested Context Providers

The app wraps 11 context providers in a deliberate order:

1. `QueryClientProvider` (TanStack React Query)
2. `ThemeProvider` (dark/light mode)
3. `CompanyProvider` (multi-tenant state)
4. `ToastProvider` (notifications)
5. `LiveUpdatesProvider` (real-time websocket events)
6. `BrowserRouter` (navigation)
7. `TooltipProvider` (Radix UI)
8. `BreadcrumbProvider` (navigation context)
9. `SidebarProvider` (layout state)
10. `PanelProvider` (properties panel)
11. `PluginLauncherProvider` (extensibility)
12. `DialogProvider` (modal management)

---

## 2. Design System & Tokens

### Color System: OKLCH

The design system uses **OKLCH color space** (modern, perceptually uniform) with CSS custom properties:

#### Light Mode

| Token | Value | Purpose |
|-------|-------|---------|
| Background | `oklch(1 0 0)` | White |
| Foreground | `oklch(0.145 0 0)` | Near-black text |
| Card | White with semi-transparent borders | Content containers |
| Muted | `oklch(0.97 0 0)` | Very light gray backgrounds |
| Accent | Light hover/focus backgrounds | Interactive states |
| Primary | `oklch(0.205 0 0)` | Dark gray/black emphasis |

#### Dark Mode

| Token | Value | Purpose |
|-------|-------|---------|
| Background | `oklch(0.145 0 0)` | Very dark gray |
| Foreground | `oklch(0.985 0 0)` | Near-white text |
| Card | `oklch(0.205 0 0)` | Dark gray containers |
| Muted | `oklch(0.269 0 0)` | Medium gray backgrounds |
| Primary | `oklch(0.985 0 0)` | White for contrast |

#### Chart Palette (5 colors)

| Token | Value | Visual |
|-------|-------|--------|
| Chart-1 | `oklch(0.646 0.222 41.116)` | Orange/Amber |
| Chart-2 | `oklch(0.6 0.118 184.704)` | Blue |
| Chart-3 | `oklch(0.398 0.07 227.392)` | Dark Blue |
| Chart-4 | `oklch(0.828 0.189 84.429)` | Yellow-Green |
| Chart-5 | `oklch(0.769 0.188 70.08)` | Orange |

### Border Radius

Mostly `0px` — square corners matching Linear's minimalist aesthetic. Select elements use `rounded-md` (0.5rem).

### Spacing Scale

Tailwind defaults with common patterns: 2px, 4px, 6px, 8px, 12px, 16px, 24px.

### Typography

- **Font stack**: System fonts (inherited)
- **Base size**: 14px (`text-sm`)
- **Heading sizes**: 12px (xs), 13px (sm), 14px (base), 16px (lg), 18px+ (xl, 2xl)
- **Font weights**: 400 (regular), 500 (medium), 600 (semibold), 700 (bold)

### Theme Implementation

- Stored in `localStorage` as `paperclip.theme`
- Applied via `dark` class on `<html>` root element
- Meta `theme-color` dynamically updates: `#18181b` (dark) / `#ffffff` (light)
- Respects `prefers-reduced-motion: reduce` system preference
- Uses `@theme` inline in Tailwind CSS v4 with CSS custom properties
- Sidebar has its own dedicated color palette

---

## 3. Component Library

### UI Foundation (shadcn/Radix-based)

All base components use **Radix UI** primitives with custom styling:

| Component | Details |
|-----------|---------|
| **Button** | Variants: default, destructive, outline, secondary, ghost, link. Sizes: xs, sm, default, lg, icon, icon-sm, icon-xs. Smooth transitions. |
| **Badge** | `rounded-full` with variants: default, secondary, destructive, outline, ghost, link |
| **Card** | Container with CardHeader, CardTitle, CardDescription, CardContent, CardFooter |
| **Dialog** | Full-screen overlay with smooth zoom/fade animations and close button |
| **Popover** | Floating panels with Radix positioning; used for status/priority selectors |
| **Command** | Search-based palette (cmdk library) with grouped sections |
| **Dropdown Menu** | Radix-based with separators, checkboxes, keyboard shortcuts |
| **Select** | Accessible dropdown |
| **Input/Textarea** | Form inputs with focus states and validation styling |
| **Tabs** | Tab navigation with animated underline |
| **Sheet** | Slide-out drawer for mobile navigation |
| **Checkbox** | Custom styled with focus rings |
| **Avatar** | User avatars with fallback initials; supports group display |
| **Tooltip** | Radix tooltip with portal rendering |
| **Scrollable Area** | Custom scrollbar styling |
| **Collapsible** | Expand/collapse sections |
| **Skeleton** | Loading placeholder components |
| **Breadcrumb** | Navigation breadcrumbs |

### Domain-Specific Components

#### Status & Indicators

- **StatusIcon** — Circular border icon per issue status (backlog, todo, in_progress, in_review, done, cancelled, blocked). Solid circle for "done."
- **StatusBadge** — Colored badge with background for all entity statuses (agent, goal, run, approval, issue)
- **PriorityIcon** — Arrow icons with color coding (critical, high, medium, low)
- **Identity** — Agent/user avatars with auto-generated initials and name labels
- **AgentStatusDot** — Small animated indicator; pulsing animation for "running" state

#### Layout & Navigation

- **Layout** — Main app shell: sidebar + breadcrumb + main content + properties panel
- **Sidebar** — 260px wide, collapsible left navigation
- **CompanyRail** — Vertical company switcher with drag-reorder (thin rail on desktop)
- **BreadcrumbBar** — Context-aware navigation breadcrumbs
- **MobileBottomNav** — Bottom navigation for mobile (auto-hide on scroll)
- **PageSkeleton** — Loading skeleton for list/detail views
- **PropertiesPanel** — Right slide-out panel (320px, collapsible with `]` key)

#### Lists & Views

- **IssuesList** — Searchable issue list with live run indicators
- **IssueRow** — Issue row with status, priority, assignee, unread dot animation
- **KanbanBoard** — Drag-and-drop board with 7 columns using `@dnd-kit/core`. Cards show issue ID, title, priority, assignee with live indicator pulse.
- **EntityRow** — Generic row for projects/goals with title, subtitle, trailing metadata
- **FilterBar** — Active filter display with remove buttons

#### Data Visualization

- **RunActivityChart** — 14-day stacked bar chart (succeeded=green, failed=red, other=gray)
- **PriorityChart** — Categorical breakdown
- **IssueStatusChart** — Status distribution
- **SuccessRateChart** — Success percentage visualization
- **QuotaBar** — Progress bar with dynamic color (green <=70%, yellow 70-90%, red >90%)
- **ActivityRow** — Dashboard activity entry with animated entrance and highlight

#### Rich Content & Forms

- **MarkdownEditor** — MDXEditor with custom theming and Catppuccin Mocha code highlighting
- **InlineEditor** — Editable fields with save/cancel
- **NewIssueDialog** — Create issue with title, description, status, priority, project, assignee
- **InlineEntitySelector** — Searchable entity picker
- **AgentConfigForm** — Complex agent configuration form
- **JsonSchemaForm** — Generic form from JSON schema

#### Special

- **CommandPalette** — `Cmd+K` global search & action palette with sections
- **CommentThread** — Discussion threads with syntax-highlighted mentions
- **LiveRunWidget** — Real-time execution status indicator
- **OnboardingWizard** — First-run setup flow
- **AsciiArtAnimation** — Custom ASCII art animation (paperclip sprites at 24 FPS); respects `prefers-reduced-motion`
- **WorktreeBanner** — Development branch indicator
- **PluginSlotOutlet** — Dynamic plugin injection points

---

## 4. Status & Priority Indicators

This is the heart of the Linear-inspired experience. All indicators are defined canonically in `ui/src/lib/status-colors.ts`.

### Issue Status System

| Status | Icon Color (light/dark) | Badge Background | Meaning |
|--------|------------------------|-------------------|---------|
| **backlog** | muted-foreground | muted | Not started |
| **todo** | blue-600 / blue-400 | blue-100 / blue-900/50 | Planned |
| **in_progress** | yellow-600 / yellow-400 | yellow-100 / yellow-900/50 | Actively being worked on |
| **in_review** | violet-600 / violet-400 | violet-100 / violet-900/50 | Awaiting approval |
| **done** | green-600 / green-400 | green-100 / green-900/50 | Completed (filled circle) |
| **cancelled** | neutral-500 | muted | Abandoned |
| **blocked** | red-600 / red-400 | red-100 / red-900/50 | Stuck, needs unblocking |

### Priority System

| Priority | Icon | Color (light/dark) |
|----------|------|-------------------|
| **critical** | AlertTriangle | red-600 / red-400 |
| **high** | ArrowUp | orange-600 / orange-400 |
| **medium** | Minus | yellow-600 / yellow-400 |
| **low** | ArrowDown | blue-600 / blue-400 |

### Agent Status Indicators

| Status | Badge Style | Dot Color | Animation |
|--------|-------------|-----------|-----------|
| **active** | green-100/green-900/50 | green-400 | None |
| **running** | cyan-100/cyan-900/50 | cyan-400 | `animate-pulse` |
| **paused** | orange-100/orange-900/50 | yellow-400 | None |
| **idle** | yellow-100/yellow-900/50 | yellow-400 | None |
| **pending_approval** | amber-100/amber-900/50 | amber-400 | None |
| **error** | red-100/red-900/50 | red-400 | None |
| **archived** | muted | neutral-400 | None |

### Run/Execution Status

| Status | Badge | Meaning |
|--------|-------|---------|
| **pending** | yellow | Queued |
| **succeeded** | green | Completed successfully |
| **failed** | red | Execution error |
| **timed_out** | orange | Max duration exceeded |
| **terminated** | red | Manually stopped |

### Approval Status

| Status | Badge |
|--------|-------|
| **pending_approval** | amber |
| **revision_requested** | amber |
| **approved** | green |
| **rejected** | red |

### Visual Hierarchy

Status/priority icons appear consistently in:
- Issue rows (left side on mobile, inline on desktop)
- Kanban cards (inline with assignee)
- Command palette results
- Issue detail headers
- Dashboard activity feed

---

## 5. Linear-Inspired UI Patterns

### Command Palette (`Cmd+K`)

- Global search & action palette
- Sections: Actions, Pages, Issues, Agents, Projects
- Shows issue identifiers, avatars, keyboard shortcuts
- Smooth overlay modal with search input
- Top 10 results per section (case-insensitive search)
- Live updates from API as you type

### Keyboard Shortcuts

| Key | Action |
|-----|--------|
| `Cmd/Ctrl+K` | Open command palette |
| `C` | Create new issue (blocked in inputs) |
| `[` | Toggle sidebar |
| `]` | Toggle properties panel |
| `Escape` | Close modals/popovers |

### Sidebar Navigation

- 260px fixed width with company brand color indicator (4px colored square)
- Sections: Quick actions, Work (Issues, Goals), Projects, Agents, Company
- Active state with background highlight
- Live run count badge on Dashboard
- Unread count badge on Inbox

### Issue Detail View

- Breadcrumb navigation at top
- **Left**: Issue title, description, comments, documents
- **Right**: Properties panel (status, priority, assignee, dates)
- Tab-based organization for transcript/activity
- Comment threading with agent mentions

### List Views

- Minimal borders (top and bottom only)
- Hover states with `bg-accent/50` transition
- Metadata inline (desktop) or below title (mobile)
- Status/priority icons left-aligned
- Unread indicators (blue dot that fades)

### Modals & Dialogs

- Centered with semi-transparent backdrop
- Smooth zoom-in animation: `zoom-in-[0.97]`
- Slide from top: `slide-in-from-top-[1%]`
- Max-width 640px for most dialogs

---

## 6. Animations & Transitions

### CSS Transitions

- Global transition on color, background-color, border-color, box-shadow, opacity
- Duration: 150ms (fast UI) or 200ms (observable changes)
- Easing: default cubic-bezier

### Keyframe Animations

| Animation | Purpose | Details |
|-----------|---------|---------|
| **dashboard-activity-enter** | Activity row entry | 520ms: translate from -14px, scale 0.985, blur 4px; overshoot to Y=2px then settle. `cubic-bezier(0.16, 1, 0.3, 1)` |
| **dashboard-activity-highlight** | Activity highlight glow | 920ms: inset border accent -> transparent, background glow fade |
| **animate-pulse** | Agent "running" status | Opacity fade in/out loop |
| **Radix built-in** | Dialogs/dropdowns | fade-in-0, zoom-in-95, slide-in (100ms) |

### Dialog Animations

```css
data-[state=open]:animate-in
data-[state=closed]:animate-out
data-[state=closed]:fade-out-0
data-[state=open]:fade-in-0
data-[state=closed]:zoom-out-[0.97]
data-[state=open]:zoom-in-[0.97]
```

### Reduced Motion Support

- Respects `prefers-reduced-motion: reduce`
- AsciiArtAnimation completely disabled
- Other animations gracefully degrade

### Editor Theming

- MDXEditor with Catppuccin Mocha dark theme
- Background: `#1e1e2e`, text: `#cdd6f4`
- Code mirror syntax highlighting with custom colors

### Custom Scrollbars

- Dark mode: 8px wide, dark background with hover effects
- Auto-hide: transparent until container hover
- 150ms transitions

---

## 7. Layout Patterns

### Master Layout

```
+----------------------------------------------+
| Skip-to-main-content (sr-only)               |
+----------------------------------------------+
| WorktreeBanner (if in dev worktree)          |
+----------------------------------------------+
| CompanyRail (48px) | Sidebar (260px)         |
|                    | - Company name + search  |
|                    | - New Issue button       |
|                    | - Dashboard, Inbox       |
|                    | - Projects (expandable)  |
|                    | - Agents (expandable)    |
|                    | - Company section        |
+----------------------------------------------+
| BreadcrumbBar (sticky on mobile)             |
| Main Content Outlet (flex-1, scrollable)     |
| PropertiesPanel (right, hidden < 768px)      |
+----------------------------------------------+
| MobileBottomNav (visible < 768px)            |
+----------------------------------------------+
| CommandPalette (overlay)                     |
| Modals, Toasts (bottom-right)               |
+----------------------------------------------+
```

### Responsive Breakpoints

| Breakpoint | Layout |
|------------|--------|
| Mobile (`< 640px`) | Single column, bottom nav, slide-out sidebar |
| Tablet (`640-1024px`) | Sidebar visible but narrower, toggleable panel |
| Desktop (`> 1024px`) | Full 3-column: sidebar + main + properties |

### Mobile Navigation

- **Swipe gestures**: 30px left edge zone, 50px minimum swipe distance
- **Auto-hide bottom nav**: on scroll (8px delta threshold)
- **Backdrop**: `bg-black/50` when sidebar open
- **Transitions**: `duration-100 ease-out`

---

## 8. Data Visualization

### Charts

Custom React components (no charting library dependency):

- **14-day activity chart**: Stacked bar chart (green=succeeded, red=failed, gray=other)
- Date labels every 7 days (M/D format)
- Legend with colored dots
- Responsive to viewport

### Progress Indicators

**QuotaBar**: Linear progress bar with dynamic fill:
- Green: 0-70%
- Yellow: 70-90%
- Red: 90-100%+
- Optional "deficit notch" indicator
- 150ms width transitions

### Visual Feedback

- **Drag-and-drop**: opacity-30 when dragging, shadow-lg in overlay
- **Hover**: `bg-accent/50` with smooth transition
- **Focus rings**: `ring-ring/50 ring-[3px]`
- **Invalid states**: `ring-destructive/20 border-destructive`

### Mermaid Diagrams

- Mermaid v11.12.0 for diagram rendering
- Custom styling with subtle backgrounds
- Error messages with dedicated color

---

## 9. Notable UI Quality Details

### Polished Interactions

1. **Live Indicators**: Pulsing cyan-400 dot on issues being actively worked on (with opacity-75 halo)
2. **Unread Dot Fading**: Issue list unread dots smoothly fade over 300ms
3. **Activity Animation**: Dashboard entries animate in (520ms entrance) + 920ms highlight glow that fades
4. **Drag-and-drop Feedback**: Kanban cards show ring shadow as overlay, ghosted opacity when dragging
5. **Auto-hide Mobile Nav**: Navigation hides/shows on scroll direction (8px hysteresis)
6. **Sidebar Swipe**: Mobile swipe from left edge (30px zone)
7. **Properties Panel Transition**: Width transitions smoothly (200ms)
8. **Contrast Text Calculation**: Dynamic black/white text on colored backgrounds (WCAG luminance)

### Typography Polish

- `text-wrap: balance` on headings
- `tabular-nums` on counts/dates
- Monospace for identifiers and codes
- Consistent hierarchy

### Accessibility

- ARIA labels on buttons
- Focus rings: `focus-visible:ring-[3px] focus-visible:ring-ring/50`
- Skip-to-main-content link (sr-only, shows on focus)
- Semantic HTML (nav, main, aside, proper roles)
- Color not relied upon alone (icon shapes differ per status)
- 44px minimum touch targets on mobile

### Performance

- Virtualized scrolling in list views
- Lazy component loading via React Router
- Query caching with 30s stale time
- Debounced search (300ms) and autosave (800ms)
- Image lazy loading (avatars)
- CSS transitions over JS animations

### Brand Elements

- Company brand color indicator in sidebar (4px colored square)
- Dynamic theming based on brand color
- Issue prefix in breadcrumbs
- Custom icons per entity type (Hexagon=projects, CircleDot=issues, Target=goals, Bot=agents)

---

## 10. Tech Stack Summary

### Core

| Dep | Version |
|-----|---------|
| React | 19.0.0 |
| React Router | 7.1.5 |
| Vite | 6.1.0 |
| TailwindCSS | 4.0.7 |
| TypeScript | 5.7.3 |

### Component Primitives

- **Radix UI** — headless accessible components
- **cmdk** — command palette
- **@dnd-kit** — drag and drop
- **Lucide React** — icons

### Styling

- **TailwindCSS v4** — utility-first CSS
- **class-variance-authority (CVA)** — component variants
- **tailwind-merge** — class merging
- **clsx** — conditional classes

### State & Data

- **TanStack React Query v5** — server state
- **React Context** — UI state

### Rich Content

- **MDXEditor v3.52.4** — markdown editing
- **react-markdown** — rendering
- **remark-gfm** — GitHub Flavored Markdown
- **Mermaid v11.12.0** — diagrams
- **@tailwindcss/typography** — prose styling

---

## Conclusion

Paperclip's UI achieves the "Linear quality bar" through:

1. **Modern Design Foundations** — OKLCH color, square corners, minimal borders, full dark mode
2. **Component-Driven Architecture** — 100+ components built from Radix UI with consistent theming
3. **Status-Centric UX** — 7 issue states, 4 priorities, 7 agent states, all color-coded with distinct icons
4. **Responsive & Accessible** — Mobile-first, swipe gestures, keyboard-first, ARIA labels, WCAG contrast
5. **Lively Animations** — Entrance animations, hover effects, smooth transitions, reduced-motion support
6. **Real-time Everything** — Websocket-driven live updates, pulsing indicators, no polling required
7. **Developer-Friendly Stack** — TailwindCSS v4, React 19, Vite 6, TypeScript — modern and maintainable
