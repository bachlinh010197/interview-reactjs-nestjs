# 📘 NestJS - Phần 6: Database & ORM

[⬅️ Middleware/Guards/Pipes](./05-middleware-guards-pipes.md) | [Phần tiếp: Auth ➡️](./07-auth.md)

---

## 1. TypeORM Integration

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

  @Column({ select: false })
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

## 2. Prisma Integration

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

## 🔥 CÂU HỎI LEVEL MIDDLE → SENIOR

### 3. Database Migration Strategy

**Câu hỏi: Cách quản lý database migrations trong production?**

**Trả lời:**

```typescript
// TypeORM — CLI migrations
// package.json scripts:
// "migration:generate": "typeorm migration:generate -d src/data-source.ts src/migrations/Migration"
// "migration:run": "typeorm migration:run -d src/data-source.ts"
// "migration:revert": "typeorm migration:revert -d src/data-source.ts"

// Ví dụ migration file
export class AddUserAvatarColumn1700000000000 implements MigrationInterface {
  public async up(queryRunner: QueryRunner): Promise<void> {
    await queryRunner.addColumn('users', new TableColumn({
      name: 'avatar_url',
      type: 'varchar',
      isNullable: true,
    }));

    // Data migration (nếu cần)
    await queryRunner.query(`
      UPDATE users SET avatar_url = CONCAT('https://avatar.example.com/', id)
      WHERE avatar_url IS NULL
    `);
  }

  public async down(queryRunner: QueryRunner): Promise<void> {
    await queryRunner.dropColumn('users', 'avatar_url');
  }
}

// Prisma migrations
// npx prisma migrate dev --name add_avatar_url
// npx prisma migrate deploy (production)
```

**Best practices:**
- **KHÔNG BAO GIỜ** dùng `synchronize: true` trong production
- Mỗi migration phải có cả `up()` và `down()`
- Test migration trên staging trước production
- Tách data migration và schema migration
- Dùng CI/CD pipeline chạy migration tự động

---

### 4. Repository Pattern & Unit of Work

**Câu hỏi: Cách implement Repository pattern đúng cách trong NestJS?**

**Trả lời:**

```typescript
// Generic repository interface
export interface IRepository<T> {
  findById(id: number): Promise<T | null>;
  findAll(options?: FindOptions): Promise<T[]>;
  create(data: Partial<T>): Promise<T>;
  update(id: number, data: Partial<T>): Promise<T>;
  delete(id: number): Promise<void>;
}

// Custom repository với TypeORM
@Injectable()
export class UserRepository {
  constructor(
    @InjectRepository(User)
    private readonly repo: Repository<User>,
  ) {}

  async findByEmail(email: string): Promise<User | null> {
    return this.repo.findOne({
      where: { email },
      select: ['id', 'email', 'name', 'password', 'role'],
    });
  }

  async findActiveUsers(pagination: PaginationDto): Promise<[User[], number]> {
    return this.repo
      .createQueryBuilder('user')
      .where('user.deletedAt IS NULL')
      .andWhere('user.isActive = :active', { active: true })
      .orderBy('user.createdAt', 'DESC')
      .skip((pagination.page - 1) * pagination.limit)
      .take(pagination.limit)
      .getManyAndCount();
  }

  async bulkCreate(users: Partial<User>[]): Promise<User[]> {
    const entities = this.repo.create(users);
    return this.repo.save(entities, { chunk: 100 }); // Insert theo batch 100
  }
}

// Service chỉ chứa business logic, không biết về database
@Injectable()
export class UserService {
  constructor(private readonly userRepo: UserRepository) {}

  async registerUser(dto: RegisterDto): Promise<User> {
    const existing = await this.userRepo.findByEmail(dto.email);
    if (existing) throw new ConflictException('Email already exists');

    const hashedPassword = await bcrypt.hash(dto.password, 12);
    return this.userRepo.create({ ...dto, password: hashedPassword });
  }
}
```

---

### 5. Optimistic Locking & Concurrency Control

**Câu hỏi: Cách xử lý concurrent updates trong NestJS + TypeORM?**

**Trả lời:**

```typescript
// Optimistic Locking — dùng @VersionColumn
@Entity()
export class Product {
  @PrimaryGeneratedColumn()
  id: number;

  @Column()
  stock: number;

  @VersionColumn() // TypeORM tự quản lý version
  version: number;
}

@Injectable()
export class ProductService {
  async decreaseStock(productId: number, quantity: number) {
    const maxRetries = 3;

    for (let attempt = 0; attempt < maxRetries; attempt++) {
      try {
        const product = await this.productRepo.findOneBy({ id: productId });

        if (product.stock < quantity) {
          throw new BadRequestException('Insufficient stock');
        }

        product.stock -= quantity;
        await this.productRepo.save(product);
        // TypeORM sẽ thêm WHERE version = X vào UPDATE query
        // Nếu version đã thay đổi → throw OptimisticLockVersionMismatchError
        return product;
      } catch (error) {
        if (error.name === 'OptimisticLockVersionMismatchError' && attempt < maxRetries - 1) {
          continue; // Retry
        }
        throw error;
      }
    }
  }

  // Pessimistic Locking — lock row trong DB
  async decreaseStockPessimistic(productId: number, quantity: number) {
    return this.productRepo.manager.transaction(async (manager) => {
      const product = await manager.findOne(Product, {
        where: { id: productId },
        lock: { mode: 'pessimistic_write' }, // SELECT ... FOR UPDATE
      });

      if (product.stock < quantity) {
        throw new BadRequestException('Insufficient stock');
      }

      product.stock -= quantity;
      return manager.save(product);
    });
  }
}
```
