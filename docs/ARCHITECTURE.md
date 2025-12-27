# Architecture: Module Boundaries & Integration

> **How 10 modules integrate into one cohesive product**

---

## 🎯 Architectural Vision

### Core Principles

1. **Modular but Unified**: Each module is independent but shares common systems
2. **Lazy by Default**: Modules load on demand, not upfront
3. **State Isolation**: Feature state is scoped, global state is minimal
4. **Loose Coupling**: Modules communicate through services, not direct dependencies
5. **Composition over Inheritance**: Reuse through shared components and services

---

## 📦 Module Boundaries

### Module Map

```
src/app/
├── core/                              # Singleton services, global state
│   ├── services/
│   │   ├── auth.service.ts
│   │   ├── tenant.service.ts
│   │   ├── permissions.service.ts
│   │   ├── file.service.ts
│   │   ├── search.service.ts
│   │   ├── activity.service.ts
│   │   ├── notification.service.ts
│   │   ├── mock-api.service.ts
│   │   └── indexed-db.service.ts
│   ├── stores/
│   │   └── app-shell.store.ts        # Global workspace state
│   ├── guards/
│   │   ├── auth.guard.ts
│   │   ├── tenant-access.guard.ts
│   │   └── permission.guard.ts
│   └── interceptors/
│       └── error.interceptor.ts
├── shared/                            # Reusable components, pipes, directives
│   ├── components/
│   │   ├── command-palette/
│   │   ├── search-overlay/
│   │   ├── file-picker/
│   │   ├── user-avatar/
│   │   ├── empty-state/
│   │   ├── loading-skeleton/
│   │   └── ...
│   ├── pipes/
│   │   ├── file-size.pipe.ts
│   │   ├── time-ago.pipe.ts
│   │   └── highlight.pipe.ts
│   ├── directives/
│   │   ├── has-permission.directive.ts
│   │   ├── focus-trap.directive.ts
│   │   └── auto-focus.directive.ts
│   └── utils/
│       ├── date-utils.ts
│       ├── file-utils.ts
│       └── validation-utils.ts
├── layout/                            # Shell components
│   ├── workspace-shell/
│   ├── top-bar/
│   ├── left-navigation/
│   ├── tab-bar/
│   └── status-bar/
├── features/                          # Feature modules (lazy-loaded)
│   ├── dashboard/
│   │   ├── dashboard.component.ts
│   │   ├── dashboard.routes.ts
│   │   └── widgets/
│   ├── files/
│   │   ├── files.routes.ts
│   │   ├── stores/
│   │   │   └── file-manager.store.ts
│   │   ├── components/
│   │   │   ├── file-tree/
│   │   │   ├── file-list/
│   │   │   ├── file-preview/
│   │   │   └── upload-dialog/
│   │   └── services/
│   │       ├── file-tree.service.ts
│   │       └── file-preview.service.ts
│   ├── design-system/
│   │   ├── design-system.routes.ts
│   │   ├── stores/
│   │   │   └── design-system.store.ts
│   │   └── components/
│   │       ├── token-editor/
│   │       ├── component-gallery/
│   │       └── theme-preview/
│   ├── media/
│   │   ├── media.routes.ts
│   │   ├── stores/
│   │   │   └── media.store.ts
│   │   └── components/
│   │       ├── media-grid/
│   │       ├── media-uploader/
│   │       └── collection-manager/
│   ├── kanban/
│   │   ├── kanban.routes.ts
│   │   ├── stores/
│   │   │   ├── kanban.store.ts
│   │   │   └── timeline.store.ts
│   │   └── components/
│   │       ├── board-view/
│   │       ├── timeline-view/
│   │       ├── card-detail/
│   │       └── card-editor/
│   ├── logs/
│   │   ├── logs.routes.ts
│   │   ├── stores/
│   │   │   └── logs.store.ts
│   │   └── components/
│   │       ├── log-stream/
│   │       ├── log-filters/
│   │       └── trace-viewer/
│   ├── knowledge-base/
│   │   ├── kb.routes.ts
│   │   ├── stores/
│   │   │   └── kb.store.ts
│   │   └── components/
│   │       ├── doc-editor/
│   │       ├── doc-viewer/
│   │       └── doc-tree/
│   ├── cases/
│   │   ├── cases.routes.ts
│   │   ├── stores/
│   │   │   └── cases.store.ts
│   │   └── components/
│   │       ├── case-list/
│   │       ├── case-detail/
│   │       ├── case-timeline/
│   │       └── reply-editor/
│   ├── datasets/
│   │   ├── datasets.routes.ts
│   │   ├── stores/
│   │   │   └── datasets.store.ts
│   │   └── components/
│   │       ├── dataset-grid/
│   │       ├── dataset-profiler/
│   │       └── query-builder/
│   ├── forms/
│   │   ├── forms.routes.ts
│   │   ├── stores/
│   │   │   └── forms.store.ts
│   │   └── components/
│   │       ├── form-builder/
│   │       ├── form-preview/
│   │       └── form-responses/
│   └── admin/
│       ├── admin.routes.ts
│       ├── stores/
│       │   └── admin.store.ts
│       └── components/
│           ├── user-management/
│           ├── role-management/
│           └── audit-log/
└── workers/
    ├── search.worker.ts
    └── dataset-profiler.worker.ts
```

---

## 🔄 Data Flow Architecture

### Request Flow Example: Opening a Case

```
1. User clicks case in Case Triage module
   │
   ▼
2. CasesComponent calls: casesStore.loadCase(caseId)
   │
   ▼
3. CasesStore (SignalStore) dispatches async method
   │
   ▼
4. CaseService.getCase(caseId)
   │
   ▼
5. MockApiService.get('/api/cases/123')
   │
   ▼
6. IndexedDbService.getById('cases', '123')
   │
   ▼
7. IndexedDB returns case data
   │
   ▼
8. MockApiService resolves with case
   │
   ▼
9. CaseService returns Observable/Promise
   │
   ▼
10. CasesStore updates signal: patchState({ currentCase: case })
    │
    ▼
11. CaseDetailComponent reads: casesStore.currentCase()
    │
    ▼
12. Angular detects signal change, rerenders template
```

### Cross-Module Communication Example: Linking Case to File

```
Scenario: User wants to attach a file to a case

1. User in CaseDetailComponent clicks "Attach File"
   │
   ▼
2. Opens FilePicker component (from shared/)
   │
   ▼
3. FilePicker uses FileService (core) to list files
   │
   ▼
4. User selects file
   │
   ▼
5. FilePicker emits: fileSelected(fileId)
   │
   ▼
6. CaseDetailComponent receives file ID
   │
   ▼
7. Calls: caseService.linkFile(caseId, fileId)
   │
   ▼
8. CaseService updates case.linkedFiles array
   │
   ▼
9. FileService updates file.linkedEntities array
   │
   ▼
10. ActivityService logs: "User attached file X to case Y"
    │
    ▼
11. Both stores update, UI rerenders
```

---

## 🧩 Module Integration Patterns

### Pattern 1: Shared Service (Loose Coupling)

**Use Case**: Multiple modules need access to files.

**Implementation**:
```typescript
// core/services/file.service.ts
@Injectable({ providedIn: 'root' })
export class FileService {
  // Shared across all modules
  async getFile(fileId: string): Promise<FileNode> { ... }
}

// features/files/components/file-list.component.ts
export class FileListComponent {
  private fileService = inject(FileService);  // Injected
}

// features/cases/components/case-detail.component.ts
export class CaseDetailComponent {
  private fileService = inject(FileService);  // Same instance
}
```

**Why**: Single source of truth, no direct module dependencies.

---

### Pattern 2: Event Bus (Decoupled Communication)

**Use Case**: Module A needs to notify Module B without knowing about it.

**Implementation**:
```typescript
// core/services/event-bus.service.ts
@Injectable({ providedIn: 'root' })
export class EventBus {
  private events$ = new Subject<AppEvent>();

  emit(event: AppEvent) {
    this.events$.next(event);
  }

  on<T extends AppEvent>(type: string): Observable<T> {
    return this.events$.pipe(
      filter(e => e.type === type),
      map(e => e as T)
    );
  }
}

// features/datasets/components/dataset-detail.component.ts
export class DatasetDetailComponent {
  openSourceFile() {
    this.eventBus.emit({
      type: 'file:open',
      payload: { fileId: this.dataset().fileId },
    });
  }
}

// features/files/components/file-manager.component.ts
export class FileManagerComponent {
  constructor() {
    this.eventBus.on('file:open').subscribe(event => {
      this.openFile(event.payload.fileId);
    });
  }
}
```

**Why**: Modules don't know about each other, easy to add/remove listeners.

---

### Pattern 3: Router-Based Navigation (URL as API)

**Use Case**: Deep link from one module to another.

**Implementation**:
```typescript
// features/cases/components/case-detail.component.ts
export class CaseDetailComponent {
  openRelatedLog(logId: string) {
    // Navigate to logs module with log ID
    this.router.navigate(['/t', this.tenantId, 'logs'], {
      queryParams: { logId },
    });
  }
}

// features/logs/components/log-viewer.component.ts
export class LogViewerComponent {
  ngOnInit() {
    // Read logId from URL and highlight
    this.route.queryParams.subscribe(params => {
      if (params['logId']) {
        this.highlightLog(params['logId']);
      }
    });
  }
}
```

**Why**: Bookmarkable, shareable, stateless integration.

---

### Pattern 4: Shared Component (Composition)

**Use Case**: Multiple modules need same UI (e.g., file picker, user selector).

**Implementation**:
```typescript
// shared/components/file-picker/file-picker.component.ts
@Component({
  selector: 'app-file-picker',
  standalone: true,
  template: `...`,
})
export class FilePickerComponent {
  @Output() fileSelected = new EventEmitter<string>();

  selectFile(fileId: string) {
    this.fileSelected.emit(fileId);
  }
}

// features/cases/components/case-detail.component.ts
@Component({
  selector: 'app-case-detail',
  imports: [FilePickerComponent],
  template: `
    <app-file-picker (fileSelected)="onFileSelected($event)" />
  `,
})
export class CaseDetailComponent {
  onFileSelected(fileId: string) {
    this.caseService.attachFile(this.caseId, fileId);
  }
}
```

**Why**: Consistent UI, no code duplication, composition over inheritance.

---

## 🔐 Security Boundaries

### Permission Enforcement Layers

```
┌──────────────────────────────────────────────────────┐
│ Layer 1: Route Guards                               │
│ - Protect entire modules                            │
│ - Example: Admin module requires 'admin' role       │
└──────────────────────────────────────────────────────┘
                      │
                      ▼
┌──────────────────────────────────────────────────────┐
│ Layer 2: Component-Level Permissions                │
│ - Show/hide UI elements                             │
│ - Example: Delete button only if 'files:delete'     │
└──────────────────────────────────────────────────────┘
                      │
                      ▼
┌──────────────────────────────────────────────────────┐
│ Layer 3: Service-Level Validation                   │
│ - Check permissions before API calls                │
│ - Example: FileService.delete() checks permission   │
└──────────────────────────────────────────────────────┘
                      │
                      ▼
┌──────────────────────────────────────────────────────┐
│ Layer 4: Entity-Level Permissions                   │
│ - File/case/dataset-specific permissions            │
│ - Example: file.permissions.canDelete               │
└──────────────────────────────────────────────────────┘
```

### Example: Multi-Layer Permission Check

```typescript
// Layer 1: Route Guard
export const adminRoutes: Routes = [
  {
    path: 'admin',
    canActivate: [roleGuard(['admin'])],  // Block at route level
    loadComponent: () => import('./admin.component'),
  },
];

// Layer 2: Component (template)
<button *appHasPermission="'files:delete'" (click)="deleteFile()">
  Delete
</button>

// Layer 3: Service
@Injectable()
export class FileService {
  deleteFile(fileId: string) {
    // Check permission before API call
    if (!this.permissions.hasPermission('files:delete')) {
      throw new Error('Permission denied');
    }

    return this.api.delete(`/files/${fileId}`);
  }
}

// Layer 4: Entity-level
canDeleteFile(file: FileNode): boolean {
  // Check both role permission AND file-specific permission
  return (
    this.permissions.hasPermission('files:delete') &&
    file.permissions.canDelete
  );
}
```

---

## 📊 State Ownership

### State Boundaries Matrix

| State | Owner | Scope | Lifetime |
|-------|-------|-------|----------|
| Current user | AuthService | Global | Session |
| Current tenant | TenantService | Global | Session |
| Workspace tabs | AppShellStore | Global | Persisted |
| File tree | FileManagerStore | Feature (files) | Route |
| Selected files | FileManagerStore | Feature (files) | Route |
| Case list | CasesStore | Feature (cases) | Route |
| Current case | CasesStore | Feature (cases) | Route |
| Dataset rows | DatasetsStore | Feature (datasets) | Route |
| Modal open/closed | Component signal | Component | Component |
| Form input value | Component signal | Component | Component |

### State Sharing Rules

**Rule 1**: If state is needed across routes → Global Store
**Rule 2**: If state is needed across components in a feature → Feature Store
**Rule 3**: If state is only needed in one component → Component Signal

**Example**:

```typescript
// ❌ Bad: File tree in component state
@Component({...})
export class FileTreeComponent {
  fileTree = signal<FileNode[]>([]);  // Lost on unmount!
}

// ✅ Good: File tree in feature store
export const FileManagerStore = signalStore(
  withState({ fileTree: [] }),
  // Feature store survives route changes within /files
);
```

---

## 🔌 Dependency Injection Architecture

### Provider Hierarchy

```
1. Root Providers (app.config.ts)
   - provideRouter()
   - provideHttpClient()
   - Services marked providedIn: 'root'

2. Route-Level Providers
   - Feature stores (FileManagerStore, CasesStore)
   - Feature-specific services

3. Component Providers
   - Rarely used (prefer inject() in constructors)
```

### Example: Route-Level DI

```typescript
// features/files/files.routes.ts
export const filesRoutes: Routes = [
  {
    path: '',
    providers: [
      FileManagerStore,              // Scoped to /files route
      FileTreeService,
      FilePreviewService,
    ],
    children: [
      { path: '', component: FileListComponent },
      { path: 'file/:id', component: FileDetailComponent },
    ],
  },
];

// Both FileListComponent and FileDetailComponent get the SAME instance
// When navigating away from /files, the store is destroyed
```

---

## 🎨 Styling Architecture

### Theme System

```
styles/
├── _variables.scss                  # CSS custom properties
├── _reset.scss                      # Normalize styles
├── _typography.scss                 # Font styles
├── _utilities.scss                  # Utility classes
├── themes/
│   ├── _light.scss                  # Light theme tokens
│   ├── _dark.scss                   # Dark theme tokens
│   └── _high-contrast.scss          # High contrast theme
└── main.scss                        # Entry point
```

### Component Styling Strategy

**Approach**: **Component-scoped styles with theming via CSS variables**

```scss
// Component styles (scoped with :host)
:host {
  display: block;
  padding: var(--spacing-md);
  background: var(--surface);
  border: 1px solid var(--border);
}

.file-item {
  padding: var(--spacing-sm);
  color: var(--text-primary);

  &:hover {
    background: var(--surface-hover);
  }
}
```

**Why**:
- Scoped styles prevent conflicts
- CSS variables enable runtime theming
- No style pollution across modules

---

## 📡 Error Handling Strategy

### Error Propagation

```
1. Error occurs in service (e.g., API call fails)
   │
   ▼
2. Service catches error, logs to ActivityService
   │
   ▼
3. Service rethrows or returns error state
   │
   ▼
4. Store method catches error, updates error signal
   │
   ▼
5. Component reads error signal, shows error UI
   │
   ▼
6. NotificationService shows toast notification
   │
   ▼
7. Status bar shows error count
```

### Error Handling Example

```typescript
// Service layer
@Injectable()
export class FileService {
  async uploadFile(file: File): Promise<FileNode> {
    try {
      const fileNode = await this.api.post('/files', file);

      // Log success
      this.activity.logActivity({
        action: 'uploaded',
        entityType: 'file',
        entityId: fileNode.id,
        entityName: fileNode.name,
      });

      return fileNode;
    } catch (error) {
      // Log error
      this.activity.logActivity({
        action: 'upload-failed',
        entityType: 'file',
        entityId: 'unknown',
        entityName: file.name,
        metadata: { error: error.message },
      });

      // Show notification
      this.notifications.createNotification({
        type: 'error',
        title: 'Upload Failed',
        message: `Could not upload ${file.name}`,
        priority: 'high',
      });

      throw error;  // Rethrow for store to handle
    }
  }
}

// Store layer
export const FileManagerStore = signalStore(
  withState({ error: null }),

  withMethods((store, fileService = inject(FileService)) => ({
    uploadFile: rxMethod<File>(
      pipe(
        tap(() => patchState(store, { uploading: true, error: null })),
        switchMap(file =>
          from(fileService.uploadFile(file)).pipe(
            tap(fileNode => {
              patchState(store, {
                files: [...store.files(), fileNode],
                uploading: false,
              });
            }),
            catchError(error => {
              patchState(store, {
                uploading: false,
                error: error.message,
              });
              return of(null);
            })
          )
        )
      )
    ),
  })),
);

// Component layer
@Component({
  template: `
    @if (store.error()) {
      <div class="error-banner">
        {{ store.error() }}
        <button (click)="store.clearError()">Dismiss</button>
      </div>
    }
  `
})
export class FileListComponent {
  store = inject(FileManagerStore);
}
```

---

## 🧪 Testing Strategy

### Testing Pyramid

```
                  ┌─────────────┐
                  │     E2E     │  < 10% (critical paths only)
                  └─────────────┘
                ┌───────────────────┐
                │   Component Tests │  ~30% (user interactions)
                └───────────────────┘
          ┌─────────────────────────────────┐
          │        Unit Tests               │  ~60% (services, stores, utils)
          └─────────────────────────────────┘
```

### Module Testing Example

```typescript
// Unit test: Service
describe('FileService', () => {
  let service: FileService;
  let mockApi: jasmine.SpyObj<MockApiService>;

  beforeEach(() => {
    mockApi = jasmine.createSpyObj('MockApiService', ['get', 'post']);

    TestBed.configureTestingModule({
      providers: [
        FileService,
        { provide: MockApiService, useValue: mockApi },
      ],
    });

    service = TestBed.inject(FileService);
  });

  it('should upload file', async () => {
    const file = new File(['content'], 'test.txt');
    const mockResponse = { id: '123', name: 'test.txt' };

    mockApi.post.and.returnValue(Promise.resolve(mockResponse));

    const result = await service.uploadFile(file, null, 'document');

    expect(result).toEqual(mockResponse);
    expect(mockApi.post).toHaveBeenCalledWith('/files', jasmine.any(Object));
  });
});

// Component test: Interactions
describe('FileListComponent', () => {
  it('should display files', async () => {
    const fixture = TestBed.createComponent(FileListComponent);
    const store = TestBed.inject(FileManagerStore);

    store.loadFiles();
    await fixture.whenStable();
    fixture.detectChanges();

    const fileItems = fixture.nativeElement.querySelectorAll('.file-item');
    expect(fileItems.length).toBe(store.files().length);
  });

  it('should open file on click', async () => {
    const fixture = TestBed.createComponent(FileListComponent);
    const store = TestBed.inject(FileManagerStore);
    const router = TestBed.inject(Router);
    spyOn(router, 'navigate');

    store.loadFiles();
    await fixture.whenStable();
    fixture.detectChanges();

    const firstFile = fixture.nativeElement.querySelector('.file-item');
    firstFile.click();

    expect(router.navigate).toHaveBeenCalled();
  });
});

// E2E test: Critical path
describe('File Upload Flow', () => {
  it('should upload file and see it in list', () => {
    cy.visit('/t/demo/files');

    // Upload file
    cy.get('[data-testid="upload-button"]').click();
    cy.get('input[type="file"]').attachFile('test-file.txt');
    cy.get('[data-testid="upload-confirm"]').click();

    // Verify file appears in list
    cy.contains('test-file.txt').should('be.visible');

    // Verify activity logged
    cy.get('[data-testid="activity-feed"]').should('contain', 'uploaded');
  });
});
```

---

## ✅ Architecture Checklist

**Module Organization**:
- ✅ Clear folder structure (core, shared, layout, features)
- ✅ Lazy-loaded feature modules
- ✅ Route-level providers for feature stores
- ✅ Standalone components throughout

**Data Flow**:
- ✅ Unidirectional: User → Store → Service → API → Store → Component
- ✅ Signals for reactive updates
- ✅ RxJS for async workflows

**Integration Patterns**:
- ✅ Shared services (FileService, PermissionsService)
- ✅ Event bus for decoupled communication
- ✅ Router-based deep linking
- ✅ Shared components (composition)

**Security**:
- ✅ Multi-layer permission checks (route, component, service, entity)
- ✅ RBAC with permission matrix
- ✅ Tenant isolation

**State Management**:
- ✅ Clear state ownership (global vs feature vs component)
- ✅ Three-tier architecture (AppShellStore, feature stores, signals)
- ✅ Route-scoped feature state

**Error Handling**:
- ✅ Consistent error propagation (service → store → component)
- ✅ Activity logging for audit
- ✅ User notifications

**Testing**:
- ✅ Unit tests for services and stores
- ✅ Component tests for interactions
- ✅ E2E tests for critical paths

---

**Phase 3 Complete!**

Files created:
- ✅ SHARED_SYSTEMS.md (File model, multi-tenancy, search, activity, notifications)
- ✅ docs/DATA_LAYER.md (Data models, Mock API, IndexedDB, workers)
- ✅ docs/ARCHITECTURE.md (Module boundaries, integration patterns, testing)

**Next**: Phase 4 - Module Designs (Modules 1-3: Files, Design System, Media)
