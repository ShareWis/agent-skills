---
name: ruby-on-rails-best-practices
description: Ruby on Rails architecture and coding patterns from Basecamp. Use when writing, reviewing, or refactoring Rails code to follow proven conventions for models, controllers, jobs, and concerns. Triggers on tasks involving Rails models, concerns, controllers, background jobs, or Turbo/Hotwire.
---

# Ruby on Rails Best Practices

Architecture patterns and coding conventions extracted from Basecamp's production Rails applications (Fizzy and Campfire). Contains 16 rules across 8 categories focused on code organization, maintainability, and following "The Rails Way" with Basecamp's refinements.

## When to Apply

Reference these guidelines when:

- Organizing models, concerns, and controllers
- Writing background jobs
- Implementing real-time features with Turbo Streams
- Deciding where code should live
- Writing tests for Rails applications
- Reviewing Rails code for architectural consistency

## Project Conventions First (Required)

Do not copy examples blindly. Detect and follow the project's existing conventions first, then apply these rules by intent.

### Precedence

1. Project conventions in the current codebase
2. This skill's architectural guidance
3. Example snippets in this skill

### Required Discovery Before Changes

Before implementing, inspect how the project currently handles:

- Request context and auth (`current_user`, `Current`, Devise/Warden, etc.)
- Tenant/account scoping patterns
- Test data patterns (fixtures vs factories)
- Service object usage vs model concern usage
- Controller and routing conventions
- Job error handling and queue configuration

### Adapt by Intent

- Keep the rule intent, adapt the API names to the project.
- Example: if the app uses `current_user`, do not replace it with `Current.account` just because an example uses it.
- Example: if the app standard is FactoryBot, apply testing structure guidance without forcing a fixture migration in unrelated work.

## Rules Summary

### Model Organization (HIGH)

#### model-scoped-concerns - @rules/model-scoped-concerns.md

Place model-specific concerns in `app/models/model_name/` not `app/models/concerns/`.

```ruby
# Directory structure
app/models/
├── card.rb
├── card/
│   ├── closeable.rb     # Card::Closeable
│   ├── searchable.rb    # Card::Searchable
│   └── assignable.rb    # Card::Assignable

# app/models/card.rb
class Card < ApplicationRecord
  include Closeable, Searchable, Assignable
  # Ruby resolves from Card:: namespace first
end
```

#### concern-naming - @rules/concern-naming.md

Use `-able` suffix for behavior concerns, nouns for feature concerns.

```ruby
# Behaviors: -able suffix
module Card::Closeable     # Can be closed
module Card::Searchable    # Can be searched
module User::Mentionable   # Can be mentioned

# Features: nouns
module User::Avatar        # Has avatar
module User::Role          # Has role
module Card::Mentions      # Has @mentions
```

#### template-method-concerns - @rules/template-method-concerns.md

Use template methods in shared concerns for customizable behavior.

```ruby
# app/models/concerns/searchable.rb (shared)
module Searchable
  def search_title
    raise NotImplementedError
  end
end

# app/models/card/searchable.rb (model-specific)
module Card::Searchable
  include ::Searchable

  def search_title
    title  # Implement the hook
  end
end
```

### Background Jobs (HIGH)

#### paired-async-methods - @rules/paired-async-methods.md

Pair sync methods with `_later` variants that enqueue jobs.

```ruby
# app/models/card/readable.rb
def remove_inaccessible_notifications
  # Sync implementation
end

private
  def remove_inaccessible_notifications_later
    Card::RemoveInaccessibleNotificationsJob.perform_later(self)
  end

# app/jobs/card/remove_inaccessible_notifications_job.rb
class Card::RemoveInaccessibleNotificationsJob < ApplicationJob
  def perform(card)
    card.remove_inaccessible_notifications
  end
end
```

#### thin-jobs - @rules/thin-jobs.md

Jobs call model methods. All logic lives in models.

```ruby
# Bad: Logic in job
class ProcessOrderJob < ApplicationJob
  def perform(order)
    order.items.each { |i| i.product.decrement!(:stock) }
    order.update!(status: :processing)
  end
end

# Good: Job delegates to model
class ProcessOrderJob < ApplicationJob
  def perform(order)
    order.process  # Single method call
  end
end
```

### Controllers (HIGH)

#### resource-controllers - @rules/resource-controllers.md

Create resource controllers for state changes, not custom actions.

```ruby
# Bad: Custom actions
resources :cards do
  post :close
  post :reopen
end

# Good: Resource controllers
resources :cards do
  resource :closure, only: [:create, :destroy]
end

# app/controllers/cards/closures_controller.rb
class Cards::ClosuresController < ApplicationController
  def create
    @card.close
  end

  def destroy
    @card.reopen
  end
end
```

#### scoping-concerns - @rules/scoping-concerns.md

Use concerns like `CardScoped` for nested resource setup.

```ruby
# app/controllers/concerns/card_scoped.rb
module CardScoped
  extend ActiveSupport::Concern

  included do
    before_action :set_card
  end

  private
    def set_card
      @card = Current.user.accessible_cards.find_by!(number: params[:card_id])
    end
end

# Usage
class Cards::CommentsController < ApplicationController
  include CardScoped
end
```

#### thin-controllers - @rules/thin-controllers.md

Controllers stay thin and delegate business logic to service objects.

```ruby
# Good: Thin controller, delegated service
class Cards::ClosuresController < ApplicationController
  include CardScoped

  def create
    Cards::Close.call(card: @card, actor: current_user)
  end
end
```

### Request Context (MEDIUM)

#### current-attributes - @rules/current-attributes.md

Use `Current` for request-scoped data with cascading setters.

```ruby
class Current < ActiveSupport::CurrentAttributes
  attribute :session, :user, :account

  def session=(value)
    super(value)
    self.user = session&.user
  end
end
```

#### current-in-other-contexts - @rules/current-in-other-contexts.md

`Current` is only auto-populated in web requests. Jobs, mailers, and channels need explicit setup.

```ruby
# Jobs: extend ActiveJob to serialize/restore Current.account
# Mailers from jobs: wrap in Current.with_account { mailer.deliver }
# Channels: set Current in Connection#connect
```

### Associations & Callbacks (MEDIUM)

#### association-extensions - @rules/association-extensions.md

Choose between association extensions and model class methods based on context needs.

```ruby
# Use extension when you need parent context (proxy_association.owner)
has_many :accesses do
  def grant_to(users)
    board = proxy_association.owner
    Access.insert_all(users.map { |u| { user_id: u.id, board_id: board.id, account_id: board.account_id } })
  end
end

# Use class method when operation is independent
class Access
  def self.grant(board:, users:)
    insert_all(users.map { |u| { user_id: u.id, board_id: board.id } })
  end
end
```

#### callbacks-patterns - @rules/callbacks-patterns.md

Use `after_commit` for jobs, inline lambdas for simple ops.

```ruby
# Jobs: after_commit
after_create_commit :notify_recipients_later

# Simple ops: inline lambda
after_save -> { board.touch }, if: :published?

# Conditional: remember and check pattern
before_update :remember_changes
after_update_commit :process_changes, if: :should_process?
```

### Turbo & Real-time (MEDIUM)

#### turbo-broadcasts - @rules/turbo-broadcasts.md

Explicit broadcasts from controllers, not callbacks.

```ruby
# app/models/message/broadcasts.rb
module Message::Broadcasts
  def broadcast_create
    broadcast_append_to room, :messages, target: [room, :messages]
  end
end

# Controller calls explicitly
def create
  @message = @room.messages.create!(message_params)
  @message.broadcast_create
end
```

### Testing (MEDIUM)

#### fixtures-testing - @rules/fixtures-testing.md

Use fixtures, not factories. Mirror concern structure in tests.

```ruby
# test/fixtures/cards.yml
logo:
  title: The logo isn't big enough
  board: writebook
  creator: david

# test/models/card/closeable_test.rb
class Card::CloseableTest < ActiveSupport::TestCase
  test "close creates closure" do
    card = cards(:logo)
    assert_difference -> { Closure.count } do
      card.close
    end
  end
end
```

### Code Organization (LOW-MEDIUM)

#### nested-service-objects - @rules/nested-service-objects.md

Place service objects under model namespace, not `app/services`.

```ruby
# Good: app/models/card/activity_spike/detector.rb
class Card::ActivitySpike::Detector
  def initialize(card)
    @card = card
  end

  def detect
    # ...
  end
end
```

#### code-style - @rules/code-style.md

Prefer expanded conditionals, order methods by invocation.

```ruby
# Expanded conditionals
def find_record
  if record = find_by_id(id)
    record
  else
    NullRecord.new
  end
end

# Method ordering: caller before callees
def process
  step_one
  step_two
end

private
  def step_one; end
  def step_two; end
```

## Philosophy

These patterns embody a pragmatic DDD-Lite Rails style:

1. **Thin models, thin controllers** - Models focus on data/persistence; controllers orchestrate request/response
2. **Service object layer for business logic** - Domain behavior is isolated in service objects
3. **Co-located code** - Concerns, jobs, and services near the models they serve
4. **Explicit over implicit** - Call broadcasts explicitly, not via callbacks
5. **Convention over configuration** - Follow naming patterns for predictability
6. **Callable service style** - Prefer `XXX.new(a: 1, b: 2).call` with keyword/hash-style arguments (and optional `.call` class wrapper)
7. **Single Responsibility Principle** - Each service object should own one business use case

## Complete Sub-Rules and Purposes

This catalog lists every sub-rule from `@rules/*.md` with its purpose.

### Model Organization (HIGH)
#### model-scoped-concerns - `@rules/model-scoped-concerns.md`
- Purpose: Place concerns specific to a single model in a subdirectory named after the model (`app/models/model_name/`), not in the shared `app/models/concerns/` directory.
- Sub-rules:
  - Default to model-scoped concerns (`app/models/model_name/concern.rb`)
  - Name concerns using the model namespace (`Card::Closeable`, not `CardCloseable`)
  - Include without namespace prefix - Ruby resolves `Card::Closeable` automatically
  - Use shared concerns only for true cross-model abstractions or templates
  - Nest service objects and value objects under the model namespace too

#### concern-naming - `@rules/concern-naming.md`
- Purpose: Use consistent naming patterns for concerns that communicate their purpose at a glance.
- Sub-rules:
  - Prefer `-able` suffix for behaviors and capabilities
  - Use the model namespace prefix (`Card::`, not `Card` prefix)
  - Keep names short - one word when possible
  - Be consistent across the codebase
  - Names should be guessable - a developer should be able to find concerns without searching

#### template-method-concerns - `@rules/template-method-concerns.md`
- Purpose: When multiple models need similar but not identical behavior, create a shared concern with template methods (hooks) that model-specific concerns override.
- Sub-rules:
  - Place shared template in `app/models/concerns/`
  - Place model-specific implementations in `app/models/model_name/`
  - Use `include ::Searchable` (with `::`) to reference the shared concern
  - Define required template methods that raise `NotImplementedError`
  - Provide sensible defaults for optional template methods
  - Use `_was_created` or similar hooks for post-action customization

### Background Jobs (HIGH)
#### paired-async-methods - `@rules/paired-async-methods.md`
- Purpose: When a model method needs to run asynchronously, create a paired method with the `_later` suffix that enqueues a job calling the synchronous version.
- Sub-rules:
  - Name async methods with `_later` suffix
  - Keep jobs thin - they only call the model method
  - The synchronous method contains all business logic
  - Use `_now` suffix when the async version is the default (e.g., from callbacks)
  - Make `_later` private if only used from callbacks
  - For jobs that deserialize Active Record arguments, add `discard_on ActiveJob::DeserializationError` to handle deleted records

#### thin-jobs - `@rules/thin-jobs.md`
- Purpose: Jobs should be thin wrappers that receive records and call model methods. All business logic belongs in the model layer.
- Sub-rules:
  - Jobs call one method on the received record
  - All business logic lives in models
  - Jobs handle only job-specific concerns (queues, retries, error handling)
  - Namespace jobs to mirror model structure
  - For jobs that deserialize Active Record arguments, include `discard_on ActiveJob::DeserializationError`

### Controllers (HIGH)
#### resource-controllers - `@rules/resource-controllers.md`
- Purpose: When an action doesn't map cleanly to standard CRUD verbs, introduce a new resource rather than adding custom actions. Every controller action should be one of: index, show, new, create, edit, update, destroy.
- Sub-rules:
  - Controllers only have standard CRUD actions
  - State changes become singular resources (`resource :closure`)
  - User-specific toggles are resources scoped to the parent
  - Movement/position changes are their own resources
  - Nest controllers under the parent resource (`Cards::ClosuresController`)
  - Use `module` in routes to organize without deep URL nesting

#### scoping-concerns - `@rules/scoping-concerns.md`
- Purpose: Extract parent resource lookup and authorization into reusable controller concerns like `BoardScoped`, `CardScoped`, etc. Controllers include these concerns to get consistent before_action setup.
- Sub-rules:
  - Create concerns for each parent resource (`BoardScoped`, `CardScoped`, `RoomScoped`)
  - Always scope resource lookups through `Current.user`
  - Include shared helpers for common operations (rendering, capturing state)
  - Concerns can compose other concerns (`include FilterScoped`)
  - Use `find_by!` with a custom param (like `number:`) if needed
  - Authorization checks belong in the concern (`ensure_permission_to_admin_board`)

#### thin-controllers - `@rules/thin-controllers.md`
- Purpose: In a DDD-Lite Rails architecture, controllers and ActiveRecord models stay thin. Controllers orchestrate HTTP concerns, models manage persistence/data concerns, and service objects contain business behavior.
- Sub-rules:
  - Keep controllers thin: parse params, authorize, call service, render response
  - Keep models thin: data structure, persistence, and query concerns
  - Put business behavior in service objects (DDD-Lite use cases)
  - Co-locate services under model namespaces for discoverability
  - Keep controller actions short (typically 1-5 lines of core logic)
  - Use scoping concerns to centralize resource lookup and authorization boundaries
  - Prefer callable service style with keyword arguments (`XXX.new(a: 1, b: 2).call`)
  - Apply Single Responsibility Principle to every service object

### Request Context (MEDIUM)
#### current-attributes - `@rules/current-attributes.md`
- Purpose: Use `ActiveSupport::CurrentAttributes` to store request-scoped data like the current user, account, and request metadata. Design attribute setters to cascade related values.
- Sub-rules:
  - Define `Current` as a subclass of `ActiveSupport::CurrentAttributes`
  - Use cascading setters to automatically set related attributes
  - Set Current values in controller concerns, not individual actions
  - Use `Current.user` for default values in models
  - Always reset Current in tests (teardown)
  - Access Current through the class, never store references to attributes

#### current-in-other-contexts - `@rules/current-in-other-contexts.md`
- Purpose: `Current` is only auto-populated in web requests. Jobs, mailers called from jobs, and ActionCable channels run in separate contexts where `Current` starts empty. Each context needs explicit setup.
- Sub-rules:
  - Jobs: Extend ActiveJob to serialize `Current.account` at enqueue and restore it with type validation at perform
  - Mailers from jobs: Wrap mailer calls in `Current.with_account { ... }`
  - Channels: Set Current in `Connection#connect`
  - Don't serialize `Current.user` in jobs - pass users explicitly as arguments
  - Use `Current.with_account` for temporary context changes
  - Remember: if `Current.account` is `nil` unexpectedly, you're probably in a non-request context

### Associations & Callbacks (MEDIUM)
#### association-extensions - `@rules/association-extensions.md`
- Purpose: Choose between association extensions and model class methods based on whether the operation needs parent context.
- Sub-rules:
  - **Need parent context?** Use association extension
  - **Independent operation?** Use model class method
  - **Want both?** Extension can delegate to class method
  - Access parent via `proxy_association.owner`
  - Choose based on how the code reads at the call site

#### callbacks-patterns - `@rules/callbacks-patterns.md`
- Purpose: Use callbacks strategically with consistent patterns. Prefer `after_*_commit` for async work, inline lambdas for simple operations, and the "remember and check" pattern for conditional callbacks.
- Sub-rules:
  - Use `after_*_commit` for any async work or external effects
  - Use inline lambdas for simple touch/update operations
  - Use "remember and check" when you need to detect changes in `before_*` but act in `after_commit`
  - Use `saved_change_to_*` in `after_commit` callbacks (not `*_changed?`)
  - Keep callbacks focused - one callback, one purpose
  - Define custom callbacks with `define_callbacks` for domain-specific lifecycle events

### Turbo & Real-time (MEDIUM)
#### turbo-broadcasts - `@rules/turbo-broadcasts.md`
- Purpose: Encapsulate broadcast logic in model concerns and call broadcasts explicitly from controllers. Use composite stream names for targeting specific audiences.
- Sub-rules:
  - Encapsulate broadcast logic in model concerns
  - Call broadcasts explicitly from controllers
  - Use composite stream names (`[room, :messages]`) for scoping
  - Render once, broadcast to many for multi-user broadcasts
  - Use `method: :morph` for smart DOM updates
  - Don't use callbacks for broadcasts (be explicit)
  - Custom attributes can signal client-side behavior

### Testing (MEDIUM)
#### fixtures-testing - `@rules/fixtures-testing.md`
- Purpose: Use Rails fixtures instead of factories (FactoryBot). Create a comprehensive fixture set that represents your domain, with deterministic IDs for predictable ordering.
- Sub-rules:
  - Use fixtures, not factories
  - Create a coherent dataset that represents real usage
  - Use deterministic IDs for predictable ordering
  - Reference fixtures by name in tests
  - Set up `Current` attributes in test setup
  - Mirror concern location in test file structure
  - Use `assert_enqueued_with` for job testing
  - Use `assert_turbo_stream` for Turbo response testing

### Code Organization (LOW-MEDIUM)
#### nested-service-objects - `@rules/nested-service-objects.md`
- Purpose: When you need service objects, value objects, or plain Ruby classes, place them under the model namespace they operate on rather than in a separate `app/services` directory.
- Sub-rules:
  - Place service objects under the model namespace they operate on
  - Use descriptive class names (`Detector`, `Commenter`, `Pusher`)
  - Keep the interface simple - usually `initialize` + one public method
  - Value objects are also nested under the model namespace
  - No separate `app/services` directory
  - If it doesn't clearly belong to one model, it might belong in the model it creates/modifies

#### code-style - `@rules/code-style.md`
- Purpose: Follow consistent code style conventions for readable, maintainable Ruby code. These patterns are inspired by Basecamp's coding style.
- Sub-rules:
  - Prefer expanded conditionals over guard clauses
  - Order methods by invocation hierarchy
  - Indent under `private`, no newline after modifier
  - Only use `!` suffix when a non-bang method exists
  - Break long lines at natural points
  - Use modern hash syntax
  - Use `do...end` for multi-line blocks
  - Name predicate methods with `?` suffix
  - Avoid negated method names

