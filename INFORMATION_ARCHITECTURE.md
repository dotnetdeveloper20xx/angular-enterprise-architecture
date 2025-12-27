# Information Architecture & Routing

## 🏗️ Application Shell Structure

### Layout Overview
```
┌─────────────────────────────────────────────────────────────────┐
│  Top Bar (56px)                                                 │
│  [Logo] [Tenant▾] [Search...] [Cmd+K] [🔔] [?] [Avatar▾]      │
├──────┬──────────────────────────────────────────────────────────┤
│      │  Workspace Area (dynamic split panes & tabs)            │
│      │  ┌──────────────────────────────────────────────────┐   │
│  L   │  │  Tab: Dataset Explorer | Kanban Board | Case... │   │
│  e   │  ├──────────────────────────────────────────────────┤   │
│  f   │  │                                                  │   │
│  t   │  │        Main Content Pane                        │   │
│      │  │                                                  │   │
│  N   │  │                                                  │   │
│  a   │  │                                                  │   │
│  v   │  ├──────────────────────────────────────────────────┤   │
│      │  │  Inspector / Properties Panel (collapsible)     │   │
│ 240px│  └──────────────────────────────────────────────────┘   │
│      │                                                          │
├──────┴──────────────────────────────────────────────────────────┤
│  Status Bar (24px)                                              │
│  [Activity] [Notifications: 3] [Sync: ✓] [User: Admin]         │
└─────────────────────────────────────────────────────────────────┘
```

---

## 📍 Top Bar Components

### 1. Branding & Tenant Switcher
```html
[Nexus Studio Icon] [Current Tenant ▾]
```

**Tenant Switcher Dropdown**:
- Lists all tenants user has access to
- Shows tenant avatar + name
- Quick switch (preserves current route within new tenant context)
- "Manage Tenants" link (for admins)

**Behavior**:
- Switching tenant reloads app with tenant ID in URL: `/t/{tenantId}/...`
- Last selected tenant persisted in localStorage
- Tenant context affects all data queries and permissions

---

### 2. Global Search (Cmd+Shift+F)
```html
[🔍 Search everywhere... Cmd+Shift+F]
```

**Search Scope**: All modules
- Files, media assets, documents
- Cases, tickets
- Kanban cards, timeline events
- Log entries (by message/correlation ID)
- Datasets (by name/column)
- Forms (by name/field)
- Users, teams

**Search UI**:
- Overlay panel (not full page)
- Results grouped by module with icons
- Fuzzy matching + highlights
- Keyboard navigation (↑↓ to select, Enter to open)
- Shows preview snippet for each result

**Deep Link**: Opens result in appropriate module with context highlighted

---

### 3. Command Palette (Cmd+K)
```html
[⌘K Command Palette]
```

**Command Categories**:
1. **Navigation**: "Go to Dashboard", "Open File Manager", "Switch to Kanban"
2. **Actions**: "Create Case", "New Form", "Upload Dataset", "Start Log Stream"
3. **Search**: "Search logs for...", "Find case #...", "Open file..."
4. **Settings**: "Change Theme", "Keyboard Shortcuts", "Preferences"
5. **Context-Aware**: If in Dataset Explorer → "Export CSV", "Profile Columns"

**Features**:
- Natural language parsing: "case 2491" → opens case directly
- Recent commands history
- Fuzzy matching on command names
- Aliases: "kb" for knowledge base, "ds" for design system

---

### 4. Notifications (Bell Icon)
```html
[🔔 3]
```

**Notification Types**:
- System: "Sync completed", "Background job finished"
- Case assignments: "You were assigned to Case #2491"
- Mentions: "@you in comment on Kanban card"
- Activity: "Dataset upload complete", "Form published"

**Notification Panel**:
- Dropdown (max 10 recent, then "View All")
- Mark as read/unread
- Click to navigate to source
- Filters: All, Unread, Mentions

---

### 5. Help & Quick Actions
```html
[? Help] [Avatar ▾]
```

**Help Menu**:
- Keyboard Shortcuts (opens modal with searchable list)
- Documentation (opens Knowledge Base)
- What's New (changelog)
- Report Issue

**User Menu (Avatar Dropdown)**:
- Profile
- Settings
- Theme Selector (Light/Dark/Auto)
- Sign Out

---

## 🧭 Left Navigation (Sidebar)

### Structure
```
┌─────────────────────┐
│ ☰ NEXUS STUDIO     │  ← Collapse/Expand
├─────────────────────┤
│ 🏠 Dashboard        │
│ 📁 Files            │
│ 🎨 Design System    │
│ 🖼️  Media Library   │
│ 📋 Kanban           │
│ 📊 Logs             │
│ 📚 Knowledge Base   │
│ 🎫 Case Triage      │
│ 📈 Dataset Explorer │
│ 📝 Form Builder     │
├─────────────────────┤
│ ⚙️  Admin Console   │  ← Only if user has admin role
└─────────────────────┘
```

### Navigation Behavior

**Active States**:
- Bold + highlight for current module
- Icon + label (labels hide when collapsed to icon-only)

**Collapsible**:
- Click hamburger to toggle collapsed/expanded
- Collapsed: 64px (icons only), Expanded: 240px
- Preference saved per user

**Keyboard Navigation**:
- Alt+1 through Alt+0 for quick module access
- Tab to focus sidebar, arrow keys to navigate

**Nested Items** (for future expansion):
- Files can expand to show recent/favorites
- Knowledge Base can show folder tree
- Admin can expand to Users/Roles/Audit

---

## 🗺️ Route Map & URL Structure

### Base Pattern
```
/t/{tenantId}/{module}/{...moduleRoutes}?{queryParams}#{fragmentForState}
```

### Route Definitions

#### Root & Dashboard
```
/                                  → Redirects to /t/{lastTenantId}/dashboard
/t/{tenantId}/dashboard            → Dashboard overview
```

#### 1. File Manager
```
/t/{tenantId}/files                               → Root folder
/t/{tenantId}/files/folder/{folderId}             → Specific folder
/t/{tenantId}/files/file/{fileId}                 → File preview
/t/{tenantId}/files/file/{fileId}/edit            → Edit mode
```

**Query Params**:
- `?view=list|grid|tree`
- `?sort=name|date|size&order=asc|desc`
- `?filter=images|documents|videos`

**Fragment State**:
- `#selected=file1,file2,file3` (multi-select state)
- `#scroll=500` (scroll position)

---

#### 2. Design System Playground
```
/t/{tenantId}/design-system                       → Overview/Gallery
/t/{tenantId}/design-system/tokens                → Token editor
/t/{tenantId}/design-system/tokens/{category}     → Specific token category (colors, spacing, typography)
/t/{tenantId}/design-system/components            → Component gallery
/t/{tenantId}/design-system/components/{name}     → Component detail/playground
/t/{tenantId}/design-system/theme                 → Theme editor
```

**Query Params**:
- `?theme=light|dark|high-contrast`
- `?component={name}` (direct link to component)

---

#### 3. Media Library
```
/t/{tenantId}/media                               → Gallery view
/t/{tenantId}/media/asset/{assetId}               → Asset detail
/t/{tenantId}/media/collections                   → Curated collections
/t/{tenantId}/media/collections/{collectionId}    → Collection detail
```

**Query Params**:
- `?type=image|video|audio|document`
- `?tags=tag1,tag2`
- `?view=grid|masonry|list`

---

#### 4. Kanban + Timeline
```
/t/{tenantId}/kanban                              → Board list
/t/{tenantId}/kanban/board/{boardId}              → Board view
/t/{tenantId}/kanban/board/{boardId}/card/{cardId} → Card detail modal
/t/{tenantId}/kanban/timeline                     → Timeline view
/t/{tenantId}/kanban/timeline/{boardId}           → Board timeline
```

**Query Params**:
- `?view=board|timeline|both`
- `?filter=assignee:{userId}|status:{status}`
- `?groupBy=status|assignee|priority`

**Fragment State**:
- `#card={cardId}` (open card modal)
- `#timeRange=2024-01-01,2024-01-31`

---

#### 5. Log Viewer / Observability Console
```
/t/{tenantId}/logs                                → Live log stream
/t/{tenantId}/logs/search                         → Search logs
/t/{tenantId}/logs/trace/{traceId}                → Trace detail
```

**Query Params**:
- `?level=error|warn|info|debug`
- `?source={service}`
- `?timeRange={start},{end}` (ISO timestamps)
- `?query={searchText}`
- `?correlationId={id}`

**Fragment State**:
- `#line={lineNumber}` (scroll to specific log line)
- `#highlight={searchTerm}`

---

#### 6. Knowledge Base
```
/t/{tenantId}/kb                                  → Home/Index
/t/{tenantId}/kb/doc/{docId}                      → View document
/t/{tenantId}/kb/doc/{docId}/edit                 → Edit document
/t/{tenantId}/kb/folder/{folderId}                → Folder view
/t/{tenantId}/kb/search                           → Search results
```

**Query Params**:
- `?q={searchQuery}`
- `?tag={tag}`
- `?author={userId}`

**Fragment State**:
- `#heading={slugified-heading}` (scroll to heading)
- `#version={versionId}` (view specific version)

---

#### 7. Admin Console
```
/t/{tenantId}/admin                               → Admin dashboard
/t/{tenantId}/admin/users                         → User management
/t/{tenantId}/admin/users/{userId}                → User detail
/t/{tenantId}/admin/roles                         → Role management
/t/{tenantId}/admin/roles/{roleId}                → Role detail
/t/{tenantId}/admin/tenants                       → Tenant management (super admin only)
/t/{tenantId}/admin/audit                         → Audit log
/t/{tenantId}/admin/settings                      → System settings
```

**Query Params**:
- `?search={query}`
- `?role={roleId}` (filter users by role)
- `?active=true|false`

---

#### 8. Email/Case Triage Console
```
/t/{tenantId}/cases                               → Inbox/List view
/t/{tenantId}/cases/{caseId}                      → Case detail
/t/{tenantId}/cases/{caseId}/timeline             → Case timeline/history
/t/{tenantId}/cases/new                           → Create case
```

**Query Params**:
- `?status=open|pending|closed`
- `?priority=high|medium|low`
- `?assignee={userId}|unassigned`
- `?tag={tag}`

**Fragment State**:
- `#reply` (open reply box)
- `#comment={commentId}` (scroll to comment)

---

#### 9. Dataset Explorer
```
/t/{tenantId}/datasets                            → Dataset list
/t/{tenantId}/datasets/{datasetId}                → Dataset viewer
/t/{tenantId}/datasets/{datasetId}/profile        → Data profiling
/t/{tenantId}/datasets/{datasetId}/query          → Query builder
```

**Query Params**:
- `?page={pageNum}&limit={pageSize}` (pagination)
- `?sort={column}&order=asc|desc`
- `?filter={columnFilters}` (JSON-encoded)

**Fragment State**:
- `#row={rowIndex}` (scroll to row)
- `#column={columnName}` (highlight column)

---

#### 10. Form Builder Studio
```
/t/{tenantId}/forms                               → Form list
/t/{tenantId}/forms/new                           → Create new form
/t/{tenantId}/forms/{formId}                      → Form builder
/t/{tenantId}/forms/{formId}/preview              → Live preview
/t/{tenantId}/forms/{formId}/responses            → Form submissions
```

**Query Params**:
- `?mode=design|preview|code` (builder mode)

**Fragment State**:
- `#field={fieldId}` (select field in builder)

---

## 🔗 Deep-Linking Rules

### Principle
**Every meaningful UI state should be URL-encodable and shareable.**

### Examples of Deep Links

1. **Specific log error at timestamp**
   ```
   /t/acme/logs?level=error&timeRange=2024-01-15T10:00:00,2024-01-15T10:05:00#line=4231
   ```

2. **Kanban card in context of board**
   ```
   /t/acme/kanban/board/sprint-24#card=TASK-491
   ```
   Opens board with card modal overlayed

3. **Dataset filtered and sorted**
   ```
   /t/acme/datasets/customers?filter={"country":"US"}&sort=revenue&order=desc#row=50
   ```

4. **Knowledge Base doc at specific heading**
   ```
   /t/acme/kb/doc/onboarding-guide#heading=step-3-configure-permissions
   ```

5. **Case with reply box open**
   ```
   /t/acme/cases/2491#reply
   ```

6. **Design System component with theme**
   ```
   /t/acme/design-system/components/button?theme=dark
   ```

### Workspace State in URL

**Challenge**: How to encode split panes, tabs, panel visibility in URL?

**Solution**: Use fragment-based state encoding
```
#workspace={layoutConfig}
```

**Example**:
```
/t/acme/dashboard#workspace={"layout":"split-h","left":{"type":"files","path":"/docs"},"right":{"type":"dataset","id":"customers"}}
```

**Practical Approach**:
- Don't encode workspace state by default (too verbose)
- Provide "Share Workspace" button → generates URL with encoded state
- Restore from URL if `#workspace=...` present
- Otherwise, restore from localStorage session state

---

## 📦 Lazy Loading & Code Splitting

### Feature Module Strategy

Each major module is a lazy-loaded feature:

```typescript
// app.routes.ts
export const routes: Routes = [
  {
    path: 't/:tenantId',
    component: WorkspaceShellComponent,
    providers: [
      provideTenantContext(),
      provideWorkspaceState(),
    ],
    children: [
      {
        path: 'dashboard',
        loadComponent: () => import('./features/dashboard/dashboard.component'),
      },
      {
        path: 'files',
        loadChildren: () => import('./features/files/files.routes'),
        providers: [provideFileManagerState()],
      },
      {
        path: 'design-system',
        loadChildren: () => import('./features/design-system/design-system.routes'),
        providers: [provideDesignSystemState()],
      },
      {
        path: 'media',
        loadChildren: () => import('./features/media/media.routes'),
        providers: [provideMediaLibraryState()],
      },
      {
        path: 'kanban',
        loadChildren: () => import('./features/kanban/kanban.routes'),
        providers: [provideKanbanState()],
      },
      {
        path: 'logs',
        loadChildren: () => import('./features/logs/logs.routes'),
        providers: [provideLogsState()],
      },
      {
        path: 'kb',
        loadChildren: () => import('./features/knowledge-base/kb.routes'),
        providers: [provideKnowledgeBaseState()],
      },
      {
        path: 'cases',
        loadChildren: () => import('./features/cases/cases.routes'),
        providers: [provideCasesState()],
      },
      {
        path: 'datasets',
        loadChildren: () => import('./features/datasets/datasets.routes'),
        providers: [provideDatasetsState()],
      },
      {
        path: 'forms',
        loadChildren: () => import('./features/forms/forms.routes'),
        providers: [provideFormsState()],
      },
      {
        path: 'admin',
        loadChildren: () => import('./features/admin/admin.routes'),
        providers: [provideAdminState()],
        canActivate: [adminRoleGuard],
      },
    ],
  },
  {
    path: '',
    redirectTo: '/t/default/dashboard',
    pathMatch: 'full',
  },
];
```

### Route-Level Providers

**Why**: Each feature module gets its own state provider injected at route level

**Benefits**:
- State is scoped to feature (destroyed when navigating away, unless persisted)
- Clean dependency injection boundaries
- Easier to reason about state lifecycle
- Tree-shakable: unused features don't include their state logic

**Example**:
```typescript
// features/files/files-state.provider.ts
export function provideFileManagerState() {
  return [
    FileManagerStore, // NgRx SignalStore
    FileTreeService,
    FilePreviewService,
  ];
}
```

---

## 🗂️ Tabbed Workspace & Session Restore

### Tab Model

Each tab has:
```typescript
interface WorkspaceTab {
  id: string;                    // Unique tab ID
  type: ModuleType;              // 'files' | 'kanban' | 'dataset' | ...
  title: string;                 // Display title
  icon: string;                  // Material icon name
  route: string;                 // Full route path
  state?: any;                   // Module-specific state (scroll, selection, etc.)
  isPinned: boolean;             // Pinned tabs stay across sessions
  isDirty: boolean;              // Unsaved changes indicator (*)
  timestamp: number;             // Last accessed (for MRU ordering)
}
```

### Tab Operations

**Actions**:
- Open new tab: Cmd+T or "+" button
- Close tab: Click X or Cmd+W
- Pin tab: Right-click → Pin
- Duplicate tab: Right-click → Duplicate
- Close others: Right-click → Close Others
- Reorder: Drag-drop tabs

**Keyboard**:
- Cmd+1 through Cmd+9: Switch to tab 1-9
- Cmd+Alt+Left/Right: Navigate tabs
- Cmd+Shift+T: Reopen last closed tab

### Session Restore

**Storage Strategy**:
```typescript
interface WorkspaceSession {
  tenantId: string;
  userId: string;
  tabs: WorkspaceTab[];
  activeTabId: string;
  layoutConfig: {
    leftPanelWidth: number;
    rightPanelWidth: number;
    bottomPanelHeight: number;
    inspectorVisible: boolean;
  };
  timestamp: number;
}
```

**Persistence**:
- Save to localStorage on every tab/layout change (debounced 500ms)
- Key: `nexus-studio:session:{tenantId}:{userId}`
- Max size: 5MB limit → truncate old tab states if exceeded

**Restore Flow**:
1. App loads → check for `?restoreSession=false` (force fresh start)
2. Otherwise, load session from localStorage
3. Restore tabs in order (lazy-load modules as needed)
4. Restore active tab
5. Restore panel sizes and visibility

**Edge Cases**:
- If route no longer exists (permissions changed): skip tab, show notification
- If module fails to load: show error tab, allow user to close it
- If no session found: open default tab (Dashboard)

---

## 🔍 Global Search Implementation

### Search Index Structure

**What's Indexed**:
- Files: name, path, content (text files only, max 1MB)
- Media: name, tags, description, metadata
- KB Docs: title, content, tags, author
- Cases: subject, description, comments
- Kanban Cards: title, description, checklist items
- Datasets: name, column names, sample values
- Forms: name, field labels
- Logs: message text (recent only, max 10k entries)
- Users: name, email (if user has permission)

### Search Strategy

**Client-Side Full-Text Search** (runs in Web Worker):

```typescript
interface SearchIndex {
  documents: Map<string, SearchDocument>;
  invertedIndex: Map<string, Set<string>>; // term → document IDs
  termFrequency: Map<string, Map<string, number>>; // docId → term → frequency
}

interface SearchDocument {
  id: string;
  type: ModuleType;
  title: string;
  snippet: string;
  route: string;
  metadata: Record<string, any>;
  score?: number; // Computed at search time
}
```

**Ranking Algorithm**:
1. TF-IDF scoring for relevance
2. Boost recent documents (recency decay)
3. Boost exact matches over fuzzy
4. Boost title matches over content matches
5. User-specific boosting (recently accessed items)

**Search Features**:
- Fuzzy matching (Levenshtein distance ≤ 2)
- Prefix matching ("cust" matches "customer")
- AND/OR operators: "customer AND invoice"
- Field filters: "type:case status:open"
- Faceted results: group by module type

### Search UX

**Search Input**:
- Minimum 2 characters to trigger search
- Debounced 200ms
- Shows "Searching..." spinner
- Keyboard-navigable results (↑↓ Enter)

**Results Panel**:
```
┌─────────────────────────────────────┐
│ Search results for "customer 401"  │
├─────────────────────────────────────┤
│ 📁 Files (2)                        │
│   customer-401-invoice.pdf          │
│   customer-401-contract.docx        │
├─────────────────────────────────────┤
│ 🎫 Cases (1)                        │
│   #2491: Customer 401 billing issue │
├─────────────────────────────────────┤
│ 📊 Datasets (1)                     │
│   customers.csv (row 401)           │
└─────────────────────────────────────┘
```

**Empty State**:
- No results: Suggest related searches, show search tips
- No query: Show recent searches + trending

---

## 🔐 Route Guards & Permissions

### Guard Types

1. **Tenant Access Guard**
   ```typescript
   canActivate: [tenantAccessGuard]
   ```
   Checks if user has access to tenant in URL

2. **Role-Based Guard**
   ```typescript
   canActivate: [roleGuard(['admin', 'editor'])]
   ```
   Checks if user has one of required roles

3. **Feature Flag Guard**
   ```typescript
   canActivate: [featureFlagGuard('dataset-explorer')]
   ```
   Checks if feature is enabled for tenant

4. **Permission Guard**
   ```typescript
   canActivate: [permissionGuard('cases:write')]
   ```
   Checks specific permission string

### Guard Behavior

**On Failure**:
- Redirect to: `/unauthorized` with `?return={attemptedUrl}`
- Show toast: "You don't have access to this resource"
- Log audit event: "User X attempted to access {route} without permission"

**Deactivation Guards**:
```typescript
canDeactivate: [unsavedChangesGuard]
```
Warns user if navigating away from dirty form/editor

---

## 🎯 Navigation State Management

### Breadcrumb System

Auto-generated from route hierarchy:
```
Files > Projects > 2024 > Q1-Planning.xlsx
```

**Behavior**:
- Each segment is clickable
- Last segment is current page (not clickable)
- Max 5 segments (collapse middle with "...")
- Hover to see full path tooltip

### Back/Forward Navigation

**Browser History Integration**:
- All navigation uses `router.navigate()` (no direct history manipulation)
- Back/forward buttons work as expected
- Workspace tab state encoded in history state API

**History State**:
```typescript
history.state = {
  tabId: 'tab-123',
  scrollPosition: { x: 0, y: 500 },
  selection: ['file1', 'file2'],
};
```

---

## 📱 Responsive Considerations

**Mobile/Tablet** (future consideration):
- Left nav becomes slide-out drawer
- Top bar remains fixed
- Tabs become dropdown selector
- Split panes become stacked
- Command palette still works (mobile keyboards)

**Not in MVP**, but architecture supports it:
- Mobile routes: `/m/t/{tenantId}/{module}`
- Responsive breakpoints: 640px, 1024px, 1440px
- Touch-friendly hit targets (44px minimum)

---

## ✅ Information Architecture Checklist

- ✅ Clear mental model: Left nav for modules, top bar for global actions
- ✅ Every route is bookmarkable and shareable
- ✅ Deep-linking preserves context (filters, scroll, selection)
- ✅ Lazy loading optimizes initial bundle size
- ✅ Route-level providers scope state to features
- ✅ Session restore creates continuity across refreshes
- ✅ Global search works across all content
- ✅ Command palette provides power-user efficiency
- ✅ Permissions integrate at route level
- ✅ Tabbed workspace enables multitasking

---

**Next**: Workspace Shell (UI layout system details)
