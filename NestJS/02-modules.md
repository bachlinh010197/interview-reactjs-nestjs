# 📘 NestJS - Phần 2: Modules

[⬅️ Fundamentals](./01-fundamentals.md) | [Phần tiếp: Controllers ➡️](./03-controllers-routing.md)

---

## 1. Các loại Module

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

## 2. Circular Dependency

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
