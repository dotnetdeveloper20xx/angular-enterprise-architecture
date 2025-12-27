# State Management: Signals + SignalStore

> **Comprehensive guide to state management in Nexus Studio using Angular Signals and NgRx SignalStore**

---

## 🎯 What & Why

### What is Our State Strategy?

Nexus Studio uses a **signals-first** approach with three tiers:

1. **Component State** → Angular Signals (`signal()`, `computed()`, `effect()`)
2. **Feature State** → NgRx SignalStore (route-scoped stores)
3. **Global State** → NgRx SignalStore (app-level store)

### Why Not Traditional NgRx?

**Traditional NgRx** (Actions, Reducers, Effects, Selectors):
- ✗ Boilerplate-heavy (hundreds of lines for simple features)
- ✗ Requires Zone.js for change detection
- ✗ Indirect: dispatch action → reducer → selector → template
- ✗ Harder to debug (action flow through middleware)

**Signals + SignalStore**:
- ✓ Minimal boilerplate (stores are ~50 lines)
- ✓ Zoneless-ready (future-proof)
- ✓ Direct: update signal → computed auto-updates → template rerenders
- ✓ Easy to debug (reactive graph is clear)
- ✓ Better TypeScript inference
- ✓ Composable (mix stores, signals, RxJS)

**Bottom line**: SignalStore is the modern, recommended approach for Angular 19+.

---

## 📚 Fundamentals: Angular Signals

### signal() - Writable State

```typescript
import { signal } from '@angular/core';

// Create a signal
const count = signal(0);

// Read value
console.log(count());  // 0

// Update value
count.set(10);         // Replace
count.update(n => n + 1);  // Transform

// In templates
@Component({
  template: `<div>Count: {{ count() }}</div>`
})
export class MyComponent {
  count = signal(0);

  increment() {
    this.count.update(n => n + 1);
  }
}
```

**Key Points**:
- Signals are **read** with `()`: `count()`
- Signals are **written** with `.set()` or `.update()`
- Templates **automatically track** signal reads and rerender on change
- No need for `markForCheck()` or Zone.js

---

### computed() - Derived State

```typescript
import { signal, computed } from '@angular/core';

const firstName = signal('John');
const lastName = signal('Doe');

// Automatically recomputes when dependencies change
const fullName = computed(() => `${firstName()} ${lastName()}`);

console.log(fullName());  // "John Doe"

firstName.set('Jane');
console.log(fullName());  // "Jane Doe" (auto-updated!)
```

**Key Points**:
- `computed()` creates **read-only** derived state
- Automatically tracks dependencies (any signals read inside)
- **Memoized**: only recomputes when dependencies change
- Can depend on other computed signals (chain them)

**Example**: Filtering

```typescript
const items = signal([1, 2, 3, 4, 5]);
const filter = signal('even');

const filteredItems = computed(() => {
  const f = filter();
  return items().filter(item =>
    f === 'even' ? item % 2 === 0 : item % 2 !== 0
  );
});

// filteredItems() → [2, 4]
// Change filter
filter.set('odd');
// filteredItems() → [1, 3, 5] (auto-updated!)
```

---

### effect() - Side Effects

```typescript
import { effect } from '@angular/core';

const count = signal(0);

// Runs whenever `count` changes
effect(() => {
  console.log('Count changed to:', count());
  localStorage.setItem('count', String(count()));
});

count.set(5);  // Logs: "Count changed to: 5"
```

**Key Points**:
- `effect()` runs **automatically** when dependencies change
- Use for side effects: logging, persistence, analytics, DOM manipulation
- Runs in injection context (must be in constructor or `runInInjectionContext()`)
- Can be cleaned up with `destroy()` or automatically on component destroy

**Common Use Cases**:
- Persist state to localStorage
- Sync with external APIs
- Track analytics events
- Update URL query params

**Example**: Sync to URL

```typescript
effect(() => {
  const filters = this.filterState();
  const query = new URLSearchParams(filters).toString();
  this.router.navigate([], { queryParams: { ...filters } });
});
```

---

## 🏪 NgRx SignalStore

### Basic Store

```typescript
import { signalStore, withState, withMethods, withComputed } from '@ngrx/signals';
import { computed } from '@angular/core';

interface TodosState {
  todos: Todo[];
  filter: 'all' | 'active' | 'completed';
}

export const TodosStore = signalStore(
  { providedIn: 'root' },  // or route-level

  // Define state
  withState<TodosState>({
    todos: [],
    filter: 'all',
  }),

  // Add computed signals
  withComputed((store) => ({
    filteredTodos: computed(() => {
      const todos = store.todos();
      const filter = store.filter();

      if (filter === 'active') return todos.filter(t => !t.completed);
      if (filter === 'completed') return todos.filter(t => t.completed);
      return todos;
    }),

    activeCount: computed(() =>
      store.todos().filter(t => !t.completed).length
    ),
  })),

  // Add methods
  withMethods((store) => ({
    addTodo(title: string) {
      const newTodo: Todo = {
        id: generateId(),
        title,
        completed: false,
      };

      patchState(store, (state) => ({
        todos: [...state.todos, newTodo],
      }));
    },

    toggleTodo(id: string) {
      patchState(store, (state) => ({
        todos: state.todos.map(t =>
          t.id === id ? { ...t, completed: !t.completed } : t
        ),
      }));
    },

    setFilter(filter: TodosState['filter']) {
      patchState(store, { filter });
    },
  })),
);
```

**Usage in Component**:

```typescript
@Component({
  selector: 'app-todos',
  standalone: true,
  template: `
    <div>
      <input #input (keyup.enter)="store.addTodo(input.value); input.value = ''">

      <div>
        @for (todo of store.filteredTodos(); track todo.id) {
          <div (click)="store.toggleTodo(todo.id)">
            {{ todo.title }} - {{ todo.completed ? 'Done' : 'Pending' }}
          </div>
        }
      </div>

      <div>Active: {{ store.activeCount() }}</div>

      <button (click)="store.setFilter('all')">All</button>
      <button (click)="store.setFilter('active')">Active</button>
      <button (click)="store.setFilter('completed')">Completed</button>
    </div>
  `,
})
export class TodosComponent {
  store = inject(TodosStore);
}
```

---

### Entity Management with SignalStore

For collections (files, users, datasets), use **entity patterns**:

```typescript
import { signalStore, withState, withMethods } from '@ngrx/signals';
import { addEntity, updateEntity, removeEntity, setAllEntities } from '@ngrx/signals/entities';

interface File {
  id: string;
  name: string;
  size: number;
  modifiedAt: Date;
}

interface FileManagerState {
  files: File[];
  selectedFileIds: string[];
  loading: boolean;
}

export const FileManagerStore = signalStore(
  withState<FileManagerState>({
    files: [],
    selectedFileIds: [],
    loading: false,
  }),

  withMethods((store, api = inject(FileApiService)) => ({
    async loadFiles() {
      patchState(store, { loading: true });

      try {
        const files = await api.getFiles();
        patchState(store, { files, loading: false });
      } catch (error) {
        patchState(store, { loading: false });
        // Handle error
      }
    },

    addFile(file: File) {
      patchState(store, (state) => ({
        files: [...state.files, file],
      }));
    },

    updateFile(id: string, changes: Partial<File>) {
      patchState(store, (state) => ({
        files: state.files.map(f =>
          f.id === id ? { ...f, ...changes } : f
        ),
      }));
    },

    deleteFile(id: string) {
      patchState(store, (state) => ({
        files: state.files.filter(f => f.id !== id),
        selectedFileIds: state.selectedFileIds.filter(sid => sid !== id),
      }));
    },

    selectFile(id: string) {
      patchState(store, (state) => ({
        selectedFileIds: state.selectedFileIds.includes(id)
          ? state.selectedFileIds.filter(sid => sid !== id)
          : [...state.selectedFileIds, id],
      }));
    },

    selectAll() {
      patchState(store, (state) => ({
        selectedFileIds: state.files.map(f => f.id),
      }));
    },

    clearSelection() {
      patchState(store, { selectedFileIds: [] });
    },
  })),

  withComputed((store) => ({
    selectedFiles: computed(() =>
      store.files().filter(f => store.selectedFileIds().includes(f.id))
    ),

    fileCount: computed(() => store.files().length),

    totalSize: computed(() =>
      store.files().reduce((sum, f) => sum + f.size, 0)
    ),
  })),
);
```

---

## 🔄 RxJS + Signals Integration

### When to Use RxJS

**Use RxJS for**:
- Async operations (HTTP, WebSockets, timers)
- Complex async workflows (debounce, throttle, retry, cancellation)
- Pagination cursors
- Real-time streams

**Use Signals for**:
- Synchronous state
- Derived state
- UI state (toggles, selections, filters)

### rxMethod() - Async Operations in SignalStore

```typescript
import { rxMethod } from '@ngrx/signals/rxjs-interop';
import { pipe, switchMap, tap, catchError, of } from 'rxjs';

export const DatasetStore = signalStore(
  withState<DatasetState>({ ... }),

  withMethods((store, api = inject(DatasetApiService)) => ({
    // Convert signal input to observable workflow
    loadDataset: rxMethod<string>(
      pipe(
        tap(() => patchState(store, { loading: true, error: null })),
        switchMap(datasetId =>
          api.getDataset(datasetId).pipe(
            tap(dataset => patchState(store, { dataset, loading: false })),
            catchError(error => {
              patchState(store, { loading: false, error: error.message });
              return of(null);
            })
          )
        )
      )
    ),

    // Debounced search
    search: rxMethod<string>(
      pipe(
        debounceTime(300),
        distinctUntilChanged(),
        tap(() => patchState(store, { searching: true })),
        switchMap(query =>
          api.search(query).pipe(
            tap(results => patchState(store, { searchResults: results, searching: false })),
            catchError(() => {
              patchState(store, { searching: false });
              return of([]);
            })
          )
        )
      )
    ),
  })),
);
```

**Usage**:

```typescript
@Component({
  template: `
    <input [ngModel]="searchQuery()" (ngModelChange)="onSearchChange($event)" />
  `
})
export class SearchComponent {
  store = inject(DatasetStore);
  searchQuery = signal('');

  constructor() {
    // Wire signal to rxMethod
    effect(() => {
      this.store.search(this.searchQuery());
    });
  }

  onSearchChange(query: string) {
    this.searchQuery.set(query);
    // Effect automatically triggers store.search() with debouncing
  }
}
```

### toSignal() - Observable to Signal

```typescript
import { toSignal } from '@angular/core/rxjs-interop';

@Component({
  template: `
    <div>User: {{ user()?.name }}</div>
  `
})
export class UserProfileComponent {
  private userService = inject(UserService);

  // Convert observable to signal
  user = toSignal(this.userService.currentUser$, {
    initialValue: null,
  });
}
```

### toObservable() - Signal to Observable

```typescript
import { toObservable } from '@angular/core/rxjs-interop';

@Component({...})
export class AutoSaveComponent {
  formData = signal({ name: '', email: '' });

  constructor() {
    // Convert signal changes to observable for debouncing
    toObservable(this.formData)
      .pipe(
        debounceTime(1000),
        distinctUntilChanged(),
        switchMap(data => this.api.save(data))
      )
      .subscribe();
  }
}
```

---

## 🏛️ State Architecture

### Three-Tier State Model

```
┌─────────────────────────────────────────────────┐
│ 1. GLOBAL STATE (App Shell)                    │
│    Scope: Entire app                            │
│    Provider: providedIn: 'root'                 │
│    Examples:                                    │
│    - Current tenant, user, permissions          │
│    - Workspace tabs, active tab                 │
│    - Global search index                        │
│    - Notifications, theme                       │
│    Store: AppShellStore                         │
├─────────────────────────────────────────────────┤
│ 2. FEATURE STATE (Module-Scoped)               │
│    Scope: Single feature module                 │
│    Provider: Route-level providers              │
│    Examples:                                    │
│    - File tree, selected files                  │
│    - Kanban boards, cards                       │
│    - Dataset rows, filters, sort                │
│    - Case list, selected case                   │
│    Stores: FileManagerStore, KanbanStore, etc.  │
├─────────────────────────────────────────────────┤
│ 3. COMPONENT STATE (Local)                     │
│    Scope: Single component                      │
│    Provider: Component-level signals            │
│    Examples:                                    │
│    - Modal open/closed                          │
│    - Form input values                          │
│    - Dropdown expanded state                    │
│    Tools: signal(), computed()                  │
└─────────────────────────────────────────────────┘
```

### Decision Tree: Where Should This State Live?

```
Is the state needed across multiple routes?
├─ YES → Global Store (AppShellStore)
└─ NO
   └─ Is the state needed across multiple components in a feature?
      ├─ YES → Feature Store (FileManagerStore, KanbanStore, etc.)
      └─ NO
         └─ Is the state pure UI state (modal open, hover)?
            ├─ YES → Component Signal
            └─ NO → Consider Feature Store for shared logic
```

**Examples**:

| State | Location | Why |
|-------|----------|-----|
| Current user | Global | Needed everywhere (permissions, avatar, audit) |
| Open tabs | Global | Persists across route changes |
| File tree structure | Feature (Files) | Only relevant in File Manager |
| Selected files | Feature (Files) | Might be used in multiple file manager components |
| "New Folder" modal open | Component | Pure UI state, single component |
| Dataset sort column | Feature (Datasets) | Persisted in URL, shared across grid components |
| Dropdown expanded | Component | Pure UI, ephemeral |

---

## 🗂️ Route-Level Providers

### Why Route-Level?

**Benefits**:
- **Scoped Lifecycle**: Store created when route activates, destroyed when leaving
- **Lazy Loading**: Store code only loads with the feature module
- **Clean Separation**: Each feature has isolated state
- **Testing**: Easy to provide mock stores per feature

### Implementation

```typescript
// features/files/files.routes.ts
export const filesRoutes: Routes = [
  {
    path: '',
    providers: [
      FileManagerStore,       // Feature store
      FileApiService,         // Feature service
      FileSearchService,
    ],
    children: [
      {
        path: '',
        loadComponent: () => import('./file-list.component'),
      },
      {
        path: 'file/:id',
        loadComponent: () => import('./file-detail.component'),
      },
    ],
  },
];
```

**Result**: All components under `/files` can `inject(FileManagerStore)`, and each gets the same instance. When navigating away, the store is destroyed.

### Sharing State Between Features

**Problem**: Dataset Explorer wants to open a file from Files module.

**Solutions**:

1. **Global Store for Shared Entities**
   ```typescript
   // Shared file model in global store
   export const AppShellStore = signalStore(
     withState({ recentFiles: [] }),
     withMethods((store) => ({
       addRecentFile(file: File) {
         patchState(store, (state) => ({
           recentFiles: [file, ...state.recentFiles].slice(0, 10),
         }));
       },
     })),
   );
   ```

2. **Service as Mediator**
   ```typescript
   @Injectable({ providedIn: 'root' })
   export class FileIntegrationService {
     openFile(fileId: string) {
       inject(Router).navigate(['/t/default/files/file', fileId]);
     }
   }
   ```

3. **Event Bus (for Loose Coupling)**
   ```typescript
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

   // Dataset module emits event
   eventBus.emit({ type: 'file:open', fileId: '123' });

   // File module listens
   eventBus.on('file:open').subscribe(({ fileId }) => {
     fileStore.loadFile(fileId);
   });
   ```

---

## 📋 Best Practices

### 1. Keep State Flat

**Bad** (nested objects):
```typescript
interface State {
  user: {
    profile: {
      settings: {
        theme: string;
      };
    };
  };
}

// Updating is painful
patchState(store, (state) => ({
  user: {
    ...state.user,
    profile: {
      ...state.user.profile,
      settings: {
        ...state.user.profile.settings,
        theme: 'dark',
      },
    },
  },
}));
```

**Good** (flat structure):
```typescript
interface State {
  userId: string;
  profileName: string;
  profileEmail: string;
  settingsTheme: string;
}

// Updating is simple
patchState(store, { settingsTheme: 'dark' });
```

### 2. Normalize Collections

**Bad** (array of objects):
```typescript
interface State {
  users: User[];  // Need to find by ID every time
}

const user = store.users().find(u => u.id === '123');
```

**Good** (normalized map):
```typescript
interface State {
  userIds: string[];
  userEntities: Record<string, User>;
}

const user = store.userEntities()['123'];  // O(1) lookup
```

**Helper Pattern**:
```typescript
withComputed((store) => ({
  users: computed(() =>
    store.userIds().map(id => store.userEntities()[id])
  ),

  getUserById: (id: string) => computed(() =>
    store.userEntities()[id]
  ),
})),
```

### 3. Use Computed for Filters/Sorts

**Bad** (filtering in template):
```html
@for (item of items().filter(i => i.active); track item.id) {
  <!-- Re-filters on every change detection! -->
}
```

**Good** (computed signal):
```typescript
activeItems = computed(() =>
  this.items().filter(i => i.active)
);
```

```html
@for (item of activeItems(); track item.id) {
  <!-- Only recomputes when items() changes -->
}
```

### 4. Immutable Updates

**Always return new objects/arrays**:

```typescript
// ✅ Good
patchState(store, (state) => ({
  items: [...state.items, newItem],
}));

// ❌ Bad (mutates!)
patchState(store, (state) => {
  state.items.push(newItem);  // Mutation! Signals won't detect change
  return state;
});
```

### 5. Avoid Effects for Derived State

**Bad** (manual synchronization):
```typescript
count = signal(0);
double = signal(0);

constructor() {
  effect(() => {
    this.double.set(this.count() * 2);  // Manual sync
  });
}
```

**Good** (computed):
```typescript
count = signal(0);
double = computed(() => this.count() * 2);  // Auto-synced
```

### 6. Limit effect() Use

**Use `effect()` for**:
- Logging, analytics
- Persistence (localStorage, IndexedDB)
- DOM manipulation
- External API sync

**Don't use `effect()` for**:
- Deriving state (use `computed()`)
- Updating other signals (use `computed()`)
- Business logic (use methods)

### 7. Signal Naming Conventions

```typescript
// Signals: noun
count = signal(0);
selectedFile = signal<File | null>(null);

// Computed: adjective or "computed" prefix
isLoading = computed(() => this.status() === 'loading');
filteredItems = computed(() => ...);

// Methods: verb
increment() { ... }
selectFile(file: File) { ... }
```

---

## 🧪 Testing State

### Testing SignalStore

```typescript
describe('TodosStore', () => {
  let store: InstanceType<typeof TodosStore>;

  beforeEach(() => {
    TestBed.configureTestingModule({
      providers: [TodosStore],
    });

    store = TestBed.inject(TodosStore);
  });

  it('should add todo', () => {
    store.addTodo('Buy milk');

    expect(store.todos()).toHaveLength(1);
    expect(store.todos()[0].title).toBe('Buy milk');
  });

  it('should toggle todo', () => {
    store.addTodo('Buy milk');
    const todoId = store.todos()[0].id;

    store.toggleTodo(todoId);

    expect(store.todos()[0].completed).toBe(true);
  });

  it('should filter todos', () => {
    store.addTodo('Task 1');
    store.addTodo('Task 2');
    store.toggleTodo(store.todos()[0].id);

    store.setFilter('completed');

    expect(store.filteredTodos()).toHaveLength(1);
    expect(store.filteredTodos()[0].title).toBe('Task 1');
  });

  it('should compute active count', () => {
    store.addTodo('Task 1');
    store.addTodo('Task 2');
    store.toggleTodo(store.todos()[0].id);

    expect(store.activeCount()).toBe(1);
  });
});
```

### Testing Computed Signals

```typescript
it('should update computed when dependency changes', () => {
  const count = signal(1);
  const double = computed(() => count() * 2);

  expect(double()).toBe(2);

  count.set(5);

  expect(double()).toBe(10);
});
```

### Testing Effects

```typescript
it('should run effect when signal changes', () => {
  const spy = jest.fn();
  const count = signal(0);

  TestBed.runInInjectionContext(() => {
    effect(() => {
      spy(count());
    });
  });

  expect(spy).toHaveBeenCalledWith(0);

  count.set(5);

  expect(spy).toHaveBeenCalledWith(5);
});
```

---

## 🚫 Anti-Patterns to Avoid

### 1. Signal in Signal

**Bad**:
```typescript
const nested = signal(signal(0));  // Don't nest signals!
```

**Why**: Signals should hold values, not other signals. Use computed for derivation.

### 2. Mutating Signal Values

**Bad**:
```typescript
const items = signal([1, 2, 3]);
items().push(4);  // Mutation! Signal won't detect change
```

**Good**:
```typescript
items.update(arr => [...arr, 4]);  // New array
```

### 3. Effect Hell (Effects Updating Signals)

**Bad**:
```typescript
count = signal(0);
double = signal(0);

constructor() {
  effect(() => {
    this.double.set(this.count() * 2);  // Effect updating signal
  });
}
```

**Why**: Leads to circular dependencies and hard-to-debug issues.

**Good**:
```typescript
count = signal(0);
double = computed(() => this.count() * 2);
```

### 4. Overusing Global Store

**Bad**: Putting everything in AppShellStore

**Why**: Global state is hard to reason about and doesn't benefit from lazy loading.

**Good**: Use feature stores for feature-specific state, global only for truly global concerns.

### 5. Signals in Services (Usually)

**Bad**:
```typescript
@Injectable({ providedIn: 'root' })
export class UserService {
  currentUser = signal<User | null>(null);  // Service with signal
}
```

**Why**: Harder to test, harder to replace, mixes concerns.

**Good**: Use SignalStore instead of services with signals, or keep services stateless.

---

## ✅ State Management Checklist

**Fundamentals**:
- ✅ Understand `signal()`, `computed()`, `effect()`
- ✅ Know when to use each (signal for state, computed for derivation, effect for side effects)
- ✅ Immutable updates (always return new objects/arrays)

**SignalStore**:
- ✅ Use `withState()` for state shape
- ✅ Use `withComputed()` for derived state
- ✅ Use `withMethods()` for state updates
- ✅ Use `rxMethod()` for async operations

**Architecture**:
- ✅ Three-tier state (global, feature, component)
- ✅ Route-level providers for feature stores
- ✅ Flat state structure (avoid deep nesting)
- ✅ Normalized collections (entities as maps)

**RxJS Integration**:
- ✅ Use RxJS for async workflows (HTTP, debouncing, streams)
- ✅ Use `rxMethod()` to bridge signals and observables
- ✅ Use `toSignal()` to convert observables to signals
- ✅ Use `toObservable()` to convert signals to observables

**Best Practices**:
- ✅ Computed for filters/sorts (not in template)
- ✅ Limit effect() to side effects only
- ✅ Consistent naming (signals as nouns, computed as adjectives)
- ✅ Test stores, computed, effects

**Anti-Patterns**:
- ✅ Avoid nested signals
- ✅ Avoid mutating signal values
- ✅ Avoid effect hell (effects updating signals)
- ✅ Avoid overusing global store

---

**Next**: UI_SHELL.md (Component implementation details)
