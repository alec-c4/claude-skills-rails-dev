# Installation Guide

Quick installation for Claude Code, Claude Desktop, and Claude Web.

## For Claude Code (Recommended)

### Global Installation (All Projects)

```bash
# Clone to global skills directory
git clone https://github.com/alec-c4/claude-skills-rails-dev.git ~/.claude/skills/rails-dev

# Or for specific version
git clone --branch v1.0.0 https://github.com/alec-c4/claude-skills-rails-dev.git ~/.claude/skills/rails-dev
```

Skills will be available in all Claude Code sessions.

### Project-Specific Installation

```bash
# Navigate to your Rails project
cd /path/to/your-rails-project

# Clone to project skills directory
git clone https://github.com/alec-c4/claude-skills-rails-dev.git .claude/skills/rails-dev
```

Skills will only be available in this project.

### Update Skills

```bash
# Global
cd ~/.claude/skills/rails-dev
git pull origin master

# Project-specific
cd /path/to/your-project/.claude/skills/rails-dev
git pull origin master
```

## For Claude Desktop

### macOS/Linux

```bash
# Create skills directory
mkdir -p ~/.claude/skills

# Clone repository
git clone https://github.com/alec-c4/claude-skills-rails-dev.git ~/.claude/skills/rails-dev
```

### Windows

```powershell
# Create skills directory
New-Item -Path "$env:USERPROFILE\.claude\skills" -ItemType Directory -Force

# Clone repository
git clone https://github.com/alec-c4/claude-skills-rails-dev.git "$env:USERPROFILE\.claude\skills\rails-dev"
```

## For Claude Web

Claude Web doesn't support custom skills directly. Use one of these approaches:

### Option 1: Manual Context (Projects Feature)

1. Create a new project in Claude Web
2. Add key skill files as project knowledge:
   - `ruby-on-rails-development/SKILL.md` (router)
   - Skills you use most (e.g., `rails-testing/SKILL.md`)

### Option 2: Copy-Paste Context

Copy the content of needed skill files into your conversation:

```
I'm working on a Rails project. Please use these guidelines:

[Paste content of rails-testing/SKILL.md]
```

## Verification

After installation, verify skills are available:

### Claude Code

```bash
# In any Rails project
claude-code
```

Then ask: "What Rails development skills are available?"

### Claude Desktop

Start a new conversation and ask: "What Rails development skills are available?"

## Available Skills

Once installed, you have access to:

- **ruby-on-rails-development** - Router and core principles
- **rails-project-manager** - Multi-skill coordination
- **rails-testing** - RSpec, FactoryBot, TDD
- **rails-viewcomponents** - ViewComponent, Turbo, Stimulus
- **rails-business-logic** - ActiveInteraction, AASM
- **rails-api** - REST APIs with versioning
- **rails-graphql** - GraphQL implementation
- **rails-inertia** - Inertia.js SPAs
- **rails-background-jobs** - Solid Queue jobs
- **rails-devops** - Docker, Kamal, deployment
- **rails-security** - Pundit, Lockbox, auth
- **rails-analyst** - JTBD, estimation, risk analysis
- **rails-technical-writer** - Documentation

## Usage

Just describe what you need - Claude will automatically activate the appropriate skill:

```
"Write tests for User model with email validation"
→ Activates rails-testing

"Create a ButtonComponent with variants"
→ Activates rails-viewcomponents

"Add JWT authentication to my API"
→ Activates rails-api

"Help me plan a notification system"
→ Activates rails-project-manager
```

No manual skill selection needed!

## Troubleshooting

### Skills not loading

**Claude Code:**
```bash
# Check skills directory exists
ls -la ~/.claude/skills/rails-dev

# Verify SKILL.md files exist
ls -la ~/.claude/skills/rails-dev/*/SKILL.md
```

**Claude Desktop:**
- Restart Claude Desktop
- Check `~/.claude/skills` directory exists
- Verify repository is cloned correctly

### Wrong skill activates

The router (`ruby-on-rails-development/SKILL.md`) analyzes your request to select the best skill. If wrong skill activates, be more specific:

Instead of: "I need to test something"
Try: "Write RSpec tests for User model"

## Support

- **Documentation**: [README.md](README.md)
- **Skill Registry**: [SKILLS_REGISTRY.md](SKILLS_REGISTRY.md)
- **Issues**: [GitHub Issues](https://github.com/alec-c4/claude-skills-rails-dev/issues)

---

**Version**: 1.0.0
**Last Updated**: 2025-01-21
