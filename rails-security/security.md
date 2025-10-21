# Security & Data Protection Reference

## Table of Contents
- [Encryption at Rest (Lockbox)](#encryption-at-rest-lockbox)
- [Searchable Encryption (Blind Index)](#searchable-encryption-blind-index)
- [Authorization (Pundit)](#authorization-pundit)
- [Authentication Best Practices](#authentication-best-practices)
- [Secure Configuration](#secure-configuration)
- [Common Security Pitfalls](#common-security-pitfalls)

## Encryption at Rest (Lockbox)

Encrypt sensitive data in the database using Lockbox.

### Setup

```ruby
# Gemfile
gem "lockbox"

# Generate master key
$ rails lockbox:install

# This creates config/master.key (DO NOT COMMIT!)
```

### Model Encryption

```ruby
# app/models/user.rb
class User < ApplicationRecord
  ### additional config (accepts_nested_attribute_for, etc.)------------------------------------------------------------
  # Encrypt sensitive fields
  has_encrypted :email, :ssn, :phone, :credit_card_number
  
  # Lockbox creates virtual attributes:
  # - email_ciphertext (stored in DB)
  # - email (decrypted, read/write)
end
```

### Migration for Encrypted Fields

```ruby
class AddEncryptedFieldsToUsers < ActiveRecord::Migration[7.1]
  def change
    # Store ciphertext, not plain text
    add_column :users, :email_ciphertext, :text
    add_column :users, :ssn_ciphertext, :text
    add_column :users, :phone_ciphertext, :text
    add_column :users, :credit_card_number_ciphertext, :text
    
    # Optional: Remove unencrypted columns
    # remove_column :users, :email
  end
end
```

### Usage

```ruby
# Writing encrypted data
user = User.create!(
  email: "user@example.com",      # Automatically encrypted
  ssn: "123-45-6789",
  phone: "+1-555-0123"
)

# Reading encrypted data
user.email  # => "user@example.com" (automatically decrypted)

# Database stores encrypted
user.email_ciphertext  # => "encrypted_string..."

# Search won't work on encrypted fields alone
User.where(email: "user@example.com")  # Won't work!
# Solution: Use Blind Index (see next section)
```

### Hybrid Encryption

Use hybrid cryptography for better performance:

```ruby
class User < ApplicationRecord
  has_encrypted :large_document, hybrid: true
end
```

### Encryption Options

```ruby
class User < ApplicationRecord
  # Per-field encryption keys
  has_encrypted :email, key: ENV['EMAIL_ENCRYPTION_KEY']
  
  # Custom encryption algorithm
  has_encrypted :data, algorithm: "aes-256-gcm"
  
  # Encode as Base64
  has_encrypted :binary_data, encode: true
end
```

## Searchable Encryption (Blind Index)

Enable searching on encrypted fields using Blind Index.

### Setup

```ruby
# Gemfile
gem "blind_index"

# Requires Lockbox installation first
```

### Add Blind Index to Model

```ruby
# app/models/user.rb
class User < ApplicationRecord
  ### additional config (accepts_nested_attribute_for, etc.)------------------------------------------------------------
  has_encrypted :email, :phone
  
  # Add blind indexes for searchable fields
  blind_index :email
  blind_index :phone
end
```

### Migration for Blind Index

```ruby
class AddBlindIndexesToUsers < ActiveRecord::Migration[7.1]
  def change
    add_column :users, :email_bidx, :string
    add_column :users, :phone_bidx, :string
    
    add_index :users, :email_bidx, unique: true
    add_index :users, :phone_bidx
  end
end
```

### Backfill Existing Data

```ruby
# One-time migration to compute blind indexes
class BackfillBlindIndexes < ActiveRecord::Migration[7.1]
  def up
    User.find_each do |user|
      user.compute_email_bidx
      user.compute_phone_bidx
      user.save!
    end
  end
end
```

### Usage

```ruby
# Find by encrypted field (works with blind index)
user = User.find_by(email: "user@example.com")

# Where clause
users = User.where(email: "user@example.com")

# Case-insensitive search (configure in model)
class User < ApplicationRecord
  blind_index :email, case_sensitive: false
end
```

### Security Considerations

**What Blind Index Reveals**:
- ✓ Whether two records have the same value
- ✗ Does NOT reveal the actual value
- ✗ Does NOT reveal partial values
- ✗ Does NOT enable range queries

**Limitations**:
```ruby
# Works
User.find_by(email: "exact@match.com")

# Doesn't work
User.where("email LIKE ?", "%partial%")  # Can't do partial search
User.where("email > ?", "a@a.com")       # Can't do range queries
```

## Authorization (Pundit)

Implement authorization using Pundit policies.

### Setup

```ruby
# Gemfile
gem "pundit"

# Install
$ rails generate pundit:install

# app/controllers/application_controller.rb
class ApplicationController < ActionController::Base
  include Pundit::Authorization
  
  rescue_from Pundit::NotAuthorizedError, with: :user_not_authorized
  
  private
  
  def user_not_authorized
    flash[:alert] = "You are not authorized to perform this action."
    redirect_to(request.referrer || root_path)
  end
end
```

### Policy Structure

```ruby
# app/policies/application_policy.rb
class ApplicationPolicy
  attr_reader :user, :record
  
  def initialize(user, record)
    @user = user
    @record = record
  end
  
  def index?
    false
  end
  
  def show?
    false
  end
  
  def create?
    false
  end
  
  def new?
    create?
  end
  
  def update?
    false
  end
  
  def edit?
    update?
  end
  
  def destroy?
    false
  end
  
  class Scope
    def initialize(user, scope)
      @user = user
      @scope = scope
    end
    
    def resolve
      raise NotImplementedError
    end
    
    private
    
    attr_reader :user, :scope
  end
end
```

### Example Policy

```ruby
# app/policies/article_policy.rb
class ArticlePolicy < ApplicationPolicy
  def index?
    true  # Anyone can view list
  end
  
  def show?
    record.published? || user_is_author?
  end
  
  def create?
    user.present?
  end
  
  def update?
    user_is_author? || user&.admin?
  end
  
  def destroy?
    user_is_author? || user&.admin?
  end
  
  class Scope < Scope
    def resolve
      if user&.admin?
        scope.all
      elsif user
        scope.where(published: true).or(scope.where(user: user))
      else
        scope.where(published: true)
      end
    end
  end
  
  private
  
  def user_is_author?
    user && record.user == user
  end
end
```

### Using Policies in Controllers

```ruby
# app/controllers/articles_controller.rb
class ArticlesController < ApplicationController
  before_action :set_article, only: [:show, :edit, :update, :destroy]
  
  def index
    @articles = policy_scope(Article)
  end
  
  def show
    authorize @article
  end
  
  def new
    @article = Article.new
    authorize @article
  end
  
  def create
    @article = current_user.articles.build(article_params)
    authorize @article
    
    if @article.save
      redirect_to @article
    else
      render :new
    end
  end
  
  def edit
    authorize @article
  end
  
  def update
    authorize @article
    
    if @article.update(article_params)
      redirect_to @article
    else
      render :edit
    end
  end
  
  def destroy
    authorize @article
    @article.destroy
    redirect_to articles_path
  end
  
  private
  
  def set_article
    @article = Article.find(params[:id])
  end
end
```

### Using Policies in Views

```erb
<% if policy(@article).update? %>
  <%= link_to "Edit", edit_article_path(@article) %>
<% end %>

<% if policy(@article).destroy? %>
  <%= button_to "Delete", article_path(@article), method: :delete %>
<% end %>

<% if policy(Article).create? %>
  <%= link_to "New Article", new_article_path %>
<% end %>
```

### Policy Testing

```ruby
# spec/policies/article_policy_spec.rb
RSpec.describe ArticlePolicy do
  subject { described_class.new(user, article) }
  
  let(:article) { create(:article, user: author) }
  let(:author) { create(:user) }
  
  context "for an anonymous user" do
    let(:user) { nil }
    
    it { is_expected.to permit_action(:index) }
    it { is_expected.to permit_action(:show) } # if published
    it { is_expected.to forbid_action(:create) }
    it { is_expected.to forbid_action(:update) }
    it { is_expected.to forbid_action(:destroy) }
  end
  
  context "for a regular user" do
    let(:user) { create(:user) }
    
    it { is_expected.to permit_action(:index) }
    it { is_expected.to permit_action(:create) }
    it { is_expected.to forbid_action(:update) }
    it { is_expected.to forbid_action(:destroy) }
  end
  
  context "for the author" do
    let(:user) { author }
    
    it { is_expected.to permit_action(:update) }
    it { is_expected.to permit_action(:destroy) }
  end
  
  context "for an admin" do
    let(:user) { create(:user, :admin) }
    
    it { is_expected.to permit_action(:update) }
    it { is_expected.to permit_action(:destroy) }
  end
end
```

## Authentication Best Practices

### Devise Configuration

```ruby
# config/initializers/devise.rb
Devise.setup do |config|
  # Strong password requirements
  config.password_length = 12..128
  config.paranoid = true  # Don't reveal if email exists
  
  # Lockout after failed attempts
  config.lock_strategy = :failed_attempts
  config.maximum_attempts = 5
  config.unlock_strategy = :time
  config.unlock_in = 1.hour
  
  # Session timeout
  config.timeout_in = 30.minutes
  
  # Confirmable
  config.reconfirmable = true
  config.confirm_within = 3.days
end
```

### Secure Password Storage

```ruby
# Devise handles this automatically, but for custom auth:
class User < ApplicationRecord
  has_secure_password
  
  validates :password, length: { minimum: 12 }, if: :password_required?
  validates :password, format: {
    with: /(?=.*[a-z])(?=.*[A-Z])(?=.*\d)/,
    message: "must include lowercase, uppercase, and number"
  }, if: :password_required?
end
```

### Two-Factor Authentication

```ruby
# Gemfile
gem "devise"
gem "devise-two-factor"
gem "rqrcode"

# app/models/user.rb
class User < ApplicationRecord
  devise :two_factor_authenticatable, otp_secret_encryption_key: ENV['OTP_SECRET_KEY']
  
  has_one_time_password(encrypted: true)
end
```

## Secure Configuration

### Secrets Management

**Never commit secrets to version control**:

```bash
# .gitignore
/.env
/config/master.key
/config/credentials/*.key
```

**Use Rails Credentials**:

```bash
# Edit credentials
$ EDITOR=vim rails credentials:edit

# In credentials file:
aws:
  access_key_id: YOUR_KEY
  secret_access_key: YOUR_SECRET
stripe:
  publishable_key: pk_live_...
  secret_key: sk_live_...
```

```ruby
# Access in code
Rails.application.credentials.aws[:access_key_id]
Rails.application.credentials.stripe[:secret_key]
```

**Environment-Specific Credentials**:

```bash
$ EDITOR=vim rails credentials:edit --environment production
```

### HTTPS Enforcement

```ruby
# config/environments/production.rb
config.force_ssl = true
```

### Content Security Policy

```ruby
# config/initializers/content_security_policy.rb
Rails.application.config.content_security_policy do |policy|
  policy.default_src :self, :https
  policy.font_src    :self, :https, :data
  policy.img_src     :self, :https, :data
  policy.object_src  :none
  policy.script_src  :self, :https
  policy.style_src   :self, :https
end
```

### CORS Configuration

```ruby
# Gemfile
gem "rack-cors"

# config/initializers/cors.rb
Rails.application.config.middleware.insert_before 0, Rack::Cors do
  allow do
    origins "https://example.com"
    resource "/api/*",
      headers: :any,
      methods: [:get, :post, :patch, :put, :delete]
  end
end
```

## Common Security Pitfalls

### Mass Assignment

**Problem**:
```ruby
# VULNERABLE
def create
  User.create(params[:user])  # Allows any attribute!
end
```

**Solution**:
```ruby
# SAFE
def create
  User.create(user_params)
end

private

def user_params
  params.require(:user).permit(:name, :email, :password)
end
```

### SQL Injection

**Problem**:
```ruby
# VULNERABLE
User.where("name = '#{params[:name]}'")
```

**Solution**:
```ruby
# SAFE - Use parameterized queries
User.where("name = ?", params[:name])
User.where(name: params[:name])
```

### XSS (Cross-Site Scripting)

**Problem**:
```erb
<%# VULNERABLE %>
<%= raw @article.body %>
<%= @article.body.html_safe %>
```

**Solution**:
```erb
<%# SAFE - Escape by default %>
<%= @article.body %>

<%# Or use sanitize for allowed HTML %>
<%= sanitize @article.body, tags: %w[p br strong em] %>
```

### CSRF (Cross-Site Request Forgery)

Rails protects automatically, but ensure:

```ruby
# app/controllers/application_controller.rb
class ApplicationController < ActionController::Base
  protect_from_forgery with: :exception  # Default
end
```

```erb
<%# CSRF token included automatically in forms %>
<%= form_with model: @article do |f| %>
  <%# CSRF token added here %>
<% end %>
```

### Timing Attacks

**Problem**:
```ruby
# VULNERABLE - reveals info through timing
def valid_api_key?(provided_key)
  provided_key == stored_key
end
```

**Solution**:
```ruby
# SAFE - constant-time comparison
def valid_api_key?(provided_key)
  ActiveSupport::SecurityUtils.secure_compare(provided_key, stored_key)
end
```

### Insecure Direct Object References

**Problem**:
```ruby
# VULNERABLE - No authorization check
def show
  @article = Article.find(params[:id])
end
```

**Solution**:
```ruby
# SAFE - Authorize first
def show
  @article = Article.find(params[:id])
  authorize @article
end

# Or scope to current user
def show
  @article = current_user.articles.find(params[:id])
end
```

### Security Checklist

Before deploying:

✓ All secrets in credentials, not code
✓ SSL/HTTPS enforced in production
✓ Strong password requirements
✓ Rate limiting on auth endpoints
✓ CSRF protection enabled
✓ Mass assignment protected (strong params)
✓ Authorization checks on all actions
✓ Sensitive data encrypted (Lockbox)
✓ Brakeman scan passes
✓ bundler-audit passes
✓ Security headers configured
✓ Input validation on all user data
✓ Output escaping by default
