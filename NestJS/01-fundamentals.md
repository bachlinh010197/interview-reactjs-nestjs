# 📘 NestJS - Phần 1: Kiến Thức Nền Tảng

[⬅️ Mục lục](../README.md) | [Phần tiếp: Modules ➡️](./02-modules.md)

---

## 1. NestJS là gì? Tại sao nên dùng NestJS?

**Trả lời:**

NestJS là framework Node.js dùng để xây dựng server-side applications, được viết bằng TypeScript. Nó lấy cảm hứng từ Angular với kiến trúc modular, sử dụng Dependency Injection, decorators, và các design patterns.

**Tại sao dùng NestJS:**
- **TypeScript first**: Type-safe, dễ maintain
- **Kiến trúc modular**: Tổ chức code rõ ràng (Module, Controller, Service)
- **Dependency Injection**: Quản lý dependencies tự động
- **Hỗ trợ đa nền tảng HTTP**: Express (default) hoặc Fastify
- **Built-in support**: WebSocket, GraphQL, Microservices, CQRS
- **Testing**: Dễ viết unit test nhờ DI
- **CLI mạnh mẽ**: Tạo boilerplate nhanh chóng
- **Hệ sinh thái phong phú**: Passport, TypeORM, Prisma, Swagger...

---

## 2. Kiến trúc NestJS: Module - Controller - Service

**Câu hỏi: Giải thích kiến trúc cơ bản của NestJS.**

**Trả lời:**

NestJS theo mô hình **Module → Controller → Service (Provider)**:

- **Module**: Đơn vị tổ chức code, nhóm các thành phần liên quan
- **Controller**: Xử lý incoming requests, return responses
- **Service/Provider**: Chứa business logic, được inject vào controller

```typescript
// users.module.ts
@Module({
  imports: [TypeOrmModule.forFeature([User])],
  controllers: [UsersController],
  providers: [UsersService],
  exports: [UsersService],
})
export class UsersModule {}

// users.controller.ts
@Controller('users')
export class UsersController {
  constructor(private readonly usersService: UsersService) {}

  @Get()
  findAll(): Promise<User[]> {
    return this.usersService.findAll();
  }

  @Get(':id')
  findOne(@Param('id', ParseIntPipe) id: number): Promise<User> {
    return this.usersService.findOne(id);
  }

  @Post()
  create(@Body() createUserDto: CreateUserDto): Promise<User> {
    return this.usersService.create(createUserDto);
  }
}

// users.service.ts
@Injectable()
export class UsersService {
  constructor(
    @InjectRepository(User)
    private readonly userRepository: Repository<User>,
  ) {}

  findAll(): Promise<User[]> {
    return this.userRepository.find();
  }

  findOne(id: number): Promise<User> {
    return this.userRepository.findOneBy({ id });
  }

  create(dto: CreateUserDto): Promise<User> {
    const user = this.userRepository.create(dto);
    return this.userRepository.save(user);
  }
}
```

---

## 3. Dependency Injection (DI)

**Câu hỏi: Dependency Injection trong NestJS hoạt động như thế nào?**

**Trả lời:**

DI là design pattern cho phép inject dependencies từ bên ngoài thay vì tạo bên trong. NestJS sử dụng **IoC Container** để quản lý lifecycle và inject dependencies tự động.

```typescript
// NestJS tự động inject UsersService vào controller
@Controller('users')
export class UsersController {
  // Constructor injection (phổ biến nhất)
  constructor(private readonly usersService: UsersService) {}
}

// Service được đánh dấu @Injectable()
@Injectable()
export class UsersService {
  constructor(
    @InjectRepository(User)
    private readonly userRepo: Repository<User>,
    private readonly mailService: MailService,
  ) {}
}
```

**Lợi ích DI:**
- **Loose coupling**: Components không phụ thuộc trực tiếp
- **Testable**: Dễ mock dependencies trong test
- **Reusable**: Service có thể dùng ở nhiều nơi
- **Maintainable**: Thay đổi implementation không ảnh hưởng consumer

---

## 4. Decorators trong NestJS

**Câu hỏi: Giải thích các decorator quan trọng trong NestJS.**

**Trả lời:**

```typescript
// Class decorators
@Module({})           // Đánh dấu module
@Controller('path')   // Đánh dấu controller
@Injectable()         // Đánh dấu provider (service)

// Method decorators (HTTP)
@Get('path')          // GET request
@Post('path')         // POST request
@Put('path')          // PUT request
@Patch('path')        // PATCH request
@Delete('path')       // DELETE request

// Parameter decorators
@Param('id')          // Route parameter
@Query('page')        // Query parameter
@Body()               // Request body
@Headers('auth')      // Request header
@Req()                // Request object
@Res()                // Response object

// Other
@UseGuards()          // Apply guard
@UseInterceptors()    // Apply interceptor
@UsePipes()           // Apply pipe
@UseFilters()         // Apply exception filter
@SetMetadata()        // Set metadata cho guards/interceptors
```

---

## 5. Request Lifecycle trong NestJS (⭐ Rất quan trọng)

**Câu hỏi: Mô tả Request Lifecycle trong NestJS.**

**Trả lời:**

```
Incoming Request
    ↓
1. Middleware (global → module-specific)
    ↓
2. Guards (global → controller → route)
    ↓
3. Interceptors - Before (global → controller → route)
    ↓
4. Pipes (global → controller → route → param)
    ↓
5. Route Handler (Controller method)
    ↓
6. Interceptors - After (route → controller → global)
    ↓
7. Exception Filters (route → controller → global) [nếu có error]
    ↓
Outgoing Response
```

**Nhớ:** **M-G-I-P-H-I-F** (Middleware → Guards → Interceptors → Pipes → Handler → Interceptors → Filters)

---

## 6. NestJS vs Express

**Câu hỏi: So sánh NestJS và Express.**

**Trả lời:**

| Tiêu chí | Express | NestJS |
|---|---|---|
| Loại | Library (minimalist) | Framework (opinionated) |
| Ngôn ngữ | JavaScript | **TypeScript** (first-class) |
| Kiến trúc | Tự do, không có structure | Modular (Module/Controller/Service) |
| DI | Không có | **Built-in** |
| Testing | Tự setup | Built-in testing utilities |
| CLI | express-generator (cơ bản) | **@nestjs/cli** (mạnh mẽ) |
| Learning curve | Thấp | Trung bình - Cao |
| Scale | Khó quản lý khi lớn | Dễ scale nhờ modular |
| Microservices | Tự implement | Built-in support |

---

## 🔥 CÂU HỎI LEVEL MIDDLE → SENIOR

### 7. Lifecycle Hooks trong NestJS

**Câu hỏi: NestJS có những lifecycle hooks nào? Khi nào dùng?**

**Trả lời:**

```typescript
@Injectable()
export class AppService implements OnModuleInit, OnModuleDestroy, OnApplicationBootstrap, OnApplicationShutdown {

  // 1. Sau khi module được khởi tạo (dependencies resolved)
  async onModuleInit() {
    await this.connectToDatabase();
    console.log('Module initialized');
  }

  // 2. Sau khi tất cả modules được init, app sẵn sàng nhận request
  onApplicationBootstrap() {
    this.startBackgroundJobs();
    console.log('App is ready');
  }

  // 3. Khi module bị destroy
  async onModuleDestroy() {
    await this.disconnectFromDatabase();
  }

  // 4. Khi app shutdown (SIGTERM, SIGINT)
  async onApplicationShutdown(signal: string) {
    console.log(`Received ${signal}`); // e.g., 'SIGTERM'
    await this.cleanup();
  }
}

// Bật graceful shutdown trong main.ts
app.enableShutdownHooks();
```

**Thứ tự thực thi:**
```
onModuleInit → onApplicationBootstrap → ... app running ... → onModuleDestroy → onApplicationShutdown
```

---

### 8. Custom Decorators nâng cao

**Câu hỏi: Cách tạo custom decorator kết hợp nhiều decorators?**

**Trả lời:**

```typescript
// Decorator lấy current user
export const CurrentUser = createParamDecorator(
  (data: keyof User | undefined, ctx: ExecutionContext) => {
    const request = ctx.switchToHttp().getRequest();
    const user = request.user;
    return data ? user?.[data] : user;
  },
);

// Decorator kết hợp nhiều decorators (Composition)
export function Auth(...roles: Role[]) {
  return applyDecorators(
    UseGuards(JwtAuthGuard, RolesGuard),
    Roles(...roles),
    ApiBearerAuth(),
    ApiUnauthorizedResponse({ description: 'Unauthorized' }),
    ApiForbiddenResponse({ description: 'Forbidden' }),
  );
}

// Sử dụng — 1 decorator thay vì 5
@Controller('admin')
export class AdminController {
  @Get('dashboard')
  @Auth(Role.ADMIN) // ← Gọn gàng!
  getDashboard() {
    return { message: 'Admin dashboard' };
  }
}

// Custom class decorator cho logging
export function LogExecutionTime(): MethodDecorator {
  return (target, propertyKey, descriptor: PropertyDescriptor) => {
    const original = descriptor.value;
    descriptor.value = async function (...args: any[]) {
      const start = Date.now();
      const result = await original.apply(this, args);
      console.log(`${String(propertyKey)} took ${Date.now() - start}ms`);
      return result;
    };
  };
}
```

---

### 9. NestJS với Fastify — khi nào và tại sao?

**Câu hỏi: So sánh Express adapter vs Fastify adapter trong NestJS.**

**Trả lời:**

| | Express | Fastify |
|---|---|---|
| Performance | Chậm hơn | **~2x nhanh hơn** |
| Ecosystem | Rất lớn, nhiều middleware | Đang phát triển |
| Schema validation | Không built-in | JSON Schema built-in |
| TypeScript | @types/express | Native TypeScript |
| Plugin system | Middleware-based | Plugin-based (encapsulated) |

```typescript
// Chuyển sang Fastify
import { NestFactory } from '@nestjs/core';
import { FastifyAdapter, NestFastifyApplication } from '@nestjs/platform-fastify';

async function bootstrap() {
  const app = await NestFactory.create<NestFastifyApplication>(
    AppModule,
    new FastifyAdapter({ logger: true }),
  );

  // Fastify listen trên 0.0.0.0 (quan trọng cho Docker)
  await app.listen(3000, '0.0.0.0');
}
```

**Dùng Fastify khi:** High-throughput APIs, microservices, performance-critical apps.
**Giữ Express khi:** Cần middleware ecosystem lớn, team quen Express.
