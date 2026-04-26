# 📘 NestJS - Phần 4: Providers & Services

[⬅️ Controllers](./03-controllers-routing.md) | [Phần tiếp: Middleware/Guards/Pipes ➡️](./05-middleware-guards-pipes.md)

---

## 1. Custom Providers

**Câu hỏi: Giải thích các loại custom provider trong NestJS.**

**Trả lời:**

```typescript
@Module({
  providers: [
    // 1. useClass: Thay thế implementation
    {
      provide: LoggerService,
      useClass: process.env.NODE_ENV === 'production'
        ? ProductionLoggerService
        : DevelopmentLoggerService,
    },

    // 2. useValue: Cung cấp giá trị cố định
    {
      provide: 'API_KEY',
      useValue: process.env.API_KEY,
    },
    {
      provide: 'CONFIG',
      useValue: { retryAttempts: 3, timeout: 5000 },
    },

    // 3. useFactory: Tạo provider động, có thể inject dependencies
    {
      provide: 'DATABASE_CONNECTION',
      useFactory: async (configService: ConfigService) => {
        const options = configService.get('database');
        return createConnection(options);
      },
      inject: [ConfigService],
    },

    // 4. useExisting: Alias cho provider đã tồn tại
    {
      provide: 'AliasedLogger',
      useExisting: LoggerService,
    },
  ],
})
export class AppModule {}

// Inject custom provider
@Injectable()
export class ApiService {
  constructor(
    @Inject('API_KEY') private readonly apiKey: string,
    @Inject('DATABASE_CONNECTION') private readonly db: Connection,
  ) {}
}
```

---

## 2. Provider Scopes

**Câu hỏi: Giải thích các scope của provider.**

**Trả lời:**

```typescript
// DEFAULT (Singleton): Một instance duy nhất cho toàn app
@Injectable() // hoặc @Injectable({ scope: Scope.DEFAULT })
export class SingletonService {}

// REQUEST: Tạo instance mới cho mỗi request
@Injectable({ scope: Scope.REQUEST })
export class RequestScopedService {
  constructor(@Inject(REQUEST) private readonly request: Request) {
    // Truy cập request object
  }
}

// TRANSIENT: Tạo instance mới mỗi khi inject
@Injectable({ scope: Scope.TRANSIENT })
export class TransientService {
  private readonly id = Math.random();
}
```

**Lưu ý:** Scope REQUEST và TRANSIENT ảnh hưởng performance vì tạo instance mới. Bubble up: nếu Service A (DEFAULT) inject Service B (REQUEST) → Service A cũng trở thành REQUEST scope.

---

## 🔥 CÂU HỎI LEVEL MIDDLE → SENIOR

### 3. Injection Token & Symbol-based Injection

**Câu hỏi: Khi nào cần dùng Injection Token? Khác gì string token?**

**Trả lời:**

```typescript
// ❌ String token: dễ conflict, không type-safe
{ provide: 'CONFIG', useValue: { ... } }

// ✅ InjectionToken: unique, type-safe
// tokens.ts
export const DATABASE_CONFIG = Symbol('DATABASE_CONFIG');
export const CACHE_MANAGER = Symbol('CACHE_MANAGER');

// Hoặc dùng class-based token (recommended)
export abstract class PaymentGateway {
  abstract charge(amount: number): Promise<void>;
}

@Injectable()
export class StripeGateway implements PaymentGateway {
  async charge(amount: number) { /* Stripe logic */ }
}

@Module({
  providers: [
    {
      provide: PaymentGateway,      // Abstract class as token
      useClass: StripeGateway,       // Concrete implementation
    },
  ],
})
export class PaymentModule {}

// Inject bằng abstract class → swap implementation dễ dàng
@Injectable()
export class OrderService {
  constructor(private readonly payment: PaymentGateway) {}

  async checkout(orderId: string) {
    await this.payment.charge(100); // Không biết/quan tâm đó là Stripe hay PayPal
  }
}
```

---

### 4. Multi-provider Pattern

**Câu hỏi: Cách inject nhiều implementations cho cùng 1 token?**

**Trả lời:**

```typescript
// Notification strategy pattern — nhiều channels
export const NOTIFICATION_CHANNEL = Symbol('NOTIFICATION_CHANNEL');

export interface NotificationChannel {
  send(userId: string, message: string): Promise<void>;
}

@Injectable()
export class EmailChannel implements NotificationChannel {
  async send(userId: string, message: string) { /* send email */ }
}

@Injectable()
export class SmsChannel implements NotificationChannel {
  async send(userId: string, message: string) { /* send SMS */ }
}

@Injectable()
export class PushChannel implements NotificationChannel {
  async send(userId: string, message: string) { /* push notification */ }
}

@Module({
  providers: [
    EmailChannel,
    SmsChannel,
    PushChannel,
    {
      provide: NOTIFICATION_CHANNEL,
      useFactory: (email: EmailChannel, sms: SmsChannel, push: PushChannel) => {
        return [email, sms, push]; // Array of channels
      },
      inject: [EmailChannel, SmsChannel, PushChannel],
    },
  ],
  exports: [NOTIFICATION_CHANNEL],
})
export class NotificationModule {}

// Inject tất cả channels
@Injectable()
export class NotificationService {
  constructor(
    @Inject(NOTIFICATION_CHANNEL)
    private readonly channels: NotificationChannel[],
  ) {}

  async notifyAll(userId: string, message: string) {
    await Promise.allSettled(
      this.channels.map(channel => channel.send(userId, message)),
    );
  }
}
```

---

### 5. Lazy-loaded Modules & Dynamic Module Loading

**Câu hỏi: Cách lazy-load module trong NestJS?**

**Trả lời:**

```typescript
import { LazyModuleLoader } from '@nestjs/core';

@Injectable()
export class ReportService {
  constructor(private readonly lazyModuleLoader: LazyModuleLoader) {}

  async generateReport(type: string) {
    // Module chỉ được load khi cần — không tốn memory lúc startup
    const { ReportModule } = await import('./report.module');
    const moduleRef = await this.lazyModuleLoader.load(() => ReportModule);

    const reportService = moduleRef.get(ReportGeneratorService);
    return reportService.generate(type);
  }
}
```

**Use cases:** Report generation, admin tools, ít sử dụng nhưng tốn tài nguyên.
