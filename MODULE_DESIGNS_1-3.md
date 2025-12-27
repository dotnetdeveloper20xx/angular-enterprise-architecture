# Module Designs: Files, Design System, Media Library

> **Detailed specifications for modules 1-3 with implementation-ready designs**

---

## 📁 Module 1: File Manager + Previewer

### Overview
Central file repository for all document types (PDFs, docs, text files, archives). Provides VS Code-like file explorer with multi-select, drag-drop, quick preview, and search.

---

### Primary Screens & Layout

#### Main View: Split Three-Pane Layout

```
┌────────────────────────────────────────────────────────────────┐
│ File Manager                                          [⚙️] [🔍] │
├──────────┬────────────────────────────────┬────────────────────┤
│          │                                │                    │
│  Tree    │   File List (Grid/List View)  │    Preview Pane    │
│  View    │                                │                    │
│          │  ┌──────────────────────────┐  │  ┌──────────────┐  │
│  📁 Root │  │ 📄 report.pdf            │  │  │              │  │
│  ├ 📁 Pr │  │ 📊 data.xlsx             │  │  │  [Preview]   │  │
│  │ └ 📁  │  │ 📝 notes.txt             │  │  │   of         │  │
│  ├ 📁 20 │  │ 🖼️ screenshot.png        │  │  │  selected    │  │
│  │ ├ Q1  │  │ 📦 archive.zip           │  │  │   file       │  │
│  │ └ Q2  │  │                          │  │  │              │  │
│  └ 📁 Do │  │ [Virtual scroll for      │  │  │              │  │
│          │  │  10k+ files]             │  │  │              │  │
│ [250px]  │  │                          │  │  │              │  │
│          │  └──────────────────────────┘  │  └──────────────┘  │
│          │                                │                    │
│          │  [Breadcrumbs: Root > Projects │  [File metadata]   │
│          │   > 2024]                      │  Size: 2.3 MB      │
│          │                                │  Modified: 2h ago  │
│          │  Status: 1,234 items | 42 sel │  Type: PDF         │
└──────────┴────────────────────────────────┴────────────────────┘
```

**Pane Behavior**:
- **Left (Tree)**: Collapsible folder hierarchy, 200-300px width, resizable
- **Center (List)**: Main file list, takes remaining space, virtual scrolling
- **Right (Preview)**: Collapsible preview pane, 300-500px, can be closed

---

### Hard UI Features

#### 1. **Virtual Tree with 50k+ Nodes**
- **What**: Tree view that renders only visible nodes (CDK Virtual Scroll for trees)
- **Why**: File systems can have deep hierarchies with thousands of folders
- **Implementation**: Custom virtual scrolling strategy that tracks expanded/collapsed state

#### 2. **Multi-Select with Keyboard Navigation**
- **What**: Select multiple files with Cmd+Click, Shift+Click, Cmd+A, arrow keys
- **Why**: Batch operations (move, delete, download)
- **Implementation**: Selection model service tracking selected IDs, keyboard event handlers

#### 3. **Drag-and-Drop File Upload**
- **What**: Drag files from desktop → drop on folder in tree or list → upload
- **Why**: Intuitive file upload, no clunky file picker dialog
- **Implementation**: CDK Drag-Drop with file input, FileReader API, progress tracking

#### 4. **Drag-and-Drop File Move**
- **What**: Drag files between folders within the app
- **Why**: Organize files quickly
- **Implementation**: CDK Drag-Drop with drop validation, optimistic updates

#### 5. **Quick Preview (Spacebar)**
- **What**: Press spacebar on selected file → preview modal opens (macOS Quick Look style)
- **Why**: Quickly view file contents without opening in new tab
- **Implementation**: Modal with file-type-specific renderers (PDF.js, image, text, etc.)

#### 6. **Contextual Actions (Right-Click Menu)**
- **What**: Right-click file/folder → context menu with actions (rename, delete, share, etc.)
- **Why**: Discoverability of actions, familiar UX
- **Implementation**: CDK Overlay positioned at mouse coords, dynamic menu items based on permissions

#### 7. **Inline Rename (F2)**
- **What**: Press F2 on selected file → name becomes editable input
- **Why**: Fast renaming without modal
- **Implementation**: Toggle between text display and input, focus management, validation

#### 8. **Bulk Upload with Progress**
- **What**: Upload multiple files simultaneously, see progress per file + total
- **Why**: Uploading 100+ files one-by-one is painful
- **Implementation**: Queue manager, parallel uploads (max 3 concurrent), progress bars

#### 9. **Virtual Scrolling for 10k+ Files**
- **What**: Render only visible files in list, smooth scrolling for massive directories
- **Why**: Directories can have thousands of files (e.g., logs, exports)
- **Implementation**: CDK Virtual Scroll, item size estimation, scroll anchoring

#### 10. **Advanced Search & Filters**
- **What**: Filter by file type, date range, size, tags; search file names and contents
- **Why**: Find files quickly in large repositories
- **Implementation**: Filter panel with chips, debounced search, backend query

#### 11. **File Versioning UI**
- **What**: View previous versions of a file, restore, compare
- **Why**: Track changes, rollback mistakes
- **Implementation**: Version timeline component, diff viewer

#### 12. **Breadcrumb Navigation with Dropdown**
- **What**: Breadcrumbs show current path, click any segment to jump, dropdown to see siblings
- **Why**: Quick navigation without collapsing tree
- **Implementation**: Breadcrumb component with menu trigger on each segment

---

### Component Breakdown

#### Core Components

**1. FileManagerShellComponent**
- **Responsibility**: Main container, orchestrates three panes
- **State**: Layout config (pane widths), active view mode
- **Template**:
```html
<div class="file-manager-shell">
  <app-file-tree
    [treeData]="store.fileTree()"
    [selectedId]="store.currentFolderId()"
    (folderSelected)="store.navigateToFolder($event)"
  />

  <app-file-list
    [files]="store.currentFolderFiles()"
    [viewMode]="viewMode()"
    [selection]="store.selectedFileIds()"
    (selectionChange)="store.setSelection($event)"
    (fileOpen)="openFile($event)"
  />

  <app-file-preview
    [file]="store.previewFile()"
    (close)="store.closePreview()"
  />
</div>
```

**2. FileTreeComponent**
- **Responsibility**: Hierarchical tree view with expand/collapse
- **Features**: Virtual scrolling, lazy loading, drag-drop target
- **ARIA**: `role="tree"`, `role="treeitem"`, `aria-expanded`, `aria-level`
- **Keyboard**: Arrow keys (↑↓←→), Enter to expand, Space to select

**3. FileListComponent**
- **Responsibility**: Grid or list view of files
- **Features**: Virtual scroll, multi-select, drag-drop, context menu
- **View Modes**: Grid (thumbnails), List (table), Compact
- **ARIA**: `role="grid"`, `role="gridcell"` for grid view; `role="listbox"` for list
- **Keyboard**: Arrow keys, Space/Enter to select, Cmd+A for all

**4. FilePreviewComponent**
- **Responsibility**: Render file preview based on type
- **Supported Types**:
  - **PDF**: PDF.js viewer with zoom, search, page navigation
  - **Image**: Image with zoom, pan
  - **Text/Code**: Syntax-highlighted code viewer (Monaco or Prism)
  - **Video/Audio**: HTML5 player
  - **Unknown**: Download button + file icon
- **ARIA**: `role="document"` or `role="img"` depending on type
- **Keyboard**: Esc to close, arrow keys for pagination (PDFs)

**5. FileUploadDialogComponent**
- **Responsibility**: Drag-drop or file picker upload
- **Features**: Multiple files, progress bars, cancel uploads
- **ARIA**: `role="dialog"`, `aria-describedby` for instructions
- **Keyboard**: Tab through controls, Enter to submit, Esc to cancel

**6. FileContextMenuComponent**
- **Responsibility**: Right-click action menu
- **Actions**: Open, Rename, Delete, Move, Copy, Share, Download, Properties
- **ARIA**: `role="menu"`, `role="menuitem"`
- **Keyboard**: Arrow keys to navigate, Enter to select, Esc to close

**7. BreadcrumbNavigationComponent**
- **Responsibility**: Show current path, navigate to parent folders
- **Features**: Click to jump, dropdown for sibling folders
- **ARIA**: `role="navigation"`, `aria-label="Breadcrumb"`
- **Keyboard**: Tab to each segment, Enter to activate

---

### State Design

#### FileManagerStore (NgRx SignalStore)

```typescript
export interface FileManagerState {
  // File tree (for left pane)
  fileTree: FileTreeNode[];
  expandedFolderIds: Set<string>;

  // Current folder
  currentFolderId: string | null;
  currentFolderFiles: FileNode[];

  // Selection
  selectedFileIds: string[];

  // Preview
  previewFile: FileNode | null;

  // View config
  viewMode: 'grid' | 'list' | 'compact';
  sortBy: 'name' | 'modifiedAt' | 'size';
  sortOrder: 'asc' | 'desc';

  // Filters
  filters: FileFilters;

  // Upload
  uploadQueue: UploadTask[];

  // UI state
  loading: boolean;
  error: string | null;
}

export const FileManagerStore = signalStore(
  { providedIn: 'root' },

  withState<FileManagerState>({
    fileTree: [],
    expandedFolderIds: new Set(),
    currentFolderId: null,
    currentFolderFiles: [],
    selectedFileIds: [],
    previewFile: null,
    viewMode: 'grid',
    sortBy: 'name',
    sortOrder: 'asc',
    filters: {},
    uploadQueue: [],
    loading: false,
    error: null,
  }),

  withComputed((store) => ({
    // Filtered and sorted files
    displayFiles: computed(() => {
      let files = store.currentFolderFiles();

      // Apply filters
      if (store.filters().type) {
        files = files.filter(f => f.type === store.filters().type);
      }

      // Apply sorting
      files = files.sort((a, b) => {
        const aVal = a[store.sortBy()];
        const bVal = b[store.sortBy()];
        const order = store.sortOrder() === 'asc' ? 1 : -1;
        return aVal < bVal ? -order : order;
      });

      return files;
    }),

    // Selected files (full objects, not just IDs)
    selectedFiles: computed(() =>
      store.currentFolderFiles().filter(f =>
        store.selectedFileIds().includes(f.id)
      )
    ),

    // Can delete? (check permissions)
    canDeleteSelection: computed(() =>
      store.selectedFiles().every(f => f.permissions.canDelete)
    ),
  })),

  withMethods((store, fileService = inject(FileService)) => ({
    // Load file tree
    async loadFileTree() {
      patchState(store, { loading: true, error: null });

      try {
        const tree = await fileService.getFileTree();
        patchState(store, { fileTree: tree, loading: false });
      } catch (error) {
        patchState(store, { loading: false, error: error.message });
      }
    },

    // Navigate to folder
    async navigateToFolder(folderId: string) {
      patchState(store, { loading: true, currentFolderId: folderId });

      try {
        const files = await fileService.listFiles(folderId);
        patchState(store, { currentFolderFiles: files, loading: false });
      } catch (error) {
        patchState(store, { loading: false, error: error.message });
      }
    },

    // Toggle folder expand/collapse
    toggleFolder(folderId: string) {
      patchState(store, (state) => {
        const expanded = new Set(state.expandedFolderIds);
        if (expanded.has(folderId)) {
          expanded.delete(folderId);
        } else {
          expanded.add(folderId);
        }
        return { expandedFolderIds: expanded };
      });
    },

    // Set selection
    setSelection(fileIds: string[]) {
      patchState(store, { selectedFileIds: fileIds });
    },

    // Upload files
    uploadFiles: rxMethod<File[]>(
      pipe(
        tap(files => {
          const tasks = files.map(file => ({
            id: generateId(),
            file,
            progress: 0,
            status: 'pending' as const,
          }));
          patchState(store, (state) => ({
            uploadQueue: [...state.uploadQueue, ...tasks],
          }));
        }),
        switchMap(files =>
          from(files).pipe(
            mergeMap(file =>
              fileService.uploadFile(file, store.currentFolderId(), 'document').pipe(
                tap(uploadedFile => {
                  // Add to current folder
                  patchState(store, (state) => ({
                    currentFolderFiles: [...state.currentFolderFiles, uploadedFile],
                  }));
                })
              ),
              3 // Max 3 concurrent uploads
            )
          )
        )
      )
    ),

    // Delete files
    async deleteFiles(fileIds: string[]) {
      patchState(store, { loading: true });

      try {
        await Promise.all(fileIds.map(id => fileService.deleteFile(id)));

        patchState(store, (state) => ({
          currentFolderFiles: state.currentFolderFiles.filter(f => !fileIds.includes(f.id)),
          selectedFileIds: [],
          loading: false,
        }));
      } catch (error) {
        patchState(store, { loading: false, error: error.message });
      }
    },
  })),
);
```

---

### Error & Empty States

#### Loading State
```html
<div class="file-list-skeleton">
  @for (i of [1,2,3,4,5,6,7,8]; track i) {
    <div class="file-item-skeleton">
      <div class="skeleton-icon"></div>
      <div class="skeleton-text"></div>
    </div>
  }
</div>
```

#### Empty State
```html
<app-empty-state
  icon="folder_open"
  title="This folder is empty"
  message="Drag files here to upload or click the upload button"
>
  <button mat-raised-button color="primary">
    <mat-icon>upload</mat-icon>
    Upload Files
  </button>
</app-empty-state>
```

#### Error State
```html
<app-error-state
  icon="error_outline"
  title="Failed to load files"
  [message]="store.error()"
>
  <button mat-button (click)="store.loadFiles()">
    <mat-icon>refresh</mat-icon>
    Retry
  </button>
</app-error-state>
```

---

### Accessibility

**ARIA Patterns Used**:
- **Tree View** (left pane): `role="tree"`, `role="treeitem"`, `aria-expanded`, `aria-level`, `aria-setsize`, `aria-posinset`
- **Grid** (file list in grid mode): `role="grid"`, `role="row"`, `role="gridcell"`
- **Listbox** (file list in list mode): `role="listbox"`, `role="option"`, `aria-selected`
- **Menu** (context menu): `role="menu"`, `role="menuitem"`
- **Dialog** (upload modal): `role="dialog"`, `aria-labelledby`, `aria-describedby`

**Keyboard Shortcuts**:
- `↑↓←→`: Navigate files/folders
- `Space`: Select file
- `Enter`: Open file/folder
- `F2`: Rename
- `Delete`: Delete selected
- `Cmd+A`: Select all
- `Cmd+C/V`: Copy/paste
- `Cmd+Z`: Undo
- `Cmd+F`: Focus search

**Screen Reader Support**:
- File count announcements: "Showing 42 of 1,234 files"
- Selection announcements: "3 files selected"
- Upload progress: "Uploading report.pdf, 45% complete"

---

## 🎨 Module 2: Design System Playground

### Overview
Living style guide and token editor for the design system. Designers and developers can view all components, edit design tokens in real-time, test themes, and run accessibility checks.

---

### Primary Screens & Layout

#### Main View: Token Editor + Component Gallery Split

```
┌────────────────────────────────────────────────────────────────┐
│ Design System                      [Light ▾] [Export] [Docs]   │
├──────────────────┬─────────────────────────────────────────────┤
│                  │                                             │
│  Token Browser   │         Component Gallery                   │
│                  │                                             │
│  🎨 Colors       │  ┌─────────────────────────────────────┐   │
│  ├ Primary       │  │  Button Component                   │   │
│  ├ Secondary     │  │  ┌──────┐ ┌──────┐ ┌──────┐        │   │
│  ├ Success       │  │  │ Save │ │Cancel│ │Delete│        │   │
│  └ Error         │  │  └──────┘ └──────┘ └──────┘        │   │
│                  │  │  Variants: Primary | Secondary |    │   │
│  📏 Spacing      │  │            Outlined | Text          │   │
│  ├ xs (4px)      │  │                                     │   │
│  ├ sm (8px)      │  │  Sizes: Small | Medium | Large      │   │
│  ├ md (16px)     │  │                                     │   │
│  └ ...           │  │  States: Default | Hover | Active | │   │
│                  │  │          Disabled | Focus           │   │
│  ✏️ Typography   │  └─────────────────────────────────────┘   │
│  ├ Font Family   │                                             │
│  ├ Font Size     │  [Live preview updates as tokens change]   │
│  └ Line Height   │                                             │
│                  │  [Accessibility score: 94/100]              │
│  🔲 Borders      │  [Contrast ratios: AAA ✓]                  │
│  🌑 Shadows      │                                             │
│                  │  Code: [HTML] [CSS] [Angular]              │
│ [280px]          │  <button mat-raised-button color="primary">│
└──────────────────┴─────────────────────────────────────────────┘
```

---

### Hard UI Features

#### 1. **Live Token Editing with Instant Preview**
- **What**: Edit color hex value → all components using that token update instantly
- **Why**: See impact of token changes in real-time
- **Implementation**: CSS custom properties, reactive token service, change detection

#### 2. **Theme Switcher with Animation**
- **What**: Switch between Light/Dark/High-Contrast themes with smooth transition
- **Why**: Test components across themes
- **Implementation**: CSS variable injection, transition animations, theme service

#### 3. **Component Variant Matrix**
- **What**: Display all variants of a component in a grid (e.g., Button: primary × small, primary × medium, secondary × small, etc.)
- **Why**: See all states at once for visual QA
- **Implementation**: Nested loops generating all combinations, dynamic component instantiation

#### 4. **Accessibility Audit Panel**
- **What**: Run automated a11y checks on components, show contrast ratios, ARIA issues
- **Why**: Ensure components meet WCAG standards
- **Implementation**: axe-core integration, contrast calculation, results panel

#### 5. **Token Search & Filter**
- **What**: Search tokens by name, filter by category, show usage count
- **Why**: Find tokens quickly in large design systems (100+ tokens)
- **Implementation**: Fuzzy search, category tabs, usage tracking

#### 6. **Token Dependency Graph**
- **What**: Visualize which tokens reference other tokens (e.g., `color-primary-light` derives from `color-primary`)
- **Why**: Understand token relationships, avoid circular references
- **Implementation**: D3.js or custom graph visualization, dependency resolver

#### 7. **Code Export (Multi-Format)**
- **What**: Export tokens as CSS variables, SCSS variables, JSON, TypeScript
- **Why**: Use tokens in different environments
- **Implementation**: Template generation, file download, clipboard copy

#### 8. **Component Playground (Interactive)**
- **What**: Change component props (size, color, disabled) in UI, see live preview
- **Why**: Test component behavior without editing code
- **Implementation**: Dynamic forms based on component inputs, property binding

#### 9. **Responsive Preview**
- **What**: View components at different viewport sizes (mobile, tablet, desktop)
- **Why**: Ensure responsive behavior
- **Implementation**: Iframe with variable width, breakpoint presets

#### 10. **Theme Comparison (Side-by-Side)**
- **What**: View same component in Light and Dark themes side-by-side
- **Why**: Ensure consistency across themes
- **Implementation**: Duplicate component instances with different theme contexts

---

### Component Breakdown

**1. DesignSystemShellComponent**
- **Responsibility**: Main container with token browser + gallery
- **State**: Active theme, active component, filter state

**2. TokenBrowserComponent**
- **Responsibility**: Hierarchical list of tokens, grouped by category
- **Features**: Search, filter, edit inline
- **ARIA**: `role="tree"` for hierarchy

**3. TokenEditorComponent**
- **Responsibility**: Edit single token (color picker, size input, etc.)
- **Features**: Input validation, preview swatch, undo/redo
- **ARIA**: Proper labels for inputs, live region for announcements

**4. ComponentGalleryComponent**
- **Responsibility**: Display all components with variants
- **Features**: Category tabs, search, isolated rendering
- **ARIA**: `role="tablist"` for categories

**5. ComponentPlaygroundComponent**
- **Responsibility**: Interactive component demo with prop controls
- **Features**: Knobs for props, code snippet, copy button
- **ARIA**: Form controls properly labeled

**6. AccessibilityPanelComponent**
- **Responsibility**: Show a11y audit results
- **Features**: Contrast checker, ARIA validator, keyboard nav test
- **ARIA**: `role="region"`, `aria-label="Accessibility audit"`

**7. ThemePreviewComponent**
- **Responsibility**: Preview component in different themes
- **Features**: Theme tabs, side-by-side comparison
- **ARIA**: `role="tablist"` for theme switcher

---

### State Design

#### DesignSystemStore

```typescript
export interface DesignSystemState {
  // Tokens
  tokens: DesignToken[];
  tokenCategories: TokenCategory[];
  editingTokenId: string | null;

  // Theme
  activeTheme: 'light' | 'dark' | 'high-contrast';
  customThemes: CustomTheme[];

  // Component gallery
  activeComponentId: string | null;
  components: ComponentDefinition[];

  // A11y
  a11yResults: A11yAuditResult[];

  // UI
  searchQuery: string;
  activeCategory: TokenCategory | null;
}

export const DesignSystemStore = signalStore(
  withState<DesignSystemState>({ ... }),

  withComputed((store) => ({
    // Filtered tokens
    filteredTokens: computed(() => {
      let tokens = store.tokens();

      if (store.searchQuery()) {
        tokens = tokens.filter(t =>
          t.name.toLowerCase().includes(store.searchQuery().toLowerCase())
        );
      }

      if (store.activeCategory()) {
        tokens = tokens.filter(t => t.category === store.activeCategory());
      }

      return tokens;
    }),

    // Token by ID lookup
    getTokenById: (id: string) => computed(() =>
      store.tokens().find(t => t.id === id)
    ),

    // CSS variables map
    cssVariables: computed(() => {
      const vars: Record<string, string> = {};
      store.tokens().forEach(token => {
        vars[token.cssVariable] = String(token.value);
      });
      return vars;
    }),
  })),

  withMethods((store, tokenService = inject(TokenService)) => ({
    // Load tokens
    async loadTokens() {
      const tokens = await tokenService.getAll();
      patchState(store, { tokens });
    },

    // Update token value
    updateToken(tokenId: string, value: string | number) {
      patchState(store, (state) => ({
        tokens: state.tokens.map(t =>
          t.id === tokenId ? { ...t, value } : t
        ),
      }));

      // Apply to DOM immediately
      this.applyCSSVariables();
    },

    // Apply CSS variables to document
    applyCSSVariables() {
      const vars = store.cssVariables();
      Object.entries(vars).forEach(([key, value]) => {
        document.documentElement.style.setProperty(key, value);
      });
    },

    // Switch theme
    setTheme(theme: 'light' | 'dark' | 'high-contrast') {
      patchState(store, { activeTheme: theme });
      document.documentElement.setAttribute('data-theme', theme);
    },

    // Export tokens
    exportTokens(format: 'css' | 'scss' | 'json' | 'ts'): string {
      const tokens = store.tokens();

      switch (format) {
        case 'css':
          return tokens.map(t => `${t.cssVariable}: ${t.value};`).join('\n');
        case 'scss':
          return tokens.map(t => `${t.scssVariable}: ${t.value};`).join('\n');
        case 'json':
          return JSON.stringify(tokens, null, 2);
        case 'ts':
          return `export const tokens = ${JSON.stringify(tokens, null, 2)};`;
      }
    },
  })),
);
```

---

### Accessibility

**ARIA Patterns**:
- **Tree** (token browser): `role="tree"`, `role="treeitem"`
- **Tabs** (component categories): `role="tablist"`, `role="tab"`, `role="tabpanel"`
- **Form** (token editor): Proper labels, `aria-describedby` for help text
- **Region** (a11y panel): `role="region"`, `aria-label`

**Keyboard Shortcuts**:
- `Cmd+K`: Focus search
- `Cmd+E`: Toggle edit mode
- `Cmd+S`: Save changes
- `Cmd+Shift+C`: Copy CSS
- `1-9`: Switch themes (1=Light, 2=Dark, etc.)

---

## 🖼️ Module 3: Media Library + Gallery Curation

### Overview
Digital asset management for images, videos, audio. Features masonry grid, infinite scroll, metadata editor, collections (curated galleries), and AI-powered tagging.

---

### Primary Screens & Layout

#### Main View: Grid with Sidebar

```
┌────────────────────────────────────────────────────────────────┐
│ Media Library                   [Grid ▾] [Upload] [🔍]         │
├──────────────┬─────────────────────────────────────────────────┤
│              │                                                 │
│  Filters     │         Masonry Grid (Infinite Scroll)         │
│              │                                                 │
│  Type        │  ┌────┐ ┌──────┐ ┌────┐                        │
│  ☑ Images    │  │    │ │      │ │    │ ┌──────┐ ┌────┐      │
│  ☐ Videos    │  │img1│ │ img2 │ │img3│ │      │ │img5│      │
│  ☐ Audio     │  │    │ │      │ │    │ │ img4 │ │    │      │
│              │  └────┘ └──────┘ └────┘ │      │ └────┘      │
│  Tags        │                         └──────┘              │
│  ☑ Product   │  [Variable height tiles, optimized layout]    │
│  ☑ Marketing │                                                 │
│  ☐ Archive   │  On hover: ✓ [select] | 👁️ [preview] | ...   │
│              │                                                 │
│  Date        │  Lazy loaded images (intersection observer)   │
│  Last 7 days │                                                 │
│              │  Virtual scroll container (1000s of images)    │
│  Size        │                                                 │
│  0-10MB      │  Status: Showing 42 of 1,234 | 5 selected     │
│              │                                                 │
│ Collections  │                                                 │
│ 📁 Brand     │                                                 │
│ 📁 Products  │                                                 │
│              │                                                 │
└──────────────┴─────────────────────────────────────────────────┘
```

#### Detail View (Overlay)

```
┌────────────────────────────────────────────────────────────────┐
│ [←] sunset.jpg                                          [✕]    │
├────────────────────────────────┬───────────────────────────────┤
│                                │                               │
│                                │  Metadata                     │
│         [Large Preview]        │  Type: Image/JPEG             │
│                                │  Dimensions: 3840×2160        │
│         [Zoom controls]        │  Size: 2.3 MB                 │
│                                │  Created: Jan 15, 2024        │
│                                │                               │
│                                │  Tags: [sunset] [beach] [+]   │
│      [← Previous] [Next →]     │                               │
│                                │  Collections:                 │
│                                │  • Summer 2024                │
│                                │  • Featured Images            │
│                                │                               │
│                                │  Color Palette:               │
│                                │  🟠🟡🔵🟢🔴                   │
│                                │                               │
│                                │  [Download] [Share] [Delete]  │
└────────────────────────────────┴───────────────────────────────┘
```

---

### Hard UI Features

#### 1. **Masonry Grid Layout (Pinterest-Style)**
- **What**: Variable-height tiles that pack efficiently without gaps
- **Why**: Images have different aspect ratios, fixed grid wastes space
- **Implementation**: CSS Grid with `grid-auto-rows`, Masonry layout library, or custom algorithm

#### 2. **Infinite Scroll with Intersection Observer**
- **What**: Load more images as user scrolls to bottom
- **Why**: Don't load 10k images upfront, progressive loading
- **Implementation**: IntersectionObserver API, sentinel element, pagination

#### 3. **Lazy Image Loading with Blur-Up**
- **What**: Show blurred placeholder → load low-res → load high-res
- **Why**: Perceived performance, avoid layout shift
- **Implementation**: Canvas blur, progressive JPEG, Intersection Observer

#### 4. **Bulk Image Upload with Thumbnails**
- **What**: Upload 100+ images, show thumbnail previews before upload
- **Why**: Verify uploads before committing
- **Implementation**: FileReader for thumbnails, upload queue, progress tracking

#### 5. **Drag-and-Drop to Collections**
- **What**: Drag images from grid → drop on collection in sidebar → add to collection
- **Why**: Organize images quickly
- **Implementation**: CDK Drag-Drop, drop validation, optimistic UI

#### 6. **Lightbox with Keyboard Navigation**
- **What**: Click image → lightbox opens → arrow keys to navigate, Esc to close
- **Why**: View images in detail without losing context
- **Implementation**: Modal overlay, keyboard event handling, preload adjacent images

#### 7. **Advanced Filters with Chips**
- **What**: Multi-select filters (type, tags, date) displayed as removable chips
- **Why**: Complex queries visually represented
- **Implementation**: Filter service, chip component, URL query sync

#### 8. **AI Auto-Tagging (Simulated)**
- **What**: Upload image → AI suggests tags (e.g., "sunset", "beach", "ocean")
- **Why**: Easier discovery, less manual tagging
- **Implementation**: Mock AI service (returns random relevant tags), tag suggestion UI

#### 9. **Color Palette Extraction**
- **What**: Extract dominant colors from image, display as swatches
- **Why**: Search by color, visual consistency in collections
- **Implementation**: Canvas API to sample pixels, K-means clustering, color quantization

#### 10. **Metadata Bulk Edit**
- **What**: Select multiple images → edit tags/collections for all at once
- **Why**: Batch operations save time
- **Implementation**: Multi-select model, bulk update API, optimistic UI

#### 11. **Image Crop & Resize (In-Browser)**
- **What**: Crop image to square, resize to 1024px, download
- **Why**: Basic editing without leaving app
- **Implementation**: Canvas API, Cropperjs, file download

---

### Component Breakdown

**1. MediaLibraryShellComponent**
- **Responsibility**: Main container with filters + grid
- **State**: Active filters, view mode, selection

**2. MediaGridComponent**
- **Responsibility**: Masonry grid with infinite scroll
- **Features**: Lazy loading, intersection observer, hover actions
- **ARIA**: `role="grid"`, `role="gridcell"`, `aria-label` per image

**3. MediaCardComponent**
- **Responsibility**: Single image/video tile
- **Features**: Thumbnail, overlay actions (select, preview, delete)
- **ARIA**: `role="button"` for clickable card, alt text for image

**4. MediaLightboxComponent**
- **Responsibility**: Full-screen image viewer
- **Features**: Zoom, pan, navigate, metadata sidebar
- **ARIA**: `role="dialog"`, focus trap, Esc to close
- **Keyboard**: ←→ for prev/next, Esc to close, +/- to zoom

**5. MediaUploaderComponent**
- **Responsibility**: Bulk upload with drag-drop
- **Features**: Thumbnail previews, progress bars, cancel
- **ARIA**: `role="dialog"`, progress `aria-live`

**6. MediaFiltersPanelComponent**
- **Responsibility**: Filter sidebar with type, tags, date, size
- **Features**: Checkboxes, chips, range sliders
- **ARIA**: `role="form"`, fieldsets with legends

**7. CollectionManagerComponent**
- **Responsibility**: Create/edit/delete collections
- **Features**: Drag-drop images, rename, cover image selection
- **ARIA**: `role="tree"` for collection list

**8. MetadataEditorComponent**
- **Responsibility**: Edit image metadata (tags, title, description)
- **Features**: Tag autocomplete, collection picker
- **ARIA**: Form labels, tag list `role="list"`

---

### State Design

#### MediaStore

```typescript
export interface MediaState {
  // Assets
  assets: MediaAsset[];
  totalCount: number;
  currentPage: number;

  // Collections
  collections: Collection[];

  // Filters
  filters: MediaFilters;

  // Selection
  selectedAssetIds: string[];

  // Lightbox
  lightboxAssetId: string | null;
  lightboxIndex: number;

  // Upload
  uploadQueue: UploadTask[];

  // UI
  viewMode: 'grid' | 'list';
  loading: boolean;
  error: string | null;
}

export const MediaStore = signalStore(
  withState<MediaState>({ ... }),

  withComputed((store) => ({
    // Filtered assets
    filteredAssets: computed(() => {
      let assets = store.assets();

      if (store.filters().type) {
        assets = assets.filter(a => a.type === store.filters().type);
      }

      if (store.filters().tags?.length) {
        assets = assets.filter(a =>
          store.filters().tags.some(tag => a.tags.includes(tag))
        );
      }

      return assets;
    }),

    // Selected assets
    selectedAssets: computed(() =>
      store.assets().filter(a => store.selectedAssetIds().includes(a.id))
    ),

    // Lightbox asset
    lightboxAsset: computed(() =>
      store.assets().find(a => a.id === store.lightboxAssetId())
    ),
  })),

  withMethods((store, mediaService = inject(MediaService)) => ({
    // Load more assets (infinite scroll)
    async loadMore() {
      patchState(store, { loading: true });

      try {
        const page = store.currentPage() + 1;
        const newAssets = await mediaService.getAssets({
          page,
          filters: store.filters(),
        });

        patchState(store, (state) => ({
          assets: [...state.assets, ...newAssets],
          currentPage: page,
          loading: false,
        }));
      } catch (error) {
        patchState(store, { loading: false, error: error.message });
      }
    },

    // Upload assets
    uploadAssets: rxMethod<File[]>(
      pipe(
        tap(files => {
          const tasks = files.map(file => ({
            id: generateId(),
            file,
            progress: 0,
            status: 'pending' as const,
          }));
          patchState(store, (state) => ({
            uploadQueue: [...state.uploadQueue, ...tasks],
          }));
        }),
        switchMap(files =>
          from(files).pipe(
            mergeMap(file =>
              mediaService.uploadAsset(file).pipe(
                tap(asset => {
                  patchState(store, (state) => ({
                    assets: [asset, ...state.assets],
                  }));
                })
              ),
              3 // Max 3 concurrent
            )
          )
        )
      )
    ),

    // Add to collection
    async addToCollection(assetIds: string[], collectionId: string) {
      await mediaService.addToCollection(assetIds, collectionId);

      patchState(store, (state) => ({
        assets: state.assets.map(a =>
          assetIds.includes(a.id)
            ? { ...a, collections: [...a.collections, collectionId] }
            : a
        ),
      }));
    },

    // Extract color palette (run in worker)
    async extractColors(assetId: string) {
      const asset = store.assets().find(a => a.id === assetId);
      if (!asset) return;

      const colors = await mediaService.extractColors(asset.url);

      patchState(store, (state) => ({
        assets: state.assets.map(a =>
          a.id === assetId ? { ...a, colorPalette: colors } : a
        ),
      }));
    },
  })),
);

interface MediaAsset {
  id: string;
  type: 'image' | 'video' | 'audio';
  url: string;
  thumbnailUrl: string;
  width: number;
  height: number;
  size: number;
  mimeType: string;
  tags: string[];
  collections: string[];
  colorPalette?: string[];
  createdAt: Date;
}
```

---

### Accessibility

**ARIA Patterns**:
- **Grid** (masonry layout): `role="grid"`, `role="gridcell"`
- **Dialog** (lightbox): `role="dialog"`, focus trap, `aria-labelledby`
- **Form** (filters): Proper labels, fieldsets
- **List** (collections): `role="list"`, `role="listitem"`

**Keyboard Shortcuts**:
- `Space`: Select image
- `Enter`: Open lightbox
- `←→`: Navigate in lightbox
- `Esc`: Close lightbox
- `Delete`: Delete selected
- `Cmd+A`: Select all

**Screen Reader**:
- Image alt text from metadata or filename
- "Image 1 of 42 selected"
- "Uploading sunset.jpg, 67% complete"

---

## ✅ Modules 1-3 Checklist

### Module 1: File Manager
- ✅ 12 hard UI features (virtual tree, drag-drop, multi-select, quick preview, etc.)
- ✅ Complete component breakdown (7 components)
- ✅ FileManagerStore with state management
- ✅ Error/loading/empty states
- ✅ Accessibility (tree, grid, menu, dialog ARIA patterns)

### Module 2: Design System
- ✅ 10 hard UI features (live token editing, theme switcher, a11y audit, etc.)
- ✅ Complete component breakdown (7 components)
- ✅ DesignSystemStore with token management
- ✅ Accessibility (tree, tabs, forms, regions)

### Module 3: Media Library
- ✅ 11 hard UI features (masonry grid, infinite scroll, lightbox, color extraction, etc.)
- ✅ Complete component breakdown (8 components)
- ✅ MediaStore with infinite scroll and upload queue
- ✅ Accessibility (grid, dialog, forms, lists)

---

**Next**: Phase 5 - Modules 4-6 (Kanban, Logs, Knowledge Base)
