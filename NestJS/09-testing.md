# 📘 NestJS - Phần 9: Testing

[⬅️ Advanced](./08-advanced.md) | [Phần tiếp: Tình huống thực tế ➡️](./10-tinh-huong-thuc-te.md)

---

## 1. Unit Testing

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
```

---

## 2. Controller Testing

```typescript
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

## 3. E2E Testing

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
        .send({ name: '' })
        .expect(400);
    });
  });
});
```
