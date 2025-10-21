# Inertia.js Reference Guide

Comprehensive guide for building SPAs with Inertia.js, React/Vue/Svelte, and Rails.

## Table of Contents

- [Complete Setup](#complete-setup)
- [React Examples](#react-examples)
- [Vue Examples](#vue-examples)
- [Svelte Examples](#svelte-examples)
- [Advanced Patterns](#advanced-patterns)
- [Server-Side Rendering (SSR)](#server-side-rendering-ssr)
- [Testing Strategies](#testing-strategies)
- [Performance Optimization](#performance-optimization)

---

## Complete Setup

### Step-by-Step Installation

```bash
# 1. Add gems
bundle add inertia_rails vite_rails

# 2. Install Inertia
rails inertia:install

# 3. Choose framework (React, Vue, or Svelte)
# This creates:
# - app/frontend/entrypoints/application.js
# - app/frontend/pages/ directory
# - Updates Vite config
```

### Complete Configuration

```ruby
# config/initializers/inertia_rails.rb
InertiaRails.configure do |config|
  # Version for cache busting (forces client refresh when assets change)
  config.version = ViteRuby.digest

  # Deep merge mode for nested shared data
  config.deep_merge_shared_data = true

  # Share data with ALL pages
  config.share do |controller|
    {
      # Current user data
      auth: {
        user: controller.current_user&.as_json(
          only: [:id, :name, :email, :avatar_url],
          methods: [:admin?]
        ),
        # Authorization helper
        can: lambda { |action, resource|
          controller.current_user&.can?(action, resource)
        }
      },

      # Flash messages
      flash: {
        success: controller.flash[:success],
        error: controller.flash[:error],
        notice: controller.flash[:notice],
        alert: controller.flash[:alert]
      },

      # App-wide settings
      app: {
        name: Rails.application.class.module_parent_name,
        env: Rails.env,
        locale: I18n.locale
      },

      # CSRF token for non-Inertia requests
      csrf_token: controller.form_authenticity_token
    }
  end

  # SSR configuration (optional)
  config.ssr_enabled = Rails.env.production?
  config.ssr_url = ENV.fetch('INERTIA_SSR_URL', 'http://localhost:13714')
end
```

### Vite Configuration

```javascript
// vite.config.js
import { defineConfig } from 'vite'
import RubyPlugin from 'vite-plugin-ruby'
import react from '@vitejs/plugin-react'  // or vue/svelte

export default defineConfig({
  plugins: [
    RubyPlugin(),
    react()
  ],
  resolve: {
    alias: {
      '@': '/app/frontend'
    }
  }
})
```

---

## React Examples

### Complete Entry Point

```javascript
// app/frontend/entrypoints/application.js
import { createInertiaApp } from '@inertiajs/react'
import { createRoot } from 'react-dom/client'
import '../styles/application.css'

createInertiaApp({
  // Page resolution
  resolve: name => {
    const pages = import.meta.glob('../pages/**/*.jsx', { eager: true })
    const page = pages[`../pages/${name}.jsx`]

    if (!page) {
      throw new Error(`Page not found: ${name}`)
    }

    return page
  },

  // Setup function
  setup({ el, App, props }) {
    createRoot(el).render(<App {...props} />)
  },

  // Progress bar
  progress: {
    delay: 250,
    color: '#29d',
    includeCSS: true,
    showSpinner: true
  }
})
```

### Complete Index Page

```jsx
// app/frontend/pages/Articles/Index.jsx
import { Link, usePage, router } from '@inertiajs/react'
import { useState, useMemo } from 'react'
import debounce from 'lodash/debounce'
import AppLayout from '@/layouts/AppLayout'

export default function Index({ articles, pagination, filters }) {
  const { auth } = usePage().props
  const [search, setSearch] = useState(filters.search || '')

  // Debounced search
  const handleSearch = useMemo(
    () => debounce(value => {
      router.get('/articles', { search: value }, {
        preserveState: true,
        replace: true
      })
    }, 300),
    []
  )

  return (
    <div className="container mx-auto px-4">
      {/* Header */}
      <div className="flex justify-between items-center mb-6">
        <h1 className="text-3xl font-bold">Articles</h1>
        {auth.can('create', 'Article') && (
          <Link
            href="/articles/new"
            className="btn btn-primary"
          >
            New Article
          </Link>
        )}
      </div>

      {/* Search */}
      <div className="mb-6">
        <input
          type="search"
          value={search}
          onChange={e => {
            setSearch(e.target.value)
            handleSearch(e.target.value)
          }}
          placeholder="Search articles..."
          className="form-input w-full"
        />
      </div>

      {/* Articles List */}
      <div className="grid gap-4">
        {articles.length === 0 ? (
          <p className="text-gray-500">No articles found</p>
        ) : (
          articles.map(article => (
            <Link
              key={article.id}
              href={`/articles/${article.id}`}
              className="card hover:shadow-lg transition"
            >
              <div className="flex justify-between items-start">
                <div className="flex-1">
                  <h2 className="text-xl font-semibold">{article.title}</h2>
                  <p className="text-gray-600 mt-2">{article.excerpt}</p>
                  <div className="text-sm text-gray-500 mt-2">
                    By {article.author_name} • {article.created_at}
                  </div>
                </div>
                {article.featured && (
                  <span className="badge badge-primary">Featured</span>
                )}
              </div>
            </Link>
          ))
        )}
      </div>

      {/* Pagination */}
      {pagination.total_pages > 1 && (
        <div className="flex justify-center gap-2 mt-6">
          {pagination.current_page > 1 && (
            <Link
              href={`/articles?page=${pagination.current_page - 1}`}
              className="btn btn-secondary"
            >
              Previous
            </Link>
          )}

          {Array.from({ length: pagination.total_pages }, (_, i) => i + 1).map(page => (
            <Link
              key={page}
              href={`/articles?page=${page}`}
              className={`btn ${page === pagination.current_page ? 'btn-primary' : 'btn-secondary'}`}
            >
              {page}
            </Link>
          ))}

          {pagination.current_page < pagination.total_pages && (
            <Link
              href={`/articles?page=${pagination.current_page + 1}`}
              className="btn btn-secondary"
            >
              Next
            </Link>
          )}
        </div>
      )}
    </div>
  )
}

Index.layout = page => <AppLayout>{page}</AppLayout>
```

### Complete Form with Validation

```jsx
// app/frontend/pages/Articles/Form.jsx
import { useForm, usePage } from '@inertiajs/react'
import { useEffect } from 'react'
import AppLayout from '@/layouts/AppLayout'

export default function Form({ article }) {
  const { flash } = usePage().props
  const isEdit = !!article

  const { data, setData, post, put, processing, errors, reset } = useForm({
    title: article?.title || '',
    body: article?.body || '',
    excerpt: article?.excerpt || '',
    published: article?.published || false,
    featured: article?.featured || false,
    category_id: article?.category_id || '',
    tag_ids: article?.tag_ids || []
  })

  function handleSubmit(e) {
    e.preventDefault()

    if (isEdit) {
      put(`/articles/${article.id}`, {
        preserveScroll: true,
        onSuccess: () => reset('title', 'body', 'excerpt'),
        onError: (errors) => {
          console.error('Validation errors:', errors)
        }
      })
    } else {
      post('/articles', {
        preserveScroll: true,
        onSuccess: () => {
          // Redirect handled by controller
        }
      })
    }
  }

  // Auto-generate excerpt
  useEffect(() => {
    if (!data.excerpt && data.body) {
      setData('excerpt', data.body.slice(0, 200))
    }
  }, [data.body])

  return (
    <div className="container mx-auto px-4 max-w-2xl">
      <h1 className="text-3xl font-bold mb-6">
        {isEdit ? 'Edit Article' : 'New Article'}
      </h1>

      <form onSubmit={handleSubmit} className="space-y-6">
        {/* Title */}
        <div>
          <label htmlFor="title" className="block font-semibold mb-2">
            Title *
          </label>
          <input
            id="title"
            type="text"
            value={data.title}
            onChange={e => setData('title', e.target.value)}
            className={`form-input w-full ${errors.title ? 'border-red-500' : ''}`}
            disabled={processing}
          />
          {errors.title && (
            <div className="text-red-600 text-sm mt-1">{errors.title}</div>
          )}
        </div>

        {/* Body */}
        <div>
          <label htmlFor="body" className="block font-semibold mb-2">
            Body *
          </label>
          <textarea
            id="body"
            value={data.body}
            onChange={e => setData('body', e.target.value)}
            rows="15"
            className={`form-input w-full ${errors.body ? 'border-red-500' : ''}`}
            disabled={processing}
          />
          {errors.body && (
            <div className="text-red-600 text-sm mt-1">{errors.body}</div>
          )}
          <div className="text-sm text-gray-500 mt-1">
            {data.body.length} characters
          </div>
        </div>

        {/* Excerpt */}
        <div>
          <label htmlFor="excerpt" className="block font-semibold mb-2">
            Excerpt
          </label>
          <textarea
            id="excerpt"
            value={data.excerpt}
            onChange={e => setData('excerpt', e.target.value)}
            rows="3"
            className="form-input w-full"
            placeholder="Auto-generated from body if left empty"
            disabled={processing}
          />
        </div>

        {/* Checkboxes */}
        <div className="space-y-2">
          <label className="flex items-center">
            <input
              type="checkbox"
              checked={data.published}
              onChange={e => setData('published', e.target.checked)}
              className="mr-2"
              disabled={processing}
            />
            <span>Publish immediately</span>
          </label>

          <label className="flex items-center">
            <input
              type="checkbox"
              checked={data.featured}
              onChange={e => setData('featured', e.target.checked)}
              className="mr-2"
              disabled={processing}
            />
            <span>Feature on homepage</span>
          </label>
        </div>

        {/* Actions */}
        <div className="flex gap-4">
          <button
            type="submit"
            disabled={processing}
            className="btn btn-primary"
          >
            {processing
              ? (isEdit ? 'Updating...' : 'Creating...')
              : (isEdit ? 'Update Article' : 'Create Article')
            }
          </button>

          <Link
            href="/articles"
            className="btn btn-secondary"
            disabled={processing}
          >
            Cancel
          </Link>
        </div>
      </form>
    </div>
  )
}

Form.layout = page => <AppLayout>{page}</AppLayout>
```

---

## Vue Examples

### Complete Entry Point

```javascript
// app/frontend/entrypoints/application.js
import { createInertiaApp } from '@inertiajs/vue3'
import { createApp, h } from 'vue'
import '../styles/application.css'

createInertiaApp({
  resolve: name => {
    const pages = import.meta.glob('../pages/**/*.vue', { eager: true })
    return pages[`../pages/${name}.vue`]
  },

  setup({ el, App, props, plugin }) {
    createApp({ render: () => h(App, props) })
      .use(plugin)
      .mount(el)
  },

  progress: {
    color: '#29d'
  }
})
```

### Complete Index Page (Vue)

```vue
<!-- app/frontend/pages/Articles/Index.vue -->
<script setup>
import { Link, router, usePage } from '@inertiajs/vue3'
import { ref, computed } from 'vue'
import { debounce } from 'lodash'
import AppLayout from '@/layouts/AppLayout.vue'

const props = defineProps({
  articles: Array,
  pagination: Object,
  filters: Object
})

const page = usePage()
const auth = computed(() => page.props.auth)

const search = ref(props.filters.search || '')

const handleSearch = debounce((value) => {
  router.get('/articles', { search: value }, {
    preserveState: true,
    replace: true
  })
}, 300)
</script>

<template>
  <AppLayout>
    <div class="container mx-auto px-4">
      <!-- Header -->
      <div class="flex justify-between items-center mb-6">
        <h1 class="text-3xl font-bold">Articles</h1>
        <Link
          v-if="auth.can('create', 'Article')"
          href="/articles/new"
          class="btn btn-primary"
        >
          New Article
        </Link>
      </div>

      <!-- Search -->
      <div class="mb-6">
        <input
          v-model="search"
          @input="handleSearch(search)"
          type="search"
          placeholder="Search articles..."
          class="form-input w-full"
        />
      </div>

      <!-- Articles List -->
      <div class="grid gap-4">
        <p v-if="articles.length === 0" class="text-gray-500">
          No articles found
        </p>

        <Link
          v-for="article in articles"
          :key="article.id"
          :href="`/articles/${article.id}`"
          class="card hover:shadow-lg transition"
        >
          <div class="flex justify-between items-start">
            <div class="flex-1">
              <h2 class="text-xl font-semibold">{{ article.title }}</h2>
              <p class="text-gray-600 mt-2">{{ article.excerpt }}</p>
              <div class="text-sm text-gray-500 mt-2">
                By {{ article.author_name }} • {{ article.created_at }}
              </div>
            </div>
            <span v-if="article.featured" class="badge badge-primary">
              Featured
            </span>
          </div>
        </Link>
      </div>

      <!-- Pagination -->
      <div v-if="pagination.total_pages > 1" class="flex justify-center gap-2 mt-6">
        <Link
          v-if="pagination.current_page > 1"
          :href="`/articles?page=${pagination.current_page - 1}`"
          class="btn btn-secondary"
        >
          Previous
        </Link>

        <Link
          v-for="page in pagination.total_pages"
          :key="page"
          :href="`/articles?page=${page}`"
          :class="['btn', page === pagination.current_page ? 'btn-primary' : 'btn-secondary']"
        >
          {{ page }}
        </Link>

        <Link
          v-if="pagination.current_page < pagination.total_pages"
          :href="`/articles?page=${pagination.current_page + 1}`"
          class="btn btn-secondary"
        >
          Next
        </Link>
      </div>
    </div>
  </AppLayout>
</template>
```

---

## Svelte Examples

### Complete Entry Point

```javascript
// app/frontend/entrypoints/application.js
import { createInertiaApp } from '@inertiajs/svelte'
import '../styles/application.css'

createInertiaApp({
  resolve: name => {
    const pages = import.meta.glob('../pages/**/*.svelte', { eager: true })
    return pages[`../pages/${name}.svelte`]
  },

  setup({ el, App }) {
    new App({ target: el })
  },

  progress: {
    color: '#29d'
  }
})
```

### Complete Index Page (Svelte)

```svelte
<!-- app/frontend/pages/Articles/Index.svelte -->
<script>
  import { inertia, Link, router, page } from '@inertiajs/svelte'
  import { debounce } from 'lodash'
  import AppLayout from '@/layouts/AppLayout.svelte'

  export let articles = []
  export let pagination = {}
  export let filters = {}

  $: auth = $page.props.auth

  let search = filters.search || ''

  const handleSearch = debounce((value) => {
    router.get('/articles', { search: value }, {
      preserveState: true,
      replace: true
    })
  }, 300)

  $: handleSearch(search)
</script>

<AppLayout>
  <div class="container mx-auto px-4">
    <!-- Header -->
    <div class="flex justify-between items-center mb-6">
      <h1 class="text-3xl font-bold">Articles</h1>
      {#if auth.can('create', 'Article')}
        <Link href="/articles/new" class="btn btn-primary">
          New Article
        </Link>
      {/if}
    </div>

    <!-- Search -->
    <div class="mb-6">
      <input
        bind:value={search}
        type="search"
        placeholder="Search articles..."
        class="form-input w-full"
      />
    </div>

    <!-- Articles List -->
    <div class="grid gap-4">
      {#if articles.length === 0}
        <p class="text-gray-500">No articles found</p>
      {:else}
        {#each articles as article (article.id)}
          <Link
            href="/articles/{article.id}"
            class="card hover:shadow-lg transition"
          >
            <div class="flex justify-between items-start">
              <div class="flex-1">
                <h2 class="text-xl font-semibold">{article.title}</h2>
                <p class="text-gray-600 mt-2">{article.excerpt}</p>
                <div class="text-sm text-gray-500 mt-2">
                  By {article.author_name} • {article.created_at}
                </div>
              </div>
              {#if article.featured}
                <span class="badge badge-primary">Featured</span>
              {/if}
            </div>
          </Link>
        {/each}
      {/if}
    </div>

    <!-- Pagination -->
    {#if pagination.total_pages > 1}
      <div class="flex justify-center gap-2 mt-6">
        {#if pagination.current_page > 1}
          <Link
            href="/articles?page={pagination.current_page - 1}"
            class="btn btn-secondary"
          >
            Previous
          </Link>
        {/if}

        {#each Array(pagination.total_pages) as _, i}
          <Link
            href="/articles?page={i + 1}"
            class="btn {i + 1 === pagination.current_page ? 'btn-primary' : 'btn-secondary'}"
          >
            {i + 1}
          </Link>
        {/each}

        {#if pagination.current_page < pagination.total_pages}
          <Link
            href="/articles?page={pagination.current_page + 1}"
            class="btn btn-secondary"
          >
            Next
          </Link>
        {/if}
      </div>
    {/if}
  </div>
</AppLayout>
```

---

## Advanced Patterns

### Modal Windows

```jsx
// app/frontend/pages/Articles/Index.jsx
import { useState } from 'react'
import { router } from '@inertiajs/react'
import Modal from '@/components/Modal'

export default function Index({ articles, showModal, modalArticle }) {
  function openArticleModal(article) {
    router.visit(`/articles/${article.id}/modal`, {
      preserveState: true,
      only: ['showModal', 'modalArticle']
    })
  }

  function closeModal() {
    router.visit('/articles', {
      preserveState: true,
      only: ['showModal', 'modalArticle']
    })
  }

  return (
    <div>
      {articles.map(article => (
        <div key={article.id}>
          <button onClick={() => openArticleModal(article)}>
            Quick View
          </button>
        </div>
      ))}

      {showModal && (
        <Modal onClose={closeModal}>
          <h2>{modalArticle.title}</h2>
          <p>{modalArticle.body}</p>
        </Modal>
      )}
    </div>
  )
}
```

```ruby
# app/controllers/articles_controller.rb
def modal
  article = Article.find(params[:id])

  render inertia: 'Articles/Index', props: {
    articles: Article.all.as_json,
    showModal: true,
    modalArticle: article.as_json
  }
end
```

### Infinite Scrolling

```jsx
import { router } from '@inertiajs/react'
import { useEffect, useState } from 'react'

export default function InfiniteList({ articles, pagination }) {
  const [allArticles, setAllArticles] = useState(articles)
  const [loading, setLoading] = useState(false)

  function loadMore() {
    if (loading || !pagination.has_next_page) return

    setLoading(true)
    router.visit(`/articles?page=${pagination.current_page + 1}`, {
      preserveState: true,
      preserveScroll: true,
      only: ['articles', 'pagination'],
      onSuccess: (page) => {
        setAllArticles(prev => [...prev, ...page.props.articles])
        setLoading(false)
      }
    })
  }

  useEffect(() => {
    const observer = new IntersectionObserver(entries => {
      if (entries[0].isIntersecting) loadMore()
    })

    const sentinel = document.querySelector('#sentinel')
    if (sentinel) observer.observe(sentinel)

    return () => observer.disconnect()
  }, [pagination])

  return (
    <div>
      {allArticles.map(article => (
        <ArticleCard key={article.id} article={article} />
      ))}

      {pagination.has_next_page && (
        <div id="sentinel" className="py-4 text-center">
          {loading ? 'Loading...' : 'Load more'}
        </div>
      )}
    </div>
  )
}
```

### Optimistic Updates

```jsx
import { router } from '@inertiajs/react'
import { useState } from 'react'

export default function LikeButton({ article }) {
  const [liked, setLiked] = useState(article.liked_by_current_user)
  const [count, setCount] = useState(article.likes_count)

  function toggleLike() {
    // Optimistic update
    setLiked(!liked)
    setCount(liked ? count - 1 : count + 1)

    router.post(`/articles/${article.id}/like`, {}, {
      preserveScroll: true,
      onError: () => {
        // Revert on error
        setLiked(liked)
        setCount(count)
      }
    })
  }

  return (
    <button onClick={toggleLike}>
      {liked ? '❤️' : '🤍'} {count}
    </button>
  )
}
```

---

## Server-Side Rendering (SSR)

### Setup SSR Server

```bash
# Install SSR dependencies
yarn add @inertiajs/server
```

```javascript
// ssr/server.js
import { createServer } from '@inertiajs/server'
import { createInertiaApp } from '@inertiajs/react'
import { renderToString } from 'react-dom/server'
import React from 'react'

createServer(page =>
  createInertiaApp({
    page,
    render: renderToString,
    resolve: name => {
      const pages = import.meta.glob('../pages/**/*.jsx', { eager: true })
      return pages[`../pages/${name}.jsx`]
    },
    setup: ({ App, props }) => <App {...props} />
  })
)
```

```json
// package.json
{
  "scripts": {
    "ssr:build": "vite build --ssr ssr/server.js",
    "ssr:serve": "node ssr/server.js"
  }
}
```

### Enable SSR in Rails

```ruby
# config/initializers/inertia_rails.rb
InertiaRails.configure do |config|
  config.ssr_enabled = true
  config.ssr_url = 'http://localhost:13714'
end
```

---

## Testing Strategies

### Component Testing (Vitest + React Testing Library)

```javascript
// app/frontend/pages/Articles/__tests__/Index.test.jsx
import { render, screen, fireEvent } from '@testing-library/react'
import { describe, it, expect, vi } from 'vitest'
import Index from '../Index'

describe('Articles Index', () => {
  const mockArticles = [
    { id: 1, title: 'First Article', excerpt: 'Excerpt 1', author_name: 'John' },
    { id: 2, title: 'Second Article', excerpt: 'Excerpt 2', author_name: 'Jane' }
  ]

  it('renders articles list', () => {
    render(<Index articles={mockArticles} pagination={{}} filters={{}} />)

    expect(screen.getByText('First Article')).toBeInTheDocument()
    expect(screen.getByText('Second Article')).toBeInTheDocument()
  })

  it('handles search input', async () => {
    const { container } = render(
      <Index articles={mockArticles} pagination={{}} filters={{}} />
    )

    const searchInput = container.querySelector('input[type="search"]')
    fireEvent.change(searchInput, { target: { value: 'test' } })

    expect(searchInput.value).toBe('test')
  })
})
```

### Integration Testing (RSpec + Capybara)

```ruby
RSpec.describe "Articles Management", type: :system, js: true do
  let(:user) { create(:user) }

  before do
    login_as(user)
    driven_by(:selenium_chrome_headless)
  end

  it "creates new article" do
    visit new_article_path

    fill_in "Title", with: "My New Article"
    fill_in "Body", with: "This is the article content"
    check "Publish immediately"

    click_button "Create Article"

    expect(page).to have_text("Article created successfully")
    expect(page).to have_text("My New Article")
  end

  it "searches articles", :js do
    create(:article, title: "Rails Guide")
    create(:article, title: "React Tutorial")

    visit articles_path

    fill_in "Search articles...", with: "Rails"

    # Wait for debounced search
    sleep 0.5

    expect(page).to have_text("Rails Guide")
    expect(page).not_to have_text("React Tutorial")
  end
end
```

---

## Performance Optimization

### Lazy Loading Pages

```javascript
// app/frontend/entrypoints/application.js
createInertiaApp({
  resolve: name => {
    // Lazy load instead of eager
    const pages = import.meta.glob('../pages/**/*.jsx')
    return pages[`../pages/${name}.jsx`]()
  }
})
```

### Partial Reloads

```ruby
# Only reload specific props
def index
  render inertia: 'Articles/Index', props: {
    articles: -> { Article.all.as_json },  # Always include
    filters: -> { params[:filters] },       # Always include
    categories: inertia.lazy(-> {           # Load only when requested
      Category.all.as_json
    })
  }
end
```

```jsx
// Request specific data
router.reload({ only: ['articles', 'categories'] })
```

### Prefetching

```jsx
import { router } from '@inertiajs/react'

function ArticleLink({ article }) {
  return (
    <Link
      href={`/articles/${article.id}`}
      onMouseEnter={() => {
        router.prefetch(`/articles/${article.id}`)
      }}
    >
      {article.title}
    </Link>
  )
}
```

### Memory Management

```jsx
import { router } from '@inertiajs/react'
import { useEffect } from 'react'

export default function MyPage() {
  useEffect(() => {
    // Cleanup on unmount
    return () => {
      // Clear caches, cancel requests, etc.
    }
  }, [])

  // Remember scroll position
  router.remember({
    data: someState,
    scrollPosition: window.scrollY
  })
}
```

---

## Best Practices Summary

### ✅ Always Do

1. **Use Inertia Links** - Never use `<a>` tags
2. **Server-side validation** - Client validation is UX only
3. **Share common data** - Auth, flash, etc.
4. **Preserve scroll** - On form submissions
5. **Handle loading states** - Show feedback to users
6. **Use layouts** - Keep nav/footer consistent
7. **Test both frontend and backend** - Full coverage
8. **Lazy load pages** - Better performance
9. **Use TypeScript** - Type safety for props
10. **Debounce searches** - Reduce server load

### ❌ Never Do

1. **Don't use window.location** - Breaks Inertia
2. **Don't create REST APIs** - Inertia doesn't need them
3. **Don't fetch data client-side** - Server provides all data
4. **Don't store app state globally** - Use page props
5. **Don't bypass Inertia router** - Breaks SPA behavior
6. **Don't trust client validation** - Always validate server-side
7. **Don't skip error handling** - Handle all edge cases
8. **Don't ignore flash messages** - Show user feedback
9. **Don't forget CSRF** - Rails handles it automatically
10. **Don't over-complicate** - Inertia is meant to be simple

---

**Remember**: Inertia.js is a bridge, not a framework. It connects your Rails backend with React/Vue/Svelte frontend while keeping the simplicity of server-side routing and the interactivity of SPAs.
