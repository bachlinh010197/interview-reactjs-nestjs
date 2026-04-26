# 📘 NestJS - Phần 3: Controllers & Routing

[⬅️ Modules](./02-modules.md) | [Phần tiếp: Providers ➡️](./04-providers-services.md)

---

## 1. DTOs và Validation

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
    whitelist: true,            // Loại bỏ properties không có trong DTO
    forbidNonWhitelisted: true, // Throw error nếu có property lạ
    transform: true,            // Tự động transform type
    transformOptions: {
      enableImplicitConversion: true,
    },
  }));
  await app.listen(3000);
}
```

---

## 2. File Upload

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
