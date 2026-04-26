# 📘 TÀI LIỆU PHỎNG VẤN NESTJS

> Tài liệu tổng hợp câu hỏi & trả lời phỏng vấn NestJS từ cơ bản đến nâng cao.

---

## I. KIẾN THỨC NỀN TẢNG (FUNDAMENTALS)

### 1. NestJS là gì? Tại sao nên dùng NestJS?

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

### 2. Kiến trúc NestJS: Module - Controller - Service

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
  exports: [UsersService], // Cho phép module khác dùng UsersService
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

### 3. Dependency Injection (DI)

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
    private readonly mailService: MailService, // DI tự động
  ) {}
}
```

**Lợi ích DI:**
- **Loose coupling**: Components không phụ thuộc trực tiếp
- **Testable**: Dễ mock dependencies trong test
- **Reusable**: Service có thể dùng ở nhiều nơi
- **Maintainable**: Thay đổi implementation không ảnh hưởng consumer

---

### 4. Decorators trong NestJS

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

// Ví dụ tổng hợp
@Controller('products')
@UseGuards(JwtAuthGuard)
export class ProductsController {
  @Get()
  @UseInterceptors(CacheInterceptor)
  findAll(
    @Query('page', new DefaultValuePipe(1), ParseIntPipe) page: number,
    @Query('limit', new DefaultValuePipe(10), ParseIntPipe) limit: number,
  ) {
    return this.productsService.paginate({ page, limit });
  }

  @Post()
  @Roles('admin')
  @UseGuards(RolesGuard)
  create(@Body(ValidationPipe) dto: CreateProductDto) {
    return this.productsService.create(dto);
  }
}
```

---

### 5. Request Lifecycle trong NestJS

**Câu hỏi: Mô tả Request Lifecycle trong NestJS.**

**Trả lời:**

Thứ tự xử lý request trong NestJS:

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

---

### 6. NestJS vs Express

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

## II. MODULES

### 1. Các loại Module

**Câu hỏi: NestJS có những loại module nào?**

**Trả lời:**

```typescript
// 1. Feature Module: Nhóm logic theo tính năng
@Module({
  imports: [TypeOrmModule.forFeature([User])],
  controllers: [UsersController],
  providers: [UsersService],
  exports: [UsersService],
})
export class UsersModule {}

// 2. Shared Module: Module được nhiều module khác import
@Module({
  providers: [HelperService, LoggerService],
  exports: [HelperService, LoggerService],
})
export class SharedModule {}

// 3. Global Module: Tự động available ở mọi nơi
@Global()
@Module({
  providers: [ConfigService],
  exports: [ConfigService],
})
export class ConfigModule {}

// 4. Dynamic Module: Module có thể cấu hình
@Module({})
export class DatabaseModule {
  static forRoot(options: DatabaseOptions): DynamicModule {
    return {
      module: DatabaseModule,
      global: true,
      providers: [
        {
          provide: 'DATABASE_OPTIONS',
          useValue: options,
        },
        DatabaseService,
      ],
      exports: [DatabaseService],
    };
  }

  static forFeature(entities: any[]): DynamicModule {
    return {
      module: DatabaseModule,
      providers: entities.map(entity => ({
        provide: `${entity.name}_REPOSITORY`,
        useFactory: (db: DatabaseService) => db.getRepository(entity),
        inject: [DatabaseService],
      })),
      exports: entities.map(entity => `${entity.name}_REPOSITORY`),
    };
  }
}

// Sử dụng Dynamic Module
@Module({
  imports: [
    DatabaseModule.forRoot({ host: 'localhost', port: 5432 }),
    DatabaseModule.forFeature([User, Product]),
  ],
})
export class AppModule {}
```

---

### 2. Circular Dependency

**Câu hỏi: Circular dependency là gì? Cách xử lý trong NestJS?**

**Trả lời:**

Circular dependency xảy ra khi Module A import Module B, đồng thời Module B cũng import Module A.

```typescript
// Cách xử lý: forwardRef()

// Module level
@Module({
  imports: [forwardRef(() => ModuleB)],
  exports: [ServiceA],
})
export class ModuleA {}

@Module({
  imports: [forwardRef(() => ModuleA)],
  exports: [ServiceB],
})
export class ModuleB {}

// Provider level
@Injectable()
export class ServiceA {
  constructor(
    @Inject(forwardRef(() => ServiceB))
    private readonly serviceB: ServiceB,
  ) {}
}
```

**Best practice:** Tránh circular dependency bằng cách tách logic chung ra shared module.

---

## III. CONTROLLERS & ROUTING

### 1. DTOs và Validation

**Câu hỏi: DTO là gì? Cách validate request data?**

**Trả lời:**

DTO (Data Transfer Object) định nghĩa cấu trúc dữ liệu truyền qua network. Kết hợp với `class-validator` và `class-transformer` để validate.

```typescript
// create-user.dto.ts
import { IsEmail, IsString, MinLength, IsOptional, IsEnum } from 'class-validator';
import { Transform } from 'class-transformer';

export class CreateUserDto {
  @IsString()
  @MinLength(2, { message: 'Tên phải có ít nhất 2 ký tự' })
  name: string;

  @IsEmail({}, { message: 'Email không hợp lệ' })
  @Transform(({ value }) => value.toLowerCase().trim())
  email: string;

  @IsString()
  @MinLength(8)
  password: string;

  @IsOptional()
  @IsEnum(UserRole)
  role?: UserRole;
}

// update-user.dto.ts - Dùng PartialType để tất cả fields optional
import { PartialType } from '@nestjs/mapped-types';

export class UpdateUserDto extends PartialType(CreateUserDto) {}

// Pagination DTO
export class PaginationDto {
  @IsOptional()
  @Type(() => Number)
  @IsInt()
  @Min(1)
  page?: number = 1;

  @IsOptional()
  @Type(() => Number)
  @IsInt()
  @Min(1)
  @Max(100)
  limit?: number = 10;
}

// Bật global validation pipe trong main.ts
async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  app.useGlobalPipes(new ValidationPipe({
    whitelist: true,          // Loại bỏ properties không có trong DTO
    forbidNonWhitelisted: true, // Throw error nếu có property lạ
    transform: true,          // Tự động transform type
    transformOptions: {
      enableImplicitConversion: true,
    },
  }));
  await app.listen(3000);
}
```

---

### 2. File Upload

**Câu hỏi: Cách xử lý file upload trong NestJS?**

**Trả lời:**

```typescript
import { FileInterceptor, FilesInterceptor } from '@nestjs/platform-express';
import { diskStorage } from 'multer';
import { extname } from 'path';

@Controller('upload')
export class UploadController {
  // Upload single file
  @Post('single')
  @UseInterceptors(FileInterceptor('file', {
    storage: diskStorage({
      destination: './uploads',
      filename: (req, file, cb) => {
        const uniqueSuffix = Date.now() + '-' + Math.round(Math.random() * 1e9);
        cb(null, `${uniqueSuffix}${extname(file.originalname)}`);
      },
    }),
    fileFilter: (req, file, cb) => {
      if (!file.mimetype.match(/\/(jpg|jpeg|png|gif)$/)) {
        cb(new BadRequestException('Only image files are allowed'), false);
      }
      cb(null, true);
    },
    limits: { fileSize: 5 * 1024 * 1024 }, // 5MB
  }))
  uploadFile(@UploadedFile() file: Express.Multer.File) {
    return { filename: file.filename, size: file.size };
  }

  // Upload multiple files
  @Post('multiple')
  @UseInterceptors(FilesInterceptor('files', 10))
  uploadFiles(@UploadedFiles() files: Express.Multer.File[]) {
    return files.map(f => ({ filename: f.filename, size: f.size }));
  }
}
```

---

## IV. PROVIDERS & SERVICES

### 1. Custom Providers

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
      inject: [ConfigService], // Dependencies cho factory function
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

### 2. Provider Scopes

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

## V. MIDDLEWARE, GUARDS, INTERCEPTORS, PIPES, FILTERS

### 1. Middleware

**Câu hỏi: Middleware trong NestJS là gì?**

**Trả lời:**

Middleware chạy **trước route handler**, có access vào `request`, `response`, và `next()`. Giống Express middleware.

```typescript
// logger.middleware.ts
@Injectable()
export class LoggerMiddleware implements NestMiddleware {
  use(req: Request, res: Response, next: NextFunction) {
    const start = Date.now();
    console.log(`[${req.method}] ${req.url} - Start`);

    res.on('finish', () => {
      const duration = Date.now() - start;
      console.log(`[${req.method}] ${req.url} - ${res.statusCode} - ${duration}ms`);
    });

    next();
  }
}

// Apply middleware trong module
@Module({})
export class AppModule implements NestModule {
  configure(consumer: MiddlewareConsumer) {
    consumer
      .apply(LoggerMiddleware, CorsMiddleware)
      .exclude(
        { path: 'health', method: RequestMethod.GET },
      )
      .forRoutes('*'); // Hoặc specific controller: UsersController
  }
}

// Function middleware (đơn giản)
export function helmet(req: Request, res: Response, next: NextFunction) {
  // security headers
  next();
}
```

---

### 2. Guards

**Câu hỏi: Guards là gì? Cách implement authentication và authorization?**

**Trả lời:**

Guards quyết định request có được xử lý hay không (true/false). Chạy **sau middleware, trước interceptors**.

```typescript
// jwt-auth.guard.ts
@Injectable()
export class JwtAuthGuard extends AuthGuard('jwt') {
  canActivate(context: ExecutionContext) {
    return super.canActivate(context);
  }

  handleRequest(err: any, user: any) {
    if (err || !user) {
      throw new UnauthorizedException('Token không hợp lệ');
    }
    return user;
  }
}

// roles.guard.ts
@Injectable()
export class RolesGuard implements CanActivate {
  constructor(private readonly reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    const requiredRoles = this.reflector.getAllAndOverride<string[]>('roles', [
      context.getHandler(),
      context.getClass(),
    ]);

    if (!requiredRoles) return true;

    const { user } = context.switchToHttp().getRequest();
    return requiredRoles.some(role => user.roles?.includes(role));
  }
}

// roles.decorator.ts
export const Roles = (...roles: string[]) => SetMetadata('roles', roles);

// Sử dụng
@Controller('admin')
@UseGuards(JwtAuthGuard, RolesGuard)
export class AdminController {
  @Get('dashboard')
  @Roles('admin')
  getDashboard() {
    return { message: 'Admin dashboard' };
  }

  @Get('reports')
  @Roles('admin', 'manager')
  getReports() {
    return { message: 'Reports' };
  }
}
```

---

### 3. Interceptors

**Câu hỏi: Interceptors dùng để làm gì? Cho ví dụ.**

**Trả lời:**

Interceptors xử lý logic **trước và sau** route handler. Use cases: logging, transform response, caching, timeout.

```typescript
// transform-response.interceptor.ts
@Injectable()
export class TransformInterceptor<T> implements NestInterceptor<T, Response<T>> {
  intercept(context: ExecutionContext, next: CallHandler): Observable<Response<T>> {
    return next.handle().pipe(
      map(data => ({
        statusCode: context.switchToHttp().getResponse().statusCode,
        message: 'Success',
        data,
        timestamp: new Date().toISOString(),
      })),
    );
  }
}

// logging.interceptor.ts
@Injectable()
export class LoggingInterceptor implements NestInterceptor {
  private readonly logger = new Logger(LoggingInterceptor.name);

  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    const request = context.switchToHttp().getRequest();
    const { method, url } = request;
    const now = Date.now();

    return next.handle().pipe(
      tap(() => {
        this.logger.log(`${method} ${url} - ${Date.now() - now}ms`);
      }),
    );
  }
}

// timeout.interceptor.ts
@Injectable()
export class TimeoutInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    return next.handle().pipe(
      timeout(5000),
      catchError(err => {
        if (err instanceof TimeoutError) {
          throw new RequestTimeoutException();
        }
        throw err;
      }),
    );
  }
}

// cache.interceptor.ts
@Injectable()
export class HttpCacheInterceptor implements NestInterceptor {
  constructor(private readonly cacheManager: Cache) {}

  async intercept(context: ExecutionContext, next: CallHandler): Promise<Observable<any>> {
    const request = context.switchToHttp().getRequest();
    const key = `cache:${request.url}`;

    const cached = await this.cacheManager.get(key);
    if (cached) return of(cached);

    return next.handle().pipe(
      tap(data => this.cacheManager.set(key, data, 60)), // TTL 60s
    );
  }
}

// Apply globally
app.useGlobalInterceptors(new TransformInterceptor());
```

---

### 4. Pipes

**Câu hỏi: Pipes trong NestJS dùng để làm gì?**

**Trả lời:**

Pipes có 2 mục đích: **Transformation** (chuyển đổi data) và **Validation** (kiểm tra data).

```typescript
// Built-in pipes
@Get(':id')
findOne(@Param('id', ParseIntPipe) id: number) {
  // id đã được convert từ string → number
  // Nếu không phải number → throw BadRequestException
}

@Get()
findAll(
  @Query('page', new DefaultValuePipe(1), ParseIntPipe) page: number,
  @Query('active', new DefaultValuePipe(true), ParseBoolPipe) active: boolean,
) {}

// Custom Validation Pipe
@Injectable()
export class ParseDatePipe implements PipeTransform<string, Date> {
  transform(value: string, metadata: ArgumentMetadata): Date {
    const date = new Date(value);
    if (isNaN(date.getTime())) {
      throw new BadRequestException(`"${value}" is not a valid date`);
    }
    return date;
  }
}

// Custom Transformation Pipe
@Injectable()
export class TrimPipe implements PipeTransform {
  transform(value: any) {
    if (typeof value === 'string') return value.trim();
    if (typeof value === 'object' && value !== null) {
      Object.keys(value).forEach(key => {
        if (typeof value[key] === 'string') {
          value[key] = value[key].trim();
        }
      });
    }
    return value;
  }
}

// Sử dụng
@Get('events')
findByDate(@Query('date', ParseDatePipe) date: Date) {
  return this.eventsService.findByDate(date);
}
```

---

### 5. Exception Filters

**Câu hỏi: Cách xử lý errors/exceptions trong NestJS?**

**Trả lời:**

```typescript
// Built-in exceptions
throw new BadRequestException('Dữ liệu không hợp lệ');
throw new UnauthorizedException('Chưa đăng nhập');
throw new ForbiddenException('Không có quyền');
throw new NotFoundException('Không tìm thấy');
throw new ConflictException('Email đã tồn tại');
throw new InternalServerErrorException('Lỗi server');

// Custom Exception Filter
@Catch()
export class AllExceptionsFilter implements ExceptionFilter {
  private readonly logger = new Logger(AllExceptionsFilter.name);

  catch(exception: unknown, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse<Response>();
    const request = ctx.getRequest<Request>();

    let status = HttpStatus.INTERNAL_SERVER_ERROR;
    let message = 'Internal server error';

    if (exception instanceof HttpException) {
      status = exception.getStatus();
      const exResponse = exception.getResponse();
      message = typeof exResponse === 'string'
        ? exResponse
        : (exResponse as any).message;
    }

    this.logger.error(`${request.method} ${request.url} - ${status}`, exception);

    response.status(status).json({
      statusCode: status,
      message,
      timestamp: new Date().toISOString(),
      path: request.url,
    });
  }
}

// Custom business exception
export class BusinessException extends HttpException {
  constructor(message: string, errorCode: string) {
    super({ message, errorCode, statusCode: 422 }, 422);
  }
}

// Apply globally
app.useGlobalFilters(new AllExceptionsFilter());
```

---

### 6. Execution Order (Rất quan trọng!)

**Câu hỏi: Thứ tự thực thi của Middleware, Guards, Interceptors, Pipes, Filters?**

**Trả lời:**

```
Request → Middleware → Guards → Interceptors (before) → Pipes → Handler → Interceptors (after) → Response
                                                                    ↓ (nếu error)
                                                              Exception Filters
```

**Nhớ:** **M-G-I-P-H-I-F** (Middleware → Guards → Interceptors → Pipes → Handler → Interceptors → Filters)

**Scope priority:** Global → Controller → Route handler

---

## VI. DATABASE & ORM

### 1. TypeORM Integration

**Câu hỏi: Cách tích hợp TypeORM trong NestJS?**

**Trả lời:**

```typescript
// app.module.ts
@Module({
  imports: [
    TypeOrmModule.forRoot({
      type: 'postgres',
      host: 'localhost',
      port: 5432,
      username: 'postgres',
      password: 'password',
      database: 'mydb',
      entities: [__dirname + '/**/*.entity{.ts,.js}'],
      synchronize: false, // ❌ KHÔNG dùng true trong production
      migrations: ['dist/migrations/*{.ts,.js}'],
    }),
    UsersModule,
  ],
})
export class AppModule {}

// user.entity.ts
@Entity('users')
export class User {
  @PrimaryGeneratedColumn()
  id: number;

  @Column({ unique: true })
  email: string;

  @Column()
  name: string;

  @Column({ select: false }) // Không trả về khi query
  password: string;

  @Column({ type: 'enum', enum: UserRole, default: UserRole.USER })
  role: UserRole;

  @CreateDateColumn()
  createdAt: Date;

  @UpdateDateColumn()
  updatedAt: Date;

  @DeleteDateColumn() // Soft delete
  deletedAt: Date;

  // Relations
  @OneToMany(() => Post, post => post.author)
  posts: Post[];

  @ManyToMany(() => Role, role => role.users)
  @JoinTable()
  roles: Role[];
}

// users.service.ts
@Injectable()
export class UsersService {
  constructor(
    @InjectRepository(User)
    private readonly userRepo: Repository<User>,
  ) {}

  // Repository methods
  async findAll(paginationDto: PaginationDto) {
    const { page, limit } = paginationDto;
    const [items, total] = await this.userRepo.findAndCount({
      skip: (page - 1) * limit,
      take: limit,
      relations: ['posts', 'roles'],
      order: { createdAt: 'DESC' },
    });
    return { items, total, page, limit, totalPages: Math.ceil(total / limit) };
  }

  // Query Builder (cho query phức tạp)
  async search(keyword: string) {
    return this.userRepo
      .createQueryBuilder('user')
      .leftJoinAndSelect('user.posts', 'post')
      .where('user.name ILIKE :keyword', { keyword: `%${keyword}%` })
      .orWhere('user.email ILIKE :keyword', { keyword: `%${keyword}%` })
      .orderBy('user.createdAt', 'DESC')
      .getMany();
  }

  // Transaction
  async transferCredits(fromId: number, toId: number, amount: number) {
    return this.userRepo.manager.transaction(async (manager) => {
      const from = await manager.findOneBy(User, { id: fromId });
      const to = await manager.findOneBy(User, { id: toId });

      if (from.credits < amount) {
        throw new BadRequestException('Không đủ credits');
      }

      from.credits -= amount;
      to.credits += amount;

      await manager.save([from, to]);
    });
  }
}
```

---

### 2. Prisma Integration

**Câu hỏi: So sánh TypeORM và Prisma. Cách dùng Prisma trong NestJS?**

**Trả lời:**

| | TypeORM | Prisma |
|---|---|---|
| Schema | Decorators trên Entity class | Schema file riêng (.prisma) |
| Migrations | CLI hoặc auto-sync | `prisma migrate` (mạnh mẽ) |
| Type safety | Tốt nhưng có edge cases | **Excellent** (generated types) |
| Query | Repository + QueryBuilder | Prisma Client (intuitive API) |
| Performance | Tốt | Tốt hơn nhờ query optimization |
| Learning curve | Trung bình | Thấp |

```typescript
// prisma.service.ts
@Injectable()
export class PrismaService extends PrismaClient implements OnModuleInit, OnModuleDestroy {
  async onModuleInit() {
    await this.$connect();
  }

  async onModuleDestroy() {
    await this.$disconnect();
  }
}

// users.service.ts (Prisma)
@Injectable()
export class UsersService {
  constructor(private readonly prisma: PrismaService) {}

  async findAll(params: { page: number; limit: number }) {
    const { page, limit } = params;
    const [items, total] = await Promise.all([
      this.prisma.user.findMany({
        skip: (page - 1) * limit,
        take: limit,
        include: { posts: true },
        orderBy: { createdAt: 'desc' },
      }),
      this.prisma.user.count(),
    ]);
    return { items, total, page, limit };
  }

  // Transaction
  async transfer(fromId: number, toId: number, amount: number) {
    return this.prisma.$transaction([
      this.prisma.user.update({
        where: { id: fromId },
        data: { credits: { decrement: amount } },
      }),
      this.prisma.user.update({
        where: { id: toId },
        data: { credits: { increment: amount } },
      }),
    ]);
  }
}
```

---

## VII. AUTHENTICATION & AUTHORIZATION

### 1. JWT Authentication

**Câu hỏi: Cách implement JWT authentication trong NestJS?**

**Trả lời:**

```typescript
// auth.module.ts
@Module({
  imports: [
    UsersModule,
    PassportModule,
    JwtModule.registerAsync({
      imports: [ConfigModule],
      useFactory: (config: ConfigService) => ({
        secret: config.get('JWT_SECRET'),
        signOptions: { expiresIn: '15m' },
      }),
      inject: [ConfigService],
    }),
  ],
  controllers: [AuthController],
  providers: [AuthService, LocalStrategy, JwtStrategy],
  exports: [AuthService],
})
export class AuthModule {}

// auth.service.ts
@Injectable()
export class AuthService {
  constructor(
    private readonly usersService: UsersService,
    private readonly jwtService: JwtService,
  ) {}

  async validateUser(email: string, password: string): Promise<User | null> {
    const user = await this.usersService.findByEmail(email);
    if (user && await bcrypt.compare(password, user.password)) {
      return user;
    }
    return null;
  }

  async login(user: User) {
    const payload = { sub: user.id, email: user.email, role: user.role };

    return {
      accessToken: this.jwtService.sign(payload),
      refreshToken: this.jwtService.sign(payload, { expiresIn: '7d' }),
    };
  }

  async refreshToken(token: string) {
    try {
      const payload = this.jwtService.verify(token);
      const user = await this.usersService.findById(payload.sub);
      return this.login(user);
    } catch {
      throw new UnauthorizedException('Invalid refresh token');
    }
  }
}

// local.strategy.ts (username/password login)
@Injectable()
export class LocalStrategy extends PassportStrategy(Strategy) {
  constructor(private readonly authService: AuthService) {
    super({ usernameField: 'email' });
  }

  async validate(email: string, password: string): Promise<User> {
    const user = await this.authService.validateUser(email, password);
    if (!user) {
      throw new UnauthorizedException('Email hoặc mật khẩu không đúng');
    }
    return user;
  }
}

// jwt.strategy.ts (protect routes)
@Injectable()
export class JwtStrategy extends PassportStrategy(Strategy) {
  constructor(private readonly config: ConfigService) {
    super({
      jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken(),
      ignoreExpiration: false,
      secretOrKey: config.get('JWT_SECRET'),
    });
  }

  async validate(payload: any) {
    return { id: payload.sub, email: payload.email, role: payload.role };
  }
}

// auth.controller.ts
@Controller('auth')
export class AuthController {
  constructor(private readonly authService: AuthService) {}

  @Post('login')
  @UseGuards(LocalAuthGuard)
  async login(@Req() req) {
    return this.authService.login(req.user);
  }

  @Post('register')
  async register(@Body(ValidationPipe) dto: RegisterDto) {
    return this.authService.register(dto);
  }

  @Post('refresh')
  async refresh(@Body('refreshToken') token: string) {
    return this.authService.refreshToken(token);
  }

  @Get('profile')
  @UseGuards(JwtAuthGuard)
  getProfile(@Req() req) {
    return req.user;
  }
}

// Custom decorator để lấy current user
export const CurrentUser = createParamDecorator(
  (data: string, ctx: ExecutionContext) => {
    const request = ctx.switchToHttp().getRequest();
    const user = request.user;
    return data ? user?.[data] : user;
  },
);

// Sử dụng
@Get('profile')
@UseGuards(JwtAuthGuard)
getProfile(@CurrentUser() user: User) {
  return user;
}

@Get('my-email')
@UseGuards(JwtAuthGuard)
getEmail(@CurrentUser('email') email: string) {
  return { email };
}
```

---

### 2. Role-Based Access Control (RBAC)

**Câu hỏi: Cách implement RBAC trong NestJS?**

**Trả lời:**

```typescript
// roles.enum.ts
export enum Role {
  USER = 'user',
  ADMIN = 'admin',
  MANAGER = 'manager',
}

// roles.decorator.ts
export const ROLES_KEY = 'roles';
export const Roles = (...roles: Role[]) => SetMetadata(ROLES_KEY, roles);

// roles.guard.ts
@Injectable()
export class RolesGuard implements CanActivate {
  constructor(private readonly reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    const requiredRoles = this.reflector.getAllAndOverride<Role[]>(ROLES_KEY, [
      context.getHandler(),
      context.getClass(),
    ]);

    if (!requiredRoles) return true; // No roles required = public

    const { user } = context.switchToHttp().getRequest();
    return requiredRoles.some(role => user.role === role);
  }
}

// Sử dụng
@Controller('users')
@UseGuards(JwtAuthGuard, RolesGuard)
export class UsersController {
  @Get()
  @Roles(Role.ADMIN)
  findAll() { /* Chỉ admin */ }

  @Delete(':id')
  @Roles(Role.ADMIN)
  remove(@Param('id') id: string) { /* Chỉ admin */ }

  @Get('profile')
  // Không có @Roles → mọi authenticated user đều access được
  getProfile(@CurrentUser() user: User) {
    return user;
  }
}
```

---

## VIII. ADVANCED CONCEPTS

### 1. Microservices

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
  // Request-Response pattern
  @MessagePattern({ cmd: 'create_order' })
  createOrder(@Payload() data: CreateOrderDto) {
    return this.ordersService.create(data);
  }

  // Event-based pattern
  @EventPattern('order_created')
  handleOrderCreated(@Payload() data: OrderCreatedEvent) {
    this.notificationService.sendOrderConfirmation(data);
  }
}

// Client - gọi microservice từ HTTP controller
@Controller('orders')
export class OrdersHttpController {
  constructor(@Inject('ORDER_SERVICE') private readonly client: ClientProxy) {}

  @Post()
  createOrder(@Body() dto: CreateOrderDto) {
    return this.client.send({ cmd: 'create_order' }, dto);
  }
}
```

---

### 2. WebSocket / Gateway

**Câu hỏi: Cách implement real-time communication trong NestJS?**

**Trả lời:**

```typescript
@WebSocketGateway({
  cors: { origin: '*' },
  namespace: '/chat',
})
export class ChatGateway implements OnGatewayInit, OnGatewayConnection, OnGatewayDisconnect {
  @WebSocketServer()
  server: Server;

  private logger = new Logger('ChatGateway');

  afterInit(server: Server) {
    this.logger.log('WebSocket Gateway initialized');
  }

  handleConnection(client: Socket) {
    this.logger.log(`Client connected: ${client.id}`);
  }

  handleDisconnect(client: Socket) {
    this.logger.log(`Client disconnected: ${client.id}`);
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

### 3. Task Scheduling

**Câu hỏi: Cách implement cron jobs trong NestJS?**

**Trả lời:**

```typescript
// Cài đặt: npm install @nestjs/schedule

@Injectable()
export class TasksService {
  private readonly logger = new Logger(TasksService.name);

  // Chạy mỗi 30 giây
  @Cron('*/30 * * * * *')
  handleHealthCheck() {
    this.logger.debug('Health check running...');
  }

  // Chạy mỗi ngày lúc 2:00 AM
  @Cron('0 2 * * *')
  async handleDailyCleanup() {
    await this.cleanupExpiredSessions();
  }

  // Chạy mỗi 10 giây
  @Interval(10000)
  handleInterval() {
    this.logger.debug('Interval task');
  }

  // Chạy 1 lần sau 5 giây
  @Timeout(5000)
  handleTimeout() {
    this.logger.debug('One-time timeout task');
  }
}
```

---

### 4. Configuration Management

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
        JWT_EXPIRATION: Joi.string().default('15m'),
      }),
    }),
  ],
})
export class AppModule {}

// Typed configuration
// database.config.ts
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

### 5. Swagger/OpenAPI Documentation

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
    .addTag('users')
    .addTag('products')
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

  @ApiPropertyOptional({ enum: UserRole, default: UserRole.USER })
  @IsOptional()
  @IsEnum(UserRole)
  role?: UserRole;
}

// Controller với Swagger
@ApiTags('users')
@ApiBearerAuth()
@Controller('users')
export class UsersController {
  @Get()
  @ApiOperation({ summary: 'Lấy danh sách users' })
  @ApiQuery({ name: 'page', required: false, type: Number })
  @ApiResponse({ status: 200, description: 'Thành công', type: [UserResponseDto] })
  findAll(@Query() pagination: PaginationDto) {}

  @Post()
  @ApiOperation({ summary: 'Tạo user mới' })
  @ApiResponse({ status: 201, description: 'Tạo thành công' })
  @ApiResponse({ status: 400, description: 'Dữ liệu không hợp lệ' })
  @ApiResponse({ status: 409, description: 'Email đã tồn tại' })
  create(@Body() dto: CreateUserDto) {}
}
```

---

## IX. TESTING

### 1. Unit Testing

**Câu hỏi: Cách viết unit test trong NestJS?**

**Trả lời:**

```typescript
// users.service.spec.ts
describe('UsersService', () => {
  let service: UsersService;
  let repository: Repository<User>;

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      providers: [
        UsersService,
        {
          provide: getRepositoryToken(User),
          useValue: {
            find: jest.fn(),
            findOneBy: jest.fn(),
            create: jest.fn(),
            save: jest.fn(),
            delete: jest.fn(),
          },
        },
      ],
    }).compile();

    service = module.get<UsersService>(UsersService);
    repository = module.get<Repository<User>>(getRepositoryToken(User));
  });

  describe('findAll', () => {
    it('should return array of users', async () => {
      const users = [{ id: 1, name: 'John', email: 'john@test.com' }];
      jest.spyOn(repository, 'find').mockResolvedValue(users as User[]);

      const result = await service.findAll();
      expect(result).toEqual(users);
      expect(repository.find).toHaveBeenCalled();
    });
  });

  describe('findOne', () => {
    it('should return a user', async () => {
      const user = { id: 1, name: 'John' };
      jest.spyOn(repository, 'findOneBy').mockResolvedValue(user as User);

      const result = await service.findOne(1);
      expect(result).toEqual(user);
    });

    it('should throw NotFoundException', async () => {
      jest.spyOn(repository, 'findOneBy').mockResolvedValue(null);
      await expect(service.findOne(999)).rejects.toThrow(NotFoundException);
    });
  });
});

// users.controller.spec.ts
describe('UsersController', () => {
  let controller: UsersController;
  let service: UsersService;

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      controllers: [UsersController],
      providers: [
        {
          provide: UsersService,
          useValue: {
            findAll: jest.fn().mockResolvedValue([]),
            findOne: jest.fn().mockResolvedValue({ id: 1, name: 'John' }),
            create: jest.fn().mockResolvedValue({ id: 1, name: 'John' }),
          },
        },
      ],
    }).compile();

    controller = module.get<UsersController>(UsersController);
    service = module.get<UsersService>(UsersService);
  });

  it('should return all users', async () => {
    const result = await controller.findAll();
    expect(result).toEqual([]);
    expect(service.findAll).toHaveBeenCalled();
  });
});
```

---

### 2. E2E Testing

**Câu hỏi: Cách viết E2E test?**

**Trả lời:**

```typescript
// test/users.e2e-spec.ts
describe('UsersController (e2e)', () => {
  let app: INestApplication;
  let authToken: string;

  beforeAll(async () => {
    const moduleFixture: TestingModule = await Test.createTestingModule({
      imports: [AppModule],
    }).compile();

    app = moduleFixture.createNestApplication();
    app.useGlobalPipes(new ValidationPipe({ whitelist: true }));
    await app.init();

    // Login to get token
    const loginRes = await request(app.getHttpServer())
      .post('/auth/login')
      .send({ email: 'admin@test.com', password: 'password' });
    authToken = loginRes.body.accessToken;
  });

  afterAll(async () => {
    await app.close();
  });

  describe('GET /users', () => {
    it('should return 401 without token', () => {
      return request(app.getHttpServer())
        .get('/users')
        .expect(401);
    });

    it('should return users list', () => {
      return request(app.getHttpServer())
        .get('/users')
        .set('Authorization', `Bearer ${authToken}`)
        .expect(200)
        .expect(res => {
          expect(Array.isArray(res.body)).toBe(true);
        });
    });
  });

  describe('POST /users', () => {
    it('should create a user', () => {
      return request(app.getHttpServer())
        .post('/users')
        .set('Authorization', `Bearer ${authToken}`)
        .send({ name: 'Test User', email: 'test@test.com', password: '12345678' })
        .expect(201)
        .expect(res => {
          expect(res.body.email).toBe('test@test.com');
        });
    });

    it('should reject invalid data', () => {
      return request(app.getHttpServer())
        .post('/users')
        .set('Authorization', `Bearer ${authToken}`)
        .send({ name: '' }) // Invalid
        .expect(400);
    });
  });
});
```

---

## X. CÂU HỎI TÌNH HUỐNG THỰC TẾ

### 1. Thiết kế REST API cho hệ thống E-commerce

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

### 2. Cách xử lý Rate Limiting

**Trả lời:**

```typescript
// npm install @nestjs/throttler

@Module({
  imports: [
    ThrottlerModule.forRoot([
      {
        name: 'short',
        ttl: 1000,    // 1 giây
        limit: 3,      // 3 requests
      },
      {
        name: 'medium',
        ttl: 10000,   // 10 giây
        limit: 20,
      },
      {
        name: 'long',
        ttl: 60000,   // 1 phút
        limit: 100,
      },
    ]),
  ],
  providers: [
    {
      provide: APP_GUARD,
      useClass: ThrottlerGuard,
    },
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

### 3. Cách xử lý Long-running Tasks

**Trả lời:**

Dùng **Bull Queue** (Redis-based job queue):

```typescript
// npm install @nestjs/bull bull

// app.module.ts
@Module({
  imports: [
    BullModule.forRoot({
      redis: { host: 'localhost', port: 6379 },
    }),
    BullModule.registerQueue({ name: 'email' }),
    BullModule.registerQueue({ name: 'report' }),
  ],
})
export class AppModule {}

// email.processor.ts
@Processor('email')
export class EmailProcessor {
  private readonly logger = new Logger(EmailProcessor.name);

  @Process('send-welcome')
  async handleWelcomeEmail(job: Job<{ email: string; name: string }>) {
    this.logger.log(`Sending welcome email to ${job.data.email}`);
    await this.mailService.sendWelcome(job.data);
  }

  @Process('send-report')
  async handleReport(job: Job) {
    // Long-running task: generate PDF report
    await this.reportService.generatePdf(job.data);
  }

  @OnQueueCompleted()
  onCompleted(job: Job) {
    this.logger.log(`Job ${job.id} completed`);
  }

  @OnQueueFailed()
  onFailed(job: Job, error: Error) {
    this.logger.error(`Job ${job.id} failed: ${error.message}`);
  }
}

// Sử dụng trong service
@Injectable()
export class UsersService {
  constructor(@InjectQueue('email') private readonly emailQueue: Queue) {}

  async register(dto: RegisterDto) {
    const user = await this.create(dto);

    // Add job to queue (non-blocking)
    await this.emailQueue.add('send-welcome', {
      email: user.email,
      name: user.name,
    }, {
      attempts: 3,           // Retry 3 lần nếu fail
      backoff: 5000,         // Đợi 5s giữa các retry
      removeOnComplete: true,
    });

    return user;
  }
}
```

---

### 4. Cách implement Caching

**Trả lời:**

```typescript
// npm install @nestjs/cache-manager cache-manager cache-manager-redis-yet

@Module({
  imports: [
    CacheModule.registerAsync({
      isGlobal: true,
      imports: [ConfigModule],
      useFactory: (config: ConfigService) => ({
        store: redisStore,
        host: config.get('REDIS_HOST'),
        port: config.get('REDIS_PORT'),
        ttl: 300, // 5 minutes default
      }),
      inject: [ConfigService],
    }),
  ],
})
export class AppModule {}

// Auto-caching với Interceptor
@Controller('products')
export class ProductsController {
  @Get()
  @UseInterceptors(CacheInterceptor)
  @CacheTTL(60) // 60 seconds
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

    // Check cache
    const cached = await this.cacheManager.get<Product>(cacheKey);
    if (cached) return cached;

    // Fetch from DB
    const product = await this.productRepo.findOneBy({ id });

    // Save to cache
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

### 5. Health Checks

**Trả lời:**

```typescript
// npm install @nestjs/terminus

@Controller('health')
export class HealthController {
  constructor(
    private readonly health: HealthCheckService,
    private readonly http: HttpHealthIndicator,
    private readonly db: TypeOrmHealthIndicator,
    private readonly disk: DiskHealthIndicator,
    private readonly memory: MemoryHealthIndicator,
  ) {}

  @Get()
  @HealthCheck()
  check() {
    return this.health.check([
      () => this.db.pingCheck('database'),
      () => this.http.pingCheck('api', 'https://api.example.com'),
      () => this.disk.checkStorage('storage', { path: '/', thresholdPercent: 0.9 }),
      () => this.memory.checkHeap('memory_heap', 300 * 1024 * 1024), // 300MB
    ]);
  }
}
```

---

## XI. TIPS PHỎNG VẤN NESTJS

1. **Nắm vững Request Lifecycle**: M-G-I-P-H-I-F
2. **Hiểu DI sâu**: Custom providers, scopes, injection tokens
3. **Biết thiết kế module**: Khi nào dùng shared, global, dynamic module
4. **Security**: JWT, Guards, RBAC, rate limiting, helmet, CORS
5. **Performance**: Caching, queue, database indexing, pagination
6. **Testing**: Unit test (mock DI), E2E test (supertest)
7. **Thực hành**: Xây dựng một project CRUD hoàn chỉnh trước buổi phỏng vấn

---

## XII. CÂU HỎI NHANH (QUICK FIRE)

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
