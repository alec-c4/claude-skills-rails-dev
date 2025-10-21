# Rails Skills Registry

> **Version:** 1.0.0
> **Last Updated:** 2025-10-21
> **Compatible Rails:** 7.0, 7.1, 7.2, 8.0, 8.1
> **Compatible Ruby:** 3.2, 3.3+

This registry defines all available Rails development skills and their relationships.

---

## 📋 Quick Reference

| Skill | Category | Use When |
|-------|----------|----------|
| [rails-project-manager](#rails-project-manager) | Coordination | Planning features, breaking down tasks |
| [rails-testing](#rails-testing) | Development | Writing tests, TDD workflows |
| [rails-viewcomponents](#rails-viewcomponents) | Development | Building UI components, Turbo/Stimulus |
| [rails-security](#rails-security) | Infrastructure | Authorization, encryption, security |
| [rails-business-logic](#rails-business-logic) | Development | Interactions, state machines |
| [rails-background-jobs](#rails-background-jobs) | Infrastructure | Background tasks, job queues |
| [rails-api](#rails-api) | API | REST endpoints, serialization |
| [rails-graphql](#rails-graphql) | API | GraphQL schemas, mutations |
| [rails-inertia](#rails-inertia) | API | Inertia.js SPAs with React/Vue |
| [rails-devops](#rails-devops) | Infrastructure | Docker, CI/CD, deployment |
| [rails-analyst](#rails-analyst) | Analysis | Performance profiling, optimization |
| [rails-technical-writer](#rails-technical-writer) | Analysis | Documentation, README, API docs |

---

## 🎯 Coordination Skills

### rails-project-manager

**Version:** 1.0.0
**Category:** Coordination
**Status:** Stable

**Description:**
Project coordination and task management. Analyzes requirements, breaks down tasks into stages, coordinates other skills.

**Use When:**
- Planning new features or major refactorings
- Breaking down complex requirements into stages
- Coordinating multiple aspects (testing, security, UI, backend)
- Managing multi-step development workflows

**Keywords:**
`plan`, `coordinate`, `stages`, `breakdown`, `manage`, `organize`, `strategy`

**Can Delegate To:**
All other skills

**Dependencies:**
None

**Related Resources:**
- `IMPLEMENTATION_PLAN.md` template
- Project management patterns

**Example Triggers:**
- "Help me plan implementation of..."
- "Break down this feature into stages"
- "What's the best approach for..."

---

## 💻 Development Skills

### rails-testing

**Version:** 1.0.0
**Category:** Development
**Status:** Stable

**Description:**
RSpec, FactoryBot, TDD workflows. Covers unit tests, integration tests, system tests, component tests.

**Use When:**
- Writing tests (model, controller, request, system, component)
- Implementing TDD workflow
- Creating FactoryBot factories
- Setting up test coverage with SimpleCov

**Keywords:**
`test`, `spec`, `rspec`, `factory`, `tdd`, `coverage`, `capybara`, `system test`

**Dependencies:**
None

**Shared Resources:**
- `testing.md` - Comprehensive testing guide

**Rails Versions:**
- Rails 7.0-7.1: RSpec 6.x
- Rails 7.2-8.1: RSpec 7.x (when available)

**Example Triggers:**
- "Write tests for User model"
- "Create factory for Article"
- "Implement TDD for..."

---

### rails-viewcomponents

**Version:** 1.0.0
**Category:** Development
**Status:** Stable

**Description:**
ViewComponent, Turbo, Stimulus, Hotwire. Build modern, component-based UIs.

**Use When:**
- Creating ViewComponents
- Building Turbo Frames and Streams
- Writing Stimulus controllers
- Implementing custom confirmation modals

**Keywords:**
`component`, `turbo`, `stimulus`, `hotwire`, `view`, `frontend`, `viewcomponent`, `lookbook`

**Dependencies:**
None

**Shared Resources:**
- `frontend.md` - Frontend patterns

**Rails Versions:**
- Rails 7.0+: ViewComponent 3.x, Turbo 7.x, Stimulus 3.x
- Rails 8.0+: ViewComponent 3.x, Turbo 8.x, Stimulus 3.x

**Example Triggers:**
- "Create ButtonComponent with variants"
- "Implement inline editing with Turbo Frame"
- "Add Stimulus controller for..."

---

### rails-business-logic

**Version:** 1.0.0
**Category:** Development
**Status:** Stable

**Description:**
ActiveInteraction, AASM state machines, ActiveDecorator. Business logic patterns.

**Use When:**
- Creating business operations (instead of service objects)
- Implementing state machines
- Adding presentation logic with decorators

**Keywords:**
`interaction`, `state machine`, `aasm`, `decorator`, `service`, `business logic`, `workflow`

**Dependencies:**
None

**Shared Resources:**
- `business-logic.md` - Business logic patterns

**Rails Versions:**
- All: ActiveInteraction 5.x, AASM 5.x, ActiveDecorator 1.x

**Example Triggers:**
- "Create interaction for user registration"
- "Add state machine for Order"
- "Implement decorator for..."

---

## 🌐 API Skills

### rails-api

**Version:** 1.0.0
**Category:** API
**Status:** Stable

**Description:**
REST API specialist. RESTful endpoints, serialization, JWT authentication, API versioning.

**Use When:**
- Creating REST API endpoints
- Implementing serialization (ActiveModel::Serializers, Blueprinter)
- JWT/token authentication
- API versioning strategies

**Keywords:**
`api`, `rest`, `jwt`, `endpoint`, `serializer`, `json`, `authentication`, `versioning`

**Dependencies:**
None

**Rails Versions:**
- Rails 7.0+: API mode with strong parameters
- Rails 8.0+: Enhanced API features

**Example Triggers:**
- "Create REST API for Articles"
- "Implement JWT authentication"
- "Add API versioning"

---

### rails-graphql

**Version:** 1.0.0
**Category:** API
**Status:** Stable

**Description:**
GraphQL specialist. Schemas, resolvers, mutations, subscriptions, DataLoader.

**Use When:**
- Creating GraphQL schemas and types
- Implementing query resolvers
- Building mutations
- Setting up subscriptions (real-time)
- Preventing N+1 with DataLoader

**Keywords:**
`graphql`, `mutation`, `resolver`, `subscription`, `schema`, `dataloader`, `query`

**Dependencies:**
None

**Shared Resources:**
- `graphql-reference.md` - GraphQL patterns

**Rails Versions:**
- Rails 7.0+: graphql-ruby 2.x
- Rails 8.0+: graphql-ruby 2.x with enhanced features

**Example Triggers:**
- "Create GraphQL schema for User"
- "Implement mutation for..."
- "Add subscription for real-time updates"

---

### rails-inertia

**Version:** 1.0.0
**Category:** API
**Status:** Stable

**Description:**
Inertia.js specialist. Build SPAs with React/Vue/Svelte and Rails backend.

**Use When:**
- Creating Inertia pages (React/Vue/Svelte components)
- Handling forms with Inertia
- Implementing client-side routing
- Managing shared props and flash messages

**Keywords:**
`inertia`, `react`, `vue`, `svelte`, `spa`, `ssr`, `vite`, `client-side`

**Dependencies:**
None

**Rails Versions:**
- Rails 7.0+: Inertia Rails 3.x
- Rails 8.0+: Enhanced Vite integration

**Example Triggers:**
- "Create Inertia page for articles list"
- "Implement form with Inertia"
- "Setup SSR for Inertia"

---

## 🏗️ Infrastructure Skills

### rails-background-jobs

**Version:** 1.0.0
**Category:** Infrastructure
**Status:** Stable

**Description:**
Solid Queue specialist. Background jobs, scheduled tasks, recurring jobs.

**Use When:**
- Creating background jobs
- Scheduling delayed tasks
- Setting up recurring jobs (cron-like)
- Testing jobs
- Monitoring job queues

**Keywords:**
`job`, `queue`, `background`, `sidekiq`, `solid queue`, `async`, `recurring`, `scheduled`

**Dependencies:**
None

**Shared Resources:**
- `background-jobs.md` - Job patterns

**Rails Versions:**
- Rails 7.1+: Solid Queue (recommended)
- Rails 8.0+: Solid Queue with Mission Control

**Example Triggers:**
- "Create job for sending emails"
- "Setup recurring cleanup job"
- "Implement retry strategy for..."

---

### rails-devops

**Version:** 1.0.0
**Category:** Infrastructure
**Status:** Stable

**Description:**
DevOps specialist. Docker, CI/CD, deployment, monitoring, production config.

**Use When:**
- Docker containerization
- CI/CD pipeline setup (GitHub Actions, GitLab CI)
- Production configuration
- Database backups
- Monitoring and logging

**Keywords:**
`docker`, `deploy`, `ci`, `cd`, `kubernetes`, `production`, `monitoring`, `heroku`, `fly.io`

**Dependencies:**
None

**Shared Resources:**
- `devops-reference.md` - DevOps patterns

**Rails Versions:**
- Rails 7.0+: Standard deployment patterns
- Rails 8.0+: Kamal 2.x for deployment

**Example Triggers:**
- "Setup Docker for Rails app"
- "Create GitHub Actions pipeline"
- "Configure production environment"

---

### rails-security

**Version:** 1.0.0
**Category:** Infrastructure
**Status:** Stable

**Description:**
Security specialist. Pundit, Lockbox, Blind Index, authentication, authorization.

**Use When:**
- Implementing Pundit policies
- Encrypting sensitive data (Lockbox)
- Searchable encryption (Blind Index)
- Configuring authentication (Devise)
- Fixing security vulnerabilities

**Keywords:**
`security`, `pundit`, `lockbox`, `encrypt`, `policy`, `auth`, `authorization`, `blind index`, `devise`

**Dependencies:**
None

**Shared Resources:**
- `security.md` - Security guide

**Rails Versions:**
- Rails 7.0+: Lockbox 1.x, Pundit 2.x
- Rails 8.0+: Enhanced encryption features

**Example Triggers:**
- "Create Pundit policy for Article"
- "Encrypt email field with searchability"
- "Add role-based authorization"

---

## 📊 Analysis Skills

### rails-analyst

**Version:** 1.0.0
**Category:** Analysis
**Status:** Stable

**Description:**
Business and systems analyst. Feature decomposition, task estimation, JTBD, use cases, architecture documentation, risk analysis, project weakness analysis.

**Use When:**
- Decomposing features into implementable tasks
- Estimating complexity (story points, time)
- Writing JTBD (Jobs To Be Done)
- Creating use cases and user stories
- Documenting architecture decisions (ADRs)
- Identifying risks and weaknesses
- Requirements gathering

**Keywords:**
`decompose`, `estimate`, `jtbd`, `use case`, `architecture`, `adr`, `risk`, `requirements`, `story points`, `user story`, `epic`, `breakdown`

**Dependencies:**
None

**Rails Versions:**
- All versions

**Example Triggers:**
- "Decompose this feature into tasks"
- "Write JTBD for user authentication"
- "Create use cases for article publishing"
- "Identify risks in this feature"
- "Estimate complexity of commenting system"
- "Document architecture decision for..."

---

### rails-technical-writer

**Version:** 1.0.0
**Category:** Analysis
**Status:** Stable

**Description:**
Technical documentation specialist. README, API docs, ADRs, setup guides.

**Use When:**
- Writing or updating README
- Creating API documentation
- Writing architectural decision records (ADRs)
- Setup/installation guides
- Inline code documentation

**Keywords:**
`documentation`, `readme`, `docs`, `adr`, `guide`, `api doc`, `swagger`, `openapi`

**Dependencies:**
None

**Rails Versions:**
- All: Standard documentation practices

**Example Triggers:**
- "Write README for this project"
- "Create API documentation for..."
- "Document the architecture decision for..."

---

## 🔄 Skill Routing

When Claude receives a request, it matches keywords to activate the appropriate skill:

| Keywords Found | Activated Skill |
|----------------|-----------------|
| test, spec, factory | rails-testing |
| component, turbo, stimulus | rails-viewcomponents |
| security, pundit, encrypt | rails-security |
| interaction, state machine | rails-business-logic |
| job, queue, background | rails-background-jobs |
| api, rest, jwt | rails-api |
| graphql, mutation | rails-graphql |
| inertia, react, vue, svelte | rails-inertia |
| docker, deploy, ci | rails-devops |
| decompose, estimate, jtbd, use case, adr | rails-analyst |
| documentation, readme | rails-technical-writer |
| plan, coordinate, breakdown | rails-project-manager |

**Default:** If no clear match, `rails-project-manager` coordinates.

---

## 🔧 Extending the Skills

### Adding a New Skill

1. **Create skill directory:**
   ```bash
   mkdir .claude/skills/rails-new-skill
   ```

2. **Create SKILL.md:**
   ```markdown
   ---
   name: rails-new-skill
   description: Brief description
   ---
   # Skill content...
   ```

3. **Update this registry:**
   - Add entry in appropriate category
   - Define version, keywords, dependencies
   - Add routing rules

4. **Test the skill:**
   - Try various keyword combinations
   - Verify it activates correctly

### Updating an Existing Skill

1. **Update skill content** in `SKILL.md`
2. **Increment version** in registry entry
3. **Update "Last Updated" date** in registry
4. **Document changes** in skill's changelog

### Version Management

Format: `MAJOR.MINOR.PATCH`

- **MAJOR**: Breaking changes, new Rails version support
- **MINOR**: New features, significant additions
- **PATCH**: Bug fixes, minor improvements

---

## 📝 Contributing

To contribute updates:

1. Fork the repository
2. Update skill content or registry
3. Increment version numbers
4. Submit pull request with clear description

---

## 📜 License

MIT License - See LICENSE file for details

---

**Maintained by:** Community
**Repository:** https://github.com/alec-c4/claude-skills-rails-dev
**Issues:** https://github.com/alec-c4/claude-skills-rails-dev/issues
