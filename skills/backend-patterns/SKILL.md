---
name: backend-patterns
description: Backend architecture patterns, API design, database optimization, and server-side best practices for Node.js, Express, and Next.js API routes.
origin: https://github.com/affaan-m/everything-claude-code
---

# Backend Development Patterns

Backend architecture patterns and best practices for scalable server-side applications. Code snippets use TypeScript/Node.js, but the underlying patterns (repository, service layer, caching, error handling, rate limiting) apply to any backend stack.

## When to Activate

- Designing REST or GraphQL API endpoints
- Implementing repository, service, or controller layers
- Optimizing database queries (N+1, indexing, connection pooling)
- Adding caching (Redis, in-memory, HTTP cache headers)
- Setting up background jobs or async processing
- Structuring error handling and validation for APIs
- Building middleware (auth, logging, rate limiting)

## Layered Architecture

### Repository Pattern — isolate data access
```typescript
interface MarketRepository {
  findAll(filters?: MarketFilters): Promise<Market[]>
  findById(id: string): Promise<Market | null>
  create(data: CreateMarketDto): Promise<Market>
  update(id: string, data: UpdateMarketDto): Promise<Market>
  delete(id: string): Promise<void>
}

class SupabaseMarketRepository implements MarketRepository {
  async findAll(filters?: MarketFilters): Promise<Market[]> {
    let query = supabase.from('markets').select('*')
    if (filters?.status) { query = query.eq('status', filters.status) }
    if (filters?.limit) { query = query.limit(filters.limit) }
    const { data, error } = await query
    if (error) throw new Error(error.message)
    return data
  }
}
```

### Service Layer — business logic lives here, not in controllers
```typescript
class MarketService {
  constructor(private marketRepo: MarketRepository) {}

  async searchMarkets(query: string, limit: number = 10): Promise<Market[]> {
    const embedding = await generateEmbedding(query)
    const results = await this.vectorSearch(embedding, limit)
    const markets = await this.marketRepo.findByIds(results.map(r => r.id))
    return markets.sort((a, b) => {
      const scoreA = results.find(r => r.id === a.id)?.score || 0
      const scoreB = results.find(r => r.id === b.id)?.score || 0
      return scoreA - scoreB
    })
  }
}
```

### Middleware Pattern
```typescript
export function withAuth(handler: NextApiHandler): NextApiHandler {
  return async (req, res) => {
    const token = req.headers.authorization?.replace('Bearer ', '')
    if (!token) return res.status(401).json({ error: 'Unauthorized' })
    try {
      const user = await verifyToken(token)
      req.user = user
      return handler(req, res)
    } catch {
      return res.status(401).json({ error: 'Invalid token' })
    }
  }
}
```

## Database Patterns

### Select only what you need
```typescript
// Good
const { data } = await supabase
  .from('markets')
  .select('id, name, status, volume')
  .eq('status', 'active')
  .order('volume', { ascending: false })
  .limit(10)

// Bad — fetches all columns
const { data } = await supabase.from('markets').select('*')
```

### N+1 Prevention — batch fetch, then map
```typescript
// Bad: N queries inside a loop
for (const market of markets) {
  market.creator = await getUser(market.creator_id)
}

// Good: 1 batch query
const creatorIds = markets.map(m => m.creator_id)
const creators = await getUsers(creatorIds)
const creatorMap = new Map(creators.map(c => [c.id, c]))
markets.forEach(m => { m.creator = creatorMap.get(m.creator_id) })
```

### Transactions — use DB-level atomicity for multi-step writes
```typescript
async function createMarketWithPosition(marketData, positionData) {
  const { data, error } = await supabase.rpc('create_market_with_position', {
    market_data: marketData,
    position_data: positionData
  })
  if (error) throw new Error('Transaction failed')
  return data
}
```

## Caching

### Cache-aside with invalidation
```typescript
class CachedMarketRepository implements MarketRepository {
  constructor(private baseRepo: MarketRepository, private redis: RedisClient) {}

  async findById(id: string): Promise<Market | null> {
    const cached = await this.redis.get(`market:${id}`)
    if (cached) return JSON.parse(cached)
    const market = await this.baseRepo.findById(id)
    if (market) await this.redis.setex(`market:${id}`, 300, JSON.stringify(market))
    return market
  }

  async invalidateCache(id: string): Promise<void> {
    await this.redis.del(`market:${id}`)
  }
}
```

## Error Handling

### Typed application errors
```typescript
class ApiError extends Error {
  constructor(
    public statusCode: number,
    public message: string,
    public isOperational = true
  ) {
    super(message)
    Object.setPrototypeOf(this, ApiError.prototype)
  }
}

export function errorHandler(error: unknown): Response {
  if (error instanceof ApiError) {
    return NextResponse.json({ success: false, error: error.message }, { status: error.statusCode })
  }
  if (error instanceof z.ZodError) {
    return NextResponse.json({ success: false, error: 'Validation failed', details: error.errors }, { status: 400 })
  }
  console.error('Unexpected error:', error)
  return NextResponse.json({ success: false, error: 'Internal server error' }, { status: 500 })
}
```

### Retry with exponential backoff
```typescript
async function fetchWithRetry<T>(fn: () => Promise<T>, maxRetries = 3): Promise<T> {
  let lastError: Error
  for (let i = 0; i < maxRetries; i++) {
    try {
      return await fn()
    } catch (error) {
      lastError = error as Error
      if (i < maxRetries - 1) {
        await new Promise(resolve => setTimeout(resolve, Math.pow(2, i) * 1000))
      }
    }
  }
  throw lastError!
}
```

## Authentication & Authorization

### JWT validation
```typescript
export function verifyToken(token: string): JWTPayload {
  try {
    return jwt.verify(token, process.env.JWT_SECRET!) as JWTPayload
  } catch {
    throw new ApiError(401, 'Invalid token')
  }
}
```

### Role-based access control
```typescript
const rolePermissions: Record<User['role'], Permission[]> = {
  admin: ['read', 'write', 'delete', 'admin'],
  moderator: ['read', 'write', 'delete'],
  user: ['read', 'write']
}

export function requirePermission(permission: Permission) {
  return (handler) => async (request: Request) => {
    const user = await requireAuth(request)
    if (!hasPermission(user, permission)) throw new ApiError(403, 'Insufficient permissions')
    return handler(request, user)
  }
}
```

## Rate Limiting

```typescript
class RateLimiter {
  private requests = new Map<string, number[]>()

  async checkLimit(identifier: string, maxRequests: number, windowMs: number): Promise<boolean> {
    const now = Date.now()
    const recentRequests = (this.requests.get(identifier) || []).filter(t => now - t < windowMs)
    if (recentRequests.length >= maxRequests) return false
    recentRequests.push(now)
    this.requests.set(identifier, recentRequests)
    return true
  }
}
```

## Background Jobs

```typescript
class JobQueue<T> {
  private queue: T[] = []
  private processing = false

  async add(job: T): Promise<void> {
    this.queue.push(job)
    if (!this.processing) this.process()
  }

  private async process(): Promise<void> {
    this.processing = true
    while (this.queue.length > 0) {
      const job = this.queue.shift()!
      try { await this.execute(job) } catch (error) { console.error('Job failed:', error) }
    }
    this.processing = false
  }
}
```

## Logging

Emit structured JSON logs — easier to query in log aggregation tools.

```typescript
class Logger {
  log(level: 'info' | 'warn' | 'error', message: string, context?: LogContext) {
    console.log(JSON.stringify({
      timestamp: new Date().toISOString(),
      level,
      message,
      ...context
    }))
  }

  error(message: string, error: Error, context?: LogContext) {
    this.log('error', message, { ...context, error: error.message, stack: error.stack })
  }
}
```
