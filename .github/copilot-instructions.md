# Project Guidelines

<!-- 
  This is a template for copilot-instructions.md.
  Replace the placeholders below with your project-specific conventions.
  This file tells AI agents how to work within your codebase.
-->

## Project Context

### Objective
- <!-- Describe what this project does and the goal of the current initiative -->
- <!-- e.g., "Build a new payment processing service" or "Modernize legacy billing system to cloud-native" -->

### GitHub Copilot Memory Usage
- This project leverages GitHub Copilot Memory to improve repository-aware analysis and recommendations.
- Use repository conventions and prior validated memories to guide analysis.
- Keep outputs consistent with current code and project templates.

## Code Style

### Conventions
- <!-- Primary language and framework (e.g., "TypeScript with React", "Java 21 with Spring Boot", "Python with FastAPI") -->
- <!-- Naming conventions (e.g., "Use camelCase for variables, PascalCase for classes") -->
- Follow existing patterns in nearby files instead of reformatting.
- Keep changes minimal and scoped; avoid broad whitespace-only or formatting-only edits.

### Architecture
- <!-- Describe your project structure (e.g., "src/ for source, tests/ for tests, docs/ for documentation") -->
- <!-- List key modules or areas of the codebase -->

### Build and Test
- <!-- Build command (e.g., "npm run build", "mvn clean install", "cargo build") -->
- <!-- Test command (e.g., "npm test", "mvn test", "pytest") -->
- <!-- Lint command (e.g., "npm run lint", "eslint .", "flake8") -->
- Prefer documenting verification steps performed rather than claiming tests that were not run.

### Pitfalls
- <!-- Known gotchas or constraints specific to your project -->
- <!-- e.g., "Do not modify the migration order in db/migrations/" -->
- <!-- e.g., "Batch jobs are order-sensitive; avoid changing job order" -->

## Conventions
- <!-- Reference key documentation files (e.g., "Treat README.md as the source of truth for...") -->
- <!-- Licensing requirements (e.g., "Keep Apache-2.0 licensing headers intact") -->

## Design Principles

All code, workflows, actions, and artifacts in this repository **must** follow these principles:

### KISS — Keep It Simple, Stupid
- Prefer the simplest solution that meets the requirement.
- Avoid unnecessary abstractions, indirect calls, or overly generic patterns when a direct approach works.

### YAGNI — You Aren't Gonna Need It
- Do not add features, inputs, or parameters for hypothetical future use cases.
- Implement only what the current task requires.

### DRY — Don't Repeat Yourself
- Extract shared logic into reusable components instead of duplicating.
- Centralize configuration in `squad-config.json`.

### Agent Assignment Guide
- **Orchestrator**: Use when the task requires more than one agent, involves 3+ changes, or needs phased coordination.
- **Planner**: Use when 2+ files are affected, cross-component impacts exist, or there are open questions before implementation.
- **Coder**: Use for single-scope changes with clear requirements and known patterns to follow.
- **Reviewer**: Use after Coder completes work and before merging — reviews for quality, security, and convention adherence.
