# DeepFlow Frontend — Technical Brief

> **Purpose:** Specification for building the DeepFlow web application frontend.
> **Target:** Claude Code / Codex implementation agent.
> **Layout reference:** [wireframes.html](wireframes.html) — Option B (chat always on).
> **Design system:** [guidelines/](index.html) — full component specs, colour, typography, spacing.

---

## 1. Product Overview

DeepFlow is an AI-powered workflow orchestration platform. Users create hierarchical DAG-based workflows, assign tasks to humans and AI agents, and interact with an always-on AI chat panel. This is a **reskin of an existing product** — the DAG engine, chat backend, Slack integration, and @mentions are already in production. This brief covers the **frontend shell** only.

### Core Views
1. **Graph View** — DAG canvas with draggable nodes, edges, zones, detail panel
2. **List View** — Sortable task table with expandable parent/child rows
3. **Workspace (Dashboard)** — Landing page with project cards, stats, attention items
4. **Gallery** — Project browser with templates and project cards
5. **Active Task** — Task-focused working state with contextual chat
6. **Settings** — App configuration with section-based navigation

### Fixed Layout (Option B)
| Panel | Width | Behaviour |
|-------|-------|-----------|
| Sidebar | 56px | Always collapsed (icons only). Expands to 240px on hover/click in non-Option-B mode |
| Chat Panel | 320px | Always visible. Context updates per view/task |
| Top Bar | 44px h | Breadcrumbs + view switcher + search + filters |
| Hub Bar | 36px / ~220px | Collapsible. Graph + Active Task views only |
| Detail Panel | 360px | Graph + Active Task views. Overlay in List. Absent in Workspace/Gallery/Settings |
| Content Area | Remaining | 464px (with detail) / 824px (without detail) of 1200px |

---

## 2. Tech Stack

### Core
| Layer | Choice | Rationale |
|-------|--------|-----------|
| Framework | **Next.js 15 (App Router)** | RSC for initial load, server actions, middleware for auth |
| Language | **TypeScript (strict)** | Non-negotiable |
| Styling | **Tailwind CSS 4** + CSS custom properties | Design tokens from guidelines map directly to CSS vars |
| Component library | **Radix UI (headless)** | Accessible primitives. No visual opinions to fight |
| Graph engine | **React Flow (xyflow)** | Mature DAG library. Custom nodes, edges, minimap, zoom, keyboard nav |
| State management | **Zustand** | Lightweight, no boilerplate. Slices per domain |
| Data fetching | **TanStack Query v5** | Cache, dedup, background refresh. Pairs with WebSocket invalidation |
| Forms | **React Hook Form + Zod** | Validation shares schemas with backend |
| i18n | **next-intl** | App Router native. ICU message format. SSR-compatible |
| Icons | **Lucide React** | Consistent, tree-shakeable. Supplement with custom DeepFlow icons |

### Auth
| Layer | Choice | Rationale |
|-------|--------|-----------|
| Provider | **Keycloak** | SSO, RBAC, org-scoped tenancy |
| Client library | **next-auth v5** (Auth.js) with Keycloak provider | Handles token refresh, session management, middleware protection |
| Token flow | Authorization Code + PKCE | Server-side token exchange. Access token in httpOnly cookie. Refresh token server-side only |
| RBAC | Keycloak realm roles → mapped to app permissions | 5 roles: Viewer, Editor, Manager, Admin, Owner (per guidelines §27 Permissions) |
| Multi-tenancy | Keycloak organisations / groups → tenant context | Org switcher in sidebar. All API calls scoped to active tenant |

### Real-time
| Layer | Choice | Rationale |
|-------|--------|-----------|
| Transport | **WebSocket** (native, via backend gateway) | Server-pushed updates: task status changes, AI chat messages, collaboration presence |
| Client | Custom hook `useWebSocket` wrapping native WebSocket | Auto-reconnect, exponential backoff, heartbeat ping |
| Pattern | **Event → TanStack Query invalidation** | WS message arrives → invalidate relevant query key → UI re-fetches fresh data. No dual state |
| Events | `task.updated`, `task.created`, `task.deleted`, `chat.message`, `presence.update`, `workflow.status` | Granular invalidation. Not full-page refresh |
| Fallback | SSE for environments blocking WS | Same event schema, different transport |

### Persistence (Client-side)
| Layer | Choice | Rationale |
|-------|--------|-----------|
| Engine | **IndexedDB** via `idb-keyval` | Lightweight wrapper. No complex queries needed |
| Persisted data | View preferences, filter states, sidebar state, theme, layout choice (A/B), recent projects, draft chat messages | Survives page refresh and browser restart |
| Zustand middleware | `persist` middleware with IndexedDB storage adapter | Automatic hydration. Selective persistence per slice |
| Sync | Local-first, server-authoritative | Preferences sync to user profile API on change. IndexedDB is fast cache. Server is source of truth |
| TTL | Cached query data: 5 min stale, 30 min GC | TanStack Query `gcTime` + `staleTime` |

---

## 3. Architecture

### Project Structure
```
src/
├── app/                          # Next.js App Router
│   ├── (auth)/                   # Auth routes (login, callback)
│   ├── (app)/                    # Authenticated shell
│   │   ├── layout.tsx            # Shell: sidebar + chat + content slot
│   │   ├── page.tsx              # Workspace (dashboard)
│   │   ├── projects/
│   │   │   ├── page.tsx          # Gallery view
│   │   │   └── [projectId]/
│   │   │       ├── page.tsx      # Graph view (default)
│   │   │       ├── list/page.tsx # List view
│   │   │       └── [taskId]/     # Active task view
│   │   └── settings/
│   │       ├── page.tsx
│   │       └── [section]/page.tsx
│   ├── api/                      # API routes (BFF)
│   └── layout.tsx                # Root layout (providers)
├── components/
│   ├── shell/                    # Sidebar, ChatPanel, TopBar, HubBar
│   ├── graph/                    # React Flow custom nodes, edges, zones
│   ├── list/                     # Table, expandable rows
│   ├── workspace/                # Project cards, stats, attention banner
│   ├── gallery/                  # Template cards, project cards
│   ├── detail/                   # Detail panel, tabs, forms
│   ├── chat/                     # Message list, input, action chips
│   └── ui/                       # Radix primitives styled w/ design tokens
├── stores/                       # Zustand stores
│   ├── layout.ts                 # Sidebar, panel visibility, Option A/B
│   ├── graph.ts                  # Selected node, zoom, viewport
│   ├── chat.ts                   # Messages (optimistic), draft input
│   ├── filters.ts                # Active filters, sort, search
│   └── preferences.ts            # Theme, locale, view defaults (persisted)
├── hooks/                        # Custom hooks
│   ├── useWebSocket.ts
│   ├── useAuth.ts
│   ├── useKeyboardNav.ts
│   └── useMediaQuery.ts
├── lib/
│   ├── api/                      # API client, types, endpoints
│   ├── auth/                     # Keycloak config, token helpers
│   ├── i18n/                     # next-intl config, message loaders
│   └── ws/                       # WebSocket client, event types
├── messages/                     # i18n translation files
│   ├── en.json
│   └── ...
├── styles/
│   ├── tokens.css                # CSS custom properties from design guidelines
│   └── globals.css
└── types/                        # Shared TypeScript types
    ├── task.ts
    ├── project.ts
    ├── workflow.ts
    └── user.ts
```

### Zustand Store Design

```typescript
// stores/layout.ts
interface LayoutStore {
  sidebarExpanded: boolean        // 56px vs 240px
  chatPanelVisible: boolean       // always true in Option B
  detailPanelOpen: boolean        // task detail panel
  detailPanelTaskId: string | null
  hubBarExpanded: boolean         // collapsed vs full dashboard
  layoutMode: 'overlay' | 'always-on'  // Option A vs B
  
  toggleSidebar: () => void
  openDetail: (taskId: string) => void
  closeDetail: () => void
  toggleHubBar: () => void
  setLayoutMode: (mode: 'overlay' | 'always-on') => void
}

// stores/filters.ts — persisted to IndexedDB
interface FiltersStore {
  activeFilters: Record<string, FilterValue>
  sortField: string
  sortDirection: 'asc' | 'desc'
  searchQuery: string
  viewPreference: Record<string, 'graph' | 'list' | 'kanban'>  // per project
  
  setFilter: (key: string, value: FilterValue) => void
  clearFilters: () => void
  setSort: (field: string, direction: 'asc' | 'desc') => void
  setSearch: (query: string) => void
  setViewPreference: (projectId: string, view: string) => void
}

// stores/graph.ts
interface GraphStore {
  selectedNodeId: string | null
  viewport: { x: number; y: number; zoom: number }
  contextLevel: 'L1' | 'L2' | 'L3'
  
  selectNode: (id: string | null) => void
  setViewport: (viewport: Viewport) => void
  drillDown: (nodeId: string) => void
  drillUp: () => void
}
```

---

## 4. Internationalisation (i18n)

### Strategy
- **Library:** next-intl (App Router native)
- **Default locale:** `en-GB`
- **Message format:** ICU MessageFormat (plurals, select, dates)
- **File structure:** `messages/{locale}.json` — flat namespace, dot-separated keys
- **Loading:** Server-side for initial render, client-side for dynamic content
- **RTL:** CSS logical properties throughout (`margin-inline-start` not `margin-left`). Tailwind logical utilities plugin
- **Date/Time:** `Intl.DateTimeFormat` — no moment/dayjs. Relative times via `Intl.RelativeTimeFormat`
- **Numbers:** `Intl.NumberFormat` — percentages, counts

### Key Decisions
- All user-facing strings in message files. No hardcoded text in components
- Component-level namespacing: `chat.input.placeholder`, `graph.zoom.label`
- Locale detection: browser preference → user profile setting → org default
- Locale switcher in Settings → Profile

### Example
```json
{
  "workspace.greeting": "Good {timeOfDay}, {name}",
  "workspace.stats.dueToday": "{count, plural, one {# task} other {# tasks}} due today",
  "hub.progress": "{percent}% complete",
  "task.status.inProgress": "In Progress",
  "chat.placeholder.task": "Ask about this task…",
  "chat.placeholder.general": "Ask about your workflow…"
}
```

---

## 5. Keyboard Navigation

### Global Shortcuts
| Key | Action |
|-----|--------|
| `⌘ + K` | Command palette (search anything) |
| `⌘ + /` | Focus chat input |
| `⌘ + B` | Toggle sidebar |
| `⌘ + .` | Toggle detail panel |
| `⌘ + 1/2/3` | Switch view (Graph/Kanban/List) |
| `Escape` | Close active panel/modal/overlay |

### Graph View
| Key | Action |
|-----|--------|
| `Tab` / `Shift+Tab` | Cycle through nodes |
| `Enter` | Open detail for focused node / drill into group |
| `Backspace` | Drill up one level |
| `Arrow keys` | Pan canvas (when canvas focused) |
| `+` / `-` | Zoom in/out |
| `0` | Fit to screen |
| `Space` | Toggle node selection |

### List View
| Key | Action |
|-----|--------|
| `↑` / `↓` | Move between rows |
| `Enter` | Open detail panel for selected row |
| `←` / `→` | Collapse/expand parent row |
| `Space` | Toggle task checkbox |
| `x` | Mark as done |

### Chat Panel
| Key | Action |
|-----|--------|
| `Enter` | Send message |
| `Shift + Enter` | New line |
| `↑` (empty input) | Edit last message |
| `Escape` | Blur chat input |

### Implementation
- **Library:** Custom `useKeyboardNav` hook + Radix `roving-tabindex` for lists
- **Scope:** Shortcuts are context-aware. Graph shortcuts only fire when graph is focused
- **Discovery:** `?` opens keyboard shortcut overlay (like GitHub)
- **Conflict resolution:** Component focus determines which shortcut scope is active

---

## 6. Accessibility (a11y)

### Standards
- **Target:** WCAG 2.2 Level AA
- **Testing:** axe-core in CI, manual screen reader testing (VoiceOver + NVDA)

### Requirements
| Area | Implementation |
|------|---------------|
| **Semantic HTML** | `<nav>`, `<main>`, `<aside>`, `<section>`, `<header>`. No div soup |
| **Landmarks** | Sidebar = `navigation`, Chat = `complementary`, Content = `main`, Detail = `complementary` |
| **Focus management** | Visible focus rings (2px brand blue). Focus trapped in modals. Focus returned on close |
| **Colour contrast** | All text meets 4.5:1 (AA). Large text 3:1. Status colours paired with shapes/text (never colour alone) |
| **Status indicators** | ○ ▶ ❚❚ ❗ ✓ ✕ — shape + colour + label. Colourblind safe by design |
| **Screen reader** | `aria-label` on icon-only buttons. `aria-live` on chat messages and toast notifications. `role="status"` on hub bar updates |
| **Graph a11y** | Nodes are focusable with `role="treeitem"`. Edge relationships described via `aria-describedby`. Alternative list view always available |
| **Reduced motion** | `prefers-reduced-motion` media query. Disable all transitions/animations. Instant state changes |
| **Keyboard** | Everything operable via keyboard (see §5). No mouse-only interactions |
| **Skip links** | "Skip to main content", "Skip to chat" hidden links at top |
| **Headings** | Proper heading hierarchy per view. One `<h1>` per page |
| **Forms** | All inputs have visible labels. Error messages linked via `aria-describedby`. Required fields marked |
| **Avatars** | Human (circle) vs AI (hexagon) distinction also communicated via `aria-label`: "Assigned to DeepFlow Agent (AI)" |

---

## 7. Responsive Behaviour

### Breakpoints
| Breakpoint | Range | Layout |
|------------|-------|--------|
| **Desktop XL** | ≥ 1440px | Full layout: sidebar + chat + canvas + detail |
| **Desktop** | 1200–1439px | Same layout, panels may flex slightly |
| **Tablet** | 768–1199px | Chat becomes toggleable drawer. Sidebar hidden (hamburger). Detail as full-screen overlay |
| **Mobile** | < 768px | Single-panel view. Bottom nav. Chat as full-screen. No graph canvas (List view default) |

### Panel Behaviour by Breakpoint
| Panel | Desktop (≥1200) | Small window / Tablet (768–1199) | Mobile (<768) |
|-------|---------|--------|--------|
| Sidebar | 56px fixed | 56px fixed (mouse present) or hidden (touch-only) | Hidden, bottom tab bar replaces |
| Chat | 320px fixed | 280px fixed (if mouse) or toggleable drawer (touch) | Full-screen tab |
| Top Bar | Full width | Full width | Simplified (no breadcrumbs) |
| Hub Bar | Collapsible | Collapsed-only, expand = overlay | Sticky card header |
| Graph Canvas | Inline | Inline — full editing if mouse detected | **Not shown** — defaults to List |
| Detail Panel | 360px inline | Full-screen overlay | Bottom sheet |
| Task Table | Full width | Horizontal scroll, fewer columns | Card layout |

### Key Decision: Small Viewports ≠ Touch-Only
Many users run laptops with multiple windows at ~1024px. A small viewport with a mouse pointer is NOT the same as a tablet with touch. Detect input method (`pointer: fine` media query) to decide:
- **Mouse present at small viewport:** keep sidebar (56px), keep chat panel (280px), enable graph editing
- **Touch-only at small viewport:** hide sidebar (hamburger), chat as drawer, graph is read-only

### Implementation
- **CSS:** Tailwind responsive prefixes (`md:`, `lg:`, `xl:`) + `@media (pointer: fine)` for input detection
- **Layout detection:** `useMediaQuery` hook → drives Zustand layout store. Separate `pointer` and `viewport` states
- **Conditional rendering:** Graph canvas component not mounted on mobile (not just hidden). Saves memory/CPU
- **Touch:** Graph canvas supports pinch-to-zoom and two-finger pan on touch devices (read-only navigation)
- **Bottom tab bar (mobile):** Home, Tasks, Chat, Notifications, You

---

## 8. Component Architecture

### Shell (always rendered)
```
<ShellLayout>                         ← app/(app)/layout.tsx
  <Sidebar />                         ← 56px, icon nav, org switcher, user
  <ChatPanel />                        ← 320px, contextual, always visible
  <div className="flex flex-col flex-1">
    <TopBar />                         ← breadcrumbs, view switcher, search
    <HubBar />                         ← conditional: graph/active views only
    <main>{children}</main>            ← routed content
  </div>
  <DetailPanel />                      ← conditional: 360px slide-in
</ShellLayout>
```

### Key Components
| Component | Responsibilities |
|-----------|-----------------|
| `Sidebar` | **Top zone:** Org switcher (logo + name, dropdown to switch). **Mid zone:** Dashboard, Tasks & Projects (expandable → pinned/recent projects), Agents & Automations (expandable → active agents with status dots). **Bottom zone:** Notification bell (dot badge collapsed, count expanded), Settings, User profile/avatar. Collapsed: 56px icon-only with tooltips + hover flyout mini-panels. Expanded: 240px (auto-expand in Settings view only). `aria-label="Main navigation"` |
| `ChatPanel` | Message list, input, context badge, action chips. Subscribes to WS for new messages. `aria-label="AI chat"` |
| `TopBar` | Breadcrumbs (driven by route), view switcher (Graph/Kanban/List), filter button, search. Dynamic per view |
| `HubBar` | Collapsed: progress + stats. Expanded: donut, breakdown, due dates, assignees. Animated toggle |
| `DetailPanel` | Tabs: Overview, Sub-tasks, Files, Comments, History. Task data via TanStack Query. Editable fields |
| `GraphCanvas` | React Flow wrapper. Custom node component. Custom edge component. Level zones. Minimap. Zoom controls |
| `TaskTable` | Virtualized table (TanStack Table). Expandable rows. Sticky headers. Sortable columns. Row selection |
| `ProjectCard` | Health meter, progress bar, status counts, avatar stack. Click → navigate to project |

---

## 9. Data Flow

### API Layer
- **BFF pattern:** Next.js API routes proxy to backend. Attach auth tokens server-side
- **REST endpoints:** `/api/projects`, `/api/projects/:id/tasks`, `/api/tasks/:id`, `/api/chat/messages`
- **Response format:** JSON:API-ish with included relationships. Pagination via cursor
- **Error handling:** Typed error responses. Toast for user-facing errors. Retry for transient failures

### WebSocket Events → UI Updates
```
WS event: task.updated { id, status, assignee }
  → invalidateQueries(['tasks', projectId])
  → if task is selected in detail panel: refetch task detail
  → if graph is mounted: React Flow updates node appearance

WS event: chat.message { role, content, taskId }
  → append to chat store (optimistic)
  → scroll to bottom
  → if action chips present: render inline buttons

WS event: presence.update { userId, status, cursor }
  → update presence store
  → render avatar cursor on graph canvas (collaboration)
```

### Optimistic Updates
- Task status change: update UI immediately, revert on error
- Chat message send: append to list immediately, confirm on server ack
- Detail panel edits: save on blur/submit, optimistic update with rollback

---

## 10. Performance Budget

| Metric | Target | Measurement |
|--------|--------|-------------|
| **LCP** | < 2.5s | Largest contentful paint (dashboard cards or graph canvas) |
| **FID** | < 100ms | First input delay |
| **CLS** | < 0.1 | No layout shift from panel rendering |
| **JS bundle (initial)** | < 200KB gzipped | Code-split by route. React Flow lazy-loaded |
| **TTI** | < 3.5s | Time to interactive on 4G |

### Strategies
- **Code splitting:** React Flow + graph components lazy-loaded (only on graph routes)
- **Virtualisation:** Task table uses `@tanstack/react-virtual` for 1000+ row lists
- **Image optimisation:** Next.js `<Image>` for avatars and thumbnails
- **Font loading:** `font-display: swap`. Subset Euclid Circular B to Latin
- **Prefetch:** Prefetch adjacent routes on hover (Next.js `<Link>`)

---

## 11. Testing Strategy

| Layer | Tool | Coverage |
|-------|------|----------|
| **Unit** | Vitest | Zustand stores, utility functions, hooks |
| **Component** | Testing Library + Vitest | All components. Keyboard nav. a11y assertions |
| **Integration** | Playwright | Critical paths: login → dashboard → project → task → chat interaction |
| **Visual regression** | Playwright screenshots | Component snapshots at each breakpoint |
| **a11y** | axe-core (CI) + manual | Every component scanned. Screen reader testing quarterly |
| **Performance** | Lighthouse CI | Budget enforcement on every PR |

---

## 12. Design Tokens → Code

The [design guidelines](index.html) define tokens. Map them to Tailwind + CSS vars:

```css
/* styles/tokens.css — generated from guidelines */
:root {
  /* Brand */
  --brand-cyan: #00E5FF;
  --brand-blue: #2244FF;
  --brand-mid: #0099FF;
  --brand-gradient: linear-gradient(135deg, #00E5FF 0%, #0099FF 45%, #2244FF 100%);
  
  /* Status */
  --status-complete: #059669;
  --status-active: #2563EB;
  --status-blocked: #D97706;
  --status-failed: #DC2626;
  --status-pending: #9CA3AF;
  --status-approval: #DC2626;
  
  /* Spacing (4px base) */
  --space-1: 4px;
  --space-2: 8px;
  --space-3: 12px;
  --space-4: 16px;
  --space-6: 24px;
  --space-8: 32px;
  
  /* Radii */
  --radius-sm: 4px;
  --radius-md: 8px;
  --radius-lg: 12px;
  --radius-xl: 16px;
  --radius-full: 9999px;
  
  /* Elevation */
  --shadow-sm: 0 1px 3px rgba(0,0,0,0.06);
  --shadow-md: 0 4px 12px rgba(0,0,0,0.08);
  --shadow-lg: 0 8px 32px rgba(0,0,0,0.12);
  
  /* Typography */
  --font-family: 'Euclid Circular B', -apple-system, sans-serif;
  --text-xs: 0.6875rem;   /* 11px */
  --text-sm: 0.8125rem;   /* 13px */
  --text-base: 0.875rem;  /* 14px */
  --text-lg: 1.125rem;    /* 18px */
  --text-xl: 1.5rem;      /* 24px */
  --text-2xl: 1.875rem;   /* 30px */
}
```

---

## 13. Decisions (Resolved)

| # | Question | Decision |
|---|----------|----------|
| 1 | **Kanban view** | Defer to Phase 2. Keep placeholder in view switcher (greyed out). Also add **Map view** as a future placeholder |
| 2 | **Collaboration cursors** | Phase 3. Build presence indicators first (avatars in top bar). Cursors are higher-frequency WS events on the same infrastructure — add as polish. Throttle to 10fps, canvas-space coordinates, fade after 3s idle |
| 3 | **Offline support** | Online-only. Chat requires connection. IndexedDB is local cache for preferences only |
| 4 | **Chat history** | Stored in backend DB. Frontend fetches via API with pagination. Context scoped per-task in UI but full history queryable |
| 5 | **Mobile app** | Native app needed long-term. Phase 1: responsive web. Phase 2: evaluate React Native / Expo. Layout must support native from the start |
| 6 | **Tablet / small viewport** | Full mouse support at tablet sizes. Many users run laptop with multiple windows at ~1024px. Not touch-only — graph editing enabled if mouse is present. Responsive breakpoints accommodate small-windowed desktop, not just tablets |
| 7 | **Notifications** | Multi-channel: in-app bell + email + Slack. Configurable per notification type per user in Settings. With native mobile app, also push notifications. Settings UI: matrix of notification types × channels |
| 8 | **Backend API spec** | Assume it exists. A separate agent will extract API contracts from the repo. Frontend defines TypeScript interfaces; backend is source of truth |

---

## 14. Phasing

### Phase 1 — Shell + Core Views (4–6 weeks)
- [ ] Auth (Keycloak + next-auth)
- [ ] Shell layout (sidebar, chat panel, top bar)
- [ ] Workspace (dashboard)
- [ ] Gallery (project browser)
- [ ] Settings (layout toggle, preferences)
- [ ] i18n scaffolding (en-GB)
- [ ] Responsive shell (all breakpoints)
- [ ] Keyboard shortcuts (global)
- [ ] Design tokens integrated

### Phase 2 — Graph + List Views (4–6 weeks)
- [ ] React Flow integration (custom nodes, edges, zones)
- [ ] Graph navigation (drill-down, breadcrumbs, hub bar)
- [ ] List view (table, expandable rows, sort/filter)
- [ ] Detail panel (all tabs)
- [ ] WebSocket integration (live updates)
- [ ] Active task view
- [ ] Graph keyboard navigation

### Phase 3 — Chat + Collaboration + Polish (4–5 weeks)
- [ ] Chat panel (message rendering, action chips, context switching)
- [ ] Optimistic updates
- [ ] Collaboration presence (user avatars in top bar showing who's viewing)
- [ ] Collaboration cursors (canvas-space coordinates, 10fps throttle, 3s fade)
- [ ] Notification settings UI (matrix: notification types × channels: in-app/email/Slack/push)
- [ ] Accessibility audit + fixes
- [ ] Performance optimisation (code splitting, virtualisation)
- [ ] Visual regression tests
- [ ] Additional locales (if needed)
- [ ] Kanban view placeholder → implementation (if prioritised)
- [ ] Map view placeholder (Phase 4+)
