# 📘 NestJS - Phần 8: Advanced Concepts

[⬅️ Auth](./07-auth.md) | [Phần tiếp: Testing ➡️](./09-testing.md)

---

## 1. Microservices

**Câu hỏi: NestJS hỗ trợ microservices như thế nào?**

**Trả lời:**

NestJS hỗ trợ nhiều transport layers: TCP, Redis, NATS, RabbitMQ, Kafka, gRPC.

```typescript
// Microservice server
async function bootstrap() {
  const app = await NestFactory.createMicroservice(AppModule, {
    transport: Transport.TCP,
    options: { host: '0.0.0.0', port: 3001 },
  });
  await app.listen();
}

// Hybrid: HTTP + Microservice
async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  app.connectMicroservice({ transport: Transport.TCP, options: { port: 3001 } });
  await app.startAllMicroservices();
  await app.listen(3000);
}

// Message Patterns
@Controller()
export class OrdersController {
  @MessagePattern({ cmd: 'create_order' })
  createOrder(@Payload() data: CreateOrderDto) {
    return this.ordersService.create(data);
  }

  @EventPattern('order_created')
  handleOrderCreated(@Payload() data: OrderCreatedEvent) {
    this.notificationService.sendOrderConfirmation(data);
  }
}
```

---

## 2. WebSocket / Gateway

**Câu hỏi: Cách implement real-time communication trong NestJS?**

**Trả lời:**

```typescript
@WebSocketGateway({ cors: { origin: '*' }, namespace: '/chat' })
export class ChatGateway implements OnGatewayInit, OnGatewayConnection, OnGatewayDisconnect {
  @WebSocketServer()
  server: Server;

  afterInit(server: Server) {
    console.log('WebSocket Gateway initialized');
  }

  handleConnection(client: Socket) {
    console.log(`Client connected: ${client.id}`);
  }

  handleDisconnect(client: Socket) {
    console.log(`Client disconnected: ${client.id}`);
  }

  @SubscribeMessage('sendMessage')
  handleMessage(
    @MessageBody() data: { room: string; message: string },
    @ConnectedSocket() client: Socket,
  ) {
    this.server.to(data.room).emit('newMessage', {
      sender: client.id,
      message: data.message,
      timestamp: new Date(),
    });
  }

  @SubscribeMessage('joinRoom')
  handleJoinRoom(
    @MessageBody() room: string,
    @ConnectedSocket() client: Socket,
  ) {
    client.join(room);
    this.server.to(room).emit('userJoined', { userId: client.id });
  }
}
```

---

## 3. Task Scheduling

**Câu hỏi: Cách implement cron jobs trong NestJS?**

**Trả lời:**

```typescript
// npm install @nestjs/schedule

@Injectable()
export class TasksService {
  private readonly logger = new Logger(TasksService.name);

  @Cron('*/30 * * * * *')       // Mỗi 30 giây
  handleHealthCheck() { this.logger.debug('Health check...'); }

  @Cron('0 2 * * *')            // Mỗi ngày lúc 2:00 AM
  async handleDailyCleanup() { await this.cleanupExpiredSessions(); }

  @Interval(10000)               // Mỗi 10 giây
  handleInterval() { this.logger.debug('Interval task'); }

  @Timeout(5000)                 // 1 lần sau 5 giây
  handleTimeout() { this.logger.debug('One-time timeout task'); }
}
```

---

## 4. Configuration Management

**Câu hỏi: Cách quản lý configuration trong NestJS?**

**Trả lời:**

```typescript
// app.module.ts
@Module({
  imports: [
    ConfigModule.forRoot({
      isGlobal: true,
      envFilePath: ['.env.local', '.env'],
      validationSchema: Joi.object({
        NODE_ENV: Joi.string().valid('development', 'production', 'test').default('development'),
        PORT: Joi.number().default(3000),
        DATABASE_URL: Joi.string().required(),
        JWT_SECRET: Joi.string().required(),
      }),
    }),
  ],
})
export class AppModule {}

// Typed configuration
export default registerAs('database', () => ({
  host: process.env.DB_HOST || 'localhost',
  port: parseInt(process.env.DB_PORT, 10) || 5432,
  username: process.env.DB_USERNAME,
  password: process.env.DB_PASSWORD,
  database: process.env.DB_DATABASE,
}));

// Inject typed config
@Injectable()
export class DatabaseService {
  constructor(
    @Inject(databaseConfig.KEY)
    private readonly dbConfig: ConfigType<typeof databaseConfig>,
  ) {
    console.log(this.dbConfig.host); // Type-safe!
  }
}
```

---

## 5. Swagger/OpenAPI Documentation

**Câu hỏi: Cách tạo API documentation tự động?**

**Trả lời:**

```typescript
// main.ts
async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  const config = new DocumentBuilder()
    .setTitle('My API')
    .setDescription('API documentation')
    .setVersion('1.0')
    .addBearerAuth()
    .build();

  const document = SwaggerModule.createDocument(app, config);
  SwaggerModule.setup('api/docs', app, document);

  await app.listen(3000);
}

// DTOs với Swagger decorators
export class CreateUserDto {
  @ApiProperty({ example: 'john@example.com', description: 'Email người dùng' })
  @IsEmail()
  email: string;

  @ApiProperty({ example: 'John Doe', minLength: 2 })
  @IsString()
  @MinLength(2)
  name: string;
}

// Controller với Swagger
@ApiTags('users')
@ApiBearerAuth()
@Controller('users')
export class UsersController {
  @Get()
  @ApiOperation({ summary: 'Lấy danh sách users' })
  @ApiResponse({ status: 200, description: 'Thành công', type: [UserResponseDto] })
  findAll(@Query() pagination: PaginationDto) {}
}
```

---

## 🔥 CÂU HỎI LEVEL MIDDLE → SENIOR

### 6. CQRS Pattern trong NestJS

**Câu hỏi: CQRS là gì? Khi nào nên áp dụng?**

**Trả lời:**

CQRS (Command Query Responsibility Segregation) tách biệt **read** và **write** operations.

```typescript
// npm install @nestjs/cqrs

// Command — thay đổi state
export class CreateOrderCommand {
  constructor(
    public readonly userId: string,
    public readonly items: OrderItemDto[],
    public readonly shippingAddress: string,
  ) {}
}

@CommandHandler(CreateOrderCommand)
export class CreateOrderHandler implements ICommandHandler<CreateOrderCommand> {
  constructor(
    private readonly orderRepo: OrderRepository,
    private readonly eventBus: EventBus,
  ) {}

  async execute(command: CreateOrderCommand): Promise<string> {
    const order = await this.orderRepo.create({
      userId: command.userId,
      items: command.items,
      status: 'pending',
    });

    // Publish event
    this.eventBus.publish(new OrderCreatedEvent(order.id, command.userId));
    return order.id;
  }
}

// Query — đọc data (có thể từ read-optimized database/cache)
export class GetOrderQuery {
  constructor(public readonly orderId: string) {}
}

@QueryHandler(GetOrderQuery)
export class GetOrderHandler implements IQueryHandler<GetOrderQuery> {
  constructor(private readonly readRepo: OrderReadRepository) {}

  async execute(query: GetOrderQuery) {
    return this.readRepo.findById(query.orderId);
  }
}

// Event — react to changes
export class OrderCreatedEvent {
  constructor(public readonly orderId: string, public readonly userId: string) {}
}

@EventsHandler(OrderCreatedEvent)
export class OrderCreatedHandler implements IEventHandler<OrderCreatedEvent> {
  async handle(event: OrderCreatedEvent) {
    await this.emailService.sendOrderConfirmation(event.userId, event.orderId);
    await this.inventoryService.reserveItems(event.orderId);
  }
}

// Controller sử dụng
@Controller('orders')
export class OrdersController {
  constructor(
    private readonly commandBus: CommandBus,
    private readonly queryBus: QueryBus,
  ) {}

  @Post()
  async createOrder(@Body() dto: CreateOrderDto, @CurrentUser() user: User) {
    return this.commandBus.execute(new CreateOrderCommand(user.id, dto.items, dto.address));
  }

  @Get(':id')
  async getOrder(@Param('id') id: string) {
    return this.queryBus.execute(new GetOrderQuery(id));
  }
}
```

**Khi nào dùng CQRS?**
- Read và Write có requirements khác nhau (e.g., read cần denormalized data)
- Cần event-driven architecture
- High-read, low-write systems
- **KHÔNG dùng** cho CRUD app đơn giản — over-engineering!

---

### 7. Event-Driven Architecture với NestJS

**Câu hỏi: Cách implement event-driven patterns?**

**Trả lời:**

```typescript
// 1. In-process events với EventEmitter2
// npm install @nestjs/event-emitter

// Emit event
@Injectable()
export class OrderService {
  constructor(private readonly eventEmitter: EventEmitter2) {}

  async createOrder(dto: CreateOrderDto): Promise<Order> {
    const order = await this.orderRepo.save(dto);

    // Fire-and-forget event
    this.eventEmitter.emit('order.created', new OrderCreatedEvent(order));

    // Hoặc wait for all listeners
    await this.eventEmitter.emitAsync('order.created', new OrderCreatedEvent(order));

    return order;
  }
}

// Listen event — loosely coupled!
@Injectable()
export class InventoryListener {
  @OnEvent('order.created')
  async handleOrderCreated(event: OrderCreatedEvent) {
    await this.inventoryService.reserveStock(event.order.items);
  }
}

@Injectable()
export class EmailListener {
  @OnEvent('order.created', { async: true })
  async handleOrderCreated(event: OrderCreatedEvent) {
    await this.emailService.sendConfirmation(event.order.userId);
  }
}

@Injectable()
export class AnalyticsListener {
  @OnEvent('order.*') // Wildcard — listen tất cả order events
  handleOrderEvent(event: any) {
    this.analyticsService.track(event);
  }
}
```

---

### 8. Graceful Shutdown & Health Monitoring

**Câu hỏi: Cách xử lý graceful shutdown trong production?**

**Trả lời:**

```typescript
// main.ts
async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  // Bật shutdown hooks
  app.enableShutdownHooks();

  // Graceful shutdown timeout
  const server = await app.listen(3000);
  server.setTimeout(30000); // 30s timeout cho pending requests

  // Handle uncaught exceptions
  process.on('unhandledRejection', (reason) => {
    console.error('Unhandled Rejection:', reason);
  });
}

// Service cleanup
@Injectable()
export class AppService implements OnApplicationShutdown {
  async onApplicationShutdown(signal: string) {
    console.log(`Shutting down: ${signal}`);

    // 1. Stop accepting new requests (handled by NestJS)
    // 2. Wait for pending requests to complete
    // 3. Close database connections
    await this.dataSource.destroy();
    // 4. Close Redis connections
    await this.redisClient.quit();
    // 5. Flush logs
    await this.logger.flush();
  }
}
```
```
