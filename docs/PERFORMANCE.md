# Performance Optimization Guide

> How Nexus Studio delivers instant interactions with massive datasets (50k-100k items)

## Table of Contents
1. [Virtual Scrolling Strategy](#virtual-scrolling-strategy)
2. [Change Detection Optimization](#change-detection-optimization)
3. [Lazy Loading Patterns](#lazy-loading-patterns)
4. [Rendering Boundaries](#rendering-boundaries)
5. [Web Workers for Heavy Computation](#web-workers-for-heavy-computation)
6. [IndexedDB Optimization](#indexeddb-optimization)
7. [Bundle Size Optimization](#bundle-size-optimization)
8. [Performance Monitoring](#performance-monitoring)
9. [Performance Budget](#performance-budget)

---

## Virtual Scrolling Strategy

### What: Rendering only visible items
**Why**: Rendering 100k DOM nodes freezes the browser. Virtual scrolling renders only ~30 visible items.
**Where**: File Manager, Log Viewer, Dataset Explorer, Case Triage, Media Library

### CDK Virtual Scroll Configuration

```typescript
// files/components/file-tree-virtual/file-tree-virtual.component.ts
import { CdkVirtualScrollViewport, ScrollingModule } from '@angular/cdk/scrolling';

@Component({
  selector: 'app-file-tree-virtual',
  standalone: true,
  imports: [ScrollingModule],
  template: `
    <cdk-virtual-scroll-viewport
      [itemSize]="32"
      [minBufferPx]="800"
      [maxBufferPx]="1600"
      class="file-tree-viewport"
    >
      <div
        *cdkVirtualFor="let node of flattenedNodes(); trackBy: trackByNodeId"
        [style.padding-left.px]="node.level * 20"
        class="file-node"
      >
        <mat-icon>{{ node.icon }}</mat-icon>
        <span>{{ node.name }}</span>
      </div>
    </cdk-virtual-scroll-viewport>
  `,
  styles: [`
    .file-tree-viewport {
      height: 100%;
      width: 100%;
    }
    .file-node {
      height: 32px;
      display: flex;
      align-items: center;
    }
  `],
})
export class FileTreeVirtualComponent {
  // Signal with 50k nodes
  nodes = input.required<FileNode[]>();

  // Flatten tree into array for virtual scrolling
  flattenedNodes = computed(() => {
    return this.flattenTree(this.nodes());
  });

  private flattenTree(nodes: FileNode[]): FlatNode[] {
    const result: FlatNode[] = [];
    const traverse = (nodes: FileNode[], level: number) => {
      for (const node of nodes) {
        result.push({ ...node, level });
        if (node.expanded && node.children?.length) {
          traverse(node.children, level + 1);
        }
      }
    };
    traverse(nodes, 0);
    return result;
  }

  trackByNodeId(index: number, node: FlatNode): string {
    return node.id;
  }
}

interface FlatNode extends FileNode {
  level: number;
}
```

### Virtual Scroll Performance Tuning

**itemSize**: Fixed height per item (32px). Crucial for accurate scrollbar positioning.

**minBufferPx/maxBufferPx**: Render buffer zones.
- `minBufferPx="800"`: Render extra 25 items above/below viewport
- `maxBufferPx="1600"`: Recycle items beyond 50 items from viewport
- **Why**: Smoother scrolling at 60fps. Users don't see blank spaces when scrolling fast.

**trackBy function**: Angular uses this to identify items efficiently.
```typescript
// GOOD: Track by unique ID
trackByNodeId(index: number, node: FlatNode): string {
  return node.id; // Stable across updates
}

// BAD: Track by index
trackByIndex(index: number): number {
  return index; // Re-renders all items when array changes
}
```

### Virtual Grid (Dataset Explorer)

For tabular data with 100k rows:

```typescript
// datasets/components/dataset-grid/dataset-grid.component.ts
@Component({
  selector: 'app-dataset-grid',
  template: `
    <cdk-virtual-scroll-viewport
      [itemSize]="40"
      [minBufferPx]="1200"
      [maxBufferPx]="2400"
      class="grid-viewport"
    >
      <table>
        <thead class="grid-header">
          <tr>
            @for (col of columns(); track col.id) {
              <th [style.width.px]="col.width">{{ col.name }}</th>
            }
          </tr>
        </thead>
        <tbody>
          @for (row of virtualRows(); track row.id) {
            <tr class="grid-row">
              @for (col of columns(); track col.id) {
                <td [style.width.px]="col.width">
                  {{ getCellValue(row, col) }}
                </td>
              }
            </tr>
          }
        </tbody>
      </table>
    </cdk-virtual-scroll-viewport>
  `,
})
export class DatasetGridComponent {
  viewport = viewChild.required(CdkVirtualScrollViewport);

  allRows = input.required<DatasetRow[]>(); // 100k rows
  columns = input.required<DataColumn[]>();

  // Only render visible rows
  virtualRows = computed(() => {
    const scrollOffset = this.viewport().measureScrollOffset();
    const viewportHeight = this.viewport().getViewportSize();
    const itemSize = 40;

    const startIndex = Math.floor(scrollOffset / itemSize);
    const endIndex = Math.ceil((scrollOffset + viewportHeight) / itemSize);

    return this.allRows().slice(startIndex, endIndex + 10); // +10 buffer
  });

  getCellValue(row: DatasetRow, col: DataColumn): string {
    return row[col.id] ?? '-';
  }
}
```

---

## Change Detection Optimization

### What: Signals replace Zone.js for fine-grained reactivity
**Why**: Zone.js checks entire component tree on every async event. Signals update only affected components.
**Result**: 10x faster updates in large apps

### Signals-First Architecture

```typescript
// BEFORE: Zone.js triggers change detection on entire tree
@Component({
  template: `
    <div>{{ counter }}</div>
    <button (click)="increment()">+</button>
  `,
})
export class OldComponent {
  counter = 0;

  increment() {
    this.counter++; // Zone.js detects this and checks ALL components
  }
}

// AFTER: Signal updates only this component
@Component({
  template: `
    <div>{{ counter() }}</div>
    <button (click)="increment()">+</button>
  `,
})
export class NewComponent {
  counter = signal(0);

  increment() {
    this.counter.update(c => c + 1); // Only this component re-renders
  }
}
```

### OnPush Strategy (Still Relevant)

Even with Signals, use `OnPush` for components with inputs:

```typescript
import { ChangeDetectionStrategy } from '@angular/core';

@Component({
  selector: 'app-file-card',
  changeDetection: ChangeDetectionStrategy.OnPush, // Only check when inputs change
  template: `
    <div class="card">
      <mat-icon>{{ file().icon }}</mat-icon>
      <span>{{ file().name }}</span>
      <span>{{ formattedSize() }}</span>
    </div>
  `,
})
export class FileCardComponent {
  file = input.required<FileNode>();

  formattedSize = computed(() => {
    return formatBytes(this.file().size);
  });
}
```

**Why OnPush?** Angular skips checking this component unless:
1. `@Input()` reference changes
2. Component emits event
3. Observable emits (with `async` pipe)
4. Signal updates (with `()` syntax)

### Avoid Large Computed Signals in Loops

```typescript
// BAD: Heavy computation runs on every change
allFiles = signal<FileNode[]>([/* 50k files */]);

expensiveComputation = computed(() => {
  return this.allFiles().map(file => ({
    ...file,
    hash: crypto.subtle.digest('SHA-256', file.content), // SLOW!
  }));
});

// Template re-runs this on every render
@for (file of expensiveComputation(); track file.id) {
  <app-file-card [file]="file" />
}

// GOOD: Compute once, cache in signal
async loadFiles() {
  const files = await this.fileService.getFiles();

  // Pre-compute hashes in Web Worker
  const filesWithHashes = await this.hashWorker.computeHashes(files);

  this.allFiles.set(filesWithHashes); // Update once
}
```

---

## Lazy Loading Patterns

### What: Load modules only when user navigates to them
**Why**: Initial bundle is 200KB instead of 2MB. App starts in <1s.

### Route-Level Lazy Loading

```typescript
// app.routes.ts
export const routes: Routes = [
  {
    path: 't/:tenantId',
    component: WorkspaceShellComponent,
    children: [
      {
        path: 'files',
        loadChildren: () => import('./features/files/files.routes')
          .then(m => m.FILE_ROUTES), // Loads files.routes.ts + all file components
      },
      {
        path: 'design-system',
        loadChildren: () => import('./features/design-system/design-system.routes')
          .then(m => m.DESIGN_SYSTEM_ROUTES),
      },
      {
        path: 'datasets',
        loadChildren: () => import('./features/datasets/datasets.routes')
          .then(m => m.DATASET_ROUTES),
      },
      // ... other modules
    ],
  },
];
```

**Bundle Analysis**:
```bash
npm run build -- --stats-json
npx webpack-bundle-analyzer dist/nexus-studio/stats.json
```

Expected chunks:
- `main.js` (200KB): App shell, Material components
- `files.chunk.js` (150KB): File Manager module
- `datasets.chunk.js` (180KB): Dataset Explorer + Web Worker
- `kanban.chunk.js` (120KB): Kanban module
- ... (7 more modules)

### Component-Level Lazy Loading

For heavy components within a module:

```typescript
// datasets/dataset-shell.component.ts
import { Component, signal } from '@angular/core';

@Component({
  template: `
    <div class="dataset-shell">
      <app-dataset-toolbar />

      @if (showChart()) {
        @defer (on viewport) {
          <app-dataset-chart [data]="chartData()" />
        } @placeholder {
          <div class="chart-skeleton">Loading chart...</div>
        } @loading (minimum 500ms) {
          <mat-spinner diameter="40" />
        }
      }
    </div>
  `,
})
export class DatasetShellComponent {
  showChart = signal(false);
  chartData = signal<ChartData | null>(null);
}
```

**@defer triggers**:
- `on viewport`: Load when placeholder enters viewport
- `on idle`: Load when browser is idle
- `on timer(2s)`: Load after 2 seconds
- `on interaction`: Load when user clicks placeholder

---

## Rendering Boundaries

### What: Isolate expensive renders to specific subtrees
**Why**: Prevents cascading re-renders across the app

### Example: Status Bar Updates

```typescript
// Status bar updates every 100ms (clock, activity), but should NOT trigger re-renders elsewhere

// BEFORE: Status bar in app-shell component
@Component({
  selector: 'app-shell',
  template: `
    <app-top-bar />
    <app-left-nav />
    <router-outlet />
    <app-status-bar [clock]="currentTime" /> <!-- Re-renders entire shell every 100ms! -->
  `,
})
export class AppShellComponent {
  currentTime = signal(new Date());

  constructor() {
    setInterval(() => {
      this.currentTime.set(new Date()); // Triggers change detection
    }, 100);
  }
}

// AFTER: Status bar owns its update loop
@Component({
  selector: 'app-shell',
  template: `
    <app-top-bar />
    <app-left-nav />
    <router-outlet />
    <app-status-bar /> <!-- Self-contained, no inputs -->
  `,
})
export class AppShellComponent {}

@Component({
  selector: 'app-status-bar',
  changeDetection: ChangeDetectionStrategy.OnPush, // Isolated boundary
  template: `
    <div class="status-bar">
      <span>{{ currentTime() }}</span>
      <span>{{ activityStatus() }}</span>
    </div>
  `,
})
export class StatusBarComponent {
  private currentTime = signal(new Date());
  private activityStatus = signal('Idle');

  constructor() {
    // Only this component re-renders
    setInterval(() => {
      this.currentTime.set(new Date());
    }, 100);
  }
}
```

### Rendering Boundary Pattern

**Rule**: Components with high-frequency updates should:
1. Use `ChangeDetectionStrategy.OnPush`
2. Own their data (no inputs from parent)
3. Use local signals for state

```typescript
// High-frequency components:
// - StatusBarComponent (clock updates)
// - LogStreamComponent (live logs streaming)
// - ActivityFeedComponent (real-time activity)
// - DatasetProfilerComponent (progress updates)

@Component({
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `...`,
})
export class HighFrequencyComponent {
  // Own data, no inputs
  private data = signal<Data>(initialData);

  constructor() {
    // Internal update loop
    this.startUpdateLoop();
  }
}
```

---

## Web Workers for Heavy Computation

### What: Offload CPU-intensive tasks to background threads
**Why**: Keeps main thread responsive at 60fps. No UI freezes.
**When**: Tasks taking >50ms on main thread

### Search Indexer Worker

```typescript
// core/workers/search-indexer.worker.ts
/// <reference lib="webworker" />

interface SearchDocument {
  id: string;
  title: string;
  content: string;
  type: string;
}

interface TFIDFIndex {
  [term: string]: {
    [docId: string]: number; // TF-IDF score
  };
}

let index: TFIDFIndex = {};
let documents: SearchDocument[] = [];

addEventListener('message', (event: MessageEvent) => {
  const { type, payload } = event.data;

  switch (type) {
    case 'index':
      indexDocuments(payload.documents);
      postMessage({ type: 'indexed', payload: { count: documents.length } });
      break;

    case 'search':
      const results = search(payload.query, payload.limit);
      postMessage({ type: 'results', payload: { results } });
      break;
  }
});

function indexDocuments(docs: SearchDocument[]) {
  documents = docs;
  index = buildTFIDFIndex(docs);
}

function buildTFIDFIndex(docs: SearchDocument[]): TFIDFIndex {
  const termFrequency: Map<string, Map<string, number>> = new Map();
  const documentFrequency: Map<string, number> = new Map();

  // Calculate TF
  for (const doc of docs) {
    const terms = tokenize(doc.title + ' ' + doc.content);
    const termCounts = new Map<string, number>();

    for (const term of terms) {
      termCounts.set(term, (termCounts.get(term) || 0) + 1);
    }

    termFrequency.set(doc.id, termCounts);

    // Count document frequency
    for (const term of termCounts.keys()) {
      documentFrequency.set(term, (documentFrequency.get(term) || 0) + 1);
    }
  }

  // Calculate TF-IDF
  const tfidf: TFIDFIndex = {};
  for (const doc of docs) {
    const termCounts = termFrequency.get(doc.id)!;
    const maxFreq = Math.max(...termCounts.values());

    for (const [term, count] of termCounts) {
      const tf = count / maxFreq; // Normalized term frequency
      const idf = Math.log(docs.length / (documentFrequency.get(term) || 1));
      const score = tf * idf;

      if (!tfidf[term]) tfidf[term] = {};
      tfidf[term][doc.id] = score;
    }
  }

  return tfidf;
}

function search(query: string, limit: number): SearchDocument[] {
  const queryTerms = tokenize(query);
  const scores = new Map<string, number>();

  // Sum TF-IDF scores for query terms
  for (const term of queryTerms) {
    const termScores = index[term] || {};
    for (const [docId, score] of Object.entries(termScores)) {
      scores.set(docId, (scores.get(docId) || 0) + score);
    }
  }

  // Sort by score descending
  const ranked = Array.from(scores.entries())
    .sort((a, b) => b[1] - a[1])
    .slice(0, limit)
    .map(([docId]) => documents.find(d => d.id === docId)!);

  return ranked;
}

function tokenize(text: string): string[] {
  return text
    .toLowerCase()
    .replace(/[^\w\s]/g, ' ')
    .split(/\s+/)
    .filter(t => t.length > 2); // Ignore words < 3 chars
}
```

### Using the Search Worker

```typescript
// core/services/search.service.ts
@Injectable({ providedIn: 'root' })
export class SearchService {
  private worker: Worker;
  private results$ = new Subject<SearchResult[]>();

  constructor() {
    this.worker = new Worker(new URL('../workers/search-indexer.worker', import.meta.url));

    this.worker.onmessage = (event) => {
      if (event.data.type === 'results') {
        this.results$.next(event.data.payload.results);
      }
    };
  }

  async indexDocuments(docs: SearchDocument[]): Promise<void> {
    return new Promise((resolve) => {
      const handler = (event: MessageEvent) => {
        if (event.data.type === 'indexed') {
          this.worker.removeEventListener('message', handler);
          resolve();
        }
      };
      this.worker.addEventListener('message', handler);
      this.worker.postMessage({ type: 'index', payload: { documents: docs } });
    });
  }

  search(query: string, limit = 50): Observable<SearchResult[]> {
    this.worker.postMessage({ type: 'search', payload: { query, limit } });
    return this.results$.asObservable();
  }
}
```

### Dataset Profiler Worker

```typescript
// datasets/workers/dataset-profiler.worker.ts
/// <reference lib="webworker" />

interface ProfileRequest {
  rows: any[];
  columns: string[];
}

interface ColumnStats {
  name: string;
  type: 'number' | 'string' | 'date' | 'boolean' | 'mixed';
  nullCount: number;
  uniqueCount: number;
  min?: number;
  max?: number;
  mean?: number;
  median?: number;
  mode?: any;
}

addEventListener('message', (event: MessageEvent) => {
  const { type, payload } = event.data as { type: string; payload: ProfileRequest };

  if (type === 'profile') {
    const stats = profileDataset(payload.rows, payload.columns);
    postMessage({ type: 'complete', payload: { stats } });
  }
});

function profileDataset(rows: any[], columns: string[]): ColumnStats[] {
  return columns.map(col => {
    const values = rows.map(r => r[col]).filter(v => v != null);
    const nullCount = rows.length - values.length;
    const uniqueValues = new Set(values);

    // Detect type
    const types = values.map(v => typeof v);
    const primaryType = mostCommon(types) as ColumnStats['type'];

    const stats: ColumnStats = {
      name: col,
      type: primaryType,
      nullCount,
      uniqueCount: uniqueValues.size,
    };

    if (primaryType === 'number') {
      const nums = values as number[];
      stats.min = Math.min(...nums);
      stats.max = Math.max(...nums);
      stats.mean = nums.reduce((a, b) => a + b, 0) / nums.length;
      stats.median = calculateMedian(nums);
    }

    stats.mode = mostCommon(values);

    return stats;
  });
}

function mostCommon<T>(arr: T[]): T {
  const counts = new Map<T, number>();
  for (const val of arr) {
    counts.set(val, (counts.get(val) || 0) + 1);
  }
  return Array.from(counts.entries()).sort((a, b) => b[1] - a[1])[0][0];
}

function calculateMedian(nums: number[]): number {
  const sorted = [...nums].sort((a, b) => a - b);
  const mid = Math.floor(sorted.length / 2);
  return sorted.length % 2 === 0
    ? (sorted[mid - 1] + sorted[mid]) / 2
    : sorted[mid];
}
```

---

## IndexedDB Optimization

### What: Client-side database for offline storage
**Why**: LocalStorage limited to 5MB. IndexedDB handles 50MB-500MB.

### Efficient Queries with Indexes

```typescript
// core/services/indexeddb.service.ts
@Injectable({ providedIn: 'root' })
export class IndexedDBService {
  private db: IDBDatabase | null = null;

  async init(): Promise<void> {
    return new Promise((resolve, reject) => {
      const request = indexedDB.open('NexusStudio', 3);

      request.onerror = () => reject(request.error);
      request.onsuccess = () => {
        this.db = request.result;
        resolve();
      };

      request.onupgradeneeded = (event) => {
        const db = (event.target as IDBOpenDBRequest).result;

        // Files store
        if (!db.objectStoreNames.contains('files')) {
          const fileStore = db.createObjectStore('files', { keyPath: 'id' });
          fileStore.createIndex('tenantId', 'tenantId', { unique: false });
          fileStore.createIndex('parentId', 'parentId', { unique: false });
          fileStore.createIndex('mimeType', 'mimeType', { unique: false });
          fileStore.createIndex('createdAt', 'createdAt', { unique: false });
          fileStore.createIndex('tags', 'tags', { unique: false, multiEntry: true });
        }

        // Datasets store
        if (!db.objectStoreNames.contains('datasets')) {
          const datasetStore = db.createObjectStore('datasets', { keyPath: 'id' });
          datasetStore.createIndex('tenantId', 'tenantId', { unique: false });
          datasetStore.createIndex('createdBy', 'createdBy', { unique: false });
        }

        // Cases store
        if (!db.objectStoreNames.contains('cases')) {
          const caseStore = db.createObjectStore('cases', { keyPath: 'id' });
          caseStore.createIndex('tenantId', 'tenantId', { unique: false });
          caseStore.createIndex('status', 'status', { unique: false });
          caseStore.createIndex('priority', 'priority', { unique: false });
          caseStore.createIndex('assignedTo', 'assignedTo', { unique: false });
        }
      };
    });
  }

  // Efficient query: Get files by tenant + MIME type
  async getFilesByMimeType(tenantId: string, mimeType: string): Promise<FileNode[]> {
    const tx = this.db!.transaction('files', 'readonly');
    const store = tx.objectStore('files');
    const index = store.index('mimeType');

    const files: FileNode[] = [];

    return new Promise((resolve, reject) => {
      const request = index.openCursor(IDBKeyRange.only(mimeType));

      request.onsuccess = () => {
        const cursor = request.result;
        if (cursor) {
          const file = cursor.value as FileNode;
          if (file.tenantId === tenantId) { // Filter by tenant
            files.push(file);
          }
          cursor.continue();
        } else {
          resolve(files);
        }
      };

      request.onerror = () => reject(request.error);
    });
  }

  // Batch insert (faster than individual puts)
  async bulkPut(storeName: string, items: any[]): Promise<void> {
    const tx = this.db!.transaction(storeName, 'readwrite');
    const store = tx.objectStore(storeName);

    for (const item of items) {
      store.put(item);
    }

    return new Promise((resolve, reject) => {
      tx.oncomplete = () => resolve();
      tx.onerror = () => reject(tx.error);
    });
  }

  // Pagination with cursor
  async getPage<T>(
    storeName: string,
    page: number,
    pageSize: number
  ): Promise<T[]> {
    const tx = this.db!.transaction(storeName, 'readonly');
    const store = tx.objectStore(storeName);

    const results: T[] = [];
    let skipped = 0;
    const skip = page * pageSize;

    return new Promise((resolve, reject) => {
      const request = store.openCursor();

      request.onsuccess = () => {
        const cursor = request.result;
        if (cursor) {
          if (skipped < skip) {
            skipped++;
            cursor.continue();
          } else if (results.length < pageSize) {
            results.push(cursor.value as T);
            cursor.continue();
          } else {
            resolve(results);
          }
        } else {
          resolve(results);
        }
      };

      request.onerror = () => reject(request.error);
    });
  }
}
```

### Compound Indexes for Complex Queries

```typescript
// Scenario: Query cases by tenant + status + priority
request.onupgradeneeded = (event) => {
  const db = (event.target as IDBOpenDBRequest).result;
  const caseStore = db.createObjectStore('cases', { keyPath: 'id' });

  // Compound index for common query
  caseStore.createIndex('tenantStatus', ['tenantId', 'status'], { unique: false });
  caseStore.createIndex('tenantPriority', ['tenantId', 'priority'], { unique: false });
};

// Query using compound index
async getCasesByStatus(tenantId: string, status: CaseStatus): Promise<Case[]> {
  const tx = this.db!.transaction('cases', 'readonly');
  const store = tx.objectStore('cases');
  const index = store.index('tenantStatus');

  const range = IDBKeyRange.only([tenantId, status]);

  return new Promise((resolve, reject) => {
    const request = index.getAll(range);
    request.onsuccess = () => resolve(request.result);
    request.onerror = () => reject(request.error);
  });
}
```

---

## Bundle Size Optimization

### Tree-Shaking Material Components

```typescript
// DON'T: Import entire Material library
import * as Material from '@angular/material';

// DO: Import only what you need
import { MatButtonModule } from '@angular/material/button';
import { MatIconModule } from '@angular/material/icon';
import { MatDialogModule } from '@angular/material/dialog';

@Component({
  standalone: true,
  imports: [MatButtonModule, MatIconModule, MatDialogModule],
  template: `...`,
})
export class MyComponent {}
```

### Code Splitting with Dynamic Imports

```typescript
// Heavy library used in one feature
async exportToExcel(data: any[]) {
  // Only load SheetJS when user clicks "Export"
  const XLSX = await import('xlsx');

  const worksheet = XLSX.utils.json_to_sheet(data);
  const workbook = XLSX.utils.book_new();
  XLSX.utils.book_append_sheet(workbook, worksheet, 'Data');
  XLSX.writeFile(workbook, 'export.xlsx');
}
```

### Remove Unused Code

```bash
# Analyze bundle
npm run build -- --stats-json
npx webpack-bundle-analyzer dist/nexus-studio/stats.json

# Look for:
# - Duplicate libraries (lodash, moment)
# - Unused polyfills
# - Large libraries that can be replaced (moment → date-fns)
```

**Example optimizations**:
- Replace Moment.js (67KB) with date-fns (15KB)
- Use `lodash-es` instead of `lodash` (tree-shakable)
- Remove unused Material components

---

## Performance Monitoring

### Core Web Vitals Tracking

```typescript
// core/services/performance-monitor.service.ts
import { Injectable } from '@angular/core';

@Injectable({ providedIn: 'root' })
export class PerformanceMonitorService {
  constructor() {
    this.observeLCP();
    this.observeFID();
    this.observeCLS();
  }

  // Largest Contentful Paint (should be <2.5s)
  private observeLCP() {
    const observer = new PerformanceObserver((list) => {
      const entries = list.getEntries();
      const lastEntry = entries[entries.length - 1] as any;

      console.log('LCP:', lastEntry.renderTime || lastEntry.loadTime);

      // Send to analytics
      this.sendMetric('LCP', lastEntry.renderTime || lastEntry.loadTime);
    });

    observer.observe({ type: 'largest-contentful-paint', buffered: true });
  }

  // First Input Delay (should be <100ms)
  private observeFID() {
    const observer = new PerformanceObserver((list) => {
      const entries = list.getEntries();
      entries.forEach((entry: any) => {
        console.log('FID:', entry.processingStart - entry.startTime);
        this.sendMetric('FID', entry.processingStart - entry.startTime);
      });
    });

    observer.observe({ type: 'first-input', buffered: true });
  }

  // Cumulative Layout Shift (should be <0.1)
  private observeCLS() {
    let clsValue = 0;
    const observer = new PerformanceObserver((list) => {
      for (const entry of list.getEntries() as any[]) {
        if (!entry.hadRecentInput) {
          clsValue += entry.value;
        }
      }
      console.log('CLS:', clsValue);
      this.sendMetric('CLS', clsValue);
    });

    observer.observe({ type: 'layout-shift', buffered: true });
  }

  private sendMetric(name: string, value: number) {
    // Send to analytics service
    console.log(`[Metric] ${name}: ${value}`);
  }

  // Custom timing for route changes
  markRouteChange(routeName: string) {
    performance.mark(`route-${routeName}-start`);
  }

  measureRouteRender(routeName: string) {
    performance.mark(`route-${routeName}-end`);
    performance.measure(
      `route-${routeName}`,
      `route-${routeName}-start`,
      `route-${routeName}-end`
    );

    const measure = performance.getEntriesByName(`route-${routeName}`)[0];
    console.log(`Route ${routeName} rendered in ${measure.duration}ms`);
  }
}
```

### Long Task Monitoring

```typescript
// Detect tasks blocking main thread for >50ms
const observer = new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    console.warn(`Long task detected: ${entry.duration}ms`, entry);

    // If you see this, move work to Web Worker or use setTimeout chunking
  }
});

observer.observe({ type: 'longtask', buffered: true });
```

### Memory Leak Detection

```typescript
// Run in dev mode to detect memory leaks
@Component({
  template: `...`,
})
export class MyComponent implements OnDestroy {
  private subscription = new Subscription();

  constructor() {
    // LEAK: Forgot to unsubscribe
    interval(1000).subscribe(() => {
      console.log('tick');
    });

    // CORRECT: Unsubscribe in ngOnDestroy
    this.subscription.add(
      interval(1000).subscribe(() => {
        console.log('tick');
      })
    );
  }

  ngOnDestroy() {
    this.subscription.unsubscribe();
  }
}
```

**With Signals**: No manual unsubscribe needed

```typescript
@Component({
  template: `<div>{{ count() }}</div>`,
})
export class MyComponent {
  private count = signal(0);

  constructor() {
    // Auto-cleanup when component destroyed
    effect(() => {
      console.log('Count changed:', this.count());
    });
  }
}
```

---

## Performance Budget

### Targets for Nexus Studio

| Metric | Target | Measurement |
|--------|--------|-------------|
| Initial Bundle | <200KB gzipped | `npm run build` |
| First Contentful Paint | <1.5s | Lighthouse |
| Largest Contentful Paint | <2.5s | Web Vitals |
| Time to Interactive | <3.5s | Lighthouse |
| First Input Delay | <100ms | Web Vitals |
| Cumulative Layout Shift | <0.1 | Web Vitals |
| Virtual Scroll FPS | 60fps | Chrome DevTools |
| Route Change Time | <200ms | Performance API |
| IndexedDB Query | <50ms (1k items) | Performance API |
| Web Worker Task | <500ms (50k items) | Performance API |

### Lighthouse CI Integration

```json
// lighthouserc.json
{
  "ci": {
    "collect": {
      "url": ["http://localhost:4200"],
      "numberOfRuns": 3
    },
    "assert": {
      "preset": "lighthouse:recommended",
      "assertions": {
        "categories:performance": ["error", { "minScore": 0.9 }],
        "categories:accessibility": ["error", { "minScore": 1.0 }],
        "first-contentful-paint": ["error", { "maxNumericValue": 1500 }],
        "largest-contentful-paint": ["error", { "maxNumericValue": 2500 }],
        "cumulative-layout-shift": ["error", { "maxNumericValue": 0.1 }]
      }
    }
  }
}
```

```bash
# Run Lighthouse in CI
npm run build
npx http-server dist/nexus-studio -p 8080 &
npx @lhci/cli@0.12.x autorun
```

---

## Checklist: Pre-Release Performance Audit

- [ ] **Virtual Scrolling**: All lists >100 items use CDK Virtual Scroll
- [ ] **Change Detection**: All components use `OnPush` or Signals
- [ ] **Lazy Loading**: All feature modules lazy-loaded
- [ ] **Web Workers**: Heavy computation (search, profiling) offloaded
- [ ] **Bundle Size**: Main bundle <200KB, no duplicate libraries
- [ ] **IndexedDB**: Indexes created for common queries
- [ ] **Image Optimization**: All images lazy-loaded, WebP format
- [ ] **Memory Leaks**: No subscriptions leak, effects auto-cleanup
- [ ] **Lighthouse Score**: Performance >90, Accessibility 100
- [ ] **Real Device Testing**: Test on low-end Android device (slow CPU)

---

## Summary

**Principles**:
1. **Render less**: Virtual scrolling for large lists
2. **Update less**: Signals + OnPush for fine-grained reactivity
3. **Load less**: Lazy modules, dynamic imports
4. **Block less**: Web Workers for heavy tasks
5. **Measure always**: Core Web Vitals, Lighthouse CI

**Result**: Nexus Studio handles 100k-item datasets with 60fps scrolling, <1s route changes, and <200KB initial load.
