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
