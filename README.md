# 📘 NestJS Standard Project Guideline

> คู่มือการเริ่มต้นและวางโครงสร้างมาตรฐานสำหรับโปรเจกต์ NestJS ที่ทีมสามารถใช้ร่วมกันได้ รองรับ Multiple Database และใช้ TypeORM

---

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
npm install class-validator class-transformer @nestjs/swagger swagger-ui-express cookie-parser
```

---

## 🚀 3. Main App Setup (`main.ts`)
```ts
app.setGlobalPrefix('api');
app.useGlobalPipes(new ValidationPipe({
  transform: true,
  whitelist: true,
  forbidNonWhitelisted: false,
}));
app.use(cookieParser());
```

---

## 📚 4. Swagger Setup
```ts
const config = new DocumentBuilder()
  .setTitle('API')
  .setVersion('1.0')
  .addBasicAuth()
  .build();
const document = SwaggerModule.createDocument(app, config);
SwaggerModule.setup('api/docs', app, document);
```

---

## ⚠️ 5. Global Error Handling
```ts
app.useGlobalFilters(new HttpExceptionFilter());
```
รองรับ validation error ซ้อนหลายชั้นได้ (ใช้ `exceptionFactory` ใน `ValidationPipe`)

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
import { SellerDataSourceRead } from './seller.datasource.read';
import { SellerDataSourceWrite } from './seller.datasource.write';
import { ShopDataSourceRead } from './shop.datasource.read';
import { ShopDataSourceWrite } from './shop.datasource.write';

@Module({
    providers: [
        {
            provide: DatasourceTokens.SELLER_DB_WRITE,
            useFactory: async () => {
                if (!SellerDataSourceWrite.isInitialized) {
                    await SellerDataSourceWrite.initialize();
                }
                return SellerDataSourceWrite;
            },
        },
        {
            provide: DatasourceTokens.SELLER_DB_READ,
            useFactory: async () => {
                if (!SellerDataSourceRead.isInitialized) {
                    await SellerDataSourceRead.initialize();
                }
                return SellerDataSourceRead;
            },
        },
        {
            provide: DatasourceTokens.SHOP_DB_WRITE,
            useFactory: async () => {
                if (!ShopDataSourceWrite.isInitialized) {
                    await ShopDataSourceWrite.initialize();
                }
                return ShopDataSourceWrite;
            },
        },
        {
            provide: DatasourceTokens.SHOP_DB_READ,
            useFactory: async () => {
                if (!ShopDataSourceRead.isInitialized) {
                    await ShopDataSourceRead.initialize();
                }
                return ShopDataSourceRead;
            },
        },
    ],
    exports: [
        DatasourceTokens.SELLER_DB_WRITE,
        DatasourceTokens.SELLER_DB_READ,
        DatasourceTokens.SHOP_DB_WRITE,
        DatasourceTokens.SHOP_DB_READ,
    ],
})
export class DatabaseModule { }

export class DatasourceTokens {
    static readonly SERVER_DB_WRITE = 'SERVER_DB_WRITE';
    static readonly SERVER_DB_READ = 'SERVER_DB_READ';
    static readonly TRENDS_DB_WRITE = 'TRENDS_DB_WRITE';
    static readonly TRENDS_DB_READ = 'TRENDS_DB_READ';
}
```

```
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
    @Inject(DatasourceTokens.SHOP_DB_WRITE) private shopWrite: DataSource,
    @Inject(DatasourceTokens.SELLER_DB_READ) private sellerRead: DataSource,
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
  constructor(@Inject(DatasourceTokens.SELLER_DB_READ) private sellerRead: DataSource) {}

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
- [x] Swagger + BasicAuth
- [x] GlobalPipe + Validation Exception
- [x] GlobalFilter
- [x] Multiple DB via DI
- [x] Structured Modules (shop, seller)
- [x] DTO + Entity clear mapping
- [x] Controller/Service separation
- [x] ENV Management
