# Contributing to Nexus Studio

> Guidelines for contributing to Nexus Studio - Enterprise Operations Hub

Thank you for your interest in contributing to Nexus Studio! This document provides guidelines and best practices for contributing to this project.

## Table of Contents
1. [Code of Conduct](#code-of-conduct)
2. [Getting Started](#getting-started)
3. [Development Workflow](#development-workflow)
4. [Architecture Principles](#architecture-principles)
5. [Code Style Standards](#code-style-standards)
6. [Commit Conventions](#commit-conventions)
7. [Pull Request Process](#pull-request-process)
8. [Testing Requirements](#testing-requirements)
9. [Documentation Requirements](#documentation-requirements)
10. [Module Development Guide](#module-development-guide)

---

## Code of Conduct

### Our Standards

- **Be Respectful**: Treat all contributors with respect and professionalism
- **Be Constructive**: Provide helpful feedback and accept criticism gracefully
- **Be Collaborative**: Work together to improve the codebase
- **Be Inclusive**: Welcome contributors of all backgrounds and experience levels

### Unacceptable Behavior

- Harassment, discrimination, or personal attacks
- Trolling, insulting comments, or unconstructive criticism
- Publishing private information without consent
- Any conduct that would be inappropriate in a professional setting

---

## Getting Started

### Prerequisites

- **Node.js**: v20.x or higher
- **npm**: v10.x or higher
- **Git**: Latest version
- **IDE**: VS Code recommended (with Angular Language Service extension)

### Initial Setup

```bash
# Clone the repository
git clone https://github.com/your-org/nexus-studio.git
cd nexus-studio

# Install dependencies
npm install

# Start development server
npm start

# Open browser
# Navigate to http://localhost:4200/t/demo-tenant/files
```

### Recommended VS Code Extensions

- **Angular Language Service**: Autocomplete and type checking in templates
- **Prettier**: Code formatting
- **ESLint**: Linting
- **axe Accessibility Linter**: Accessibility checking
- **GitLens**: Git integration

---

## Development Workflow

### Branch Strategy

We use **Git Flow** with the following branches:

- `main`: Production-ready code
- `develop`: Integration branch for features
- `feature/*`: New features
- `fix/*`: Bug fixes
- `docs/*`: Documentation updates
- `refactor/*`: Code refactoring

### Creating a Feature Branch

```bash
# Start from develop
git checkout develop
git pull origin develop

# Create feature branch
git checkout -b feature/add-export-functionality

# Make changes, commit, push
git add .
git commit -m "feat(datasets): add CSV export functionality"
git push origin feature/add-export-functionality

# Create pull request on GitHub
```

### Daily Development Workflow

```bash
# 1. Pull latest changes
git checkout develop
git pull origin develop

# 2. Create/switch to your feature branch
git checkout -b feature/your-feature-name

# 3. Make changes and test
npm start  # Development server
npm test   # Run tests

# 4. Commit changes (see commit conventions below)
git add .
git commit -m "feat(module): description"

# 5. Push and create PR
git push origin feature/your-feature-name
```

---

## Architecture Principles

### Core Principles

1. **Signals-First**: Use Signals for state management, avoid RxJS unless necessary
2. **Standalone Components**: All components are standalone (no NgModules)
3. **OnPush Change Detection**: All components use `ChangeDetectionStrategy.OnPush`
4. **Route-Level State**: Feature stores scoped to routes via providers
5. **Accessibility First**: WCAG AAA compliance required for all features

### Architecture Constraints

**DO**:
- Use Signals for reactive state
- Use computed signals for derived values
- Use effects for side effects
- Use SignalStore for feature state
- Use route providers for scoped dependencies
- Use virtual scrolling for lists >100 items
- Use Web Workers for heavy computation (>50ms)

**DON'T**:
- Don't use RxJS `BehaviorSubject` or `Subject` for state (use signals)
- Don't use `@NgModule` (standalone components only)
- Don't use `async` pipe in templates (use signals with `()`)
- Don't use default change detection (always use OnPush)
- Don't render large lists without virtual scrolling
- Don't block main thread with heavy computation

### File Structure

```
src/
├── app/
│   ├── core/                    # Singleton services, guards, interceptors
│   │   ├── services/
│   │   ├── guards/
│   │   ├── interceptors/
│   │   └── workers/
│   ├── shared/                  # Shared components, directives, pipes
│   │   ├── components/
│   │   ├── directives/
│   │   ├── pipes/
│   │   └── models/
│   ├── features/                # Feature modules
│   │   ├── files/
│   │   │   ├── components/
│   │   │   ├── services/
│   │   │   ├── stores/
│   │   │   ├── models/
│   │   │   ├── files.routes.ts
│   │   │   └── index.ts
│   │   ├── design-system/
│   │   ├── media/
│   │   └── ...
│   └── workspace/               # Workspace shell components
│       ├── components/
│       └── stores/
```

---

## Code Style Standards

### TypeScript Style

```typescript
// GOOD: Descriptive names, type safety
interface FileNode {
  id: string;
  name: string;
  type: 'file' | 'folder';
  size: number;
  createdAt: Date;
}

const selectedFiles = computed(() => {
  return this.files().filter(f => this.selectedIds().includes(f.id));
});

// BAD: Generic names, any types
interface Node {
  id: any;
  data: any;
}

const x = computed(() => {
  return this.y().filter(z => this.z().includes(z));
});
```

### Component Style

```typescript
// GOOD: Standalone, OnPush, Signal inputs
@Component({
  selector: 'app-file-card',
  standalone: true,
  changeDetection: ChangeDetectionStrategy.OnPush,
  imports: [MatIconModule, MatButtonModule],
  template: `
    <div class="file-card">
      <mat-icon>{{ file().icon }}</mat-icon>
      <span>{{ file().name }}</span>
    </div>
  `,
})
export class FileCardComponent {
  file = input.required<FileNode>();
  selected = input<boolean>(false);

  fileClick = output<FileNode>();

  onClick() {
    this.fileClick.emit(this.file());
  }
}

// BAD: Module-based, default change detection, @Input
@Component({
  selector: 'app-file-card',
  template: `<div>{{ file.name }}</div>`,
})
export class FileCardComponent {
  @Input() file: any;
  @Output() click = new EventEmitter();
}
```

### Template Style

```html
<!-- GOOD: Control flow syntax, accessibility -->
<div class="file-list" role="list">
  @if (isLoading()) {
    <app-spinner />
  } @else if (files().length === 0) {
    <app-empty-state message="No files found" />
  } @else {
    @for (file of files(); track file.id) {
      <app-file-card
        [file]="file"
        [selected]="isSelected(file.id)"
        (fileClick)="onFileClick($event)"
        role="listitem"
      />
    }
  }
</div>

<!-- BAD: Old syntax, no accessibility -->
<div class="file-list">
  <app-spinner *ngIf="isLoading"></app-spinner>
  <div *ngFor="let file of files">
    <app-file-card [file]="file"></app-file-card>
  </div>
</div>
```

### CSS/SCSS Style

```scss
// GOOD: BEM naming, CSS custom properties, responsive
.file-card {
  padding: var(--spacing-md);
  background: var(--color-surface);
  border-radius: var(--border-radius-md);

  &__icon {
    width: 24px;
    height: 24px;
    color: var(--color-icon);
  }

  &__name {
    font-size: var(--font-size-base);
    font-weight: var(--font-weight-medium);
  }

  &--selected {
    background: var(--color-primary-light);
    border: 2px solid var(--color-primary);
  }

  @media (max-width: 768px) {
    padding: var(--spacing-sm);
  }
}

// BAD: Generic names, hard-coded values
.card {
  padding: 16px;
  background: #f5f5f5;
}

.name {
  font-size: 14px;
}
```

### Naming Conventions

| Type | Convention | Example |
|------|------------|---------|
| Component | PascalCase + Component suffix | `FileCardComponent` |
| Service | PascalCase + Service suffix | `FileManagerService` |
| Store | PascalCase + Store suffix | `FileManagerStore` |
| Interface | PascalCase | `FileNode`, `DataColumn` |
| Signal | camelCase | `selectedFiles`, `isLoading` |
| Function | camelCase, verb prefix | `getFiles`, `createFolder` |
| Constant | UPPER_SNAKE_CASE | `MAX_FILE_SIZE`, `API_BASE_URL` |
| CSS Class | kebab-case (BEM) | `file-card`, `file-card__icon` |

---

## Commit Conventions

### Commit Message Format

We follow **Conventional Commits** specification:

```
<type>(<scope>): <subject>

<body>

<footer>
```

**Example**:
```
feat(datasets): add CSV export functionality

Implement CSV export for datasets with column selection and formatting options.
Uses Web Worker for large datasets to prevent UI blocking.

Closes #123
```

### Commit Types

| Type | Description | Example |
|------|-------------|---------|
| `feat` | New feature | `feat(files): add drag-drop upload` |
| `fix` | Bug fix | `fix(kanban): resolve card positioning issue` |
| `docs` | Documentation | `docs(readme): update installation steps` |
| `style` | Code style (formatting, no logic change) | `style(files): format with prettier` |
| `refactor` | Code refactoring | `refactor(store): extract common state logic` |
| `perf` | Performance improvement | `perf(grid): optimize virtual scrolling` |
| `test` | Add/update tests | `test(files): add upload component tests` |
| `chore` | Build/tooling changes | `chore(deps): update angular to v19.1` |
| `ci` | CI/CD changes | `ci(github): add accessibility checks` |
| `revert` | Revert previous commit | `revert: feat(files): add drag-drop` |

### Commit Scope

Use the module or feature name:
- `files`, `design-system`, `media`, `kanban`, `logs`, `knowledge-base`
- `admin`, `cases`, `datasets`, `forms`
- `workspace`, `core`, `shared`

### Commit Guidelines

**DO**:
- Write clear, concise subject lines (<50 chars)
- Use imperative mood ("add feature", not "added feature")
- Provide context in body for complex changes
- Reference issue numbers in footer

**DON'T**:
- Don't use generic messages ("fix bug", "update code")
- Don't include multiple unrelated changes in one commit
- Don't commit broken/untested code

---

## Pull Request Process

### Before Creating a PR

- [ ] Code follows style standards
- [ ] All tests pass (`npm test`)
- [ ] New tests added for new features
- [ ] Accessibility checks pass (no axe violations)
- [ ] Documentation updated if needed
- [ ] No console.log or debugger statements
- [ ] TypeScript compilation succeeds with no errors

### PR Title Format

Same as commit format:
```
feat(datasets): add CSV export functionality
```

### PR Description Template

```markdown
## Summary
Brief description of what this PR does.

## Changes
- List of specific changes
- Another change
- One more change

## Testing
How was this tested?
- [ ] Unit tests
- [ ] Integration tests
- [ ] E2E tests
- [ ] Manual testing

## Screenshots (if applicable)
[Add screenshots here]

## Accessibility
- [ ] Keyboard navigation works
- [ ] Screen reader tested
- [ ] No axe violations
- [ ] Color contrast meets WCAG AAA

## Checklist
- [ ] Tests pass
- [ ] Documentation updated
- [ ] No breaking changes (or documented)
- [ ] Reviewed my own code
```

### PR Review Process

1. **Automated Checks**: CI runs tests, linting, accessibility checks
2. **Code Review**: At least one approval required
3. **Address Feedback**: Make requested changes
4. **Final Approval**: Merge when approved
5. **Squash and Merge**: Use squash merge to keep clean history

### Review Guidelines

**As a Reviewer**:
- Be constructive and respectful
- Explain the "why" behind suggestions
- Approve if changes are acceptable (don't block on nitpicks)
- Test the changes locally if significant

**As an Author**:
- Respond to all comments
- Ask questions if feedback is unclear
- Don't take feedback personally
- Update the PR based on feedback

---

## Testing Requirements

### Test Coverage Requirements

- **Unit Tests**: >80% coverage for all new code
- **Component Tests**: All new components must have tests
- **Integration Tests**: Required for feature modules
- **E2E Tests**: Required for new user flows

### Writing Tests

```typescript
// GOOD: Descriptive test names, arrange-act-assert
describe('FileManagerStore', () => {
  let store: InstanceType<typeof FileManagerStore>;

  beforeEach(() => {
    TestBed.configureTestingModule({
      providers: [FileManagerStore],
    });
    store = TestBed.inject(FileManagerStore);
  });

  it('should add files to state when setFiles is called', () => {
    // Arrange
    const files = [
      { id: '1', name: 'file1.txt' } as FileNode,
      { id: '2', name: 'file2.txt' } as FileNode,
    ];

    // Act
    store.setFiles(files);

    // Assert
    expect(store.files()).toEqual(files);
    expect(store.files().length).toBe(2);
  });
});

// BAD: Generic test names, unclear assertions
it('should work', () => {
  store.setFiles([{ id: '1' }]);
  expect(store.files().length).toBe(1);
});
```

### Running Tests

```bash
# Run all tests
npm test

# Run tests in watch mode
npm run test:watch

# Run tests with coverage
npm run test:coverage

# Run E2E tests
npm run e2e

# Run accessibility tests
npm run test:a11y
```

### Test Organization

```
src/app/features/files/
├── components/
│   ├── file-card/
│   │   ├── file-card.component.ts
│   │   ├── file-card.component.spec.ts  # Component test
│   │   └── file-card.component.scss
├── services/
│   ├── file.service.ts
│   └── file.service.spec.ts              # Service test
├── stores/
│   ├── file-manager.store.ts
│   └── file-manager.store.spec.ts        # Store test
└── file-manager.integration.spec.ts      # Integration test
```

---

## Documentation Requirements

### Code Documentation

```typescript
// GOOD: JSDoc for complex functions
/**
 * Flattens a tree structure into a flat array for virtual scrolling.
 *
 * @param nodes - The root nodes to flatten
 * @param expandedIds - Set of expanded folder IDs
 * @returns Flat array of nodes with level information
 *
 * @example
 * const flat = flattenTree(rootNodes, new Set(['folder-1']));
 */
function flattenTree(nodes: FileNode[], expandedIds: Set<string>): FlatNode[] {
  // Implementation
}

// GOOD: Inline comments for complex logic
const filteredFiles = computed(() => {
  const query = this.searchQuery().toLowerCase();

  // Apply text search
  let results = this.files().filter(f =>
    f.name.toLowerCase().includes(query)
  );

  // Apply MIME type filter if selected
  const mimeFilter = this.selectedMimeType();
  if (mimeFilter) {
    results = results.filter(f => f.mimeType === mimeFilter);
  }

  return results;
});
```

### README Updates

When adding a new feature:
1. Update feature list in main README.md
2. Add keyboard shortcuts to shortcuts table
3. Update documentation index with new docs

### Architecture Documentation

When making architectural changes:
1. Update ARCHITECTURE.md with rationale
2. Document patterns in relevant module design docs
3. Add examples to guide other developers

---

## Module Development Guide

### Creating a New Module

```bash
# 1. Create module directory
mkdir -p src/app/features/my-module/{components,services,stores,models}

# 2. Create route file
touch src/app/features/my-module/my-module.routes.ts

# 3. Create store
touch src/app/features/my-module/stores/my-module.store.ts

# 4. Create main shell component
ng generate component features/my-module/components/my-module-shell --standalone
```

### Module Checklist

When creating a new module, ensure:

- [ ] **Route Configuration**: Routes defined in `my-module.routes.ts`
- [ ] **Store**: SignalStore created with state, computed, methods
- [ ] **Components**: Shell component + feature components
- [ ] **Services**: API service for backend communication
- [ ] **Models**: TypeScript interfaces for data models
- [ ] **Accessibility**: ARIA patterns implemented, keyboard navigation
- [ ] **Virtual Scrolling**: If list >100 items
- [ ] **Tests**: Unit, component, integration tests
- [ ] **Documentation**: Module design doc in MODULE_DESIGNS.md

### Module Template

```typescript
// my-module.routes.ts
import { Routes } from '@angular/router';

export const MY_MODULE_ROUTES: Routes = [
  {
    path: '',
    loadComponent: () => import('./components/my-module-shell/my-module-shell.component')
      .then(m => m.MyModuleShellComponent),
    providers: [
      provideMyModuleStore(),
    ],
  },
];

// stores/my-module.store.ts
import { signalStore, withState, withComputed, withMethods } from '@ngrx/signals';

interface MyModuleState {
  items: MyItem[];
  selectedIds: string[];
  isLoading: boolean;
  error: string | null;
}

const initialState: MyModuleState = {
  items: [],
  selectedIds: [],
  isLoading: false,
  error: null,
};

export const MyModuleStore = signalStore(
  withState(initialState),

  withComputed((store) => ({
    selectedItems: computed(() => {
      return store.items().filter(item =>
        store.selectedIds().includes(item.id)
      );
    }),
  })),

  withMethods((store, myService = inject(MyService)) => ({
    async loadItems() {
      patchState(store, { isLoading: true, error: null });

      try {
        const items = await myService.getItems();
        patchState(store, { items, isLoading: false });
      } catch (error) {
        patchState(store, {
          error: 'Failed to load items',
          isLoading: false
        });
      }
    },

    selectItem(id: string) {
      const selectedIds = [...store.selectedIds(), id];
      patchState(store, { selectedIds });
    },
  })),
);

export function provideMyModuleStore() {
  return [MyModuleStore];
}
```

---

## Advanced Topics

### Performance Optimization

When adding features that handle large datasets:

1. **Use Virtual Scrolling**: For lists >100 items
2. **Use Web Workers**: For computation >50ms
3. **Optimize Change Detection**: Use OnPush and signals
4. **Lazy Load**: Heavy components with `@defer`
5. **Profile**: Use Chrome DevTools to measure performance

### State Management Patterns

```typescript
// GOOD: Optimistic updates with rollback
async moveFile(fileId: string, newParentId: string) {
  // Save current state
  const previousFiles = store.files();

  // Optimistic update
  patchState(store, {
    files: updateFileParent(previousFiles, fileId, newParentId),
  });

  try {
    await fileService.moveFile(fileId, newParentId);
  } catch (error) {
    // Rollback on error
    patchState(store, { files: previousFiles, error: 'Move failed' });
  }
}

// GOOD: Loading states for async operations
async loadData() {
  patchState(store, { isLoading: true, error: null });

  try {
    const data = await service.getData();
    patchState(store, { data, isLoading: false });
  } catch (error) {
    patchState(store, { error: error.message, isLoading: false });
  }
}
```

### Accessibility Patterns

Always include:
- ARIA roles and attributes
- Keyboard navigation
- Focus management
- Screen reader announcements
- Color contrast compliance

See [ACCESSIBILITY.md](./docs/ACCESSIBILITY.md) for complete patterns.

---

## Getting Help

### Resources

- **Documentation**: See `/docs` folder for comprehensive guides
- **Architecture**: [ARCHITECTURE.md](./docs/ARCHITECTURE.md)
- **State Management**: [STATE_MANAGEMENT.md](./docs/STATE_MANAGEMENT.md)
- **Testing**: [TESTING.md](./docs/TESTING.md)
- **Accessibility**: [ACCESSIBILITY.md](./docs/ACCESSIBILITY.md)

### Questions

- **GitHub Discussions**: Ask questions and share ideas
- **GitHub Issues**: Report bugs or request features
- **Code Review**: Tag maintainers in PRs for guidance

### Maintainers

Core maintainers who can help with technical questions:
- @maintainer1 - Architecture, State Management
- @maintainer2 - UI/UX, Accessibility
- @maintainer3 - Testing, Performance

---

## License

By contributing to Nexus Studio, you agree that your contributions will be licensed under the same license as the project.

---

## Summary

**Key Principles**:
1. **Quality First**: Write clean, tested, accessible code
2. **Collaboration**: Communicate clearly, review thoughtfully
3. **Learning**: Ask questions, share knowledge, grow together
4. **Standards**: Follow conventions, maintain consistency
5. **User Focus**: Build features that delight users

**Thank you for contributing to Nexus Studio!** 🎉
