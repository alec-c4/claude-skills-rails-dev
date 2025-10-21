# Testing Requirements Reference

## Table of Contents
- [Test Coverage Expectations](#test-coverage-expectations)
- [Testing Tools](#testing-tools)
- [Test Organization](#test-organization)
- [Unit Tests](#unit-tests)
- [Integration Tests](#integration-tests)
- [System Tests](#system-tests)
- [Component Tests](#component-tests)

## Test Coverage Expectations

### Coverage Targets by Layer

| Layer | Coverage Target | What to Test |
|-------|----------------|--------------|
| Models | 100% | All validations, scopes, methods |
| Interactions | 100% | All business logic paths |
| Components | 90%+ | Rendering, conditional logic |
| Controllers | 80%+ | Success/failure paths, auth |
| System Tests | Key flows | Critical user journeys |

### What to Test

**✓ Always Test**:
- Model validations and associations
- Business logic in interactions
- Component rendering and variants
- Authorization rules
- API endpoints
- Background jobs
- Critical user workflows

**✗ Don't Test**:
- Framework code (Rails itself)
- Third-party gems (unless integration)
- Private methods in isolation (test through public interface)
- Obvious getters/setters

## Testing Tools

### Core Tools

```ruby
# Gemfile
group :development, :test do
  gem "rspec-rails"        # Testing framework
  gem "factory_bot_rails"  # Test data factories
  gem "faker"              # Fake data generation
  gem "shoulda-matchers"   # One-liner matchers (optional)
end

group :test do
  gem "capybara"           # System test DSL
  gem "selenium-webdriver" # Browser automation
  gem "webdrivers"         # Driver management
  gem "database_cleaner-active_record"  # Test DB cleanup
  gem "simplecov", require: false       # Coverage reporting
end
```

### RSpec Configuration

```ruby
# spec/rails_helper.rb
require 'spec_helper'
require 'rspec/rails'
require 'capybara/rspec'

# Configure Database Cleaner
RSpec.configure do |config|
  config.use_transactional_fixtures = false
  
  config.before(:suite) do
    DatabaseCleaner.clean_with(:truncation)
  end
  
  config.before(:each) do
    DatabaseCleaner.strategy = :transaction
  end
  
  config.before(:each, type: :system) do
    DatabaseCleaner.strategy = :truncation
  end
  
  config.before(:each) do
    DatabaseCleaner.start
  end
  
  config.after(:each) do
    DatabaseCleaner.clean
  end
end

# Configure FactoryBot
RSpec.configure do |config|
  config.include FactoryBot::Syntax::Methods
end

# Configure Capybara for system tests
Capybara.default_max_wait_time = 5
Capybara.javascript_driver = :selenium_chrome_headless
```

### Coverage Configuration

```ruby
# spec/spec_helper.rb
if ENV['COVERAGE']
  require 'simplecov'
  SimpleCov.start 'rails' do
    add_filter '/spec/'
    add_filter '/config/'
    add_filter '/vendor/'
    
    add_group 'Models', 'app/models'
    add_group 'Controllers', 'app/controllers'
    add_group 'Interactions', 'app/interactions'
    add_group 'Components', 'app/components'
    add_group 'Jobs', 'app/jobs'
    
    minimum_coverage 80
  end
end
```

## Test Organization

### Directory Structure

```
spec/
├── models/              # Model unit tests
│   ├── user_spec.rb
│   └── article_spec.rb
├── interactions/        # Business logic tests
│   └── users/
│       └── create_spec.rb
├── components/          # ViewComponent tests
│   └── button_component_spec.rb
├── requests/            # API/Controller integration tests
│   └── articles_spec.rb
├── system/              # End-to-end browser tests
│   └── article_management_spec.rb
├── jobs/                # Background job tests
│   └── email_notification_job_spec.rb
├── policies/            # Authorization tests
│   └── article_policy_spec.rb
├── mailers/             # Mailer tests
│   └── user_mailer_spec.rb
├── factories/           # FactoryBot factories
│   ├── users.rb
│   └── articles.rb
└── support/             # Test helpers
    ├── auth_helpers.rb
    └── shared_contexts.rb
```

### Naming Conventions

```ruby
# spec/models/user_spec.rb
RSpec.describe User, type: :model do
  # ...
end

# spec/interactions/users/create_spec.rb
RSpec.describe Users::Create, type: :interaction do
  # ...
end

# spec/components/button_component_spec.rb
RSpec.describe ButtonComponent, type: :component do
  # ...
end
```

## Unit Tests

### Model Testing

```ruby
# spec/models/article_spec.rb
RSpec.describe Article, type: :model do
  describe "associations" do
    it { is_expected.to belong_to(:user) }
    it { is_expected.to have_many(:comments).dependent(:destroy) }
  end
  
  describe "validations" do
    subject { build(:article) }
    
    it { is_expected.to validate_presence_of(:title) }
    it { is_expected.to validate_presence_of(:body) }
    it { is_expected.to validate_uniqueness_of(:slug).case_insensitive }
    it { is_expected.to validate_length_of(:title).is_at_most(255) }
  end
  
  describe "scopes" do
    let!(:published_article) { create(:article, :published) }
    let!(:draft_article) { create(:article, :draft) }
    
    describe ".published" do
      it "returns only published articles" do
        expect(Article.published).to contain_exactly(published_article)
      end
    end
  end
  
  describe "#publish!" do
    let(:article) { create(:article, :draft) }
    
    it "sets published_at timestamp" do
      expect { article.publish! }
        .to change(article, :published_at).from(nil)
    end
    
    it "changes status to published" do
      expect { article.publish! }
        .to change(article, :status).from("draft").to("published")
    end
  end
end
```

### Interaction Testing

```ruby
# spec/interactions/users/create_spec.rb
RSpec.describe Users::Create, type: :interaction do
  describe ".run" do
    let(:valid_params) do
      {
        email: "user@example.com",
        name: "John Doe",
        password: "SecurePassword123"
      }
    end
    
    context "with valid parameters" do
      it "creates a new user" do
        expect { described_class.run(valid_params) }
          .to change(User, :count).by(1)
      end
      
      it "returns valid outcome" do
        outcome = described_class.run(valid_params)
        expect(outcome).to be_valid
      end
      
      it "returns created user" do
        outcome = described_class.run(valid_params)
        expect(outcome.result).to be_a(User)
        expect(outcome.result.email).to eq("user@example.com")
      end
      
      it "sends welcome email" do
        expect { described_class.run(valid_params) }
          .to have_enqueued_mail(UserMailer, :welcome)
      end
    end
    
    context "with invalid parameters" do
      it "returns invalid outcome for missing email" do
        outcome = described_class.run(valid_params.merge(email: nil))
        expect(outcome).not_to be_valid
        expect(outcome.errors[:email]).to be_present
      end
      
      it "does not create user" do
        expect { described_class.run(valid_params.merge(email: nil)) }
          .not_to change(User, :count)
      end
    end
    
    context "when email already exists" do
      before { create(:user, email: "user@example.com") }
      
      it "returns invalid outcome" do
        outcome = described_class.run(valid_params)
        expect(outcome).not_to be_valid
      end
    end
  end
end
```

## Integration Tests

### Request Specs (Controllers)

```ruby
# spec/requests/articles_spec.rb
RSpec.describe "Articles", type: :request do
  let(:user) { create(:user) }
  
  before { sign_in user }
  
  describe "GET /articles" do
    let!(:articles) { create_list(:article, 3, user: user) }
    
    it "returns success response" do
      get articles_path
      expect(response).to have_http_status(:success)
    end
    
    it "renders articles" do
      get articles_path
      expect(response.body).to include(articles.first.title)
    end
  end
  
  describe "POST /articles" do
    let(:valid_attributes) do
      {
        title: "New Article",
        body: "Article content"
      }
    end
    
    context "with valid parameters" do
      it "creates new article" do
        expect {
          post articles_path, params: { article: valid_attributes }
        }.to change(Article, :count).by(1)
      end
      
      it "redirects to article" do
        post articles_path, params: { article: valid_attributes }
        expect(response).to redirect_to(article_path(Article.last))
      end
      
      it "sets flash notice" do
        post articles_path, params: { article: valid_attributes }
        expect(flash[:notice]).to match(/successfully created/i)
      end
    end
    
    context "with invalid parameters" do
      it "does not create article" do
        expect {
          post articles_path, params: { article: { title: "" } }
        }.not_to change(Article, :count)
      end
      
      it "renders new template" do
        post articles_path, params: { article: { title: "" } }
        expect(response).to have_http_status(:unprocessable_entity)
      end
    end
  end
  
  describe "PATCH /articles/:id" do
    let(:article) { create(:article, user: user) }
    
    context "with valid parameters" do
      it "updates article" do
        patch article_path(article), params: {
          article: { title: "Updated Title" }
        }
        
        article.reload
        expect(article.title).to eq("Updated Title")
      end
    end
  end
  
  describe "DELETE /articles/:id" do
    let!(:article) { create(:article, user: user) }
    
    it "destroys article" do
      expect {
        delete article_path(article)
      }.to change(Article, :count).by(-1)
    end
    
    it "redirects to articles index" do
      delete article_path(article)
      expect(response).to redirect_to(articles_path)
    end
  end
  
  describe "authorization" do
    let(:other_user) { create(:user) }
    let(:article) { create(:article, user: other_user) }
    
    it "prevents editing other user's article" do
      patch article_path(article), params: {
        article: { title: "Hacked" }
      }
      
      expect(response).to have_http_status(:forbidden)
    end
  end
end
```

## System Tests

### End-to-End Testing

```ruby
# spec/system/article_management_spec.rb
RSpec.describe "Article Management", type: :system do
  let(:user) { create(:user) }
  
  before do
    driven_by(:selenium_chrome_headless)
    login_as(user)
  end
  
  describe "Creating an article" do
    it "allows user to create article" do
      visit articles_path
      click_on "New Article"
      
      fill_in "Title", with: "My New Article"
      fill_in "Body", with: "This is the article content"
      
      click_on "Create Article"
      
      expect(page).to have_content("Article was successfully created")
      expect(page).to have_content("My New Article")
      expect(page).to have_content("This is the article content")
    end
    
    it "shows validation errors" do
      visit new_article_path
      
      fill_in "Title", with: ""
      click_on "Create Article"
      
      expect(page).to have_content("Title can't be blank")
    end
  end
  
  describe "Editing an article" do
    let!(:article) { create(:article, user: user, title: "Original Title") }
    
    it "allows user to edit their article" do
      visit article_path(article)
      click_on "Edit"
      
      fill_in "Title", with: "Updated Title"
      click_on "Update Article"
      
      expect(page).to have_content("Article was successfully updated")
      expect(page).to have_content("Updated Title")
      expect(page).not_to have_content("Original Title")
    end
  end
  
  describe "Deleting an article" do
    let!(:article) { create(:article, user: user) }
    
    it "allows user to delete their article", js: true do
      visit article_path(article)
      
      accept_confirm do
        click_on "Delete"
      end
      
      expect(page).to have_content("Article was successfully deleted")
      expect(page).not_to have_content(article.title)
    end
  end
  
  describe "Searching articles" do
    let!(:ruby_article) { create(:article, title: "Ruby Tips", user: user) }
    let!(:rails_article) { create(:article, title: "Rails Guide", user: user) }
    
    it "filters articles by search query" do
      visit articles_path
      
      fill_in "Search", with: "Ruby"
      click_on "Search"
      
      expect(page).to have_content("Ruby Tips")
      expect(page).not_to have_content("Rails Guide")
    end
  end
end
```

## Component Tests

### ViewComponent Testing

```ruby
# spec/components/button_component_spec.rb
RSpec.describe ButtonComponent, type: :component do
  describe "rendering" do
    it "renders button with text" do
      render_inline(ButtonComponent.new(text: "Click me"))
      
      expect(page).to have_button("Click me")
    end
    
    it "applies variant classes" do
      render_inline(ButtonComponent.new(text: "Save", variant: :primary))
      
      expect(page).to have_css("button.btn-primary")
    end
    
    it "renders secondary variant" do
      render_inline(ButtonComponent.new(text: "Cancel", variant: :secondary))
      
      expect(page).to have_css("button.btn-secondary")
    end
    
    it "renders danger variant" do
      render_inline(ButtonComponent.new(text: "Delete", variant: :danger))
      
      expect(page).to have_css("button.btn-danger")
    end
  end
  
  describe "with custom attributes" do
    it "adds data attributes" do
      render_inline(ButtonComponent.new(
        text: "Submit",
        data: { turbo_confirm: "Are you sure?" }
      ))
      
      expect(page).to have_css('button[data-turbo-confirm="Are you sure?"]')
    end
    
    it "adds custom classes" do
      render_inline(ButtonComponent.new(
        text: "Button",
        class: "custom-class"
      ))
      
      expect(page).to have_css("button.custom-class")
    end
  end
  
  describe "with icon" do
    it "renders icon with text" do
      render_inline(ButtonComponent.new(
        text: "Save",
        icon: "check"
      ))
      
      expect(page).to have_css("svg")
      expect(page).to have_content("Save")
    end
  end
end
```

### Testing with Lookbook Previews

```ruby
# spec/components/previews/button_component_preview.rb
class ButtonComponentPreview < ViewComponent::Preview
  # @param text
  # @param variant select { choices: [primary, secondary, danger] }
  def playground(text: "Click me", variant: :primary)
    render ButtonComponent.new(text: text, variant: variant.to_sym)
  end
  
  def primary
    render ButtonComponent.new(text: "Primary Button", variant: :primary)
  end
  
  def secondary
    render ButtonComponent.new(text: "Secondary Button", variant: :secondary)
  end
  
  def danger
    render ButtonComponent.new(text: "Delete", variant: :danger)
  end
  
  def with_icon
    render ButtonComponent.new(text: "Save", icon: "check", variant: :primary)
  end
end
```

## FactoryBot Factories

### Factory Best Practices

```ruby
# spec/factories/users.rb
FactoryBot.define do
  factory :user do
    sequence(:email) { |n| "user#{n}@example.com" }
    name { Faker::Name.name }
    password { "SecurePassword123" }
    role { "user" }
    confirmed_at { Time.current }
    
    trait :admin do
      role { "admin" }
    end
    
    trait :unconfirmed do
      confirmed_at { nil }
    end
    
    trait :with_articles do
      transient do
        articles_count { 3 }
      end
      
      after(:create) do |user, evaluator|
        create_list(:article, evaluator.articles_count, user: user)
      end
    end
  end
end

# spec/factories/articles.rb
FactoryBot.define do
  factory :article do
    association :user
    title { Faker::Lorem.sentence }
    body { Faker::Lorem.paragraphs(number: 3).join("\n\n") }
    status { "draft" }
    
    trait :published do
      status { "published" }
      published_at { Time.current }
    end
    
    trait :with_comments do
      transient do
        comments_count { 5 }
      end
      
      after(:create) do |article, evaluator|
        create_list(:comment, evaluator.comments_count, article: article)
      end
    end
  end
end
```

### Using Factories

```ruby
# Create instance without saving
user = build(:user)

# Create and save instance
user = create(:user)

# Create with overrides
user = create(:user, email: "specific@example.com", role: "admin")

# Create with traits
admin = create(:user, :admin)
unconfirmed_user = create(:user, :unconfirmed)
user_with_articles = create(:user, :with_articles, articles_count: 5)

# Create list
users = create_list(:user, 10)
admins = create_list(:user, 3, :admin)

# Build attributes hash (for params)
user_attributes = attributes_for(:user)
```
