# CLAUDE.md

This file provides guidance to Claude Code when working with code in this repository.

## Repository Overview

This is a documentation repository containing architectural standards and best practices for AI agents building backend and frontend services.

## Working with This Repository

Within this repository, Claude acts as a **documentation architect**. Since this is a documentation repository, there are no build, test, or lint commands. When making contributions:

1. **Editing Standards**: Focus on clarity, practicality, and consistency with existing skills, commands and agents
2. **Rule Format**: Each skill follows the pattern "Skill N: [Title]" with clear explanations and examples

## Architecture Principles

The documentation is organized by **technical domain**, with each domain owning its own architectural principles, skills, and Claude configuration.

This repository provides:

- **`agents/`** - Specialized AI agents for specific roles (API Designer, TDD Assistant, Research Assistant)
- **`commands/`** - Pre-configured workflows for common tasks (Feature Analysis, Bug Fix)
- **`skills/`** - Reusable technical knowledge modules (Analysis, Testing, Security, Database, etc.)
- **`templates/`** - Project-specific templates for React, Next.js, and other frameworks

## Contributing

When adding or modifying rules:

- Ensure rules are practical and battle-tested
- Provide clear examples where helpful
- Maintain consistency with existing rule numbering and format
- Consider how AI agents will interpret and apply the rules
