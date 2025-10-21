# Technical Writing Reference Guide

Comprehensive guide for documenting Rails applications with templates and examples.

## Table of Contents

- [Complete README Template](#complete-readme-template)
- [Setup Guides for All Platforms](#setup-guides-for-all-platforms)
- [Architecture Decision Records](#architecture-decision-records)
- [Feature Documentation](#feature-documentation)
- [API Documentation](#api-documentation)
- [Migration Guides](#migration-guides)
- [Troubleshooting Documentation](#troubleshooting-documentation)

---

## Complete README Template

```markdown
# Project Name

[![CI](https://github.com/org/repo/workflows/CI/badge.svg)](https://github.com/org/repo/actions)
[![Coverage](https://codecov.io/gh/org/repo/branch/main/graph/badge.svg)](https://codecov.io/gh/org/repo)

One-sentence description of what this application does.

Detailed paragraph explaining the purpose, target users, and key value proposition.

## Features

- **User Management** - Registration, authentication, email confirmation
- **Role-Based Access** - Admin, manager, and user roles with permissions
- **Real-Time Updates** - WebSocket-based notifications via Turbo Streams
- **Background Processing** - Async job handling with Solid Queue
- **Multi-Tenant** - Organization-based data isolation
- **API** - RESTful API with JWT authentication (v1)
- **Internationalization** - Support for English, Spanish, French

## Tech Stack

### Backend
- **Framework**: Ruby on Rails 7.1.3
- **Language**: Ruby 3.3.0
- **Database**: PostgreSQL 15
- **Cache**: Redis 7.2
- **Search**: Elasticsearch 8.x (optional)

### Frontend
- **CSS Framework**: Tailwind CSS 3.4
- **JavaScript**: Hotwire (Turbo + Stimulus)
- **Components**: ViewComponent
- **Build Tool**: Vite

### Infrastructure
- **Deployment**: Kamal 2.0
- **Background Jobs**: Solid Queue
- **File Storage**: Active Storage + S3
- **Monitoring**: Sentry, New Relic
- **Email**: SendGrid

### Development
- **Testing**: RSpec, FactoryBot, Capybara
- **Code Quality**: RuboCop, Brakeman, SimpleCov
- **Documentation**: YARD, OpenAPI

## Prerequisites

- **Ruby**: 3.3.0 (see `.ruby-version`)
- **PostgreSQL**: 15 or higher
- **Redis**: 7.0 or higher
- **Node.js**: 20 or higher
- **Yarn**: 1.22 or higher
- **Docker**: 24+ (optional, for development)

## Quick Start

### Using Docker (Recommended for Development)

```bash
# Clone repository
git clone git@github.com:org/project.git
cd project

# Start all services
docker-compose up -d

# Setup database
docker-compose exec web bundle exec rails db:setup

# Access application
open http://localhost:3000
```

### Local Setup

#### 1. Install dependencies

```bash
bundle install
yarn install
```

#### 2. Configure environment

```bash
# Copy environment template
cp .env.example .env

# Edit .env and configure:
# - DATABASE_URL
# - REDIS_URL
# - SECRET_KEY_BASE (generate with: rails secret)
# - LOCKBOX_MASTER_KEY (generate with: rails lockbox:install)
```

#### 3. Setup database

```bash
# Create databases
bundle exec rails db:create

# Run migrations
bundle exec rails db:migrate

# Seed data (creates admin user and sample data)
bundle exec rails db:seed
```

Default admin credentials:
- Email: `admin@example.com`
- Password: `password` (change immediately!)

#### 4. Start development server

```bash
# All services (Rails + Vite + Jobs)
bin/dev

# Or separately:
bundle exec rails server           # Rails (port 3000)
yarn run vite dev                   # Vite (port 5173)
bundle exec rails solid_queue:start # Background jobs
```

Visit: http://localhost:3000

## Testing

### Run all tests

```bash
bundle exec rspec
```

### Run specific tests

```bash
# Single file
bundle exec rspec spec/models/user_spec.rb

# Single test (by line)
bundle exec rspec spec/models/user_spec.rb:15

# By tag
bundle exec rspec --tag type:system
```

### With coverage

```bash
COVERAGE=true bundle exec rspec
open coverage/index.html
```

### System tests

```bash
# Headless (default)
bundle exec rspec spec/system

# With visible browser
HEADLESS=false bundle exec rspec spec/system
```

## Code Quality

### Linting

```bash
# Ruby linting
bundle exec rubocop

# Auto-fix issues
bundle exec rubocop -A

# ERB linting
bundle exec erb_lint --lint-all
```

### Security Scanning

```bash
# Security vulnerabilities
bundle exec brakeman --no-pager

# Dependency audit
bundle-audit check --update
yarn npm audit
```

### Type Checking (if using Sorbet)

```bash
bundle exec srb tc
```

## Database

### Migrations

```bash
# Create migration
rails generate migration AddFieldToModel field:type

# Run migrations
bundle exec rails db:migrate

# Rollback
bundle exec rails db:rollback

# Reset database (WARNING: destroys all data)
bundle exec rails db:reset
```

### Console

```bash
# Production console (use with caution!)
RAILS_ENV=production bundle exec rails console
```

## Background Jobs

### Monitor queue

```bash
# Via Mission Control
open http://localhost:3000/jobs

# Via console
bundle exec rails console
> SolidQueue::Job.count
> SolidQueue::Job.last
```

### Run jobs manually

```bash
# Start worker
bundle exec rails solid_queue:start

# Inline mode (for debugging)
# config/environments/development.rb
config.active_job.queue_adapter = :inline
```

## Deployment

### Staging

```bash
# Deploy to staging
kamal deploy -d staging

# Check status
kamal app logs -d staging
```

### Production

```bash
# Deploy to production
kamal deploy -d production

# Rollback
kamal rollback -d production
```

See [DEPLOYMENT.md](docs/DEPLOYMENT.md) for detailed deployment instructions.

## Development

### Project Structure

```
app/
├── components/        # ViewComponents
├── interactions/      # Business logic (ActiveInteraction)
├── jobs/             # Background jobs
├── models/           # ActiveRecord models
├── policies/         # Authorization (Pundit)
├── serializers/      # API serializers
└── views/            # ERB templates

spec/
├── components/       # Component specs
├── factories/        # FactoryBot factories
├── interactions/     # Interaction specs
├── models/          # Model specs
├── requests/        # API/Request specs
└── system/          # System/Feature specs
```

### Coding Conventions

- Follow [Ruby Style Guide](https://rubystyle.guide/)
- Use RuboCop configuration (`.rubocop.yml`)
- Write tests first (TDD)
- Keep interactions focused (single responsibility)
- Document complex business logic

### Adding a New Feature

1. Create feature branch: `git checkout -b feature/description`
2. Write failing tests
3. Implement feature
4. Ensure all tests pass
5. Update documentation
6. Create pull request

## API Documentation

API documentation available at: http://localhost:3000/api-docs

### Authentication

```bash
# Login to get JWT token
curl -X POST http://localhost:3000/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"user@example.com","password":"password"}'

# Use token in requests
curl http://localhost:3000/api/v1/articles \
  -H "Authorization: Bearer YOUR_TOKEN_HERE"
```

See [API.md](docs/API.md) for complete API documentation.

## Environment Variables

| Variable | Description | Required | Default |
|----------|-------------|----------|---------|
| `DATABASE_URL` | PostgreSQL connection string | Yes | - |
| `REDIS_URL` | Redis connection string | Yes | - |
| `SECRET_KEY_BASE` | Rails secret key | Yes | - |
| `LOCKBOX_MASTER_KEY` | Encryption key | Yes | - |
| `SENDGRID_API_KEY` | Email service key | No | - |
| `AWS_ACCESS_KEY_ID` | S3 access key | No | - |
| `AWS_SECRET_ACCESS_KEY` | S3 secret key | No | - |
| `SENTRY_DSN` | Error tracking | No | - |

## Troubleshooting

### Database connection errors

```bash
# Check PostgreSQL is running
pg_isready

# Restart PostgreSQL
brew services restart postgresql@15
```

### Redis connection errors

```bash
# Check Redis is running
redis-cli ping

# Restart Redis
brew services restart redis
```

### Asset compilation errors

```bash
# Clear cache
bundle exec rails tmp:clear

# Rebuild assets
yarn run vite build
```

See [TROUBLESHOOTING.md](docs/TROUBLESHOOTING.md) for more solutions.

## Contributing

We welcome contributions! Please see [CONTRIBUTING.md](CONTRIBUTING.md) for:
- Code of conduct
- Development setup
- Pull request process
- Coding standards

## Support

- **Documentation**: [docs/](docs/)
- **Issues**: [GitHub Issues](https://github.com/org/repo/issues)
- **Discussions**: [GitHub Discussions](https://github.com/org/repo/discussions)
- **Security**: security@example.com

## License

This project is licensed under the MIT License - see [LICENSE](LICENSE) file for details.

## Authors

- **Your Name** - *Initial work* - [@username](https://github.com/username)

See [CONTRIBUTORS.md](CONTRIBUTORS.md) for the full list of contributors.

---

**Version**: 1.0.0
**Last Updated**: 2025-10-21
```

---

## Setup Guides for All Platforms

### macOS Setup

```markdown
# Development Setup - macOS

## 1. Install Homebrew

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

## 2. Install Ruby

```bash
# Install rbenv
brew install rbenv ruby-build

# Install Ruby 3.3.0
rbenv install 3.3.0
rbenv global 3.3.0

# Add to shell (~/.zshrc or ~/.bash_profile)
echo 'eval "$(rbenv init - zsh)"' >> ~/.zshrc
source ~/.zshrc

# Verify
ruby -v  # Should show 3.3.0
```

## 3. Install PostgreSQL

```bash
# Install PostgreSQL 15
brew install postgresql@15

# Start service
brew services start postgresql@15

# Add to PATH
echo 'export PATH="/opt/homebrew/opt/postgresql@15/bin:$PATH"' >> ~/.zshrc
source ~/.zshrc

# Create database user (optional)
createuser -s postgres
```

## 4. Install Redis

```bash
brew install redis
brew services start redis
```

## 5. Install Node.js

```bash
# Using Homebrew
brew install node@20

# Or using nvm (recommended)
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.0/install.sh | bash
nvm install 20
nvm use 20
```

## 6. Install Yarn

```bash
npm install -g yarn
```

## 7. Clone Project

```bash
git clone git@github.com:org/project.git
cd project
```

## 8. Setup Application

```bash
# Install dependencies
bundle install
yarn install

# Configure environment
cp .env.example .env
# Edit .env with your settings

# Setup database
bundle exec rails db:create db:migrate db:seed
```

## 9. Start Development Server

```bash
bin/dev
```

Visit: http://localhost:3000

## Troubleshooting

### Issue: `pg_config not found`
```bash
export PATH="/opt/homebrew/opt/postgresql@15/bin:$PATH"
bundle install
```

### Issue: `Permission denied @ rb_sysopen`
```bash
sudo chown -R $(whoami) /opt/homebrew
```
```

### Linux (Ubuntu/Debian) Setup

```markdown
# Development Setup - Linux (Ubuntu/Debian)

## 1. Update System

```bash
sudo apt update
sudo apt upgrade -y
```

## 2. Install Dependencies

```bash
sudo apt install -y \
  git \
  curl \
  build-essential \
  libssl-dev \
  libreadline-dev \
  zlib1g-dev \
  libpq-dev \
  libsqlite3-dev
```

## 3. Install Ruby

```bash
# Install rbenv
curl -fsSL https://github.com/rbenv/rbenv-installer/raw/HEAD/bin/rbenv-installer | bash

# Add to ~/.bashrc
echo 'export PATH="$HOME/.rbenv/bin:$PATH"' >> ~/.bashrc
echo 'eval "$(rbenv init - bash)"' >> ~/.bashrc
source ~/.bashrc

# Install Ruby
rbenv install 3.3.0
rbenv global 3.3.0
```

## 4. Install PostgreSQL

```bash
# Add PostgreSQL repository
sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'
wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add -
sudo apt update

# Install PostgreSQL 15
sudo apt install -y postgresql-15

# Start service
sudo systemctl start postgresql
sudo systemctl enable postgresql

# Create user
sudo -u postgres createuser -s $USER
```

## 5. Install Redis

```bash
sudo apt install -y redis-server
sudo systemctl start redis-server
sudo systemctl enable redis-server
```

## 6. Install Node.js

```bash
# Using NodeSource
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs

# Install Yarn
npm install -g yarn
```

## 7. Setup Project

```bash
git clone git@github.com:org/project.git
cd project
bundle install
yarn install
cp .env.example .env
bundle exec rails db:create db:migrate db:seed
```

## 8. Start Server

```bash
bin/dev
```
```

### Windows (WSL2) Setup

```markdown
# Development Setup - Windows (WSL2)

## 1. Enable WSL2

```powershell
# Run in PowerShell as Administrator
wsl --install -d Ubuntu
```

Restart computer when prompted.

## 2. Configure Ubuntu

Open Ubuntu from Start menu and create user account.

## 3. Update System

```bash
sudo apt update && sudo apt upgrade -y
```

## 4-8. Follow Linux Setup

Continue with steps 2-8 from Linux (Ubuntu) setup guide above.

## WSL-Specific Tips

### Access from Windows
- Project files: `\\wsl$\Ubuntu\home\username\project`
- VS Code: Install "Remote - WSL" extension

### Performance
```bash
# Mount drives with metadata (in /etc/wsl.conf)
[automount]
options = "metadata"
```
```

---

## Architecture Decision Records

### ADR Template

```markdown
# ADR-{number}: {Title}

## Status
[Proposed | Accepted | Deprecated | Superseded by ADR-XXX]

## Context
What is the issue that we're seeing that is motivating this decision or change?

Include:
- Current situation
- Requirements
- Constraints
- Forces at play

## Decision
What is the change that we're proposing and/or doing?

Be specific and concrete. Use active voice.

## Rationale
Why did we choose this option over alternatives?

### Alternatives Considered
1. **Option A** - Why rejected
2. **Option B** - Why rejected

## Consequences

### Positive
- Benefit 1
- Benefit 2

### Negative
- Drawback 1
- Mitigation plan

### Neutral
- Other changes required

## Implementation
- Step 1
- Step 2
- Migration plan (if needed)

## Compliance
How will we ensure this decision is followed?

## Notes
Additional information, links to discussions, etc.

## References
- Link to related docs
- Link to PRs
- Link to discussions

---
**Author**: Name
**Date**: 2025-01-15
**Review Date**: 2025-07-15 (6 months)
```

### Complete ADR Example

```markdown
# ADR-003: Use Solid Queue Instead of Sidekiq

## Status
Accepted (2025-01-15)

## Context
We need a background job processing system that:
- Handles scheduled jobs
- Provides retry logic
- Scales with application growth
- Requires minimal infrastructure
- Has good monitoring/debugging tools

Currently evaluating:
- **Sidekiq** (current industry standard)
- **Solid Queue** (new Rails 7.1+ native solution)
- **Good Job** (Postgres-based alternative)

### Current Pain Points
- Redis dependency adds infrastructure complexity
- Sidekiq Pro license costs $179/month
- Redis memory pressure during traffic spikes
- Difficult to debug failed jobs across environments

## Decision
We will use Solid Queue as our background job processing system.

## Rationale

### Why Solid Queue?
1. **No additional infrastructure** - Uses existing PostgreSQL
2. **Native Rails integration** - First-class Rails 7.1+ support
3. **Mission Control UI** - Built-in web interface for monitoring
4. **Lower operational cost** - No Redis, no Sidekiq Pro license
5. **Simpler deployment** - One less service to manage
6. **Sufficient for our scale** - Current job volume: ~10,000/day

### Alternatives Considered

**Sidekiq:**
- ❌ Requires Redis infrastructure
- ❌ Pro license costs $179/month ($2,148/year)
- ✅ Battle-tested, mature
- ✅ Very high throughput
- **Rejected**: Cost and complexity not justified for our scale

**Good Job:**
- ✅ Postgres-based (like Solid Queue)
- ✅ Good performance
- ❌ Less mature than Solid Queue
- ❌ Smaller community
- **Rejected**: Solid Queue has better Rails integration

## Consequences

### Positive
- **Reduced costs**: Save $2,148/year on Sidekiq Pro
- **Simplified infrastructure**: Remove Redis dependency
- **Easier debugging**: Jobs in same database as app data
- **Better visibility**: Mission Control UI out of the box
- **Atomic operations**: Job + data changes in same transaction

### Negative
- **Database load**: Additional queries on PostgreSQL
- **Lower throughput**: ~5-10x slower than Redis-based solutions
- **Less mature**: Fewer production deployments than Sidekiq
- **Feature gaps**: Some Sidekiq Pro features not available

### Mitigation Plans
1. **Monitor PostgreSQL performance** - Add query monitoring
2. **Index optimization** - Ensure proper indexes on jobs table
3. **Connection pooling** - Configure separate pool for jobs
4. **Scaling plan** - If job volume exceeds 100k/day, re-evaluate

## Implementation

### Phase 1: New Jobs (Week 1)
```ruby
# Gemfile
gem 'solid_queue'

# config/environments/production.rb
config.active_job.queue_adapter = :solid_queue

# db/migrate/xxx_create_solid_queue_tables.rb
bundle exec rails solid_queue:install
bundle exec rails db:migrate
```

### Phase 2: Migration (Week 2-3)
- Migrate existing Sidekiq jobs to ActiveJob
- Test in staging
- Deploy to production
- Monitor for 1 week

### Phase 3: Cleanup (Week 4)
- Remove Sidekiq gem
- Remove Redis from infrastructure
- Update deployment scripts
- Update documentation

## Compliance
- All new background jobs MUST use ActiveJob
- Jobs MUST be idempotent
- Jobs MUST have timeout configured
- Failed jobs MUST be monitored in Mission Control

## Performance Benchmarks
Tested with 10,000 jobs:
- Solid Queue: 2.5 minutes
- Sidekiq: 18 seconds

At our current scale (10k/day), 2.5 minutes is acceptable.

## Notes
- Re-evaluate decision if job volume exceeds 100,000/day
- Consider separate database for jobs if primary DB shows strain
- Monitor database connection pool usage

## References
- [Solid Queue Documentation](https://github.com/rails/solid_queue)
- [Mission Control](https://github.com/rails/mission_control-jobs)
- [Internal: Job Volume Analysis](https://internal-wiki.com/job-analysis)
- [Benchmarks](https://internal-wiki.com/benchmarks)

---
**Author**: Engineering Team
**Date**: 2025-01-15
**Review Date**: 2025-07-15
```

---

## Feature Documentation

### Multi-Tenant Architecture Example

```markdown
# Multi-Tenant Architecture

## Overview
This application supports multiple organizations (tenants) with complete data isolation. Each organization has its own set of users, data, and configuration.

## Architecture

### Database Schema
Each tenant-scoped table includes `organization_id`:

```sql
CREATE TABLE organizations (
  id BIGINT PRIMARY KEY,
  name VARCHAR NOT NULL,
  subdomain VARCHAR UNIQUE NOT NULL,
  settings JSONB DEFAULT '{}'::jsonb
);

CREATE TABLE articles (
  id BIGINT PRIMARY KEY,
  organization_id BIGINT NOT NULL REFERENCES organizations(id),
  title VARCHAR NOT NULL,
  body TEXT,
  created_at TIMESTAMP NOT NULL,
  updated_at TIMESTAMP NOT NULL
);

CREATE INDEX idx_articles_organization ON articles(organization_id);
```

### Tenant Resolution

**By Subdomain:**
```
acme.myapp.com    → Organization(subdomain: 'acme')
contoso.myapp.com → Organization(subdomain: 'contoso')
```

**Implementation:**
```ruby
# app/controllers/application_controller.rb
class ApplicationController < ActionController::Base
  set_current_tenant_through_filter
  before_action :set_tenant

  private

  def set_tenant
    subdomain = request.subdomain
    organization = Organization.find_by!(subdomain: subdomain)
    set_current_tenant(organization)
  rescue ActiveRecord::RecordNotFound
    redirect_to root_url(subdomain: false), alert: "Organization not found"
  end
end
```

### Model Configuration

```ruby
# app/models/article.rb
class Article < ApplicationRecord
  belongs_to :organization
  acts_as_tenant :organization

  validates :title, presence: true
  validates :organization, presence: true
end

# Automatic scoping:
Article.all  # Only returns articles for current tenant
Article.create(title: "Test")  # Automatically sets organization_id
```

### Background Jobs

```ruby
# app/jobs/export_data_job.rb
class ExportDataJob < ApplicationJob
  queue_as :default

  def perform(organization_id, user_id)
    organization = Organization.find(organization_id)

    ActsAsTenant.with_tenant(organization) do
      user = User.find(user_id)
      articles = Article.all  # Scoped to organization

      # Generate export
      ExportService.generate(user, articles)
    end
  end
end
```

### Testing

```ruby
# spec/models/article_spec.rb
RSpec.describe Article do
  let(:organization) { create(:organization) }

  before do
    ActsAsTenant.current_tenant = organization
  end

  it "scopes to current tenant" do
    article = create(:article)
    expect(Article.all).to include(article)
  end

  it "prevents cross-tenant access" do
    other_org = create(:organization)
    other_article = nil

    ActsAsTenant.with_tenant(other_org) do
      other_article = create(:article)
    end

    expect(Article.all).not_to include(other_article)
  end
end
```

## Security Considerations

1. **Never bypass tenant scoping** without explicit reason
2. **Always validate organization_id** in controllers
3. **Use `with_tenant`** for admin operations
4. **Audit cross-tenant queries** regularly

## Gotchas

### Migrations
```ruby
# DON'T do this in migrations:
Article.update_all(status: 'active')  # Updates ALL organizations!

# DO this instead:
Organization.find_each do |org|
  ActsAsTenant.with_tenant(org) do
    Article.update_all(status: 'active')  # Scoped to org
  end
end
```

### Rake Tasks
```ruby
# Don't forget tenant context!
task export: :environment do
  Organization.find_each do |org|
    ActsAsTenant.with_tenant(org) do
      # Your task code here
    end
  end
end
```

### Seeds
```ruby
# db/seeds.rb
org = Organization.create!(name: "Demo", subdomain: "demo")

ActsAsTenant.with_tenant(org) do
  User.create!(email: "admin@demo.com", password: "password")
  Article.create!(title: "Welcome", body: "...")
end
```

## Monitoring

Track tenant activity:
```ruby
# app/models/organization.rb
class Organization < ApplicationRecord
  has_many :users
  has_many :articles

  def active_users_count
    users.where("last_sign_in_at > ?", 30.days.ago).count
  end

  def storage_used
    articles.sum(:content_size)
  end
end
```

## References
- [ActsAsTenant Documentation](https://github.com/ErwinM/acts_as_tenant)
- [Multi-Tenancy Strategies](https://docs.example.com/multitenancy)
- ADR-005: Multi-Tenant Architecture Decision
```

---

## Migration Guides

### Breaking Change Example

```markdown
# Migration Guide: API v1 to v2

## Overview
API v2 introduces breaking changes to improve consistency and performance.

**Timeline:**
- v2 Released: 2025-01-15
- v1 Deprecated: 2025-01-15
- v1 Sunset: 2025-07-15 (6 months)

## Breaking Changes

### 1. Authentication Response Structure

**v1 (Old):**
```json
{
  "token": "abc123",
  "user": { "id": 1, "name": "John" }
}
```

**v2 (New):**
```json
{
  "access_token": "abc123",
  "token_type": "Bearer",
  "expires_in": 3600,
  "refresh_token": "def456",
  "user": {
    "id": 1,
    "type": "users",
    "attributes": { "name": "John" }
  }
}
```

**Migration:**
```javascript
// Before (v1)
const token = response.token
const user = response.user

// After (v2)
const token = response.access_token
const user = response.user.attributes
```

### 2. Pagination Format

**v1 (Old):**
```json
{
  "articles": [...],
  "page": 1,
  "total": 100
}
```

**v2 (New):**
```json
{
  "data": [...],
  "meta": {
    "current_page": 1,
    "total_pages": 10,
    "total_count": 100,
    "per_page": 10
  }
}
```

**Migration:**
```javascript
// Before (v1)
const articles = response.articles
const currentPage = response.page

// After (v2)
const articles = response.data
const currentPage = response.meta.current_page
```

### 3. Error Format

**v1 (Old):**
```json
{ "error": "Invalid email" }
```

**v2 (New):**
```json
{
  "errors": [
    {
      "status": "422",
      "code": "invalid_email",
      "title": "Validation Failed",
      "detail": "Email is invalid",
      "source": { "pointer": "/data/attributes/email" }
    }
  ]
}
```

## Migration Steps

### Step 1: Update Base URL

```javascript
// Change from
const API_URL = 'https://api.example.com/v1'

// To
const API_URL = 'https://api.example.com/v2'
```

### Step 2: Update Authentication

```javascript
// Update login handler
async function login(email, password) {
  const response = await fetch(`${API_URL}/auth/login`, {
    method: 'POST',
    body: JSON.stringify({ email, password })
  })

  const data = await response.json()

  // v2: Store both tokens
  localStorage.setItem('access_token', data.access_token)
  localStorage.setItem('refresh_token', data.refresh_token)

  // v2: Extract user from attributes
  return data.user.attributes
}
```

### Step 3: Update Request Headers

```javascript
// v1
headers: {
  'Authorization': `Token ${token}`
}

// v2
headers: {
  'Authorization': `Bearer ${accessToken}`
}
```

### Step 4: Update Response Handling

```javascript
// v1
function handleArticles(response) {
  return response.articles
}

// v2
function handleArticles(response) {
  return response.data.map(article => article.attributes)
}
```

### Step 5: Update Error Handling

```javascript
// v1
if (response.error) {
  showError(response.error)
}

// v2
if (response.errors) {
  response.errors.forEach(error => {
    showError(error.detail)
  })
}
```

## Testing Checklist

- [ ] Authentication flow works
- [ ] Token refresh works
- [ ] List endpoints return correct data
- [ ] Pagination works correctly
- [ ] Error handling works
- [ ] All API calls use v2 endpoints
- [ ] Removed all v1 references

## Support

Questions? Contact: api-support@example.com
```

---

**Remember**: Good documentation is maintained, accurate, and helpful. Update it with every significant change!
