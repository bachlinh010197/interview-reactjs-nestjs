# 📘 System Design cho Backend Developer

[⬅️ Tình huống thực tế](./10-tinh-huong-thuc-te.md) | [Phần tiếp: Database Expert ➡️](./12-database-expert.md)

---

## 1. Thiết kế hệ thống URL Shortener (như Bit.ly)

**Câu hỏi:** Thiết kế hệ thống rút gọn URL có thể xử lý hàng triệu request/ngày.

**Trả lời:**

**Requirements:**
- Tạo short URL từ long URL
- Redirect short URL → long URL (read-heavy: ~100:1 read/write ratio)
- URL hết hạn sau thời gian nhất định
- Analytics: đếm số click

**Ước tính:**
- 100M URLs/tháng (write) → ~40 URL/s
- 10B redirects/tháng (read) → ~4000 redirect/s
- Storage: 100M × 500 bytes = ~50GB/tháng

**Kiến trúc:**

```
Client → CDN → Load Balancer → API Servers (NestJS)
                                    │
                        ┌───────────┼───────────┐
                        ▼           ▼           ▼
                    Redis Cache   PostgreSQL   Analytics
                   (hot URLs)    (persistent)  (Kafka → ClickHouse)
```

**Short URL Generation (6-7 ký tự base62):**

```typescript
// Approach 1: Base62 encode auto-increment ID
@Injectable()
export class UrlService {
  private readonly BASE62 = '0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ';

  private encode(num: number): string {
    let result = '';
    while (num > 0) {
      result = this.BASE62[num % 62] + result;
      num = Math.floor(num / 62);
    }
    return result.padStart(7, '0');
  }

  // Approach 2: Counter-based (distributed)
  // Dùng Redis INCR để tạo unique counter, tránh collision
  async createShortUrl(longUrl: string, userId?: string): Promise<string> {
    // Check duplicate
    const existing = await this.cache.get(`long:${longUrl}`);
    if (existing) return existing;

    // Tạo unique ID
    const counter = await this.redis.incr('url:counter');
    const shortCode = this.encode(counter);

    // Lưu vào DB + Cache
    await this.urlRepo.save({
      shortCode,
      longUrl,
      userId,
      expiresAt: new Date(Date.now() + 365 * 24 * 60 * 60 * 1000),
    });

    await this.cache.set(`short:${shortCode}`, longUrl, 86400); // Cache 24h
    await this.cache.set(`long:${longUrl}`, shortCode, 86400);

    return `https://short.ly/${shortCode}`;
  }

  async redirect(shortCode: string): Promise<string> {
    // 1. Check cache first
    let longUrl = await this.cache.get(`short:${shortCode}`);

    if (!longUrl) {
      // 2. Fallback to DB
      const url = await this.urlRepo.findOne({ where: { shortCode } });
      if (!url || url.expiresAt < new Date()) {
        throw new NotFoundException('URL not found or expired');
      }
      longUrl = url.longUrl;
      await this.cache.set(`short:${shortCode}`, longUrl, 86400);
    }

    // 3. Async analytics (non-blocking)
    this.eventEmitter.emit('url.clicked', { shortCode, timestamp: Date.now() });

    return longUrl;
  }
}
```

**Key Decisions:**
- **301 vs 302 Redirect**: 302 (temporary) → để track analytics. 301 (permanent) → browser cache, không track được
- **Cache strategy**: Cache-aside với Redis (hot URLs chiếm 80% traffic)
- **Database**: PostgreSQL cho persistence, Redis cho read cache
- **Analytics**: Kafka queue → batch write vào ClickHouse (không block redirect)

---

## 2. Thiết kế hệ thống Chat Real-time (như Slack)

**Câu hỏi:** Thiết kế hệ thống chat hỗ trợ 1:1 và group chat, online status.

**Trả lời:**

**Kiến trúc:**

```
                    ┌─────────────────────┐
                    │    Load Balancer     │
                    │  (Sticky Sessions)   │
                    └──────────┬──────────┘
                               │
              ┌────────────────┼────────────────┐
              ▼                ▼                ▼
        ┌──────────┐    ┌──────────┐    ┌──────────┐
        │ WS Server│    │ WS Server│    │ WS Server│
        │  Node 1  │    │  Node 2  │    │  Node 3  │
        └────┬─────┘    └────┬─────┘    └────┬─────┘
             │               │               │
             └───────────────┼───────────────┘
                             ▼
                    ┌─────────────────┐
                    │   Redis Pub/Sub │ ← Đồng bộ giữa các nodes
                    └────────┬────────┘
                             │
              ┌──────────────┼──────────────┐
              ▼              ▼              ▼
        ┌──────────┐  ┌──────────┐  ┌───────────┐
        │PostgreSQL│  │ MongoDB  │  │    S3     │
        │ (Users,  │  │(Messages │  │  (Files,  │
        │  Rooms)  │  │  History)│  │  Images)  │
        └──────────┘  └──────────┘  └───────────┘
```

```typescript
// NestJS WebSocket Gateway với Redis adapter
@WebSocketGateway({ cors: true })
export class ChatGateway implements OnGatewayConnection, OnGatewayDisconnect {
  @WebSocketServer() server: Server;

  constructor(
    private readonly chatService: ChatService,
    private readonly presenceService: PresenceService,
  ) {}

  async handleConnection(client: Socket) {
    const userId = this.extractUserId(client);
    await this.presenceService.setOnline(userId, client.id);

    // Join tất cả rooms user thuộc về
    const rooms = await this.chatService.getUserRooms(userId);
    rooms.forEach(room => client.join(`room:${room.id}`));

    // Broadcast online status
    this.server.emit('user:online', { userId });
  }

  async handleDisconnect(client: Socket) {
    const userId = this.extractUserId(client);
    await this.presenceService.setOffline(userId);
    this.server.emit('user:offline', { userId });
  }

  @SubscribeMessage('message:send')
  async handleMessage(
    @ConnectedSocket() client: Socket,
    @MessageBody() data: { roomId: string; content: string; type: 'text' | 'image' },
  ) {
    const userId = this.extractUserId(client);

    // Lưu message
    const message = await this.chatService.saveMessage({
      roomId: data.roomId,
      senderId: userId,
      content: data.content,
      type: data.type,
    });

    // Broadcast to room
    this.server.to(`room:${data.roomId}`).emit('message:new', message);

    // Push notification cho offline users
    const offlineMembers = await this.chatService.getOfflineMembers(data.roomId);
    await this.pushService.sendPush(offlineMembers, {
      title: 'New message',
      body: data.content,
    });
  }

  @SubscribeMessage('typing:start')
  handleTyping(
    @ConnectedSocket() client: Socket,
    @MessageBody() data: { roomId: string },
  ) {
    const userId = this.extractUserId(client);
    client.to(`room:${data.roomId}`).emit('typing:update', { userId, isTyping: true });
  }
}

// Presence Service — tracking online/offline
@Injectable()
export class PresenceService {
  constructor(private readonly redis: Redis) {}

  async setOnline(userId: string, socketId: string) {
    await this.redis.hset('presence', userId, JSON.stringify({
      status: 'online',
      socketId,
      lastSeen: Date.now(),
    }));
  }

  async setOffline(userId: string) {
    await this.redis.hset('presence', userId, JSON.stringify({
      status: 'offline',
      lastSeen: Date.now(),
    }));
  }

  async getOnlineUsers(userIds: string[]): Promise<string[]> {
    const pipeline = this.redis.pipeline();
    userIds.forEach(id => pipeline.hget('presence', id));
    const results = await pipeline.exec();
    return userIds.filter((_, i) => {
      const data = results[i][1] ? JSON.parse(results[i][1]) : null;
      return data?.status === 'online';
    });
  }
}
```

**Scaling WebSocket:**
- **Redis Pub/Sub adapter**: Đồng bộ messages giữa nhiều WS server nodes
- **Sticky sessions**: Load balancer dùng IP hash hoặc cookie để giữ connection trên cùng node
- **Horizontal scaling**: Mỗi node giữ subset connections, Redis làm message bus

---

## 3. Thiết kế hệ thống Rate Limiting phân tán

**Câu hỏi:** Thiết kế rate limiter cho API có multiple server instances.

**Trả lời:**

**Algorithms:**

| Algorithm | Mô tả | Ưu điểm | Nhược điểm |
|---|---|---|---|
| **Fixed Window** | Đếm trong cửa sổ thời gian cố định | Đơn giản | Burst ở biên window |
| **Sliding Window Log** | Log mỗi request, đếm trong window | Chính xác | Tốn memory |
| **Sliding Window Counter** | Kết hợp fixed + sliding | Cân bằng | Xấp xỉ |
| **Token Bucket** | Thêm token đều đặn, mỗi request tiêu 1 token | Cho phép burst | Phức tạp hơn |
| **Leaky Bucket** | Queue xử lý request ở tốc độ cố định | Smooth output | Có thể delay |

```typescript
// Sliding Window Counter với Redis (phân tán)
@Injectable()
export class RateLimiterService {
  constructor(private readonly redis: Redis) {}

  async isAllowed(
    key: string,           // e.g., "ratelimit:user:123" hoặc "ratelimit:ip:1.2.3.4"
    limit: number,         // max requests
    windowMs: number,      // window size in ms
  ): Promise<{ allowed: boolean; remaining: number; retryAfter?: number }> {
    const now = Date.now();
    const windowStart = now - windowMs;

    // Lua script — atomic operation (không race condition)
    const script = `
      -- Xóa entries cũ ngoài window
      redis.call('ZREMRANGEBYSCORE', KEYS[1], 0, ARGV[1])
      -- Đếm entries trong window
      local count = redis.call('ZCARD', KEYS[1])
      if count < tonumber(ARGV[3]) then
        -- Thêm request mới
        redis.call('ZADD', KEYS[1], ARGV[2], ARGV[2] .. ':' .. math.random())
        redis.call('PEXPIRE', KEYS[1], ARGV[4])
        return {1, tonumber(ARGV[3]) - count - 1}
      else
        -- Lấy thời gian request cũ nhất để tính retry-after
        local oldest = redis.call('ZRANGE', KEYS[1], 0, 0, 'WITHSCORES')
        return {0, 0, oldest[2]}
      end
    `;

    const [allowed, remaining, oldest] = await this.redis.eval(
      script, 1, key,
      windowStart.toString(),
      now.toString(),
      limit.toString(),
      windowMs.toString(),
    ) as [number, number, string?];

    return {
      allowed: allowed === 1,
      remaining,
      retryAfter: allowed === 0
        ? Math.ceil((parseFloat(oldest) + windowMs - now) / 1000)
        : undefined,
    };
  }
}

// NestJS Guard tích hợp
@Injectable()
export class DistributedRateLimitGuard implements CanActivate {
  constructor(
    private readonly rateLimiter: RateLimiterService,
    private readonly reflector: Reflector,
  ) {}

  async canActivate(context: ExecutionContext): Promise<boolean> {
    const request = context.switchToHttp().getRequest();
    const response = context.switchToHttp().getResponse();

    // Lấy config từ decorator
    const config = this.reflector.get<{ limit: number; window: number }>(
      'rateLimit', context.getHandler(),
    ) || { limit: 100, window: 60000 }; // Default: 100 req/min

    const key = `ratelimit:${request.user?.id || request.ip}`;
    const result = await this.rateLimiter.isAllowed(key, config.limit, config.window);

    // Set response headers
    response.setHeader('X-RateLimit-Limit', config.limit);
    response.setHeader('X-RateLimit-Remaining', result.remaining);

    if (!result.allowed) {
      response.setHeader('Retry-After', result.retryAfter);
      throw new HttpException('Too Many Requests', HttpStatus.TOO_MANY_REQUESTS);
    }

    return true;
  }
}

// Custom decorator
export const RateLimit = (limit: number, windowMs: number) =>
  SetMetadata('rateLimit', { limit, window: windowMs });

// Sử dụng
@Controller('auth')
export class AuthController {
  @Post('login')
  @RateLimit(5, 60000) // 5 requests per minute
  login() {}

  @Post('forgot-password')
  @RateLimit(3, 300000) // 3 requests per 5 minutes
  forgotPassword() {}
}
```

---

## 4. Thiết kế hệ thống Notification Service

**Câu hỏi:** Thiết kế notification service hỗ trợ email, SMS, push notification, in-app.

**Trả lời:**

**Kiến trúc:**

```
                    ┌──────────────────┐
                    │  API Gateway     │
                    └────────┬─────────┘
                             │
                    ┌────────▼─────────┐
                    │ Notification API │
                    │    (NestJS)      │
                    └────────┬─────────┘
                             │
                    ┌────────▼─────────┐
                    │   Message Queue  │ (RabbitMQ / Kafka)
                    │                  │
                    └───┬────┬────┬────┘
                        │    │    │
              ┌─────────┤    │    ├─────────┐
              ▼         ▼    ▼    ▼         ▼
         ┌────────┐ ┌──────┐ ┌──────┐ ┌────────┐
         │ Email  │ │ SMS  │ │ Push │ │ In-App │
         │Worker  │ │Worker│ │Worker│ │ Worker │
         └────────┘ └──────┘ └──────┘ └────────┘
              │         │       │          │
              ▼         ▼       ▼          ▼
          SendGrid   Twilio   Firebase  WebSocket
```

```typescript
// notification.module.ts — Strategy Pattern
interface NotificationChannel {
  send(notification: NotificationPayload): Promise<SendResult>;
  supports(type: ChannelType): boolean;
}

@Injectable()
export class EmailChannel implements NotificationChannel {
  supports(type: ChannelType) { return type === 'email'; }

  async send(payload: NotificationPayload): Promise<SendResult> {
    // Render template
    const html = await this.templateService.render(payload.template, payload.data);
    // Send via SendGrid/SES
    return this.mailer.send({
      to: payload.recipient.email,
      subject: payload.subject,
      html,
    });
  }
}

@Injectable()
export class PushChannel implements NotificationChannel {
  supports(type: ChannelType) { return type === 'push'; }

  async send(payload: NotificationPayload): Promise<SendResult> {
    const tokens = await this.deviceService.getTokens(payload.recipient.id);
    return this.firebase.messaging().sendEachForMulticast({
      tokens,
      notification: { title: payload.title, body: payload.body },
      data: payload.metadata,
    });
  }
}

// Notification Service — orchestrator
@Injectable()
export class NotificationService {
  constructor(
    @Inject('NOTIFICATION_CHANNELS')
    private readonly channels: NotificationChannel[],
    @InjectQueue('notifications')
    private readonly queue: Queue,
  ) {}

  // API endpoint → queue (non-blocking)
  async send(dto: SendNotificationDto) {
    // Check user preferences (opt-out, quiet hours)
    const prefs = await this.prefService.getUserPrefs(dto.userId);
    const activeChannels = dto.channels.filter(ch => !prefs.muted.includes(ch));

    // Add to queue per channel
    for (const channel of activeChannels) {
      await this.queue.add(channel, {
        ...dto,
        channel,
        scheduledAt: this.applyQuietHours(prefs, dto.scheduledAt),
      }, {
        attempts: 3,
        backoff: { type: 'exponential', delay: 5000 },
        priority: dto.priority, // 1 = high, 10 = low
      });
    }
  }

  // Batch notification (e.g., digest email)
  async sendDigest(userId: string) {
    const unread = await this.notificationRepo.findUnread(userId);
    if (unread.length === 0) return;

    await this.send({
      userId,
      channels: ['email'],
      template: 'daily-digest',
      data: { notifications: unread, count: unread.length },
    });
  }
}

// Queue processor
@Processor('notifications')
export class NotificationProcessor {
  @Process('email')
  async handleEmail(job: Job) {
    const channel = this.channels.find(ch => ch.supports('email'));
    const result = await channel.send(job.data);

    // Lưu delivery status
    await this.deliveryRepo.save({
      notificationId: job.data.id,
      channel: 'email',
      status: result.success ? 'delivered' : 'failed',
      error: result.error,
      deliveredAt: new Date(),
    });
  }
}
```

**Key Features:**
- **User Preferences**: Opt-in/opt-out per channel, quiet hours
- **Template Engine**: Handlebars/Mjml cho email templates
- **Delivery Tracking**: Lưu status mỗi notification (sent, delivered, read, failed)
- **Rate Limiting**: Giới hạn số notification/user/day
- **Deduplication**: Prevent gửi trùng notification

---

## 5. Thiết kế hệ thống File Upload Service

**Câu hỏi:** Thiết kế service upload file lớn (>1GB) với resume support.

**Trả lời:**

```typescript
// Chunked upload — chia file thành nhiều chunk nhỏ
@Controller('uploads')
export class UploadController {
  // 1. Initiate upload → trả về uploadId
  @Post('initiate')
  @UseGuards(JwtAuthGuard)
  async initiateUpload(@Body() dto: InitiateUploadDto, @CurrentUser() user: User) {
    const uploadId = randomUUID();
    const totalChunks = Math.ceil(dto.fileSize / CHUNK_SIZE); // CHUNK_SIZE = 5MB

    await this.redis.hset(`upload:${uploadId}`, {
      userId: user.id,
      fileName: dto.fileName,
      fileSize: dto.fileSize,
      mimeType: dto.mimeType,
      totalChunks,
      uploadedChunks: '[]',
      status: 'in_progress',
      createdAt: Date.now(),
    });

    // Generate pre-signed URLs cho mỗi chunk (direct upload to S3)
    const presignedUrls = await Promise.all(
      Array.from({ length: totalChunks }, (_, i) =>
        this.s3.getSignedUrl('putObject', {
          Bucket: 'uploads',
          Key: `temp/${uploadId}/chunk_${i}`,
          Expires: 3600,
        })
      ),
    );

    return { uploadId, totalChunks, chunkSize: CHUNK_SIZE, presignedUrls };
  }

  // 2. Notify chunk uploaded
  @Post(':uploadId/chunks/:chunkIndex')
  async chunkUploaded(
    @Param('uploadId') uploadId: string,
    @Param('chunkIndex', ParseIntPipe) chunkIndex: number,
  ) {
    const upload = await this.redis.hgetall(`upload:${uploadId}`);
    const uploaded = JSON.parse(upload.uploadedChunks);
    uploaded.push(chunkIndex);

    await this.redis.hset(`upload:${uploadId}`, 'uploadedChunks', JSON.stringify(uploaded));

    // Check if all chunks uploaded
    if (uploaded.length === parseInt(upload.totalChunks)) {
      // Trigger merge job
      await this.queue.add('merge-chunks', { uploadId });
    }

    return { uploaded: uploaded.length, total: parseInt(upload.totalChunks) };
  }

  // 3. Get upload status (for resume)
  @Get(':uploadId/status')
  async getStatus(@Param('uploadId') uploadId: string) {
    const upload = await this.redis.hgetall(`upload:${uploadId}`);
    return {
      status: upload.status,
      uploadedChunks: JSON.parse(upload.uploadedChunks),
      totalChunks: parseInt(upload.totalChunks),
    };
    // Client biết chunk nào đã upload → chỉ upload chunk còn thiếu (resume)
  }
}

// Merge processor
@Processor('uploads')
export class UploadProcessor {
  @Process('merge-chunks')
  async mergeChunks(job: Job<{ uploadId: string }>) {
    const { uploadId } = job.data;

    // S3 Multipart Upload Complete
    const parts = await this.s3.listParts({
      Bucket: 'uploads',
      Key: `temp/${uploadId}`,
    }).promise();

    await this.s3.completeMultipartUpload({
      Bucket: 'uploads',
      Key: `files/${uploadId}/${fileName}`,
      MultipartUpload: { Parts: parts },
    }).promise();

    // Cleanup temp chunks
    // Update status
    // Generate thumbnails (if image/video)
  }
}
```

---

## 6. CAP Theorem & Distributed Systems

**Câu hỏi:** Giải thích CAP theorem. Cách áp dụng khi thiết kế hệ thống?

**Trả lời:**

**CAP Theorem:** Trong distributed system, chỉ có thể đảm bảo **2 trong 3**:

| Property | Ý nghĩa |
|---|---|
| **C**onsistency | Mọi read đều nhận được data mới nhất |
| **A**vailability | Mọi request đều nhận được response (không error) |
| **P**artition Tolerance | Hệ thống vẫn hoạt động khi network bị chia cắt |

**Partition Tolerance là bắt buộc** trong distributed system → chọn giữa **CP** hoặc **AP**:

| Chọn | Hệ thống | Ví dụ |
|---|---|---|
| **CP** (Consistency + Partition) | Chấp nhận unavailable khi partition | **Banking**, inventory, booking |
| **AP** (Availability + Partition) | Chấp nhận stale data khi partition | **Social media**, DNS, caching |

```
Ví dụ thực tế:

Banking (CP):
- Chuyển tiền: PHẢI consistent → dùng distributed transaction
- Nếu network partition → trả về error thay vì sai số dư

Social Feed (AP):
- Feed hơi cũ vài giây → chấp nhận được
- Luôn available → user không bao giờ thấy error page
- Eventually consistent: data đồng bộ sau vài giây
```

**Các patterns hỗ trợ:**
- **Saga Pattern**: Distributed transaction qua event chain (compensating transactions)
- **CQRS + Event Sourcing**: Tách read/write, eventually consistent
- **Two-Phase Commit (2PC)**: Strong consistency nhưng slow, blocking

---

## 7. Horizontal Scaling Strategy

**Câu hỏi:** Giải thích cách scale hệ thống NestJS từ 1 server → millions users.

**Trả lời:**

```
Stage 1: Single Server (0-10K users)
┌──────────────────────┐
│  NestJS + PostgreSQL  │
│  + Redis (same box)   │
└──────────────────────┘

Stage 2: Separate DB (10K-100K users)
┌──────────┐     ┌──────────┐
│  NestJS  │────▶│PostgreSQL│
└──────────┘     └──────────┘
      │          ┌──────────┐
      └─────────▶│  Redis   │
                 └──────────┘

Stage 3: Load Balancer + Multiple App Servers (100K-1M)
                 ┌──────────┐
             ┌──▶│ NestJS 1 │──┐
┌─────────┐  │   └──────────┘  │  ┌──────────┐
│   LB    │──┤   ┌──────────┐  ├─▶│PostgreSQL│
│ (Nginx) │  ├──▶│ NestJS 2 │──┤  │ Primary  │
└─────────┘  │   └──────────┘  │  └────┬─────┘
             │   ┌──────────┐  │       │
             └──▶│ NestJS 3 │──┘  ┌────▼─────┐
                 └──────────┘     │ Replica(s)│ ← Read replicas
                                  └──────────┘

Stage 4: Microservices + Message Queue (1M+)
┌─────┐   ┌──────────┐   ┌──────────┐   ┌─────────┐
│ LB  │──▶│API Gateway│──▶│Auth Svc  │──▶│User DB  │
└─────┘   └────┬─────┘   └──────────┘   └─────────┘
               │          ┌──────────┐   ┌─────────┐
               ├─────────▶│Order Svc │──▶│Order DB │
               │          └────┬─────┘   └─────────┘
               │               │
               │          ┌────▼─────┐
               │          │  Kafka   │
               │          └────┬─────┘
               │          ┌────▼─────┐   ┌─────────┐
               └─────────▶│Notify Svc│──▶│Redis    │
                          └──────────┘   └─────────┘
```

**Checklist scaling:**

| Layer | Strategy |
|---|---|
| **App Server** | Stateless, horizontal scale, Docker + K8s |
| **Database** | Read replicas, connection pooling, sharding |
| **Cache** | Redis Cluster, cache-aside pattern |
| **Session** | Lưu trong Redis (không lưu in-memory) |
| **File Storage** | S3 / Object Storage (không lưu local disk) |
| **Search** | Elasticsearch (full-text search) |
| **Queue** | Kafka / RabbitMQ (async processing) |
| **CDN** | CloudFront / Cloudflare (static assets) |

---

## 8. API Gateway Pattern

**Câu hỏi:** API Gateway là gì? Cách implement trong NestJS microservices?

**Trả lời:**

API Gateway là single entry point cho tất cả clients, handle cross-cutting concerns.

**Responsibilities:**
- **Routing**: Forward request đến đúng service
- **Authentication**: Verify JWT token 1 lần
- **Rate Limiting**: Throttle requests
- **Request Aggregation**: Gộp nhiều service calls thành 1 response
- **Protocol Translation**: REST ↔ gRPC ↔ GraphQL
- **Logging/Monitoring**: Centralized logging

```typescript
// NestJS API Gateway
@Controller()
export class GatewayController {
  constructor(
    @Inject('USER_SERVICE') private readonly userClient: ClientProxy,
    @Inject('ORDER_SERVICE') private readonly orderClient: ClientProxy,
    @Inject('PRODUCT_SERVICE') private readonly productClient: ClientProxy,
  ) {}

  // Request Aggregation — 1 API call trả về data từ nhiều services
  @Get('dashboard')
  @UseGuards(JwtAuthGuard)
  async getDashboard(@CurrentUser() user: User) {
    const [profile, orders, recommendations] = await Promise.all([
      firstValueFrom(this.userClient.send({ cmd: 'get_profile' }, user.id)),
      firstValueFrom(this.orderClient.send({ cmd: 'get_recent_orders' }, user.id)),
      firstValueFrom(this.productClient.send({ cmd: 'get_recommendations' }, user.id)),
    ]);

    return { profile, orders, recommendations };
  }

  // Circuit Breaker — tránh cascade failure
  @Get('products/:id')
  async getProduct(@Param('id') id: string) {
    try {
      return await firstValueFrom(
        this.productClient.send({ cmd: 'get_product' }, id).pipe(
          timeout(3000), // Timeout 3s
          retry(2),       // Retry 2 lần
          catchError(() => of({ id, name: 'Unavailable', fromCache: true })), // Fallback
        ),
      );
    } catch {
      // Circuit breaker open → return cached data
      return this.cacheService.get(`product:${id}`);
    }
  }
}
```

---

## 9. Database Sharding

**Câu hỏi:** Khi nào cần sharding? Các chiến lược sharding?

**Trả lời:**

**Khi nào cần:** Database quá lớn cho 1 server (>1TB), write throughput cao, latency tăng.

| Strategy | Mô tả | Ưu điểm | Nhược điểm |
|---|---|---|---|
| **Range-based** | Shard theo range (user_id 1-1M → shard1) | Đơn giản, range query dễ | Hotspot nếu phân bố không đều |
| **Hash-based** | hash(user_id) % N → shard number | Phân bố đều | Khó range query, resharding phức tạp |
| **Directory-based** | Lookup table mapping key → shard | Linh hoạt | Single point of failure (lookup service) |
| **Geographic** | Shard theo region/country | Low latency | Cross-region queries phức tạp |

```typescript
// Consistent Hashing — tránh resharding khi thêm/bớt shard
@Injectable()
export class ShardRouter {
  private readonly ring: ConsistentHash;

  constructor() {
    this.ring = new ConsistentHash([
      'shard-1.db.internal',
      'shard-2.db.internal',
      'shard-3.db.internal',
    ], 150); // 150 virtual nodes per shard
  }

  getShardForUser(userId: string): string {
    return this.ring.get(userId);
  }

  getConnection(userId: string): DataSource {
    const shard = this.getShardForUser(userId);
    return this.connectionPool.get(shard);
  }
}
```

---

## 10. Caching Strategies

**Câu hỏi:** Giải thích các caching strategies. Khi nào dùng strategy nào?

**Trả lời:**

| Strategy | Mô tả | Use case |
|---|---|---|
| **Cache-Aside** | App đọc cache trước, miss → đọc DB → update cache | General purpose, đọc nhiều |
| **Write-Through** | Ghi vào cache + DB đồng thời | Cần consistency cao |
| **Write-Behind** | Ghi vào cache → async ghi DB | Write-heavy, chấp nhận eventual consistency |
| **Read-Through** | Cache tự fetch từ DB khi miss | Đơn giản hóa app logic |
| **Refresh-Ahead** | Tự refresh cache trước khi expire | Predictable access patterns |

```typescript
// Cache-Aside Pattern (phổ biến nhất)
@Injectable()
export class ProductService {
  async findById(id: number): Promise<Product> {
    const cacheKey = `product:${id}`;

    // 1. Đọc cache
    const cached = await this.redis.get(cacheKey);
    if (cached) return JSON.parse(cached);

    // 2. Cache miss → đọc DB
    const product = await this.productRepo.findOneBy({ id });
    if (!product) throw new NotFoundException();

    // 3. Update cache
    await this.redis.setex(cacheKey, 3600, JSON.stringify(product));

    return product;
  }

  async update(id: number, dto: UpdateProductDto): Promise<Product> {
    const product = await this.productRepo.save({ id, ...dto });

    // Invalidate cache (không update — tránh race condition)
    await this.redis.del(`product:${id}`);
    await this.redis.del('products:list:*'); // Invalidate list caches

    return product;
  }
}

// Cache Stampede Prevention (Thundering Herd)
@Injectable()
export class CacheService {
  async getOrSet<T>(key: string, ttl: number, factory: () => Promise<T>): Promise<T> {
    const cached = await this.redis.get(key);
    if (cached) return JSON.parse(cached);

    // Dùng distributed lock để chỉ 1 process fetch DB
    const lockKey = `lock:${key}`;
    const locked = await this.redis.set(lockKey, '1', 'EX', 10, 'NX');

    if (locked) {
      try {
        const data = await factory();
        await this.redis.setex(key, ttl, JSON.stringify(data));
        return data;
      } finally {
        await this.redis.del(lockKey);
      }
    } else {
      // Đợi process khác fetch xong
      await new Promise(resolve => setTimeout(resolve, 100));
      return this.getOrSet(key, ttl, factory); // Retry
    }
  }
}
```
