# 📘 Hexagonal & Clean Architecture

[⬅️ Database Expert](./12-database-expert.md) | [🏠 Mục lục](../README.md)

---

## 1. Clean Architecture là gì?

**Câu hỏi:** Giải thích Clean Architecture của Uncle Bob. Tại sao cần?

**Trả lời:**

Clean Architecture tổ chức code theo **layers** với **dependency rule**: dependencies chỉ trỏ **vào trong** (outer layers phụ thuộc inner layers, không bao giờ ngược lại).

```
┌─────────────────────────────────────────────────┐
│              Frameworks & Drivers                │  ← NestJS, TypeORM, Express
│  ┌─────────────────────────────────────────┐    │
│  │          Interface Adapters              │    │  ← Controllers, Repositories, Presenters
│  │  ┌─────────────────────────────────┐    │    │
│  │  │       Application Layer         │    │    │  ← Use Cases / Application Services
│  │  │  ┌─────────────────────────┐    │    │    │
│  │  │  │     Domain Layer        │    │    │    │  ← Entities, Value Objects, Domain Services
│  │  │  │   (Business Rules)      │    │    │    │
│  │  │  └─────────────────────────┘    │    │    │
│  │  └─────────────────────────────────┘    │    │
│  └─────────────────────────────────────────┘    │
└─────────────────────────────────────────────────┘
         Dependencies point INWARD →
```

**Dependency Rule:**
- **Domain** không biết gì về Database, HTTP, Framework
- **Application** biết Domain, không biết Framework
- **Infrastructure** biết tất cả nhưng chỉ implement interfaces từ inner layers

**Lợi ích:**
- **Testable**: Business logic test được mà không cần DB, HTTP
- **Framework independent**: Đổi Express → Fastify, TypeORM → Prisma mà không ảnh hưởng business logic
- **Maintainable**: Mỗi layer có responsibility rõ ràng

---

## 2. Hexagonal Architecture (Ports & Adapters)

**Câu hỏi:** Hexagonal Architecture khác gì Clean Architecture?

**Trả lời:**

Hexagonal Architecture (Alistair Cockburn) cùng ý tưởng với Clean Architecture nhưng dùng khái niệm **Ports** và **Adapters**.

```
                    ┌─────────────────────┐
       HTTP ───────▶│ Port: REST API      │
                    │ (Driving/Primary)    │
    GraphQL ───────▶│                     │
                    │   ┌─────────────┐   │
                    │   │  Application│   │
       CLI ────────▶│   │   Core      │   │──────▶ Port: Repository
                    │   │             │   │        Adapter: PostgreSQL
    Message ───────▶│   │ Domain      │   │
    Queue           │   │ Logic       │   │──────▶ Port: EmailService
                    │   │             │   │        Adapter: SendGrid
                    │   └─────────────┘   │
                    │                     │──────▶ Port: PaymentGateway
                    │ Adapter: NestJS     │        Adapter: Stripe
                    │ Adapter: Express    │
                    └─────────────────────┘

    Driving (Primary)              Driven (Secondary)
    Adapters: Input                Adapters: Output
    (HTTP, CLI, Queue)             (DB, Email, Payment)
```

**Khác biệt:**

| | Clean Architecture | Hexagonal Architecture |
|---|---|---|
| Layers | 4 layers (entities, use cases, adapters, frameworks) | Core + Ports + Adapters |
| Focus | Dependency direction | Symmetry (input/output đều qua ports) |
| Terminology | Use Cases, Entities | Ports, Adapters, Driving/Driven |
| Origin | Uncle Bob (2012) | Alistair Cockburn (2005) |

**Bản chất giống nhau:** Isolate business logic khỏi infrastructure.

---

## 3. Implement Clean Architecture trong NestJS

**Câu hỏi:** Cách cấu trúc folder NestJS theo Clean Architecture?

**Trả lời:**

```
src/
├── domain/                          # Layer 1: Domain (innermost)
│   ├── entities/
│   │   ├── user.entity.ts           # Pure domain entity (no decorators!)
│   │   ├── order.entity.ts
│   │   └── product.entity.ts
│   ├── value-objects/
│   │   ├── email.vo.ts
│   │   ├── money.vo.ts
│   │   └── address.vo.ts
│   ├── repositories/
│   │   ├── user.repository.interface.ts    # Port (interface only)
│   │   └── order.repository.interface.ts
│   ├── services/
│   │   └── pricing.domain-service.ts       # Domain logic spanning entities
│   └── exceptions/
│       ├── insufficient-funds.exception.ts
│       └── user-not-found.exception.ts
│
├── application/                     # Layer 2: Application (Use Cases)
│   ├── use-cases/
│   │   ├── create-order.use-case.ts
│   │   ├── cancel-order.use-case.ts
│   │   └── get-user-profile.use-case.ts
│   ├── dto/
│   │   ├── create-order.input.ts
│   │   └── order.output.ts
│   └── ports/                       # Secondary ports (output)
│       ├── email.service.interface.ts
│       ├── payment.gateway.interface.ts
│       └── event-publisher.interface.ts
│
├── infrastructure/                  # Layer 3: Infrastructure (Adapters)
│   ├── persistence/
│   │   ├── typeorm/
│   │   │   ├── entities/            # TypeORM entities (with decorators)
│   │   │   │   ├── user.orm-entity.ts
│   │   │   │   └── order.orm-entity.ts
│   │   │   ├── repositories/        # Adapter implementations
│   │   │   │   ├── user.typeorm-repository.ts
│   │   │   │   └── order.typeorm-repository.ts
│   │   │   └── mappers/
│   │   │       ├── user.mapper.ts   # ORM Entity ↔ Domain Entity
│   │   │       └── order.mapper.ts
│   │   └── prisma/                  # Alternative adapter
│   │       └── ...
│   ├── external-services/
│   │   ├── sendgrid-email.service.ts
│   │   ├── stripe-payment.gateway.ts
│   │   └── kafka-event-publisher.ts
│   └── config/
│       └── database.config.ts
│
└── presentation/                    # Layer 3: Presentation (Primary Adapters)
    ├── http/
    │   ├── controllers/
    │   │   ├── user.controller.ts
    │   │   └── order.controller.ts
    │   ├── dto/                     # HTTP-specific DTOs (validation decorators)
    │   │   ├── create-order.request.ts
    │   │   └── order.response.ts
    │   └── mappers/
    │       └── order.http-mapper.ts
    ├── graphql/
    │   └── resolvers/
    └── websocket/
        └── gateways/
```

---

## 4. Domain Layer — Entities & Value Objects

**Câu hỏi:** Khác biệt giữa Entity và Value Object?

**Trả lời:**

| | Entity | Value Object |
|---|---|---|
| Identity | Có ID duy nhất | Không có ID, so sánh bằng giá trị |
| Mutability | Mutable (thay đổi state) | **Immutable** |
| Lifecycle | Có lifecycle (create, update, delete) | Tạo mới khi cần thay đổi |
| Ví dụ | User, Order, Product | Email, Money, Address, DateRange |

```typescript
// Domain Entity — KHÔNG có decorators của TypeORM/class-validator!
// Pure business logic
export class Order {
  private readonly _id: string;
  private _items: OrderItem[];
  private _status: OrderStatus;
  private _totalAmount: Money;
  private readonly _createdAt: Date;

  constructor(props: {
    id: string;
    items: OrderItem[];
    customerId: string;
  }) {
    if (props.items.length === 0) {
      throw new EmptyOrderException();
    }
    this._id = props.id;
    this._items = props.items;
    this._status = OrderStatus.PENDING;
    this._totalAmount = this.calculateTotal();
    this._createdAt = new Date();
  }

  // Business logic nằm TRONG entity
  addItem(item: OrderItem): void {
    if (this._status !== OrderStatus.PENDING) {
      throw new OrderNotModifiableException(this._id);
    }
    this._items.push(item);
    this._totalAmount = this.calculateTotal();
  }

  cancel(): void {
    if (this._status === OrderStatus.SHIPPED) {
      throw new CannotCancelShippedException(this._id);
    }
    this._status = OrderStatus.CANCELLED;
  }

  confirm(): void {
    if (this._status !== OrderStatus.PENDING) {
      throw new InvalidStatusTransitionException(this._status, OrderStatus.CONFIRMED);
    }
    this._status = OrderStatus.CONFIRMED;
  }

  private calculateTotal(): Money {
    return this._items.reduce(
      (sum, item) => sum.add(item.subtotal),
      Money.zero('USD'),
    );
  }

  // Getters (no setters — encapsulation!)
  get id() { return this._id; }
  get status() { return this._status; }
  get totalAmount() { return this._totalAmount; }
  get items() { return [...this._items]; } // Return copy
}

// Value Object — Immutable, no identity
export class Money {
  private constructor(
    private readonly _amount: number,
    private readonly _currency: string,
  ) {
    if (_amount < 0) throw new NegativeAmountException();
  }

  static create(amount: number, currency: string): Money {
    return new Money(amount, currency);
  }

  static zero(currency: string): Money {
    return new Money(0, currency);
  }

  add(other: Money): Money {
    if (this._currency !== other._currency) {
      throw new CurrencyMismatchException();
    }
    return new Money(this._amount + other._amount, this._currency);
  }

  multiply(factor: number): Money {
    return new Money(this._amount * factor, this._currency);
  }

  equals(other: Money): boolean {
    return this._amount === other._amount && this._currency === other._currency;
  }

  get amount() { return this._amount; }
  get currency() { return this._currency; }
}

// Value Object: Email
export class Email {
  private readonly _value: string;

  constructor(value: string) {
    if (!this.isValid(value)) {
      throw new InvalidEmailException(value);
    }
    this._value = value.toLowerCase().trim();
  }

  private isValid(email: string): boolean {
    return /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email);
  }

  equals(other: Email): boolean {
    return this._value === other._value;
  }

  get value() { return this._value; }
}
```

---

## 5. Application Layer — Use Cases

**Câu hỏi:** Use Case là gì? Cách viết use case đúng?

**Trả lời:**

Use Case = 1 hành động cụ thể của hệ thống. Mỗi use case là 1 class với 1 method `execute()`.

```typescript
// Port: Repository interface (defined in domain/application layer)
export interface IOrderRepository {
  findById(id: string): Promise<Order | null>;
  save(order: Order): Promise<void>;
  nextId(): string;
}

export interface IPaymentGateway {
  charge(amount: Money, paymentMethod: string): Promise<PaymentResult>;
}

export interface IEventPublisher {
  publish(event: DomainEvent): Promise<void>;
}

// Use Case: Create Order
export class CreateOrderUseCase {
  constructor(
    private readonly orderRepo: IOrderRepository,       // Injected via interface
    private readonly productRepo: IProductRepository,
    private readonly paymentGateway: IPaymentGateway,
    private readonly eventPublisher: IEventPublisher,
  ) {}

  async execute(input: CreateOrderInput): Promise<CreateOrderOutput> {
    // 1. Validate business rules
    const products = await Promise.all(
      input.items.map(item => this.productRepo.findById(item.productId)),
    );

    const unavailable = products.filter(p => !p || !p.isAvailable);
    if (unavailable.length > 0) {
      throw new ProductUnavailableException(unavailable.map(p => p?.id));
    }

    // 2. Create domain entity
    const orderId = this.orderRepo.nextId();
    const orderItems = input.items.map((item, i) =>
      new OrderItem(products[i], item.quantity),
    );
    const order = new Order({ id: orderId, items: orderItems, customerId: input.customerId });

    // 3. Process payment
    const paymentResult = await this.paymentGateway.charge(
      order.totalAmount,
      input.paymentMethodId,
    );

    if (!paymentResult.success) {
      throw new PaymentFailedException(paymentResult.error);
    }

    order.confirm();

    // 4. Persist
    await this.orderRepo.save(order);

    // 5. Publish domain event
    await this.eventPublisher.publish(
      new OrderCreatedEvent(order.id, order.totalAmount, input.customerId),
    );

    // 6. Return output DTO (not domain entity!)
    return {
      orderId: order.id,
      status: order.status,
      totalAmount: order.totalAmount.amount,
      currency: order.totalAmount.currency,
    };
  }
}

// Input/Output DTOs (application layer — no framework decorators)
export interface CreateOrderInput {
  customerId: string;
  items: Array<{ productId: string; quantity: number }>;
  paymentMethodId: string;
}

export interface CreateOrderOutput {
  orderId: string;
  status: string;
  totalAmount: number;
  currency: string;
}
```

---

## 6. Infrastructure Layer — Adapters

**Câu hỏi:** Cách implement adapter pattern cho database và external services?

**Trả lời:**

```typescript
// ======== Persistence Adapter ========

// ORM Entity (infrastructure concern — có TypeORM decorators)
@Entity('orders')
export class OrderOrmEntity {
  @PrimaryColumn()
  id: string;

  @Column()
  customerId: string;

  @Column({ type: 'enum', enum: ['pending', 'confirmed', 'shipped', 'cancelled'] })
  status: string;

  @Column({ type: 'decimal' })
  totalAmount: number;

  @Column()
  currency: string;

  @OneToMany(() => OrderItemOrmEntity, item => item.order, { cascade: true })
  items: OrderItemOrmEntity[];

  @CreateDateColumn()
  createdAt: Date;
}

// Mapper: Domain Entity ↔ ORM Entity
@Injectable()
export class OrderMapper {
  toDomain(orm: OrderOrmEntity): Order {
    return Order.reconstitute({
      id: orm.id,
      customerId: orm.customerId,
      status: orm.status as OrderStatus,
      items: orm.items.map(item => OrderItem.reconstitute({
        productId: item.productId,
        quantity: item.quantity,
        price: Money.create(item.price, orm.currency),
      })),
      totalAmount: Money.create(orm.totalAmount, orm.currency),
      createdAt: orm.createdAt,
    });
  }

  toOrm(domain: Order): OrderOrmEntity {
    const orm = new OrderOrmEntity();
    orm.id = domain.id;
    orm.customerId = domain.customerId;
    orm.status = domain.status;
    orm.totalAmount = domain.totalAmount.amount;
    orm.currency = domain.totalAmount.currency;
    orm.items = domain.items.map(item => {
      const itemOrm = new OrderItemOrmEntity();
      itemOrm.productId = item.productId;
      itemOrm.quantity = item.quantity;
      itemOrm.price = item.price.amount;
      return itemOrm;
    });
    return orm;
  }
}

// Repository Adapter: implement domain interface
@Injectable()
export class OrderTypeOrmRepository implements IOrderRepository {
  constructor(
    @InjectRepository(OrderOrmEntity)
    private readonly repo: Repository<OrderOrmEntity>,
    private readonly mapper: OrderMapper,
  ) {}

  async findById(id: string): Promise<Order | null> {
    const orm = await this.repo.findOne({
      where: { id },
      relations: ['items'],
    });
    return orm ? this.mapper.toDomain(orm) : null;
  }

  async save(order: Order): Promise<void> {
    const orm = this.mapper.toOrm(order);
    await this.repo.save(orm);
  }

  nextId(): string {
    return randomUUID();
  }
}

// ======== External Service Adapter ========

// Payment Gateway Adapter
@Injectable()
export class StripePaymentGateway implements IPaymentGateway {
  constructor(private readonly configService: ConfigService) {}

  async charge(amount: Money, paymentMethodId: string): Promise<PaymentResult> {
    try {
      const stripe = new Stripe(this.configService.get('STRIPE_SECRET_KEY'));
      const intent = await stripe.paymentIntents.create({
        amount: Math.round(amount.amount * 100), // Stripe uses cents
        currency: amount.currency,
        payment_method: paymentMethodId,
        confirm: true,
      });

      return { success: true, transactionId: intent.id };
    } catch (error) {
      return { success: false, error: error.message };
    }
  }
}
```

---

## 7. Presentation Layer — Controllers

**Câu hỏi:** Controller trong Clean Architecture khác gì controller thông thường?

**Trả lời:**

Controller là **thin adapter** — chỉ:
1. Parse HTTP request → Application DTO
2. Gọi Use Case
3. Map result → HTTP response

**KHÔNG chứa business logic.**

```typescript
// HTTP-specific DTO (có validation decorators)
export class CreateOrderRequest {
  @IsString()
  customerId: string;

  @IsArray()
  @ValidateNested({ each: true })
  @Type(() => OrderItemRequest)
  items: OrderItemRequest[];

  @IsString()
  paymentMethodId: string;
}

export class OrderItemRequest {
  @IsString()
  productId: string;

  @IsInt()
  @Min(1)
  quantity: number;
}

// Controller — thin adapter
@Controller('orders')
@UseGuards(JwtAuthGuard)
export class OrderController {
  constructor(
    private readonly createOrderUseCase: CreateOrderUseCase,
    private readonly getOrderUseCase: GetOrderUseCase,
    private readonly cancelOrderUseCase: CancelOrderUseCase,
  ) {}

  @Post()
  @HttpCode(HttpStatus.CREATED)
  async createOrder(
    @Body() request: CreateOrderRequest,
    @CurrentUser('id') userId: string,
  ) {
    // 1. Map HTTP DTO → Application DTO
    const input: CreateOrderInput = {
      customerId: userId,
      items: request.items,
      paymentMethodId: request.paymentMethodId,
    };

    // 2. Execute use case
    const result = await this.createOrderUseCase.execute(input);

    // 3. Return (NestJS auto-serializes)
    return result;
  }

  @Delete(':id')
  @HttpCode(HttpStatus.NO_CONTENT)
  async cancelOrder(
    @Param('id') orderId: string,
    @CurrentUser('id') userId: string,
  ) {
    await this.cancelOrderUseCase.execute({ orderId, userId });
  }
}
```

---

## 8. Dependency Injection Wiring

**Câu hỏi:** Cách wire tất cả layers lại với nhau trong NestJS Module?

**Trả lời:**

```typescript
// order.module.ts — Wiring point
@Module({
  imports: [TypeOrmModule.forFeature([OrderOrmEntity, OrderItemOrmEntity])],
  controllers: [OrderController],
  providers: [
    // Mappers
    OrderMapper,

    // Repository: bind interface → implementation
    {
      provide: 'IOrderRepository',     // Token = interface name
      useClass: OrderTypeOrmRepository, // Adapter implementation
    },

    // External services
    {
      provide: 'IPaymentGateway',
      useClass: StripePaymentGateway,
    },
    {
      provide: 'IEventPublisher',
      useClass: KafkaEventPublisher,
    },

    // Use Cases
    {
      provide: CreateOrderUseCase,
      useFactory: (
        orderRepo: IOrderRepository,
        productRepo: IProductRepository,
        paymentGateway: IPaymentGateway,
        eventPublisher: IEventPublisher,
      ) => new CreateOrderUseCase(orderRepo, productRepo, paymentGateway, eventPublisher),
      inject: ['IOrderRepository', 'IProductRepository', 'IPaymentGateway', 'IEventPublisher'],
    },

    // Hoặc đơn giản hơn — dùng @Inject() trong use case constructor
    CreateOrderUseCase,
    GetOrderUseCase,
    CancelOrderUseCase,
  ],
  exports: ['IOrderRepository'],
})
export class OrderModule {}

// Trong use case, inject bằng token:
export class CreateOrderUseCase {
  constructor(
    @Inject('IOrderRepository') private readonly orderRepo: IOrderRepository,
    @Inject('IProductRepository') private readonly productRepo: IProductRepository,
    @Inject('IPaymentGateway') private readonly paymentGateway: IPaymentGateway,
    @Inject('IEventPublisher') private readonly eventPublisher: IEventPublisher,
  ) {}
}
```

---

## 9. Testing trong Clean Architecture

**Câu hỏi:** Clean Architecture giúp testing dễ hơn như thế nào?

**Trả lời:**

```typescript
// ✅ Test Use Case — KHÔNG cần DB, HTTP, Framework!
describe('CreateOrderUseCase', () => {
  let useCase: CreateOrderUseCase;
  let orderRepo: jest.Mocked<IOrderRepository>;
  let productRepo: jest.Mocked<IProductRepository>;
  let paymentGateway: jest.Mocked<IPaymentGateway>;
  let eventPublisher: jest.Mocked<IEventPublisher>;

  beforeEach(() => {
    // Mock tất cả dependencies (interfaces → dễ mock)
    orderRepo = {
      findById: jest.fn(),
      save: jest.fn(),
      nextId: jest.fn().mockReturnValue('order-123'),
    };
    productRepo = {
      findById: jest.fn().mockResolvedValue(
        Product.create({ id: 'prod-1', name: 'Widget', price: Money.create(10, 'USD'), stock: 100 }),
      ),
    };
    paymentGateway = {
      charge: jest.fn().mockResolvedValue({ success: true, transactionId: 'tx-1' }),
    };
    eventPublisher = {
      publish: jest.fn().mockResolvedValue(undefined),
    };

    useCase = new CreateOrderUseCase(orderRepo, productRepo, paymentGateway, eventPublisher);
  });

  it('should create order successfully', async () => {
    const result = await useCase.execute({
      customerId: 'user-1',
      items: [{ productId: 'prod-1', quantity: 2 }],
      paymentMethodId: 'pm-1',
    });

    expect(result.orderId).toBe('order-123');
    expect(result.status).toBe('confirmed');
    expect(orderRepo.save).toHaveBeenCalled();
    expect(paymentGateway.charge).toHaveBeenCalledWith(
      expect.objectContaining({ amount: 20 }),
      'pm-1',
    );
    expect(eventPublisher.publish).toHaveBeenCalled();
  });

  it('should throw when product unavailable', async () => {
    productRepo.findById.mockResolvedValue(null);

    await expect(useCase.execute({
      customerId: 'user-1',
      items: [{ productId: 'invalid', quantity: 1 }],
      paymentMethodId: 'pm-1',
    })).rejects.toThrow(ProductUnavailableException);

    expect(orderRepo.save).not.toHaveBeenCalled();
    expect(paymentGateway.charge).not.toHaveBeenCalled();
  });

  it('should throw when payment fails', async () => {
    paymentGateway.charge.mockResolvedValue({ success: false, error: 'Card declined' });

    await expect(useCase.execute({
      customerId: 'user-1',
      items: [{ productId: 'prod-1', quantity: 1 }],
      paymentMethodId: 'pm-bad',
    })).rejects.toThrow(PaymentFailedException);

    expect(orderRepo.save).not.toHaveBeenCalled();
  });
});

// ✅ Test Domain Entity — pure logic, zero dependencies
describe('Order', () => {
  it('should not allow empty order', () => {
    expect(() => new Order({ id: '1', items: [], customerId: 'u1' }))
      .toThrow(EmptyOrderException);
  });

  it('should calculate total correctly', () => {
    const order = new Order({
      id: '1',
      customerId: 'u1',
      items: [
        new OrderItem(product10USD, 2),  // $20
        new OrderItem(product5USD, 3),   // $15
      ],
    });
    expect(order.totalAmount.equals(Money.create(35, 'USD'))).toBe(true);
  });

  it('should not cancel shipped order', () => {
    const order = createShippedOrder();
    expect(() => order.cancel()).toThrow(CannotCancelShippedException);
  });
});
```

---

## 10. Khi nào nên / KHÔNG nên dùng Clean Architecture?

**Câu hỏi:** Clean Architecture có phải lúc nào cũng tốt?

**Trả lời:**

### ✅ NÊN dùng khi:
- **Domain phức tạp**: E-commerce, banking, healthcare, booking systems
- **Long-lived project**: Dự án phát triển nhiều năm, team lớn
- **Business logic quan trọng**: Logic là competitive advantage
- **Cần swap infrastructure**: Có thể đổi DB, payment provider, cloud provider
- **Team có kinh nghiệm**: Hiểu DDD, SOLID, design patterns

### ❌ KHÔNG nên dùng khi:
- **CRUD đơn giản**: Blog, TODO app → over-engineering
- **Prototype / MVP**: Cần ship nhanh, validate idea
- **Small team**: 1-2 developers → overhead lớn
- **Short-lived project**: Script, data migration, one-time tool
- **Data-centric app**: Dashboard, reporting → business logic ít

### Pragmatic approach — bắt đầu đơn giản, refactor khi cần:

```
Phase 1 (MVP): Standard NestJS
├── Controller → Service → Repository
└── Nhanh, đơn giản, đủ dùng

Phase 2 (Growing complexity): Tách Use Cases
├── Controller → UseCase → Repository
└── Business logic vào use cases

Phase 3 (Complex domain): Full Clean Architecture
├── Presentation → Application → Domain → Infrastructure
└── Khi domain đủ phức tạp để justify overhead
```

| App complexity | Architecture |
|---|---|
| Simple CRUD | Standard MVC (Controller → Service → Repo) |
| Medium complexity | Modular with Use Cases |
| Complex domain | Clean / Hexagonal Architecture |
| Microservices | Clean Architecture per service |
