# Compounding Engineering Plugin

AI-powered development tools that get smarter with every use. Make each unit of engineering work easier than the last.

## Components

| Component | Count |
|-----------|-------|
| Agents | 27 |
| Commands | 16 |
| Skills | 16 |
| MCP Servers | 2 |

## Agents

Agents are organised into categories for easier discovery.

### Review (11)

| Agent | Description |
|-------|-------------|
| `architecture-strategist` | Analyse architectural decisions and compliance |
| `chris-mccord-phoenix-reviewer` | Phoenix review from Chris McCord's perspective |
| `code-simplicity-reviewer` | Final pass for simplicity and minimalism |
| `data-integrity-guardian` | Database migrations and data integrity |
| `ecto-data-guardian` | Ecto-specific data integrity and migration safety |
| `elixir-reviewer` | Elixir code review with strict conventions |
| `kieran-code-reviewer` | Language-agnostic code review with strict conventions |
| `pattern-recognition-specialist` | Analyse code for patterns and anti-patterns |
| `performance-oracle` | Performance analysis and optimisation |
| `phoenix-reviewer` | Phoenix framework code review |
| `security-sentinel` | Security audits and vulnerability assessments |

### Research (5)

| Agent | Description |
|-------|-------------|
| `best-practices-researcher` | Gather external best practices and examples |
| `framework-docs-researcher` | Research framework documentation and best practices |
| `git-history-analyzer` | Analyse git history and code evolution |
| `phoenix-best-practices-researcher` | Research Phoenix-specific best practices |
| `repo-research-analyst` | Research repository structure and conventions |

### Design (3)

| Agent | Description |
|-------|-------------|
| `design-implementation-reviewer` | Verify UI implementations match Figma designs |
| `design-iterator` | Iteratively refine UI through systematic design iterations |
| `figma-design-sync` | Synchronise web implementations with Figma designs |

### Workflow (6)

| Agent | Description |
|-------|-------------|
| `bug-reproduction-validator` | Systematically reproduce and validate bug reports |
| `elixir-lint` | Run linting and code quality checks on Elixir and Phoenix files |
| `every-style-editor` | Edit content to conform to Every's style guide |
| `feedback-codifier` | Codify feedback patterns into reviewer agents |
| `pr-comment-resolver` | Address PR comments and implement fixes |
| `spec-flow-analyzer` | Analyse user flows and identify gaps in specifications |

### Docs (2)

| Agent | Description |
|-------|-------------|
| `concise-readme-writer` | Create concise READMEs for any library or package |
| `hex-package-readme-writer` | Create READMEs following Hex package conventions |

## Commands

### Workflow Commands

Access via `/workflows:command`:

| Command | Description |
|---------|-------------|
| `/workflows:plan` | Create implementation plans |
| `/workflows:review` | Run comprehensive code reviews |
| `/workflows:work` | Execute work items systematically |
| `/workflows:codify` | Document solved problems for knowledge base |

### Utility Commands

| Command | Description |
|---------|-------------|
| `/changelog` | Create engaging changelogs for recent merges |
| `/create-agent-skill` | Create or edit Claude Code skills |
| `/generate_command` | Generate new slash commands |
| `/heal-skill` | Fix skill documentation issues |
| `/plan_review` | Multi-agent plan review in parallel |
| `/prime` | Prime/setup command |
| `/report-bug` | Report a bug in the compounding-engineering plugin |
| `/reproduce-bug` | Reproduce bugs using logs and console |
| `/resolve_parallel` | Resolve TODO comments in parallel |
| `/resolve_pr_parallel` | Resolve PR comments in parallel |
| `/resolve_todo_parallel` | Resolve todos in parallel |
| `/triage` | Triage and prioritise issues |

## Skills

### Development Tools

| Skill | Description |
|-------|-------------|
| `andrew-kane-gem-writer` | Write Ruby gems following Andrew Kane's patterns |
| `codify-docs` | Capture solved problems as categorised documentation |
| `create-agent-skills` | Expert guidance for creating Claude Code skills |
| `dhh-ruby-style` | Write Ruby/Rails code in DHH's 37signals style |
| `dspy-ruby` | Build type-safe LLM applications with DSPy.rb |
| `frontend-design` | Create production-grade frontend interfaces |
| `hex-package-writer` | Write Hex packages following Elixir best practices |
| `skill-creator` | Guide for creating effective Claude Code skills |

### Content & Workflow

| Skill | Description |
|-------|-------------|
| `every-style-editor` | Review copy for Every's style guide compliance |
| `file-todos` | File-based todo tracking system |
| `git-worktree` | Manage Git worktrees for parallel development |

### Elixir & Phoenix

| Skill | Description |
|-------|-------------|
| `chris-mccord-phoenix-style` | Write Phoenix/LiveView code following Chris McCord's philosophy |
| `elixir-style` | Comprehensive Elixir coding standards from official guides |
| `instructor-elixir` | Structured LLM outputs using Ecto schemas with Instructor |
| `otp-patterns` | GenServer, Supervisor, Agent, Task, and OTP architecture patterns |

### Image Generation

| Skill | Description |
|-------|-------------|
| `gemini-imagegen` | Generate and edit images using Google's Gemini API |

**gemini-imagegen features:**
- Text-to-image generation
- Image editing and manipulation
- Multi-turn refinement
- Multiple reference image composition (up to 14 images)

**Requirements:**
- `GEMINI_API_KEY` environment variable
- Python packages: `google-genai`, `pillow`

## MCP Servers

| Server | Description |
|--------|-------------|
| `playwright` | Browser automation via `@playwright/mcp` |
| `context7` | Framework documentation lookup via Context7 |

### Playwright

**Tools provided:**
- `browser_navigate` - Navigate to URLs
- `browser_take_screenshot` - Take screenshots
- `browser_click` - Click elements
- `browser_fill_form` - Fill form fields
- `browser_snapshot` - Get accessibility snapshot
- `browser_evaluate` - Execute JavaScript

### Context7

**Tools provided:**
- `resolve-library-id` - Find library ID for a framework/package
- `get-library-docs` - Get documentation for a specific library

Supports 100+ frameworks including Rails, React, Next.js, Vue, Django, Laravel, and more.

MCP servers start automatically when the plugin is enabled.

## Installation

```bash
claude /plugin install compounding-engineering
```

## Known Issues

### MCP Servers Not Auto-Loading

**Issue:** The bundled MCP servers (Playwright and Context7) may not load automatically when the plugin is installed.

**Workaround:** Manually add them to your project's `.claude/settings.json`:

```json
{
  "mcpServers": {
    "playwright": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@playwright/mcp@latest"],
      "env": {}
    },
    "context7": {
      "type": "http",
      "url": "https://mcp.context7.com/mcp"
    }
  }
}
```

Or add them globally in `~/.claude/settings.json` for all projects.

## Version History

See [CHANGELOG.md](CHANGELOG.md) for detailed version history.

## Licence

MIT
