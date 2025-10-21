# Rails API Reference Guide

Comprehensive guide for building production-ready REST APIs with Rails.

## Table of Contents

- [API Versioning Strategies](#api-versioning-strategies)
- [Advanced Authentication](#advanced-authentication)
- [API Documentation with RSwag](#api-documentation-with-rswag)
- [Advanced Filtering & Sorting](#advanced-filtering--sorting)
- [Serialization Options](#serialization-options)
- [Performance & Caching](#performance--caching)
- [CORS Configuration](#cors-configuration)
- [Complete Testing Guide](#complete-testing-guide)

---

## API Versioning Strategies

### URL Versioning (Recommended)

**✅ CORRECT - This is the standard approach**

```ruby
# config/routes.rb
namespace :api do
  namespace :v1 do
    resources :articles
    resources :users
    resources :comments
  end

  # When breaking changes needed, add v2
  namespace :v2 do
    resources :articles  # Can have different structure
    resources :users
  end
end

# Routes generated:
# GET /api/v1/articles
# GET /api/v2/articles
```

**Controller organization:**
```ruby
# app/controllers/api/v1/articles_controller.rb
class Api::V1::ArticlesController < Api::V1::BaseController
  def index
    # v1 implementation
  end
end

# app/controllers/api/v2/articles_controller.rb
class Api::V2::ArticlesController < Api::V2::BaseController
  def index
    # v2 implementation with breaking changes
  end
end
```

### Header Versioning (Alternative)

```ruby
# config/routes.rb
namespace :api do
  scope module: :v1, constraints: ApiVersion.new('v1', default: true) do
    resources :articles
  end

  scope module: :v2, constraints: ApiVersion.new('v2') do
    resources :articles
  end
end

# lib/api_version.rb
class ApiVersion
  def initialize(version, default = false)
    @version = version
    @default = default
  end

  def matches?(request)
    @default || check_headers(request.headers) || check_accept(request.headers)
  end

  private

  def check_headers(headers)
    headers['Accept-Version'] == @version
  end

  def check_accept(headers)
    accept = headers['Accept']
    accept && accept.include?("application/vnd.myapp.#{@version}+json")
  end
end
```

**Client usage:**
```bash
# Using Accept-Version header
curl -H "Accept-Version: v2" https://api.example.com/articles

# Using custom media type
curl -H "Accept: application/vnd.myapp.v2+json" https://api.example.com/articles
```

### Deprecation Strategy

```ruby
# app/controllers/api/v1/base_controller.rb
class Api::V1::BaseController < Api::BaseController
  before_action :add_deprecation_headers

  private

  def add_deprecation_headers
    response.headers['X-API-Deprecation'] = 'This API version is deprecated'
    response.headers['X-API-Sunset-Date'] = '2025-12-31'
    response.headers['X-API-Migration-Guide'] = 'https://docs.example.com/api/v1-to-v2'
  end
end
```

---

## Advanced Authentication

### JWT with Refresh Tokens

```ruby
# app/controllers/api/v1/auth_controller.rb
class Api::V1::AuthController < Api::BaseController
  skip_before_action :authenticate, only: [:login, :refresh]

  def login
    user = User.find_by(email: params[:email])

    if user&.authenticate(params[:password])
      access_token = encode_token(user_id: user.id, exp: 15.minutes.from_now.to_i)
      refresh_token = encode_refresh_token(user_id: user.id)

      # Store refresh token
      user.update(refresh_token: refresh_token)

      render json: {
        access_token: access_token,
        refresh_token: refresh_token,
        token_type: 'Bearer',
        expires_in: 900,  # 15 minutes
        user: UserSerializer.new(user)
      }
    else
      render json: { error: 'Invalid credentials' }, status: :unauthorized
    end
  end

  def refresh
    refresh_token = params[:refresh_token]
    decoded = JWT.decode(refresh_token, refresh_secret)[0]
    user = User.find(decoded['user_id'])

    if user.refresh_token == refresh_token
      access_token = encode_token(user_id: user.id, exp: 15.minutes.from_now.to_i)

      render json: {
        access_token: access_token,
        token_type: 'Bearer',
        expires_in: 900
      }
    else
      render json: { error: 'Invalid refresh token' }, status: :unauthorized
    end
  rescue JWT::DecodeError, ActiveRecord::RecordNotFound
    render json: { error: 'Invalid refresh token' }, status: :unauthorized
  end

  def logout
    current_user.update(refresh_token: nil)
    head :no_content
  end

  private

  def encode_token(payload)
    JWT.encode(payload, Rails.application.credentials.secret_key_base)
  end

  def encode_refresh_token(payload)
    payload[:exp] = 7.days.from_now.to_i
    JWT.encode(payload, refresh_secret)
  end

  def refresh_secret
    Rails.application.credentials.dig(:jwt, :refresh_secret)
  end
end
```

### OAuth2 Provider (Doorkeeper)

```ruby
# Gemfile
gem 'doorkeeper'

# Configure
rails generate doorkeeper:install

# config/initializers/doorkeeper.rb
Doorkeeper.configure do
  orm :active_record

  resource_owner_authenticator do
    User.find_by(id: session[:user_id]) || redirect_to(login_url)
  end

  grant_flows %w[authorization_code client_credentials]

  access_token_expires_in 2.hours
  use_refresh_token
end

# Protect API with OAuth
class Api::V1::BaseController < ActionController::API
  before_action :doorkeeper_authorize!

  private

  def current_user
    @current_user ||= User.find(doorkeeper_token.resource_owner_id) if doorkeeper_token
  end
end
```

### API Key with Scopes

```ruby
# app/models/api_key.rb
class ApiKey < ApplicationRecord
  belongs_to :user
  before_create :generate_key

  SCOPES = %w[read write admin].freeze

  scope :active, -> { where(revoked_at: nil) }

  def revoke!
    update(revoked_at: Time.current)
  end

  def can?(scope)
    return false if revoked?
    scopes.include?(scope.to_s) || scopes.include?('admin')
  end

  def revoked?
    revoked_at.present?
  end

  private

  def generate_key
    self.token = SecureRandom.hex(32)
  end
end

# app/controllers/api/base_controller.rb
def authenticate
  api_key = request.headers['X-API-Key']
  return unauthorized unless api_key

  @api_key = ApiKey.active.find_by(token: api_key)
  return unauthorized unless @api_key

  @current_user = @api_key.user
end

def require_scope(scope)
  unauthorized unless @api_key&.can?(scope)
end

# Usage in controller
class Api::V1::ArticlesController < Api::BaseController
  before_action -> { require_scope(:write) }, only: [:create, :update, :destroy]

  def create
    # Only API keys with 'write' or 'admin' scope can access
  end
end
```

---

## API Documentation with RSwag

### Setup

```ruby
# Gemfile
group :development, :test do
  gem 'rswag'
end

bundle install
rails g rswag:install
```

### Complete API Spec

```ruby
# spec/requests/api/v1/articles_spec.rb
require 'swagger_helper'

RSpec.describe 'Articles API', type: :request do
  path '/api/v1/articles' do
    get 'Retrieves all articles' do
      tags 'Articles'
      produces 'application/json'
      security [Bearer: []]

      parameter name: :page, in: :query, type: :integer, required: false, description: 'Page number'
      parameter name: :per_page, in: :query, type: :integer, required: false, description: 'Items per page'
      parameter name: :status, in: :query, type: :string, enum: ['draft', 'published'], required: false
      parameter name: :author_id, in: :query, type: :integer, required: false

      response '200', 'articles found' do
        schema type: :object,
          properties: {
            data: {
              type: :array,
              items: {
                type: :object,
                properties: {
                  id: { type: :integer },
                  type: { type: :string },
                  attributes: {
                    type: :object,
                    properties: {
                      title: { type: :string },
                      excerpt: { type: :string },
                      published_at: { type: :string, format: :datetime }
                    }
                  },
                  relationships: {
                    type: :object,
                    properties: {
                      author: {
                        type: :object,
                        properties: {
                          data: {
                            type: :object,
                            properties: {
                              id: { type: :integer },
                              type: { type: :string }
                            }
                          }
                        }
                      }
                    }
                  }
                }
              }
            },
            meta: {
              type: :object,
              properties: {
                current_page: { type: :integer },
                total_pages: { type: :integer },
                total_count: { type: :integer }
              }
            }
          }

        let(:Authorization) { "Bearer #{token}" }

        run_test! do |response|
          data = JSON.parse(response.body)
          expect(data['data']).to be_an(Array)
        end
      end

      response '401', 'unauthorized' do
        run_test!
      end
    end

    post 'Creates an article' do
      tags 'Articles'
      consumes 'application/json'
      produces 'application/json'
      security [Bearer: []]

      parameter name: :article, in: :body, schema: {
        type: :object,
        properties: {
          article: {
            type: :object,
            properties: {
              title: { type: :string },
              body: { type: :string },
              published: { type: :boolean }
            },
            required: ['title', 'body']
          }
        }
      }

      response '201', 'article created' do
        let(:Authorization) { "Bearer #{token}" }
        let(:article) { { article: { title: 'Test', body: 'Content' } } }

        run_test! do |response|
          data = JSON.parse(response.body)
          expect(data['data']['attributes']['title']).to eq('Test')
        end
      end

      response '422', 'invalid request' do
        let(:Authorization) { "Bearer #{token}" }
        let(:article) { { article: { title: '' } } }

        run_test!
      end
    end
  end

  path '/api/v1/articles/{id}' do
    parameter name: :id, in: :path, type: :integer

    get 'Retrieves an article' do
      tags 'Articles'
      produces 'application/json'
      security [Bearer: []]

      response '200', 'article found' do
        let(:Authorization) { "Bearer #{token}" }
        let(:id) { create(:article).id }

        run_test!
      end

      response '404', 'article not found' do
        let(:Authorization) { "Bearer #{token}" }
        let(:id) { 999999 }

        run_test!
      end
    end
  end
end
```

### Generate Swagger Docs

```bash
# Generate swagger documentation
SWAGGER_DRY_RUN=0 RAILS_ENV=test bundle exec rake rswag:specs:swaggerize

# Access docs at
http://localhost:3000/api-docs
```

---

## Advanced Filtering & Sorting

### Complex Filter System

```ruby
# app/controllers/api/v1/articles_controller.rb
class Api::V1::ArticlesController < Api::BaseController
  def index
    @articles = Article.all
    @articles = ArticleFilter.new(@articles, filter_params).call
    @articles = @articles.page(params[:page]).per(params[:per_page] || 20)

    render json: @articles, meta: pagination_meta(@articles)
  end

  private

  def filter_params
    params.permit(:status, :author_id, :category_id, :from_date, :to_date, :search, :sort, :order, tag_ids: [])
  end
end

# app/services/article_filter.rb
class ArticleFilter
  def initialize(scope, params)
    @scope = scope
    @params = params
  end

  def call
    filter_by_status
    filter_by_author
    filter_by_category
    filter_by_date_range
    filter_by_tags
    search
    sort
    @scope
  end

  private

  def filter_by_status
    return unless @params[:status].present?
    @scope = @scope.where(status: @params[:status])
  end

  def filter_by_author
    return unless @params[:author_id].present?
    @scope = @scope.where(author_id: @params[:author_id])
  end

  def filter_by_category
    return unless @params[:category_id].present?
    @scope = @scope.where(category_id: @params[:category_id])
  end

  def filter_by_date_range
    if @params[:from_date].present?
      @scope = @scope.where('created_at >= ?', @params[:from_date])
    end

    if @params[:to_date].present?
      @scope = @scope.where('created_at <= ?', @params[:to_date])
    end
  end

  def filter_by_tags
    return unless @params[:tag_ids].present?
    @scope = @scope.joins(:tags).where(tags: { id: @params[:tag_ids] }).distinct
  end

  def search
    return unless @params[:search].present?
    @scope = @scope.where('title ILIKE ? OR body ILIKE ?', "%#{@params[:search]}%", "%#{@params[:search]}%")
  end

  def sort
    column = @params[:sort]&.to_sym || :created_at
    direction = @params[:order]&.downcase == 'desc' ? :desc : :asc

    # Whitelist allowed columns
    allowed_columns = [:created_at, :updated_at, :title, :published_at]
    column = :created_at unless allowed_columns.include?(column)

    @scope = @scope.order(column => direction)
  end
end
```

**Usage:**
```bash
GET /api/v1/articles?status=published&author_id=1&tag_ids[]=2&tag_ids[]=3&sort=created_at&order=desc&search=rails
```

---

## Serialization Options

### Using Blueprinter (Faster Alternative)

```ruby
# Gemfile
gem 'blueprinter'

# app/blueprints/article_blueprint.rb
class ArticleBlueprint < Blueprinter::Base
  identifier :id

  fields :title, :excerpt, :published_at, :created_at

  association :author, blueprint: UserBlueprint

  view :detailed do
    fields :body, :updated_at
    association :comments, blueprint: CommentBlueprint
    association :tags, blueprint: TagBlueprint
  end

  field :excerpt do |article|
    article.body.truncate(200)
  end
end

# Controller
def index
  render json: ArticleBlueprint.render(@articles)
end

def show
  render json: ArticleBlueprint.render(@article, view: :detailed)
end
```

### JSONAPI Serializer (Fastest)

```ruby
# Gemfile
gem 'jsonapi-serializer'

# app/serializers/article_serializer.rb
class ArticleSerializer
  include JSONAPI::Serializer

  attributes :title, :excerpt, :published_at, :created_at

  belongs_to :author, serializer: UserSerializer
  has_many :tags, serializer: TagSerializer

  attribute :excerpt do |article|
    article.body.truncate(200)
  end

  # Conditional attributes
  attribute :draft_notes, if: Proc.new { |article, params|
    params[:current_user]&.admin?
  }
end

# Controller
def index
  options = { params: { current_user: current_user } }
  render json: ArticleSerializer.new(@articles, options).serializable_hash
end
```

---

## Performance & Caching

### Fragment Caching with Jbuilder

```ruby
# app/views/api/v1/articles/index.json.jbuilder
json.cache! ['v1', @articles] do
  json.data do
    json.array! @articles do |article|
      json.cache! ['v1', article] do
        json.id article.id
        json.type 'articles'
        json.attributes do
          json.title article.title
          json.excerpt article.excerpt
        end
      end
    end
  end

  json.meta do
    json.current_page @articles.current_page
    json.total_pages @articles.total_pages
  end
end
```

### Conditional GET (ETags & Last-Modified)

```ruby
def show
  @article = Article.find(params[:id])

  # Automatically returns 304 Not Modified if content unchanged
  fresh_when(
    etag: @article,
    last_modified: @article.updated_at,
    public: true
  )
end

# Or with explicit check
def show
  @article = Article.find(params[:id])

  if stale?(etag: @article, last_modified: @article.updated_at, public: true)
    render json: @article
  end
  # Rails automatically sends 304 if not stale
end
```

### Low-Level Caching

```ruby
def index
  @articles = Rails.cache.fetch(
    ['articles', 'v1', params[:page], params[:filters]],
    expires_in: 5.minutes
  ) do
    Article.published
      .includes(:author, :tags)
      .page(params[:page])
      .per(20)
      .to_a
  end

  render json: @articles
end
```

---

## CORS Configuration

```ruby
# Gemfile
gem 'rack-cors'

# config/initializers/cors.rb
Rails.application.config.middleware.insert_before 0, Rack::Cors do
  # Allow all origins in development
  allow do
    origins '*'
    resource '/api/*',
      headers: :any,
      methods: [:get, :post, :put, :patch, :delete, :options, :head],
      credentials: false
  end if Rails.env.development?

  # Restrict in production
  allow do
    origins 'https://app.example.com', 'https://admin.example.com'
    resource '/api/*',
      headers: :any,
      methods: [:get, :post, :put, :patch, :delete, :options, :head],
      credentials: true,
      max_age: 86400  # Cache preflight requests for 24 hours
  end if Rails.env.production?
end
```

---

## Complete Testing Guide

### Request Spec Helpers

```ruby
# spec/support/api_helpers.rb
module ApiHelpers
  def json_response
    JSON.parse(response.body)
  end

  def auth_headers(user)
    token = JWT.encode({ user_id: user.id }, Rails.application.credentials.secret_key_base)
    { 'Authorization' => "Bearer #{token}" }
  end

  def expect_json_types(data, types)
    types.each do |key, type|
      expect(data[key.to_s]).to be_a(type)
    end
  end
end

RSpec.configure do |config|
  config.include ApiHelpers, type: :request
end
```

### Comprehensive Test Suite

```ruby
# spec/requests/api/v1/articles_spec.rb
RSpec.describe 'Articles API', type: :request do
  let(:user) { create(:user) }
  let(:headers) { auth_headers(user) }

  describe 'GET /api/v1/articles' do
    context 'without authentication' do
      it 'returns 401' do
        get '/api/v1/articles'
        expect(response).to have_http_status(:unauthorized)
      end
    end

    context 'with authentication' do
      it 'returns articles' do
        create_list(:article, 3, :published)

        get '/api/v1/articles', headers: headers

        expect(response).to have_http_status(:ok)
        expect(json_response['data'].size).to eq(3)
      end

      it 'paginates results' do
        create_list(:article, 25, :published)

        get '/api/v1/articles', params: { page: 2, per_page: 10 }, headers: headers

        expect(json_response['data'].size).to eq(10)
        expect(json_response['meta']['current_page']).to eq(2)
      end

      it 'filters by status' do
        published = create(:article, :published)
        draft = create(:article, :draft)

        get '/api/v1/articles', params: { status: 'published' }, headers: headers

        ids = json_response['data'].map { |a| a['id'] }
        expect(ids).to include(published.id)
        expect(ids).not_to include(draft.id)
      end

      it 'searches articles' do
        rails_article = create(:article, title: 'Rails Guide')
        react_article = create(:article, title: 'React Tutorial')

        get '/api/v1/articles', params: { search: 'Rails' }, headers: headers

        titles = json_response['data'].map { |a| a['attributes']['title'] }
        expect(titles).to include('Rails Guide')
        expect(titles).not_to include('React Tutorial')
      end

      it 'sorts articles' do
        old = create(:article, created_at: 2.days.ago)
        new = create(:article, created_at: 1.day.ago)

        get '/api/v1/articles', params: { sort: 'created_at', order: 'desc' }, headers: headers

        ids = json_response['data'].map { |a| a['id'] }
        expect(ids.first).to eq(new.id)
        expect(ids.last).to eq(old.id)
      end

      it 'avoids N+1 queries', :n_plus_one do
        create_list(:article, 3, :with_author, :with_tags)

        expect {
          get '/api/v1/articles', headers: headers
        }.to perform_constant_number_of_queries
      end

      it 'returns proper JSON structure' do
        article = create(:article, :published)

        get '/api/v1/articles', headers: headers

        data = json_response['data'].first
        expect_json_types(data, { id: Integer, type: String })
        expect(data['attributes']).to have_key('title')
        expect(data['relationships']).to have_key('author')
      end
    end
  end

  describe 'POST /api/v1/articles' do
    let(:valid_params) do
      { article: { title: 'Test Article', body: 'Test content' } }
    end

    it 'creates article' do
      expect {
        post '/api/v1/articles', params: valid_params, headers: headers
      }.to change(Article, :count).by(1)

      expect(response).to have_http_status(:created)
      expect(response.headers['Location']).to be_present
    end

    it 'returns errors for invalid data' do
      post '/api/v1/articles', params: { article: { title: '' } }, headers: headers

      expect(response).to have_http_status(:unprocessable_entity)
      expect(json_response['errors']).to be_an(Array)
    end

    it 'associates article with current user' do
      post '/api/v1/articles', params: valid_params, headers: headers

      article = Article.last
      expect(article.author).to eq(user)
    end
  end

  describe 'PATCH /api/v1/articles/:id' do
    let(:article) { create(:article, author: user) }

    it 'updates article' do
      patch "/api/v1/articles/#{article.id}",
        params: { article: { title: 'Updated' } },
        headers: headers

      expect(response).to have_http_status(:ok)
      expect(article.reload.title).to eq('Updated')
    end

    it 'prevents unauthorized updates' do
      other_article = create(:article)

      patch "/api/v1/articles/#{other_article.id}",
        params: { article: { title: 'Hacked' } },
        headers: headers

      expect(response).to have_http_status(:forbidden)
    end
  end

  describe 'DELETE /api/v1/articles/:id' do
    let(:article) { create(:article, author: user) }

    it 'deletes article' do
      article_id = article.id

      expect {
        delete "/api/v1/articles/#{article_id}", headers: headers
      }.to change(Article, :count).by(-1)

      expect(response).to have_http_status(:no_content)
    end
  end

  describe 'caching' do
    let(:article) { create(:article) }

    it 'returns ETag header' do
      get "/api/v1/articles/#{article.id}", headers: headers

      expect(response.headers['ETag']).to be_present
    end

    it 'returns 304 when not modified' do
      get "/api/v1/articles/#{article.id}", headers: headers
      etag = response.headers['ETag']

      get "/api/v1/articles/#{article.id}", headers: headers.merge('If-None-Match' => etag)

      expect(response).to have_http_status(:not_modified)
    end
  end
end
```

---

## Best Practices Summary

### Security

1. **Always authenticate** - No public write access
2. **Use HTTPS only** - Enforce in production
3. **Rate limit** - Prevent abuse
4. **Validate input** - Whitelist parameters
5. **Sanitize output** - Never expose sensitive data
6. **Use scopes** - Limit API key permissions
7. **Log everything** - Track API usage
8. **Rotate secrets** - Invalidate old tokens

### Performance

1. **Eager load associations** - Avoid N+1
2. **Paginate everything** - Never return all records
3. **Cache aggressively** - Use ETags, fragment caching
4. **Index database** - Optimize queries
5. **Use background jobs** - For heavy operations
6. **Monitor performance** - Track slow endpoints
7. **Use CDN** - For static API responses
8. **Compress responses** - Enable gzip

### API Design

1. **Version from day one** - `/api/v1/` mandatory
2. **Use RESTful conventions** - Standard HTTP verbs
3. **Return proper status codes** - 200, 201, 404, 422, 500
4. **Consistent responses** - Same format everywhere
5. **Include metadata** - Pagination info, etc.
6. **Document everything** - OpenAPI/Swagger
7. **Support filtering** - Make data queryable
8. **Handle errors gracefully** - Clear error messages

---

**Remember**: A production-ready API is secure, performant, well-documented, and versioned from the start. Never skip versioning—it's not optional.
