# Testing Strategy

> Comprehensive testing approach for Nexus Studio's Signal-based architecture

## Table of Contents
1. [Testing Pyramid](#testing-pyramid)
2. [Unit Testing Signals](#unit-testing-signals)
3. [Component Testing](#component-testing)
4. [Testing SignalStore](#testing-signalstore)
5. [Integration Testing](#integration-testing)
6. [E2E Testing](#e2e-testing)
7. [Testing Web Workers](#testing-web-workers)
8. [Accessibility Testing](#accessibility-testing)
9. [Performance Testing](#performance-testing)

---

## Testing Pyramid

### What: Balance unit, integration, and E2E tests
**Why**: Fast feedback (unit), confidence (integration), real-user validation (E2E)

```
         /\
        /  \  E2E (10%)
       /    \  - Critical user flows
      /------\  - Smoke tests
     /        \
    / Integration \ (30%)
   /    Tests      \  - Feature modules
  /                 \  - Store + Service + Component
 /-------------------\
/   Unit Tests (60%)  \
-------------------------
  - Signals, Computed, Effects
  - Services, Utilities
  - Isolated components
```

**Nexus Studio Distribution**:
- **Unit tests**: 300+ tests, <5s total runtime
- **Integration tests**: 50+ tests, <30s runtime
- **E2E tests**: 20+ critical flows, <5min runtime

---

## Unit Testing Signals

### What: Test signal logic in isolation
**Why**: Fastest feedback, easiest to debug

### Testing Signal State

```typescript
// files/stores/file-manager.store.spec.ts
import { TestBed } from '@angular/core/testing';
import { FileManagerStore } from './file-manager.store';

describe('FileManagerStore', () => {
  let store: InstanceType<typeof FileManagerStore>;

  beforeEach(() => {
    TestBed.configureTestingModule({
      providers: [FileManagerStore],
    });
    store = TestBed.inject(FileManagerStore);
  });

  it('should initialize with empty state', () => {
    expect(store.files()).toEqual([]);
    expect(store.selectedFileIds()).toEqual([]);
    expect(store.isLoading()).toBe(false);
  });

  it('should add files', () => {
    const files = [
      { id: '1', name: 'file1.txt', type: 'file' } as FileNode,
      { id: '2', name: 'file2.txt', type: 'file' } as FileNode,
    ];

    store.setFiles(files);

    expect(store.files()).toEqual(files);
    expect(store.files().length).toBe(2);
  });

  it('should select files', () => {
    store.setFiles([
      { id: '1', name: 'file1.txt' } as FileNode,
      { id: '2', name: 'file2.txt' } as FileNode,
    ]);

    store.selectFile('1');
    expect(store.selectedFileIds()).toEqual(['1']);

    store.selectFile('2');
    expect(store.selectedFileIds()).toEqual(['1', '2']);
  });

  it('should compute selected files correctly', () => {
    const files = [
      { id: '1', name: 'file1.txt' } as FileNode,
      { id: '2', name: 'file2.txt' } as FileNode,
      { id: '3', name: 'file3.txt' } as FileNode,
    ];
    store.setFiles(files);
    store.selectFile('1');
    store.selectFile('3');

    const selected = store.selectedFiles();

    expect(selected).toEqual([files[0], files[2]]);
  });
});
```

### Testing Computed Signals

```typescript
// design-system/stores/theme.store.spec.ts
import { ThemeStore } from './theme.store';

describe('ThemeStore - Computed Signals', () => {
  let store: InstanceType<typeof ThemeStore>;

  beforeEach(() => {
    store = TestBed.inject(ThemeStore);
  });

  it('should compute CSS variables from tokens', () => {
    store.setTokens({
      'color-primary': '#007bff',
      'color-secondary': '#6c757d',
      'spacing-base': '8px',
    });

    const cssVars = store.cssVariables();

    expect(cssVars).toEqual({
      '--color-primary': '#007bff',
      '--color-secondary': '#6c757d',
      '--spacing-base': '8px',
    });
  });

  it('should recompute when tokens change', () => {
    store.setTokens({ 'color-primary': '#007bff' });
    expect(store.cssVariables()['--color-primary']).toBe('#007bff');

    store.updateToken('color-primary', '#ff0000');
    expect(store.cssVariables()['--color-primary']).toBe('#ff0000');
  });

  it('should filter tokens by category', () => {
    store.setTokens({
      'color-primary': '#007bff',
      'color-secondary': '#6c757d',
      'spacing-base': '8px',
      'spacing-large': '16px',
    });

    const colorTokens = store.tokensByCategory('color');

    expect(colorTokens).toEqual({
      'color-primary': '#007bff',
      'color-secondary': '#6c757d',
    });
  });
});
```

### Testing Effects

```typescript
// core/services/activity-logger.service.spec.ts
import { TestBed } from '@angular/core/testing';
import { signal } from '@angular/core';
import { ActivityLoggerService } from './activity-logger.service';

describe('ActivityLoggerService - Effects', () => {
  let service: ActivityLoggerService;
  let consoleSpy: jasmine.Spy;

  beforeEach(() => {
    TestBed.configureTestingModule({
      providers: [ActivityLoggerService],
    });
    service = TestBed.inject(ActivityLoggerService);
    consoleSpy = spyOn(console, 'log');
  });

  it('should log when activity changes', () => {
    const activitySignal = signal('idle');

    // Effect runs when signal changes
    service.watchActivity(activitySignal);

    activitySignal.set('active');
    TestBed.flushEffects(); // Trigger effect execution

    expect(consoleSpy).toHaveBeenCalledWith('Activity changed to: active');
  });
});
```

---

## Component Testing

### What: Test components with Signal inputs/outputs
**Why**: Verify component behavior with realistic interactions

### Testing Signal Inputs

```typescript
// files/components/file-card/file-card.component.spec.ts
import { ComponentFixture, TestBed } from '@angular/core/testing';
import { FileCardComponent } from './file-card.component';
import { signal } from '@angular/core';

describe('FileCardComponent', () => {
  let component: FileCardComponent;
  let fixture: ComponentFixture<FileCardComponent>;

  beforeEach(async () => {
    await TestBed.configureTestingModule({
      imports: [FileCardComponent],
    }).compileComponents();

    fixture = TestBed.createComponent(FileCardComponent);
    component = fixture.componentInstance;
  });

  it('should display file name', () => {
    const file = { id: '1', name: 'test.txt', type: 'file' } as FileNode;

    // Set signal input
    fixture.componentRef.setInput('file', file);
    fixture.detectChanges();

    const nameElement = fixture.nativeElement.querySelector('.file-name');
    expect(nameElement.textContent).toContain('test.txt');
  });

  it('should display file icon based on type', () => {
    const file = { id: '1', name: 'doc.pdf', mimeType: 'application/pdf' } as FileNode;

    fixture.componentRef.setInput('file', file);
    fixture.detectChanges();

    const icon = fixture.nativeElement.querySelector('mat-icon');
    expect(icon.textContent).toBe('picture_as_pdf');
  });

  it('should emit click event', () => {
    const file = { id: '1', name: 'test.txt' } as FileNode;
    const clickSpy = jasmine.createSpy('onClick');

    fixture.componentRef.setInput('file', file);
    component.fileClick.subscribe(clickSpy);

    fixture.nativeElement.querySelector('.file-card').click();

    expect(clickSpy).toHaveBeenCalledWith(file);
  });
});
```

### Testing User Interactions

```typescript
// kanban/components/kanban-card/kanban-card.component.spec.ts
import { ComponentFixture, TestBed } from '@angular/core/testing';
import { KanbanCardComponent } from './kanban-card.component';
import { DragDropModule } from '@angular/cdk/drag-drop';

describe('KanbanCardComponent - Interactions', () => {
  let component: KanbanCardComponent;
  let fixture: ComponentFixture<KanbanCardComponent>;

  beforeEach(async () => {
    await TestBed.configureTestingModule({
      imports: [KanbanCardComponent, DragDropModule],
    }).compileComponents();

    fixture = TestBed.createComponent(KanbanCardComponent);
    component = fixture.componentInstance;
  });

  it('should toggle expanded state on click', () => {
    const card = { id: '1', title: 'Task 1', description: 'Details...' } as KanbanCard;
    fixture.componentRef.setInput('card', card);
    fixture.detectChanges();

    expect(component.isExpanded()).toBe(false);

    const expandButton = fixture.nativeElement.querySelector('.expand-button');
    expandButton.click();
    fixture.detectChanges();

    expect(component.isExpanded()).toBe(true);

    const description = fixture.nativeElement.querySelector('.card-description');
    expect(description).toBeTruthy();
    expect(description.textContent).toContain('Details...');
  });

  it('should emit delete event', () => {
    const card = { id: '1', title: 'Task 1' } as KanbanCard;
    const deleteSpy = jasmine.createSpy('onDelete');

    fixture.componentRef.setInput('card', card);
    component.cardDelete.subscribe(deleteSpy);
    fixture.detectChanges();

    const deleteButton = fixture.nativeElement.querySelector('.delete-button');
    deleteButton.click();

    expect(deleteSpy).toHaveBeenCalledWith(card.id);
  });
});
```

### Testing Async Operations

```typescript
// datasets/components/dataset-profiler/dataset-profiler.component.spec.ts
import { ComponentFixture, TestBed, fakeAsync, tick } from '@angular/core/testing';
import { DatasetProfilerComponent } from './dataset-profiler.component';
import { DatasetService } from '../../services/dataset.service';

describe('DatasetProfilerComponent - Async', () => {
  let component: DatasetProfilerComponent;
  let fixture: ComponentFixture<DatasetProfilerComponent>;
  let datasetService: jasmine.SpyObj<DatasetService>;

  beforeEach(async () => {
    const serviceSpy = jasmine.createSpyObj('DatasetService', ['profileDataset']);

    await TestBed.configureTestingModule({
      imports: [DatasetProfilerComponent],
      providers: [{ provide: DatasetService, useValue: serviceSpy }],
    }).compileComponents();

    fixture = TestBed.createComponent(DatasetProfilerComponent);
    component = fixture.componentInstance;
    datasetService = TestBed.inject(DatasetService) as jasmine.SpyObj<DatasetService>;
  });

  it('should show loading state while profiling', fakeAsync(() => {
    const mockStats = { columns: [], rowCount: 1000 };
    datasetService.profileDataset.and.returnValue(
      new Promise(resolve => setTimeout(() => resolve(mockStats), 1000))
    );

    component.startProfiling();
    fixture.detectChanges();

    expect(component.isLoading()).toBe(true);
    expect(fixture.nativeElement.querySelector('mat-spinner')).toBeTruthy();

    tick(1000); // Fast-forward time
    fixture.detectChanges();

    expect(component.isLoading()).toBe(false);
    expect(component.stats()).toEqual(mockStats);
  }));

  it('should handle profiling errors', fakeAsync(() => {
    datasetService.profileDataset.and.returnValue(
      Promise.reject(new Error('Profiling failed'))
    );

    component.startProfiling();
    tick();
    fixture.detectChanges();

    expect(component.error()).toBe('Profiling failed');
    expect(fixture.nativeElement.querySelector('.error-message')).toBeTruthy();
  }));
});
```

---

## Testing SignalStore

### What: Test entire store with methods, computed signals, and side effects
**Why**: Ensures state management logic works correctly

### Complete Store Test Suite

```typescript
// kanban/stores/kanban.store.spec.ts
import { TestBed } from '@angular/core/testing';
import { KanbanStore } from './kanban.store';
import { KanbanService } from '../services/kanban.service';
import { of, throwError } from 'rxjs';

describe('KanbanStore', () => {
  let store: InstanceType<typeof KanbanStore>;
  let kanbanService: jasmine.SpyObj<KanbanService>;

  beforeEach(() => {
    const serviceSpy = jasmine.createSpyObj('KanbanService', [
      'getBoards',
      'createCard',
      'moveCard',
      'deleteCard',
    ]);

    TestBed.configureTestingModule({
      providers: [
        KanbanStore,
        { provide: KanbanService, useValue: serviceSpy },
      ],
    });

    store = TestBed.inject(KanbanStore);
    kanbanService = TestBed.inject(KanbanService) as jasmine.SpyObj<KanbanService>;
  });

  describe('initialization', () => {
    it('should start with empty state', () => {
      expect(store.cards()).toEqual([]);
      expect(store.columns()).toEqual([]);
      expect(store.isLoading()).toBe(false);
    });
  });

  describe('loadBoards', () => {
    it('should load boards successfully', async () => {
      const mockBoards = [
        { id: '1', name: 'Board 1', columns: [], cards: [] },
      ];
      kanbanService.getBoards.and.returnValue(of(mockBoards));

      await store.loadBoards();

      expect(store.isLoading()).toBe(false);
      expect(store.columns().length).toBeGreaterThan(0);
    });

    it('should set error on load failure', async () => {
      kanbanService.getBoards.and.returnValue(
        throwError(() => new Error('Network error'))
      );

      await store.loadBoards();

      expect(store.error()).toBe('Failed to load boards');
    });
  });

  describe('createCard', () => {
    it('should add card optimistically', async () => {
      const newCard = { title: 'New Task', columnId: 'col-1' };
      kanbanService.createCard.and.returnValue(
        of({ id: 'card-1', ...newCard } as KanbanCard)
      );

      await store.createCard(newCard);

      const cards = store.cards();
      expect(cards.some(c => c.title === 'New Task')).toBe(true);
    });

    it('should rollback on failure', async () => {
      const initialCards = store.cards();
      kanbanService.createCard.and.returnValue(
        throwError(() => new Error('Create failed'))
      );

      await store.createCard({ title: 'Task', columnId: 'col-1' });

      expect(store.cards()).toEqual(initialCards); // Reverted
      expect(store.error()).toBe('Failed to create card');
    });
  });

  describe('moveCard', () => {
    beforeEach(() => {
      store.setCards([
        { id: 'card-1', columnId: 'col-1', position: 0 } as KanbanCard,
        { id: 'card-2', columnId: 'col-1', position: 1 } as KanbanCard,
      ]);
      store.setColumns([
        { id: 'col-1', name: 'To Do' } as KanbanColumn,
        { id: 'col-2', name: 'In Progress' } as KanbanColumn,
      ]);
    });

    it('should move card between columns', async () => {
      kanbanService.moveCard.and.returnValue(of({}));

      await store.moveCard('card-1', 'col-2', 0);

      const card = store.cards().find(c => c.id === 'card-1');
      expect(card?.columnId).toBe('col-2');
    });

    it('should update computed cardsByColumn', async () => {
      kanbanService.moveCard.and.returnValue(of({}));

      await store.moveCard('card-1', 'col-2', 0);

      const col2Cards = store.cardsByColumn()['col-2'];
      expect(col2Cards.some(c => c.id === 'card-1')).toBe(true);
    });
  });

  describe('computed signals', () => {
    it('should compute cardsByColumn correctly', () => {
      store.setCards([
        { id: '1', columnId: 'col-1' } as KanbanCard,
        { id: '2', columnId: 'col-1' } as KanbanCard,
        { id: '3', columnId: 'col-2' } as KanbanCard,
      ]);

      const byColumn = store.cardsByColumn();

      expect(byColumn['col-1'].length).toBe(2);
      expect(byColumn['col-2'].length).toBe(1);
    });

    it('should filter cards by search query', () => {
      store.setCards([
        { id: '1', title: 'Fix bug in login' } as KanbanCard,
        { id: '2', title: 'Add dark mode' } as KanbanCard,
        { id: '3', title: 'Fix bug in signup' } as KanbanCard,
      ]);

      store.setSearchQuery('bug');

      const filtered = store.filteredCards();
      expect(filtered.length).toBe(2);
      expect(filtered.every(c => c.title.includes('bug'))).toBe(true);
    });
  });
});
```

---

## Integration Testing

### What: Test feature modules with stores, services, and components together
**Why**: Catch issues in how pieces interact

### Feature Module Integration Test

```typescript
// files/file-manager.integration.spec.ts
import { ComponentFixture, TestBed } from '@angular/core/testing';
import { FileManagerShellComponent } from './components/file-manager-shell/file-manager-shell.component';
import { FileManagerStore } from './stores/file-manager.store';
import { FileService } from './services/file.service';
import { provideHttpClient } from '@angular/common/http';
import { provideHttpClientTesting, HttpTestingController } from '@angular/common/http/testing';

describe('File Manager Integration', () => {
  let component: FileManagerShellComponent;
  let fixture: ComponentFixture<FileManagerShellComponent>;
  let store: InstanceType<typeof FileManagerStore>;
  let httpMock: HttpTestingController;

  beforeEach(async () => {
    await TestBed.configureTestingModule({
      imports: [FileManagerShellComponent],
      providers: [
        FileManagerStore,
        FileService,
        provideHttpClient(),
        provideHttpClientTesting(),
      ],
    }).compileComponents();

    fixture = TestBed.createComponent(FileManagerShellComponent);
    component = fixture.componentInstance;
    store = TestBed.inject(FileManagerStore);
    httpMock = TestBed.inject(HttpTestingController);
  });

  afterEach(() => {
    httpMock.verify(); // Ensure no outstanding requests
  });

  it('should load and display files on init', () => {
    fixture.detectChanges(); // Triggers ngOnInit

    const req = httpMock.expectOne('/api/files?tenantId=test-tenant');
    expect(req.request.method).toBe('GET');

    req.flush([
      { id: '1', name: 'file1.txt', type: 'file' },
      { id: '2', name: 'file2.txt', type: 'file' },
    ]);

    fixture.detectChanges();

    const fileCards = fixture.nativeElement.querySelectorAll('.file-card');
    expect(fileCards.length).toBe(2);
  });

  it('should create new file and update UI', async () => {
    const newFile = { name: 'new-file.txt', type: 'file', parentId: 'root' };

    component.createFile(newFile);

    const req = httpMock.expectOne('/api/files');
    expect(req.request.method).toBe('POST');
    expect(req.request.body).toEqual(newFile);

    req.flush({ id: '3', ...newFile });

    fixture.detectChanges();

    expect(store.files().some(f => f.name === 'new-file.txt')).toBe(true);
  });

  it('should handle file selection and update UI', () => {
    store.setFiles([
      { id: '1', name: 'file1.txt' } as FileNode,
      { id: '2', name: 'file2.txt' } as FileNode,
    ]);
    fixture.detectChanges();

    const firstCard = fixture.nativeElement.querySelector('.file-card');
    firstCard.click();
    fixture.detectChanges();

    expect(firstCard.classList.contains('selected')).toBe(true);
    expect(store.selectedFileIds()).toContain('1');
  });
});
```

---

## E2E Testing

### What: Test complete user flows across modules
**Why**: Validate real-world scenarios work end-to-end

### Playwright E2E Tests

```typescript
// e2e/file-manager.spec.ts
import { test, expect } from '@playwright/test';

test.describe('File Manager', () => {
  test.beforeEach(async ({ page }) => {
    await page.goto('http://localhost:4200/t/test-tenant/files');
    await page.waitForSelector('.file-manager-shell');
  });

  test('should upload and display file', async ({ page }) => {
    // Click upload button
    await page.click('[aria-label="Upload file"]');

    // Upload file
    const fileInput = page.locator('input[type="file"]');
    await fileInput.setInputFiles('./test-fixtures/sample.pdf');

    // Wait for upload to complete
    await page.waitForSelector('.file-card:has-text("sample.pdf")');

    // Verify file appears in list
    const fileCard = page.locator('.file-card:has-text("sample.pdf")');
    await expect(fileCard).toBeVisible();
  });

  test('should navigate folder hierarchy', async ({ page }) => {
    // Expand folder
    await page.click('.folder-node:has-text("Documents") .expand-icon');

    // Wait for children to load
    await page.waitForSelector('.folder-node[data-level="1"]');

    // Click subfolder
    await page.click('.folder-node:has-text("Projects")');

    // Verify breadcrumb
    const breadcrumb = page.locator('.breadcrumb');
    await expect(breadcrumb).toContainText('Documents / Projects');
  });

  test('should search files globally', async ({ page }) => {
    // Type in search box
    await page.fill('[aria-label="Search files"]', 'invoice');

    // Wait for search results
    await page.waitForSelector('.search-results');

    // Verify results
    const results = page.locator('.search-result-item');
    const count = await results.count();
    expect(count).toBeGreaterThan(0);

    // Each result should contain "invoice"
    for (let i = 0; i < count; i++) {
      const text = await results.nth(i).textContent();
      expect(text?.toLowerCase()).toContain('invoice');
    }
  });

  test('should drag and drop file to folder', async ({ page }) => {
    // Get file and target folder
    const file = page.locator('.file-card:has-text("report.pdf")');
    const folder = page.locator('.folder-node:has-text("Archive")');

    // Drag file to folder
    await file.dragTo(folder);

    // Wait for move to complete
    await page.waitForTimeout(500);

    // Verify file moved
    await page.click('.folder-node:has-text("Archive")');
    await expect(page.locator('.file-card:has-text("report.pdf")')).toBeVisible();
  });
});

test.describe('Cross-Module Integration', () => {
  test('should attach file to case from File Manager', async ({ page }) => {
    // Navigate to File Manager
    await page.goto('http://localhost:4200/t/test-tenant/files');

    // Select file
    await page.click('.file-card:has-text("evidence.pdf")');

    // Right-click context menu
    await page.click('.file-card:has-text("evidence.pdf")', { button: 'right' });

    // Click "Attach to Case"
    await page.click('[aria-label="Attach to Case"]');

    // Select case from dialog
    await page.fill('[aria-label="Search cases"]', 'Case #12345');
    await page.click('.case-result:has-text("#12345")');
    await page.click('button:has-text("Attach")');

    // Navigate to Case Triage
    await page.goto('http://localhost:4200/t/test-tenant/cases/12345');

    // Verify file is attached
    await expect(page.locator('.attachment:has-text("evidence.pdf")')).toBeVisible();
  });
});
```

---

## Testing Web Workers

### What: Test Web Worker logic independently
**Why**: Workers run in separate thread, need special setup

### Testing Search Worker

```typescript
// core/workers/search-indexer.worker.spec.ts
describe('SearchIndexerWorker', () => {
  let worker: Worker;

  beforeEach(() => {
    worker = new Worker(new URL('./search-indexer.worker', import.meta.url));
  });

  afterEach(() => {
    worker.terminate();
  });

  it('should index documents', (done) => {
    const documents = [
      { id: '1', title: 'Angular Guide', content: 'Learn Angular signals' },
      { id: '2', title: 'React Tutorial', content: 'Learn React hooks' },
    ];

    worker.onmessage = (event) => {
      if (event.data.type === 'indexed') {
        expect(event.data.payload.count).toBe(2);
        done();
      }
    };

    worker.postMessage({ type: 'index', payload: { documents } });
  });

  it('should search documents', (done) => {
    const documents = [
      { id: '1', title: 'Angular Signals', content: 'Reactive state management' },
      { id: '2', title: 'Vue Composition', content: 'Vue reactive system' },
      { id: '3', title: 'React Hooks', content: 'React state management' },
    ];

    // Index first
    worker.postMessage({ type: 'index', payload: { documents } });

    // Then search
    worker.onmessage = (event) => {
      if (event.data.type === 'results') {
        const results = event.data.payload.results;

        // Should find documents with "reactive"
        expect(results.length).toBe(2);
        expect(results.some((r: any) => r.id === '1')).toBe(true);
        expect(results.some((r: any) => r.id === '2')).toBe(true);

        done();
      }
    };

    worker.postMessage({ type: 'search', payload: { query: 'reactive', limit: 10 } });
  });
});
```

---

## Accessibility Testing

### What: Automated + manual accessibility testing
**Why**: Ensure WCAG AAA compliance

### Automated A11y Testing

```typescript
// files/components/file-tree/file-tree.component.spec.ts
import { ComponentFixture, TestBed } from '@angular/core/testing';
import { FileTreeComponent } from './file-tree.component';
import { axe, toHaveNoViolations } from 'jasmine-axe';

expect.extend(toHaveNoViolations);

describe('FileTreeComponent - Accessibility', () => {
  let component: FileTreeComponent;
  let fixture: ComponentFixture<FileTreeComponent>;

  beforeEach(async () => {
    await TestBed.configureTestingModule({
      imports: [FileTreeComponent],
    }).compileComponents();

    fixture = TestBed.createComponent(FileTreeComponent);
    component = fixture.componentInstance;
  });

  it('should have no accessibility violations', async () => {
    fixture.componentRef.setInput('nodes', [
      { id: '1', name: 'Folder 1', type: 'folder', children: [] },
      { id: '2', name: 'File 1', type: 'file' },
    ]);
    fixture.detectChanges();

    const results = await axe(fixture.nativeElement);
    expect(results).toHaveNoViolations();
  });

  it('should have proper ARIA attributes', () => {
    fixture.detectChanges();

    const tree = fixture.nativeElement.querySelector('[role="tree"]');
    expect(tree).toBeTruthy();

    const treeItems = fixture.nativeElement.querySelectorAll('[role="treeitem"]');
    expect(treeItems.length).toBeGreaterThan(0);
  });

  it('should support keyboard navigation', () => {
    fixture.componentRef.setInput('nodes', [
      { id: '1', name: 'Item 1', type: 'file' },
      { id: '2', name: 'Item 2', type: 'file' },
    ]);
    fixture.detectChanges();

    const tree = fixture.nativeElement.querySelector('[role="tree"]');

    // ArrowDown should move focus
    tree.dispatchEvent(new KeyboardEvent('keydown', { key: 'ArrowDown' }));
    fixture.detectChanges();

    const focusedItem = fixture.nativeElement.querySelector('[aria-selected="true"]');
    expect(focusedItem).toBeTruthy();
  });
});
```

### E2E Accessibility Testing

```typescript
// e2e/accessibility.spec.ts
import { test, expect } from '@playwright/test';
import AxeBuilder from '@axe-core/playwright';

test.describe('Accessibility', () => {
  test('File Manager should be accessible', async ({ page }) => {
    await page.goto('http://localhost:4200/t/test-tenant/files');

    const accessibilityScanResults = await new AxeBuilder({ page })
      .withTags(['wcag2a', 'wcag2aa', 'wcag21aa'])
      .analyze();

    expect(accessibilityScanResults.violations).toEqual([]);
  });

  test('should navigate with keyboard only', async ({ page }) => {
    await page.goto('http://localhost:4200/t/test-tenant/files');

    // Tab through interface
    await page.keyboard.press('Tab'); // Focus on first element
    await page.keyboard.press('Tab'); // Next element

    // Verify focus visible
    const focusedElement = await page.evaluate(() => document.activeElement?.tagName);
    expect(focusedElement).toBeTruthy();

    // Test keyboard shortcuts
    await page.keyboard.press('Control+F'); // Open search
    await expect(page.locator('[aria-label="Search files"]')).toBeFocused();
  });
});
```

---

## Performance Testing

### What: Measure rendering performance and memory usage
**Why**: Ensure app stays fast with large datasets

### Performance Benchmarks

```typescript
// files/components/file-tree-virtual/file-tree-virtual.component.perf.spec.ts
import { ComponentFixture, TestBed } from '@angular/core/testing';
import { FileTreeVirtualComponent } from './file-tree-virtual.component';
import { generateMockFiles } from '../../testing/mock-data';

describe('FileTreeVirtualComponent - Performance', () => {
  let component: FileTreeVirtualComponent;
  let fixture: ComponentFixture<FileTreeVirtualComponent>;

  beforeEach(async () => {
    await TestBed.configureTestingModule({
      imports: [FileTreeVirtualComponent],
    }).compileComponents();

    fixture = TestBed.createComponent(FileTreeVirtualComponent);
    component = fixture.componentInstance;
  });

  it('should render 50k nodes in <200ms', () => {
    const nodes = generateMockFiles(50000);

    const startTime = performance.now();

    fixture.componentRef.setInput('nodes', nodes);
    fixture.detectChanges();

    const endTime = performance.now();
    const renderTime = endTime - startTime;

    console.log(`Rendered 50k nodes in ${renderTime}ms`);
    expect(renderTime).toBeLessThan(200);
  });

  it('should scroll at 60fps', async () => {
    const nodes = generateMockFiles(10000);
    fixture.componentRef.setInput('nodes', nodes);
    fixture.detectChanges();

    const viewport = fixture.nativeElement.querySelector('cdk-virtual-scroll-viewport');

    const frameTimings: number[] = [];
    let lastTime = performance.now();

    // Simulate scroll
    for (let i = 0; i < 60; i++) {
      viewport.scrollTop += 100;

      const currentTime = performance.now();
      frameTimings.push(currentTime - lastTime);
      lastTime = currentTime;

      await new Promise(resolve => requestAnimationFrame(resolve));
    }

    const avgFrameTime = frameTimings.reduce((a, b) => a + b) / frameTimings.length;
    const fps = 1000 / avgFrameTime;

    console.log(`Average FPS: ${fps}`);
    expect(fps).toBeGreaterThan(55); // Allow small variance from 60fps
  });
});
```

---

## Summary: Testing Checklist

### Before Committing
- [ ] All new code has unit tests (>80% coverage)
- [ ] Components tested with Signal inputs
- [ ] SignalStores tested (state + computed + methods)
- [ ] Accessibility: No axe violations
- [ ] Keyboard navigation works

### Before Releasing
- [ ] Integration tests pass for all feature modules
- [ ] E2E tests pass for critical user flows
- [ ] Performance benchmarks meet targets (see PERFORMANCE.md)
- [ ] Manual accessibility testing with screen reader
- [ ] Cross-browser testing (Chrome, Firefox, Safari, Edge)

### CI/CD Pipeline
```yaml
# .github/workflows/test.yml
name: Test
on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: 20

      - run: npm ci
      - run: npm run test -- --watch=false --code-coverage
      - run: npm run test:a11y
      - run: npm run build

      - name: E2E Tests
        run: |
          npm run start &
          npx wait-on http://localhost:4200
          npm run e2e

      - name: Upload Coverage
        uses: codecov/codecov-action@v3
```

**Result**: Comprehensive test suite ensures Nexus Studio is reliable, accessible, and performant.
