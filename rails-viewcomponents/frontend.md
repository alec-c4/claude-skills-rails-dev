# Frontend & Components Reference

## Table of Contents
- [Hotwire Stack](#hotwire-stack)
- [Turbo Frames & Streams](#turbo-frames--streams)
- [Stimulus Controllers](#stimulus-controllers)
- [ViewComponent Patterns](#viewcomponent-patterns)
- [Confirmation Modals](#confirmation-modals)
- [Form Patterns](#form-patterns)

## Hotwire Stack

Modern Rails frontend using Turbo and Stimulus for SPA-like interactions without heavy JavaScript frameworks.

### Core Technologies

**Turbo**: Server-rendered HTML over WebSocket
- Turbo Drive: Automatic page acceleration
- Turbo Frames: Decomposed pages
- Turbo Streams: Partial page updates

**Stimulus**: Modest JavaScript framework
- Progressive enhancement
- Data attributes for behavior
- Lifecycle callbacks

### Setup

```ruby
# Gemfile
gem "turbo-rails"
gem "stimulus-rails"
gem "importmap-rails"

# app/javascript/application.js
import "@hotwired/turbo-rails"
import "./controllers"
```

## Turbo Frames & Streams

### Turbo Frames

Decompose pages into independent contexts:

```erb
<%# app/views/articles/index.html.erb %>
<h1>Articles</h1>

<div id="search-form">
  <%= turbo_frame_tag "search" do %>
    <%= form_with url: articles_path, method: :get do |f| %>
      <%= f.text_field :q, placeholder: "Search..." %>
      <%= f.submit "Search" %>
    <% end %>
  <% end %>
</div>

<%= turbo_frame_tag "articles" do %>
  <%= render @articles %>
<% end %>
```

```erb
<%# app/views/articles/_article.html.erb %>
<%= turbo_frame_tag dom_id(article) do %>
  <article>
    <h2><%= article.title %></h2>
    <p><%= article.body %></p>
    
    <%= link_to "Edit", edit_article_path(article) %>
    <%= button_to "Delete", article_path(article), method: :delete %>
  </article>
<% end %>
```

**Edit in-place**:

```erb
<%# app/views/articles/edit.html.erb %>
<%= turbo_frame_tag dom_id(@article) do %>
  <%= form_with model: @article do |f| %>
    <%= f.text_field :title %>
    <%= f.text_area :body %>
    <%= f.submit "Update" %>
    <%= link_to "Cancel", article_path(@article) %>
  <% end %>
<% end %>
```

### Turbo Streams

Update multiple parts of page:

```ruby
# app/controllers/articles_controller.rb
class ArticlesController < ApplicationController
  def create
    @article = Article.new(article_params)
    
    respond_to do |format|
      if @article.save
        format.turbo_stream do
          render turbo_stream: [
            turbo_stream.prepend("articles", partial: "article", locals: { article: @article }),
            turbo_stream.update("form", partial: "form", locals: { article: Article.new })
          ]
        end
        format.html { redirect_to @article }
      else
        format.html { render :new, status: :unprocessable_entity }
      end
    end
  end
  
  def destroy
    @article = Article.find(params[:id])
    @article.destroy
    
    respond_to do |format|
      format.turbo_stream { render turbo_stream: turbo_stream.remove(@article) }
      format.html { redirect_to articles_path }
    end
  end
end
```

**Turbo Stream Actions**:
- `append` - Add to end
- `prepend` - Add to beginning
- `replace` - Replace entire element
- `update` - Replace element contents
- `remove` - Remove element
- `before` - Insert before
- `after` - Insert after

### Broadcasting Updates

Real-time updates via WebSocket:

```ruby
# app/models/article.rb
class Article < ApplicationRecord
  after_create_commit -> { broadcast_prepend_to "articles" }
  after_update_commit -> { broadcast_replace_to "articles" }
  after_destroy_commit -> { broadcast_remove_to "articles" }
end
```

```erb
<%# app/views/articles/index.html.erb %>
<%= turbo_stream_from "articles" %>

<div id="articles">
  <%= render @articles %>
</div>
```

## Stimulus Controllers

### Basic Controller

```javascript
// app/javascript/controllers/dropdown_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = ["menu"]
  static values = { open: Boolean }
  
  connect() {
    console.log("Dropdown controller connected")
  }
  
  toggle() {
    this.openValue = !this.openValue
  }
  
  openValueChanged() {
    if (this.openValue) {
      this.menuTarget.classList.remove("hidden")
    } else {
      this.menuTarget.classList.add("hidden")
    }
  }
  
  hide(event) {
    if (!this.element.contains(event.target)) {
      this.openValue = false
    }
  }
}
```

```erb
<%# Usage in view %>
<div data-controller="dropdown" data-action="click@window->dropdown#hide">
  <button data-action="dropdown#toggle">
    Options
  </button>
  
  <div data-dropdown-target="menu" class="hidden">
    <a href="#">Edit</a>
    <a href="#">Delete</a>
  </div>
</div>
```

### Common Stimulus Patterns

**Form Validation**:

```javascript
// app/javascript/controllers/form_validation_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = ["field", "error", "submit"]
  
  validate() {
    const isValid = this.fieldTargets.every(field => field.checkValidity())
    this.submitTarget.disabled = !isValid
  }
  
  showError(event) {
    const field = event.target
    const errorTarget = this.errorTargets.find(
      error => error.dataset.field === field.name
    )
    
    if (errorTarget) {
      errorTarget.textContent = field.validationMessage
      errorTarget.classList.toggle("hidden", field.validity.valid)
    }
  }
}
```

**Auto-save**:

```javascript
// app/javascript/controllers/autosave_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = ["form", "status"]
  static values = { delay: { type: Number, default: 1000 } }
  
  connect() {
    this.timeout = null
  }
  
  save() {
    clearTimeout(this.timeout)
    
    this.timeout = setTimeout(() => {
      this.submitForm()
    }, this.delayValue)
  }
  
  submitForm() {
    this.showStatus("Saving...")
    
    const formData = new FormData(this.formTarget)
    const url = this.formTarget.action
    
    fetch(url, {
      method: "PATCH",
      body: formData,
      headers: {
        "X-CSRF-Token": document.querySelector("[name='csrf-token']").content
      }
    })
    .then(response => {
      if (response.ok) {
        this.showStatus("Saved")
      } else {
        this.showStatus("Error saving")
      }
    })
  }
  
  showStatus(message) {
    this.statusTarget.textContent = message
  }
}
```

## ViewComponent Patterns

### Component with Slots

```ruby
# app/components/card_component.rb
class CardComponent < ViewComponent::Base
  renders_one :header
  renders_one :footer
  renders_many :actions
  
  def initialize(variant: :default, **options)
    @variant = variant
    @options = options
  end
end
```

```erb
<%# app/components/card_component.html.erb %>
<div class="card card-<%= @variant %>" <%= html_attributes(@options) %>>
  <% if header %>
    <div class="card-header">
      <%= header %>
    </div>
  <% end %>
  
  <div class="card-body">
    <%= content %>
  </div>
  
  <% if footer || actions? %>
    <div class="card-footer">
      <%= footer %>
      
      <% if actions? %>
        <div class="card-actions">
          <% actions.each do |action| %>
            <%= action %>
          <% end %>
        </div>
      <% end %>
    </div>
  <% end %>
</div>
```

```erb
<%# Usage %>
<%= render CardComponent.new(variant: :primary) do |card| %>
  <% card.with_header do %>
    <h3>Card Title</h3>
  <% end %>
  
  <p>This is the card body content.</p>
  
  <% card.with_action do %>
    <%= link_to "Edit", edit_path %>
  <% end %>
  
  <% card.with_action do %>
    <%= link_to "Delete", delete_path, method: :delete %>
  <% end %>
<% end %>
```

### Component with Variants

```ruby
# app/components/badge_component.rb
class BadgeComponent < ViewComponent::Base
  VARIANTS = {
    primary: "bg-blue-100 text-blue-800",
    success: "bg-green-100 text-green-800",
    warning: "bg-yellow-100 text-yellow-800",
    danger: "bg-red-100 text-red-800"
  }.freeze
  
  def initialize(text:, variant: :primary, **options)
    @text = text
    @variant = variant
    @options = options
  end
  
  def variant_classes
    VARIANTS[@variant]
  end
end
```

### Component with Conditional Rendering

```ruby
# app/components/alert_component.rb
class AlertComponent < ViewComponent::Base
  def initialize(message:, type: :info, dismissible: true)
    @message = message
    @type = type
    @dismissible = dismissible
  end
  
  def render?
    @message.present?
  end
  
  def icon_name
    {
      info: "information-circle",
      success: "check-circle",
      warning: "exclamation-triangle",
      danger: "x-circle"
    }[@type]
  end
end
```

### Component Collection

```ruby
# app/components/list_component.rb
class ListComponent < ViewComponent::Base
  renders_many :items, ListItemComponent
  
  def initialize(**options)
    @options = options
  end
end

class ListItemComponent < ViewComponent::Base
  def initialize(text:, **options)
    @text = text
    @options = options
  end
end
```

```erb
<%= render ListComponent.new do |list| %>
  <% @articles.each do |article| %>
    <% list.with_item(text: article.title) %>
  <% end %>
<% end %>
```

## Confirmation Modals

Use custom modals instead of browser `confirm()` for better UX.

### Reusable Confirmation Controller

```javascript
// app/javascript/controllers/confirmation_modal_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = ["modal", "title", "message", "confirmButton", "cancelButton"]
  
  static ask(caller, options = {}) {
    const controller = caller.application.getControllerForElementAndIdentifier(
      document.querySelector("[data-controller~='confirmation-modal']"),
      "confirmation-modal"
    )
    
    if (controller) {
      controller.show(options)
    }
  }
  
  show(options = {}) {
    this.onConfirm = options.onConfirm || (() => {})
    this.onCancel = options.onCancel || (() => {})
    
    this.titleTarget.textContent = options.title || "Confirm"
    this.messageTarget.textContent = options.message || "Are you sure?"
    this.confirmButtonTarget.textContent = options.confirmText || "Confirm"
    this.cancelButtonTarget.textContent = options.cancelText || "Cancel"
    
    if (options.confirmClass) {
      this.confirmButtonTarget.className = options.confirmClass
    }
    
    this.modalTarget.classList.remove("hidden")
  }
  
  hide() {
    this.modalTarget.classList.add("hidden")
  }
  
  confirm() {
    this.onConfirm()
    this.hide()
  }
  
  cancel() {
    this.onCancel()
    this.hide()
  }
}
```

### Confirmation Modal Partial

```erb
<%# app/views/shared/_confirmation_modal.html.erb %>
<div data-controller="confirmation-modal">
  <div data-confirmation-modal-target="modal" 
       class="hidden fixed inset-0 bg-gray-600 bg-opacity-50 overflow-y-auto h-full w-full z-50">
    <div class="relative top-20 mx-auto p-5 border w-96 shadow-lg rounded-md bg-white">
      <div class="mt-3">
        <h3 data-confirmation-modal-target="title" class="text-lg font-medium text-gray-900 mb-4">
          Confirm
        </h3>
        
        <p data-confirmation-modal-target="message" class="text-sm text-gray-500 mb-4">
          Are you sure?
        </p>
        
        <div class="flex justify-end gap-3">
          <button data-confirmation-modal-target="cancelButton"
                  data-action="confirmation-modal#cancel"
                  class="px-4 py-2 bg-gray-200 text-gray-800 rounded hover:bg-gray-300">
            Cancel
          </button>
          
          <button data-confirmation-modal-target="confirmButton"
                  data-action="confirmation-modal#confirm"
                  class="px-4 py-2 bg-red-600 text-white rounded hover:bg-red-700">
            Confirm
          </button>
        </div>
      </div>
    </div>
  </div>
</div>
```

### Usage Example

```javascript
// app/javascript/controllers/article_actions_controller.js
import { Controller } from "@hotwired/stimulus"
import ConfirmationModalController from "./confirmation_modal_controller"

export default class extends Controller {
  confirmDelete(event) {
    event.preventDefault()
    
    const articleTitle = event.currentTarget.dataset.articleTitle
    const form = event.currentTarget.closest('form')
    
    ConfirmationModalController.ask(this, {
      title: "Delete Article",
      message: `Are you sure you want to delete "${articleTitle}"? This action cannot be undone.`,
      confirmText: "Delete",
      cancelText: "Cancel",
      confirmClass: "px-4 py-2 bg-red-600 text-white rounded hover:bg-red-700",
      onConfirm: () => form.requestSubmit()
    })
  }
}
```

```erb
<%# In view %>
<%= button_to "Delete", 
    article_path(article), 
    method: :delete,
    data: {
      controller: "article-actions",
      action: "article-actions#confirmDelete",
      article_title: article.title
    },
    class: "text-red-600 hover:underline" %>
```

**Why custom modals?**:
- ✓ Consistent visual design
- ✓ Customizable styling
- ✓ Better UX with accessible close
- ✓ Support for i18n
- ✓ Keyboard navigation
- ✓ Can include additional context

**Never use**:
```erb
<%# WRONG - Browser confirm dialog %>
<%= button_to "Delete", article_path, method: :delete, data: { turbo_confirm: "Are you sure?" } %>
```

## Form Patterns

### Form with Turbo

```erb
<%# app/views/articles/new.html.erb %>
<%= turbo_frame_tag "article_form" do %>
  <%= form_with model: @article, data: { turbo_frame: "_top" } do |f| %>
    <div class="field">
      <%= f.label :title %>
      <%= f.text_field :title, class: "form-input" %>
      <%= f.error_message :title %>
    </div>
    
    <div class="field">
      <%= f.label :body %>
      <%= f.text_area :body, class: "form-input" %>
      <%= f.error_message :body %>
    </div>
    
    <div class="actions">
      <%= f.submit "Create Article", class: "btn btn-primary" %>
      <%= link_to "Cancel", articles_path, class: "btn btn-secondary" %>
    </div>
  <% end %>
<% end %>
```

### Form with Stimulus Validation

```erb
<%= form_with model: @article, 
    data: { 
      controller: "form-validation",
      action: "input->form-validation#validate"
    } do |f| %>
  
  <%= f.text_field :title,
      required: true,
      data: { form_validation_target: "field" },
      class: "form-input" %>
  
  <span data-form-validation-target="error" 
        data-field="title" 
        class="text-red-600 text-sm hidden"></span>
  
  <%= f.submit "Save",
      data: { form_validation_target: "submit" },
      class: "btn btn-primary" %>
<% end %>
```

### Nested Forms

```ruby
# app/models/article.rb
class Article < ApplicationRecord
  has_many :tags, dependent: :destroy
  accepts_nested_attributes_for :tags, allow_destroy: true, reject_if: :all_blank
end
```

```erb
<%= form_with model: @article do |f| %>
  <%= f.text_field :title %>
  
  <div data-controller="nested-form">
    <%= f.fields_for :tags do |tag_fields| %>
      <div data-nested-form-target="item">
        <%= tag_fields.text_field :name %>
        <%= tag_fields.check_box :_destroy %>
        <%= tag_fields.label :_destroy, "Remove" %>
      </div>
    <% end %>
    
    <button type="button" data-action="nested-form#add">
      Add Tag
    </button>
  </div>
  
  <%= f.submit %>
<% end %>
```
