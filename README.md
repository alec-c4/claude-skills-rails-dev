# Rails Development Skills

> **Version:** 1.0.0
> **Status:** Stable
> **License:** MIT
> **Rails Support:** 7.0, 7.1, 7.2, 8.0, 8.1
> **Ruby Support:** 3.2, 3.3+

Professional Rails development skills for Claude Code. Modular, extensible, and community-driven.

---

## 🎯 What Is This?

A collection of **13 specialized AI skills** for Ruby on Rails development. Each skill focuses on a specific area (testing, security, API, etc.) and follows Rails best practices.

**Key Features:**
- ✅ **Automatic routing** - Claude selects the right specialist based on your request
- ✅ **TDD enforced** - All specialists follow test-first development
- ✅ **Modular** - Each skill is independent and versioned
- ✅ **Extensible** - Easy to add new skills or update existing ones
- ✅ **Open Source** - Community contributions welcome

---

## 🚀 Quick Start

### Installation

**Global installation (recommended):**
```bash
git clone https://github.com/alec-c4/claude-skills-rails-dev.git ~/.claude/skills
```

**Project-specific installation:**
```bash
git clone https://github.com/alec-c4/claude-skills-rails-dev.git /path/to/your-project/.claude/skills
```

**Start using** - Claude will automatically activate skills based on your requests!

**Update skills:**
```bash
cd ~/.claude/skills
git pull origin master
```

### Usage

Just describe what you want:

```bash
# Automatically activates rails-testing
"Write tests for User model with email validation"

# Automatically activates rails-viewcomponents
"Create a ButtonComponent with primary and danger variants"

# Automatically activates rails-security
"Add Pundit authorization to Articles controller"

# Complex tasks activate rails-project-manager
"Help me build a notification system"
```

No configuration needed - it just works! 🎉

---

## 📚 Available Skills

### 🎯 Coordination
| Skill | Purpose | Version |
|-------|---------|---------|
| **rails-project-manager** | Coordinates tasks, plans features | 1.0.0 |
| **ruby-on-rails-development** | Routes to specialists, core principles | 1.0.0 |

### 💻 Development
| Skill | Purpose | Version |
|-------|---------|---------|
| **rails-testing** | RSpec, FactoryBot, TDD workflows | 1.0.0 |
| **rails-viewcomponents** | ViewComponent, Turbo, Stimulus | 1.0.0 |
| **rails-business-logic** | ActiveInteraction, AASM, decorators | 1.0.0 |

### 🌐 API & Frontend
| Skill | Purpose | Version |
|-------|---------|---------|
| **rails-api** | REST APIs, JWT, serialization | 1.0.0 |
| **rails-graphql** | GraphQL schemas, mutations | 1.0.0 |
| **rails-inertia** | Inertia.js SPAs with React/Vue/Svelte | 1.0.0 |

### 🏗️ Infrastructure
| Skill | Purpose | Version |
|-------|---------|---------|
| **rails-background-jobs** | Solid Queue, scheduled tasks | 1.0.0 |
| **rails-devops** | Docker, CI/CD, deployment | 1.0.0 |
| **rails-security** | Pundit, Lockbox, authentication | 1.0.0 |

### 📊 Analysis & Documentation
| Skill | Purpose | Version |
|-------|---------|---------|
| **rails-analyst** | JTBD, use cases, estimation, risk analysis | 1.0.0 |
| **rails-technical-writer** | Documentation, README, guides | 1.0.0 |

**Full details:** See [SKILLS_REGISTRY.md](SKILLS_REGISTRY.md)

---

## 💡 How It Works

### Automatic Skill Selection

Claude analyzes your request and activates the appropriate skill:

```
YOU: "Optimize the slow dashboard query"

CLAUDE: [Detects keywords: "optimize", "slow", "query"]
        [Activates: rails-analyst]

        Analyzing query performance...
        Found N+1 query...
        Adding eager loading...
```

### Multi-Skill Coordination

For complex tasks, `rails-project-manager` coordinates multiple skills:

```
YOU: "Build a commenting system"

CLAUDE: [Activates: rails-project-manager]

        Planning implementation...

        Stage 1: Model & Tests → rails-testing
        Stage 2: Business Logic → rails-business-logic
        Stage 3: API Endpoints → rails-api
        Stage 4: UI Components → rails-viewcomponents

        Starting Stage 1...
```

---

## 🎓 Core Principles

All skills follow these **NON-NEGOTIABLE** principles:

### 1. Test-First Development (TDD)
```ruby
# RED → GREEN → REFACTOR
# Write test first, make it pass, then improve
```

### 2. YAGNI (You Aren't Gonna Need It)
```ruby
# Don't create abstractions until actually needed
```

### 3. Convention Over Configuration
```ruby
# Follow Rails conventions unless compelling reason
```

### 4. Security by Design
```ruby
# Authorization, encryption, HTTPS by default
```

### 5. Incremental Progress
```ruby
# Every commit: compiles + tests pass + deployable
```

---

## 🔧 Extending the System

### Adding a New Skill

1. **Create directory:**
   ```bash
   mkdir .claude/skills/rails-new-skill
   ```

2. **Create SKILL.md:**
   ```markdown
   ---
   name: rails-new-skill
   description: Brief description
   ---

   # Content...
   ```

3. **Register in [SKILLS_REGISTRY.md](SKILLS_REGISTRY.md):**
   - Add skill entry with version, keywords
   - Define routing rules
   - Document dependencies

4. **Test:**
   ```bash
   # Try various requests that should trigger your skill
   ```

### Updating Existing Skill

1. **Update skill content** in `rails-xxx/SKILL.md`
2. **Increment version** in `SKILLS_REGISTRY.md`
3. **Document changes** in skill's changelog
4. **Test** to ensure routing still works

### Version Format

```
MAJOR.MINOR.PATCH

MAJOR: Breaking changes, new Rails version
MINOR: New features, significant additions
PATCH: Bug fixes, minor improvements
```

---

## 📖 Documentation

| Document | Purpose |
|----------|---------|
| [SKILLS_REGISTRY.md](SKILLS_REGISTRY.md) | Complete catalog of all skills |
| [ruby-on-rails-development/SKILL.md](ruby-on-rails-development/SKILL.md) | Router & core principles |
| Each skill's `SKILL.md` | Detailed skill documentation |

---

## 🤝 Contributing

We welcome contributions!

### Ways to Contribute

1. **Report Issues**
   - Skill not activating correctly?
   - Documentation unclear?
   - Bug in skill logic?

2. **Improve Skills**
   - Add examples
   - Fix errors
   - Improve explanations
   - Update for new Rails versions

3. **Add New Skills**
   - Follow the extension guide
   - Ensure clear routing keywords
   - Include comprehensive examples

4. **Improve Documentation**
   - Clarify instructions
   - Add use cases
   - Fix typos

### Contribution Process

1. Fork the repository
2. Create feature branch (`git checkout -b feature/amazing-skill`)
3. Make changes
4. Update `SKILLS_REGISTRY.md` with version bumps
5. Commit (`git commit -m 'Add amazing skill'`)
6. Push (`git push origin feature/amazing-skill`)
7. Open Pull Request

---

## 🛣️ Roadmap

### Version 1.1 (Planned)
- [ ] Rails 8.1 specific features
- [ ] Enhanced Kamal 2.x deployment patterns
- [ ] Solid Cache integration
- [ ] Solid Cable patterns

### Version 1.2 (Ideas)
- [ ] Multi-database patterns
- [ ] Advanced Hotwire patterns
- [ ] Rails 9.0 preparation
- [ ] Performance benchmarking skill

---

## ✅ Quality Standards

All skills must:
- ✅ Follow TDD principles
- ✅ Include practical examples
- ✅ Document Rails version compatibility
- ✅ Have clear routing keywords
- ✅ Pass quality checks (RuboCop, Brakeman, etc.)

---

## 📊 Stats

- **Total Skills:** 13
- **Categories:** 5
- **Rails Versions Supported:** 7.0 - 8.1
- **Lines of Documentation:** 10,000+
- **Code Examples:** 200+

---

## 🙏 Acknowledgments

Built with best practices from:
- Rails official guides
- Rails community conventions
- Modern Rails patterns (ViewComponent, Hotwire, etc.)
- Production experience

---

## 📜 License

MIT License - See [LICENSE](LICENSE) file for details

---

## 📞 Support

- **Documentation:** [SKILLS_REGISTRY.md](SKILLS_REGISTRY.md)
- **Issues:** [GitHub Issues](https://github.com/alec-c4/claude-skills-rails-dev/issues)
- **Discussions:** [GitHub Discussions](https://github.com/alec-c4/claude-skills-rails-dev/discussions)

---

## 🌟 Show Your Support

If you find this helpful:
- ⭐ Star the repository
- 📢 Share with your team
- 🤝 Contribute improvements
- 💬 Provide feedback

---

**Made with ❤️ for the Rails community**

**Current Version:** 1.0.0
**Last Updated:** 2025-10-21
**Maintained By:** Community
