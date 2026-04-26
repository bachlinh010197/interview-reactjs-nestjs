# 📘 NestJS - Phần 5: Middleware, Guards, Interceptors, Pipes, Filters

[⬅️ Providers](./04-providers-services.md) | [Phần tiếp: Database ➡️](./06-database-orm.md)

> ⭐ **Đây là phần rất quan trọng, hay bị hỏi trong phỏng vấn!**

---

## 1. Middleware

**Câu hỏi: Middleware trong NestJS là gì?**

**Trả lời:**

Middleware chạy **trước route handler**, có access vào `request`, `response`, và `next()`. Giống Express middleware.

```typescript
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
      .exclude({ path: 'health', method: RequestMethod.GET })
      .forRoutes('*');
  }
}
```

---

## 2. Guards

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

## 3. Interceptors

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
```

---

## 4. Pipes

**Câu hỏi: Pipes trong NestJS dùng để làm gì?**

**Trả lời:**

Pipes có 2 mục đích: **Transformation** (chuyển đổi data) và **Validation** (kiểm tra data).

```typescript
// Built-in pipes
@Get(':id')
findOne(@Param('id', ParseIntPipe) id: number) {
  // id đã được convert từ string → number
}

@Get()
findAll(
  @Query('page', new DefaultValuePipe(1), ParseIntPipe) page: number,
  @Query('active', new DefaultValuePipe(true), ParseBoolPipe) active: boolean,
) {}

// Custom Pipe
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
```

---

## 5. Exception Filters

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

// Apply globally
app.useGlobalFilters(new AllExceptionsFilter());
```

---

## 6. Execution Order (⭐ Rất quan trọng!)

**Câu hỏi: Thứ tự thực thi của Middleware, Guards, Interceptors, Pipes, Filters?**

**Trả lời:**

```
Request → Middleware → Guards → Interceptors (before) → Pipes → Handler → Interceptors (after) → Response
                                                                    ↓ (nếu error)
                                                              Exception Filters
```

**Nhớ:** **M-G-I-P-H-I-F**

**Scope priority:** Global → Controller → Route handler
