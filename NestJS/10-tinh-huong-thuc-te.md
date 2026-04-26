# 📘 NestJS - Phần 10: Câu Hỏi Tình Huống & Tips

[⬅️ Testing](./09-testing.md) | [🏠 Mục lục](../README.md)

---

## 1. Thiết kế REST API cho hệ thống E-commerce

**Câu hỏi:** Bạn được yêu cầu thiết kế API cho hệ thống e-commerce. Mô tả cách bạn tổ chức?

**Trả lời:**

```
src/
├── modules/
│   ├── auth/           # Authentication module
│   ├── users/          # User management
│   ├── products/       # Product CRUD
│   ├── categories/     # Category management
│   ├── cart/           # Shopping cart
│   ├── orders/         # Order management
│   ├── payments/       # Payment processing
│   ├── notifications/  # Email/SMS notifications
│   └── uploads/        # File upload (images)
├── common/
│   ├── decorators/     # Custom decorators
│   ├── filters/        # Exception filters
│   ├── guards/         # Auth, Roles guards
│   ├── interceptors/   # Transform, Logging
│   ├── pipes/          # Validation pipes
│   └── dto/            # Shared DTOs (pagination, etc.)
├── config/             # Configuration files
├── database/           # Migrations, seeds
└── main.ts
```

**API Endpoints:**
```
POST   /auth/register
POST   /auth/login
POST   /auth/refresh

GET    /products?page=1&limit=10&category=electronics
GET    /products/:id
POST   /products          (Admin)
PUT    /products/:id       (Admin)
DELETE /products/:id       (Admin)

POST   /cart/items
PUT    /cart/items/:id
DELETE /cart/items/:id
GET    /cart

POST   /orders
GET    /orders
GET    /orders/:id
PATCH  /orders/:id/status  (Admin)
```

---

## 2. Cách xử lý Rate Limiting

**Trả lời:**

```typescript
// npm install @nestjs/throttler

@Module({
  imports: [
    ThrottlerModule.forRoot([
      { name: 'short', ttl: 1000, limit: 3 },
      { name: 'medium', ttl: 10000, limit: 20 },
      { name: 'long', ttl: 60000, limit: 100 },
    ]),
  ],
  providers: [
    { provide: APP_GUARD, useClass: ThrottlerGuard },
  ],
})
export class AppModule {}

// Custom rate limit cho specific endpoints
@Controller('auth')
export class AuthController {
  @Post('login')
  @Throttle({ short: { limit: 5, ttl: 60000 } }) // 5 attempts per minute
  login() {}

  @SkipThrottle() // Bỏ qua rate limit
  @Get('status')
  status() {}
}
```

---

## 3. Cách xử lý Long-running Tasks

**Trả lời:**

Dùng **Bull Queue** (Redis-based job queue):

```typescript
// npm install @nestjs/bull bull

@Module({
  imports: [
    BullModule.forRoot({ redis: { host: 'localhost', port: 6379 } }),
    BullModule.registerQueue({ name: 'email' }),
  ],
})
export class AppModule {}

// email.processor.ts
@Processor('email')
export class EmailProcessor {
  @Process('send-welcome')
  async handleWelcomeEmail(job: Job<{ email: string; name: string }>) {
    await this.mailService.sendWelcome(job.data);
  }

  @OnQueueFailed()
  onFailed(job: Job, error: Error) {
    console.error(`Job ${job.id} failed: ${error.message}`);
  }
}

// Sử dụng trong service
@Injectable()
export class UsersService {
  constructor(@InjectQueue('email') private readonly emailQueue: Queue) {}

  async register(dto: RegisterDto) {
    const user = await this.create(dto);

    await this.emailQueue.add('send-welcome', {
      email: user.email,
      name: user.name,
    }, {
      attempts: 3,
      backoff: 5000,
      removeOnComplete: true,
    });

    return user;
  }
}
```

---

## 4. Cách implement Caching

**Trả lời:**

```typescript
// Auto-caching với Interceptor
@Controller('products')
export class ProductsController {
  @Get()
  @UseInterceptors(CacheInterceptor)
  @CacheTTL(60)
  @CacheKey('all-products')
  findAll() {
    return this.productsService.findAll();
  }
}

// Manual caching
@Injectable()
export class ProductsService {
  constructor(@Inject(CACHE_MANAGER) private readonly cacheManager: Cache) {}

  async findById(id: number): Promise<Product> {
    const cacheKey = `product:${id}`;
    const cached = await this.cacheManager.get<Product>(cacheKey);
    if (cached) return cached;

    const product = await this.productRepo.findOneBy({ id });
    await this.cacheManager.set(cacheKey, product, 300);
    return product;
  }

  async update(id: number, dto: UpdateProductDto): Promise<Product> {
    const product = await this.productRepo.save({ id, ...dto });
    // Invalidate cache
    await this.cacheManager.del(`product:${id}`);
    await this.cacheManager.del('all-products');
    return product;
  }
}
```

---

## 5. Health Checks

```typescript
// npm install @nestjs/terminus

@Controller('health')
export class HealthController {
  constructor(
    private readonly health: HealthCheckService,
    private readonly db: TypeOrmHealthIndicator,
    private readonly memory: MemoryHealthIndicator,
  ) {}

  @Get()
  @HealthCheck()
  check() {
    return this.health.check([
      () => this.db.pingCheck('database'),
      () => this.memory.checkHeap('memory_heap', 300 * 1024 * 1024),
    ]);
  }
}
```

---

## 🎯 TIPS PHỎNG VẤN NESTJS

1. **Nắm vững Request Lifecycle**: M-G-I-P-H-I-F
2. **Hiểu DI sâu**: Custom providers, scopes, injection tokens
3. **Biết thiết kế module**: Khi nào dùng shared, global, dynamic module
4. **Security**: JWT, Guards, RBAC, rate limiting, helmet, CORS
5. **Performance**: Caching, queue, database indexing, pagination
6. **Testing**: Unit test (mock DI), E2E test (supertest)
7. **Thực hành**: Xây dựng một project CRUD hoàn chỉnh trước buổi phỏng vấn

---

## 📋 CÂU HỎI NHANH (QUICK FIRE)

| Câu hỏi | Trả lời ngắn |
|---|---|
| NestJS dùng gì bên dưới? | Express (default) hoặc Fastify |
| Cách tạo project mới? | `nest new project-name` |
| Cách tạo module? | `nest g module users` |
| Cách tạo controller? | `nest g controller users` |
| Cách tạo service? | `nest g service users` |
| Cách tạo CRUD resource? | `nest g resource users` |
| Pipe dùng để làm gì? | Validation & Transformation |
| Guard dùng để làm gì? | Authorization (cho phép hay không) |
| Interceptor dùng để làm gì? | Transform response, logging, caching |
| Filter dùng để làm gì? | Xử lý exceptions |
| Middleware vs Guard? | Middleware: general (logging), Guard: auth logic |
| @Injectable() là gì? | Đánh dấu class là provider, có thể inject |
| Scope DEFAULT là gì? | Singleton - 1 instance cho toàn app |

---

## 🔥 CÂU HỎI TÌNH HUỐNG LEVEL SENIOR

### 6. Thiết kế Multi-tenant Architecture

**Câu hỏi:** Cách thiết kế NestJS API hỗ trợ multi-tenancy?

**Trả lời:**

**3 Strategies:**

| Strategy | Isolation | Complexity | Cost |
|---|---|---|---|
| **Shared DB, shared schema** (tenant_id column) | Thấp | Thấp | Thấp |
| **Shared DB, separate schema** | Trung bình | Trung bình | Trung bình |
| **Separate DB per tenant** | Cao | Cao | Cao |

```typescript
// Strategy 1: tenant_id column (phổ biến nhất)

// Middleware extract tenant từ header/subdomain
@Injectable()
export class TenantMiddleware implements NestMiddleware {
  use(req: Request, res: Response, next: NextFunction) {
    const tenantId = req.headers['x-tenant-id'] as string
      || req.hostname.split('.')[0]; // subdomain-based

    if (!tenantId) throw new BadRequestException('Tenant ID required');

    req['tenantId'] = tenantId;
    next();
  }
}

// Tenant-aware repository
@Injectable({ scope: Scope.REQUEST })
export class TenantAwareRepository<T> {
  constructor(
    @InjectRepository(Product) private readonly repo: Repository<Product>,
    @Inject(REQUEST) private readonly request: Request,
  ) {}

  private get tenantId(): string {
    return this.request['tenantId'];
  }

  async findAll(): Promise<Product[]> {
    return this.repo.find({ where: { tenantId: this.tenantId } });
  }

  async create(dto: Partial<Product>): Promise<Product> {
    return this.repo.save({ ...dto, tenantId: this.tenantId });
  }
}
```

---

### 7. API Versioning Strategy

**Câu hỏi:** Cách implement API versioning trong NestJS?

**Trả lời:**

```typescript
// main.ts — Enable versioning
app.enableVersioning({
  type: VersioningType.URI,        // /api/v1/users
  // type: VersioningType.HEADER,  // X-API-Version: 1
  // type: VersioningType.MEDIA_TYPE, // Accept: application/json;v=1
  defaultVersion: '1',
});

// Controller với multiple versions
@Controller('users')
export class UsersController {
  @Get()
  @Version('1')
  findAllV1() {
    // V1: trả về format cũ
    return this.usersService.findAll();
  }

  @Get()
  @Version('2')
  findAllV2() {
    // V2: trả về format mới với pagination metadata
    return this.usersService.findAllWithMeta();
  }
}

// Hoặc tách controller theo version
@Controller({ path: 'users', version: '1' })
export class UsersV1Controller { }

@Controller({ path: 'users', version: '2' })
export class UsersV2Controller { }
```

---

### 8. Distributed Tracing & Observability

**Câu hỏi:** Cách implement logging, tracing, metrics trong NestJS microservices?

**Trả lời:**

```typescript
// Correlation ID interceptor — tracing requests across services
@Injectable()
export class CorrelationIdInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    const request = context.switchToHttp().getRequest();

    // Lấy từ header hoặc tạo mới
    const correlationId = request.headers['x-correlation-id'] || randomUUID();
    request.correlationId = correlationId;

    // Thêm vào response header
    const response = context.switchToHttp().getResponse();
    response.setHeader('x-correlation-id', correlationId);

    return next.handle().pipe(
      tap({
        next: () => {
          this.logger.log({
            correlationId,
            method: request.method,
            url: request.url,
            statusCode: response.statusCode,
          });
        },
        error: (error) => {
          this.logger.error({
            correlationId,
            method: request.method,
            url: request.url,
            error: error.message,
            stack: error.stack,
          });
        },
      }),
    );
  }
}

// Structured logging với Pino
// npm install nestjs-pino pino-http
@Module({
  imports: [
    LoggerModule.forRoot({
      pinoHttp: {
        level: process.env.NODE_ENV === 'production' ? 'info' : 'debug',
        transport: process.env.NODE_ENV !== 'production'
          ? { target: 'pino-pretty' }
          : undefined,
        serializers: {
          req: (req) => ({ method: req.method, url: req.url }),
          res: (res) => ({ statusCode: res.statusCode }),
        },
      },
    }),
  ],
})
export class AppModule {}
```

---

### 9. Database Connection Pooling & Performance

**Câu hỏi:** Cách tối ưu database performance trong NestJS?

**Trả lời:**

```typescript
// 1. Connection pooling
TypeOrmModule.forRoot({
  type: 'postgres',
  host: 'localhost',
  extra: {
    max: 20,          // Max connections in pool
    min: 5,           // Min connections
    idleTimeoutMillis: 30000,
  },
  logging: ['error', 'warn'], // Chỉ log errors trong production
})

// 2. Query optimization — select chỉ cần thiết
const users = await this.userRepo.find({
  select: ['id', 'name', 'email'],  // Không select *
  relations: ['posts'],
  where: { isActive: true },
  take: 20,
  cache: 60000,  // Cache 60s
});

// 3. Pagination với cursor-based (tốt hơn offset cho dataset lớn)
async findWithCursor(cursor?: string, limit = 20) {
  const qb = this.userRepo.createQueryBuilder('user')
    .orderBy('user.createdAt', 'DESC')
    .take(limit + 1); // +1 để check hasMore

  if (cursor) {
    qb.where('user.createdAt < :cursor', { cursor: new Date(cursor) });
  }

  const items = await qb.getMany();
  const hasMore = items.length > limit;

  return {
    items: hasMore ? items.slice(0, -1) : items,
    hasMore,
    nextCursor: hasMore ? items[items.length - 2].createdAt.toISOString() : null,
  };
}

// 4. N+1 query problem — dùng relations hoặc QueryBuilder join
// ❌ N+1: 1 query users + N queries cho mỗi user's posts
const users = await this.userRepo.find();
for (const user of users) {
  user.posts = await this.postRepo.find({ where: { authorId: user.id } });
}

// ✅ Eager loading: 1-2 queries
const users = await this.userRepo.find({ relations: ['posts'] });
```
