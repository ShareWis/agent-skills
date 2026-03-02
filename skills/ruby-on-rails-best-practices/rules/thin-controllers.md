---
title: Keep Controllers and Models Thin with Service Objects
impact: HIGH
tags: [controllers, services, models, architecture, ddd-lite]
---

# Keep Controllers and Models Thin with Service Objects

In a DDD-Lite Rails architecture, controllers and ActiveRecord models stay thin. Controllers orchestrate HTTP concerns, models manage persistence/data concerns, and service objects contain business behavior.

## Why

- **Clear separation of concerns**: Delivery, behavior, and data responsibilities are isolated
- **Better testability**: Use-case logic is unit-testable in service objects
- **Safer refactoring**: Business workflows are not spread across controllers/callbacks
- **Readable code paths**: One service class usually maps to one business use case
- **Pragmatic DDD-Lite**: Gains structure without introducing full DDD complexity

## DDD-Lite Responsibility Split

- **Controllers**: Params, auth, response format, HTTP status codes
- **Models (ActiveRecord)**: Associations, validations, scopes, persistence helpers
- **Service objects**: Commands/use cases, transactions, orchestration, side effects

## Bad: Fat Controller

```ruby
class Cards::ClosuresController < ApplicationController
  def create
    card = current_user.accessible_cards.find(params[:card_id])
    return head :forbidden unless card.closeable?

    card.transaction do
      card.update!(status: :closed, closed_at: Time.current)
      card.closure.create!(user: current_user)
      card.events.create!(action: :closed, creator: current_user)
    end

    card.watchers.each do |watcher|
      NotificationMailer.card_closed(card, watcher).deliver_later
    end

    respond_to do |format|
      format.turbo_stream
      format.json { head :no_content }
    end
  end
end
```

Problems:

- Business behavior lives in delivery layer
- Transactions and side effects are duplicated across entry points
- Hard to reuse from jobs/CLI/internal workflows

## Bad: Fat Model

```ruby
class Card < ApplicationRecord
  def close!(actor:)
    return false unless closeable?

    transaction do
      update!(status: :closed, closed_at: Time.current)
      closure.create!(user: actor)
      events.create!(action: :closed, creator: actor)
      sync_to_external_system
      recalculate_board_counters
      notify_watchers_later
    end

    true
  end
end
```

Problems:

- Model mixes persistence with cross-cutting workflow orchestration
- Business use-case behavior is hard to see and evolve
- Service boundaries become implicit and fragile

## Good: Thin Controller + Service Object

```ruby
# app/controllers/cards/closures_controller.rb
class Cards::ClosuresController < ApplicationController
  include CardScoped

  def create
    ok = Card::Closure::Create.call(card: @card, actor: current_user)
    return head :unprocessable_entity unless ok

    respond_to do |format|
      format.turbo_stream
      format.json { head :no_content }
    end
  end

  def destroy
    ok = Card::Closure::Destroy.call(card: @card, actor: current_user)
    return head :unprocessable_entity unless ok

    respond_to do |format|
      format.turbo_stream
      format.json { head :no_content }
    end
  end
end
```

```ruby
# app/models/card/closure/create.rb
class Card::Closure::Create
  def self.call(...)
    new(...).call
  end

  def initialize(card:, actor:)
    @card = card
    @actor = actor
  end

  def call
    return false unless card.closeable?

    Card.transaction do
      card.update!(status: :closed, closed_at: Time.current)
      card.closure.create!(user: actor)
      card.events.create!(action: :closed, creator: actor)
    end

    Card::NotifyWatchersJob.perform_later(card)
    true
  end

  private
    attr_reader :card, :actor
end
```

```ruby
# app/models/card.rb
class Card < ApplicationRecord
  has_one :closure, dependent: :destroy
  has_many :events, dependent: :destroy

  scope :open, -> { where(status: :open) }

  def closeable?
    status != "closed"
  end
end
```

## Controller Action Patterns

### Simple CRUD

Direct ActiveRecord operations are still fine when no workflow orchestration is needed:

```ruby
class Cards::CommentsController < ApplicationController
  include CardScoped

  def create
    @comment = @card.comments.create!(comment_params)
  end
end
```

### Business State Changes

Use services for behavior-heavy commands:

```ruby
class Cards::AssignmentsController < ApplicationController
  include CardScoped

  def update
    Card::Assignment::Toggle.call(card: @card, actor: current_user)
  end
end
```

### Multi-Model Workflows

Keep orchestration in a use-case service:

```ruby
class BoardsController < ApplicationController
  def create
    @board = Board::Creation::Create.call(
      actor: current_user,
      attributes: board_params
    )

    redirect_to @board
  end
end
```

## Service Object Guidelines

1. Single Responsibility Principle: one class per business use case
2. Prefer callable style: `XXX.new(a: 1, b: 2).call`
3. Prefer keyword/hash-style inputs (`actor:`, `record:`, `attributes:`)
4. Transactions belong in the service when orchestrating multiple writes
5. Return a clear success/failure outcome (boolean, result object, or exception policy)
6. Optional convenience wrapper is fine: define `self.call(...) = new(...).call`

## Location and Namespacing

Co-locate service objects under the domain model namespace:

- `app/models/card/closure/create.rb` -> `Card::Closure::Create`
- `app/models/board/creation/create.rb` -> `Board::Creation::Create`
- `app/models/card/assignment/toggle.rb` -> `Card::Assignment::Toggle`

This keeps behavior discoverable near related data models while preserving a service-layer architecture.

## Model Responsibilities in DDD-Lite

Models should primarily contain:

- Schema-level validations and associations
- Query/scoping helpers
- Lightweight predicates and persistence helpers

Models should avoid:

- Multi-step use-case orchestration
- External API flow coordination
- Delivery-specific branching logic

## Rules

1. Keep controllers thin: parse params, authorize, call service, render response
2. Keep models thin: data structure, persistence, and query concerns
3. Put business behavior in service objects (DDD-Lite use cases)
4. Co-locate services under model namespaces for discoverability
5. Keep controller actions short (typically 1-5 lines of core logic)
6. Use scoping concerns to centralize resource lookup and authorization boundaries
7. Prefer callable service style with keyword arguments (`XXX.new(a: 1, b: 2).call`)
8. Apply Single Responsibility Principle to every service object
