# Background Jobs Reference

## Table of Contents
- [Solid Queue Setup](#solid-queue-setup)
- [Job Creation](#job-creation)
- [Scheduling Jobs](#scheduling-jobs)
- [Recurring Jobs](#recurring-jobs)
- [Job Testing](#job-testing)
- [Monitoring](#monitoring)

## Solid Queue Setup

Modern database-backed job queue for Rails 7.1+.

### Installation

```ruby
# Gemfile
gem "solid_queue"
gem "mission_control-jobs"  # Web UI for monitoring
```

```bash
# Install
$ bin/rails solid_queue:install

# This creates:
# - db/queue_schema.rb
# - config/queue.yml
# - config/recurring.yml
```

### Configuration

```yaml
# config/queue.yml
production:
  dispatchers:
    - polling_interval: 1
      batch_size: 500
  workers:
    - queues: "*"
      threads: 5
      processes: 3
      polling_interval: 0.1

development:
  dispatchers:
    - polling_interval: 1
  workers:
    - queues: "*"
      threads: 3
      processes: 1
      polling_interval: 1
```

### Application Configuration

```ruby
# config/application.rb
config.active_job.queue_adapter = :solid_queue
config.solid_queue.connects_to = { database: { writing: :queue } }

# config/database.yml
production:
  primary:
    # ... main database config
  queue:
    adapter: postgresql
    database: myapp_queue_production
    # ... rest of queue db config
```

### Running Workers

```bash
# Development
$ bin/jobs

# Production (systemd service recommended)
$ bundle exec rake solid_queue:start
```

## Job Creation

### Basic Job

```ruby
# app/jobs/send_welcome_email_job.rb
class SendWelcomeEmailJob < ApplicationJob
  queue_as :default
  
  def perform(user_id)
    user = User.find(user_id)
    UserMailer.welcome(user).deliver_now
  end
end
```

### Queue Names

```ruby
class SendWelcomeEmailJob < ApplicationJob
  queue_as :mailers  # Specific queue
  
  # Or dynamic queue
  queue_as do
    user.premium? ? :high_priority : :default
  end
  
  def perform(user)
    # ...
  end
end
```

### Job Priority

```ruby
class UrgentNotificationJob < ApplicationJob
  queue_as :notifications
  
  # Higher number = higher priority
  def perform(user_id)
    queue_adapter.enqueue self, priority: 10
  end
end
```

### Retry Configuration

```ruby
class ProcessPaymentJob < ApplicationJob
  queue_as :payments
  
  # Retry up to 5 times
  retry_on PaymentGatewayError, wait: :exponentially_longer, attempts: 5
  
  # Don't retry certain errors
  discard_on InvalidCardError
  
  # Custom retry logic
  retry_on ActiveRecord::Deadlocked, wait: 5.seconds, attempts: 3
  
  def perform(order_id)
    order = Order.find(order_id)
    PaymentGateway.charge(order)
  end
end
```

### Job Callbacks

```ruby
class ReportGenerationJob < ApplicationJob
  before_perform :log_start
  after_perform :log_completion
  around_perform :measure_time
  
  def perform(report_id)
    report = Report.find(report_id)
    report.generate!
  end
  
  private
  
  def log_start
    Rails.logger.info "Starting report generation for #{arguments.first}"
  end
  
  def log_completion
    Rails.logger.info "Completed report generation for #{arguments.first}"
  end
  
  def measure_time
    start = Time.current
    yield
    duration = Time.current - start
    Rails.logger.info "Report generation took #{duration} seconds"
  end
end
```

## Scheduling Jobs

### Enqueue Immediately

```ruby
# Enqueue now
SendWelcomeEmailJob.perform_later(user.id)

# Enqueue with options
SendWelcomeEmailJob.set(queue: :high_priority, priority: 10)
  .perform_later(user.id)
```

### Delayed Execution

```ruby
# Run in 1 hour
SendReminderJob.set(wait: 1.hour).perform_later(user.id)

# Run at specific time
SendNewsletterJob.set(wait_until: Date.tomorrow.noon).perform_later

# Run in 2 days
ExportDataJob.set(wait: 2.days).perform_later(user.id)
```

### Bulk Enqueuing

```ruby
# Enqueue multiple jobs
User.find_each do |user|
  SendWelcomeEmailJob.perform_later(user.id)
end

# Better: Use perform_all_later (Rails 7.1+)
jobs = User.pluck(:id).map do |user_id|
  SendWelcomeEmailJob.new(user_id)
end

ActiveJob.perform_all_later(jobs)
```

### Conditional Enqueuing

```ruby
class User < ApplicationRecord
  after_create :send_welcome_email
  
  private
  
  def send_welcome_email
    SendWelcomeEmailJob.perform_later(id) if confirmed?
  end
end
```

## Recurring Jobs

### Configuration

```yaml
# config/recurring.yml
production:
  cleanup_old_records:
    class: CleanupJob
    schedule: every day at 2am
  
  send_daily_digest:
    class: DailyDigestJob
    schedule: every day at 8am
    args: ["digest"]
  
  process_payments:
    class: ProcessPaymentsJob
    schedule: every 15 minutes
  
  generate_reports:
    class: GenerateReportsJob
    schedule: every monday at 9am
    args: ["weekly"]

development:
  test_job:
    class: TestJob
    schedule: every 5 minutes
```

### Recurring Job Class

```ruby
# app/jobs/cleanup_job.rb
class CleanupJob < ApplicationJob
  queue_as :maintenance
  
  def perform
    # Clean old records
    OldRecord.where("created_at < ?", 90.days.ago).delete_all
    
    # Clean expired sessions
    ActiveRecord::SessionStore::Session.where("updated_at < ?", 30.days.ago).delete_all
    
    # Clean old logs
    Rails.logger.info "Cleanup completed"
  end
end
```

### Schedule Syntax

```yaml
# Every X minutes/hours/days
schedule: every 5 minutes
schedule: every 2 hours
schedule: every day

# Specific times
schedule: every day at 3pm
schedule: every monday at 9am
schedule: every 1st of month at 8am

# Multiple times
schedule: every day at 9am, 3pm, 9pm

# With timezone
schedule: every day at 9am America/New_York
```

## Job Testing

### Basic Job Test

```ruby
# spec/jobs/send_welcome_email_job_spec.rb
RSpec.describe SendWelcomeEmailJob, type: :job do
  describe "#perform" do
    let(:user) { create(:user) }
    
    it "sends welcome email" do
      expect {
        described_class.perform_now(user.id)
      }.to change { ActionMailer::Base.deliveries.count }.by(1)
    end
    
    it "sends email to correct user" do
      described_class.perform_now(user.id)
      
      mail = ActionMailer::Base.deliveries.last
      expect(mail.to).to include(user.email)
      expect(mail.subject).to match(/welcome/i)
    end
  end
  
  describe "enqueuing" do
    it "enqueues job" do
      expect {
        described_class.perform_later(user.id)
      }.to have_enqueued_job(described_class).with(user.id)
    end
    
    it "enqueues on correct queue" do
      expect {
        described_class.perform_later(user.id)
      }.to have_enqueued_job.on_queue("mailers")
    end
    
    it "schedules delayed job" do
      expect {
        described_class.set(wait: 1.hour).perform_later(user.id)
      }.to have_enqueued_job.at(1.hour.from_now).with(user.id)
    end
  end
end
```

### Testing Retries

```ruby
RSpec.describe ProcessPaymentJob do
  describe "retry behavior" do
    let(:order) { create(:order) }
    
    it "retries on payment gateway error" do
      allow(PaymentGateway).to receive(:charge).and_raise(PaymentGatewayError)
      
      expect {
        described_class.perform_now(order.id)
      }.to raise_error(PaymentGatewayError)
      
      expect {
        described_class.perform_later(order.id)
      }.to have_enqueued_job.with(order.id)
    end
    
    it "discards on invalid card error" do
      allow(PaymentGateway).to receive(:charge).and_raise(InvalidCardError)
      
      expect {
        described_class.perform_now(order.id)
      }.not_to raise_error
      
      # Job should be discarded, not retried
      expect {
        described_class.perform_later(order.id)
      }.not_to have_enqueued_job
    end
  end
end
```

### Testing with perform_enqueued_jobs

```ruby
RSpec.describe "User registration", type: :request do
  include ActiveJob::TestHelper
  
  it "sends welcome email after registration" do
    perform_enqueued_jobs do
      post users_path, params: {
        user: { email: "user@example.com", name: "John" }
      }
    end
    
    expect(ActionMailer::Base.deliveries.count).to eq(1)
    mail = ActionMailer::Base.deliveries.last
    expect(mail.to).to include("user@example.com")
  end
end
```

## Monitoring

### Mission Control

Web UI for monitoring jobs:

```ruby
# config/routes.rb
Rails.application.routes.draw do
  mount MissionControl::Jobs::Engine, at: "/jobs"
end
```

Access at: `http://localhost:3000/jobs`

Features:
- View queued, running, and failed jobs
- Retry failed jobs
- Pause/resume queues
- View job history
- Monitor performance

### Logging

```ruby
class MyJob < ApplicationJob
  around_perform :log_performance
  
  def perform(user_id)
    Rails.logger.info "Processing user #{user_id}"
    # ... job logic
  end
  
  private
  
  def log_performance
    start = Time.current
    yield
    duration = Time.current - start
    
    Rails.logger.info "Job completed in #{duration}s"
  end
end
```

### Error Tracking

```ruby
class MyJob < ApplicationJob
  rescue_from StandardError do |exception|
    # Log to error tracking service
    ErrorTracker.notify(exception, job: self.class.name, arguments: arguments)
    
    # Re-raise to trigger retry
    raise exception
  end
  
  def perform(user_id)
    # ... job logic
  end
end
```

### Metrics

```ruby
class ApplicationJob < ActiveJob::Base
  around_perform :track_metrics
  
  private
  
  def track_metrics
    start = Time.current
    
    begin
      yield
      duration = Time.current - start
      
      # Track success metrics
      Metrics.increment("jobs.success", tags: ["job:#{self.class.name}"])
      Metrics.timing("jobs.duration", duration, tags: ["job:#{self.class.name}"])
    rescue => e
      # Track failure metrics
      Metrics.increment("jobs.failure", tags: ["job:#{self.class.name}", "error:#{e.class}"])
      raise
    end
  end
end
```

## Best Practices

### Keep Jobs Idempotent

Jobs should be safe to run multiple times:

```ruby
# GOOD - Idempotent
class UpdateUserStatusJob < ApplicationJob
  def perform(user_id)
    user = User.find(user_id)
    user.update(status: "active") unless user.active?
  end
end

# BAD - Not idempotent
class IncrementCounterJob < ApplicationJob
  def perform(user_id)
    user = User.find(user_id)
    user.increment!(:login_count)  # Dangerous if job runs twice
  end
end
```

### Pass IDs, Not Objects

```ruby
# GOOD - Pass ID
SendEmailJob.perform_later(user.id)

class SendEmailJob < ApplicationJob
  def perform(user_id)
    user = User.find(user_id)  # Fetch fresh data
    UserMailer.welcome(user).deliver_now
  end
end

# BAD - Pass object (can cause stale data)
SendEmailJob.perform_later(user)
```

### Break Large Jobs into Smaller Ones

```ruby
# GOOD - Parent job enqueues smaller jobs
class ProcessBatchJob < ApplicationJob
  def perform(batch_id)
    batch = Batch.find(batch_id)
    
    batch.items.find_each do |item|
      ProcessItemJob.perform_later(item.id)
    end
  end
end

# BAD - One huge job
class ProcessAllItemsJob < ApplicationJob
  def perform
    Item.find_each do |item|  # Could timeout
      item.process!
    end
  end
end
```

### Handle Failures Gracefully

```ruby
class SendNewsletterJob < ApplicationJob
  retry_on MailerError, wait: :exponentially_longer, attempts: 5
  
  discard_on ActiveRecord::RecordNotFound do |job, error|
    Rails.logger.error "User not found: #{job.arguments.first}"
  end
  
  def perform(user_id)
    user = User.find(user_id)
    NewsletterMailer.send_to(user).deliver_now
  rescue => e
    # Log error but don't retry
    ErrorTracker.notify(e, user_id: user_id)
    raise
  end
end
```

### Set Appropriate Timeouts

```ruby
class LongRunningJob < ApplicationJob
  # Set execution timeout
  queue_with_priority 5
  
  def perform
    Timeout.timeout(5.minutes) do
      # Long-running task
    end
  rescue Timeout::Error
    Rails.logger.error "Job timed out after 5 minutes"
    raise  # Will trigger retry
  end
end
```
