# Shared Systems & Cross-Cutting Concerns

> **The foundation that makes 10 modules behave as one unified product**

---

## 🎯 What & Why

### What Are Shared Systems?

Shared systems are the **cross-cutting concerns** that multiple modules depend on:
- Unified data models (files, users, tenants)
- Multi-tenancy + permissions engine
- Global search indexing
- Activity/audit logging
- Notification infrastructure
- Design token system

### Why Are They Critical?

**Without shared systems**: Each module would reimplement the same concepts differently, leading to:
- Inconsistent UX (files work differently in each module)
- Broken workflows (can't link a case to a dataset)
- Duplicated code and bugs
- No global search (each module searches separately)

**With shared systems**: Modules integrate seamlessly:
- Files work the same way everywhere
- Cross-module references just work
- Single source of truth for permissions
- Global search finds anything
- Consistent audit trail

---

## 📁 Unified File/Asset Model

### The Problem

Five modules deal with files/assets:
1. **File Manager**: General-purpose files (docs, PDFs, etc.)
2. **Media Library**: Images, videos, audio files
3. **Knowledge Base**: Markdown documents
4. **Dataset Explorer**: CSV, JSON, Parquet data files
5. **Form Builder**: Form schema JSON files

**Without unification**: Each module has its own "file" concept, no interoperability.

**With unification**: One `FileNode` model, specialized by type.

---

### Core File Model

```typescript
/**
 * Base file/asset entity used across all modules.
 * Think of this as a "universal file descriptor" like in a filesystem.
 */
export interface FileNode {
  // Identity
  id: string;                          // UUID
  tenantId: string;                    // Multi-tenant isolation

  // Core Metadata
  name: string;                        // "customer-data.csv"
  path: string;                        // "/projects/2024/customer-data.csv"
  type: FileNodeType;                  // Determines which module handles it
  mimeType: string;                    // "text/csv", "image/png", etc.

  // Filesystem Properties
  size: number;                        // Bytes
  createdAt: Date;
  modifiedAt: Date;
  createdBy: string;                   // User ID
  modifiedBy: string;                  // User ID

  // Hierarchy
  parentId: string | null;             // Null for root items
  isDirectory: boolean;
  childCount?: number;                 // For directories, how many children

  // Content
  contentUrl?: string;                 // URL to actual content (blob, IndexedDB key, etc.)
  thumbnailUrl?: string;               // Preview image (for media, PDFs)

  // Permissions
  permissions: FilePermissions;
  isPublic: boolean;                   // Public link accessible
  shareToken?: string;                 // For public sharing

  // Extended Metadata (type-specific)
  metadata: FileMetadata;

  // References
  tags: string[];                      // User-defined tags
  linkedEntities: EntityReference[];   // Links to cases, kanban cards, etc.

  // State
  isLocked: boolean;                   // Locked for editing
  lockedBy?: string;                   // User ID who locked it
  version: number;                     // Version number (for versioned files)
  previousVersionId?: string;          // Link to previous version

  // Search
  searchableText?: string;             // Full-text indexed content
}

export type FileNodeType =
  | 'folder'
  | 'document'          // PDFs, Word docs, text files (File Manager)
  | 'image'             // PNG, JPG, SVG (Media Library)
  | 'video'             // MP4, WebM (Media Library)
  | 'audio'             // MP3, WAV (Media Library)
  | 'markdown'          // Markdown docs (Knowledge Base)
  | 'dataset'           // CSV, JSON, Parquet (Dataset Explorer)
  | 'form-schema'       // JSON form definitions (Form Builder)
  | 'code'              // JS, TS, Python, etc.
  | 'archive'           // ZIP, TAR
  | 'unknown';          // Unrecognized type

export interface FilePermissions {
  canRead: boolean;
  canWrite: boolean;
  canDelete: boolean;
  canShare: boolean;
  inheritFromParent: boolean;
}

export interface FileMetadata {
  // Image-specific
  width?: number;
  height?: number;
  colorSpace?: string;
  exif?: Record<string, any>;

  // Video-specific
  duration?: number;                   // Seconds
  codec?: string;
  bitrate?: number;

  // Dataset-specific
  rowCount?: number;
  columnCount?: number;
  columns?: ColumnMetadata[];

  // Document-specific
  pageCount?: number;
  author?: string;
  title?: string;

  // Custom metadata
  [key: string]: any;
}

export interface ColumnMetadata {
  name: string;
  type: 'string' | 'number' | 'boolean' | 'date' | 'unknown';
  nullable: boolean;
  uniqueCount?: number;
  min?: number;
  max?: number;
  mean?: number;
}

export interface EntityReference {
  entityType: 'case' | 'kanban-card' | 'kb-doc' | 'form' | 'user';
  entityId: string;
  label: string;                       // Display text
  createdAt: Date;
}
```

---

### File Operations API

```typescript
/**
 * Unified file service used by all modules.
 * Abstracts storage layer (IndexedDB, mock API, future: real backend).
 */
@Injectable({ providedIn: 'root' })
export class FileService {
  private api = inject(MockApiService);
  private db = inject(IndexedDbService);

  /**
   * List files in a directory.
   * Used by: File Manager, Media Library, Knowledge Base
   */
  async listFiles(parentId: string | null, options?: ListOptions): Promise<FileNode[]> {
    const files = await this.api.get<FileNode[]>('/files', {
      params: {
        parentId: parentId || 'root',
        type: options?.type,
        sortBy: options?.sortBy || 'name',
        sortOrder: options?.sortOrder || 'asc',
      },
    });

    return files;
  }

  /**
   * Get file by ID.
   */
  async getFile(fileId: string): Promise<FileNode> {
    return this.api.get<FileNode>(`/files/${fileId}`);
  }

  /**
   * Upload new file.
   * Handles file content storage + metadata creation.
   */
  async uploadFile(
    file: File,
    parentId: string | null,
    type: FileNodeType
  ): Promise<FileNode> {
    // Store content in IndexedDB
    const contentKey = await this.db.storeBlob(file);

    // Create metadata
    const fileNode: Partial<FileNode> = {
      name: file.name,
      type,
      mimeType: file.type,
      size: file.size,
      parentId,
      contentUrl: `idb://${contentKey}`,
      createdAt: new Date(),
      modifiedAt: new Date(),
      isDirectory: false,
      permissions: this.getDefaultPermissions(),
      metadata: await this.extractMetadata(file, type),
      tags: [],
      linkedEntities: [],
      isLocked: false,
      version: 1,
    };

    // Save to mock API
    return this.api.post<FileNode>('/files', fileNode);
  }

  /**
   * Create directory.
   */
  async createDirectory(name: string, parentId: string | null): Promise<FileNode> {
    const folder: Partial<FileNode> = {
      name,
      type: 'folder',
      isDirectory: true,
      parentId,
      size: 0,
      createdAt: new Date(),
      modifiedAt: new Date(),
      permissions: this.getDefaultPermissions(),
      metadata: {},
      tags: [],
      linkedEntities: [],
      isLocked: false,
      version: 1,
    };

    return this.api.post<FileNode>('/files', folder);
  }

  /**
   * Update file metadata (rename, tag, etc.).
   */
  async updateFile(fileId: string, updates: Partial<FileNode>): Promise<FileNode> {
    return this.api.patch<FileNode>(`/files/${fileId}`, updates);
  }

  /**
   * Delete file (soft delete with trash).
   */
  async deleteFile(fileId: string, permanent: boolean = false): Promise<void> {
    if (permanent) {
      await this.api.delete(`/files/${fileId}`);
    } else {
      // Move to trash
      await this.api.patch(`/files/${fileId}`, { trashedAt: new Date() });
    }
  }

  /**
   * Move file to different directory.
   */
  async moveFile(fileId: string, newParentId: string): Promise<FileNode> {
    return this.api.patch<FileNode>(`/files/${fileId}`, { parentId: newParentId });
  }

  /**
   * Get file content (as Blob).
   */
  async getFileContent(fileId: string): Promise<Blob> {
    const file = await this.getFile(fileId);
    if (!file.contentUrl) throw new Error('No content URL');

    // Extract IndexedDB key from URL
    const key = file.contentUrl.replace('idb://', '');
    return this.db.getBlob(key);
  }

  /**
   * Search files globally.
   * Used by global search, module-specific searches.
   */
  async searchFiles(query: string, filters?: FileSearchFilters): Promise<FileNode[]> {
    return this.api.get<FileNode[]>('/files/search', {
      params: { q: query, ...filters },
    });
  }

  /**
   * Get file tree (for File Manager sidebar).
   */
  async getFileTree(rootId: string | null = null): Promise<FileTreeNode[]> {
    const files = await this.listFiles(rootId);
    const folders = files.filter(f => f.isDirectory);

    return Promise.all(
      folders.map(async folder => ({
        ...folder,
        children: await this.getFileTree(folder.id),
      }))
    );
  }

  /**
   * Link file to another entity (case, kanban card, etc.).
   */
  async linkToEntity(fileId: string, reference: EntityReference): Promise<void> {
    const file = await this.getFile(fileId);
    const linkedEntities = [...file.linkedEntities, reference];
    await this.updateFile(fileId, { linkedEntities });
  }

  private async extractMetadata(file: File, type: FileNodeType): Promise<FileMetadata> {
    const metadata: FileMetadata = {};

    if (type === 'image') {
      // Extract image dimensions
      const img = await this.loadImage(file);
      metadata.width = img.width;
      metadata.height = img.height;
    } else if (type === 'dataset') {
      // Parse CSV headers
      const text = await file.text();
      const lines = text.split('\n');
      metadata.rowCount = lines.length - 1; // Exclude header
      metadata.columns = this.parseCSVHeaders(lines[0]);
    }

    return metadata;
  }

  private loadImage(file: File): Promise<HTMLImageElement> {
    return new Promise((resolve, reject) => {
      const img = new Image();
      img.onload = () => resolve(img);
      img.onerror = reject;
      img.src = URL.createObjectURL(file);
    });
  }

  private parseCSVHeaders(headerLine: string): ColumnMetadata[] {
    return headerLine.split(',').map(name => ({
      name: name.trim(),
      type: 'unknown',
      nullable: true,
    }));
  }

  private getDefaultPermissions(): FilePermissions {
    return {
      canRead: true,
      canWrite: true,
      canDelete: true,
      canShare: true,
      inheritFromParent: true,
    };
  }
}

interface ListOptions {
  type?: FileNodeType;
  sortBy?: 'name' | 'modifiedAt' | 'size';
  sortOrder?: 'asc' | 'desc';
}

interface FileSearchFilters {
  type?: FileNodeType;
  tags?: string[];
  modifiedAfter?: Date;
  modifiedBefore?: Date;
}

interface FileTreeNode extends FileNode {
  children?: FileTreeNode[];
}
```

---

## 🔐 Multi-Tenancy & Permissions

### Tenant Model

```typescript
/**
 * Tenant: Isolated workspace for a team/organization.
 * All data (files, cases, users, etc.) belongs to a tenant.
 */
export interface Tenant {
  id: string;                          // UUID
  name: string;                        // "Acme Corp"
  slug: string;                        // "acme" (used in URLs)

  // Branding
  logo?: string;                       // URL to logo
  primaryColor?: string;               // Brand color

  // Settings
  settings: TenantSettings;

  // Subscription
  plan: 'free' | 'pro' | 'enterprise';
  maxUsers: number;
  maxStorage: number;                  // Bytes

  // Metadata
  createdAt: Date;
  ownerId: string;                     // User ID of tenant owner

  // Feature Flags
  features: FeatureFlags;
}

export interface TenantSettings {
  allowPublicSharing: boolean;
  requireMFA: boolean;
  sessionTimeout: number;              // Minutes
  defaultTheme: 'light' | 'dark';
  customDomain?: string;
}

export interface FeatureFlags {
  datasetExplorer: boolean;
  formBuilder: boolean;
  advancedAnalytics: boolean;
  apiAccess: boolean;
  auditLog: boolean;
  [key: string]: boolean;
}
```

### User Model

```typescript
/**
 * User: Person with access to one or more tenants.
 */
export interface User {
  id: string;                          // UUID
  email: string;
  name: string;
  avatar?: string;

  // Multi-tenant memberships
  tenants: UserTenantMembership[];

  // Preferences
  preferences: UserPreferences;

  // Metadata
  createdAt: Date;
  lastLoginAt?: Date;
  isActive: boolean;
}

export interface UserTenantMembership {
  tenantId: string;
  roleId: string;                      // Role within this tenant
  joinedAt: Date;
  invitedBy?: string;                  // User ID
}

export interface UserPreferences {
  theme: 'light' | 'dark' | 'high-contrast';
  language: string;
  timezone: string;
  notifications: NotificationPreferences;
}

export interface NotificationPreferences {
  email: boolean;
  inApp: boolean;
  desktop: boolean;
  digestFrequency: 'realtime' | 'daily' | 'weekly' | 'never';
}
```

### Role-Based Access Control (RBAC)

```typescript
/**
 * Role: Collection of permissions.
 * Users are assigned roles within a tenant.
 */
export interface Role {
  id: string;
  tenantId: string;
  name: string;                        // "Admin", "Editor", "Viewer"
  description: string;

  // Permissions
  permissions: Permission[];

  // System roles (can't be deleted)
  isSystemRole: boolean;

  // Metadata
  createdAt: Date;
  createdBy: string;
}

/**
 * Permission: Fine-grained access control.
 * Format: "resource:action" (e.g., "files:read", "cases:write")
 */
export type Permission = string;

/**
 * Permission matrix: What each role can do.
 */
export const PERMISSION_MATRIX: Record<string, Permission[]> = {
  // Admin: Full access
  admin: [
    'files:read',
    'files:write',
    'files:delete',
    'files:share',
    'cases:read',
    'cases:write',
    'cases:delete',
    'cases:assign',
    'datasets:read',
    'datasets:write',
    'datasets:delete',
    'kb:read',
    'kb:write',
    'kb:delete',
    'forms:read',
    'forms:write',
    'forms:delete',
    'forms:publish',
    'users:read',
    'users:write',
    'users:delete',
    'roles:read',
    'roles:write',
    'audit:read',
  ],

  // Editor: Can create/edit, but not delete or manage users
  editor: [
    'files:read',
    'files:write',
    'files:share',
    'cases:read',
    'cases:write',
    'cases:assign',
    'datasets:read',
    'datasets:write',
    'kb:read',
    'kb:write',
    'forms:read',
    'forms:write',
  ],

  // Viewer: Read-only access
  viewer: [
    'files:read',
    'cases:read',
    'datasets:read',
    'kb:read',
    'forms:read',
  ],
};
```

### Permissions Service

```typescript
/**
 * Service to check user permissions.
 * Used throughout the app to show/hide features, guard routes, etc.
 */
@Injectable({ providedIn: 'root' })
export class PermissionsService {
  private authService = inject(AuthService);
  private tenantService = inject(TenantService);

  private currentUser = this.authService.currentUser;
  private currentTenant = this.tenantService.currentTenant;

  /**
   * Check if current user has a permission.
   */
  hasPermission(permission: Permission): boolean {
    const user = this.currentUser();
    const tenant = this.currentTenant();
    if (!user || !tenant) return false;

    // Find user's role in current tenant
    const membership = user.tenants.find(t => t.tenantId === tenant.id);
    if (!membership) return false;

    // Get role permissions
    const role = this.getRoleById(membership.roleId);
    if (!role) return false;

    return role.permissions.includes(permission);
  }

  /**
   * Check if current user has a specific role.
   */
  hasRole(roleName: string): boolean {
    const user = this.currentUser();
    const tenant = this.currentTenant();
    if (!user || !tenant) return false;

    const membership = user.tenants.find(t => t.tenantId === tenant.id);
    if (!membership) return false;

    const role = this.getRoleById(membership.roleId);
    return role?.name.toLowerCase() === roleName.toLowerCase();
  }

  /**
   * Check multiple permissions (AND logic).
   */
  hasAllPermissions(permissions: Permission[]): boolean {
    return permissions.every(p => this.hasPermission(p));
  }

  /**
   * Check multiple permissions (OR logic).
   */
  hasAnyPermission(permissions: Permission[]): boolean {
    return permissions.some(p => this.hasPermission(p));
  }

  /**
   * Check if user can access a specific file.
   * Respects file-level permissions + role permissions.
   */
  canAccessFile(file: FileNode, action: 'read' | 'write' | 'delete' | 'share'): boolean {
    // Check role permission first
    if (!this.hasPermission(`files:${action}`)) return false;

    // Then check file-level permissions
    switch (action) {
      case 'read': return file.permissions.canRead;
      case 'write': return file.permissions.canWrite;
      case 'delete': return file.permissions.canDelete;
      case 'share': return file.permissions.canShare;
    }
  }

  /**
   * Check if feature flag is enabled for current tenant.
   */
  hasFeature(feature: keyof FeatureFlags): boolean {
    const tenant = this.currentTenant();
    return tenant?.features[feature] ?? false;
  }

  private getRoleById(roleId: string): Role | null {
    // In real app, would fetch from store/service
    // For now, return mock based on roleId
    return null;
  }
}
```

### Permission Directives

```typescript
/**
 * Structural directive to show/hide elements based on permissions.
 * Usage: <button *appHasPermission="'files:delete'">Delete</button>
 */
@Directive({
  selector: '[appHasPermission]',
  standalone: true,
})
export class HasPermissionDirective {
  private viewContainer = inject(ViewContainerRef);
  private templateRef = inject(TemplateRef);
  private permissions = inject(PermissionsService);

  @Input() set appHasPermission(permission: Permission) {
    if (this.permissions.hasPermission(permission)) {
      this.viewContainer.createEmbeddedView(this.templateRef);
    } else {
      this.viewContainer.clear();
    }
  }
}

/**
 * Directive to disable elements based on permissions.
 * Usage: <button [appDisableIfNoPermission]="'files:delete'">Delete</button>
 */
@Directive({
  selector: '[appDisableIfNoPermission]',
  standalone: true,
})
export class DisableIfNoPermissionDirective {
  private element = inject(ElementRef);
  private renderer = inject(Renderer2);
  private permissions = inject(PermissionsService);

  @Input() set appDisableIfNoPermission(permission: Permission) {
    if (!this.permissions.hasPermission(permission)) {
      this.renderer.setAttribute(this.element.nativeElement, 'disabled', 'true');
      this.renderer.addClass(this.element.nativeElement, 'disabled');
    }
  }
}
```

---

## 🔍 Global Search Indexing

### Search Architecture

```
┌─────────────────────────────────────────────────┐
│ User types in search box                        │
└────────────────┬────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────┐
│ Search Service (debounced 200ms)                │
└────────────────┬────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────┐
│ Post message to Search Worker                   │
│ (runs search in background thread)              │
└────────────────┬────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────┐
│ Search Worker                                   │
│ - Tokenizes query                               │
│ - Looks up in inverted index                    │
│ - Ranks results (TF-IDF)                        │
│ - Returns top 50 results                        │
└────────────────┬────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────┐
│ Search Service receives results                 │
│ - Groups by module                              │
│ - Highlights matching terms                     │
│ - Updates UI signal                             │
└─────────────────────────────────────────────────┘
```

### Search Document Model

```typescript
/**
 * Indexed document for full-text search.
 * Every searchable entity (file, case, card, etc.) becomes a SearchDocument.
 */
export interface SearchDocument {
  id: string;                          // Entity ID
  type: SearchDocumentType;            // Which module
  title: string;                       // Primary text (shown in results)
  content: string;                     // Full searchable text
  url: string;                         // Deep link to entity

  // Metadata (for filtering)
  tenantId: string;
  createdAt: Date;
  modifiedAt: Date;
  createdBy: string;
  tags: string[];

  // Module-specific metadata
  metadata: Record<string, any>;

  // Search ranking boost
  boost: number;                       // 1.0 = normal, 2.0 = double weight
}

export type SearchDocumentType =
  | 'file'
  | 'case'
  | 'kanban-card'
  | 'kb-doc'
  | 'dataset'
  | 'form'
  | 'user'
  | 'log-entry';
```

### Search Worker

```typescript
// workers/search-worker.ts

/**
 * Web Worker for full-text search.
 * Keeps UI responsive even when searching 100k+ documents.
 */

interface SearchIndex {
  documents: Map<string, SearchDocument>;
  invertedIndex: Map<string, Set<string>>;  // term → document IDs
  termFrequency: Map<string, number>;       // term → document count
}

const searchIndex: SearchIndex = {
  documents: new Map(),
  invertedIndex: new Map(),
  termFrequency: new Map(),
};

/**
 * Handle messages from main thread.
 */
self.onmessage = (event: MessageEvent) => {
  const { type, payload } = event.data;

  switch (type) {
    case 'index':
      indexDocuments(payload.documents);
      self.postMessage({ type: 'indexed', count: payload.documents.length });
      break;

    case 'search':
      const results = search(payload.query, payload.filters);
      self.postMessage({ type: 'results', results });
      break;

    case 'clear':
      clearIndex();
      self.postMessage({ type: 'cleared' });
      break;
  }
};

/**
 * Add documents to search index.
 */
function indexDocuments(documents: SearchDocument[]) {
  documents.forEach(doc => {
    searchIndex.documents.set(doc.id, doc);

    // Tokenize content
    const tokens = tokenize(doc.title + ' ' + doc.content);

    // Build inverted index
    tokens.forEach(token => {
      if (!searchIndex.invertedIndex.has(token)) {
        searchIndex.invertedIndex.set(token, new Set());
      }
      searchIndex.invertedIndex.get(token)!.add(doc.id);

      // Update term frequency
      const count = searchIndex.termFrequency.get(token) || 0;
      searchIndex.termFrequency.set(token, count + 1);
    });
  });
}

/**
 * Search indexed documents.
 */
function search(query: string, filters?: SearchFilters): SearchResult[] {
  const queryTokens = tokenize(query);
  const matchingDocs = new Map<string, number>(); // doc ID → score

  // Find documents matching query tokens
  queryTokens.forEach(token => {
    const docIds = searchIndex.invertedIndex.get(token);
    if (!docIds) return;

    docIds.forEach(docId => {
      const doc = searchIndex.documents.get(docId)!;

      // Apply filters
      if (filters) {
        if (filters.type && doc.type !== filters.type) return;
        if (filters.tags && !filters.tags.some(tag => doc.tags.includes(tag))) return;
      }

      // Calculate TF-IDF score
      const score = calculateTFIDF(token, docId);
      const currentScore = matchingDocs.get(docId) || 0;
      matchingDocs.set(docId, currentScore + score * doc.boost);
    });
  });

  // Sort by score and return top 50
  const results = Array.from(matchingDocs.entries())
    .map(([docId, score]) => {
      const doc = searchIndex.documents.get(docId)!;
      return {
        ...doc,
        score,
        highlights: findHighlights(doc, queryTokens),
      };
    })
    .sort((a, b) => b.score - a.score)
    .slice(0, 50);

  return results;
}

/**
 * Tokenize text into searchable terms.
 */
function tokenize(text: string): string[] {
  return text
    .toLowerCase()
    .replace(/[^\w\s]/g, ' ')           // Remove punctuation
    .split(/\s+/)                       // Split on whitespace
    .filter(token => token.length > 2)  // Min 3 chars
    .filter(token => !isStopWord(token)); // Remove "the", "is", etc.
}

/**
 * Calculate TF-IDF score for a term in a document.
 */
function calculateTFIDF(term: string, docId: string): number {
  const doc = searchIndex.documents.get(docId)!;
  const content = (doc.title + ' ' + doc.content).toLowerCase();

  // Term Frequency (TF): How many times term appears in doc
  const termCount = (content.match(new RegExp(term, 'g')) || []).length;
  const tf = termCount / content.split(/\s+/).length;

  // Inverse Document Frequency (IDF): How rare is this term
  const docCount = searchIndex.documents.size;
  const termDocCount = searchIndex.termFrequency.get(term) || 1;
  const idf = Math.log(docCount / termDocCount);

  return tf * idf;
}

/**
 * Find text snippets to highlight in results.
 */
function findHighlights(doc: SearchDocument, queryTokens: string[]): string[] {
  const content = doc.content.toLowerCase();
  const highlights: string[] = [];

  queryTokens.forEach(token => {
    const index = content.indexOf(token);
    if (index !== -1) {
      const start = Math.max(0, index - 40);
      const end = Math.min(content.length, index + token.length + 40);
      const snippet = doc.content.substring(start, end);
      highlights.push(snippet);
    }
  });

  return highlights.slice(0, 3); // Max 3 highlights per result
}

function isStopWord(word: string): boolean {
  const stopWords = new Set(['the', 'is', 'at', 'which', 'on', 'a', 'an', 'and', 'or', 'but']);
  return stopWords.has(word);
}

function clearIndex() {
  searchIndex.documents.clear();
  searchIndex.invertedIndex.clear();
  searchIndex.termFrequency.clear();
}

interface SearchFilters {
  type?: SearchDocumentType;
  tags?: string[];
}

interface SearchResult extends SearchDocument {
  score: number;
  highlights: string[];
}
```

### Search Service (Main Thread)

```typescript
/**
 * Service to interact with search worker.
 */
@Injectable({ providedIn: 'root' })
export class SearchService {
  private worker: Worker;
  private searchResults = signal<SearchResult[]>([]);
  private isSearching = signal(false);

  constructor() {
    // Initialize search worker
    this.worker = new Worker(new URL('../workers/search-worker.ts', import.meta.url), {
      type: 'module',
    });

    // Listen to worker messages
    this.worker.onmessage = (event) => {
      const { type, results } = event.data;

      if (type === 'results') {
        this.searchResults.set(results);
        this.isSearching.set(false);
      }
    };

    // Index all searchable content on init
    this.buildInitialIndex();
  }

  /**
   * Perform search (debounced).
   */
  search = rxMethod<string>(
    pipe(
      debounceTime(200),
      distinctUntilChanged(),
      tap(() => this.isSearching.set(true)),
      tap(query => {
        this.worker.postMessage({
          type: 'search',
          payload: { query },
        });
      })
    )
  );

  /**
   * Add documents to index.
   */
  indexDocuments(documents: SearchDocument[]) {
    this.worker.postMessage({
      type: 'index',
      payload: { documents },
    });
  }

  /**
   * Build initial search index from all modules.
   */
  private async buildInitialIndex() {
    const documents: SearchDocument[] = [];

    // Index files
    const files = await inject(FileService).searchFiles('');
    documents.push(...files.map(this.fileToSearchDoc));

    // Index cases (would fetch from CaseService)
    // Index kanban cards
    // Index KB docs
    // etc.

    this.indexDocuments(documents);
  }

  private fileToSearchDoc(file: FileNode): SearchDocument {
    return {
      id: file.id,
      type: 'file',
      title: file.name,
      content: file.searchableText || '',
      url: `/t/${file.tenantId}/files/file/${file.id}`,
      tenantId: file.tenantId,
      createdAt: file.createdAt,
      modifiedAt: file.modifiedAt,
      createdBy: file.createdBy,
      tags: file.tags,
      metadata: { mimeType: file.mimeType, size: file.size },
      boost: 1.0,
    };
  }
}
```

---

## 📝 Activity & Audit Log

### Activity Event Model

```typescript
/**
 * Activity: User action that should be logged and shown in activity feed.
 */
export interface ActivityEvent {
  id: string;
  tenantId: string;

  // Who
  userId: string;
  userName: string;
  userAvatar?: string;

  // What
  action: ActivityAction;
  entityType: string;                  // 'file', 'case', 'kanban-card', etc.
  entityId: string;
  entityName: string;                  // Display name

  // When
  timestamp: Date;

  // Where (optional, for deep linking)
  url?: string;

  // Additional context
  metadata?: Record<string, any>;

  // For audit compliance
  ipAddress?: string;
  userAgent?: string;
}

export type ActivityAction =
  | 'created'
  | 'updated'
  | 'deleted'
  | 'viewed'
  | 'shared'
  | 'commented'
  | 'assigned'
  | 'uploaded'
  | 'downloaded'
  | 'exported'
  | 'imported';
```

### Activity Service

```typescript
/**
 * Service to log and query activity events.
 */
@Injectable({ providedIn: 'root' })
export class ActivityService {
  private api = inject(MockApiService);
  private auth = inject(AuthService);

  /**
   * Log an activity event.
   */
  async logActivity(event: Omit<ActivityEvent, 'id' | 'timestamp' | 'userId' | 'userName'>): Promise<void> {
    const user = this.auth.currentUser();
    if (!user) return;

    const fullEvent: ActivityEvent = {
      ...event,
      id: generateId(),
      userId: user.id,
      userName: user.name,
      userAvatar: user.avatar,
      timestamp: new Date(),
    };

    await this.api.post('/activity', fullEvent);
  }

  /**
   * Get recent activity (for activity feed).
   */
  async getRecentActivity(limit: number = 50): Promise<ActivityEvent[]> {
    return this.api.get<ActivityEvent[]>('/activity', {
      params: { limit, sortBy: 'timestamp', sortOrder: 'desc' },
    });
  }

  /**
   * Get activity for a specific entity (for entity timeline).
   */
  async getEntityActivity(entityId: string): Promise<ActivityEvent[]> {
    return this.api.get<ActivityEvent[]>(`/activity/entity/${entityId}`);
  }
}
```

---

## 🔔 Notifications

### Notification Model

```typescript
/**
 * Notification: In-app alert for user.
 */
export interface Notification {
  id: string;
  tenantId: string;
  userId: string;                      // Recipient

  // Content
  type: NotificationType;
  title: string;
  message: string;
  icon?: string;

  // Action
  actionUrl?: string;                  // Where to go on click
  actionLabel?: string;                // "View Case", "Open File", etc.

  // State
  read: boolean;
  readAt?: Date;

  // Metadata
  createdAt: Date;
  expiresAt?: Date;                    // Auto-dismiss after date

  // Priority (affects visual styling)
  priority: 'low' | 'normal' | 'high';
}

export type NotificationType =
  | 'mention'               // "@you in comment"
  | 'assignment'            // "You were assigned to case #123"
  | 'update'                // "Case #123 was updated"
  | 'system'                // "Maintenance scheduled"
  | 'success'               // "File uploaded successfully"
  | 'warning'               // "Storage almost full"
  | 'error';                // "Upload failed"
```

### Notification Service

```typescript
/**
 * Service to manage notifications.
 */
@Injectable({ providedIn: 'root' })
export class NotificationService {
  private api = inject(MockApiService);
  private auth = inject(AuthService);

  private notifications = signal<Notification[]>([]);

  unreadCount = computed(() =>
    this.notifications().filter(n => !n.read).length
  );

  constructor() {
    // Load notifications on init
    this.loadNotifications();

    // Poll for new notifications every 30s
    interval(30000).subscribe(() => this.loadNotifications());
  }

  async loadNotifications() {
    const user = this.auth.currentUser();
    if (!user) return;

    const notifications = await this.api.get<Notification[]>('/notifications', {
      params: { userId: user.id },
    });

    this.notifications.set(notifications);
  }

  async markAsRead(notificationId: string) {
    await this.api.patch(`/notifications/${notificationId}`, { read: true, readAt: new Date() });

    this.notifications.update(notifications =>
      notifications.map(n =>
        n.id === notificationId ? { ...n, read: true, readAt: new Date() } : n
      )
    );
  }

  async markAllAsRead() {
    const user = this.auth.currentUser();
    if (!user) return;

    await this.api.post('/notifications/mark-all-read', { userId: user.id });

    this.notifications.update(notifications =>
      notifications.map(n => ({ ...n, read: true, readAt: new Date() }))
    );
  }

  async createNotification(notification: Omit<Notification, 'id' | 'createdAt' | 'read'>) {
    const fullNotification: Notification = {
      ...notification,
      id: generateId(),
      createdAt: new Date(),
      read: false,
    };

    await this.api.post('/notifications', fullNotification);
  }
}
```

---

## ✅ Shared Systems Checklist

**Unified File/Asset Model**:
- ✅ `FileNode` interface for all file types
- ✅ Unified `FileService` API
- ✅ Type-specific metadata
- ✅ Cross-module entity references
- ✅ Version control support

**Multi-Tenancy & Permissions**:
- ✅ Tenant model with feature flags
- ✅ User model with multi-tenant membership
- ✅ Role-based access control (RBAC)
- ✅ Permission matrix (admin, editor, viewer)
- ✅ `PermissionsService` for access checks
- ✅ Permission directives for UI

**Global Search**:
- ✅ `SearchDocument` model
- ✅ Search worker for background indexing
- ✅ TF-IDF ranking algorithm
- ✅ Tokenization and stop words
- ✅ Highlighting matched terms
- ✅ `SearchService` with debouncing

**Activity & Audit**:
- ✅ `ActivityEvent` model
- ✅ `ActivityService` for logging
- ✅ Entity timeline support
- ✅ Audit compliance fields (IP, user agent)

**Notifications**:
- ✅ `Notification` model
- ✅ `NotificationService` with polling
- ✅ Unread count computed signal
- ✅ Mark as read/unread functionality

---

**Next**: DATA_LAYER.md (Mock API, IndexedDB, full data models)
