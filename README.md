# Laravel Best Practices

A structured repository for creating and maintaining Laravel Best Practices optimized for agents and LLMs.

## Installation

### Option 1: Install this skill using [skills](https://github.com/vercel-labs/skills):

```bash
# GitHub shorthand
npx skills add PauloFelipeM/agent-laravel-skills

# Install globally (available across all projects)
npx skills add PauloFelipeM/agent-laravel-skills --global

# Install for specific agents
npx skills add PauloFelipeM/agent-laravel-skills -a claude-code -a cursor
```

#### Supported Agents

- Claude Code
- OpenCode
- Codex
- Cursor
- Antigravity
- Roo Code

### Option 2: Project-level (via MD file, using CLAUDE.md file as an example):
Download this repository and add a reference in your project's CLAUDE.md so Claude loads the rules for this project:

```bash
# Laravel Best Practices
Follow the Laravel best practices defined in .claude/skills/laravel-best-practices/AGENTS.md
```

# Laravel Best Practices
Follow the Laravel best practices defined in .claude/skills/laravel-best-practices/AGENTS.md

## Structure

- `rules/` - Individual rule files (one per rule)
  - `_sections.md` - Section metadata (titles, impacts, descriptions)
  - `_template.md` - Template for creating new rules
  - `area-description.md` - Individual rule files
- `scripts/` - Build scripts and utilities
- `metadata.json` - Document metadata (version, organization, abstract)
- __`AGENTS.md`__ - Compiled output (generated)

## Getting Started

1. Install dependencies:
   ```bash
   cd scripts && npm install
   ```

2. Build AGENTS.md from rules:
   ```bash
   npm run build
   # or
   ./scripts/build.sh
   ```

## How it works

When you're writing or reviewing Laravel code in your project, just ask naturally:

- "Review this controller"
- "Create a new service for payments"
- "Is this migration correct?"

The skill will be applied automatically when relevant.

## Creating a New Rule

1. Copy `rules/_template.md` to `rules/area-description.md`
2. Choose the appropriate area prefix:
   - `arch-` for Architecture (Section 1)
   - `di-` for Dependency Injection (Section 2)
   - `error-` for Error Handling (Section 3)
   - `security-` for Security (Section 4)
   - `perf-` for Performance (Section 5)
   - `test-` for Testing (Section 6)
   - `db-` for Database & ORM (Section 7)
   - `api-` for API Design (Section 8)
   - `micro-` for Microservices (Section 9)
   - `devops-` for DevOps & Deployment (Section 10)
3. Fill in the frontmatter and content
4. Ensure you have clear examples with explanations
5. Run the build script to regenerate AGENTS.md

## Rule File Structure

Each rule file should follow this structure:

```markdown
---
title: Rule Title Here
impact: MEDIUM
impactDescription: Optional description
tags: tag1, tag2, tag3
---

## Rule Title Here

Brief explanation of the rule and why it matters.

**Incorrect (description of what's wrong):**

\```php
// Bad code example
\```

**Correct (description of what's right):**

\```php
// Good code example
\```

Reference: [Laravel Documentation](https://laravel.com/docs)
```

## Impact Levels

| Level | Description |
|-------|-------------|
| CRITICAL | Violations cause runtime errors, security vulnerabilities, or architectural breakdown |
| HIGH | Significant impact on reliability, security, or maintainability |
| MEDIUM-HIGH | Notable impact on quality and developer experience |
| MEDIUM | Moderate impact on code quality and best practices |
| LOW-MEDIUM | Minor improvements for consistency and maintainability |

## Scripts

- `npm run build` (in scripts/) - Compile rules into AGENTS.md
