# Business Logic Reference

## Table of Contents
- [Interactions (ActiveInteraction)](#interactions-activeinteraction)
- [State Machines (AASM)](#state-machines-aasm)
- [Decorators (ActiveDecorator)](#decorators-activedecorator)
- [Service Objects vs Interactions](#service-objects-vs-interactions)

## Interactions (ActiveInteraction)

Use ActiveInteraction for business logic instead of service objects.

### Why Interactions?

**Advantages over service objects**:
- ✓ Built-in type checking
- ✓ Automatic validation
- ✓ Explicit contracts (inputs/outputs)
- ✓ Composable
- ✓ Testable
- ✓ Self-documenting

### Setup

```ruby
# Gemfile
gem "active_interaction", "~> 5.3"
```

### Basic Interaction

```ruby
# app/interactions/users/create.rb
module Users
  class Create < ActiveInteraction::Base
    # Define inputs with types
    string :email
    string :name
    string :password, default: nil
    boolean :send_welcome_email, default: true
    
    # Validations
    validates :email, presence: true, format: { with: URI::MailTo::EMAIL_REGEXP }
    validates :name, presence: true
    validates :password, length: { minimum: 12 }, allow_nil: true
    
    # Main logic
    def execute
      user = User.create!(
        email: email,
        name: name,
        password: password || generate_password
      )
      
      UserMailer.welcome(user).deliver_later if send_welcome_email
      
      user  # Return value becomes outcome.result
    end
    
    private
    
    def generate_password
      SecureRandom.alphanumeric(32)
    end
  end
end
```

### Running Interactions

```ruby
# In controller
def create
  outcome = Users::Create.run(
    email: params[:email],
    name: params[:name],
    password: params[:password]
  )
  
  if outcome.valid?
    @user = outcome.result
    redirect_to @user, notice: "User created"
  else
    @errors = outcome.errors
    render :new, status: :unprocessable_entity
  end
end

# With bang method (raises on error)
user = Users::Create.run!(
  email: "user@example.com",
  name: "John Doe"
)
```

### Input Types

```ruby
class MyInteraction < ActiveInteraction::Base
  # Primitives
  string :name
  integer :age
  float :price
  boolean :active
  symbol :status
  
  # Objects
  date :birthday
  time :created_at
  date_time :scheduled_at
  
  # Complex types
  array :tags
  hash :metadata
  
  # Model instances
  object :user, class: User
  
  # Arrays of specific types
  array :emails, default: [] do
    string
  end
  
  # Hashes with typed values
  hash :settings do
    boolean :notifications
    integer :max_items
  end
  
  # Optional (nilable)
  string :optional_field, default: nil
  
  # With custom default
  integer :count, default: 0
  array :items, default: -> { [] }
end
```

### Validation

```ruby
class Articles::Create < ActiveInteraction::Base
  string :title
  string :body
  object :author, class: User
  
  validates :title, presence: true, length: { minimum: 5, maximum: 255 }
  validates :body, presence: true, length: { minimum: 100 }
  validate :author_can_create_articles
  
  def execute
    Article.create!(
      title: title,
      body: body,
      author: author
    )
  end
  
  private
  
  def author_can_create_articles
    unless author.can_create_articles?
      errors.add(:author, "is not allowed to create articles")
    end
  end
end
```

### Composing Interactions

```ruby
# app/interactions/users/register.rb
module Users
  class Register < ActiveInteraction::Base
    string :email
    string :name
    string :password
    
    def execute
      # Compose other interactions
      user = compose(Users::Create,
        email: email,
        name: name,
        password: password
      )
      
      # compose raises if nested interaction fails
      # errors are merged automatically
      
      compose(Users::SendWelcomeEmail, user: user)
      
      user
    end
  end
end
```

### Error Handling

```ruby
class MyInteraction < ActiveInteraction::Base
  def execute
    # Manually add errors
    errors.add(:base, "Something went wrong")
    
    # Return early if needed
    return if errors.any?
    
    # Or raise to halt execution
    raise ActiveInteraction::InvalidInteractionError, "Custom error"
  end
end

# In controller
outcome = MyInteraction.run(params)

if outcome.valid?
  # Success
  result = outcome.result
else
  # Failed
  outcome.errors.full_messages
  outcome.errors[:field_name]
end
```

### Testing Interactions

```ruby
# spec/interactions/users/create_spec.rb
RSpec.describe Users::Create do
  describe ".run" do
    let(:valid_inputs) do
      {
        email: "user@example.com",
        name: "John Doe",
        password: "SecurePassword123"
      }
    end
    
    context "with valid inputs" do
      it "creates user" do
        expect { described_class.run(valid_inputs) }
          .to change(User, :count).by(1)
      end
      
      it "returns valid outcome" do
        outcome = described_class.run(valid_inputs)
        expect(outcome).to be_valid
      end
      
      it "returns created user" do
        outcome = described_class.run(valid_inputs)
        expect(outcome.result).to be_a(User)
        expect(outcome.result.email).to eq("user@example.com")
      end
      
      it "sends welcome email" do
        expect { described_class.run(valid_inputs) }
          .to have_enqueued_mail(UserMailer, :welcome)
      end
    end
    
    context "with invalid inputs" do
      it "returns invalid outcome for missing email" do
        outcome = described_class.run(valid_inputs.except(:email))
        expect(outcome).not_to be_valid
        expect(outcome.errors[:email]).to be_present
      end
      
      it "validates email format" do
        outcome = described_class.run(valid_inputs.merge(email: "invalid"))
        expect(outcome).not_to be_valid
        expect(outcome.errors[:email]).to include("is invalid")
      end
    end
  end
end
```

## State Machines (AASM)

Manage object states and transitions with AASM.

### Setup

```ruby
# Gemfile
gem "aasm"
```

### Basic State Machine

```ruby
# app/models/order.rb
class Order < ApplicationRecord
  include AASM
  
  aasm column: :status do
    state :pending, initial: true
    state :paid
    state :processing
    state :shipped
    state :delivered
    state :cancelled
    state :refunded
    
    event :pay do
      transitions from: :pending, to: :paid
      
      after do
        OrderMailer.payment_received(self).deliver_later
      end
    end
    
    event :process do
      transitions from: :paid, to: :processing
    end
    
    event :ship do
      transitions from: :processing, to: :shipped
      
      after do
        TrackingService.create_shipment(self)
        OrderMailer.shipped(self).deliver_later
      end
    end
    
    event :deliver do
      transitions from: :shipped, to: :delivered
    end
    
    event :cancel do
      transitions from: [:pending, :paid], to: :cancelled
      
      before do
        refund_payment if paid?
      end
      
      after do
        OrderMailer.cancelled(self).deliver_later
      end
    end
    
    event :refund do
      transitions from: :delivered, to: :refunded
      
      after do
        process_refund
      end
    end
  end
  
  private
  
  def refund_payment
    PaymentService.refund(self)
  end
  
  def process_refund
    PaymentService.process_refund(self)
  end
end
```

### Usage

```ruby
order = Order.create!
order.pending?  # => true
order.status    # => "pending"

# Trigger transitions
order.pay!
order.paid?  # => true

# Check if transition allowed
order.may_ship?  # => false (must process first)
order.may_process?  # => true

order.process!
order.ship!
order.shipped?  # => true

# Get available events
order.aasm.events  # => [:deliver, :cancel]

# Get available states
order.aasm.states.map(&:name)  # => [:pending, :paid, :processing, ...]
```

### Guards

Prevent transitions based on conditions:

```ruby
class Order < ApplicationRecord
  include AASM
  
  aasm do
    state :pending, initial: true
    state :paid
    state :shipped
    
    event :pay do
      transitions from: :pending, to: :paid, guard: :payment_valid?
    end
    
    event :ship do
      transitions from: :paid, to: :shipped, guard: :address_present?
    end
  end
  
  def payment_valid?
    payment_method.present? && total > 0
  end
  
  def address_present?
    shipping_address.present?
  end
end

# Usage
order.pay!  # Raises AASM::InvalidTransition if guard fails
order.pay   # Returns false if guard fails (no exception)
```

### Multiple Transitions

```ruby
class Document < ApplicationRecord
  include AASM
  
  aasm do
    state :draft, initial: true
    state :submitted
    state :approved
    state :published
    state :rejected
    
    event :submit do
      transitions from: :draft, to: :submitted
    end
    
    event :approve do
      transitions from: [:submitted, :rejected], to: :approved
    end
    
    event :reject do
      transitions from: :submitted, to: :rejected
    end
    
    event :publish do
      transitions from: :approved, to: :published
    end
    
    event :unpublish do
      transitions from: :published, to: :draft
    end
  end
end
```

### Scopes

AASM automatically creates scopes:

```ruby
Order.pending    # => All pending orders
Order.paid       # => All paid orders
Order.shipped    # => All shipped orders

# Combine with other scopes
Order.paid.where(user: current_user)
```

### Callbacks

```ruby
class Order < ApplicationRecord
  include AASM
  
  aasm do
    state :pending, initial: true
    state :paid
    
    event :pay do
      before do
        validate_payment!
      end
      
      after do
        send_receipt
        update_inventory
      end
      
      success do
        # Run only if transition succeeds
        Rails.logger.info "Order #{id} paid successfully"
      end
      
      error do |e|
        # Run if transition fails
        Rails.logger.error "Failed to pay order #{id}: #{e.message}"
      end
      
      transitions from: :pending, to: :paid
    end
  end
end
```

### Testing State Machines

```ruby
# spec/models/order_spec.rb
RSpec.describe Order do
  describe "state machine" do
    let(:order) { create(:order) }
    
    it "starts in pending state" do
      expect(order).to be_pending
    end
    
    describe "pay event" do
      it "transitions from pending to paid" do
        expect { order.pay! }
          .to change(order, :status).from("pending").to("paid")
      end
      
      it "sends payment receipt" do
        expect { order.pay! }
          .to have_enqueued_mail(OrderMailer, :payment_received)
      end
    end
    
    describe "ship event" do
      context "when order is paid" do
        before { order.pay! }
        
        it "allows shipping" do
          expect(order.may_ship?).to be true
        end
        
        it "transitions to shipped" do
          expect { order.ship! }
            .to change(order, :status).from("paid").to("shipped")
        end
      end
      
      context "when order is pending" do
        it "does not allow shipping" do
          expect(order.may_ship?).to be false
        end
        
        it "raises error" do
          expect { order.ship! }.to raise_error(AASM::InvalidTransition)
        end
      end
    end
  end
end
```

## Decorators (ActiveDecorator)

Use decorators for presentation logic instead of helpers or models.

### Setup

```ruby
# Gemfile
gem "active_decorator"
```

### Basic Decorator

```ruby
# app/decorators/user_decorator.rb
module UserDecorator
  def full_name
    "#{first_name} #{last_name}"
  end
  
  def avatar_url(size: :medium)
    if avatar.attached?
      helpers.url_for(avatar.variant(resize_to_limit: avatar_size(size)))
    else
      helpers.image_url("default-avatar.png")
    end
  end
  
  def formatted_created_at
    created_at.strftime("%B %d, %Y")
  end
  
  def status_badge
    case status
    when "active"
      helpers.content_tag(:span, "Active", class: "badge badge-success")
    when "inactive"
      helpers.content_tag(:span, "Inactive", class: "badge badge-secondary")
    else
      helpers.content_tag(:span, status.titleize, class: "badge badge-default")
    end
  end
  
  private
  
  def avatar_size(size)
    { small: [50, 50], medium: [100, 100], large: [200, 200] }[size]
  end
  
  # Access helpers
  def helpers
    ActionController::Base.helpers
  end
end
```

### Usage in Views

```erb
<%# Automatically decorated in views %>
<%= @user.full_name %>
<%= image_tag @user.avatar_url(size: :large) %>
<%= @user.formatted_created_at %>
<%= @user.status_badge %>

<%# Collections are decorated automatically %>
<% @users.each do |user| %>
  <p><%= user.full_name %></p>
<% end %>
```

### Manual Decoration

```ruby
# If needed in controller or elsewhere
user = User.find(1)
decorated_user = ActiveDecorator::Decorator.instance.decorate(user)
decorated_user.full_name
```

### Testing Decorators

```ruby
# spec/decorators/user_decorator_spec.rb
RSpec.describe UserDecorator do
  let(:user) { create(:user, first_name: "John", last_name: "Doe") }
  
  describe "#full_name" do
    it "combines first and last name" do
      expect(user.full_name).to eq("John Doe")
    end
  end
  
  describe "#formatted_created_at" do
    it "formats date as Month Day, Year" do
      user.update(created_at: Date.new(2025, 1, 15))
      expect(user.formatted_created_at).to eq("January 15, 2025")
    end
  end
  
  describe "#status_badge" do
    it "returns success badge for active status" do
      user.update(status: "active")
      expect(user.status_badge).to include("badge-success")
      expect(user.status_badge).to include("Active")
    end
  end
end
```

## Service Objects vs Interactions

### Why NOT Service Objects?

**Problems with traditional service objects**:
- No standard interface
- Manual type checking
- Manual validation
- Hard to compose
- Inconsistent error handling
- Verbose boilerplate

**Example service object (verbose)**:

```ruby
# DON'T DO THIS
class UserCreationService
  attr_reader :errors
  
  def initialize(params)
    @params = params
    @errors = []
  end
  
  def call
    validate_params
    return false if errors.any?
    
    create_user
  end
  
  private
  
  def validate_params
    errors << "Email required" unless @params[:email].present?
    errors << "Invalid email" unless valid_email?(@params[:email])
    errors << "Name required" unless @params[:name].present?
  end
  
  def valid_email?(email)
    email =~ URI::MailTo::EMAIL_REGEXP
  end
  
  def create_user
    # ... implementation
  end
end
```

### Use Interactions Instead

**Same logic with ActiveInteraction (concise)**:

```ruby
# DO THIS
class Users::Create < ActiveInteraction::Base
  string :email
  string :name
  
  validates :email, presence: true, format: { with: URI::MailTo::EMAIL_REGEXP }
  validates :name, presence: true
  
  def execute
    User.create!(email: email, name: name)
  end
end
```

**Benefits**:
- ✓ Less boilerplate
- ✓ Automatic type checking
- ✓ Standard validation
- ✓ Consistent interface
- ✓ Better error handling
- ✓ Composable out of the box
