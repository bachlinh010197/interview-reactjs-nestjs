# 📘 Database Expert — Câu Hỏi Phỏng Vấn Nâng Cao

[⬅️ System Design](./11-system-design.md) | [Phần tiếp: Clean Architecture ➡️](./13-clean-architecture.md)

---

## 1. SQL vs NoSQL — Khi nào dùng gì?

**Câu hỏi:** So sánh SQL và NoSQL. Cách chọn database cho dự án?

**Trả lời:**

| Tiêu chí | SQL (PostgreSQL, MySQL) | NoSQL (MongoDB, DynamoDB) |
|---|---|---|
| Data model | Relational (tables, rows) | Document, Key-Value, Graph, Column |
| Schema | Fixed schema | Flexible / schemaless |
| Scaling | Vertical (scale up) | Horizontal (scale out) |
| Transactions | ACID (strong) | BASE (eventual consistency) |
| Query | SQL (powerful joins) | API-based (limited joins) |
| Consistency | Strong consistency | Eventual consistency (tunable) |

**Quy tắc chọn:**

| Dùng SQL khi | Dùng NoSQL khi |
|---|---|
| Data có relationships phức tạp | Data không có relationships rõ ràng |
| Cần ACID transactions (banking) | Cần horizontal scaling (millions writes/s) |
| Schema ổn định | Schema thay đổi thường xuyên |
| Complex queries (JOIN, GROUP BY) | Simple key-value lookups |
| Report, analytics | Real-time, high throughput |

```
E-commerce:
├── Users, Orders, Payments → PostgreSQL (ACID, relations)
├── Product Catalog → MongoDB (flexible attributes per category)
├── Shopping Cart → Redis (fast, temporary)
├── Search → Elasticsearch (full-text search)
├── Chat Messages → MongoDB (high write, flexible)
├── Session → Redis (fast read/write, TTL)
└── Analytics → ClickHouse (column-oriented, aggregations)
```

---

## 2. Database Indexing sâu

**Câu hỏi:** Giải thích các loại index. Khi nào tạo index? Khi nào index gây hại?

**Trả lời:**

**Các loại index trong PostgreSQL:**

| Loại | Dùng khi | Ví dụ |
|---|---|---|
| **B-Tree** (default) | Equality, range, sorting | `WHERE id = 1`, `ORDER BY created_at` |
| **Hash** | Chỉ equality | `WHERE email = 'x@y.com'` |
| **GIN** | Full-text search, JSONB, Array | `WHERE tags @> '{nestjs}'` |
| **GiST** | Geometric, range types | PostGIS spatial queries |
| **BRIN** | Large sequential data (timestamp) | Time-series data |
| **Partial** | Subset of rows | `WHERE status = 'active'` (chỉ index active rows) |
| **Composite** | Multi-column queries | `WHERE tenant_id = X AND created_at > Y` |

```sql
-- B-Tree: phổ biến nhất
CREATE INDEX idx_users_email ON users(email);

-- Composite Index: thứ tự columns quan trọng!
-- Query WHERE tenant_id = X AND status = 'active' ORDER BY created_at DESC
CREATE INDEX idx_orders_tenant_status_date
  ON orders(tenant_id, status, created_at DESC);
-- ✅ Covering: tenant_id trước (equality), sau đó status, cuối cùng sort

-- Partial Index: chỉ index rows cần thiết → nhỏ hơn, nhanh hơn
CREATE INDEX idx_orders_pending
  ON orders(created_at)
  WHERE status = 'pending';
-- Chỉ index orders đang pending, không phải tất cả millions of completed orders

-- GIN Index cho JSONB
CREATE INDEX idx_products_attrs ON products USING GIN(attributes);
-- SELECT * FROM products WHERE attributes @> '{"color": "red"}';

-- Expression Index
CREATE INDEX idx_users_email_lower ON users(LOWER(email));
-- SELECT * FROM users WHERE LOWER(email) = 'john@example.com';

-- Covering Index (INCLUDE) — tránh table lookup
CREATE INDEX idx_users_email_covering
  ON users(email) INCLUDE (name, avatar_url);
-- PostgreSQL đọc trực tiếp từ index, không cần đọc table
```

**Khi nào index gây hại?**
- **Write-heavy tables**: Mỗi INSERT/UPDATE phải update tất cả indexes
- **Low cardinality**: Column có ít giá trị unique (boolean, status) → index không hiệu quả
- **Small tables**: Dưới ~1000 rows → sequential scan nhanh hơn index scan
- **Quá nhiều indexes**: Tốn disk space, slow writes

```sql
-- Kiểm tra index usage
SELECT schemaname, tablename, indexname, idx_scan, idx_tup_read
FROM pg_stat_user_indexes
WHERE idx_scan = 0  -- Indexes chưa bao giờ được dùng → nên xóa!
ORDER BY pg_relation_size(indexrelid) DESC;

-- Analyze query plan
EXPLAIN ANALYZE SELECT * FROM orders WHERE user_id = 123;
-- Xem có dùng index không, cost bao nhiêu
```

---

## 3. Query Optimization

**Câu hỏi:** Cách tối ưu slow queries? Giải thích EXPLAIN ANALYZE.

**Trả lời:**

```sql
-- EXPLAIN ANALYZE output:
EXPLAIN ANALYZE
SELECT u.name, COUNT(o.id) as order_count, SUM(o.total) as total_spent
FROM users u
JOIN orders o ON u.id = o.user_id
WHERE o.created_at > '2024-01-01'
GROUP BY u.id
HAVING SUM(o.total) > 1000
ORDER BY total_spent DESC
LIMIT 10;

-- Đọc kết quả:
-- Seq Scan on orders    ← ❌ Sequential scan → cần index!
-- cost=0.00..15420.00   ← Estimated cost
-- actual time=0.5..120ms ← Thời gian thực tế
-- rows=50000            ← Số rows scan
-- Buffers: shared hit=1234 read=567  ← Cache hit ratio
```

**Các bước tối ưu:**

```sql
-- 1. Thêm index cho WHERE và JOIN conditions
CREATE INDEX idx_orders_created_at ON orders(created_at);
CREATE INDEX idx_orders_user_id ON orders(user_id);

-- 2. Composite index nếu query luôn dùng cả 2
CREATE INDEX idx_orders_user_date ON orders(user_id, created_at);

-- 3. Tránh SELECT * — chỉ select columns cần thiết
SELECT u.name, COUNT(o.id)  -- Không SELECT u.*, o.*

-- 4. Pagination: dùng cursor thay vì OFFSET
-- ❌ OFFSET lớn rất chậm (scan skip rows)
SELECT * FROM products ORDER BY id LIMIT 20 OFFSET 100000;

-- ✅ Cursor-based pagination
SELECT * FROM products
WHERE id > :last_seen_id  -- Dùng index, rất nhanh
ORDER BY id
LIMIT 20;

-- 5. Batch queries thay vì N+1
-- ❌ N+1: 1 query users + N queries orders
for user in users:
    SELECT * FROM orders WHERE user_id = user.id;

-- ✅ Batch: 2 queries total
SELECT * FROM users WHERE id IN (1,2,3...);
SELECT * FROM orders WHERE user_id IN (1,2,3...);

-- 6. Materialized View cho complex reports
CREATE MATERIALIZED VIEW monthly_sales AS
SELECT
  date_trunc('month', created_at) as month,
  product_id,
  SUM(quantity) as total_qty,
  SUM(total) as total_revenue
FROM orders
GROUP BY 1, 2;

-- Refresh định kỳ (e.g., mỗi giờ)
REFRESH MATERIALIZED VIEW CONCURRENTLY monthly_sales;
```

---

## 4. Database Transactions & Isolation Levels

**Câu hỏi:** Giải thích các isolation levels. Khi nào dùng level nào?

**Trả lời:**

| Isolation Level | Dirty Read | Non-repeatable Read | Phantom Read | Performance |
|---|---|---|---|---|
| **READ UNCOMMITTED** | ✅ Có thể | ✅ Có thể | ✅ Có thể | Nhanh nhất |
| **READ COMMITTED** (PostgreSQL default) | ❌ Không | ✅ Có thể | ✅ Có thể | Nhanh |
| **REPEATABLE READ** | ❌ Không | ❌ Không | ✅ Có thể | Trung bình |
| **SERIALIZABLE** | ❌ Không | ❌ Không | ❌ Không | Chậm nhất |

```typescript
// NestJS + TypeORM transaction với isolation level
@Injectable()
export class TransferService {
  async transferMoney(fromId: number, toId: number, amount: number) {
    return this.dataSource.transaction(
      'SERIALIZABLE', // Cao nhất cho banking
      async (manager) => {
        const from = await manager.findOne(Account, {
          where: { id: fromId },
          lock: { mode: 'pessimistic_write' },
        });
        const to = await manager.findOne(Account, {
          where: { id: toId },
          lock: { mode: 'pessimistic_write' },
        });

        if (from.balance < amount) {
          throw new BadRequestException('Insufficient funds');
        }

        from.balance -= amount;
        to.balance += amount;

        await manager.save([from, to]);

        // Audit log
        await manager.save(Transaction, {
          fromAccountId: fromId,
          toAccountId: toId,
          amount,
          type: 'transfer',
        });
      },
    );
  }
}
```

**Các anomalies:**
- **Dirty Read**: Đọc data chưa commit từ transaction khác
- **Non-repeatable Read**: Đọc lại cùng row → data khác (transaction khác đã commit)
- **Phantom Read**: Query lại → có thêm/bớt rows (transaction khác INSERT/DELETE)

---

## 5. Database Replication

**Câu hỏi:** Giải thích replication strategies. Cách implement read replica trong NestJS?

**Trả lời:**

| Strategy | Mô tả | Consistency |
|---|---|---|
| **Single-Leader** | 1 primary (write) + N replicas (read) | Eventual |
| **Multi-Leader** | Nhiều primary, mỗi cái accept writes | Conflict resolution cần |
| **Leaderless** | Bất kỳ node nào cũng read/write (Cassandra) | Quorum-based |

```typescript
// TypeORM read replica configuration
TypeOrmModule.forRoot({
  type: 'postgres',
  replication: {
    master: {
      host: 'primary.db.internal',
      port: 5432,
      username: 'app',
      password: process.env.DB_PASSWORD,
      database: 'myapp',
    },
    slaves: [
      { host: 'replica-1.db.internal', port: 5432, username: 'app_readonly', password: process.env.DB_PASSWORD, database: 'myapp' },
      { host: 'replica-2.db.internal', port: 5432, username: 'app_readonly', password: process.env.DB_PASSWORD, database: 'myapp' },
    ],
  },
});

// TypeORM tự động route: SELECT → replica, INSERT/UPDATE/DELETE → primary
// Hoặc force primary cho read-after-write consistency:
@Injectable()
export class UserService {
  async createAndReturn(dto: CreateUserDto): Promise<User> {
    const user = await this.userRepo.save(dto); // → primary

    // Đọc lại từ primary (không dùng replica → tránh replication lag)
    return this.userRepo
      .createQueryBuilder('user')
      .setQueryRunner(this.dataSource.createQueryRunner('master'))
      .where('user.id = :id', { id: user.id })
      .getOne();
  }
}
```

**Replication Lag:**
- Replica có thể chậm hơn primary vài ms → vài giây
- **Read-after-write consistency**: Sau khi user write → đọc từ primary thay vì replica
- **Monotonic reads**: Đảm bảo user không thấy data "quay ngược thời gian"

---

## 6. Connection Pooling & Performance

**Câu hỏi:** Tại sao cần connection pooling? Cách configure?

**Trả lời:**

**Vấn đề không có pool:** Mỗi request tạo 1 DB connection mới → overhead TCP handshake + SSL + authentication → ~50-100ms per connection.

**Connection Pool:** Tạo trước N connections, tái sử dụng → ~0ms per request.

```typescript
// TypeORM pooling config
TypeOrmModule.forRoot({
  type: 'postgres',
  extra: {
    // Pool size = (core_count * 2) + disk_count
    // Ví dụ: 4 cores, 1 disk → pool = 9
    max: 20,                  // Max connections
    min: 5,                   // Min idle connections
    idleTimeoutMillis: 30000, // Close idle connection after 30s
    connectionTimeoutMillis: 5000, // Timeout khi không có connection available
    maxUses: 7500,            // Close connection after N uses (prevent leak)
  },
})

// PgBouncer — external connection pooler (production)
// App → PgBouncer (pool 1000 connections) → PostgreSQL (max 100 real connections)
// Mode:
//   - session: 1 client = 1 connection (giống không pooling)
//   - transaction: connection released sau mỗi transaction ✅
//   - statement: connection released sau mỗi statement (không hỗ trợ multi-statement transaction)
```

**Monitoring pool health:**
```typescript
@Injectable()
export class DatabaseHealthIndicator extends HealthIndicator {
  async isHealthy(): Promise<HealthIndicatorResult> {
    const pool = this.dataSource.driver.master;
    return this.getStatus('database', true, {
      totalConnections: pool.totalCount,
      idleConnections: pool.idleCount,
      waitingRequests: pool.waitingCount,
    });
  }
}
```

---

## 7. Data Modeling Patterns

**Câu hỏi:** Giải thích các patterns modeling data phổ biến.

**Trả lời:**

### Soft Delete

```typescript
// Không xóa thật — đánh dấu deletedAt
@Entity()
export class User {
  @DeleteDateColumn()
  deletedAt: Date | null;
}

// TypeORM tự động thêm WHERE deletedAt IS NULL vào mọi query
// Muốn query cả deleted: .withDeleted()
await this.userRepo.find(); // Chỉ active users
await this.userRepo.find({ withDeleted: true }); // Tất cả
await this.userRepo.softRemove(user); // SET deletedAt = NOW()
await this.userRepo.restore(user.id); // SET deletedAt = NULL
```

### Polymorphic Associations

```typescript
// Scenario: Comments cho cả Product, Post, Article
// Approach 1: Một bảng chung với type + reference_id
@Entity()
export class Comment {
  @Column()
  commentableType: 'product' | 'post' | 'article';

  @Column()
  commentableId: number;

  @Column()
  content: string;
}

// Approach 2: Separate join tables (normalized, type-safe)
@Entity()
export class ProductComment {
  @ManyToOne(() => Product) product: Product;
  @Column() content: string;
}

@Entity()
export class PostComment {
  @ManyToOne(() => Post) post: Post;
  @Column() content: string;
}
```

### Tree Structure (Hierarchical Data)

```typescript
// Category: Electronics > Phones > Smartphones
// Approach: Materialized Path
@Entity()
@Tree('materialized-path')
export class Category {
  @PrimaryGeneratedColumn()
  id: number;

  @Column()
  name: string;

  @TreeParent()
  parent: Category;

  @TreeChildren()
  children: Category[];

  // Tự động tạo column mpath: '1.3.7.'
  // Query con cháu: WHERE mpath LIKE '1.3.%'
}

// Adjacency List (simple)
// Closure Table (fast queries, tốn space)
// Nested Set (fast reads, slow writes)
```

### Event Sourcing

```typescript
// Lưu TẤT CẢ events thay vì current state
// State = replay tất cả events từ đầu

@Entity()
export class AccountEvent {
  @PrimaryGeneratedColumn()
  id: number;

  @Column()
  accountId: string;

  @Column()
  type: 'CREATED' | 'DEPOSITED' | 'WITHDRAWN' | 'TRANSFERRED';

  @Column('jsonb')
  data: Record<string, any>; // { amount: 100, to: 'acc-456' }

  @Column()
  version: number; // Optimistic concurrency

  @CreateDateColumn()
  occurredAt: Date;
}

// Rebuild state từ events
function buildAccountState(events: AccountEvent[]): Account {
  return events.reduce((state, event) => {
    switch (event.type) {
      case 'CREATED':
        return { ...state, balance: 0, status: 'active' };
      case 'DEPOSITED':
        return { ...state, balance: state.balance + event.data.amount };
      case 'WITHDRAWN':
        return { ...state, balance: state.balance - event.data.amount };
      default:
        return state;
    }
  }, {} as Account);
}
```

---

## 8. Redis Patterns nâng cao

**Câu hỏi:** Redis không chỉ là cache. Các use cases nâng cao?

**Trả lời:**

```typescript
// 1. Distributed Lock (Redlock)
@Injectable()
export class LockService {
  async withLock<T>(key: string, ttlMs: number, fn: () => Promise<T>): Promise<T> {
    const lockValue = randomUUID();
    const acquired = await this.redis.set(
      `lock:${key}`, lockValue, 'PX', ttlMs, 'NX',
    );

    if (!acquired) throw new ConflictException('Resource is locked');

    try {
      return await fn();
    } finally {
      // Chỉ release nếu lock vẫn thuộc về mình (Lua script atomic)
      const script = `
        if redis.call("get", KEYS[1]) == ARGV[1] then
          return redis.call("del", KEYS[1])
        else return 0 end
      `;
      await this.redis.eval(script, 1, `lock:${key}`, lockValue);
    }
  }
}

// Sử dụng
await this.lockService.withLock(`order:${orderId}`, 5000, async () => {
  await this.processPayment(orderId);
});

// 2. Leaderboard (Sorted Set)
@Injectable()
export class LeaderboardService {
  async addScore(userId: string, score: number) {
    await this.redis.zincrby('leaderboard:weekly', score, userId);
  }

  async getTopPlayers(limit = 10) {
    // ZREVRANGE: top scores descending
    return this.redis.zrevrange('leaderboard:weekly', 0, limit - 1, 'WITHSCORES');
  }

  async getUserRank(userId: string) {
    const rank = await this.redis.zrevrank('leaderboard:weekly', userId);
    return rank !== null ? rank + 1 : null; // 0-indexed → 1-indexed
  }
}

// 3. Rate Counter (Sliding Window)
async function isRateLimited(userId: string, limit: number, windowSec: number): Promise<boolean> {
  const key = `rate:${userId}`;
  const now = Date.now();
  const pipeline = this.redis.pipeline();
  pipeline.zremrangebyscore(key, 0, now - windowSec * 1000); // Remove old
  pipeline.zadd(key, now, `${now}:${Math.random()}`);        // Add current
  pipeline.zcard(key);                                         // Count
  pipeline.pexpire(key, windowSec * 1000);                    // TTL
  const results = await pipeline.exec();
  return results[2][1] > limit;
}

// 4. Pub/Sub — Real-time notifications
// Publisher
await this.redis.publish('notifications:user:123', JSON.stringify({
  type: 'new_message',
  data: { from: 'John', message: 'Hello!' },
}));

// Subscriber
const sub = this.redis.duplicate();
await sub.subscribe('notifications:user:123');
sub.on('message', (channel, message) => {
  const notification = JSON.parse(message);
  this.websocket.sendToUser('123', notification);
});

// 5. Session Storage
@Injectable()
export class SessionService {
  async createSession(userId: string, metadata: any): Promise<string> {
    const sessionId = randomUUID();
    await this.redis.setex(
      `session:${sessionId}`,
      86400, // 24h TTL
      JSON.stringify({ userId, ...metadata, createdAt: Date.now() }),
    );
    return sessionId;
  }

  async getSession(sessionId: string) {
    const data = await this.redis.get(`session:${sessionId}`);
    return data ? JSON.parse(data) : null;
  }

  // Revoke all sessions of a user (e.g., password changed)
  async revokeAllSessions(userId: string) {
    const keys = await this.redis.keys(`session:*`);
    for (const key of keys) {
      const session = JSON.parse(await this.redis.get(key));
      if (session?.userId === userId) await this.redis.del(key);
    }
  }
}
```

---

## 9. Migration Best Practices

**Câu hỏi:** Cách deploy database migration zero-downtime?

**Trả lời:**

**Nguyên tắc vàng:** Migration phải **backward compatible** — code cũ vẫn chạy được với schema mới.

```
❌ Dangerous: Rename column trong 1 step
ALTER TABLE users RENAME COLUMN name TO full_name;
→ Code cũ vẫn query SELECT name → ERROR!

✅ Safe: 3-step migration

Step 1 (Migration): Thêm column mới
  ALTER TABLE users ADD COLUMN full_name VARCHAR;
  UPDATE users SET full_name = name;

Step 2 (Deploy code): Code mới đọc/ghi cả 2 columns
  SELECT COALESCE(full_name, name) as full_name FROM users;
  UPDATE users SET name = :value, full_name = :value;

Step 3 (Migration): Xóa column cũ (sau khi tất cả instances dùng code mới)
  ALTER TABLE users DROP COLUMN name;
```

**Expand-Contract Pattern:**

```
Phase 1: EXPAND (add new)
├── Add new column/table
├── Backfill data
└── Deploy code writing to BOTH old + new

Phase 2: MIGRATE (switch)
├── Deploy code reading from NEW
└── Verify data integrity

Phase 3: CONTRACT (remove old)
├── Deploy code removing OLD writes
└── Drop old column/table
```
