# Workspace Shell: IDE-Like Layout System

> **Complete specification for the Nexus Studio workspace shell—the foundation that makes all 10 modules feel like one cohesive product.**

---

## 🎯 What & Why

### What is the Workspace Shell?

The workspace shell is the **container system** that hosts all modules. Think of it like:
- **VS Code's workbench**: Split panes, tabs, panels, activity bar
- **Browser dev tools**: Resizable, dockable, collapsible panels
- **Figma's canvas**: Flexible layout that adapts to workflow

### Why This Matters

**Without a shell**: Each module would feel like a separate app (jarring transitions, no multitasking, lost context)

**With a shell**:
- Open multiple modules simultaneously (dataset + case + logs in split view)
- Persist workspace state across sessions
- Consistent UX patterns across all modules
- Professional, tool-like feel (not a typical web app)

---

## 🏗️ Shell Architecture Overview

### Visual Structure

```
┌────────────────────────────────────────────────────────────────┐
│ TOP BAR (56px, fixed)                                          │
│ [Logo] [Tenant▾] [Search] [Cmd+K] [Notifications] [Profile]   │
├──────┬─────────────────────────────────────────────────────────┤
│      │ WORKSPACE AREA (flex, fills remaining height)           │
│      │ ┌──────────────────────────────────────────────────────┐│
│      │ │ TAB BAR (40px)                                       ││
│      │ │ [📁 Files] [📊 Dataset] [🎫 Case #2491 *] [+]       ││
│ LEFT │ ├──────────────────────────────────────────────────────┤│
│ NAV  │ │                                                      ││
│      │ │ MAIN CONTENT AREA (flex-grow)                       ││
│ 240px│ │                                                      ││
│ (or  │ │ [Active tab's module component renders here]        ││
│ 64px)│ │                                                      ││
│      │ │                                                      ││
│      │ ├──────────────────────────────────────────────────────┤│
│      │ │ BOTTOM PANEL (resizable, 0-400px, collapsible)      ││
│      │ │ [Inspector / Properties / Terminal]                 ││
│      │ └──────────────────────────────────────────────────────┘│
├──────┴─────────────────────────────────────────────────────────┤
│ STATUS BAR (24px, fixed)                                        │
│ [Activity] [Errors: 0] [Warnings: 2] [Sync Status] [User]      │
└────────────────────────────────────────────────────────────────┘
```

### Component Hierarchy

```typescript
WorkspaceShellComponent                    // Root shell container
├── TopBarComponent                        // Fixed header
│   ├── TenantSwitcherComponent
│   ├── GlobalSearchComponent
│   ├── CommandPaletteButtonComponent
│   ├── NotificationCenterComponent
│   └── UserMenuComponent
├── LeftNavigationComponent                // Collapsible sidebar
│   └── NavigationItemComponent (×11)      // 10 modules + admin
├── WorkspaceAreaComponent                 // Main flex container
│   ├── TabBarComponent                    // Tabbed navigation
│   │   └── TabComponent (×N)              // One per open tab
│   ├── ContentPaneContainerComponent      // Hosts router outlet
│   │   └── <router-outlet>                // Module content renders here
│   └── BottomPanelComponent               // Collapsible inspector
│       └── InspectorContentComponent      // Context-sensitive content
└── StatusBarComponent                     // Fixed footer
```

---

## 📐 Layout System Details

### Grid Layout Strategy

**CSS Grid** for main structure (fixed header/footer, flexible middle):

```scss
.workspace-shell {
  display: grid;
  grid-template-rows: 56px 1fr 24px;  // top-bar | workspace | status-bar
  grid-template-columns: auto 1fr;     // left-nav | content
  height: 100vh;
  overflow: hidden;
}

.top-bar {
  grid-column: 1 / -1;  // Span full width
  grid-row: 1;
}

.left-nav {
  grid-column: 1;
  grid-row: 2;
  width: var(--left-nav-width, 240px);  // CSS variable for collapse
  transition: width 200ms ease;
}

.workspace-area {
  grid-column: 2;
  grid-row: 2;
  display: flex;
  flex-direction: column;
  overflow: hidden;
}

.status-bar {
  grid-column: 1 / -1;
  grid-row: 3;
}
```

### Workspace Area Layout

**Flexbox** for vertical stacking with resizable bottom panel:

```scss
.workspace-area {
  display: flex;
  flex-direction: column;

  .tab-bar {
    flex: 0 0 40px;  // Fixed height
  }

  .content-pane-container {
    flex: 1 1 auto;  // Grows to fill available space
    overflow: hidden;
    position: relative;
  }

  .bottom-panel {
    flex: 0 0 var(--bottom-panel-height, 0px);  // Resizable, default collapsed
    border-top: 1px solid var(--border-color);
    overflow: hidden;
  }

  .resize-handle {
    height: 4px;
    cursor: ns-resize;
    background: transparent;

    &:hover {
      background: var(--primary-color);
    }
  }
}
```

---

## 🗂️ Tabbed Workspace System

### Tab Data Model

```typescript
interface WorkspaceTab {
  // Identity
  id: string;                       // UUID
  type: ModuleType;                 // 'files' | 'kanban' | 'dataset' | ...

  // Display
  title: string;                    // "Customer Dataset"
  icon: string;                     // Material icon name
  tooltip?: string;                 // Full path or description

  // Navigation
  route: string;                    // "/t/acme/datasets/customers"
  routeParams?: Record<string, any>;
  queryParams?: Record<string, any>;
  fragment?: string;

  // State
  state?: TabState;                 // Module-specific serializable state
  scrollPosition?: { x: number; y: number };
  selection?: string[];             // Selected items

  // Metadata
  isPinned: boolean;                // Survives "Close All"
  isDirty: boolean;                 // Has unsaved changes (show *)
  canClose: boolean;                // Allow closing (some tabs forced open)
  timestamp: number;                // Last accessed (for MRU sorting)

  // Split view (future)
  splitGroup?: 'left' | 'right';    // Which split pane this tab belongs to
}

type ModuleType =
  | 'dashboard'
  | 'files'
  | 'design-system'
  | 'media'
  | 'kanban'
  | 'logs'
  | 'knowledge-base'
  | 'cases'
  | 'datasets'
  | 'forms'
  | 'admin';

interface TabState {
  // Module-specific state (examples):
  // Files: { expandedFolders: string[], selectedFiles: string[] }
  // Dataset: { filters: FilterConfig[], sortColumn: string, sortOrder: 'asc'|'desc' }
  // Kanban: { boardId: string, selectedCardId?: string }
  [key: string]: any;
}
```

### Tab Operations

#### Creating Tabs

```typescript
// In AppShellStore (SignalStore)
export const AppShellStore = signalStore(
  { providedIn: 'root' },
  withState<WorkspaceShellState>({
    tabs: [],
    activeTabId: null,
    // ... other state
  }),
  withMethods((store) => ({
    openTab(config: Partial<WorkspaceTab>) {
      const newTab: WorkspaceTab = {
        id: generateId(),
        type: config.type!,
        title: config.title!,
        icon: config.icon || getDefaultIcon(config.type!),
        route: config.route!,
        isPinned: config.isPinned ?? false,
        isDirty: false,
        canClose: config.canClose ?? true,
        timestamp: Date.now(),
        ...config,
      };

      patchState(store, (state) => ({
        tabs: [...state.tabs, newTab],
        activeTabId: newTab.id,
      }));

      // Navigate to the tab's route
      inject(Router).navigateByUrl(newTab.route);
    },

    closeTab(tabId: string) {
      const tab = store.tabs().find(t => t.id === tabId);

      // Check if dirty
      if (tab?.isDirty) {
        // Show confirmation dialog (handled in component)
        return { confirmed: false, tab };
      }

      patchState(store, (state) => {
        const remainingTabs = state.tabs.filter(t => t.id !== tabId);
        const wasActive = state.activeTabId === tabId;

        // If closing active tab, activate the next one
        let newActiveId = state.activeTabId;
        if (wasActive && remainingTabs.length > 0) {
          const closedIndex = state.tabs.findIndex(t => t.id === tabId);
          // Activate tab to the right, or left if last tab
          const nextIndex = Math.min(closedIndex, remainingTabs.length - 1);
          newActiveId = remainingTabs[nextIndex].id;
        }

        return {
          tabs: remainingTabs,
          activeTabId: newActiveId,
        };
      });

      return { confirmed: true };
    },

    activateTab(tabId: string) {
      const tab = store.tabs().find(t => t.id === tabId);
      if (!tab) return;

      patchState(store, {
        activeTabId: tabId,
      });

      // Update timestamp for MRU
      patchState(store, (state) => ({
        tabs: state.tabs.map(t =>
          t.id === tabId
            ? { ...t, timestamp: Date.now() }
            : t
        ),
      }));

      // Navigate
      inject(Router).navigateByUrl(tab.route);
    },

    pinTab(tabId: string) {
      patchState(store, (state) => ({
        tabs: state.tabs.map(t =>
          t.id === tabId ? { ...t, isPinned: !t.isPinned } : t
        ),
      }));
    },

    markTabDirty(tabId: string, isDirty: boolean) {
      patchState(store, (state) => ({
        tabs: state.tabs.map(t =>
          t.id === tabId ? { ...t, isDirty } : t
        ),
      }));
    },

    updateTabState(tabId: string, state: Partial<TabState>) {
      patchState(store, (s) => ({
        tabs: s.tabs.map(t =>
          t.id === tabId
            ? { ...t, state: { ...t.state, ...state } }
            : t
        ),
      }));
    },

    reorderTabs(fromIndex: number, toIndex: number) {
      patchState(store, (state) => {
        const tabs = [...state.tabs];
        const [movedTab] = tabs.splice(fromIndex, 1);
        tabs.splice(toIndex, 0, movedTab);
        return { tabs };
      });
    },
  })),
);
```

### Tab UI Component

```typescript
@Component({
  selector: 'app-tab',
  standalone: true,
  imports: [CommonModule, MatIconModule, MatTooltipModule],
  template: `
    <div
      class="tab"
      [class.active]="isActive()"
      [class.pinned]="tab().isPinned"
      [class.dirty]="tab().isDirty"
      (click)="activate()"
      [matTooltip]="tab().tooltip || tab().route"
    >
      <mat-icon class="tab-icon">{{ tab().icon }}</mat-icon>
      <span class="tab-title">{{ tab().title }}</span>
      @if (tab().isDirty) {
        <span class="dirty-indicator">●</span>
      }
      @if (tab().canClose) {
        <button
          class="close-button"
          (click)="close($event)"
          [attr.aria-label]="'Close ' + tab().title"
        >
          <mat-icon>close</mat-icon>
        </button>
      }
    </div>
  `,
  styles: [`
    .tab {
      display: flex;
      align-items: center;
      gap: 8px;
      padding: 8px 12px;
      border-right: 1px solid var(--border-color);
      cursor: pointer;
      user-select: none;
      background: var(--surface);
      transition: background 150ms;

      &:hover {
        background: var(--surface-hover);
      }

      &.active {
        background: var(--surface-active);
        border-bottom: 2px solid var(--primary-color);
      }

      &.pinned .tab-icon {
        color: var(--primary-color);
      }
    }

    .tab-icon {
      font-size: 16px;
      width: 16px;
      height: 16px;
    }

    .tab-title {
      font-size: 13px;
      white-space: nowrap;
      max-width: 150px;
      overflow: hidden;
      text-overflow: ellipsis;
    }

    .dirty-indicator {
      color: var(--warning-color);
      font-size: 18px;
      line-height: 1;
    }

    .close-button {
      display: none;
      background: transparent;
      border: none;
      padding: 2px;
      cursor: pointer;
      border-radius: 4px;

      &:hover {
        background: var(--error-color-light);
      }

      mat-icon {
        font-size: 14px;
        width: 14px;
        height: 14px;
      }
    }

    .tab:hover .close-button {
      display: flex;
    }
  `],
})
export class TabComponent {
  tab = input.required<WorkspaceTab>();
  isActive = input.required<boolean>();

  private store = inject(AppShellStore);

  activate() {
    this.store.activateTab(this.tab().id);
  }

  close(event: Event) {
    event.stopPropagation();

    if (this.tab().isDirty) {
      // Show confirmation dialog
      const dialogRef = inject(MatDialog).open(ConfirmCloseDialogComponent, {
        data: { tabTitle: this.tab().title },
      });

      dialogRef.afterClosed().subscribe(confirmed => {
        if (confirmed) {
          this.store.closeTab(this.tab().id);
        }
      });
    } else {
      this.store.closeTab(this.tab().id);
    }
  }
}
```

### Tab Bar with Drag-Drop Reordering

```typescript
@Component({
  selector: 'app-tab-bar',
  standalone: true,
  imports: [CommonModule, CdkDrag, CdkDropList, TabComponent],
  template: `
    <div class="tab-bar">
      <div
        class="tab-list"
        cdkDropList
        cdkDropListOrientation="horizontal"
        (cdkDropListDropped)="onTabDrop($event)"
      >
        @for (tab of tabs(); track tab.id) {
          <app-tab
            [tab]="tab"
            [isActive]="tab.id === activeTabId()"
            cdkDrag
          />
        }
      </div>

      <button
        class="new-tab-button"
        (click)="openNewTab()"
        matTooltip="New Tab (Cmd+T)"
      >
        <mat-icon>add</mat-icon>
      </button>

      <button
        class="tab-actions-menu"
        [matMenuTriggerFor]="menu"
        matTooltip="Tab Actions"
      >
        <mat-icon>more_vert</mat-icon>
      </button>

      <mat-menu #menu="matMenu">
        <button mat-menu-item (click)="closeAllTabs()">Close All Tabs</button>
        <button mat-menu-item (click)="closeOtherTabs()">Close Other Tabs</button>
        <button mat-menu-item (click)="closeTabsToRight()">Close Tabs to Right</button>
        <button mat-menu-item (click)="reopenClosedTab()">Reopen Closed Tab</button>
      </mat-menu>
    </div>
  `,
})
export class TabBarComponent {
  private store = inject(AppShellStore);

  tabs = this.store.tabs;
  activeTabId = this.store.activeTabId;

  onTabDrop(event: CdkDragDrop<WorkspaceTab[]>) {
    this.store.reorderTabs(event.previousIndex, event.currentIndex);
  }

  // ... methods for tab actions
}
```

---

## 🔲 Resizable Panels

### Bottom Panel (Inspector)

```typescript
@Component({
  selector: 'app-bottom-panel',
  standalone: true,
  template: `
    <div
      class="resize-handle"
      (mousedown)="startResize($event)"
    ></div>

    <div class="bottom-panel" [style.height.px]="height()">
      @if (height() > 0) {
        <div class="panel-header">
          <span class="panel-title">{{ title() }}</span>
          <button (click)="collapse()">
            <mat-icon>expand_more</mat-icon>
          </button>
        </div>

        <div class="panel-content">
          <ng-content />
        </div>
      }
    </div>
  `,
})
export class BottomPanelComponent {
  height = signal(0);  // 0 = collapsed
  title = input<string>('Inspector');

  private minHeight = 100;
  private maxHeight = 600;
  private defaultHeight = 300;

  startResize(event: MouseEvent) {
    event.preventDefault();

    const startY = event.clientY;
    const startHeight = this.height();

    const onMouseMove = (e: MouseEvent) => {
      const delta = startY - e.clientY;  // Negative because Y grows downward
      const newHeight = Math.max(
        this.minHeight,
        Math.min(this.maxHeight, startHeight + delta)
      );
      this.height.set(newHeight);
    };

    const onMouseUp = () => {
      document.removeEventListener('mousemove', onMouseMove);
      document.removeEventListener('mouseup', onMouseUp);

      // Persist to store
      inject(AppShellStore).setBottomPanelHeight(this.height());
    };

    document.addEventListener('mousemove', onMouseMove);
    document.addEventListener('mouseup', onMouseUp);
  }

  collapse() {
    this.height.set(0);
    inject(AppShellStore).setBottomPanelHeight(0);
  }

  expand() {
    this.height.set(this.defaultHeight);
    inject(AppShellStore).setBottomPanelHeight(this.defaultHeight);
  }
}
```

### Left Navigation Collapse

```typescript
@Component({
  selector: 'app-left-navigation',
  standalone: true,
  template: `
    <nav
      class="left-nav"
      [class.collapsed]="isCollapsed()"
      [style.width.px]="isCollapsed() ? 64 : 240"
    >
      <button
        class="toggle-button"
        (click)="toggle()"
        [matTooltip]="isCollapsed() ? 'Expand Sidebar (Cmd+B)' : 'Collapse Sidebar (Cmd+B)'"
      >
        <mat-icon>{{ isCollapsed() ? 'menu' : 'menu_open' }}</mat-icon>
      </button>

      <div class="nav-items">
        @for (item of navItems; track item.route) {
          <a
            class="nav-item"
            [routerLink]="item.route"
            routerLinkActive="active"
            [matTooltip]="isCollapsed() ? item.label : ''"
            matTooltipPosition="right"
          >
            <mat-icon>{{ item.icon }}</mat-icon>
            @if (!isCollapsed()) {
              <span class="nav-label">{{ item.label }}</span>
            }
          </a>
        }
      </div>
    </nav>
  `,
})
export class LeftNavigationComponent {
  private store = inject(AppShellStore);
  isCollapsed = this.store.leftNavCollapsed;

  navItems = [
    { route: '/t/default/dashboard', icon: 'home', label: 'Dashboard' },
    { route: '/t/default/files', icon: 'folder', label: 'Files' },
    { route: '/t/default/design-system', icon: 'palette', label: 'Design System' },
    // ... rest of navigation items
  ];

  toggle() {
    this.store.toggleLeftNav();
  }
}
```

---

## ⌨️ Keyboard System

### Keymap Strategy

**Three-Level Keyboard System**:

1. **Global Shortcuts**: Work everywhere in the app
2. **Context Shortcuts**: Active when specific module/component has focus
3. **Override Shortcuts**: Module-specific that override globals

### Keyboard Service

```typescript
export interface KeyboardShortcut {
  key: string;                    // "Cmd+K", "Cmd+Shift+F", "Escape"
  description: string;            // "Open Command Palette"
  handler: () => void;
  context?: 'global' | string;    // 'global' | 'files' | 'dataset' | ...
  preventDefault?: boolean;       // Default: true
}

@Injectable({ providedIn: 'root' })
export class KeyboardService {
  private shortcuts = new Map<string, KeyboardShortcut[]>();
  private activeContext = signal<string>('global');

  constructor() {
    // Listen to all keyboard events
    fromEvent<KeyboardEvent>(document, 'keydown')
      .pipe(
        filter(e => this.shouldHandle(e)),
        map(e => this.normalizeKey(e)),
      )
      .subscribe(keyCombo => {
        this.handleShortcut(keyCombo);
      });
  }

  registerShortcut(shortcut: KeyboardShortcut) {
    const context = shortcut.context || 'global';
    const shortcuts = this.shortcuts.get(context) || [];
    shortcuts.push(shortcut);
    this.shortcuts.set(context, shortcuts);
  }

  unregisterShortcut(key: string, context: string = 'global') {
    const shortcuts = this.shortcuts.get(context) || [];
    this.shortcuts.set(
      context,
      shortcuts.filter(s => s.key !== key)
    );
  }

  setContext(context: string) {
    this.activeContext.set(context);
  }

  private handleShortcut(keyCombo: string) {
    // Try context-specific shortcuts first
    const contextShortcuts = this.shortcuts.get(this.activeContext()) || [];
    const contextMatch = contextShortcuts.find(s => s.key === keyCombo);

    if (contextMatch) {
      contextMatch.handler();
      return;
    }

    // Fall back to global shortcuts
    const globalShortcuts = this.shortcuts.get('global') || [];
    const globalMatch = globalShortcuts.find(s => s.key === keyCombo);

    if (globalMatch) {
      globalMatch.handler();
    }
  }

  private normalizeKey(event: KeyboardEvent): string {
    const parts: string[] = [];

    if (event.ctrlKey || event.metaKey) parts.push('Cmd');
    if (event.shiftKey) parts.push('Shift');
    if (event.altKey) parts.push('Alt');

    // Normalize key name
    let key = event.key;
    if (key === ' ') key = 'Space';
    if (key.length === 1) key = key.toUpperCase();

    parts.push(key);

    return parts.join('+');
  }

  private shouldHandle(event: KeyboardEvent): boolean {
    // Don't handle if typing in input/textarea
    const target = event.target as HTMLElement;
    if (target.tagName === 'INPUT' || target.tagName === 'TEXTAREA') {
      return false;
    }

    // Don't handle if contenteditable
    if (target.isContentEditable) {
      return false;
    }

    return true;
  }
}
```

### Global Shortcuts Registration

```typescript
export function provideGlobalKeyboardShortcuts() {
  return {
    provide: APP_INITIALIZER,
    multi: true,
    useFactory: (keyboard: KeyboardService, store: AppShellStore, router: Router) => {
      return () => {
        // Command Palette
        keyboard.registerShortcut({
          key: 'Cmd+K',
          description: 'Open Command Palette',
          handler: () => store.openCommandPalette(),
        });

        // Global Search
        keyboard.registerShortcut({
          key: 'Cmd+Shift+F',
          description: 'Global Search',
          handler: () => store.openGlobalSearch(),
        });

        // Tab Management
        keyboard.registerShortcut({
          key: 'Cmd+T',
          description: 'New Tab',
          handler: () => store.openNewTab(),
        });

        keyboard.registerShortcut({
          key: 'Cmd+W',
          description: 'Close Tab',
          handler: () => store.closeActiveTab(),
        });

        keyboard.registerShortcut({
          key: 'Cmd+Shift+T',
          description: 'Reopen Closed Tab',
          handler: () => store.reopenLastClosedTab(),
        });

        // Tab Navigation
        for (let i = 1; i <= 9; i++) {
          keyboard.registerShortcut({
            key: `Cmd+${i}`,
            description: `Switch to Tab ${i}`,
            handler: () => store.activateTabAtIndex(i - 1),
          });
        }

        // Module Navigation
        keyboard.registerShortcut({
          key: 'Alt+1',
          description: 'Go to Dashboard',
          handler: () => router.navigate(['/t/default/dashboard']),
        });

        // ... Alt+2 through Alt+0 for other modules

        // Panel Toggles
        keyboard.registerShortcut({
          key: 'Cmd+B',
          description: 'Toggle Sidebar',
          handler: () => store.toggleLeftNav(),
        });

        keyboard.registerShortcut({
          key: 'Cmd+J',
          description: 'Toggle Bottom Panel',
          handler: () => store.toggleBottomPanel(),
        });

        // Help
        keyboard.registerShortcut({
          key: 'Cmd+/',
          description: 'Show Keyboard Shortcuts',
          handler: () => store.openKeyboardShortcutsDialog(),
        });

        // Escape
        keyboard.registerShortcut({
          key: 'Escape',
          description: 'Close Overlay / Cancel',
          handler: () => store.closeTopOverlay(),
        });
      };
    },
    deps: [KeyboardService, AppShellStore, Router],
  };
}
```

---

## 🎯 Command Palette

### Command System

```typescript
export interface Command {
  id: string;
  label: string;                  // "Go to Files"
  description?: string;           // "Open the File Manager module"
  icon?: string;                  // Material icon
  keywords?: string[];            // ["explorer", "browse"]
  category: CommandCategory;
  handler: () => void | Promise<void>;
  context?: string;               // Show only in specific contexts
  shortcut?: string;              // "Cmd+Shift+F"
}

export type CommandCategory =
  | 'navigation'      // Go to modules
  | 'action'          // Create, delete, upload
  | 'search'          // Search commands
  | 'settings'        // Preferences, theme
  | 'help';           // Documentation, shortcuts

export interface CommandProvider {
  getCommands(): Command[] | Promise<Command[]>;
  context?: string;  // Module-specific provider
}
```

### Command Palette Component

```typescript
@Component({
  selector: 'app-command-palette',
  standalone: true,
  template: `
    <div class="command-palette-overlay" (click)="close()">
      <div class="command-palette" (click)="$event.stopPropagation()">
        <input
          #searchInput
          class="command-search"
          [(ngModel)]="searchQuery"
          (ngModelChange)="onSearchChange($event)"
          placeholder="Type a command or search..."
          [attr.aria-label]="'Command palette search'"
        />

        <div class="command-list" role="listbox">
          @if (filteredCommands().length === 0) {
            <div class="no-results">No commands found</div>
          } @else {
            @for (category of groupedCommands(); track category.name) {
              <div class="command-category">
                <div class="category-label">{{ category.label }}</div>
                @for (cmd of category.commands; track cmd.id; let idx = $index) {
                  <div
                    class="command-item"
                    [class.selected]="selectedIndex() === idx"
                    (click)="executeCommand(cmd)"
                    (mouseenter)="selectedIndex.set(idx)"
                    role="option"
                    [attr.aria-selected]="selectedIndex() === idx"
                  >
                    @if (cmd.icon) {
                      <mat-icon>{{ cmd.icon }}</mat-icon>
                    }
                    <div class="command-info">
                      <div class="command-label">{{ cmd.label }}</div>
                      @if (cmd.description) {
                        <div class="command-description">{{ cmd.description }}</div>
                      }
                    </div>
                    @if (cmd.shortcut) {
                      <kbd class="command-shortcut">{{ cmd.shortcut }}</kbd>
                    }
                  </div>
                }
              </div>
            }
          }
        </div>

        <div class="command-footer">
          <span>↑↓ Navigate</span>
          <span>Enter Select</span>
          <span>Esc Close</span>
        </div>
      </div>
    </div>
  `,
})
export class CommandPaletteComponent implements OnInit, AfterViewInit {
  @ViewChild('searchInput') searchInput!: ElementRef<HTMLInputElement>;

  searchQuery = signal('');
  selectedIndex = signal(0);

  private commandService = inject(CommandService);
  private allCommands = signal<Command[]>([]);

  filteredCommands = computed(() => {
    const query = this.searchQuery().toLowerCase();
    if (!query) return this.allCommands();

    return this.allCommands().filter(cmd =>
      cmd.label.toLowerCase().includes(query) ||
      cmd.description?.toLowerCase().includes(query) ||
      cmd.keywords?.some(k => k.toLowerCase().includes(query))
    );
  });

  groupedCommands = computed(() => {
    const commands = this.filteredCommands();
    const groups = new Map<CommandCategory, Command[]>();

    commands.forEach(cmd => {
      const existing = groups.get(cmd.category) || [];
      existing.push(cmd);
      groups.set(cmd.category, existing);
    });

    return Array.from(groups.entries()).map(([category, cmds]) => ({
      name: category,
      label: this.getCategoryLabel(category),
      commands: cmds,
    }));
  });

  async ngOnInit() {
    this.allCommands.set(await this.commandService.getAllCommands());
  }

  ngAfterViewInit() {
    // Focus search input
    this.searchInput.nativeElement.focus();

    // Keyboard navigation
    fromEvent<KeyboardEvent>(this.searchInput.nativeElement, 'keydown')
      .subscribe(event => {
        if (event.key === 'ArrowDown') {
          event.preventDefault();
          this.selectedIndex.update(i =>
            Math.min(i + 1, this.filteredCommands().length - 1)
          );
        } else if (event.key === 'ArrowUp') {
          event.preventDefault();
          this.selectedIndex.update(i => Math.max(i - 1, 0));
        } else if (event.key === 'Enter') {
          event.preventDefault();
          const cmd = this.filteredCommands()[this.selectedIndex()];
          if (cmd) this.executeCommand(cmd);
        }
      });
  }

  executeCommand(command: Command) {
    command.handler();
    this.close();
  }

  close() {
    inject(AppShellStore).closeCommandPalette();
  }

  private getCategoryLabel(category: CommandCategory): string {
    const labels: Record<CommandCategory, string> = {
      navigation: 'Navigation',
      action: 'Actions',
      search: 'Search',
      settings: 'Settings',
      help: 'Help',
    };
    return labels[category];
  }
}
```

### Command Service

```typescript
@Injectable({ providedIn: 'root' })
export class CommandService {
  private providers: CommandProvider[] = [];

  registerProvider(provider: CommandProvider) {
    this.providers.push(provider);
  }

  async getAllCommands(): Promise<Command[]> {
    const commandArrays = await Promise.all(
      this.providers.map(p => p.getCommands())
    );

    return commandArrays.flat();
  }

  // Built-in commands
  getBuiltInCommands(): Command[] {
    return [
      {
        id: 'nav.dashboard',
        label: 'Go to Dashboard',
        icon: 'home',
        category: 'navigation',
        keywords: ['home'],
        handler: () => inject(Router).navigate(['/t/default/dashboard']),
      },
      {
        id: 'nav.files',
        label: 'Go to Files',
        icon: 'folder',
        category: 'navigation',
        keywords: ['explorer', 'browse'],
        handler: () => inject(Router).navigate(['/t/default/files']),
      },
      // ... more navigation commands

      {
        id: 'action.new-file',
        label: 'Create New File',
        icon: 'note_add',
        category: 'action',
        handler: () => inject(FileManagerStore).createNewFile(),
      },

      {
        id: 'settings.theme',
        label: 'Change Theme',
        icon: 'palette',
        category: 'settings',
        handler: () => inject(ThemeService).openThemePicker(),
      },

      {
        id: 'help.shortcuts',
        label: 'Keyboard Shortcuts',
        icon: 'keyboard',
        category: 'help',
        shortcut: 'Cmd+/',
        handler: () => inject(AppShellStore).openKeyboardShortcutsDialog(),
      },
    ];
  }
}
```

---

## 🎛️ Focus Management

### Focus Trap in Modals

```typescript
@Directive({
  selector: '[appFocusTrap]',
  standalone: true,
})
export class FocusTrapDirective implements OnInit, OnDestroy {
  private element = inject(ElementRef);
  private firstFocusable: HTMLElement | null = null;
  private lastFocusable: HTMLElement | null = null;
  private previouslyFocused: HTMLElement | null = null;

  ngOnInit() {
    // Store previously focused element
    this.previouslyFocused = document.activeElement as HTMLElement;

    // Find all focusable elements
    this.updateFocusableElements();

    // Focus first element
    setTimeout(() => this.firstFocusable?.focus(), 0);

    // Listen for Tab key
    fromEvent<KeyboardEvent>(this.element.nativeElement, 'keydown')
      .pipe(
        filter(e => e.key === 'Tab'),
        takeUntilDestroyed(),
      )
      .subscribe(event => this.handleTab(event));
  }

  ngOnDestroy() {
    // Restore focus to previously focused element
    this.previouslyFocused?.focus();
  }

  private updateFocusableElements() {
    const focusableSelector =
      'a[href], button, input, textarea, select, [tabindex]:not([tabindex="-1"])';

    const elements = this.element.nativeElement.querySelectorAll(focusableSelector);
    const visible = Array.from(elements).filter(el =>
      (el as HTMLElement).offsetParent !== null
    );

    this.firstFocusable = visible[0] as HTMLElement;
    this.lastFocusable = visible[visible.length - 1] as HTMLElement;
  }

  private handleTab(event: KeyboardEvent) {
    if (event.shiftKey) {
      // Shift+Tab: move backwards
      if (document.activeElement === this.firstFocusable) {
        event.preventDefault();
        this.lastFocusable?.focus();
      }
    } else {
      // Tab: move forwards
      if (document.activeElement === this.lastFocusable) {
        event.preventDefault();
        this.firstFocusable?.focus();
      }
    }
  }
}
```

### Focus Restoration

```typescript
@Injectable({ providedIn: 'root' })
export class FocusService {
  private focusStack: HTMLElement[] = [];

  pushFocus() {
    const activeElement = document.activeElement as HTMLElement;
    if (activeElement) {
      this.focusStack.push(activeElement);
    }
  }

  popFocus() {
    const element = this.focusStack.pop();
    if (element) {
      element.focus();
    }
  }

  saveFocus(key: string) {
    const activeElement = document.activeElement as HTMLElement;
    if (activeElement) {
      sessionStorage.setItem(`focus:${key}`, this.getSelector(activeElement));
    }
  }

  restoreFocus(key: string) {
    const selector = sessionStorage.getItem(`focus:${key}`);
    if (selector) {
      const element = document.querySelector(selector) as HTMLElement;
      element?.focus();
    }
  }

  private getSelector(element: HTMLElement): string {
    if (element.id) return `#${element.id}`;
    // Fallback: generate path
    // (simplified; real implementation would be more robust)
    return element.tagName;
  }
}
```

---

## 💾 Session Persistence

### Session State Model

```typescript
interface WorkspaceSession {
  version: number;                     // Schema version for migrations
  tenantId: string;
  userId: string;
  timestamp: number;

  // Workspace Layout
  layout: {
    leftNavWidth: number;              // 64 (collapsed) or 240 (expanded)
    leftNavCollapsed: boolean;
    bottomPanelHeight: number;         // 0 (collapsed) or 100-600
    rightPanelWidth: number;           // Future: split view
  };

  // Tabs
  tabs: WorkspaceTab[];
  activeTabId: string | null;
  closedTabs: WorkspaceTab[];          // For "Reopen Closed Tab"

  // UI State
  theme: 'light' | 'dark' | 'high-contrast';
  commandPaletteRecent: string[];      // Recent command IDs
  globalSearchRecent: string[];        // Recent search queries
}
```

### Session Store Methods

```typescript
export const AppShellStore = signalStore(
  { providedIn: 'root' },
  withState<WorkspaceShellState>(initialState),
  withMethods((store) => ({
    saveSession() {
      const session: WorkspaceSession = {
        version: 1,
        tenantId: store.currentTenantId(),
        userId: store.currentUserId(),
        timestamp: Date.now(),
        layout: {
          leftNavWidth: store.leftNavCollapsed() ? 64 : 240,
          leftNavCollapsed: store.leftNavCollapsed(),
          bottomPanelHeight: store.bottomPanelHeight(),
          rightPanelWidth: 0,
        },
        tabs: store.tabs(),
        activeTabId: store.activeTabId(),
        closedTabs: store.closedTabs(),
        theme: store.theme(),
        commandPaletteRecent: store.recentCommands(),
        globalSearchRecent: store.recentSearches(),
      };

      const key = `nexus-session:${session.tenantId}:${session.userId}`;
      localStorage.setItem(key, JSON.stringify(session));
    },

    restoreSession(): boolean {
      const tenantId = this.getCurrentTenantId();  // From URL or default
      const userId = this.getCurrentUserId();      // From auth

      const key = `nexus-session:${tenantId}:${userId}`;
      const stored = localStorage.getItem(key);

      if (!stored) return false;

      try {
        const session: WorkspaceSession = JSON.parse(stored);

        // Validate schema version
        if (session.version !== 1) {
          // Run migration if needed
          this.migrateSession(session);
        }

        // Restore state
        patchState(store, {
          leftNavCollapsed: session.layout.leftNavCollapsed,
          bottomPanelHeight: session.layout.bottomPanelHeight,
          tabs: session.tabs,
          activeTabId: session.activeTabId,
          closedTabs: session.closedTabs,
          theme: session.theme,
          recentCommands: session.commandPaletteRecent,
          recentSearches: session.globalSearchRecent,
        });

        return true;
      } catch (error) {
        console.error('Failed to restore session:', error);
        return false;
      }
    },

    clearSession() {
      const tenantId = store.currentTenantId();
      const userId = store.currentUserId();
      const key = `nexus-session:${tenantId}:${userId}`;
      localStorage.removeItem(key);
    },
  })),

  // Auto-save on state changes (debounced)
  withHooks({
    onInit(store) {
      effect(() => {
        // Track all state that should trigger save
        store.tabs();
        store.activeTabId();
        store.leftNavCollapsed();
        store.bottomPanelHeight();

        // Debounce save (don't save on every keystroke)
        setTimeout(() => store.saveSession(), 500);
      });
    },
  }),
);
```

---

## ✅ Workspace Shell Checklist

**Layout System**:
- ✅ CSS Grid for main structure (header, nav, content, footer)
- ✅ Flexbox for vertical stacking in workspace area
- ✅ Resizable bottom panel with drag handle
- ✅ Collapsible left navigation (240px ↔ 64px)

**Tabbed Workspace**:
- ✅ Tab data model with state, pinned, dirty indicators
- ✅ Tab operations: open, close, activate, pin, reorder
- ✅ Drag-drop tab reordering
- ✅ Context menu with tab actions
- ✅ MRU (most recently used) tracking

**Keyboard System**:
- ✅ Three-level shortcuts (global, context, override)
- ✅ Keyboard service with registration API
- ✅ Normalized key combos (Cmd vs Ctrl, cross-platform)
- ✅ Input exclusion (don't interfere with typing)

**Command Palette**:
- ✅ Command model with categories, icons, shortcuts
- ✅ Fuzzy search with keyword matching
- ✅ Grouped results by category
- ✅ Keyboard navigation (↑↓ Enter Esc)
- ✅ Recent commands tracking

**Focus Management**:
- ✅ Focus trap in modals
- ✅ Focus restoration after modal close
- ✅ Focus stack for nested overlays
- ✅ Skip-to-main link for a11y

**Session Persistence**:
- ✅ Save workspace state to localStorage
- ✅ Restore tabs, layout, theme on reload
- ✅ Debounced auto-save (don't thrash storage)
- ✅ Schema versioning for migrations

---

**Next**: STATE_MANAGEMENT.md (Signals + SignalStore patterns)
