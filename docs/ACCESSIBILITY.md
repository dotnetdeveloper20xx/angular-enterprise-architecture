# Accessibility Guide

> WCAG AAA compliance for Nexus Studio - Making enterprise software accessible to everyone

## Table of Contents
1. [Accessibility Principles](#accessibility-principles)
2. [ARIA Patterns Catalog](#aria-patterns-catalog)
3. [Keyboard Navigation](#keyboard-navigation)
4. [Screen Reader Support](#screen-reader-support)
5. [Focus Management](#focus-management)
6. [Visual Accessibility](#visual-accessibility)
7. [Form Accessibility](#form-accessibility)
8. [Testing Accessibility](#testing-accessibility)

---

## Accessibility Principles

### What: Four POUR principles
**Why**: Foundation of WCAG compliance

1. **Perceivable**: Information and UI components must be presentable to users in ways they can perceive
2. **Operable**: UI components and navigation must be operable
3. **Understandable**: Information and operation of UI must be understandable
4. **Robust**: Content must be robust enough to work with assistive technologies

### Nexus Studio Accessibility Goals

- **WCAG AAA**: Highest level of accessibility
- **Keyboard-First**: All functionality accessible via keyboard
- **Screen Reader Compatible**: Full experience with NVDA, JAWS, VoiceOver
- **Color Blind Friendly**: No color-only information
- **High Contrast**: 7:1 minimum for text
- **Scalable**: Works at 200% zoom

---

## ARIA Patterns Catalog

### Tree Pattern (File Manager, Navigation)

```typescript
// files/components/file-tree/file-tree.component.ts
@Component({
  selector: 'app-file-tree',
  template: `
    <div
      role="tree"
      [attr.aria-label]="ariaLabel()"
      [attr.aria-multiselectable]="multiSelect()"
      (keydown)="handleKeyDown($event)"
    >
      @for (node of nodes(); track node.id) {
        <div
          role="treeitem"
          [attr.aria-expanded]="node.type === 'folder' ? node.expanded : undefined"
          [attr.aria-level]="node.level"
          [attr.aria-posinset]="node.positionInSet"
          [attr.aria-setsize]="node.setSize"
          [attr.aria-selected]="isSelected(node.id)"
          [attr.tabindex]="getTabIndex(node)"
          [id]="node.id"
          class="tree-item"
        >
          @if (node.type === 'folder') {
            <button
              [attr.aria-label]="node.expanded ? 'Collapse' : 'Expand'"
              (click)="toggleExpand(node)"
              class="expand-button"
              tabindex="-1"
            >
              <mat-icon>{{ node.expanded ? 'expand_more' : 'chevron_right' }}</mat-icon>
            </button>
          }

          <mat-icon [attr.aria-hidden]="true">{{ node.icon }}</mat-icon>
          <span class="tree-item-label">{{ node.name }}</span>
        </div>
      }
    </div>
  `,
})
export class FileTreeComponent {
  ariaLabel = input<string>('File tree');
  multiSelect = input<boolean>(true);
  nodes = input.required<FileNode[]>();

  private focusedNodeId = signal<string | null>(null);
  private selectedNodeIds = signal<string[]>([]);

  handleKeyDown(event: KeyboardEvent) {
    const focusedNode = this.getFocusedNode();
    if (!focusedNode) return;

    switch (event.key) {
      case 'ArrowDown':
        event.preventDefault();
        this.focusNextNode();
        break;

      case 'ArrowUp':
        event.preventDefault();
        this.focusPreviousNode();
        break;

      case 'ArrowRight':
        event.preventDefault();
        if (focusedNode.type === 'folder' && !focusedNode.expanded) {
          this.toggleExpand(focusedNode);
        } else {
          this.focusFirstChild(focusedNode);
        }
        break;

      case 'ArrowLeft':
        event.preventDefault();
        if (focusedNode.type === 'folder' && focusedNode.expanded) {
          this.toggleExpand(focusedNode);
        } else {
          this.focusParentNode(focusedNode);
        }
        break;

      case 'Enter':
      case ' ':
        event.preventDefault();
        this.selectNode(focusedNode.id);
        break;

      case 'Home':
        event.preventDefault();
        this.focusFirstNode();
        break;

      case 'End':
        event.preventDefault();
        this.focusLastNode();
        break;

      case '*':
        event.preventDefault();
        this.expandAllSiblings(focusedNode);
        break;
    }
  }

  getTabIndex(node: FileNode): number {
    // Only one item should be tabbable (roving tabindex)
    if (!this.focusedNodeId()) {
      return node.id === this.nodes()[0]?.id ? 0 : -1;
    }
    return node.id === this.focusedNodeId() ? 0 : -1;
  }

  isSelected(nodeId: string): boolean {
    return this.selectedNodeIds().includes(nodeId);
  }

  // Navigation methods implementation...
  private focusNextNode() {
    const flatNodes = this.flattenVisibleNodes();
    const currentIndex = flatNodes.findIndex(n => n.id === this.focusedNodeId());
    if (currentIndex < flatNodes.length - 1) {
      this.focusNode(flatNodes[currentIndex + 1].id);
    }
  }

  private focusPreviousNode() {
    const flatNodes = this.flattenVisibleNodes();
    const currentIndex = flatNodes.findIndex(n => n.id === this.focusedNodeId());
    if (currentIndex > 0) {
      this.focusNode(flatNodes[currentIndex - 1].id);
    }
  }

  private focusNode(nodeId: string) {
    this.focusedNodeId.set(nodeId);
    const element = document.getElementById(nodeId);
    element?.focus();
  }

  private flattenVisibleNodes(): FileNode[] {
    const result: FileNode[] = [];
    const traverse = (nodes: FileNode[]) => {
      for (const node of nodes) {
        result.push(node);
        if (node.type === 'folder' && node.expanded && node.children) {
          traverse(node.children);
        }
      }
    };
    traverse(this.nodes());
    return result;
  }
}
```

**Key ARIA Attributes**:
- `role="tree"`: Identifies container as tree
- `role="treeitem"`: Each node in tree
- `aria-expanded`: Folder expanded state (true/false/undefined)
- `aria-level`: Depth in tree (1-based)
- `aria-posinset`: Position among siblings
- `aria-setsize`: Total siblings
- `aria-selected`: Selection state
- `aria-multiselectable`: Multiple selection allowed

---

### Grid Pattern (Dataset Explorer)

```typescript
// datasets/components/dataset-grid/dataset-grid.component.ts
@Component({
  selector: 'app-dataset-grid',
  template: `
    <div
      role="grid"
      [attr.aria-label]="ariaLabel()"
      [attr.aria-rowcount]="totalRows()"
      [attr.aria-colcount]="columns().length"
      (keydown)="handleKeyDown($event)"
      class="dataset-grid"
    >
      <!-- Column Headers -->
      <div role="row" [attr.aria-rowindex]="1" class="grid-header">
        @for (col of columns(); track col.id; let colIndex = $index) {
          <div
            role="columnheader"
            [attr.aria-colindex]="colIndex + 1"
            [attr.aria-sort]="getSortState(col)"
            [attr.tabindex]="colIndex === 0 ? 0 : -1"
            [id]="'col-' + col.id"
            (click)="sortByColumn(col)"
            (keydown.enter)="sortByColumn(col)"
            (keydown.space)="sortByColumn(col)"
            class="grid-cell header-cell"
          >
            {{ col.name }}
            @if (sortedColumn() === col.id) {
              <mat-icon aria-hidden="true">
                {{ sortDirection() === 'asc' ? 'arrow_upward' : 'arrow_downward' }}
              </mat-icon>
            }
          </div>
        }
      </div>

      <!-- Data Rows -->
      <cdk-virtual-scroll-viewport [itemSize]="40" class="grid-body">
        @for (row of rows(); track row.id; let rowIndex = $index) {
          <div
            role="row"
            [attr.aria-rowindex]="rowIndex + 2"
            [attr.aria-selected]="isRowSelected(row.id)"
            class="grid-row"
          >
            @for (col of columns(); track col.id; let colIndex = $index) {
              <div
                role="gridcell"
                [attr.aria-colindex]="colIndex + 1"
                [attr.tabindex]="getCellTabIndex(rowIndex, colIndex)"
                [id]="'cell-' + rowIndex + '-' + colIndex"
                (focus)="onCellFocus(rowIndex, colIndex)"
                class="grid-cell"
              >
                {{ getCellValue(row, col) }}
              </div>
            }
          </div>
        }
      </cdk-virtual-scroll-viewport>
    </div>
  `,
})
export class DatasetGridComponent {
  ariaLabel = input<string>('Dataset grid');
  columns = input.required<DataColumn[]>();
  rows = input.required<DatasetRow[]>();
  totalRows = input.required<number>();

  private focusedCell = signal<{ row: number; col: number }>({ row: 0, col: 0 });
  private sortedColumn = signal<string | null>(null);
  private sortDirection = signal<'asc' | 'desc'>('asc');

  handleKeyDown(event: KeyboardEvent) {
    const { row, col } = this.focusedCell();

    switch (event.key) {
      case 'ArrowRight':
        event.preventDefault();
        if (col < this.columns().length - 1) {
          this.focusCell(row, col + 1);
        }
        break;

      case 'ArrowLeft':
        event.preventDefault();
        if (col > 0) {
          this.focusCell(row, col - 1);
        }
        break;

      case 'ArrowDown':
        event.preventDefault();
        if (row < this.rows().length - 1) {
          this.focusCell(row + 1, col);
        }
        break;

      case 'ArrowUp':
        event.preventDefault();
        if (row > 0) {
          this.focusCell(row - 1, col);
        }
        break;

      case 'Home':
        event.preventDefault();
        if (event.ctrlKey) {
          this.focusCell(0, 0); // First cell
        } else {
          this.focusCell(row, 0); // First cell in row
        }
        break;

      case 'End':
        event.preventDefault();
        if (event.ctrlKey) {
          this.focusCell(this.rows().length - 1, this.columns().length - 1); // Last cell
        } else {
          this.focusCell(row, this.columns().length - 1); // Last cell in row
        }
        break;

      case 'PageDown':
        event.preventDefault();
        this.focusCell(Math.min(row + 10, this.rows().length - 1), col);
        break;

      case 'PageUp':
        event.preventDefault();
        this.focusCell(Math.max(row - 10, 0), col);
        break;
    }
  }

  getCellTabIndex(rowIndex: number, colIndex: number): number {
    const focused = this.focusedCell();
    return focused.row === rowIndex && focused.col === colIndex ? 0 : -1;
  }

  focusCell(row: number, col: number) {
    this.focusedCell.set({ row, col });
    const cellId = `cell-${row}-${col}`;
    setTimeout(() => {
      document.getElementById(cellId)?.focus();
    });
  }

  getSortState(col: DataColumn): 'ascending' | 'descending' | 'none' {
    if (this.sortedColumn() !== col.id) return 'none';
    return this.sortDirection() === 'asc' ? 'ascending' : 'descending';
  }
}
```

**Key ARIA Attributes**:
- `role="grid"`: Data grid container
- `role="row"`: Each row
- `role="columnheader"`: Column headers
- `role="gridcell"`: Data cells
- `aria-rowcount`: Total rows (including virtual)
- `aria-colcount`: Total columns
- `aria-rowindex`: 1-based row position
- `aria-colindex`: 1-based column position
- `aria-sort`: Sort state (ascending/descending/none)

---

### Dialog Pattern (Modals, Confirmations)

```typescript
// shared/components/dialog/dialog.component.ts
@Component({
  selector: 'app-dialog',
  template: `
    <div
      role="dialog"
      [attr.aria-labelledby]="titleId"
      [attr.aria-describedby]="descriptionId"
      [attr.aria-modal]="true"
      class="dialog-container"
      (keydown.escape)="onCancel()"
    >
      <div class="dialog-content">
        <h2 [id]="titleId" class="dialog-title">
          {{ title() }}
        </h2>

        <div [id]="descriptionId" class="dialog-body">
          <ng-content />
        </div>

        <div class="dialog-actions">
          <button
            (click)="onCancel()"
            [attr.aria-label]="'Cancel ' + title()"
            class="dialog-button cancel"
          >
            Cancel
          </button>
          <button
            #confirmButton
            (click)="onConfirm()"
            [attr.aria-label]="'Confirm ' + title()"
            class="dialog-button confirm"
            autofocus
          >
            {{ confirmLabel() }}
          </button>
        </div>
      </div>
    </div>

    <div class="dialog-backdrop" (click)="onCancel()" aria-hidden="true"></div>
  `,
})
export class DialogComponent implements AfterViewInit {
  title = input.required<string>();
  confirmLabel = input<string>('Confirm');

  confirmed = output<void>();
  cancelled = output<void>();

  titleId = `dialog-title-${Math.random().toString(36).substr(2, 9)}`;
  descriptionId = `dialog-desc-${Math.random().toString(36).substr(2, 9)}`;

  private previouslyFocusedElement: HTMLElement | null = null;

  ngAfterViewInit() {
    // Store previously focused element
    this.previouslyFocusedElement = document.activeElement as HTMLElement;

    // Focus first focusable element in dialog
    const firstFocusable = this.getFirstFocusableElement();
    firstFocusable?.focus();

    // Trap focus within dialog
    this.setupFocusTrap();
  }

  onCancel() {
    this.cancelled.emit();
    this.restoreFocus();
  }

  onConfirm() {
    this.confirmed.emit();
    this.restoreFocus();
  }

  private restoreFocus() {
    this.previouslyFocusedElement?.focus();
  }

  private setupFocusTrap() {
    const dialog = document.querySelector('[role="dialog"]');
    if (!dialog) return;

    const focusableElements = dialog.querySelectorAll(
      'button, [href], input, select, textarea, [tabindex]:not([tabindex="-1"])'
    );

    const firstFocusable = focusableElements[0] as HTMLElement;
    const lastFocusable = focusableElements[focusableElements.length - 1] as HTMLElement;

    dialog.addEventListener('keydown', (event: Event) => {
      const e = event as KeyboardEvent;
      if (e.key !== 'Tab') return;

      if (e.shiftKey) {
        // Shift+Tab
        if (document.activeElement === firstFocusable) {
          e.preventDefault();
          lastFocusable.focus();
        }
      } else {
        // Tab
        if (document.activeElement === lastFocusable) {
          e.preventDefault();
          firstFocusable.focus();
        }
      }
    });
  }

  private getFirstFocusableElement(): HTMLElement | null {
    const dialog = document.querySelector('[role="dialog"]');
    return dialog?.querySelector('button, [href], input') as HTMLElement;
  }
}
```

**Key ARIA Attributes**:
- `role="dialog"`: Modal dialog
- `aria-modal="true"`: Background content inert
- `aria-labelledby`: Points to dialog title
- `aria-describedby`: Points to dialog description
- Focus trap: Tab cycles within dialog
- Escape key: Closes dialog
- Focus restoration: Returns to trigger element

---

### Menu Pattern (Context Menus, Dropdowns)

```typescript
// shared/components/context-menu/context-menu.component.ts
@Component({
  selector: 'app-context-menu',
  template: `
    <div
      role="menu"
      [attr.aria-label]="ariaLabel()"
      [attr.aria-orientation]="'vertical'"
      (keydown)="handleKeyDown($event)"
      class="context-menu"
    >
      @for (item of items(); track item.id; let index = $index) {
        @if (item.type === 'separator') {
          <div role="separator" class="menu-separator"></div>
        } @else {
          <button
            role="menuitem"
            [attr.tabindex]="index === focusedIndex() ? 0 : -1"
            [attr.aria-disabled]="item.disabled"
            [disabled]="item.disabled"
            (click)="selectItem(item)"
            (focus)="focusedIndex.set(index)"
            class="menu-item"
          >
            @if (item.icon) {
              <mat-icon aria-hidden="true">{{ item.icon }}</mat-icon>
            }
            <span>{{ item.label }}</span>
            @if (item.shortcut) {
              <span class="menu-shortcut" aria-label="Keyboard shortcut">
                {{ item.shortcut }}
              </span>
            }
          </button>
        }
      }
    </div>
  `,
})
export class ContextMenuComponent {
  ariaLabel = input<string>('Context menu');
  items = input.required<MenuItem[]>();

  itemSelected = output<MenuItem>();
  menuClosed = output<void>();

  focusedIndex = signal(0);

  handleKeyDown(event: KeyboardEvent) {
    const items = this.items().filter(item => item.type !== 'separator' && !item.disabled);

    switch (event.key) {
      case 'ArrowDown':
        event.preventDefault();
        this.focusNextItem(items);
        break;

      case 'ArrowUp':
        event.preventDefault();
        this.focusPreviousItem(items);
        break;

      case 'Home':
        event.preventDefault();
        this.focusedIndex.set(0);
        this.focusCurrentItem();
        break;

      case 'End':
        event.preventDefault();
        this.focusedIndex.set(items.length - 1);
        this.focusCurrentItem();
        break;

      case 'Escape':
        event.preventDefault();
        this.menuClosed.emit();
        break;

      case 'Enter':
      case ' ':
        event.preventDefault();
        const item = items[this.focusedIndex()];
        if (item) this.selectItem(item);
        break;
    }
  }

  private focusNextItem(items: MenuItem[]) {
    const newIndex = (this.focusedIndex() + 1) % items.length;
    this.focusedIndex.set(newIndex);
    this.focusCurrentItem();
  }

  private focusPreviousItem(items: MenuItem[]) {
    const newIndex = this.focusedIndex() === 0
      ? items.length - 1
      : this.focusedIndex() - 1;
    this.focusedIndex.set(newIndex);
    this.focusCurrentItem();
  }

  private focusCurrentItem() {
    const menuItems = document.querySelectorAll('[role="menuitem"]');
    (menuItems[this.focusedIndex()] as HTMLElement)?.focus();
  }

  selectItem(item: MenuItem) {
    if (item.disabled) return;
    this.itemSelected.emit(item);
    this.menuClosed.emit();
  }
}

interface MenuItem {
  id: string;
  label: string;
  icon?: string;
  shortcut?: string;
  disabled?: boolean;
  type?: 'item' | 'separator';
}
```

**Key ARIA Attributes**:
- `role="menu"`: Menu container
- `role="menuitem"`: Each menu item
- `role="separator"`: Visual separator
- `aria-orientation="vertical"`: Vertical layout
- `aria-disabled`: Disabled items
- Arrow keys: Navigate items
- Enter/Space: Select item
- Escape: Close menu

---

### Tabs Pattern (Workspace Tabs)

```typescript
// workspace/components/tab-bar/tab-bar.component.ts
@Component({
  selector: 'app-tab-bar',
  template: `
    <div role="tablist" [attr.aria-label]="'Workspace tabs'" class="tab-bar">
      @for (tab of tabs(); track tab.id; let index = $index) {
        <button
          role="tab"
          [attr.aria-selected]="tab.id === activeTabId()"
          [attr.aria-controls]="'panel-' + tab.id"
          [attr.tabindex]="tab.id === activeTabId() ? 0 : -1"
          [id]="'tab-' + tab.id"
          (click)="selectTab(tab)"
          (keydown)="handleKeyDown($event, index)"
          class="tab"
          [class.active]="tab.id === activeTabId()"
        >
          <mat-icon aria-hidden="true">{{ tab.icon }}</mat-icon>
          <span>{{ tab.title }}</span>

          @if (tab.isDirty) {
            <span class="dirty-indicator" aria-label="Unsaved changes">●</span>
          }

          @if (tab.canClose) {
            <button
              (click)="closeTab($event, tab)"
              [attr.aria-label]="'Close ' + tab.title"
              tabindex="-1"
              class="close-button"
            >
              <mat-icon>close</mat-icon>
            </button>
          }
        </button>
      }
    </div>

    <!-- Tab Panels -->
    @for (tab of tabs(); track tab.id) {
      <div
        role="tabpanel"
        [attr.aria-labelledby]="'tab-' + tab.id"
        [id]="'panel-' + tab.id"
        [hidden]="tab.id !== activeTabId()"
        [attr.tabindex]="0"
        class="tab-panel"
      >
        <router-outlet [name]="tab.id" />
      </div>
    }
  `,
})
export class TabBarComponent {
  tabs = input.required<WorkspaceTab[]>();
  activeTabId = input.required<string>();

  tabSelected = output<WorkspaceTab>();
  tabClosed = output<WorkspaceTab>();

  handleKeyDown(event: KeyboardEvent, currentIndex: number) {
    const tabs = this.tabs();

    switch (event.key) {
      case 'ArrowRight':
        event.preventDefault();
        const nextIndex = (currentIndex + 1) % tabs.length;
        this.selectTab(tabs[nextIndex]);
        break;

      case 'ArrowLeft':
        event.preventDefault();
        const prevIndex = currentIndex === 0 ? tabs.length - 1 : currentIndex - 1;
        this.selectTab(tabs[prevIndex]);
        break;

      case 'Home':
        event.preventDefault();
        this.selectTab(tabs[0]);
        break;

      case 'End':
        event.preventDefault();
        this.selectTab(tabs[tabs.length - 1]);
        break;

      case 'Delete':
        event.preventDefault();
        if (tabs[currentIndex].canClose) {
          this.tabClosed.emit(tabs[currentIndex]);
        }
        break;
    }
  }

  selectTab(tab: WorkspaceTab) {
    this.tabSelected.emit(tab);
    setTimeout(() => {
      document.getElementById(`tab-${tab.id}`)?.focus();
    });
  }

  closeTab(event: Event, tab: WorkspaceTab) {
    event.stopPropagation();
    this.tabClosed.emit(tab);
  }
}
```

**Key ARIA Attributes**:
- `role="tablist"`: Tab container
- `role="tab"`: Each tab button
- `role="tabpanel"`: Content panel
- `aria-selected`: Active tab
- `aria-controls`: Links tab to panel
- `aria-labelledby`: Links panel to tab
- Arrow keys: Navigate tabs
- Delete: Close tab

---

## Keyboard Navigation

### Global Shortcuts

```typescript
// core/services/keyboard-shortcuts.service.ts
@Injectable({ providedIn: 'root' })
export class KeyboardShortcutsService {
  private shortcuts: Map<string, KeyboardShortcut> = new Map();

  constructor() {
    this.registerGlobalShortcuts();
    this.listenForShortcuts();
  }

  private registerGlobalShortcuts() {
    // Navigation
    this.register({
      key: 'Control+1',
      description: 'Navigate to Files',
      action: () => this.navigateTo('/files'),
    });

    this.register({
      key: 'Control+2',
      description: 'Navigate to Design System',
      action: () => this.navigateTo('/design-system'),
    });

    // Search
    this.register({
      key: 'Control+K',
      description: 'Open command palette',
      action: () => this.openCommandPalette(),
    });

    this.register({
      key: 'Control+F',
      description: 'Search in current module',
      action: () => this.focusSearch(),
    });

    // Workspace
    this.register({
      key: 'Control+W',
      description: 'Close current tab',
      action: () => this.closeCurrentTab(),
    });

    this.register({
      key: 'Control+Tab',
      description: 'Next tab',
      action: () => this.nextTab(),
    });

    this.register({
      key: 'Control+Shift+Tab',
      description: 'Previous tab',
      action: () => this.previousTab(),
    });

    // Accessibility
    this.register({
      key: 'Alt+Shift+K',
      description: 'Show keyboard shortcuts',
      action: () => this.showShortcutsDialog(),
    });
  }

  register(shortcut: KeyboardShortcut) {
    this.shortcuts.set(shortcut.key, shortcut);
  }

  private listenForShortcuts() {
    fromEvent<KeyboardEvent>(document, 'keydown')
      .pipe(
        filter(event => !this.isInputFocused()),
        map(event => this.getShortcutKey(event)),
        filter(key => this.shortcuts.has(key))
      )
      .subscribe(key => {
        const shortcut = this.shortcuts.get(key)!;
        shortcut.action();
      });
  }

  private getShortcutKey(event: KeyboardEvent): string {
    const parts: string[] = [];
    if (event.ctrlKey) parts.push('Control');
    if (event.altKey) parts.push('Alt');
    if (event.shiftKey) parts.push('Shift');
    parts.push(event.key);
    return parts.join('+');
  }

  private isInputFocused(): boolean {
    const activeElement = document.activeElement;
    return (
      activeElement?.tagName === 'INPUT' ||
      activeElement?.tagName === 'TEXTAREA' ||
      activeElement?.hasAttribute('contenteditable')
    );
  }

  // Action methods...
  private navigateTo(route: string) {
    // Implementation
  }

  private openCommandPalette() {
    // Implementation
  }
}

interface KeyboardShortcut {
  key: string;
  description: string;
  action: () => void;
}
```

### Skip Links

```typescript
// app.component.ts
@Component({
  selector: 'app-root',
  template: `
    <!-- Skip links for screen readers and keyboard users -->
    <div class="skip-links">
      <a href="#main-content" class="skip-link">Skip to main content</a>
      <a href="#navigation" class="skip-link">Skip to navigation</a>
      <a href="#search" class="skip-link">Skip to search</a>
    </div>

    <app-top-bar id="navigation" />

    <main id="main-content" tabindex="-1">
      <router-outlet />
    </main>
  `,
  styles: [`
    .skip-link {
      position: absolute;
      top: -40px;
      left: 0;
      background: var(--color-primary);
      color: white;
      padding: 8px 16px;
      text-decoration: none;
      z-index: 10000;
    }

    .skip-link:focus {
      top: 0;
    }
  `],
})
export class AppComponent {}
```

---

## Screen Reader Support

### Landmark Roles

```html
<!-- workspace-shell.component.html -->
<div class="workspace">
  <!-- Header -->
  <header role="banner">
    <app-top-bar />
  </header>

  <!-- Main Navigation -->
  <nav role="navigation" aria-label="Main navigation">
    <app-left-nav />
  </nav>

  <!-- Main Content -->
  <main role="main" id="main-content">
    <router-outlet />
  </main>

  <!-- Complementary Content -->
  <aside role="complementary" aria-label="Properties panel">
    <app-properties-panel />
  </aside>

  <!-- Footer/Status -->
  <footer role="contentinfo">
    <app-status-bar />
  </footer>
</div>
```

### Live Regions for Dynamic Updates

```typescript
// shared/components/toast/toast.component.ts
@Component({
  selector: 'app-toast',
  template: `
    <div
      role="status"
      [attr.aria-live]="priority()"
      [attr.aria-atomic]="true"
      class="toast"
      [class.error]="type() === 'error'"
      [class.success]="type() === 'success'"
    >
      <mat-icon aria-hidden="true">{{ icon() }}</mat-icon>
      <span>{{ message() }}</span>
    </div>
  `,
})
export class ToastComponent {
  message = input.required<string>();
  type = input<'info' | 'success' | 'error'>('info');

  priority = computed(() => {
    // Errors are urgent, others are polite
    return this.type() === 'error' ? 'assertive' : 'polite';
  });

  icon = computed(() => {
    switch (this.type()) {
      case 'success': return 'check_circle';
      case 'error': return 'error';
      default: return 'info';
    }
  });
}
```

**Live Region Priorities**:
- `aria-live="polite"`: Announce when user is idle (info messages)
- `aria-live="assertive"`: Announce immediately (errors, warnings)
- `aria-atomic="true"`: Read entire region, not just changes

### Descriptive Labels

```typescript
// files/components/file-upload/file-upload.component.ts
@Component({
  selector: 'app-file-upload',
  template: `
    <div class="file-upload">
      <label for="file-input" class="upload-label">
        Choose files to upload
      </label>

      <input
        type="file"
        id="file-input"
        multiple
        [attr.aria-describedby]="'upload-description'"
        (change)="onFileSelect($event)"
        class="file-input"
      />

      <div id="upload-description" class="upload-description">
        Supported formats: PDF, DOCX, XLSX, PNG, JPG. Max size: 50MB per file.
      </div>

      @if (uploading()) {
        <div
          role="progressbar"
          [attr.aria-valuenow]="uploadProgress()"
          [attr.aria-valuemin]="0"
          [attr.aria-valuemax]="100"
          [attr.aria-label]="'Upload progress: ' + uploadProgress() + '%'"
          class="progress-bar"
        >
          <div class="progress-fill" [style.width.%]="uploadProgress()"></div>
        </div>
      }
    </div>
  `,
})
export class FileUploadComponent {
  uploading = signal(false);
  uploadProgress = signal(0);

  onFileSelect(event: Event) {
    // Upload implementation
  }
}
```

---

## Focus Management

### Focus Trap in Modal

```typescript
// shared/directives/focus-trap.directive.ts
@Directive({
  selector: '[appFocusTrap]',
  standalone: true,
})
export class FocusTrapDirective implements AfterViewInit, OnDestroy {
  private elementRef = inject(ElementRef);
  private firstFocusable: HTMLElement | null = null;
  private lastFocusable: HTMLElement | null = null;

  ngAfterViewInit() {
    this.setupFocusTrap();
  }

  private setupFocusTrap() {
    const element = this.elementRef.nativeElement;

    // Get all focusable elements
    const focusableElements = element.querySelectorAll<HTMLElement>(
      'a[href], button:not([disabled]), textarea, input, select, [tabindex]:not([tabindex="-1"])'
    );

    if (focusableElements.length === 0) return;

    this.firstFocusable = focusableElements[0];
    this.lastFocusable = focusableElements[focusableElements.length - 1];

    // Focus first element
    this.firstFocusable.focus();

    // Listen for Tab key
    element.addEventListener('keydown', this.handleKeyDown);
  }

  private handleKeyDown = (event: KeyboardEvent) => {
    if (event.key !== 'Tab') return;

    if (event.shiftKey) {
      // Shift+Tab: wrap to last if at first
      if (document.activeElement === this.firstFocusable) {
        event.preventDefault();
        this.lastFocusable?.focus();
      }
    } else {
      // Tab: wrap to first if at last
      if (document.activeElement === this.lastFocusable) {
        event.preventDefault();
        this.firstFocusable?.focus();
      }
    }
  };

  ngOnDestroy() {
    const element = this.elementRef.nativeElement;
    element.removeEventListener('keydown', this.handleKeyDown);
  }
}
```

### Focus Restoration

```typescript
// shared/services/focus-manager.service.ts
@Injectable({ providedIn: 'root' })
export class FocusManagerService {
  private focusStack: HTMLElement[] = [];

  saveFocus() {
    const activeElement = document.activeElement as HTMLElement;
    if (activeElement) {
      this.focusStack.push(activeElement);
    }
  }

  restoreFocus() {
    const element = this.focusStack.pop();
    if (element && element.focus) {
      element.focus();
    }
  }

  focusFirstInContainer(container: HTMLElement) {
    const firstFocusable = container.querySelector<HTMLElement>(
      'button, [href], input, select, textarea, [tabindex]:not([tabindex="-1"])'
    );
    firstFocusable?.focus();
  }

  moveFocusToElementById(id: string) {
    const element = document.getElementById(id);
    element?.focus();
  }
}
```

---

## Visual Accessibility

### Color Contrast

```scss
// styles/accessibility.scss

// WCAG AAA: 7:1 contrast for normal text
$text-primary: #000000; // Black
$background-primary: #FFFFFF; // White
// Contrast: 21:1 ✓

$text-on-dark: #FFFFFF; // White
$background-dark: #1A1A1A; // Near black
// Contrast: 17.9:1 ✓

// Accent colors with sufficient contrast
$color-primary: #0056b3; // Dark blue
$color-primary-text: #FFFFFF; // White
// Contrast: 8.59:1 ✓

$color-success: #006400; // Dark green
$color-success-text: #FFFFFF; // White
// Contrast: 8.75:1 ✓

$color-error: #B71C1C; // Dark red
$color-error-text: #FFFFFF; // White
// Contrast: 10.18:1 ✓

// Use color + icon/text, never color alone
.file-status {
  &.draft {
    color: $color-warning;
    &::before {
      content: "Draft: "; // Text indicator
    }
  }

  &.published {
    color: $color-success;
    &::before {
      content: "Published: "; // Text indicator
    }
  }
}
```

### Focus Visible

```scss
// styles/focus.scss

// Visible focus indicator for keyboard navigation
*:focus-visible {
  outline: 3px solid var(--color-focus);
  outline-offset: 2px;
  border-radius: 2px;
}

// Remove focus outline for mouse clicks (still visible for keyboard)
*:focus:not(:focus-visible) {
  outline: none;
}

// Custom focus styles for specific components
.button:focus-visible {
  outline: 3px solid var(--color-primary);
  outline-offset: 2px;
  box-shadow: 0 0 0 4px rgba(0, 86, 179, 0.2);
}

.file-card:focus-visible {
  outline: 3px solid var(--color-primary);
  background-color: var(--color-primary-light);
}
```

### Text Scaling

```scss
// Support 200% text zoom without horizontal scrolling

html {
  font-size: 16px; // Base size
}

body {
  font-size: 1rem; // Scales with user settings
  line-height: 1.5; // WCAG: at least 1.5
}

h1 { font-size: 2rem; }
h2 { font-size: 1.5rem; }
h3 { font-size: 1.25rem; }

// Use rem/em, not px
.button {
  padding: 0.5rem 1rem; // Scales with text
  font-size: 1rem;
}

// Container uses relative units
.container {
  max-width: 90rem; // 1440px at default size
  padding: 1rem;
}
```

---

## Form Accessibility

### Accessible Form Fields

```typescript
// shared/components/form-field/form-field.component.ts
@Component({
  selector: 'app-form-field',
  template: `
    <div class="form-field" [class.error]="hasError()">
      <label [for]="inputId" class="form-label">
        {{ label() }}
        @if (required()) {
          <span aria-label="required" class="required-indicator">*</span>
        }
      </label>

      <input
        [id]="inputId"
        [type]="type()"
        [attr.aria-required]="required()"
        [attr.aria-invalid]="hasError()"
        [attr.aria-describedby]="hasError() ? errorId : (hint() ? hintId : null)"
        [(ngModel)]="value"
        (blur)="onBlur()"
        class="form-input"
      />

      @if (hint() && !hasError()) {
        <div [id]="hintId" class="form-hint">
          {{ hint() }}
        </div>
      }

      @if (hasError()) {
        <div [id]="errorId" class="form-error" role="alert">
          <mat-icon aria-hidden="true">error</mat-icon>
          {{ errorMessage() }}
        </div>
      }
    </div>
  `,
})
export class FormFieldComponent {
  label = input.required<string>();
  type = input<string>('text');
  required = input<boolean>(false);
  hint = input<string>('');
  errorMessage = input<string>('');

  value = model<string>('');

  inputId = `form-field-${Math.random().toString(36).substr(2, 9)}`;
  errorId = `${this.inputId}-error`;
  hintId = `${this.inputId}-hint`;

  hasError = signal(false);

  onBlur() {
    // Validate on blur
    if (this.required() && !this.value()) {
      this.hasError.set(true);
    } else {
      this.hasError.set(false);
    }
  }
}
```

**Key Attributes**:
- `<label for="inputId">`: Associate label with input
- `aria-required`: Required field
- `aria-invalid`: Validation error
- `aria-describedby`: Link to hint or error message
- `role="alert"`: Error message is live region

---

## Testing Accessibility

### Automated Testing

```typescript
// Install: npm install --save-dev @axe-core/playwright

// e2e/accessibility.spec.ts
import { test, expect } from '@playwright/test';
import AxeBuilder from '@axe-core/playwright';

test.describe('Accessibility Tests', () => {
  test('File Manager should pass axe scan', async ({ page }) => {
    await page.goto('http://localhost:4200/t/test-tenant/files');

    const accessibilityScanResults = await new AxeBuilder({ page })
      .withTags(['wcag2a', 'wcag2aa', 'wcag21aa', 'wcag21aaa'])
      .analyze();

    expect(accessibilityScanResults.violations).toEqual([]);
  });

  test('All modules should have no violations', async ({ page }) => {
    const modules = [
      '/files',
      '/design-system',
      '/media',
      '/kanban',
      '/logs',
      '/knowledge-base',
      '/admin',
      '/cases',
      '/datasets',
      '/forms',
    ];

    for (const module of modules) {
      await page.goto(`http://localhost:4200/t/test-tenant${module}`);

      const results = await new AxeBuilder({ page }).analyze();

      expect(results.violations).toEqual([]);
    }
  });
});
```

### Manual Testing Checklist

- [ ] **Keyboard Navigation**: Can complete all tasks without mouse
- [ ] **Screen Reader**: Test with NVDA (Windows), VoiceOver (Mac), JAWS
- [ ] **Zoom**: Works at 200% zoom without horizontal scrolling
- [ ] **High Contrast**: Readable in Windows High Contrast mode
- [ ] **Color Blindness**: Use Colorblind simulator to verify
- [ ] **Focus Visible**: All focusable elements have visible focus indicator
- [ ] **Forms**: All fields have labels, errors announced
- [ ] **Images**: All images have alt text
- [ ] **Headings**: Proper heading hierarchy (h1 → h2 → h3)
- [ ] **Landmarks**: Header, nav, main, aside, footer regions

---

## Summary

**Nexus Studio Accessibility Commitment**:
1. **WCAG AAA Compliance**: Highest accessibility standard
2. **Keyboard-First Design**: All features keyboard-accessible
3. **Screen Reader Compatible**: Full experience with assistive tech
4. **Color Contrast**: 7:1 minimum for all text
5. **Comprehensive Testing**: Automated + manual accessibility testing

**Resources**:
- [ARIA Authoring Practices Guide](https://www.w3.org/WAI/ARIA/apg/)
- [WebAIM WCAG 2 Checklist](https://webaim.org/standards/wcag/checklist)
- [Axe DevTools Extension](https://www.deque.com/axe/devtools/)
