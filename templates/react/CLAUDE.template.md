# React TypeScript Configuration

## Project Information

**Project**: {{PROJECT_NAME}}
**Stack**: {{TECH_STACK}}
**Framework**: React {{REACT_VERSION}} + TypeScript {{TYPESCRIPT_VERSION}}
**Build Tool**: {{BUILD_TOOL}}

IMPORTANT: This is a base template to adapt according to project specifications. Every line must be carefully reviewed and verified before using Claude Code.

## Core Rules

### 1. Mandatory Analysis Before Coding

**BEFORE writing any code, you MUST**:

1. **Analyze existing code**

   - Search for similar components/utils/hooks
   - Identify patterns and the architecture in use

2. **Understand the requirement**

   - Clarify objectives
   - Define acceptance criteria
   - Identify constraints

3. **Design the solution**

   - Choose the architectural approach
   - Plan testing strategy

4. **Document decisions**
   - Justify technical choices
   - Document alternatives considered
   - Explain trade-offs

**Reference**: `skills/analysis/SKILL.md`

### 2. Feature-Based Architecture

This section must be updated according to the architecture followed by the {{PROJECT_NAME}} project.

```
src/
├── components/          # Reusable components (Atomic Design)
│   ├── atoms/           # Buttons, inputs, labels
│   ├── molecules/       # FormField, SearchBar, Card
│   ├── organisms/       # Header, DataTable, Form
│   └── templates/       # DashboardTemplate, AuthTemplate
│
├── features/            # Business features
│   └── [feature-name]/
│       ├── components/  # Feature-specific components
│       ├── hooks/       # Custom hooks
│       ├── services/    # API services
│       ├── types/       # TypeScript types
│       ├── utils/       # Utilities
│       └── store/       # Local state management
│
├── hooks/               # Global reusable hooks
├── services/            # Shared services (API, storage)
├── store/               # Global state management
├── types/               # Global types
├── utils/               # Global utilities
├── config/              # Configuration
└── lib/                 # Configured libraries
```

**Reference**: `skills/architecture/SKILL.md`

### 3. TypeScript Code Standards

#### TypeScript Strict Mode

```json
{
  "compilerOptions": {
    "strict": true,
    "noImplicitAny": true,
    "strictNullChecks": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noImplicitReturns": true
  }
}
```

#### Naming Conventions

- **Component Files**: `PascalCase.tsx` (e.g., `UserProfile.tsx`)
- **Hook Files**: `camelCase.ts` (e.g., `useAuth.ts`)
- **Utility Files**: `camelCase.ts` (e.g., `formatDate.ts`)
- **Type Files**: `camelCase.types.ts` (e.g., `user.types.ts`)
- **Service Files**: `camelCase.service.ts` (e.g., `user.service.ts`)

### 4. SOLID Principles

- **S**: Single Responsibility - One component = one responsibility
- **O**: Open/Closed - Open for extension, closed for modification
- **L**: Liskov Substitution - Child components respect the contract
- **I**: Interface Segregation - Specific props, no overloaded interfaces
- **D**: Dependency Inversion - Depend on abstractions, not implementations

**Reference**: `skills/solid-principles/SKILL.md`

## Technology Stack

### Frontend

- **Framework**: React {{REACT_VERSION}}
- **Language**: TypeScript {{TYPESCRIPT_VERSION}}
- **Build Tool**: {{BUILD_TOOL}}
- **Routing**: React Router / Next.js App Router
- **State Management**: TanStack Query / Zustand / React Context
- **Forms**: React Hook Form + Zod
- **Styling**: Tailwind CSS
- **UI Components**: Shadcn / MantineUI

### Development Tools

- **Package Manager**: pnpm (recommended) / npm
- **Linting**: ESLint + TypeScript ESLint
- **Formatting**: Prettier
- **Testing**: Vitest + React Testing Library + Playwright
- **Git Hooks**: Husky + lint-staged
- **Commit Convention**: Conventional Commits

### Mandatory Testing

1. **Unit Tests**: All components, hooks, and utilities
2. **Integration Tests**: Complete features
3. **E2E Tests**: Critical user journeys

**Reference**: `skills/testing/SKILL.md`

## Code Quality

### Required Tools

- [x] ESLint configured with strict rules
- [x] Prettier for formatting
- [x] TypeScript in strict mode
- [x] Husky for git hooks
- [x] Lint-staged for pre-commit
- [x] Commitlint for Conventional Commits

### Quality Metrics

- **TypeScript Errors**: 0
- **Test Coverage**: > 80%
- **Build Size**: < 500kb (initial)
- **Lighthouse Score**: > 90

## Git Workflow

### Branches

```
main (production)
  └── dev (integration)
      ├── feature/* (new features)
      ├── fix/* (bug fixes)
      ├── refactor/* (refactoring)
      └── hotfix/* (urgent fixes)
```

### Commit Convention

**Types**: feat, fix, hotfix, docs, refactor, test, build, ci, chore

**Examples**:

```bash
feat(auth): add OAuth2 authentication
fix(validation): resolve email regex issue
docs(readme): update installation instructions
refactor(hooks): extract common logic
test(auth): add login component tests
```

## Documentation

### Code Documentation

- **JSDoc** for complex components
- **Comments** to explain "why", not "what"

## Security

### Security Checklist

- [ ] Input validation
- [ ] HTML sanitization (DOMPurify)
- [ ] XSS protection
- [ ] CSRF protection
- [ ] HTTPS everywhere
- [ ] CSP headers configured
- [ ] Secrets in environment variables

### Critical Points

1. **NEVER** hardcode secrets
2. **ALWAYS** validate inputs on both client AND server
3. **ALWAYS** sanitize user-provided HTML
4. **ALWAYS** use HTTPS
5. **ALWAYS** use `rel="noopener noreferrer"` on external links

**Reference**: `rules/security/SKILL.md`

## Performance

### Required Optimizations

1. **Code Splitting**: Lazy loading for routes
2. **Memoization**: React.memo, useMemo, useCallback
3. **Virtual Lists**: For lists > 100 items
4. **Images**: Optimization and lazy loading
5. **Bundle**: Analysis and optimization

### Performance Metrics

- **First Contentful Paint**: < 1.8s
- **Time to Interactive**: < 3.8s
- **Largest Contentful Paint**: < 2.5s
- **Cumulative Layout Shift**: < 0.1
- **Total Blocking Time**: < 300ms

## State Management

### Strategy

- **React Query**: Server state (API data)
- **Zustand**: Client state (global UI state)
- **React Hook Form**: Form state
- **URL State**: Search params, filters
