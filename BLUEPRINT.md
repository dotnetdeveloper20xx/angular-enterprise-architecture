# Nexus Studio: Top-Level Blueprint

> **One-Page Executive Summary of the Entire Project**

---

## 🎯 What We're Building

**Nexus Studio** - An enterprise-grade, all-in-one operations platform for digital product teams, built with Angular 19+ and modern patterns (Signals, standalone components, NgRx SignalStore).

**The Pitch**: Think VS Code meets Notion meets Jira—a unified workspace where teams manage files, design systems, media, projects, logs, documentation, cases, datasets, and forms without context-switching across 10+ tools.

---

## 🏗️ Architecture at a Glance

### Technology Stack
```
┌─────────────────────────────────────────────────────┐
│ Angular 19+ (Signals, Standalone Components)       │
├─────────────────────────────────────────────────────┤
│ State: NgRx SignalStore + RxJS                     │
│ UI: Angular Material + CDK (Drag-Drop, Virtual Scroll) │
│ Persistence: IndexedDB + localStorage               │
│ Workers: Web Workers for heavy computation          │
│ Async: RxJS + Signals interop                      │
│ Testing: Jest + Cypress + Accessibility Tooling    │
└─────────────────────────────────────────────────────┘
```

### Core Patterns
- **Signals-First**: `signal()`, `computed()`, `effect()` for reactive state
- **Functional Providers**: Route-level DI with `provideFoo()` functions
- **Standalone Everything**: No NgModules, tree-shakable by default
- **Lazy Loading**: Each feature module code-splits automatically
- **Signal-Based Inputs**: `input<T>()` for component inputs where applicable

---

## 🧩 The 10 Integrated Modules

| # | Module | Primary Use Case | Hard UI Feature Examples |
|---|--------|------------------|--------------------------|
| 1 | **File Manager + Previewer** | Central file repository | Virtual tree (50k nodes), drag-drop upload, preview pane, multi-select |
| 2 | **Design System Playground** | Living style guide | Live token editing, theme switcher, component gallery, a11y checks |
| 3 | **Media Library** | Digital asset management | Masonry grid, infinite scroll, metadata editor, collections |
| 4 | **Kanban + Timeline** | Project management | Drag-drop cards, timeline scrubbing, split view, real-time sync |
| 5 | **Log Viewer** | Observability console | Virtual scroll (100k logs), log streaming, correlation tracing |
| 6 | **Knowledge Base** | Internal wiki | Markdown editor, version history, table of contents, full-text search |
| 7 | **Admin Console** | Multi-tenant management | User/role matrix, audit log viewer, permission editor |
| 8 | **Case Triage** | Support ticket system | Email-style inbox, case timeline, SLA indicators, assignment |
| 9 | **Dataset Explorer** | CSV/JSON data analysis | Virtual grid (100k rows), column profiling, query builder, export |
| 10 | **Form Builder** | Dynamic form designer | Drag-drop schema builder, live preview, conditional logic, validation |

---

## 🔗 Unified Product Narrative

**They're Not Separate Apps—They're One Workspace**

1. **Shared Entities**: Files, Users, Tenants, Roles flow across all modules
2. **Cross-Module Links**: Kanban card → links to dataset → links to case → links to log entry
3. **Unified Search**: One search box finds content across all 10 modules
4. **Single Shell**: IDE-like workspace with split panes, tabs, dockable panels
5. **Consistent State**: All modules use same state patterns (Signals + SignalStore)
6. **Global Actions**: Command palette (Cmd+K) works everywhere
7. **Shared Workflows**: Upload dataset → profile in Explorer → document in KB → link in Case

**Example User Journey**:
> Customer Success Engineer receives case (#2491) in **Case Triage** → sees linked error in **Log Viewer** → finds related dataset in **Dataset Explorer** → creates **Kanban** card for engineering → documents root cause in **Knowledge Base** → all in one workspace, one URL.

---

## 🎨 Workspace Shell (The "IDE" Experience)

### Layout System
```
┌─────────────────────────────────────────────────────────┐
│ [Logo] [Tenant▾] [Search] [Cmd+K] [🔔] [Avatar▾]      │ ← Top Bar
├──────┬──────────────────────────────────────────────────┤
│ Nav  │  ┌──────────────────────────────────────────┐   │
│      │  │ Tabs: Dataset | Kanban | Case #2491 | + │   │
│ 🏠   │  ├──────────────────────────────────────────┤   │
│ 📁   │  │                                          │   │
│ 🎨   │  │  Main Content Pane                       │   │
│ 🖼️   │  │  (current tab's module renders here)    │   │
│ 📋   │  │                                          │   │
│ 📊   │  ├──────────────────────────────────────────┤   │
│ 📚   │  │  Inspector Panel (collapsible)           │   │
│ 🎫   │  └──────────────────────────────────────────┘   │
│ 📈   │                                                  │
│ 📝   │                                                  │
│ ⚙️   │                                                  │
├──────┴──────────────────────────────────────────────────┤
│ Status: 3 notifications | Sync: ✓ | User: Admin         │ ← Status Bar
└─────────────────────────────────────────────────────────┘
```

### Key Shell Features
- **Resizable Split Panes**: Drag borders to resize (Angular CDK)
- **Dockable Panels**: Drag panels to different dock zones (top/bottom/left/right)
- **Tabbed Workspace**: Multi-tab support with pinned tabs, dirty indicators (*)
- **Session Restore**: Restore tabs, scroll positions, selections on reload
- **Command Palette**: Cmd+K for fuzzy command search (à la VS Code)
- **Keyboard Shortcuts**: Full keyboard navigation, focus management
- **Context Menus**: Right-click anywhere for contextual actions

---

## 📊 State Management Strategy

### Three-Tier State Architecture

```typescript
┌─────────────────────────────────────────────────────┐
│ 1. SHELL STATE (Global)                            │
│    - Current tenant, user, permissions              │
│    - Workspace tabs, layout, active tab             │
│    - Global search index, notifications             │
│    → NgRx SignalStore (AppShellStore)               │
├─────────────────────────────────────────────────────┤
│ 2. FEATURE STATE (Module-Scoped)                   │
│    - File tree, Kanban boards, Dataset rows         │
│    - Provided at route level, scoped to feature     │
│    → NgRx SignalStore (FileStore, KanbanStore, etc.)│
├─────────────────────────────────────────────────────┤
│ 3. COMPONENT STATE (Local)                         │
│    - Form inputs, filter state, UI toggles          │
│    - Purely local, no need for store                │
│    → signal(), computed(), effect()                 │
└─────────────────────────────────────────────────────┘
```

### When to Use What

| State Type | Tool | Example |
|------------|------|---------|
| Global app state | SignalStore (app-level) | User, Tenant, Permissions, Tabs |
| Feature domain state | SignalStore (route-level) | File tree, Kanban boards, Cases |
| Async workflows | RxJS + rxMethod() | API calls, pagination, search |
| Local UI state | signal() / computed() | Modal open/closed, filter inputs |
| Derived state | computed() | Filtered lists, aggregate counts |
| Side effects | effect() | Persist to localStorage, log events |

### State Flow Pattern
```
User Action (click/keyboard)
  → Dispatch Store Method
    → Update State (signal)
      → Computed Signals Auto-Update
        → Template Re-Renders (Signal-Based)
          → effect() for Side Effects (if needed)
```

**No Zone.js Required**: Signals enable zoneless change detection (future-proof)

---

## 🗃️ Data Layer (Mock Backend Strategy)

### Mock API Architecture
```typescript
// services/mock-api/
├── mock-api.service.ts          // Orchestrates all mocks
├── entities/
│   ├── tenant.mock.ts
│   ├── user.mock.ts
│   ├── file.mock.ts
│   ├── case.mock.ts
│   ├── dataset.mock.ts
│   └── ...
└── db/
    └── indexed-db.service.ts    // IndexedDB wrapper
```

### What's Mocked
- **REST-like API**: `get()`, `post()`, `put()`, `delete()` methods
- **Pagination**: Cursor-based and offset-based
- **Filtering/Sorting**: Client-side query engine
- **Search**: Full-text search with ranking (in Web Worker)
- **Latency Simulation**: Configurable delays (e.g., 200ms) for realism
- **Error Simulation**: Random failures, rate limits, validation errors

### Persistence Strategy
**IndexedDB** for:
- User-created content (files, forms, KB docs)
- User preferences (theme, layout, recent searches)
- Cached data (datasets, log history)

**localStorage** for:
- Workspace session (tabs, scroll, selections)
- Auth token (JWT simulation)
- Last selected tenant

**Web Workers** for:
- Dataset profiling (calculate stats for 100k rows)
- Search indexing (build inverted index)
- File transformations (CSV → JSON, image resize)
- Log parsing (parse 100k log lines)

---

## 🚀 Performance Engineering

### Virtualization
- **Lists**: CDK Virtual Scroll for 10k+ items (logs, datasets, file lists)
- **Grids**: Custom virtual grid for Dataset Explorer (100k rows × 50 cols)
- **Trees**: Virtual tree scrolling for File Manager (50k nodes)

### Change Detection
- **OnPush Everywhere**: All components use `ChangeDetectionStrategy.OnPush`
- **Signal-Driven**: Templates use signals → automatic minimal re-renders
- **TrackBy Functions**: For all `@for` loops to prevent unnecessary DOM churn
- **Memoization**: Heavy computations in `computed()` auto-memoize

### Lazy Loading
- **Route-Level**: Each module lazy-loads when navigated to
- **Component-Level**: Heavy components (`defer` blocks for future Angular versions)
- **Data**: Paginate everything, load on demand

### Rendering Optimization
- **Debouncing**: Search, resize events debounced (200-500ms)
- **Throttling**: Scroll events throttled (16ms = 60fps)
- **Web Workers**: Heavy work off main thread (no UI freezing)

---

## ♿ Accessibility (A11y)

### ARIA Patterns Used
- **Tree View**: File Manager, Knowledge Base folder tree (`role="tree"`)
- **Tabs**: Workspace tabs (`role="tablist"`, `role="tab"`, `role="tabpanel"`)
- **Grid**: Dataset Explorer (`role="grid"`, `role="gridcell"`)
- **Combobox**: Search, Command Palette (`role="combobox"`, `aria-expanded`)
- **Dialog**: Modals (`role="dialog"`, focus trapping, Esc to close)
- **Menu**: Context menus (`role="menu"`, `role="menuitem"`)
- **Live Regions**: Notifications, status updates (`aria-live="polite"`)

### Keyboard Navigation
- **Global Shortcuts**: Cmd+K (command palette), Cmd+Shift+F (search), Cmd+/ (help)
- **Tab Management**: Cmd+W (close), Cmd+T (new), Cmd+1-9 (switch tabs)
- **Focus Management**: Focus returns to trigger after modal close, skip-to-main link
- **Escape Behavior**: Closes modals, clears search, exits fullscreen
- **Arrow Navigation**: Lists, grids, trees, menus

### Visual Accessibility
- **High Contrast Mode**: Separate theme with WCAG AAA compliance
- **Focus Indicators**: Visible focus rings (3px solid)
- **Color Blindness**: No info conveyed by color alone (icons + text)
- **Text Scaling**: UI works at 200% zoom

---

## 🧪 Testing Strategy

### Unit Tests (Jest)
- **State Stores**: Test state transitions, computed signals, effects
- **Services**: Mock API, IndexedDB, Web Worker communication
- **Utilities**: Search ranking, parsers, validators
- **Coverage Target**: >80%

### Component Tests (Angular Testing Library)
- **User Interactions**: Click, keyboard, drag-drop
- **Accessibility**: Query by role, check ARIA attributes
- **State Integration**: Provide mock stores, test state updates
- **Coverage Target**: All user-facing components

### E2E Tests (Cypress)
- **Critical Paths**: Login → upload dataset → create case → link to kanban card
- **Cross-Module**: Verify links between modules work (case → logs)
- **Session Restore**: Close/reopen browser, verify state restored
- **Accessibility**: Cypress-axe for automated a11y checks

---

## 📅 Build Roadmap

### MVP (Phase 1: 2-3 Weeks)
**Goal**: Prove the workspace shell + 2 flagship modules

**Includes**:
- ✅ Workspace Shell (split panes, tabs, session restore)
- ✅ File Manager (tree view, upload, preview)
- ✅ Dataset Explorer (grid view, sorting, filtering, virtualization)
- ✅ Global Search (basic, single module)
- ✅ Mock API + IndexedDB
- ✅ Authentication (simulated)

**Acceptance**: Can open app, switch tabs, upload CSV, view 10k rows, restore session

---

### v1 (Phase 2: 4-6 Weeks)
**Goal**: All 10 modules with consistent patterns

**Adds**:
- Design System Playground
- Media Library
- Kanban + Timeline
- Log Viewer
- Knowledge Base
- Admin Console
- Case Triage
- Form Builder

**Acceptance**: All modules navigable, cross-module links work, global search works everywhere

---

### v2 (Phase 3: 2-3 Weeks Polish)
**Goal**: Production-ready polish

**Adds**:
- Offline mode (full IndexedDB persistence)
- Import/Export (backup workspace data)
- Plugin architecture (custom modules)
- Audit completeness (full audit trail)
- Performance profiling (Lighthouse >90)
- A11y audit (WCAG AAA compliance)
- E2E test coverage (all critical paths)

**Acceptance**: App feels production-ready, no mock limitations visible

---

## 📚 Developer Documentation Pack

### 9 Markdown Files (Mentoring-Focused)

1. **README.md** - Project overview, setup, commands
2. **ARCHITECTURE.md** - Module boundaries, routing, data layer
3. **STATE_MANAGEMENT.md** - Signals, stores, effects, patterns
4. **UI_SHELL.md** - Split panes, docking, tabs, keyboard
5. **DATA_LAYER.md** - Mock API, IndexedDB, workers, pagination
6. **PERFORMANCE.md** - Virtualization, rendering, profiling
7. **ACCESSIBILITY.md** - ARIA patterns, keyboard contracts
8. **TESTING.md** - Unit/component/e2e strategy + examples
9. **CONTRIBUTING.md** - Standards, naming, PR checklist

**Format**: What/Why/How structure, code examples, rationale for junior devs

---

## 🎯 Success Metrics

### Technical Excellence
- ✅ Built with Angular 19+ and modern patterns (Signals, standalone)
- ✅ Handles 100k+ rows without performance degradation
- ✅ Lighthouse Performance score >90
- ✅ WCAG AAA accessibility compliance
- ✅ <3s initial load time (lazy loading optimized)

### Product Coherence
- ✅ Feels like ONE product (not 10 mini-apps)
- ✅ Shared entities and workflows across modules
- ✅ Consistent UI patterns and state management
- ✅ Deep-linking and cross-module navigation works

### Portfolio Impact
- ✅ Demonstrates senior/architect-level skills
- ✅ Showcases "hard UI" patterns (virtualization, docking, workers)
- ✅ Exceeds reference apps (Kendo Personal Finance) in complexity
- ✅ Mentoring material helps others learn advanced patterns

---

## 🏁 What's Next

**Phase 1 Complete ✅**: Product vision, architecture, routing defined

**Phase 2 Ahead**: Workspace Shell design + State Management patterns

**Phase 3 Ahead**: Shared systems (File model, Search, Multi-tenancy)

**Phases 4-6**: Detailed module designs (all 10 modules)

**Phase 7**: Performance + Testing strategy

**Phase 8**: Build roadmap + Final documentation

---

## 💡 Key Takeaways

1. **Unified Product**: Not a collection of demos—a real internal tool suite
2. **Modern Angular**: Leverages latest patterns (Signals, standalone, functional providers)
3. **Hard UI**: Tackles complex patterns most devs avoid (virtualization, docking, workers)
4. **Scalable Architecture**: Multi-tenant, RBAC, offline-first, designed for growth
5. **Mentoring Value**: Documentation teaches "why" not just "how"

---

**Status**: Blueprint Complete ✅
**Next Document**: README.md (Phase 1 Final Deliverable)
