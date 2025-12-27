# Module Designs: Admin, Cases, Datasets, Forms

> **Final module specifications completing all 10 modules with implementation-ready designs**

---

## ⚙️ Module 7: Admin Console for Multi-Tenant App

### Overview
Administrative control panel for managing tenants, users, roles, permissions, and system configuration. Includes audit log viewer, feature flag management, and system health monitoring.

---

### Primary Screens & Layout

#### Main View: Tabbed Admin Interface

```
┌────────────────────────────────────────────────────────────────┐
│ Admin Console                            [System Status: ✓]    │
├────────────────────────────────────────────────────────────────┤
│ [Users] [Roles] [Tenants] [Audit Log] [Settings]              │
├────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Users (234)                          [+ Invite User] [Filter] │
│  ──────────────────────────────────────────────────────────    │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │ Name          Email           Role      Status  Actions  │ │
│  ├──────────────────────────────────────────────────────────┤ │
│  │ 👤 Alice      alice@co  ──┐  Admin     Active   [•••]   │ │
│  │ 👤 Bob        bob@co       │  Editor    Active   [•••]   │ │
│  │ 👤 Carol      carol@co     │  Viewer    Pending  [•••]   │ │
│  │ 👤 Dave       dave@co      │  Admin     Active   [•••]   │ │
│  │ ...           ...          │  ...       ...      ...     │ │
│  │                            │                             │ │
│  │ [Virtual scroll for 1000s users]                        │ │
│  │                            │                             │ │
│  └────────────────────────────┴─────────────────────────────┘ │
│                                                                 │
│  Showing 4 of 234 • Page 1 of 12                               │
│                                                                 │
│  Bulk Actions: [Change Role] [Deactivate] [Export]            │
└────────────────────────────────────────────────────────────────┘
```

#### User Detail Panel (Slide-Over)

```
┌────────────────────────────────────────────────────────────────┐
│ Edit User: Alice Smith                                  [✕]    │
├────────────────────────────────────────────────────────────────┤
│                                                                 │
│  👤 [Avatar]                                                   │
│                                                                 │
│  Name: [Alice Smith                     ]                      │
│  Email: [alice@company.com              ]                      │
│  Status: [Active ▾]                                            │
│                                                                 │
│  Tenant Memberships:                                           │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │ Acme Corp        Role: Admin      [Edit] [Remove]        │ │
│  │ Beta Inc         Role: Editor     [Edit] [Remove]        │ │
│  └──────────────────────────────────────────────────────────┘ │
│  [+ Add to Tenant]                                             │
│                                                                 │
│  Permissions (from Admin role):                                │
│  ✓ files:read, files:write, files:delete                      │
│  ✓ cases:read, cases:write, cases:assign                      │
│  ✓ users:read, users:write                                    │
│  ✓ audit:read                                                  │
│  [View all 24 permissions]                                     │
│                                                                 │
│  Last Login: 2 hours ago                                       │
│  Created: Jan 15, 2024                                         │
│                                                                 │
│  [Save Changes] [Delete User] [Reset Password]                │
└────────────────────────────────────────────────────────────────┘
```

#### Audit Log View

```
┌────────────────────────────────────────────────────────────────┐
│ Audit Log                               [Export] [Filter]      │
├────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Filters: User [All ▾] Action [All ▾] Date [Last 7 days ▾]    │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │ Timestamp        User    Action         Entity    Details │ │
│  ├──────────────────────────────────────────────────────────┤ │
│  │ 10:45:23  👤Alice  deleted     Case      #2491          │ │
│  │ 10:44:12  👤Bob    updated     User      carol@co        │ │
│  │ 10:43:01  👤Carol  accessed    Dataset   customers.csv   │ │
│  │ 10:42:50  👤Dave   created     Role      Contractor      │ │
│  │ ...       ...      ...         ...       ...             │ │
│  │                                                           │ │
│  │ [Virtual scroll for millions of audit events]            │ │
│  │                                                           │ │
│  └──────────────────────────────────────────────────────────┘ │
│                                                                 │
│  12,345 events • Export to CSV for compliance reports          │
└────────────────────────────────────────────────────────────────┘
```

---

### Hard UI Features

#### 1. **User/Role Matrix (Inline Editing)**
- **What**: Grid showing users × roles, click cell to assign/remove role
- **Why**: Bulk role management, visual overview
- **Implementation**: Editable table component, checkbox cells, optimistic updates

#### 2. **Bulk User Operations**
- **What**: Select multiple users (checkboxes) → apply action to all (change role, deactivate, export)
- **Why**: Admin efficiency for large user bases
- **Implementation**: Selection model, bulk update API, confirmation dialogs

#### 3. **Advanced Filtering & Search**
- **What**: Filter users by role, status, tenant, last login date; search by name/email
- **Why**: Find users quickly in large organizations (1000s of users)
- **Implementation**: Filter panel with chips, debounced search, URL query sync

#### 4. **Audit Log with Export**
- **What**: View all system actions, filter by user/action/entity, export to CSV for compliance
- **Why**: Security audits, compliance (SOC 2, GDPR)
- **Implementation**: Virtual scroll for millions of events, export service, date range picker

#### 5. **Permission Visualization (Tree View)**
- **What**: Hierarchical tree of permissions, expandable categories, checkboxes to grant/revoke
- **Why**: Understand complex permission structures
- **Implementation**: Tree component, permission aggregation, inheritance display

#### 6. **Role Template Editor**
- **What**: Create role from template (Admin, Editor, Viewer), customize permissions
- **Why**: Consistency, quick role creation
- **Implementation**: Template definitions, permission picker, validation

#### 7. **Tenant Switcher (Admin View All)**
- **What**: Admin can impersonate any tenant to troubleshoot issues
- **Why**: Support, debugging
- **Implementation**: Tenant context override, security logging, warning banner

#### 8. **Feature Flag Management**
- **What**: Toggle features on/off per tenant (e.g., enable Dataset Explorer for Pro tier)
- **Why**: Progressive rollout, A/B testing, tier gating
- **Implementation**: Feature flag store, toggle UI, sync to backend

#### 9. **User Invitation Flow**
- **What**: Invite user by email → select role → send invitation → user sets password
- **Why**: Onboarding, access control
- **Implementation**: Multi-step wizard, email template, token generation

#### 10. **Activity Dashboard (Metrics)**
- **What**: Charts showing active users, actions per day, storage usage, API calls
- **Why**: System health monitoring, capacity planning
- **Implementation**: Chart library (Chart.js), metrics aggregation, time series

---

### Component Breakdown

**1. AdminShellComponent**
- **Responsibility**: Main container with tabbed interface
- **State**: Active tab, filter state
- **Template**:
```html
<div class="admin-shell">
  <mat-tab-group [(selectedIndex)]="activeTabIndex()">
    <mat-tab label="Users">
      <app-user-management />
    </mat-tab>
    <mat-tab label="Roles">
      <app-role-management />
    </mat-tab>
    <mat-tab label="Tenants">
      <app-tenant-management />
    </mat-tab>
    <mat-tab label="Audit Log">
      <app-audit-log />
    </mat-tab>
    <mat-tab label="Settings">
      <app-system-settings />
    </mat-tab>
  </mat-tab-group>
</div>
```

**2. UserManagementComponent**
- **Responsibility**: User list with CRUD operations
- **Features**: Virtual scroll, multi-select, inline edit, bulk actions
- **ARIA**: `role="table"`, `role="row"`, `role="cell"`
- **Keyboard**: Arrow keys to navigate, Space to select, Enter to edit

**3. UserEditorComponent**
- **Responsibility**: Slide-over panel for user details
- **Features**: Form validation, tenant membership editor, password reset
- **ARIA**: `role="dialog"`, focus trap
- **Keyboard**: Tab through fields, Esc to close, Cmd+S to save

**4. RoleManagementComponent**
- **Responsibility**: Role list and permission editor
- **Features**: Permission tree, role templates, usage count
- **ARIA**: `role="tree"` for permissions
- **Keyboard**: Arrow keys for tree navigation

**5. PermissionTreeComponent**
- **Responsibility**: Hierarchical permission selector
- **Features**: Expand/collapse categories, check/uncheck, inheritance indicators
- **ARIA**: `role="tree"`, `role="treeitem"`, `role="checkbox"`
- **Keyboard**: Arrow keys, Space to toggle

**6. AuditLogComponent**
- **Responsibility**: Event log viewer
- **Features**: Virtual scroll, filters, export, detail expansion
- **ARIA**: `role="log"`, `role="article"` per event
- **Keyboard**: Arrow keys, Enter for details

**7. TenantManagementComponent**
- **Responsibility**: Tenant CRUD and settings
- **Features**: Tenant list, feature flags, storage quotas, billing info
- **ARIA**: `role="table"`
- **Keyboard**: Standard table navigation

**8. FeatureFlagTogglesComponent**
- **Responsibility**: Feature flag switches per tenant
- **Features**: Toggle switches, descriptions, rollout percentage
- **ARIA**: `role="switch"`, `aria-checked`
- **Keyboard**: Space to toggle

---

### State Design

#### AdminStore

```typescript
export interface AdminState {
  // Users
  users: User[];
  totalUsers: number;
  currentPage: number;
  selectedUserIds: string[];

  // Roles
  roles: Role[];

  // Tenants
  tenants: Tenant[];

  // Audit log
  auditEvents: ActivityEvent[];

  // Filters
  userFilters: UserFilters;
  auditFilters: AuditFilters;

  // UI
  userEditorOpen: boolean;
  editingUserId: string | null;
  loading: boolean;
  error: string | null;
}

export const AdminStore = signalStore(
  { providedIn: 'root' },

  withState<AdminState>({
    users: [],
    totalUsers: 0,
    currentPage: 1,
    selectedUserIds: [],
    roles: [],
    tenants: [],
    auditEvents: [],
    userFilters: {},
    auditFilters: {},
    userEditorOpen: false,
    editingUserId: null,
    loading: false,
    error: null,
  }),

  withComputed((store) => ({
    // Filtered users
    filteredUsers: computed(() => {
      let users = store.users();

      if (store.userFilters().role) {
        users = users.filter(u =>
          u.tenants.some(t => t.roleId === store.userFilters().role)
        );
      }

      if (store.userFilters().status) {
        users = users.filter(u => u.isActive === (store.userFilters().status === 'active'));
      }

      if (store.userFilters().search) {
        const query = store.userFilters().search.toLowerCase();
        users = users.filter(u =>
          u.name.toLowerCase().includes(query) ||
          u.email.toLowerCase().includes(query)
        );
      }

      return users;
    }),

    // Selected users
    selectedUsers: computed(() =>
      store.users().filter(u => store.selectedUserIds().includes(u.id))
    ),

    // Can perform bulk action
    canBulkAction: computed(() => store.selectedUserIds().length > 0),
  })),

  withMethods((store, adminService = inject(AdminService)) => ({
    // Load users
    async loadUsers(page: number = 1) {
      patchState(store, { loading: true, currentPage: page });

      try {
        const { users, total } = await adminService.getUsers({
          page,
          limit: 50,
          filters: store.userFilters(),
        });

        patchState(store, { users, totalUsers: total, loading: false });
      } catch (error) {
        patchState(store, { loading: false, error: error.message });
      }
    },

    // Create user
    async createUser(userData: Partial<User>) {
      const newUser = await adminService.createUser(userData);

      patchState(store, (state) => ({
        users: [newUser, ...state.users],
        totalUsers: state.totalUsers + 1,
      }));
    },

    // Update user
    async updateUser(userId: string, updates: Partial<User>) {
      await adminService.updateUser(userId, updates);

      patchState(store, (state) => ({
        users: state.users.map(u => u.id === userId ? { ...u, ...updates } : u),
      }));
    },

    // Delete user
    async deleteUser(userId: string) {
      await adminService.deleteUser(userId);

      patchState(store, (state) => ({
        users: state.users.filter(u => u.id !== userId),
        totalUsers: state.totalUsers - 1,
      }));
    },

    // Bulk change role
    async bulkChangeRole(userIds: string[], roleId: string) {
      await adminService.bulkUpdateUsers(userIds, { roleId });

      patchState(store, (state) => ({
        users: state.users.map(u =>
          userIds.includes(u.id)
            ? { ...u, tenants: u.tenants.map(t => ({ ...t, roleId })) }
            : u
        ),
        selectedUserIds: [],
      }));
    },

    // Load audit log
    async loadAuditLog() {
      patchState(store, { loading: true });

      try {
        const events = await adminService.getAuditLog(store.auditFilters());
        patchState(store, { auditEvents: events, loading: false });
      } catch (error) {
        patchState(store, { loading: false, error: error.message });
      }
    },

    // Export audit log
    async exportAuditLog(): Promise<string> {
      const events = store.auditEvents();
      const csv = this.convertToCSV(events);
      return csv;
    },

    // Toggle feature flag
    async toggleFeatureFlag(tenantId: string, feature: string, enabled: boolean) {
      await adminService.updateTenantFeatureFlag(tenantId, feature, enabled);

      patchState(store, (state) => ({
        tenants: state.tenants.map(t =>
          t.id === tenantId
            ? { ...t, features: { ...t.features, [feature]: enabled } }
            : t
        ),
      }));
    },

    // Helper: Convert to CSV
    convertToCSV(events: ActivityEvent[]): string {
      const headers = 'Timestamp,User,Action,Entity,Details\n';
      const rows = events.map(e =>
        `${e.timestamp.toISOString()},${e.userName},${e.action},${e.entityType},${e.entityName}`
      ).join('\n');
      return headers + rows;
    },
  })),
);

interface UserFilters {
  role?: string;
  status?: 'active' | 'inactive';
  search?: string;
  tenant?: string;
}

interface AuditFilters {
  userId?: string;
  action?: string;
  entityType?: string;
  dateRange?: { start: Date; end: Date };
}
```

---

### Accessibility

**ARIA Patterns**:
- **Table** (user list): `role="table"`, `role="row"`, `role="columnheader"`, `role="cell"`
- **Dialog** (user editor): `role="dialog"`, focus trap
- **Tree** (permission editor): `role="tree"`, `role="treeitem"`, `role="checkbox"`
- **Switch** (feature flags): `role="switch"`, `aria-checked`
- **Tabs** (admin sections): `role="tablist"`, `role="tab"`, `role="tabpanel"`

**Keyboard Shortcuts**:
- `N`: New user
- `E`: Edit selected user
- `Delete`: Delete selected users (with confirmation)
- `Cmd+A`: Select all
- `Cmd+F`: Focus search
- `Cmd+E`: Export
- `Cmd+S`: Save changes

**Screen Reader**:
- "User table: 234 users, 4 selected"
- "Alice Smith, Admin role, Active status"
- "Bulk action: Change role for 4 users"
- "Audit log: 12,345 events"

---

## 🎫 Module 8: Email/Case Triage Console

### Overview
Support ticket management system with email-style inbox, case timeline, SLA tracking, assignment, and templated responses. Handles high volumes (1000s of cases) with smart filtering and priority queues.

---

### Primary Screens & Layout

#### Main View: Inbox + Case Detail Split

```
┌────────────────────────────────────────────────────────────────┐
│ Cases                       [New Case] [Refresh] [Settings]    │
├──────────────┬─────────────────────────────────────────────────┤
│              │                                                 │
│  Filters     │  Case List (Virtual Scroll)                    │
│              │  ┌───────────────────────────────────────────┐ │
│  Status      │  │ ⚡ CASE-2491  Failed payment processing   │ │
│  ☑ Open      │  │ Priority: High | Assigned: Alice | 2h ago│ │
│  ☑ Pending   │  │ SLA: 4h remaining ░░░░░░░░░▓▓ 75%        │ │
│  ☐ Resolved  │  ├───────────────────────────────────────────┤ │
│  ☐ Closed    │  │   CASE-2490  Login issues                │ │
│              │  │ Priority: Med | Unassigned | 1d ago      │ │
│  Priority    │  │ SLA: Overdue ▓▓▓▓▓▓▓▓▓▓ 100% ⚠️         │ │
│  ☑ Critical  │  ├───────────────────────────────────────────┤ │
│  ☑ High      │  │   CASE-2489  Feature request             │ │
│  ☑ Medium    │  │ Priority: Low | Assigned: Bob | 3d ago   │ │
│  ☐ Low       │  │ SLA: On track ░░ 20%                     │ │
│              │  ├───────────────────────────────────────────┤ │
│  Assignee    │  │ [Showing 3 of 1,234 cases]               │ │
│  ☑ Me        │  └───────────────────────────────────────────┘ │
│  ☐ Unassign  │                                                 │
│  ☐ Team      │  [Click case to view details in right pane]   │
│              │                                                 │
│ [Clear All]  │                                                 │
└──────────────┴─────────────────────────────────────────────────┘
```

#### Case Detail Pane (Right Side)

```
┌────────────────────────────────────────────────────────────────┐
│ CASE-2491: Failed payment processing                    [✕]    │
├────────────────────────────────────────────────────────────────┤
│  From: customer@company.com                                    │
│  Priority: [High ▾]  Status: [Open ▾]  Assigned: [Alice ▾]    │
│  Tags: [🏷️ billing] [🏷️ urgent] [+ Add]                       │
│  SLA Deadline: 2:30 PM (4h remaining)  Progress: ░░░░▓▓ 75%   │
│                                                                 │
│  Timeline:                                                      │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │ 👤 Customer (10:23 AM)                                   │ │
│  │ Hi, I tried to process a payment but it failed.         │ │
│  │ Error: "Card declined"                                   │ │
│  │                                                          │ │
│  │ Attachments: screenshot.png                              │ │
│  ├──────────────────────────────────────────────────────────┤ │
│  │ 🤖 System (10:24 AM)                                     │ │
│  │ Auto-assigned to Alice (Payment team)                   │ │
│  ├──────────────────────────────────────────────────────────┤ │
│  │ 👤 Alice (10:45 AM)                                      │ │
│  │ Thanks for reporting! I'll investigate this.            │ │
│  │                                                          │ │
│  │ [Internal note: Checked logs, seems like bank issue]    │ │
│  ├──────────────────────────────────────────────────────────┤ │
│  │ [Type reply...]                                          │ │
│  │                                                          │ │
│  │ [Templates ▾] [📎 Attach] [Internal Note]               │ │
│  │                                           [Send Reply]   │ │
│  └──────────────────────────────────────────────────────────┘ │
│                                                                 │
│  Linked: 📊 Logs (5 errors) | 📁 invoice.pdf                  │
│                                                                 │
│  [Resolve] [Close] [Escalate] [Merge]                         │
└────────────────────────────────────────────────────────────────┘
```

---

### Hard UI Features

#### 1. **Virtual Scrolling for Case List (1000s of Cases)**
- **What**: Inbox-style list with virtual scroll for thousands of cases
- **Why**: Support teams handle massive case volumes
- **Implementation**: CDK Virtual Scroll, fixed item height, selection model

#### 2. **SLA Progress Indicators**
- **What**: Visual progress bar showing time until SLA deadline, color-coded (green → yellow → red)
- **Why**: Prioritize cases about to breach SLA
- **Implementation**: Computed time remaining, progress bar component, warning thresholds

#### 3. **Smart Assignment (Auto-Route)**
- **What**: Automatically assign cases to users based on keywords, tags, or round-robin
- **Why**: Distribute workload, route to experts
- **Implementation**: Rule engine, keyword matching, load balancing algorithm

#### 4. **Templated Responses**
- **What**: Dropdown of pre-written responses (e.g., "Password reset instructions"), insert with one click
- **Why**: Faster responses, consistency
- **Implementation**: Template library, variable substitution (`{{customer.name}}`), editor integration

#### 5. **Rich Text Reply Editor**
- **What**: Formatting toolbar (bold, italic, list, code), attachments, internal notes
- **Why**: Professional responses, context for team
- **Implementation**: Rich text editor (Quill.js or TipTap), attachment uploader, note toggle

#### 6. **Case Timeline View**
- **What**: Chronological feed of customer messages, agent replies, system events, status changes
- **Why**: Full context at a glance
- **Implementation**: Timeline component, event types, expand/collapse

#### 7. **Bulk Case Operations**
- **What**: Select multiple cases → assign to user, change status, add tags, close
- **Why**: Batch processing (e.g., close all resolved cases from last week)
- **Implementation**: Multi-select, bulk action menu, confirmation dialog

#### 8. **Advanced Filtering & Saved Views**
- **What**: Filter by status, priority, assignee, tags, date range; save filter as "My Queue"
- **Why**: Custom workflows per agent
- **Implementation**: Filter service, saved view presets, URL persistence

#### 9. **Case Merging (Duplicate Detection)**
- **What**: Merge duplicate cases → combine timelines, close duplicates
- **Why**: Reduce clutter, unified customer history
- **Implementation**: Similarity detection (subject matching), merge UI, timeline consolidation

#### 10. **Linked Entities (Cross-Reference)**
- **What**: Link case to logs, datasets, files, kanban cards; show in sidebar
- **Why**: Context from other modules (e.g., "Customer reported error" → link to error logs)
- **Implementation**: Entity reference picker, link storage, cross-module navigation

---

### Component Breakdown

**1. CasesShellComponent**
- **Responsibility**: Main container with list + detail split
- **State**: Active case, filter state, selection

**2. CaseListComponent**
- **Responsibility**: Virtual scrolling case inbox
- **Features**: SLA indicators, priority badges, multi-select, unread count
- **ARIA**: `role="listbox"`, `role="option"`
- **Keyboard**: Arrow keys, Enter to open, Space to select

**3. CaseDetailComponent**
- **Responsibility**: Case viewer/editor in right pane
- **Features**: Timeline, reply editor, status/assignment pickers
- **ARIA**: `role="main"`, `role="form"` for reply
- **Keyboard**: Tab through fields, Cmd+Enter to send

**4. CaseTimelineComponent**
- **Responsibility**: Chronological event feed
- **Features**: Message bubbles, system events, collapse old messages
- **ARIA**: `role="feed"`, `role="article"` per event
- **Keyboard**: Arrow keys to navigate events

**5. ReplyEditorComponent**
- **Responsibility**: Rich text editor for replies
- **Features**: Formatting, templates, attachments, internal note toggle
- **ARIA**: `role="textbox"`, `aria-multiline="true"`
- **Keyboard**: Standard text editing, Cmd+B/I for formatting

**6. SLAIndicatorComponent**
- **Responsibility**: Visual SLA progress bar
- **Features**: Time remaining, color coding, tooltip with deadline
- **ARIA**: `role="progressbar"`, `aria-valuenow/min/max`
- **Keyboard**: N/A (visual only)

**7. CaseFiltersPanelComponent**
- **Responsibility**: Filter sidebar
- **Features**: Checkboxes, date picker, saved views
- **ARIA**: `role="form"`, fieldsets
- **Keyboard**: Standard form navigation

**8. BulkActionsMenuComponent**
- **Responsibility**: Actions for selected cases
- **Features**: Assign, close, tag, merge
- **ARIA**: `role="menu"`, `role="menuitem"`
- **Keyboard**: Arrow keys, Enter to select

---

### State Design

#### CasesStore

```typescript
export interface CasesState {
  // Cases
  cases: Case[];
  totalCases: number;
  currentPage: number;

  // Active case
  activeCaseId: string | null;

  // Filters
  filters: CaseFilters;

  // Selection
  selectedCaseIds: string[];

  // Templates
  replyTemplates: ReplyTemplate[];

  // UI
  replyDraft: string;
  loading: boolean;
  error: string | null;
}

export const CasesStore = signalStore(
  { providedIn: 'root' },

  withState<CasesState>({
    cases: [],
    totalCases: 0,
    currentPage: 1,
    activeCaseId: null,
    filters: { status: ['open', 'pending'] },
    selectedCaseIds: [],
    replyTemplates: [],
    replyDraft: '',
    loading: false,
    error: null,
  }),

  withComputed((store) => ({
    // Active case
    activeCase: computed(() =>
      store.cases().find(c => c.id === store.activeCaseId())
    ),

    // Filtered cases
    filteredCases: computed(() => {
      let cases = store.cases();

      if (store.filters().status?.length) {
        cases = cases.filter(c => store.filters().status.includes(c.status));
      }

      if (store.filters().priority?.length) {
        cases = cases.filter(c => store.filters().priority.includes(c.priority));
      }

      if (store.filters().assignee) {
        if (store.filters().assignee === 'me') {
          const userId = inject(AuthService).currentUser().id;
          cases = cases.filter(c => c.assignedTo === userId);
        } else if (store.filters().assignee === 'unassigned') {
          cases = cases.filter(c => !c.assignedTo);
        }
      }

      return cases;
    }),

    // SLA status per case
    slaStatus: computed(() => {
      const status = new Map<string, SLAStatus>();

      store.cases().forEach(c => {
        if (!c.slaDeadline) {
          status.set(c.id, { status: 'none' });
          return;
        }

        const now = new Date();
        const deadline = new Date(c.slaDeadline);
        const totalTime = deadline.getTime() - new Date(c.createdAt).getTime();
        const elapsed = now.getTime() - new Date(c.createdAt).getTime();
        const progress = Math.min(100, (elapsed / totalTime) * 100);

        if (now > deadline) {
          status.set(c.id, { status: 'overdue', progress: 100 });
        } else if (progress > 75) {
          status.set(c.id, { status: 'warning', progress });
        } else {
          status.set(c.id, { status: 'ok', progress });
        }
      });

      return status;
    }),
  })),

  withMethods((store, casesService = inject(CasesService)) => ({
    // Load cases
    async loadCases(page: number = 1) {
      patchState(store, { loading: true, currentPage: page });

      try {
        const { cases, total } = await casesService.getCases({
          page,
          limit: 50,
          filters: store.filters(),
        });

        patchState(store, { cases, totalCases: total, loading: false });
      } catch (error) {
        patchState(store, { loading: false, error: error.message });
      }
    },

    // Open case
    setActiveCase(caseId: string) {
      patchState(store, { activeCaseId: caseId });
    },

    // Add reply
    async addReply(caseId: string, content: string, isInternal: boolean = false) {
      const comment: CaseComment = {
        id: generateId(),
        caseId,
        authorId: inject(AuthService).currentUser().id,
        authorName: inject(AuthService).currentUser().name,
        content,
        isInternal,
        createdAt: new Date(),
        attachments: [],
      };

      await casesService.addComment(caseId, comment);

      patchState(store, (state) => ({
        cases: state.cases.map(c =>
          c.id === caseId
            ? { ...c, comments: [...c.comments, comment], updatedAt: new Date() }
            : c
        ),
        replyDraft: '',
      }));
    },

    // Update case
    async updateCase(caseId: string, updates: Partial<Case>) {
      await casesService.updateCase(caseId, updates);

      patchState(store, (state) => ({
        cases: state.cases.map(c =>
          c.id === caseId ? { ...c, ...updates } : c
        ),
      }));
    },

    // Bulk assign
    async bulkAssign(caseIds: string[], userId: string) {
      await Promise.all(
        caseIds.map(id => casesService.updateCase(id, { assignedTo: userId }))
      );

      patchState(store, (state) => ({
        cases: state.cases.map(c =>
          caseIds.includes(c.id) ? { ...c, assignedTo: userId } : c
        ),
        selectedCaseIds: [],
      }));
    },

    // Merge cases
    async mergeCases(primaryCaseId: string, duplicateIds: string[]) {
      await casesService.mergeCases(primaryCaseId, duplicateIds);

      patchState(store, (state) => ({
        cases: state.cases.filter(c => !duplicateIds.includes(c.id)),
      }));
    },

    // Insert template
    insertTemplate(templateId: string) {
      const template = store.replyTemplates().find(t => t.id === templateId);
      if (!template) return;

      // Replace variables
      let content = template.content;
      const activeCase = store.activeCase();
      if (activeCase) {
        content = content.replace('{{customer.name}}', activeCase.customerName);
        content = content.replace('{{case.id}}', activeCase.id);
      }

      patchState(store, { replyDraft: content });
    },
  })),
);

interface CaseFilters {
  status?: CaseStatus[];
  priority?: CasePriority[];
  assignee?: 'me' | 'unassigned' | 'team' | string;
  tags?: string[];
  dateRange?: { start: Date; end: Date };
}

interface SLAStatus {
  status: 'ok' | 'warning' | 'overdue' | 'none';
  progress?: number;
}

interface ReplyTemplate {
  id: string;
  name: string;
  content: string;
  category: string;
}
```

---

### Accessibility

**ARIA Patterns**:
- **Listbox** (case list): `role="listbox"`, `role="option"`
- **Feed** (timeline): `role="feed"`, `role="article"`
- **Textbox** (reply editor): `role="textbox"`, `aria-multiline="true"`
- **Progressbar** (SLA): `role="progressbar"`, `aria-valuenow`
- **Menu** (bulk actions): `role="menu"`, `role="menuitem"`

**Keyboard Shortcuts**:
- `N`: New case
- `R`: Reply to active case
- `A`: Assign case
- `C`: Close case
- `T`: Add tag
- `J/K`: Navigate cases (Gmail-style)
- `Enter`: Open case
- `Esc`: Close detail pane

**Screen Reader**:
- "Case list: 1,234 cases, 3 selected"
- "CASE-2491, High priority, SLA warning, 4 hours remaining"
- "Assigned to Alice, opened 2 hours ago"
- "5 comments in timeline"

---

## 📈 Module 9: Dataset Explorer

### Overview
CSV/JSON data viewer and profiler for analyzing tabular data. Features virtual grid for 100k+ rows, column statistics, query builder, data profiling (in Web Worker), and export capabilities.

---

### Primary Screens & Layout

#### Main View: Data Grid + Profiler Sidebar

```
┌────────────────────────────────────────────────────────────────┐
│ Dataset: customers.csv          [Profile] [Query] [Export]    │
├────────────────────────────────────────────────────────────────┤
│  10,234 rows × 12 columns • 2.3 MB • Last profiled: 2h ago    │
├────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │ ID    Name           Email          City      Revenue    │ │
│  ├──────────────────────────────────────────────────────────┤ │
│  │ 1     Alice Smith    alice@co       NYC       $45,230   │ │
│  │ 2     Bob Jones      bob@co         LA        $32,100   │ │
│  │ 3     Carol White    carol@co       Chicago   $78,900   │ │
│  │ 4     Dave Brown     dave@co        Houston   $23,450   │ │
│  │ ...   ...            ...            ...       ...       │ │
│  │                                                          │ │
│  │ [Virtual scroll for 100k+ rows]                         │ │
│  │ [Column sorting, filtering, resizing]                   │ │
│  │                                                          │ │
│  └──────────────────────────────────────────────────────────┘ │
│                                                                 │
│  Showing rows 1-20 of 10,234                    [Page controls]│
└────────────────────────────────────────────────────────────────┘
```

#### Column Profiler Panel (Slide-Over)

```
┌────────────────────────────────────────────────────────────────┐
│ Column: Revenue                                          [✕]    │
├────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Type: Number                                                  │
│  Nullable: No                                                  │
│                                                                 │
│  Statistics:                                                    │
│  Count: 10,234                                                 │
│  Unique: 8,421 (82%)                                           │
│  Nulls: 0 (0%)                                                 │
│                                                                 │
│  Min: $1,250                                                   │
│  Max: $982,340                                                 │
│  Mean: $45,670                                                 │
│  Median: $38,200                                               │
│                                                                 │
│  Distribution:                                                  │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │      █                                                   │ │
│  │    ███                                                   │ │
│  │  ██████                                                  │ │
│  │ ████████                                                 │ │
│  │ ────────────────────────────────────────────────────     │ │
│  │ 0     25K    50K    75K   100K                          │ │
│  └──────────────────────────────────────────────────────────┘ │
│                                                                 │
│  Top Values:                                                    │
│  $45,000 (142 occurrences)                                     │
│  $50,000 (128 occurrences)                                     │
│  $40,000 (115 occurrences)                                     │
│                                                                 │
│  [Export column] [Create filter]                               │
└────────────────────────────────────────────────────────────────┘
```

---

### Hard UI Features

#### 1. **Virtual Grid for 100k+ Rows**
- **What**: Grid that renders only visible rows/columns, smooth scrolling for massive datasets
- **Why**: CSV files can have millions of rows, rendering all would crash browser
- **Implementation**: Custom virtual scroll grid (not CDK, need 2D virtualization), buffer zones, sticky headers

#### 2. **Data Profiling in Web Worker**
- **What**: Calculate column stats (min, max, mean, median, distribution) in background thread
- **Why**: Don't freeze UI while processing 100k rows
- **Implementation**: Dataset profiler worker (from DATA_LAYER.md), message passing, progress indicator

#### 3. **Column Sorting & Filtering**
- **What**: Click column header to sort, add filters (e.g., Revenue > 50000)
- **Why**: Explore data, find patterns
- **Implementation**: Client-side sort/filter for small datasets, query builder for large ones

#### 4. **Query Builder (Visual SQL)**
- **What**: Build queries with dropdowns (WHERE Revenue > 50000 AND City = 'NYC')
- **Why**: Non-technical users can filter without writing SQL
- **Implementation**: Query builder component, AST generation, client-side evaluation

#### 5. **Data Type Inference**
- **What**: Auto-detect column types (number, string, date, boolean) from sample values
- **Why**: Proper sorting, aggregations, visualizations
- **Implementation**: Type inference algorithm (check first 100 rows), type override option

#### 6. **Column Statistics Panel**
- **What**: Click column → see stats (count, unique, nulls, min/max, distribution chart)
- **Why**: Understand data quality and distribution
- **Implementation**: Profiler worker results, chart library (Chart.js), slide-over panel

#### 7. **Export Filtered Data**
- **What**: Apply filters → export filtered rows as CSV/JSON/Excel
- **Why**: Share subsets with team
- **Implementation**: Filter result set, serialize to format, download link

#### 8. **Cell Editing (Inline)**
- **What**: Double-click cell → edit value → save
- **Why**: Quick data corrections
- **Implementation**: Editable grid cells, validation, dirty state tracking

#### 9. **Column Operations (Bulk)**
- **What**: Select column → rename, delete, change type, create calculated column
- **Why**: Data transformation
- **Implementation**: Column menu, transform service, recompute grid

#### 10. **Search/Highlight Across Dataset**
- **What**: Search for value → highlight all matching cells
- **Why**: Find specific records
- **Implementation**: Full-dataset search (worker for large datasets), cell highlighting

---

### Component Breakdown

**1. DatasetExplorerShellComponent**
- **Responsibility**: Main container with grid + toolbar
- **State**: Active dataset, filter state, profiling status

**2. VirtualDataGridComponent**
- **Responsibility**: 2D virtual scrolling grid
- **Features**: Sticky headers, column resize, sort, cell selection
- **ARIA**: `role="grid"`, `role="row"`, `role="columnheader"`, `role="gridcell"`
- **Keyboard**: Arrow keys to navigate cells, Enter to edit, Cmd+C to copy

**3. ColumnHeaderComponent**
- **Responsibility**: Grid column header with sort/filter
- **Features**: Click to sort, dropdown for filter, resize handle
- **ARIA**: `role="columnheader"`, `aria-sort`
- **Keyboard**: Enter to sort, Space for menu

**4. DataProfilerPanelComponent**
- **Responsibility**: Column statistics sidebar
- **Features**: Stats, distribution chart, top values
- **ARIA**: `role="region"`, `aria-label="Column statistics"`
- **Keyboard**: Tab through stats, Esc to close

**5. QueryBuilderComponent**
- **Responsibility**: Visual query interface
- **Features**: Add/remove conditions, combine with AND/OR, preview results
- **ARIA**: `role="form"`, proper labels
- **Keyboard**: Tab through controls, Enter to add condition

**6. DataExportDialogComponent**
- **Responsibility**: Export options
- **Features**: Format selection (CSV/JSON/Excel), include headers, download
- **ARIA**: `role="dialog"`
- **Keyboard**: Tab through options, Enter to export

**7. ColumnMenuComponent**
- **Responsibility**: Column operations dropdown
- **Features**: Rename, delete, change type, profile, filter
- **ARIA**: `role="menu"`, `role="menuitem"`
- **Keyboard**: Arrow keys, Enter to select

---

### State Design

#### DatasetsStore

```typescript
export interface DatasetsState {
  // Datasets
  datasets: Dataset[];
  activeDatasetId: string | null;

  // Data
  rows: any[];                       // Actual data rows
  columns: DatasetColumn[];

  // Profiling
  profilingStatus: 'idle' | 'profiling' | 'complete' | 'error';
  profilingProgress: number;
  stats: DatasetStats | null;

  // Query/Filters
  filters: DataFilter[];
  sortBy: { column: string; order: 'asc' | 'desc' } | null;

  // Pagination (for large datasets)
  currentPage: number;
  pageSize: number;

  // Selection
  selectedCells: CellSelection[];
  selectedColumn: string | null;

  // UI
  columnProfilerOpen: boolean;
  queryBuilderOpen: boolean;
  loading: boolean;
  error: string | null;
}

export const DatasetsStore = signalStore(
  { providedIn: 'root' },

  withState<DatasetsState>({
    datasets: [],
    activeDatasetId: null,
    rows: [],
    columns: [],
    profilingStatus: 'idle',
    profilingProgress: 0,
    stats: null,
    filters: [],
    sortBy: null,
    currentPage: 1,
    pageSize: 100,
    selectedCells: [],
    selectedColumn: null,
    columnProfilerOpen: false,
    queryBuilderOpen: false,
    loading: false,
    error: null,
  }),

  withComputed((store) => ({
    // Active dataset
    activeDataset: computed(() =>
      store.datasets().find(d => d.id === store.activeDatasetId())
    ),

    // Filtered & sorted rows
    displayRows: computed(() => {
      let rows = store.rows();

      // Apply filters
      store.filters().forEach(filter => {
        rows = rows.filter(row => this.evaluateFilter(row, filter));
      });

      // Apply sorting
      if (store.sortBy()) {
        const { column, order } = store.sortBy()!;
        rows = rows.sort((a, b) => {
          const aVal = a[column];
          const bVal = b[column];
          const comparison = aVal < bVal ? -1 : aVal > bVal ? 1 : 0;
          return order === 'asc' ? comparison : -comparison;
        });
      }

      return rows;
    }),

    // Paginated rows
    paginatedRows: computed(() => {
      const start = (store.currentPage() - 1) * store.pageSize();
      const end = start + store.pageSize();
      return store.displayRows().slice(start, end);
    }),

    // Total pages
    totalPages: computed(() =>
      Math.ceil(store.displayRows().length / store.pageSize())
    ),

    // Column stats for selected column
    selectedColumnStats: computed(() => {
      const colName = store.selectedColumn();
      if (!colName || !store.stats()) return null;

      return store.stats()!.columnStats.find(c => c.name === colName);
    }),
  })),

  withMethods((store, datasetService = inject(DatasetService)) => ({
    // Load dataset
    async loadDataset(datasetId: string) {
      patchState(store, { loading: true, activeDatasetId: datasetId });

      try {
        const dataset = await datasetService.getDataset(datasetId);
        const rows = await datasetService.getRows(datasetId);

        patchState(store, {
          rows,
          columns: dataset.columns,
          stats: dataset.stats,
          loading: false,
        });

        // Profile if not already profiled
        if (!dataset.stats) {
          this.profileDataset();
        }
      } catch (error) {
        patchState(store, { loading: false, error: error.message });
      }
    },

    // Profile dataset in worker
    async profileDataset() {
      patchState(store, { profilingStatus: 'profiling', profilingProgress: 0 });

      const worker = new Worker(
        new URL('../workers/dataset-profiler.worker.ts', import.meta.url),
        { type: 'module' }
      );

      worker.onmessage = (event) => {
        const { type, payload } = event.data;

        if (type === 'progress') {
          patchState(store, { profilingProgress: payload.progress });
        } else if (type === 'complete') {
          patchState(store, {
            stats: payload.stats,
            profilingStatus: 'complete',
            profilingProgress: 100,
          });
          worker.terminate();
        }
      };

      worker.postMessage({
        type: 'profile',
        payload: {
          rows: store.rows(),
          columns: store.columns().map(c => c.name),
        },
      });
    },

    // Add filter
    addFilter(filter: DataFilter) {
      patchState(store, (state) => ({
        filters: [...state.filters, filter],
      }));
    },

    // Remove filter
    removeFilter(filterId: string) {
      patchState(store, (state) => ({
        filters: state.filters.filter(f => f.id !== filterId),
      }));
    },

    // Sort by column
    sortByColumn(column: string) {
      patchState(store, (state) => {
        const current = state.sortBy;
        let order: 'asc' | 'desc' = 'asc';

        // Toggle sort order if same column
        if (current?.column === column) {
          order = current.order === 'asc' ? 'desc' : 'asc';
        }

        return { sortBy: { column, order } };
      });
    },

    // Export data
    exportData(format: 'csv' | 'json'): string {
      const rows = store.displayRows();

      if (format === 'json') {
        return JSON.stringify(rows, null, 2);
      } else {
        // CSV
        const headers = store.columns().map(c => c.name).join(',');
        const csvRows = rows.map(row =>
          store.columns().map(c => `"${row[c.name]}"`).join(',')
        ).join('\n');
        return headers + '\n' + csvRows;
      }
    },

    // Open column profiler
    openColumnProfiler(column: string) {
      patchState(store, {
        selectedColumn: column,
        columnProfilerOpen: true,
      });
    },

    // Evaluate filter (helper)
    evaluateFilter(row: any, filter: DataFilter): boolean {
      const value = row[filter.column];

      switch (filter.operator) {
        case '==': return value == filter.value;
        case '!=': return value != filter.value;
        case '>': return value > filter.value;
        case '<': return value < filter.value;
        case 'contains': return String(value).includes(filter.value);
        default: return true;
      }
    },
  })),
);

interface DataFilter {
  id: string;
  column: string;
  operator: '==' | '!=' | '>' | '<' | 'contains';
  value: any;
}

interface CellSelection {
  row: number;
  column: string;
}
```

---

### Accessibility

**ARIA Patterns**:
- **Grid** (data table): `role="grid"`, `role="row"`, `role="gridcell"`, `role="columnheader"`
- **Dialog** (export/query): `role="dialog"`
- **Form** (query builder): Proper labels, fieldsets
- **Region** (profiler): `role="region"`, `aria-label`

**Keyboard Shortcuts**:
- `Arrow keys`: Navigate cells
- `Enter`: Edit cell
- `Tab`: Next cell
- `Cmd+C`: Copy cell/selection
- `Cmd+F`: Search dataset
- `Cmd+E`: Export
- `P`: Profile dataset

**Screen Reader**:
- "Data grid: 10,234 rows, 12 columns"
- "Column Revenue, sorted ascending"
- "Cell: Row 5, Revenue column, value $45,670"
- "Filter applied: Revenue greater than $50,000, showing 2,341 rows"

---

## 📝 Module 10: Form Builder Studio

### Overview
Drag-and-drop form designer for creating dynamic forms with schema-driven rendering. Features field palette, live preview, validation rules, conditional logic, and form response viewer.

---

### Primary Screens & Layout

#### Main View: Designer + Live Preview Split

```
┌────────────────────────────────────────────────────────────────┐
│ Form: Customer Feedback          [Preview] [Publish] [Code]    │
├──────────────┬─────────────────────────────────┬───────────────┤
│              │                                 │               │
│  Field Types │  Form Designer (Drop Zone)     │ Live Preview  │
│              │                                 │               │
│  📝 Text     │  ┌───────────────────────────┐ │ ┌───────────┐│
│  📧 Email    │  │ Name *                    │ │ │ Name *    ││
│  🔢 Number   │  │ [Text input]              │ │ │ [______]  ││
│  📅 Date     │  ├───────────────────────────┤ │ ├───────────┤│
│  ☑️ Checkbox │  │ Email *                   │ │ │ Email *   ││
│  🔘 Radio    │  │ [Email input]             │ │ │ [______]  ││
│  📋 Select   │  ├───────────────────────────┤ │ ├───────────┤│
│  📝 Textarea │  │ Rating                    │ │ │ Rating    ││
│  📎 File     │  │ [Star rating 1-5]         │ │ │ ⭐⭐⭐⭐⭐││
│  ⭐ Rating   │  ├───────────────────────────┤ │ ├───────────┤│
│              │  │ Comments                  │ │ │ Comments  ││
│              │  │ [Textarea]                │ │ │ [______]  ││
│              │  ├───────────────────────────┤ │ │ [______]  ││
│              │  │ [+ Add Field]             │ │ ├───────────┤│
│              │  └───────────────────────────┘ │ │ [Submit]  ││
│              │                                 │ └───────────┘│
│  [Drag field │  [Drag fields here to build]  │               │
│   to add]    │  [Click field to edit props]  │ [Interactive] │
│              │                                 │               │
└──────────────┴─────────────────────────────────┴───────────────┘
```

#### Field Properties Panel (Right Sidebar)

```
┌────────────────────────────────────────────────────────────────┐
│ Field Properties: Email                                  [✕]    │
├────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Label: [Email Address                  ]                      │
│  Name: [email                           ]                      │
│  Placeholder: [your@email.com           ]                      │
│  Help Text: [We'll never share your email]                     │
│                                                                 │
│  Required: [✓]                                                 │
│  Width: [Full ▾]                                               │
│                                                                 │
│  Validation:                                                    │
│  ☑ Email format                                                │
│  ☐ Custom pattern: [regex]                                     │
│  Error Message: [Please enter a valid email]                   │
│                                                                 │
│  Conditional Logic:                                            │
│  Show this field if:                                           │
│  [Previous Field ▾] [equals ▾] [value]                        │
│  [+ Add Condition]                                             │
│                                                                 │
│  [Delete Field] [Duplicate]                                    │
└────────────────────────────────────────────────────────────────┘
```

---

### Hard UI Features

#### 1. **Drag-and-Drop Form Builder**
- **What**: Drag field from palette → drop in form → field appears
- **Why**: Visual form creation, no code
- **Implementation**: CDK Drag-Drop, form schema model, dynamic component rendering

#### 2. **Live Preview (Real-Time)**
- **What**: Changes in designer instantly reflect in preview (e.g., add field → see in preview)
- **Why**: WYSIWYG editing
- **Implementation**: Shared form schema, reactive rendering, split pane

#### 3. **Field Reordering (Drag Within Form)**
- **What**: Drag field to new position in form
- **Why**: Adjust field order visually
- **Implementation**: CDK Drag-Drop within drop zone, position tracking

#### 4. **Conditional Logic (Show/Hide Fields)**
- **What**: Show field only if condition met (e.g., show "Company" if "Employment Status" = "Employed")
- **Why**: Dynamic forms, reduce clutter
- **Implementation**: Condition evaluator, reactive form rendering, rule engine

#### 5. **Validation Rule Builder**
- **What**: Add validation rules (required, email, min/max length, regex pattern, custom)
- **Why**: Data quality, user guidance
- **Implementation**: Validator service, error message display, validation state tracking

#### 6. **Form Schema Export (JSON/Code)**
- **What**: Export form definition as JSON, Angular template code, or React code
- **Why**: Use in other apps, version control
- **Implementation**: Schema serialization, code generation templates

#### 7. **Form Response Viewer**
- **What**: View submitted form responses in table, filter/search, export
- **Why**: Analyze submissions
- **Implementation**: Response list component, virtual scroll, export service

#### 8. **Field Templates (Pre-Built Fields)**
- **What**: Drag "Address" template → adds Street, City, State, ZIP fields
- **Why**: Common patterns, faster authoring
- **Implementation**: Template definitions, multi-field insertion

#### 9. **Form Sections (Grouping)**
- **What**: Group fields into sections with headers/descriptions
- **Why**: Organize long forms
- **Implementation**: Section container, nested drag-drop, collapsible sections

#### 10. **Calculated Fields**
- **What**: Field value computed from other fields (e.g., Total = Quantity × Price)
- **Why**: Dynamic calculations
- **Implementation**: Expression evaluator, dependency tracking, reactive updates

---

### Component Breakdown

**1. FormBuilderShellComponent**
- **Responsibility**: Main container with palette, designer, preview
- **State**: Form schema, active field, preview mode

**2. FieldPaletteComponent**
- **Responsibility**: List of draggable field types
- **Features**: Field icons, descriptions, drag source
- **ARIA**: `role="list"`, `role="listitem"`
- **Keyboard**: Tab to field, Enter to add to form

**3. FormDesignerComponent**
- **Responsibility**: Drop zone for building form
- **Features**: Drag-drop fields, click to edit, reorder
- **ARIA**: `role="form"`, proper field labels
- **Keyboard**: Arrow keys to navigate fields, Enter to edit

**4. FormPreviewComponent**
- **Responsibility**: Live preview of form
- **Features**: Interactive (can fill out form), validation errors shown
- **ARIA**: Proper form semantics, labels
- **Keyboard**: Standard form navigation

**5. FieldPropertiesComponent**
- **Responsibility**: Edit field properties
- **Features**: Label, placeholder, validation, conditional logic
- **ARIA**: `role="dialog"` or sidebar region
- **Keyboard**: Tab through inputs, Cmd+Enter to save

**6. ConditionalLogicEditorComponent**
- **Responsibility**: Build show/hide conditions
- **Features**: Field selector, operator selector, value input
- **ARIA**: `role="form"`
- **Keyboard**: Tab through controls, Enter to add condition

**7. ValidationRuleEditorComponent**
- **Responsibility**: Add validation rules
- **Features**: Rule type selector, parameters, error messages
- **ARIA**: `role="form"`
- **Keyboard**: Tab through controls

**8. FormResponsesComponent**
- **Responsibility**: View form submissions
- **Features**: Table view, search, export, individual response view
- **ARIA**: `role="table"`
- **Keyboard**: Arrow keys, Enter to view details

---

### State Design

#### FormsStore

```typescript
export interface FormsState {
  // Forms
  forms: FormSchema[];
  activeFormId: string | null;

  // Designer state
  formSchema: FormSchema | null;
  activeFieldId: string | null;

  // Responses
  responses: FormResponse[];

  // Preview
  previewMode: boolean;
  previewData: Record<string, any>;

  // UI
  propertiesPanelOpen: boolean;
  responsesViewOpen: boolean;
  loading: boolean;
  error: string | null;
}

export const FormsStore = signalStore(
  { providedIn: 'root' },

  withState<FormsState>({
    forms: [],
    activeFormId: null,
    formSchema: null,
    activeFieldId: null,
    responses: [],
    previewMode: false,
    previewData: {},
    propertiesPanelOpen: false,
    responsesViewOpen: false,
    loading: false,
    error: null,
  }),

  withComputed((store) => ({
    // Active field
    activeField: computed(() => {
      const fieldId = store.activeFieldId();
      if (!fieldId || !store.formSchema()) return null;

      return store.formSchema()!.fields.find(f => f.id === fieldId);
    }),

    // Visible fields (after conditional logic)
    visibleFields: computed(() => {
      const schema = store.formSchema();
      if (!schema) return [];

      const previewData = store.previewData();

      return schema.fields.filter(field => {
        // Check conditional logic
        const rules = schema.conditionalLogic.filter(r => r.fieldId === field.id);

        if (rules.length === 0) return true; // No conditions = always visible

        // Evaluate all conditions (AND logic)
        return rules.every(rule => {
          const value = previewData[rule.condition.fieldId];
          return this.evaluateCondition(value, rule.condition);
        });
      });
    }),

    // Form is valid (preview mode)
    formIsValid: computed(() => {
      const schema = store.formSchema();
      const data = store.previewData();
      if (!schema) return false;

      // Check all visible required fields have values
      return store.visibleFields().every(field => {
        if (!field.required) return true;
        return data[field.id] != null && data[field.id] !== '';
      });
    }),
  })),

  withMethods((store, formsService = inject(FormsService)) => ({
    // Load form
    async loadForm(formId: string) {
      patchState(store, { loading: true, activeFormId: formId });

      try {
        const form = await formsService.getForm(formId);
        patchState(store, { formSchema: form, loading: false });
      } catch (error) {
        patchState(store, { loading: false, error: error.message });
      }
    },

    // Add field
    addField(fieldType: FormFieldType, position?: number) {
      const newField: FormField = {
        id: generateId(),
        name: `field_${Date.now()}`,
        label: `New ${fieldType} Field`,
        type: fieldType,
        required: false,
        placeholder: '',
        helpText: '',
        options: fieldType === 'select' || fieldType === 'radio' ? [] : undefined,
        width: 'full',
        order: position ?? store.formSchema()!.fields.length,
      };

      patchState(store, (state) => ({
        formSchema: {
          ...state.formSchema!,
          fields: [...state.formSchema!.fields, newField],
        },
        activeFieldId: newField.id,
        propertiesPanelOpen: true,
      }));
    },

    // Update field
    updateField(fieldId: string, updates: Partial<FormField>) {
      patchState(store, (state) => ({
        formSchema: {
          ...state.formSchema!,
          fields: state.formSchema!.fields.map(f =>
            f.id === fieldId ? { ...f, ...updates } : f
          ),
        },
      }));
    },

    // Delete field
    deleteField(fieldId: string) {
      patchState(store, (state) => ({
        formSchema: {
          ...state.formSchema!,
          fields: state.formSchema!.fields.filter(f => f.id !== fieldId),
        },
        activeFieldId: null,
      }));
    },

    // Reorder fields
    reorderFields(fromIndex: number, toIndex: number) {
      patchState(store, (state) => {
        const fields = [...state.formSchema!.fields];
        const [movedField] = fields.splice(fromIndex, 1);
        fields.splice(toIndex, 0, movedField);

        return {
          formSchema: {
            ...state.formSchema!,
            fields: fields.map((f, i) => ({ ...f, order: i })),
          },
        };
      });
    },

    // Add conditional logic
    addConditionalRule(rule: ConditionalRule) {
      patchState(store, (state) => ({
        formSchema: {
          ...state.formSchema!,
          conditionalLogic: [...state.formSchema!.conditionalLogic, rule],
        },
      }));
    },

    // Update preview data (user filling out form)
    updatePreviewData(fieldId: string, value: any) {
      patchState(store, (state) => ({
        previewData: { ...state.previewData, [fieldId]: value },
      }));
    },

    // Submit form (preview mode)
    async submitForm() {
      const response: FormResponse = {
        id: generateId(),
        formId: store.activeFormId()!,
        data: store.previewData(),
        submittedAt: new Date(),
      };

      await formsService.submitResponse(response);

      patchState(store, (state) => ({
        responses: [response, ...state.responses],
        previewData: {},
      }));
    },

    // Export schema
    exportSchema(format: 'json' | 'angular' | 'react'): string {
      const schema = store.formSchema()!;

      if (format === 'json') {
        return JSON.stringify(schema, null, 2);
      } else if (format === 'angular') {
        return this.generateAngularCode(schema);
      } else {
        return this.generateReactCode(schema);
      }
    },

    // Evaluate condition (helper)
    evaluateCondition(value: any, condition: ConditionalRule['condition']): boolean {
      switch (condition.operator) {
        case '==': return value == condition.value;
        case '!=': return value != condition.value;
        case '>': return value > condition.value;
        case '<': return value < condition.value;
        case 'contains': return String(value).includes(condition.value);
        default: return true;
      }
    },

    // Code generation helpers
    generateAngularCode(schema: FormSchema): string {
      // Template generation logic
      return `<form>\n${schema.fields.map(f => `  <mat-form-field>\n    <mat-label>${f.label}</mat-label>\n    <input matInput type="${f.type}" />\n  </mat-form-field>`).join('\n')}\n</form>`;
    },

    generateReactCode(schema: FormSchema): string {
      // React JSX generation
      return `<form>\n${schema.fields.map(f => `  <input type="${f.type}" placeholder="${f.label}" />`).join('\n')}\n</form>`;
    },
  })),
);

interface FormResponse {
  id: string;
  formId: string;
  data: Record<string, any>;
  submittedAt: Date;
}
```

---

### Accessibility

**ARIA Patterns**:
- **Form** (designer & preview): Proper labels, `role="form"`
- **List** (field palette): `role="list"`, `role="listitem"`
- **Dialog** (properties): `role="dialog"`
- **Button** (field drag handles): `aria-label="Drag to reorder"`

**Keyboard Shortcuts**:
- `N`: New field
- `Cmd+D`: Duplicate field
- `Delete`: Delete selected field
- `Cmd+S`: Save form
- `Cmd+P`: Toggle preview
- `Cmd+E`: Export schema

**Screen Reader**:
- "Form designer: 5 fields"
- "Text field: Name, required"
- "Drag to reorder, use arrow keys to navigate"
- "Conditional logic: Show if Previous Field equals value"

---

## ✅ Modules 7-10 Complete

### Module 7: Admin Console
- ✅ 10 hard UI features (user/role matrix, bulk operations, audit log, etc.)
- ✅ 8 components
- ✅ AdminStore with complete user/role/tenant management
- ✅ Accessibility (table, dialog, tree, switch ARIA patterns)

### Module 8: Case Triage
- ✅ 10 hard UI features (virtual scroll, SLA indicators, templates, etc.)
- ✅ 8 components
- ✅ CasesStore with timeline and SLA tracking
- ✅ Accessibility (listbox, feed, textbox, progressbar ARIA patterns)

### Module 9: Dataset Explorer
- ✅ 10 hard UI features (virtual grid 100k+, profiling worker, query builder, etc.)
- ✅ 7 components
- ✅ DatasetsStore with profiling and filtering
- ✅ Accessibility (grid ARIA pattern with full keyboard navigation)

### Module 10: Form Builder
- ✅ 10 hard UI features (drag-drop builder, conditional logic, live preview, etc.)
- ✅ 8 components
- ✅ FormsStore with schema management and code generation
- ✅ Accessibility (form, list, dialog ARIA patterns)

---

**All 10 modules now fully specified!**

**Next**: Phase 7 - Performance & Testing Strategy
