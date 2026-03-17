---
name: coding-standards
description: Universal coding standards, best practices, and patterns for TypeScript, JavaScript, React, and Node.js development.
origin: https://github.com/affaan-m/everything-claude-code
---

Universal coding standards applicable across all projects. Code snippets use TypeScript/JavaScript, but the principles (readability, immutability, error handling, naming, structure) apply to any language or stack.

## When to Activate

- Starting a new project or module
- Reviewing code for quality and maintainability
- Refactoring existing code to follow conventions
- Enforcing naming, formatting, or structural consistency
- Setting up linting, formatting, or type-checking rules
- Onboarding new contributors to coding conventions

## Code Quality Principles

### 1. Readability First
- Code is read more than written
- Clear variable and function names
- Self-documenting code preferred over comments
- Consistent formatting

### 2. KISS (Keep It Simple, Stupid)
- Simplest solution that works
- Avoid over-engineering
- No premature optimization
- Easy to understand > clever code

### 3. DRY (Don't Repeat Yourself)
- Extract common logic into functions
- Create reusable components
- Share utilities across modules
- Avoid copy-paste programming

### 4. YAGNI (You Aren't Gonna Need It)
- Don't build features before they're needed
- Avoid speculative generality
- Add complexity only when required
- Start simple, refactor when needed

## Naming

### Variables
```typescript
// Good: Descriptive names
const marketSearchQuery = 'election'
const isUserAuthenticated = true
const totalRevenue = 1000

// Bad: Unclear names
const q = 'election'
const flag = true
const x = 1000
```

### Functions — verb-noun pattern
```typescript
// Good
async function fetchMarketData(marketId: string) { }
function calculateSimilarity(a: number[], b: number[]) { }
function isValidEmail(email: string): boolean { }

// Bad
async function market(id: string) { }
function similarity(a, b) { }
```

## Immutability (CRITICAL)

```typescript
// Always use spread / non-mutating operations
const updatedUser = { ...user, name: 'New Name' }
const updatedArray = [...items, newItem]

// Never mutate directly
user.name = 'New Name'  // Bad
items.push(newItem)     // Bad
```

## Error Handling

```typescript
// Good: Explicit status check, informative rethrow
async function fetchData(url: string) {
  try {
    const response = await fetch(url)
    if (!response.ok) {
      throw new Error(`HTTP ${response.status}: ${response.statusText}`)
    }
    return await response.json()
  } catch (error) {
    console.error('Fetch failed:', error)
    throw new Error('Failed to fetch data')
  }
}

// Bad: Silent failure, no status check
async function fetchData(url) {
  const response = await fetch(url)
  return response.json()
}
```

## Async

```typescript
// Good: Parallel when independent
const [users, markets, stats] = await Promise.all([
  fetchUsers(),
  fetchMarkets(),
  fetchStats()
])

// Bad: Sequential when unnecessary
const users = await fetchUsers()
const markets = await fetchMarkets()
const stats = await fetchStats()
```

## Type Safety

```typescript
// Good: Proper types, union literals
interface Market {
  id: string
  name: string
  status: 'active' | 'resolved' | 'closed'
  created_at: Date
}

// Bad: any
function getMarket(id: any): Promise<any> { }
```

## API Design

### REST conventions
```
GET    /api/markets              # List
GET    /api/markets/:id          # Get one
POST   /api/markets              # Create
PUT    /api/markets/:id          # Full update
PATCH  /api/markets/:id          # Partial update
DELETE /api/markets/:id          # Delete
GET /api/markets?status=active&limit=10&offset=0
```

### Response shape
```typescript
interface ApiResponse<T> {
  success: boolean
  data?: T
  error?: string
  meta?: { total: number; page: number; limit: number }
}
```

### Input validation — validate at boundaries
```typescript
import { z } from 'zod'

const CreateMarketSchema = z.object({
  name: z.string().min(1).max(200),
  description: z.string().min(1).max(2000),
  endDate: z.string().datetime(),
  categories: z.array(z.string()).min(1)
})
```

## Comments

```typescript
// Good: Explain WHY
// Exponential backoff to avoid overwhelming the API during outages
const delay = Math.min(1000 * Math.pow(2, retryCount), 30000)

// Bad: Stating the obvious
count++  // Increment counter by 1
```

## Performance

- Memoize expensive derived values; avoid recomputing on every render/call
- Lazy-load heavy modules
- Select only the columns you need from the database — never `SELECT *`

## Testing — AAA Pattern

```typescript
test('calculates similarity correctly', () => {
  // Arrange
  const vector1 = [1, 0, 0]
  const vector2 = [0, 1, 0]
  // Act
  const similarity = calculateCosineSimilarity(vector1, vector2)
  // Assert
  expect(similarity).toBe(0)
})
```

## Code Smell Detection

1. **Long functions** — split functions > 50 lines into smaller ones
2. **Deep nesting** — use early returns instead of 5+ levels
3. **Magic numbers** — use named constants (`MAX_RETRIES`, `DEBOUNCE_DELAY_MS`)
