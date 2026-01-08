# Claude Workflow - AI Agent Configuration Repository

> **Comprehensive architectural standards, best practices, and AI agent configurations for building high-quality backend and frontend applications.**

This repository provides Claude Code (and other AI agents) with structured guidance, reusable skills, specialized agents, and pre-configured commands to ensure consistent, secure, and maintainable code across all your projects.

---

## 📚 Repository Purpose

This is a **documentation and configuration repository** designed to:

- **Guide AI agents** with battle-tested architectural patterns and coding standards
- **Enforce best practices** through reusable skills and methodologies
- **Accelerate development** with specialized agents for common tasks
- **Ensure consistency** across teams and projects
- **Prevent technical debt** through mandatory analysis workflows

---

## 🗂️ Repository Structure

```
claude-workflow/
├── agents/                    # Specialized AI agents for specific roles
│   ├── api-designer.md        # REST API design specialist
│   ├── tdd-assistant.md       # Test-Driven Development coach
│   └── research-assistant.md  # Research and documentation specialist
│
├── commands/                  # Pre-configured workflows and commands
│   ├── feature-analysis.md    # Comprehensive feature analysis before coding
│   └── bugfix.md             # Structured bug fix workflow with TDD
│
├── skills/                    # Reusable technical knowledge and methodologies
│   ├── analysis/             # Mandatory pre-implementation analysis process
│   ├── api-design/           # REST API design principles (OpenAPI, versioning)
│   ├── architecture/         # Software architecture patterns
│   ├── database/             # Database design (PostgreSQL, optimization)
│   ├── security/             # Security best practices (OWASP Top 10, auth)
│   ├── solid-principles/     # SOLID principles with examples
│   └── testing/              # Testing strategies (unit, integration, E2E, TDD)
│
└── templates/                 # Project-specific templates
    ├── react/                # React/TypeScript configuration
    └── nextjs/               # Next.js configuration
```

---

## 🤖 Agents

Specialized AI agents that take on specific expert roles:

### 1. **API Designer** (`agents/api-designer.md`)

Senior API architect with expertise in:

- OpenAPI 3.1 specification
- REST API design principles
- Authentication patterns (JWT, OAuth 2.0)
- Versioning strategies
- Documentation automation

**Use when**: Designing new APIs or refactoring existing endpoints

---

### 2. **TDD Assistant** (`agents/tdd-assistant.md`)

Test-Driven Development coach with 15+ years experience:

- Enforces Red-Green-Refactor cycle
- Guides bug fixes with failing tests first
- Ensures proper test coverage (>80%)
- Promotes clean, testable code

**Use when**: Implementing features or fixing bugs following TDD methodology

---

### 3. **Research Assistant** (`agents/research-assistant.md`)

Technical researcher specialized in:

- Technology evaluation and comparison
- Documentation synthesis
- Best practice research
- Trend analysis

**Use when**: Evaluating new technologies or investigating solutions

---

## 🛠️ Commands

Pre-configured workflows for common development tasks:

### 1. **Feature Analysis** (`commands/feature-analysis.md`)

Generates comprehensive analysis reports **before** writing any code.

**Usage**:

```bash
# Invoke the feature analysis command
analyze-feature "User authentication with JWT tokens and role-based access control"
```

**Output**: Generates `docs/features/user-authentication-jwt-rbac.md` with:

- User story and acceptance criteria
- Technical specification (components, data models, API contracts)
- Architecture and design patterns
- Security considerations (OWASP Top 10)
- Performance optimization strategy
- TDD test plan (unit, integration, E2E)
- Risk assessment and mitigation
- Implementation roadmap

**Benefits**:

- Prevents costly rework by catching issues early
- Ensures shared understanding across teams
- Documents architectural decisions
- Creates actionable TDD test plans

---

### 2. **Bug Fix Workflow** (`commands/bugfix.md`)

Structured approach to fixing bugs following TDD principles.

**Process**:

1. **Reproduce**: Understand and document the bug
2. **RED**: Write failing test that reproduces the bug
3. **GREEN**: Fix with minimal code
4. **REFACTOR**: Improve code quality
5. **VERIFY**: Ensure no regressions

---

## 📖 Skills

Reusable technical knowledge modules that agents can reference:

### Core Skills

| Skill                | Description                                   | Key Topics                                            |
| -------------------- | --------------------------------------------- | ----------------------------------------------------- |
| **Analysis**         | Mandatory pre-implementation analysis process | 4-step framework, impact analysis, risk assessment    |
| **API Design**       | REST API best practices                       | Resource naming, HTTP methods, pagination, versioning |
| **Architecture**     | Software architecture patterns                | Hexagonal, Clean Architecture, Layered, MVC           |
| **Testing**          | Comprehensive testing strategies              | TDD, unit tests, integration tests, E2E, mocking      |
| **Security**         | Security best practices                       | OWASP Top 10, authentication, input validation        |
| **SOLID Principles** | Object-oriented design principles             | Single Responsibility, Open/Closed, Liskov, etc.      |

### Database Skills

| Skill          | Description                              | Coverage                                               |
| -------------- | ---------------------------------------- | ------------------------------------------------------ |
| **PostgreSQL** | PostgreSQL table design and optimization | Data types, indexing, partitioning, performance tuning |

---

## 🏗️ Architecture Principles

This repository enforces:

1. **Analysis-First Approach**: Never code without thorough analysis
2. **Test-Driven Development**: Red-Green-Refactor cycle
3. **Security by Design**: OWASP Top 10 considered from day one
4. **SOLID Principles**: Maintainable, extensible code
5. **Clean Architecture**: Clear separation of concerns
6. **Documentation as Code**: Living documentation that evolves with code

## 🔧 Technology Coverage

### Backend

- **Languages**: Node.js/TypeScript
- **Databases**: PostgreSQL (primary), MongoDB (In Progress)
- **APIs**: REST, GraphQL, OpenAPI 3.1
- **Authentication**: JWT, OAuth 2.0, API Keys

### Frontend

- **Frameworks**: React, Next.js
- **State Management**: Context API, Zustand
- **Testing**: Jest, Vitest, React Testing Library, Cypress, Playwright

---

## 📝 Documentation Standards

All documentation must be:

- **Clear**: No ambiguity in requirements or specifications
- **Actionable**: Developers can implement directly from docs
- **Testable**: Success criteria are measurable
- **Maintainable**: Easy to update as code evolves
- **Searchable**: Well-organized with clear naming

## 🎓 Philosophy

> **"Understand deeply before building"**

This repository embodies the principle that **time spent in analysis is never wasted**. By thoroughly understanding requirements, risks, and architecture before writing code, we:

- Reduce rework and technical debt
- Catch security issues early
- Build maintainable, testable systems
- Align teams around shared understanding
- Deliver higher quality software faster

---

## 🙏 Acknowledgments

This repository is built on battle-tested practices from:

- Robert C. Martin (Clean Code, SOLID principles)
- Martin Fowler (Refactoring, Architecture patterns)
- OWASP Foundation (Security best practices)
- The TDD and BDD communities
- The rules, agents, and commands from (claude-craft repository)[https://github.com/TheBeardedBearSAS/claude-craft/tree/main] and (claude code plugins)[https://github.com/wshobson/agents/tree/main]

---
