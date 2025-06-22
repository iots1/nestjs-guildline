# 📘 NestJS Standard Project Guideline

> คู่มือการเริ่มต้นและวางโครงสร้างมาตรฐานสำหรับโปรเจกต์ NestJS ที่ทีมสามารถใช้ร่วมกันได้ รองรับ Multiple Database และใช้ TypeORM

---

## 🧭 Table of Contents
1. [Initial Project Setup](#-1-initial-project-setup)
2. 2. [Required Packages](#-2-required-packages)
3. [Security (Jwt authen)](#-3-security-jwt-authen)
4. [Main App Setup (`main.ts`)](#-4-main-app-setup-maints)
5. [Swagger Setup](#-5-swagger-setup)
6. [Global Error Handling](#-6-global-error-handling)
7. [Multiple Database Concept](#-7-multiple-database-concept)
8. [Database Module Example](#-8-database-module-example)
9. [Example: Product Module](#-9-example-product-module)
10. [Auth From DB_SELLER](#-10-auth-from-db_seller)
11. [Testing Suggestion](#-11-testing-suggestion)
12. [ENV Example](#-12-env-example)
13. [Summary Checklist](#-13-summary-checklist)


## 📦 1. Initial Project Setup

### 🔧 Basic Setup
```bash
$ npm i -g @nestjs/cli
$ nest new your-project-name
```

### 📁 Folder Structure (แนะนำ)
```bash
src/
├── app.module.ts
├── main.ts
├── common/              # constants, filters, interceptors
├── database/            db connection, enums
│   ├── database.module.ts
│   ├── datasource_tokens.enum.ts
│   ├── sellers.datasource.read.ts
│   ├── sellers.datasource.write.ts
│   ├── shops.datasource.read.ts
│   └── shops.datasource.write.ts
├── entities/            table ของแต่ละ database
│   ├── sellers/         
│   └── shops/  
├── modules/
│   ├── sellers/         # auth, seller profile
│   └── shops/           # shop, product, item
└── config/              # environment configs
```

---

## 📄 2. Required Packages
```bash
npm install class-validator class-transformer @nestjs/swagger swagger-ui-express cookie-parser @nestjs/jwt passport-jwt @nestjs/passport passport
```

---

## 🔐 3. Security (Jwt authen)
```bash

```

### Setup JwtModule
```bash
npm install @nestjs/jwt passport-jwt @nestjs/passport passport
```

```ts
JwtModule.register({
  secret: process.env.JWT_SECRET,
  signOptions: { expiresIn: '1d' },
});
```

### Auth Guard
```ts
@Injectable()
export class JwtAuthGuard extends AuthGuard('jwt') {}
```

### Strategy
```ts
@Injectable()
export class JwtStrategy extends PassportStrategy(Strategy) {
  constructor() {
    super({
      jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken(),
      ignoreExpiration: false,
      secretOrKey: process.env.JWT_SECRET,
    });
  }

  async validate(payload: any) {
    return { userId: payload.sub, username: payload.username };
  }
}
```

### Controller Example
```ts
@UseGuards(JwtAuthGuard)
@Get('profile')
getProfile(@Request() req) {
  return req.user;
}
```


---

## 📚 4. Swagger Setup
```ts
  // Basic Auth
  const config = new DocumentBuilder()
    .setTitle('My API')
    .setVersion('1')
    .addTag('Trends API')
    .addBasicAuth(
      { type: 'http', scheme: 'basic', description: 'Basic authentication for API endpoints' },
      'api-basic-auth' // This key must match the name used in @ApiBearerAuth('api-basic-auth')
    )
    .build();
  const documentFactory = () => SwaggerModule.createDocument(app, config);
  SwaggerModule.setup(API_PREFIX + '/docs', app, documentFactory);

  // Jwt Auth
  const config = new DocumentBuilder()
  .setTitle('My API')
    .setVersion('1')
    .addTag('Trends API')
    .addBearerAuth(
      {
        type: 'http',
        scheme: 'bearer',
        bearerFormat: 'JWT',
        name: 'Authorization',
        in: 'header',
      },
      'access-token', // This key must match the name used in @ApiBearerAuth('access-token')
    )
    .build();

```

---

## ⚠️ 5. Global Error Handling (main.ts)
```ts
// Helper function to recursively extract validation errors
function extractValidationErrors(errors: ValidationError[]): Record<string, any> {
  return errors.reduce((acc, error) => {
    // If we have constraints, add them
    if (error.constraints) {
      acc[error.property] = Object.values(error.constraints);
    }

    // If we have children, process them too
    if (error.children && error.children.length > 0) {
      // For nested properties, create a proper path
      const childErrors = extractValidationErrors(error.children);

      // Add the child errors with proper property paths
      Object.keys(childErrors).forEach(childProp => {
        const path = error.property ? `${error.property}.${childProp}` : childProp;
        acc[path] = childErrors[childProp];
      });
    }

    return acc;
  }, {} as Record<string, any>);
}

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  
  app.useGlobalPipes(
    new ValidationPipe({
      transform: true,
      whitelist: true,
      forbidNonWhitelisted: false,
      stopAtFirstError: false, // Changed to false to collect all errors
      transformOptions: {
        enableImplicitConversion: false
      },
      // Improved exception factory that handles nested validation errors
      exceptionFactory: (errors: ValidationError[]) => {
        const formattedErrors = extractValidationErrors(errors);
        return new BadRequestException({
          message: 'Validation failed',
          errors: formattedErrors
        });
      }
    }),
  );
}
```
รองรับ validation error ซ้อนหลายชั้นได้ (ใช้ `exceptionFactory` ใน `ValidationPipe`)

```ts
import {
  ArgumentsHost,
  Catch,
  ExceptionFilter,
  HttpException,
} from '@nestjs/common';
import { JwtService } from '@nestjs/jwt';
import { Request, Response } from 'express';

@Catch(HttpException)
export class HttpExceptionFilter implements ExceptionFilter {
  constructor(
    // private readonly _logs: LogsService,
    private readonly _jwt: JwtService,
  ) {}

  catch(exception: HttpException, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse<Response>();
    const request = ctx.getRequest<Request>();
    const status = exception.getStatus();

    const exceptionData = {
      success: false,
      timestamp: new Date().toISOString(),
      message: exception.getResponse()['message'] || exception.message || null,
      errors: exception.getResponse()['errors'] || null,
      code: exception.getResponse()['code'] || 500000,
      method: request.method,
      path: request.url,
    };

    console.log(exceptionData);

    if (
      exceptionData.message !== 'Cannot GET /favicon.ico' &&
      exceptionData.message !== 'Cannot GET /'
    ) {
      const authorization =
        request.headers['authorization'] || request.headers['Authorization'];
      let userId = null;
      try {
        if (authorization) {
          if (typeof authorization === 'string') {
            userId = this._jwt.decode(authorization.split(' ')[1]).user_id;
          }
        }
      } catch (ex) {}

      const logData = {
        log_user_id: userId,
        log_device_model:
          typeof request.headers['device-model'] === 'string'
            ? request.headers['device-model']
            : undefined,
        log_device_platform: request.headers['device-platform']?.toString(),
        log_device_id: request.headers['device-id']?.toString(),
        log_device_token: request.headers['device-token']?.toString(),
        log_device_brand: request.headers['device-brand']?.toString(),
        log_device_version: request.headers['device-version']?.toString(),
        api_key: request.headers['api-key']?.toString(),
        app_version: request.headers['app-version']?.toString(),
        log_type: 2,
        log_function: 'HttpExceptionFilter',
        error_message: exceptionData.message,
        user_agent: request.headers['user-agent']?.toString(),
        debug_data: {
          request: {
            method: request.method,
            url: request.url,
            body: request.body,
          },
          response: {
            status: status,
            data: exceptionData,
          },
        },
      };
      // this._logs.create(logData);
    }

    response.status(status).json(exceptionData);
  }
}
```

---

## 🧠 6. Multiple Database Concept
> กรณีมี 2 ฐาน: `DB_SELLER`, `DB_SHOP`

| DB Name | Content |
|---------|---------|
| DB_SELLER | ข้อมูลผู้ขาย: seller, seller_user |
| DB_SHOP   | ข้อมูลร้านค้า: shop, product, product_item |

---

## 🔌 7. Database Module Example
```ts
@Module({
  providers: [
    {
      provide: DatasourceTokens.SELLERS_DB_READ,
      useFactory: async () => {
        if (!SellersDataSourceRead.isInitialized) await SellersDataSourceRead.initialize();
        return SellersDataSourceRead;
      },
    },
    {
      provide: DatasourceTokens.SHOPS_DB_WRITE,
      useFactory: async () => {
        if (!ShopsDataSourceWrite.isInitialized) await ShopsDataSourceWrite.initialize();
        return ShopsDataSourceWrite;
      },
    },
  ],
  exports: [DatasourceTokens.SELLERS_DB_READ, DatasourceTokens.SHOPS_DB_WRITE],
})
```

---

## 🛒 8. Example: Product Module
### DTO
```ts
export class CreateProductDTO {
  @IsString() name: string;
  @IsString() seller_id: string;
}
```

### Entity
```ts
@Entity('products')
export class Product {
  @PrimaryGeneratedColumn('uuid') product_id: string;
  @Column() name: string;
  @Column() seller_id: string;
  @Column({
      name: 'is_delete',
      type: 'bit', width: 1,
      default: () => "b'0'",
      transformer: {
          to: (value: boolean) => value ? 1 : 0,
          from: (value: Buffer) => value?.[0] === 1
      }
  })
  is_delete: boolean;
}

@OneToMany(() => ProductItem, (item) => item.product, {
    cascade: true,
})
product_items: ProductItem[];
```

### Service (Use Multiple DB)
```ts
import { Module } from '@nestjs/common';
import { DatasourceTokens } from './datasource_tokens.enum';
import { SellersDataSourceRead } from './sellers.datasource.read';
import { SellersDataSourceWrite } from './sellers.datasource.write';
import { ShopsDataSourceRead } from './shops.datasource.read';
import { ShopsDataSourceWrite } from './shops.datasource.write';

@Module({
    providers: [
        {
            provide: DatasourceTokens.SELLERS_DB_WRITE,
            useFactory: async () => {
                if (!SellersDataSourceWrite.isInitialized) {
                    await SellesrDataSourceWrite.initialize();
                }
                return SellersDataSourceWrite;
            },
        },
        {
            provide: DatasourceTokens.SELLERS_DB_READ,
            useFactory: async () => {
                if (!SellerDataSourceRead.isInitialized) {
                    await SellersDataSourceRead.initialize();
                }
                return SellersDataSourceRead;
            },
        },
        {
            provide: DatasourceTokens.SHOPS_DB_WRITE,
            useFactory: async () => {
                if (!ShopDataSourceWrite.isInitialized) {
                    await ShopSDataSourceWrite.initialize();
                }
                return ShopSDataSourceWrite;
            },
        },
        {
            provide: DatasourceTokens.SHOPS_DB_READ,
            useFactory: async () => {
                if (!ShopSDataSourceRead.isInitialized) {
                    await ShopDataSourceRead.initialize();
                }
                return ShopSDataSourceRead;
            },
        },
    ],
    exports: [
        DatasourceTokens.SELLERS_DB_WRITE,
        DatasourceTokens.SELLERS_DB_READ,
        DatasourceTokens.SHOPS_DB_WRITE,
        DatasourceTokens.SHOPS_DB_READ,
    ],
})
export class DatabaseModule { }

export class DatasourceTokens {
    static readonly SELLERS_DB_WRITE = 'SELLERS_DB_WRITE';
    static readonly SELLERS_DB_READ = 'SELLERS_DB_READ';
    static readonly SHOPS_DB_WRITE = 'SHOPS_DB_WRITE';
    static readonly SHOPS_DB_READ = 'SHOPS_DB_READ';
}
```

```ts
import * as dotenv from 'dotenv';
import { DataSource } from 'typeorm';
dotenv.config();

export const SellersDataSourceRead = new DataSource({
    type: 'mysql',
    host: process.env.DB_SELLERS_READ_HOST,
    port: +process.env.DB_PORT,
    username: process.env.DB_USERNAME,
    password: process.env.DB_PASSWORD,
    database: process.env.DB_SELLERS_READ_NAME,
    entities: [__dirname + '/../entities/sellers/**/*.entity.{ts,js}'],
    synchronize: false,
});

export const SellersDataSourceWrite = new DataSource({
    type: 'mysql',
    host: process.env.DB_SELLERS_WRITE_HOST,
    port: +process.env.DB_PORT,
    username: process.env.DB_USERNAME,
    password: process.env.DB_PASSWORD,
    database: process.env.DB_SELLERS_WRITE_NAME,
    entities: [__dirname + '/../entities/sellers/**/*.entity.{ts,js}'],
    synchronize: false,
});
```


```ts
@Injectable()
export class ProductService {
  constructor(
    @Inject(DatasourceTokens.SHOPS_DB_WRITE) private shopWrite: DataSource,
    @Inject(DatasourceTokens.SELLERS_DB_READ) private sellerRead: DataSource,
  ) {}

  async createProduct(data: CreateProductDTO): Promise<Product> {
    const seller = await this.sellerRead.manager.findOne(Seller, {
      where: { seller_id: data.seller_id },
    });
    if (!seller) throw new BadRequestException('Invalid seller ID');
    const product = this.shopWrite.manager.create(Product, data);
    return await this.shopWrite.manager.save(Product, product);
  }
}
```

### Controller
```ts
@Controller('products')
export class ProductController {
  constructor(private readonly productService: ProductService) {}

  @Post()
  async create(@Body() body: CreateProductDTO): Promise<Product> {
    return this.productService.createProduct(body);
  }
}
```

---

## 🔐 9. Auth From DB_SELLER
```ts
@Injectable()
export class AuthService {
  constructor(@Inject(DatasourceTokens.SELLERS_DB_READ) private sellerRead: DataSource) {}

  async validateUser(username: string, password: string): Promise<SellerUser> {
    const user = await this.sellerRead.manager.findOne(SellerUser, {
      where: { username, password },
      relations: ['seller'],
    });
    if (!user) throw new UnauthorizedException('Invalid credentials');
    return user;
  }
}
```

---

## 🧪 10. Testing Suggestion
```bash
npm i --save-dev @nestjs/testing
```
แนะนำใช้ `supertest` และ Mock DB ด้วย `jest.fn()`

---

## 📂 11. ENV Example
```env
DB_SELLER_HOST=localhost
DB_SELLER_PORT=3306
DB_SELLER_USER=root
DB_SELLER_PASSWORD=1234
DB_SELLER_NAME=sellers_db

DB_SHOP_HOST=localhost
DB_SHOP_PORT=3306
DB_SHOP_USER=root
DB_SHOP_PASSWORD=1234
DB_SHOP_NAME=shops_db
```

---

## ✅ Summary Checklist
- [x] Jwt Auth
- [x] Swagger 
- [x] GlobalPipe + Validation Exception
- [x] GlobalFilter
- [x] Multiple DB via DI
- [x] Structured Modules (shop, seller)
- [x] DTO + Entity clear mapping
- [x] Controller/Service separation
- [x] ENV Management
