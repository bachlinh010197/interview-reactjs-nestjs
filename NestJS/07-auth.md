# 📘 NestJS - Phần 7: Authentication & Authorization

[⬅️ Database](./06-database-orm.md) | [Phần tiếp: Advanced ➡️](./08-advanced.md)

---

## 1. JWT Authentication

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

// local.strategy.ts
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

// jwt.strategy.ts
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
```

---

## 2. Role-Based Access Control (RBAC)

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

    if (!requiredRoles) return true;

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
