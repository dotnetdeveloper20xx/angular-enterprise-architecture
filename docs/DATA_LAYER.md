# Data Layer: Mock API, Persistence & Data Models

> **Complete specification for the data layer that powers Nexus Studio without a real backend**

---

## 🎯 What & Why

### What is the Data Layer?

The data layer provides:
1. **Mock API**: Simulates a REST API with realistic delays, errors, pagination
2. **Persistence**: IndexedDB for storing large datasets client-side
3. **Data Models**: TypeScript interfaces for all entities
4. **Workers**: Background processing for heavy data operations

### Why Mock Everything?

**Benefits**:
- ✅ Build and demo the full app without backend infrastructure
- ✅ Develop frontend independently (no waiting for APIs)
- ✅ Offline-first by default
- ✅ Easy to swap for real backend later (just change service implementations)
- ✅ Realistic data for testing and demos

**Design Principle**: Architect as if a real backend exists, but implement with mocks.

---

## 📊 Complete Data Models

### Core Entities

#### 1. Case (Support Ticket)

```typescript
/**
 * Case: Support ticket or customer inquiry.
 * Used by: Case Triage Console
 */
export interface Case {
  // Identity
  id: string;                          // "CASE-2491"
  tenantId: string;

  // Content
  subject: string;
  description: string;                 // Rich text (HTML or Markdown)
  type: CaseType;
  priority: CasePriority;
  status: CaseStatus;

  // Assignment
  assignedTo?: string;                 // User ID
  team?: string;                       // Team ID

  // Customer
  customerId?: string;
  customerEmail: string;
  customerName: string;

  // Timeline
  createdAt: Date;
  updatedAt: Date;
  closedAt?: Date;
  resolvedAt?: Date;

  // SLA
  slaDeadline?: Date;
  firstResponseAt?: Date;

  // Metadata
  createdBy: string;                   // User ID
  tags: string[];
  attachments: FileReference[];

  // References
  linkedCases: string[];               // Other case IDs
  linkedFiles: string[];               // File IDs
  linkedLogs: LogReference[];          // Log entries

  // Communication
  comments: CaseComment[];
  watchers: string[];                  // User IDs

  // Custom Fields
  customFields: Record<string, any>;
}

export type CaseType = 'bug' | 'feature-request' | 'question' | 'incident' | 'task';
export type CasePriority = 'low' | 'medium' | 'high' | 'critical';
export type CaseStatus = 'new' | 'open' | 'pending' | 'resolved' | 'closed';

export interface CaseComment {
  id: string;
  caseId: string;
  authorId: string;
  authorName: string;
  content: string;                     // Rich text
  createdAt: Date;
  isInternal: boolean;                 // Internal note vs customer-facing
  attachments: FileReference[];
}

export interface FileReference {
  fileId: string;
  fileName: string;
  fileSize: number;
  mimeType: string;
}

export interface LogReference {
  logId: string;
  timestamp: Date;
  level: LogLevel;
  message: string;
  correlationId?: string;
}
```

#### 2. Kanban Card

```typescript
/**
 * Kanban Card: Task in a kanban board.
 * Used by: Kanban + Timeline module
 */
export interface KanbanCard {
  // Identity
  id: string;                          // "TASK-123"
  boardId: string;
  tenantId: string;

  // Content
  title: string;
  description?: string;                // Markdown
  type: CardType;

  // Position
  columnId: string;                    // Which column (status)
  position: number;                    // Order within column

  // Assignment
  assignees: string[];                 // User IDs
  labels: Label[];

  // Dates
  createdAt: Date;
  updatedAt: Date;
  dueDate?: Date;
  startDate?: Date;
  completedAt?: Date;

  // Progress
  checklist: ChecklistItem[];
  progress: number;                    // 0-100%

  // Metadata
  createdBy: string;
  priority: 'low' | 'medium' | 'high';
  estimatedHours?: number;
  actualHours?: number;

  // References
  linkedCases: string[];
  linkedFiles: string[];
  linkedDatasets: string[];

  // Comments
  comments: CardComment[];
  attachments: FileReference[];

  // Custom Fields
  customFields: Record<string, any>;
}

export type CardType = 'task' | 'bug' | 'feature' | 'epic';

export interface Label {
  id: string;
  name: string;
  color: string;                       // Hex color
}

export interface ChecklistItem {
  id: string;
  text: string;
  completed: boolean;
  assignee?: string;
}

export interface CardComment {
  id: string;
  authorId: string;
  authorName: string;
  content: string;
  createdAt: Date;
  reactions: Reaction[];
}

export interface Reaction {
  emoji: string;
  userIds: string[];
}
```

#### 3. Dataset

```typescript
/**
 * Dataset: Tabular data (CSV, JSON, Parquet).
 * Used by: Dataset Explorer
 */
export interface Dataset {
  // Identity
  id: string;
  tenantId: string;
  fileId?: string;                     // Link to FileNode

  // Metadata
  name: string;
  description?: string;
  source: DatasetSource;

  // Schema
  rowCount: number;
  columnCount: number;
  columns: DatasetColumn[];

  // Storage
  format: 'csv' | 'json' | 'parquet';
  size: number;                        // Bytes
  storageKey: string;                  // IndexedDB key or blob URL

  // Dates
  createdAt: Date;
  updatedAt: Date;
  lastProfiledAt?: Date;

  // Access
  createdBy: string;
  isPublic: boolean;
  tags: string[];

  // Stats (computed by profiler worker)
  stats?: DatasetStats;
}

export interface DatasetSource {
  type: 'upload' | 'import' | 'api' | 'generated';
  url?: string;
  apiEndpoint?: string;
}

export interface DatasetColumn {
  name: string;
  displayName?: string;
  type: ColumnType;
  nullable: boolean;
  unique: boolean;

  // Stats
  distinctCount?: number;
  nullCount?: number;
  min?: any;
  max?: any;
  mean?: number;
  median?: number;
  mode?: any;

  // Examples
  sampleValues?: any[];
}

export type ColumnType = 'string' | 'number' | 'boolean' | 'date' | 'datetime' | 'unknown';

export interface DatasetStats {
  totalRows: number;
  totalColumns: number;
  memoryUsage: number;                 // Bytes
  duplicateRows: number;
  completeness: number;                // % of non-null cells
  profiledAt: Date;
}
```

#### 4. Knowledge Base Document

```typescript
/**
 * KB Document: Wiki page or documentation article.
 * Used by: Knowledge Base
 */
export interface KBDocument {
  // Identity
  id: string;
  tenantId: string;
  fileId?: string;                     // Link to FileNode

  // Content
  title: string;
  content: string;                     // Markdown
  excerpt?: string;                    // Short description

  // Hierarchy
  parentId?: string;                   // Parent document
  path: string;                        // "/docs/getting-started/setup"
  level: number;                       // Depth in hierarchy

  // Metadata
  createdAt: Date;
  updatedAt: Date;
  publishedAt?: Date;
  createdBy: string;
  updatedBy: string;

  // Status
  status: 'draft' | 'published' | 'archived';
  isPublic: boolean;

  // Taxonomy
  tags: string[];
  category?: string;

  // Versioning
  version: number;
  previousVersionId?: string;

  // Table of Contents
  headings: Heading[];

  // References
  linkedDocs: string[];                // Other KB doc IDs
  linkedFiles: string[];
  linkedCases: string[];

  // Engagement
  viewCount: number;
  lastViewedAt?: Date;

  // Search
  searchableText: string;              // Extracted from content
}

export interface Heading {
  level: number;                       // 1-6 (H1-H6)
  text: string;
  slug: string;                        // "introduction"
}
```

#### 5. Form Schema

```typescript
/**
 * Form Schema: Dynamic form definition.
 * Used by: Form Builder
 */
export interface FormSchema {
  // Identity
  id: string;
  tenantId: string;
  fileId?: string;

  // Metadata
  name: string;
  description?: string;
  version: number;

  // Schema
  fields: FormField[];
  sections?: FormSection[];            // Group fields into sections

  // Behavior
  validationRules: ValidationRule[];
  conditionalLogic: ConditionalRule[];

  // Submission
  submitUrl?: string;                  // Where to POST responses
  successMessage?: string;
  redirectUrl?: string;

  // Dates
  createdAt: Date;
  updatedAt: Date;
  publishedAt?: Date;

  // Access
  createdBy: string;
  isPublic: boolean;
  status: 'draft' | 'published' | 'archived';

  // Stats
  responseCount: number;
  lastResponseAt?: Date;
}

export interface FormField {
  id: string;
  name: string;
  label: string;
  type: FormFieldType;
  required: boolean;
  placeholder?: string;
  helpText?: string;
  defaultValue?: any;

  // Options (for select, radio, checkbox)
  options?: FormFieldOption[];

  // Validation
  minLength?: number;
  maxLength?: number;
  pattern?: string;                    // Regex pattern
  min?: number;                        // For number inputs
  max?: number;

  // Layout
  width?: 'full' | 'half' | 'third';
  order: number;
}

export type FormFieldType =
  | 'text'
  | 'textarea'
  | 'email'
  | 'number'
  | 'date'
  | 'time'
  | 'datetime'
  | 'select'
  | 'radio'
  | 'checkbox'
  | 'file'
  | 'rating'
  | 'slider';

export interface FormFieldOption {
  label: string;
  value: any;
}

export interface FormSection {
  id: string;
  title: string;
  description?: string;
  fieldIds: string[];
  order: number;
}

export interface ValidationRule {
  fieldId: string;
  rule: 'required' | 'email' | 'url' | 'min' | 'max' | 'pattern' | 'custom';
  value?: any;
  message: string;
}

export interface ConditionalRule {
  fieldId: string;                     // Field to show/hide
  condition: {
    fieldId: string;                   // Field to check
    operator: '==' | '!=' | '>' | '<' | 'contains';
    value: any;
  };
  action: 'show' | 'hide' | 'enable' | 'disable';
}
```

#### 6. Log Entry

```typescript
/**
 * Log Entry: Application or system log.
 * Used by: Log Viewer / Observability Console
 */
export interface LogEntry {
  // Identity
  id: string;
  tenantId: string;

  // Content
  timestamp: Date;
  level: LogLevel;
  message: string;
  source: string;                      // Service/module name

  // Context
  correlationId?: string;              // Trace ID
  userId?: string;
  sessionId?: string;
  requestId?: string;

  // Structured Data
  metadata: Record<string, any>;       // Arbitrary JSON
  exception?: ExceptionDetails;

  // Performance
  duration?: number;                   // Milliseconds
  statusCode?: number;                 // HTTP status

  // Environment
  environment: 'development' | 'staging' | 'production';
  hostname?: string;
}

export type LogLevel = 'trace' | 'debug' | 'info' | 'warn' | 'error' | 'fatal';

export interface ExceptionDetails {
  type: string;
  message: string;
  stackTrace: string;
  innerException?: ExceptionDetails;
}
```

#### 7. Design Token

```typescript
/**
 * Design Token: Design system value (color, spacing, etc.).
 * Used by: Design System Playground
 */
export interface DesignToken {
  id: string;
  tenantId: string;

  // Token Definition
  name: string;                        // "color-primary-500"
  category: TokenCategory;
  value: string | number;              // "#1976d2" or 16
  type: TokenType;

  // Metadata
  description?: string;
  tags: string[];

  // Hierarchy
  group?: string;                      // "colors.primary"
  parent?: string;                     // Reference another token

  // Output
  cssVariable: string;                 // "--color-primary-500"
  scssVariable: string;                // "$color-primary-500"

  // Dates
  createdAt: Date;
  updatedAt: Date;
  createdBy: string;
}

export type TokenCategory =
  | 'color'
  | 'spacing'
  | 'typography'
  | 'border-radius'
  | 'shadow'
  | 'animation';

export type TokenType =
  | 'color'
  | 'dimension'                        // px, rem, em
  | 'font-family'
  | 'font-size'
  | 'font-weight'
  | 'duration'                         // ms
  | 'cubic-bezier'
  | 'other';
```

---

## 🌐 Mock API Service

### API Architecture

```typescript
/**
 * Mock API Service: Simulates a REST API.
 * Intercepts HTTP calls and returns mocked data with realistic delays.
 */
@Injectable({ providedIn: 'root' })
export class MockApiService {
  private db = inject(IndexedDbService);
  private config = {
    baseUrl: '/api',
    defaultDelay: 200,                 // ms
    errorRate: 0.02,                   // 2% random errors
  };

  /**
   * GET request.
   */
  async get<T>(endpoint: string, options?: RequestOptions): Promise<T> {
    await this.simulateDelay();
    this.maybeThrowError();

    const { collection, id } = this.parseEndpoint(endpoint);

    if (id) {
      // GET /api/cases/123
      return this.db.getById(collection, id) as T;
    } else {
      // GET /api/cases?status=open&limit=20
      let items = await this.db.getAll<any>(collection);

      // Apply filters
      if (options?.params) {
        items = this.applyFilters(items, options.params);
      }

      // Apply sorting
      if (options?.params?.sortBy) {
        items = this.applySorting(items, options.params.sortBy, options.params.sortOrder);
      }

      // Apply pagination
      if (options?.params?.limit) {
        const offset = options.params.offset || 0;
        const limit = options.params.limit;
        items = items.slice(offset, offset + limit);
      }

      return items as T;
    }
  }

  /**
   * POST request (create).
   */
  async post<T>(endpoint: string, body: any): Promise<T> {
    await this.simulateDelay();
    this.maybeThrowError();

    const { collection } = this.parseEndpoint(endpoint);

    const newItem = {
      ...body,
      id: body.id || generateId(),
      createdAt: new Date(),
      updatedAt: new Date(),
    };

    await this.db.add(collection, newItem);

    return newItem as T;
  }

  /**
   * PATCH request (update).
   */
  async patch<T>(endpoint: string, updates: Partial<any>): Promise<T> {
    await this.simulateDelay();
    this.maybeThrowError();

    const { collection, id } = this.parseEndpoint(endpoint);
    if (!id) throw new Error('PATCH requires ID');

    const existing = await this.db.getById(collection, id);
    if (!existing) throw new Error(`${collection}/${id} not found`);

    const updated = {
      ...existing,
      ...updates,
      updatedAt: new Date(),
    };

    await this.db.update(collection, id, updated);

    return updated as T;
  }

  /**
   * DELETE request.
   */
  async delete(endpoint: string): Promise<void> {
    await this.simulateDelay();
    this.maybeThrowError();

    const { collection, id } = this.parseEndpoint(endpoint);
    if (!id) throw new Error('DELETE requires ID');

    await this.db.delete(collection, id);
  }

  /**
   * Simulate network delay.
   */
  private async simulateDelay() {
    const delay = this.config.defaultDelay + Math.random() * 100;
    await new Promise(resolve => setTimeout(resolve, delay));
  }

  /**
   * Maybe throw random error (for testing error handling).
   */
  private maybeThrowError() {
    if (Math.random() < this.config.errorRate) {
      throw new Error('Simulated API error');
    }
  }

  /**
   * Parse endpoint into collection and ID.
   * "/api/cases/123" → { collection: "cases", id: "123" }
   */
  private parseEndpoint(endpoint: string): { collection: string; id?: string } {
    const parts = endpoint.replace(this.config.baseUrl + '/', '').split('/');
    return {
      collection: parts[0],
      id: parts[1],
    };
  }

  /**
   * Apply query filters to items.
   */
  private applyFilters(items: any[], params: Record<string, any>): any[] {
    let filtered = items;

    Object.entries(params).forEach(([key, value]) => {
      // Skip special params
      if (['sortBy', 'sortOrder', 'limit', 'offset'].includes(key)) return;

      filtered = filtered.filter(item => {
        const itemValue = item[key];

        // Handle array values (e.g., tags)
        if (Array.isArray(itemValue)) {
          return itemValue.includes(value);
        }

        // Handle date ranges
        if (key.endsWith('After') || key.endsWith('Before')) {
          const dateKey = key.replace(/After|Before$/, '');
          const itemDate = new Date(item[dateKey]);
          const compareDate = new Date(value);

          return key.endsWith('After')
            ? itemDate >= compareDate
            : itemDate <= compareDate;
        }

        // Exact match
        return itemValue === value;
      });
    });

    return filtered;
  }

  /**
   * Apply sorting to items.
   */
  private applySorting(items: any[], sortBy: string, sortOrder: 'asc' | 'desc' = 'asc'): any[] {
    return items.sort((a, b) => {
      const aVal = a[sortBy];
      const bVal = b[sortBy];

      let comparison = 0;

      if (aVal < bVal) comparison = -1;
      if (aVal > bVal) comparison = 1;

      return sortOrder === 'asc' ? comparison : -comparison;
    });
  }
}

interface RequestOptions {
  params?: Record<string, any>;
}
```

---

## 💾 IndexedDB Service

### Database Schema

```typescript
/**
 * IndexedDB schema for Nexus Studio.
 * Stores large datasets, file blobs, cached data.
 */
export const DB_NAME = 'nexus-studio';
export const DB_VERSION = 1;

export const STORES = {
  files: 'files',
  cases: 'cases',
  kanbanCards: 'kanban-cards',
  datasets: 'datasets',
  kbDocs: 'kb-docs',
  forms: 'forms',
  logs: 'logs',
  tokens: 'tokens',
  blobs: 'blobs',                      // Raw file content
  searchIndex: 'search-index',
  userPreferences: 'user-preferences',
} as const;
```

### IndexedDB Service Implementation

```typescript
/**
 * IndexedDB wrapper service.
 * Provides clean API for storing/retrieving data.
 */
@Injectable({ providedIn: 'root' })
export class IndexedDbService {
  private dbPromise: Promise<IDBDatabase>;

  constructor() {
    this.dbPromise = this.initDatabase();
  }

  /**
   * Initialize IndexedDB database.
   */
  private async initDatabase(): Promise<IDBDatabase> {
    return new Promise((resolve, reject) => {
      const request = indexedDB.open(DB_NAME, DB_VERSION);

      request.onerror = () => reject(request.error);
      request.onsuccess = () => resolve(request.result);

      request.onupgradeneeded = (event) => {
        const db = (event.target as IDBOpenDBRequest).result;

        // Create object stores
        Object.values(STORES).forEach(storeName => {
          if (!db.objectStoreNames.contains(storeName)) {
            const store = db.createObjectStore(storeName, { keyPath: 'id' });

            // Add indexes for common queries
            if (storeName === STORES.files) {
              store.createIndex('parentId', 'parentId', { unique: false });
              store.createIndex('type', 'type', { unique: false });
              store.createIndex('tenantId', 'tenantId', { unique: false });
            }

            if (storeName === STORES.cases) {
              store.createIndex('status', 'status', { unique: false });
              store.createIndex('assignedTo', 'assignedTo', { unique: false });
              store.createIndex('tenantId', 'tenantId', { unique: false });
            }

            // ... more indexes for other stores
          }
        });
      };
    });
  }

  /**
   * Get item by ID.
   */
  async getById<T>(storeName: string, id: string): Promise<T | undefined> {
    const db = await this.dbPromise;
    return new Promise((resolve, reject) => {
      const transaction = db.transaction(storeName, 'readonly');
      const store = transaction.objectStore(storeName);
      const request = store.get(id);

      request.onsuccess = () => resolve(request.result as T);
      request.onerror = () => reject(request.error);
    });
  }

  /**
   * Get all items in a store.
   */
  async getAll<T>(storeName: string): Promise<T[]> {
    const db = await this.dbPromise;
    return new Promise((resolve, reject) => {
      const transaction = db.transaction(storeName, 'readonly');
      const store = transaction.objectStore(storeName);
      const request = store.getAll();

      request.onsuccess = () => resolve(request.result as T[]);
      request.onerror = () => reject(request.error);
    });
  }

  /**
   * Add item to store.
   */
  async add<T>(storeName: string, item: T): Promise<void> {
    const db = await this.dbPromise;
    return new Promise((resolve, reject) => {
      const transaction = db.transaction(storeName, 'readwrite');
      const store = transaction.objectStore(storeName);
      const request = store.add(item);

      request.onsuccess = () => resolve();
      request.onerror = () => reject(request.error);
    });
  }

  /**
   * Update item in store.
   */
  async update<T>(storeName: string, id: string, item: T): Promise<void> {
    const db = await this.dbPromise;
    return new Promise((resolve, reject) => {
      const transaction = db.transaction(storeName, 'readwrite');
      const store = transaction.objectStore(storeName);
      const request = store.put(item);

      request.onsuccess = () => resolve();
      request.onerror = () => reject(request.error);
    });
  }

  /**
   * Delete item from store.
   */
  async delete(storeName: string, id: string): Promise<void> {
    const db = await this.dbPromise;
    return new Promise((resolve, reject) => {
      const transaction = db.transaction(storeName, 'readwrite');
      const store = transaction.objectStore(storeName);
      const request = store.delete(id);

      request.onsuccess = () => resolve();
      request.onerror = () => reject(request.error);
    });
  }

  /**
   * Query by index.
   */
  async queryByIndex<T>(
    storeName: string,
    indexName: string,
    value: any
  ): Promise<T[]> {
    const db = await this.dbPromise;
    return new Promise((resolve, reject) => {
      const transaction = db.transaction(storeName, 'readonly');
      const store = transaction.objectStore(storeName);
      const index = store.index(indexName);
      const request = index.getAll(value);

      request.onsuccess = () => resolve(request.result as T[]);
      request.onerror = () => reject(request.error);
    });
  }

  /**
   * Store blob (file content).
   */
  async storeBlob(blob: Blob): Promise<string> {
    const id = generateId();
    await this.add(STORES.blobs, { id, blob, createdAt: new Date() });
    return id;
  }

  /**
   * Get blob (file content).
   */
  async getBlob(id: string): Promise<Blob> {
    const result = await this.getById<{ blob: Blob }>(STORES.blobs, id);
    if (!result) throw new Error(`Blob ${id} not found`);
    return result.blob;
  }

  /**
   * Clear all data (for testing/reset).
   */
  async clearAll(): Promise<void> {
    const db = await this.dbPromise;
    const storeNames = Object.values(STORES);

    for (const storeName of storeNames) {
      await new Promise<void>((resolve, reject) => {
        const transaction = db.transaction(storeName, 'readwrite');
        const store = transaction.objectStore(storeName);
        const request = store.clear();

        request.onsuccess = () => resolve();
        request.onerror = () => reject(request.error);
      });
    }
  }
}
```

---

## 👷 Web Workers

### Dataset Profiler Worker

```typescript
// workers/dataset-profiler.worker.ts

/**
 * Dataset Profiler Worker
 * Analyzes CSV/JSON datasets to extract column stats.
 * Handles large datasets (100k+ rows) without freezing UI.
 */

interface ProfileRequest {
  datasetId: string;
  rows: any[];                         // Parsed rows
  columns: string[];                   // Column names
}

interface ProfileResult {
  datasetId: string;
  stats: DatasetStats;
  columnStats: ColumnStats[];
}

interface ColumnStats {
  name: string;
  type: ColumnType;
  distinctCount: number;
  nullCount: number;
  min?: any;
  max?: any;
  mean?: number;
  median?: number;
  mode?: any;
  sampleValues: any[];
}

self.onmessage = (event: MessageEvent<ProfileRequest>) => {
  const { datasetId, rows, columns } = event.data;

  // Profile each column
  const columnStats: ColumnStats[] = columns.map(colName =>
    profileColumn(rows, colName)
  );

  // Calculate dataset-level stats
  const stats: DatasetStats = {
    totalRows: rows.length,
    totalColumns: columns.length,
    memoryUsage: estimateMemoryUsage(rows),
    duplicateRows: countDuplicates(rows),
    completeness: calculateCompleteness(rows, columnStats),
    profiledAt: new Date(),
  };

  const result: ProfileResult = {
    datasetId,
    stats,
    columnStats,
  };

  self.postMessage(result);
};

function profileColumn(rows: any[], colName: string): ColumnStats {
  const values = rows.map(row => row[colName]);
  const nonNullValues = values.filter(v => v != null);

  // Infer type
  const type = inferType(nonNullValues);

  // Calculate stats
  const uniqueValues = new Set(nonNullValues);
  const nullCount = values.length - nonNullValues.length;

  let min, max, mean, median, mode;

  if (type === 'number') {
    const numbers = nonNullValues.map(Number);
    min = Math.min(...numbers);
    max = Math.max(...numbers);
    mean = numbers.reduce((a, b) => a + b, 0) / numbers.length;
    median = calculateMedian(numbers);
  } else if (type === 'date') {
    const dates = nonNullValues.map(v => new Date(v));
    min = new Date(Math.min(...dates.map(d => d.getTime())));
    max = new Date(Math.max(...dates.map(d => d.getTime())));
  }

  mode = calculateMode(nonNullValues);

  return {
    name: colName,
    type,
    distinctCount: uniqueValues.size,
    nullCount,
    min,
    max,
    mean,
    median,
    mode,
    sampleValues: Array.from(uniqueValues).slice(0, 10),
  };
}

function inferType(values: any[]): ColumnType {
  if (values.length === 0) return 'unknown';

  const sample = values.slice(0, 100);

  // Check if all values are numbers
  if (sample.every(v => !isNaN(Number(v)))) return 'number';

  // Check if all values are booleans
  if (sample.every(v => v === true || v === false || v === 'true' || v === 'false')) {
    return 'boolean';
  }

  // Check if all values are dates
  if (sample.every(v => !isNaN(Date.parse(v)))) return 'date';

  return 'string';
}

function calculateMedian(numbers: number[]): number {
  const sorted = numbers.slice().sort((a, b) => a - b);
  const mid = Math.floor(sorted.length / 2);
  return sorted.length % 2 === 0
    ? (sorted[mid - 1] + sorted[mid]) / 2
    : sorted[mid];
}

function calculateMode(values: any[]): any {
  const frequency = new Map<any, number>();
  values.forEach(v => {
    frequency.set(v, (frequency.get(v) || 0) + 1);
  });

  let maxFreq = 0;
  let mode = null;
  frequency.forEach((count, value) => {
    if (count > maxFreq) {
      maxFreq = count;
      mode = value;
    }
  });

  return mode;
}

function estimateMemoryUsage(rows: any[]): number {
  // Rough estimate: average row size × row count
  const sampleRow = JSON.stringify(rows[0] || {});
  return sampleRow.length * rows.length;
}

function countDuplicates(rows: any[]): number {
  const seen = new Set<string>();
  let duplicates = 0;

  rows.forEach(row => {
    const key = JSON.stringify(row);
    if (seen.has(key)) {
      duplicates++;
    } else {
      seen.add(key);
    }
  });

  return duplicates;
}

function calculateCompleteness(rows: any[], columnStats: ColumnStats[]): number {
  const totalCells = rows.length * columnStats.length;
  const nullCells = columnStats.reduce((sum, col) => sum + col.nullCount, 0);
  return ((totalCells - nullCells) / totalCells) * 100;
}
```

---

## ✅ Data Layer Checklist

**Data Models**:
- ✅ Case (support tickets)
- ✅ Kanban Card (project tasks)
- ✅ Dataset (tabular data)
- ✅ KB Document (wiki pages)
- ✅ Form Schema (dynamic forms)
- ✅ Log Entry (observability)
- ✅ Design Token (design system)

**Mock API**:
- ✅ GET/POST/PATCH/DELETE operations
- ✅ Query filtering and sorting
- ✅ Pagination support
- ✅ Realistic delays (200ms)
- ✅ Random error simulation (2%)

**IndexedDB**:
- ✅ Database initialization and schema
- ✅ CRUD operations
- ✅ Index-based queries
- ✅ Blob storage for files
- ✅ Clear/reset functionality

**Web Workers**:
- ✅ Search worker (full-text indexing)
- ✅ Dataset profiler worker (column stats)
- ✅ Message passing API
- ✅ Background processing

---

**Next**: ARCHITECTURE.md (Module boundaries and integration)
