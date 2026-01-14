# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is an Nx monorepo for TypeScript libraries. Packages are located in `packages/` and can be published independently.

## Common Commands

```bash
# Build a specific package
nx build <package-name>

# Run all tasks across affected projects
nx affected -t build test lint typecheck

# Run multiple tasks on all projects
nx run-many -t lint test build typecheck

# Typecheck a package
nx typecheck <package-name>

# Format code
nx format:write

# Generate a new publishable library
nx g @nx/js:lib packages/<name> --publishable --importPath=@my-org/<name>

# Visualize project dependencies
nx graph

# Sync TypeScript project references
nx sync

# Release packages
nx release
```

## Architecture

- **Monorepo structure**: Uses npm workspaces with Nx for orchestration
- **Package location**: All packages live in `packages/`
- **Build system**: Uses `@nx/js/typescript` plugin with SWC for compilation
- **TypeScript**: Strict mode enabled, targets ES2022, uses NodeNext module resolution

## CI Pipeline

CI runs on GitHub Actions and executes: `format:check`, `lint`, `test`, `build`, `typecheck`, and `e2e-ci` targets using Nx Cloud for distributed task execution.

<!-- nx configuration start-->
<!-- Leave the start & end comments to automatically receive updates. -->

## Nx Guidelines

- Always run tasks through `nx` (e.g., `nx run`, `nx run-many`, `nx affected`) instead of underlying tooling
- Use `nx_workspace` MCP tool to understand workspace architecture
- Use `nx_project_details` MCP tool for specific project structure and dependencies
- Use `nx_docs` MCP tool for Nx configuration questions and best practices
- Use `nx_workspace` MCP tool to diagnose project graph errors

<!-- nx configuration end-->
