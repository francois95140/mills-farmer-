# MilesPilot Backend + Sync Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the MilesPilot NestJS backend: a REST API that tells the Chrome
extension how many Flying Blue + Amex-card Miles a purchase at a given
merchant/country earns, backed by a Postgres DB kept fresh by a weekly sync
job pulling real data from the Flying Blue+ merchant catalog.

**Architecture:** Single NestJS repo (`backend/`), Docker Compose running
`postgres` + `app`. Feature modules (`merchants`, `cards`, `offers`, `users`,
`flying-blue-sync`) each own their TypeORM entities, service, and controller.
A pure `RewardCalculatorService` (no DB, no HTTP) computes Miles from an
offer + card rule + price, so it's trivially unit-testable. The sync job is a
plain HTTP client (no headless browser — see note below) that upserts
`merchants`/`merchant_domains`/`offers`/`offer_conditions` from the Flying
Blue+ public catalog API, on a weekly cron plus a manual trigger endpoint.

**Tech Stack:** NestJS 10, TypeScript (strict), TypeORM 0.3, PostgreSQL 16
(Docker), `@nestjs/schedule` (cron), `@nestjs/axios` (HTTP client), Jest +
supertest (tests), `nock` (HTTP mocking in tests).

**Spec:** [docs/superpowers/specs/2026-09-28-milespilot-backend-design.md](../specs/2026-09-28-milespilot-backend-design.md)

**Deviation from spec, discovered during planning:** the spec called for
Playwright to scrape `flyingblueplus-shopping.flyingblue.com`. Direct
inspection of the site found it calls a public, unauthenticated JSON API —
`GET /website/merchants/exploration?q=all&sort=popularity&sortDirection=asc&allowedPayFilters=2&pageSize=20&pageNumber={n}&lang=en`
— returning `{TotalPages, TotalItems, PageNumber, Data: [{MerchantId, Name,
Code, CountryCode, LogoUrl, ShopUrl, DefaultEarnAmount, DefaultEarnMiles,
DefaultEarnCurrency, Offers: [...]}]}`. This plan uses a plain HTTP client
against that endpoint instead of Playwright: simpler, no headless-browser
flakiness, same weekly-cron behavior the spec asked for. `ShopUrl` is used
directly as `tracking_url` — Flying Blue+ is a "select this payment method
at checkout" program, not an affiliate deep-link program, confirmed by
inspecting a live merchant page (the "Start Shopping" button links straight
to the merchant's own site).

## Global Constraints

- TypeScript strict mode; Node LTS.
- NestJS 10.x, TypeORM 0.3.x, PostgreSQL 16, all run via Docker Compose
  (`docker-compose.yml` at repo root: `postgres` + `app` services).
- REST API, JSON only. No GraphQL.
- No authentication in v1. The extension generates a local anonymous UUID;
  the backend creates a `users` row lazily on first `PATCH /users/:id`.
- `flying_blue_status` is stored on `users` but must **never** factor into
  any Miles calculation — no verified rule ties it to the Miles rate.
- Never return a fabricated Miles number. If no active, unexpired offer
  exists for a merchant+country, `/offers` returns `status: "UNVERIFIED"`
  and no Miles fields.
- Amex card rates (`card_reward_rules`) are seeded manually, never scraped.
- The sync job hits the public Flying Blue+ catalog API directly (see
  deviation note above) — no headless browser, no Playwright dependency.
- Cumulating Flying Blue Store Miles + card Miles is an **unverified
  assumption** — every `/offers` response that adds them together must
  include a condition/disclaimer string saying so.
- Offers are append-only: a new sync inserts a new `offers` row and flips
  `is_active = false` on the merchant's previous active offer for that
  program+country. No offer row is ever deleted or mutated after insert
  except this one `is_active` flip.

## Review Focus

- **Catalog pagination not fully walked.** The exploration API paginates
  (`TotalPages`, `pageSize=20`); a sync that only reads `pageNumber=1` will
  silently miss most of the catalog (65 merchants / 4 pages observed live).
  Covered in Task 9's sync test, which uses a 3-page fixture and asserts all
  3 pages were requested and all merchants upserted.
- **Unknown merchant/country requested from `/offers`.** A merchant slug or
  country with zero matching `offers` row (never synced, or wrong country)
  must return `UNVERIFIED`, never throw or 404. Covered in Task 7.
- **Expired offer still flagged `is_active`.** If a sync run fails or is
  delayed past `valid_until`, `/offers` must still treat a past-`valid_until`
  row as `UNVERIFIED` rather than trusting the `is_active` flag alone.
  Covered in Task 6 (calculator) and Task 7 (controller).
- **Re-sync duplicates merchants instead of updating them.** Running the
  sync twice for the same catalog must upsert the existing `merchants` row
  (matched by `slug`) and insert exactly one new `offers` row, not create a
  second `merchants` row. Covered in Task 9.
- **Concurrent sync runs (cron fires while a manual `/admin/scrape/run` is
  still in flight).** Must not double-insert active offers for the same
  merchant. Covered in Task 9 with an in-process running-flag guard.

---

## File Structure

```
backend/
  docker-compose.yml
  Dockerfile
  package.json
  tsconfig.json
  .env.example
  src/
    main.ts
    app.module.ts
    config/
      typeorm.config.ts
    common/
      enums.ts
    countries/
      country.entity.ts
    loyalty-programs/
      loyalty-program.entity.ts
    merchants/
      merchant.entity.ts
      merchant-domain.entity.ts
      merchants.module.ts
      merchants.service.ts
      merchants.controller.ts
    cards/
      card.entity.ts
      card-reward-rule.entity.ts
      cards.module.ts
      cards.service.ts
      cards.controller.ts
    offers/
      offer.entity.ts
      offer-tier.entity.ts
      offer-condition.entity.ts
      offers.module.ts
      offers.service.ts
      offers.controller.ts
      reward-calculator.service.ts
    users/
      user.entity.ts
      users.module.ts
      users.service.ts
      users.controller.ts
    flying-blue-sync/
      flying-blue-catalog-client.ts
      flying-blue-sync.service.ts
      flying-blue-sync.module.ts
      flying-blue-sync.controller.ts
      dto/catalog-response.dto.ts
  test/
    fixtures/
      catalog-page-1.json
      catalog-page-2.json
      catalog-page-3.json
    reward-calculator.service.spec.ts   (co-located unit test also allowed)
    offers.e2e-spec.ts
    flying-blue-sync.service.spec.ts
```

Migrations live under `src/migrations/` (TypeORM CLI), one per task that adds
entities, generated in that task's steps.

---

### Task 1: Project scaffold, Docker, DB connection, health check

**Files:**
- Create: `backend/package.json`
- Create: `backend/tsconfig.json`
- Create: `backend/nest-cli.json`
- Create: `backend/.env.example`
- Create: `backend/docker-compose.yml`
- Create: `backend/Dockerfile`
- Create: `backend/src/main.ts`
- Create: `backend/src/app.module.ts`
- Create: `backend/src/config/typeorm.config.ts`
- Create: `backend/src/health/health.controller.ts`
- Create: `backend/src/health/health.module.ts`
- Test: `backend/test/health.e2e-spec.ts`

**Interfaces:**
- Produces: `AppModule` (imported by every later module's e2e bootstrap),
  `typeOrmConfig(): TypeOrmModuleOptions` (used by `AppModule` and by
  Task 9/10's test Postgres config).

- [ ] **Step 1: Scaffold `package.json`**

```json
{
  "name": "milespilot-backend",
  "version": "0.1.0",
  "private": true,
  "scripts": {
    "build": "nest build",
    "start": "nest start",
    "start:dev": "nest start --watch",
    "test": "jest",
    "test:e2e": "jest --config ./test/jest-e2e.json",
    "typeorm": "typeorm-ts-node-commonjs -d src/config/typeorm.config.ts"
  },
  "dependencies": {
    "@nestjs/axios": "^3.0.2",
    "@nestjs/common": "^10.4.4",
    "@nestjs/config": "^3.2.3",
    "@nestjs/core": "^10.4.4",
    "@nestjs/platform-express": "^10.4.4",
    "@nestjs/schedule": "^4.1.1",
    "@nestjs/typeorm": "^10.0.2",
    "axios": "^1.7.7",
    "class-transformer": "^0.5.1",
    "class-validator": "^0.14.1",
    "pg": "^8.13.0",
    "reflect-metadata": "^0.2.2",
    "rxjs": "^7.8.1",
    "typeorm": "^0.3.20",
    "uuid": "^10.0.0"
  },
  "devDependencies": {
    "@nestjs/cli": "^10.4.5",
    "@nestjs/schematics": "^10.2.3",
    "@nestjs/testing": "^10.4.4",
    "@types/jest": "^29.5.13",
    "@types/node": "^22.7.4",
    "@types/supertest": "^6.0.2",
    "@types/uuid": "^10.0.0",
    "jest": "^29.7.0",
    "nock": "^13.5.5",
    "supertest": "^7.0.0",
    "ts-jest": "^29.2.5",
    "ts-node": "^10.9.2",
    "typescript": "^5.6.3"
  }
}
```

- [ ] **Step 2: `tsconfig.json`**

```json
{
  "compilerOptions": {
    "module": "commonjs",
    "declaration": false,
    "target": "ES2021",
    "sourceMap": true,
    "outDir": "./dist",
    "baseUrl": "./",
    "strict": true,
    "experimentalDecorators": true,
    "emitDecoratorMetadata": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true
  }
}
```

- [ ] **Step 3: `nest-cli.json`**

```json
{
  "$schema": "https://json.schemastore.org/nest-cli",
  "collection": "@nestjs/schematics",
  "sourceRoot": "src"
}
```

- [ ] **Step 4: `.env.example`**

```
DB_HOST=postgres
DB_PORT=5432
DB_USER=milespilot
DB_PASSWORD=milespilot
DB_NAME=milespilot
PORT=3000
```

- [ ] **Step 5: `docker-compose.yml`**

```yaml
version: "3.9"
services:
  postgres:
    image: postgres:16
    restart: unless-stopped
    environment:
      POSTGRES_USER: milespilot
      POSTGRES_PASSWORD: milespilot
      POSTGRES_DB: milespilot
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U milespilot"]
      interval: 5s
      timeout: 5s
      retries: 10
  app:
    build: .
    command: npm run start:dev
    environment:
      DB_HOST: postgres
      DB_PORT: "5432"
      DB_USER: milespilot
      DB_PASSWORD: milespilot
      DB_NAME: milespilot
      PORT: "3000"
    ports:
      - "3000:3000"
    depends_on:
      postgres:
        condition: service_healthy
    volumes:
      - ./src:/app/src
volumes:
  pgdata:
```

- [ ] **Step 6: `Dockerfile`**

```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY package.json package-lock.json* ./
RUN npm install
COPY . .
EXPOSE 3000
CMD ["npm", "run", "start:dev"]
```

- [ ] **Step 7: `src/config/typeorm.config.ts`**

```typescript
import { TypeOrmModuleOptions } from '@nestjs/typeorm';

export function typeOrmConfig(): TypeOrmModuleOptions {
  return {
    type: 'postgres',
    host: process.env.DB_HOST ?? 'localhost',
    port: parseInt(process.env.DB_PORT ?? '5432', 10),
    username: process.env.DB_USER ?? 'milespilot',
    password: process.env.DB_PASSWORD ?? 'milespilot',
    database: process.env.DB_NAME ?? 'milespilot',
    autoLoadEntities: true,
    synchronize: process.env.NODE_ENV === 'test',
    migrationsRun: process.env.NODE_ENV !== 'test',
    migrations: [__dirname + '/../migrations/*.{ts,js}'],
  };
}
```

`synchronize: true` only under `NODE_ENV=test` (e2e tests build schema from
entities directly against a throwaway test DB); real dev/prod always goes
through migrations.

- [ ] **Step 8: `src/health/health.controller.ts`**

```typescript
import { Controller, Get } from '@nestjs/common';

@Controller('health')
export class HealthController {
  @Get()
  check() {
    return { status: 'ok' };
  }
}
```

- [ ] **Step 9: `src/health/health.module.ts`**

```typescript
import { Module } from '@nestjs/common';
import { HealthController } from './health.controller';

@Module({ controllers: [HealthController] })
export class HealthModule {}
```

- [ ] **Step 10: `src/app.module.ts`**

```typescript
import { Module } from '@nestjs/common';
import { ConfigModule } from '@nestjs/config';
import { TypeOrmModule } from '@nestjs/typeorm';
import { ScheduleModule } from '@nestjs/schedule';
import { typeOrmConfig } from './config/typeorm.config';
import { HealthModule } from './health/health.module';

@Module({
  imports: [
    ConfigModule.forRoot({ isGlobal: true }),
    TypeOrmModule.forRoot(typeOrmConfig()),
    ScheduleModule.forRoot(),
    HealthModule,
  ],
})
export class AppModule {}
```

- [ ] **Step 11: `src/main.ts`**

```typescript
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  await app.listen(process.env.PORT ?? 3000);
}
bootstrap();
```

- [ ] **Step 12: Write the failing e2e test**

`backend/test/jest-e2e.json`:

```json
{
  "moduleFileExtensions": ["js", "json", "ts"],
  "rootDir": ".",
  "testEnvironment": "node",
  "testRegex": ".e2e-spec.ts$",
  "transform": { "^.+\\.(t|j)s$": "ts-jest" }
}
```

`backend/test/health.e2e-spec.ts`:

```typescript
import { Test } from '@nestjs/testing';
import { INestApplication } from '@nestjs/common';
import * as request from 'supertest';
import { AppModule } from '../src/app.module';

describe('Health (e2e)', () => {
  let app: INestApplication;

  beforeAll(async () => {
    process.env.NODE_ENV = 'test';
    process.env.DB_NAME = 'milespilot_test';
    const moduleRef = await Test.createTestingModule({
      imports: [AppModule],
    }).compile();
    app = moduleRef.createNestApplication();
    await app.init();
  });

  afterAll(async () => {
    await app.close();
  });

  it('GET /health returns ok', () => {
    return request(app.getHttpServer())
      .get('/health')
      .expect(200)
      .expect({ status: 'ok' });
  });
});
```

- [ ] **Step 13: Run it, confirm it fails without a DB up**

```bash
cd backend && npm install && docker compose up -d postgres && sleep 3 && npm run test:e2e
```

Expected: PASS once `postgres` is healthy (this test doesn't need any
entity, just the app booting against the DB) — if it fails, read the error:
either Postgres isn't reachable (check `docker compose ps`) or `AppModule`
has a wiring bug. Fix before continuing; do not move to Task 2 with this
red.

- [ ] **Step 14: Commit**

```bash
git add backend/
git commit -m "chore: scaffold NestJS backend with Docker Postgres and health check"
```

---

### Task 2: Countries and loyalty programs

**Files:**
- Create: `backend/src/countries/country.entity.ts`
- Create: `backend/src/loyalty-programs/loyalty-program.entity.ts`
- Create: `backend/src/migrations/<timestamp>-CreateCountriesAndPrograms.ts`
- Create: `backend/src/database/seed.ts`
- Test: `backend/test/countries-seed.e2e-spec.ts`

**Interfaces:**
- Consumes: `typeOrmConfig()` from Task 1.
- Produces: `Country` entity (`id: string (uuid)`, `code: string`,
  `name: string`), `LoyaltyProgram` entity (`id: string (uuid)`,
  `code: string`, `name: string`) — both imported by Task 3 (`merchants`),
  Task 4 (`cards`), Task 5 (`offers`), Task 8 (`users`).

- [ ] **Step 1: `Country` entity**

```typescript
// src/countries/country.entity.ts
import { Column, Entity, PrimaryGeneratedColumn } from 'typeorm';

@Entity('countries')
export class Country {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @Column({ unique: true })
  code: string; // ISO 3166-1 alpha-2, e.g. "FR"

  @Column()
  name: string;
}
```

- [ ] **Step 2: `LoyaltyProgram` entity**

```typescript
// src/loyalty-programs/loyalty-program.entity.ts
import { Column, Entity, PrimaryGeneratedColumn } from 'typeorm';

@Entity('loyalty_programs')
export class LoyaltyProgram {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @Column({ unique: true })
  code: string; // e.g. "FLYING_BLUE"

  @Column()
  name: string;
}
```

- [ ] **Step 3: Generate and write the migration**

```bash
cd backend && npx typeorm-ts-node-commonjs migration:create src/migrations/CreateCountriesAndPrograms
```

Edit the generated file to:

```typescript
import { MigrationInterface, QueryRunner, Table } from 'typeorm';

export class CreateCountriesAndPrograms1727520000000
  implements MigrationInterface
{
  public async up(queryRunner: QueryRunner): Promise<void> {
    await queryRunner.createTable(
      new Table({
        name: 'countries',
        columns: [
          { name: 'id', type: 'uuid', isPrimary: true, default: 'gen_random_uuid()' },
          { name: 'code', type: 'varchar', isUnique: true },
          { name: 'name', type: 'varchar' },
        ],
      }),
    );
    await queryRunner.createTable(
      new Table({
        name: 'loyalty_programs',
        columns: [
          { name: 'id', type: 'uuid', isPrimary: true, default: 'gen_random_uuid()' },
          { name: 'code', type: 'varchar', isUnique: true },
          { name: 'name', type: 'varchar' },
        ],
      }),
    );
  }

  public async down(queryRunner: QueryRunner): Promise<void> {
    await queryRunner.dropTable('loyalty_programs');
    await queryRunner.dropTable('countries');
  }
}
```

Rename the file to match the class's timestamp prefix
(`1727520000000-CreateCountriesAndPrograms.ts`) so TypeORM's migration
runner picks it up in order.

- [ ] **Step 4: Seed script**

```typescript
// src/database/seed.ts
import { DataSource } from 'typeorm';
import { typeOrmConfig } from '../config/typeorm.config';
import { Country } from '../countries/country.entity';
import { LoyaltyProgram } from '../loyalty-programs/loyalty-program.entity';

async function seed() {
  const dataSource = new DataSource(typeOrmConfig() as any);
  await dataSource.initialize();

  const countryRepo = dataSource.getRepository(Country);
  const countries = [
    { code: 'FR', name: 'France' },
    { code: 'NL', name: 'Netherlands' },
    { code: 'BE', name: 'Belgium' },
  ];
  for (const c of countries) {
    const existing = await countryRepo.findOne({ where: { code: c.code } });
    if (!existing) await countryRepo.save(countryRepo.create(c));
  }

  const programRepo = dataSource.getRepository(LoyaltyProgram);
  const existingProgram = await programRepo.findOne({
    where: { code: 'FLYING_BLUE' },
  });
  if (!existingProgram) {
    await programRepo.save(
      programRepo.create({ code: 'FLYING_BLUE', name: 'Flying Blue' }),
    );
  }

  await dataSource.destroy();
  console.log('Seed complete');
}

seed().catch((err) => {
  console.error(err);
  process.exit(1);
});
```

Add to `package.json` scripts: `"seed": "ts-node -r tsconfig-paths/register src/database/seed.ts"`.

- [ ] **Step 5: Write the failing e2e test**

```typescript
// test/countries-seed.e2e-spec.ts
import { DataSource } from 'typeorm';
import { typeOrmConfig } from '../src/config/typeorm.config';
import { Country } from '../src/countries/country.entity';
import { LoyaltyProgram } from '../src/loyalty-programs/loyalty-program.entity';

describe('Countries and loyalty programs seed', () => {
  let dataSource: DataSource;

  beforeAll(async () => {
    process.env.NODE_ENV = 'test';
    process.env.DB_NAME = 'milespilot_test';
    dataSource = new DataSource({
      ...(typeOrmConfig() as any),
      entities: [Country, LoyaltyProgram],
      synchronize: true,
    });
    await dataSource.initialize();
  });

  afterAll(async () => {
    await dataSource.destroy();
  });

  it('stores and reads back a country and a loyalty program', async () => {
    const countryRepo = dataSource.getRepository(Country);
    const saved = await countryRepo.save(
      countryRepo.create({ code: 'FR', name: 'France' }),
    );
    const found = await countryRepo.findOne({ where: { code: 'FR' } });
    expect(found?.id).toBe(saved.id);

    const programRepo = dataSource.getRepository(LoyaltyProgram);
    const savedProgram = await programRepo.save(
      programRepo.create({ code: 'FLYING_BLUE', name: 'Flying Blue' }),
    );
    expect(savedProgram.code).toBe('FLYING_BLUE');
  });
});
```

- [ ] **Step 6: Run it to verify it fails**

```bash
cd backend && npm run test:e2e -- countries-seed
```

Expected: FAIL — `relation "countries" does not exist` (no test DB created
yet).

- [ ] **Step 7: Create the test database and re-run**

```bash
docker compose exec postgres psql -U milespilot -c "CREATE DATABASE milespilot_test;"
npm run test:e2e -- countries-seed
```

Expected: PASS (entities in this test set `synchronize: true` directly, so
the table is created on the fly).

- [ ] **Step 8: Commit**

```bash
git add src/countries src/loyalty-programs src/migrations src/database test/countries-seed.e2e-spec.ts package.json
git commit -m "feat: add Country and LoyaltyProgram entities, migration, seed"
```

---

### Task 3: Merchants (entities + resolve endpoint)

**Files:**
- Create: `backend/src/merchants/merchant.entity.ts`
- Create: `backend/src/merchants/merchant-domain.entity.ts`
- Create: `backend/src/merchants/merchants.service.ts`
- Create: `backend/src/merchants/merchants.controller.ts`
- Create: `backend/src/merchants/merchants.module.ts`
- Modify: `backend/src/app.module.ts` (import `MerchantsModule`)
- Modify: `backend/src/migrations/` (new migration)
- Test: `backend/test/merchants.e2e-spec.ts`

**Interfaces:**
- Consumes: nothing beyond TypeORM/Nest.
- Produces: `Merchant` entity (`id`, `name`, `slug`, `logoUrl`, `createdAt`),
  `MerchantDomain` entity (`id`, `merchant`, `domain`, `matchType:
  'exact'|'suffix'`), `MerchantsService.findBySlug(slug: string):
  Promise<Merchant | null>`, `MerchantsService.upsertFromSync(input: {
  slug: string; name: string; logoUrl: string; domain: string }):
  Promise<Merchant>` (used by Task 9's sync), `GET
  /merchants/resolve?domain=` → `{ merchantSlug: string } | 404`.

- [ ] **Step 1: `Merchant` entity**

```typescript
// src/merchants/merchant.entity.ts
import {
  Column,
  CreateDateColumn,
  Entity,
  PrimaryGeneratedColumn,
} from 'typeorm';

@Entity('merchants')
export class Merchant {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @Column()
  name: string;

  @Column({ unique: true })
  slug: string;

  @Column({ nullable: true })
  logoUrl?: string;

  @CreateDateColumn()
  createdAt: Date;
}
```

- [ ] **Step 2: `MerchantDomain` entity**

```typescript
// src/merchants/merchant-domain.entity.ts
import {
  Column,
  Entity,
  ManyToOne,
  PrimaryGeneratedColumn,
} from 'typeorm';
import { Merchant } from './merchant.entity';

export type DomainMatchType = 'exact' | 'suffix';

@Entity('merchant_domains')
export class MerchantDomain {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @ManyToOne(() => Merchant, { onDelete: 'CASCADE' })
  merchant: Merchant;

  @Column({ unique: true })
  domain: string; // e.g. "belvilla.com"

  @Column({ type: 'varchar', default: 'suffix' })
  matchType: DomainMatchType;
}
```

- [ ] **Step 3: `MerchantsService`**

```typescript
// src/merchants/merchants.service.ts
import { Injectable } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import { Merchant } from './merchant.entity';
import { MerchantDomain } from './merchant-domain.entity';

export interface UpsertMerchantInput {
  slug: string;
  name: string;
  logoUrl?: string;
  domain: string;
}

@Injectable()
export class MerchantsService {
  constructor(
    @InjectRepository(Merchant)
    private readonly merchantRepo: Repository<Merchant>,
    @InjectRepository(MerchantDomain)
    private readonly domainRepo: Repository<MerchantDomain>,
  ) {}

  findBySlug(slug: string): Promise<Merchant | null> {
    return this.merchantRepo.findOne({ where: { slug } });
  }

  async resolveDomain(domain: string): Promise<Merchant | null> {
    const normalized = domain.replace(/^www\./, '').toLowerCase();
    const match = await this.domainRepo.findOne({
      where: { domain: normalized },
      relations: ['merchant'],
    });
    return match?.merchant ?? null;
  }

  async upsertFromSync(input: UpsertMerchantInput): Promise<Merchant> {
    let merchant = await this.merchantRepo.findOne({
      where: { slug: input.slug },
    });
    if (merchant) {
      merchant.name = input.name;
      merchant.logoUrl = input.logoUrl;
    } else {
      merchant = this.merchantRepo.create({
        slug: input.slug,
        name: input.name,
        logoUrl: input.logoUrl,
      });
    }
    merchant = await this.merchantRepo.save(merchant);

    const normalizedDomain = input.domain.replace(/^www\./, '').toLowerCase();
    let domainRow = await this.domainRepo.findOne({
      where: { domain: normalizedDomain },
    });
    if (domainRow) {
      domainRow.merchant = merchant;
    } else {
      domainRow = this.domainRepo.create({
        domain: normalizedDomain,
        merchant,
        matchType: 'suffix',
      });
    }
    await this.domainRepo.save(domainRow);

    return merchant;
  }
}
```

- [ ] **Step 4: `MerchantsController`**

```typescript
// src/merchants/merchants.controller.ts
import { Controller, Get, NotFoundException, Query } from '@nestjs/common';
import { MerchantsService } from './merchants.service';

@Controller('merchants')
export class MerchantsController {
  constructor(private readonly merchantsService: MerchantsService) {}

  @Get('resolve')
  async resolve(@Query('domain') domain: string) {
    if (!domain) throw new NotFoundException('domain query param required');
    const merchant = await this.merchantsService.resolveDomain(domain);
    if (!merchant) throw new NotFoundException('merchant not found');
    return { merchantSlug: merchant.slug };
  }
}
```

- [ ] **Step 5: `MerchantsModule`**

```typescript
// src/merchants/merchants.module.ts
import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';
import { Merchant } from './merchant.entity';
import { MerchantDomain } from './merchant-domain.entity';
import { MerchantsService } from './merchants.service';
import { MerchantsController } from './merchants.controller';

@Module({
  imports: [TypeOrmModule.forFeature([Merchant, MerchantDomain])],
  providers: [MerchantsService],
  controllers: [MerchantsController],
  exports: [MerchantsService],
})
export class MerchantsModule {}
```

- [ ] **Step 6: Wire into `AppModule`**

Add `import { MerchantsModule } from './merchants/merchants.module';` and
`MerchantsModule` to the `imports` array in `src/app.module.ts`.

- [ ] **Step 7: Migration**

```bash
npx typeorm-ts-node-commonjs migration:create src/migrations/CreateMerchants
```

```typescript
import { MigrationInterface, QueryRunner, Table, TableForeignKey } from 'typeorm';

export class CreateMerchants1727520100000 implements MigrationInterface {
  public async up(queryRunner: QueryRunner): Promise<void> {
    await queryRunner.createTable(
      new Table({
        name: 'merchants',
        columns: [
          { name: 'id', type: 'uuid', isPrimary: true, default: 'gen_random_uuid()' },
          { name: 'name', type: 'varchar' },
          { name: 'slug', type: 'varchar', isUnique: true },
          { name: 'logo_url', type: 'varchar', isNullable: true },
          { name: 'created_at', type: 'timestamptz', default: 'now()' },
        ],
      }),
    );
    await queryRunner.createTable(
      new Table({
        name: 'merchant_domains',
        columns: [
          { name: 'id', type: 'uuid', isPrimary: true, default: 'gen_random_uuid()' },
          { name: 'merchant_id', type: 'uuid' },
          { name: 'domain', type: 'varchar', isUnique: true },
          { name: 'match_type', type: 'varchar', default: "'suffix'" },
        ],
      }),
    );
    await queryRunner.createForeignKey(
      'merchant_domains',
      new TableForeignKey({
        columnNames: ['merchant_id'],
        referencedTableName: 'merchants',
        referencedColumnNames: ['id'],
        onDelete: 'CASCADE',
      }),
    );
  }

  public async down(queryRunner: QueryRunner): Promise<void> {
    await queryRunner.dropTable('merchant_domains');
    await queryRunner.dropTable('merchants');
  }
}
```

Rename to `1727520100000-CreateMerchants.ts`.

- [ ] **Step 8: Write the failing e2e test**

```typescript
// test/merchants.e2e-spec.ts
import { Test } from '@nestjs/testing';
import { INestApplication } from '@nestjs/common';
import * as request from 'supertest';
import { AppModule } from '../src/app.module';
import { MerchantsService } from '../src/merchants/merchants.service';

describe('Merchants (e2e)', () => {
  let app: INestApplication;
  let merchantsService: MerchantsService;

  beforeAll(async () => {
    process.env.NODE_ENV = 'test';
    process.env.DB_NAME = 'milespilot_test';
    const moduleRef = await Test.createTestingModule({
      imports: [AppModule],
    }).compile();
    app = moduleRef.createNestApplication();
    merchantsService = moduleRef.get(MerchantsService);
    await app.init();
  });

  afterAll(async () => {
    await app.close();
  });

  it('resolves a known domain to its merchant slug', async () => {
    await merchantsService.upsertFromSync({
      slug: 'belvilla',
      name: 'Belvilla',
      logoUrl: 'https://example.com/logo.png',
      domain: 'www.belvilla.com',
    });

    await request(app.getHttpServer())
      .get('/merchants/resolve?domain=belvilla.com')
      .expect(200)
      .expect({ merchantSlug: 'belvilla' });
  });

  it('returns 404 for an unknown domain', () => {
    return request(app.getHttpServer())
      .get('/merchants/resolve?domain=totally-unknown-shop.example')
      .expect(404);
  });
});
```

- [ ] **Step 9: Run it to verify it fails**

```bash
npm run test:e2e -- merchants
```

Expected: FAIL — table `merchants` doesn't exist under `NODE_ENV=test`
unless `synchronize` picks up the new entity via `autoLoadEntities`. Since
`typeOrmConfig()` sets `synchronize: true` in test mode already, this should
actually pass once the module compiles — if it fails instead with a
compile/DI error, that's the real red to fix first.

- [ ] **Step 10: Fix until it passes**

```bash
npm run test:e2e -- merchants
```

Expected: PASS.

- [ ] **Step 11: Commit**

```bash
git add src/merchants src/migrations src/app.module.ts test/merchants.e2e-spec.ts
git commit -m "feat: add Merchant/MerchantDomain entities and resolve endpoint"
```

---

### Task 4: Cards and card reward rules

**Files:**
- Create: `backend/src/cards/card.entity.ts`
- Create: `backend/src/cards/card-reward-rule.entity.ts`
- Create: `backend/src/cards/cards.service.ts`
- Create: `backend/src/cards/cards.controller.ts`
- Create: `backend/src/cards/cards.module.ts`
- Modify: `backend/src/app.module.ts`
- Modify: `backend/src/database/seed.ts` (seed the 4 Amex cards)
- Create: `backend/src/migrations/<timestamp>-CreateCards.ts`
- Test: `backend/test/cards.e2e-spec.ts`

**Interfaces:**
- Consumes: `LoyaltyProgram` entity (Task 2).
- Produces: `Card` entity (`id`, `name`, `issuer`, `program`),
  `CardRewardRule` entity (`id`, `card`, `milesPer10Eur`, `category`,
  `validFrom`, `validUntil`), `CardsService.findActiveRuleForCard(cardId:
  string): Promise<CardRewardRule | null>` (used by Task 6/7),
  `GET /cards` → `Card[]`.

- [ ] **Step 1: `Card` entity**

```typescript
// src/cards/card.entity.ts
import { Column, Entity, ManyToOne, PrimaryGeneratedColumn } from 'typeorm';
import { LoyaltyProgram } from '../loyalty-programs/loyalty-program.entity';

@Entity('cards')
export class Card {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @Column()
  name: string; // e.g. "Amex KLM Gold"

  @Column()
  issuer: string; // e.g. "American Express"

  @ManyToOne(() => LoyaltyProgram)
  program: LoyaltyProgram;
}
```

- [ ] **Step 2: `CardRewardRule` entity**

```typescript
// src/cards/card-reward-rule.entity.ts
import { Column, Entity, ManyToOne, PrimaryGeneratedColumn } from 'typeorm';
import { Card } from './card.entity';

@Entity('card_reward_rules')
export class CardRewardRule {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @ManyToOne(() => Card, { onDelete: 'CASCADE' })
  card: Card;

  @Column({ type: 'numeric', precision: 10, scale: 2 })
  milesPer10Eur: number;

  @Column({ nullable: true })
  category?: string;

  @Column({ type: 'timestamptz' })
  validFrom: Date;

  @Column({ type: 'timestamptz', nullable: true })
  validUntil?: Date;
}
```

- [ ] **Step 3: `CardsService`**

```typescript
// src/cards/cards.service.ts
import { Injectable } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { LessThanOrEqual, MoreThan, Repository } from 'typeorm';
import { IsNull, Or } from 'typeorm';
import { Card } from './card.entity';
import { CardRewardRule } from './card-reward-rule.entity';

@Injectable()
export class CardsService {
  constructor(
    @InjectRepository(Card) private readonly cardRepo: Repository<Card>,
    @InjectRepository(CardRewardRule)
    private readonly ruleRepo: Repository<CardRewardRule>,
  ) {}

  findAll(): Promise<Card[]> {
    return this.cardRepo.find();
  }

  async findActiveRuleForCard(
    cardId: string,
    now: Date = new Date(),
  ): Promise<CardRewardRule | null> {
    const rules = await this.ruleRepo.find({
      where: { card: { id: cardId }, validFrom: LessThanOrEqual(now) },
      order: { validFrom: 'DESC' },
    });
    const active = rules.find(
      (r) => !r.validUntil || r.validUntil > now,
    );
    return active ?? null;
  }
}
```

- [ ] **Step 4: `CardsController`**

```typescript
// src/cards/cards.controller.ts
import { Controller, Get } from '@nestjs/common';
import { CardsService } from './cards.service';

@Controller('cards')
export class CardsController {
  constructor(private readonly cardsService: CardsService) {}

  @Get()
  findAll() {
    return this.cardsService.findAll();
  }
}
```

- [ ] **Step 5: `CardsModule`**

```typescript
// src/cards/cards.module.ts
import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';
import { Card } from './card.entity';
import { CardRewardRule } from './card-reward-rule.entity';
import { CardsService } from './cards.service';
import { CardsController } from './cards.controller';

@Module({
  imports: [TypeOrmModule.forFeature([Card, CardRewardRule])],
  providers: [CardsService],
  controllers: [CardsController],
  exports: [CardsService],
})
export class CardsModule {}
```

- [ ] **Step 6: Wire into `AppModule`**

Add `CardsModule` import and to `imports` array.

- [ ] **Step 7: Migration**

```bash
npx typeorm-ts-node-commonjs migration:create src/migrations/CreateCards
```

```typescript
import { MigrationInterface, QueryRunner, Table, TableForeignKey } from 'typeorm';

export class CreateCards1727520200000 implements MigrationInterface {
  public async up(queryRunner: QueryRunner): Promise<void> {
    await queryRunner.createTable(
      new Table({
        name: 'cards',
        columns: [
          { name: 'id', type: 'uuid', isPrimary: true, default: 'gen_random_uuid()' },
          { name: 'name', type: 'varchar' },
          { name: 'issuer', type: 'varchar' },
          { name: 'program_id', type: 'uuid' },
        ],
      }),
    );
    await queryRunner.createForeignKey(
      'cards',
      new TableForeignKey({
        columnNames: ['program_id'],
        referencedTableName: 'loyalty_programs',
        referencedColumnNames: ['id'],
      }),
    );
    await queryRunner.createTable(
      new Table({
        name: 'card_reward_rules',
        columns: [
          { name: 'id', type: 'uuid', isPrimary: true, default: 'gen_random_uuid()' },
          { name: 'card_id', type: 'uuid' },
          { name: 'miles_per_10_eur', type: 'numeric', precision: 10, scale: 2 },
          { name: 'category', type: 'varchar', isNullable: true },
          { name: 'valid_from', type: 'timestamptz' },
          { name: 'valid_until', type: 'timestamptz', isNullable: true },
        ],
      }),
    );
    await queryRunner.createForeignKey(
      'card_reward_rules',
      new TableForeignKey({
        columnNames: ['card_id'],
        referencedTableName: 'cards',
        referencedColumnNames: ['id'],
        onDelete: 'CASCADE',
      }),
    );
  }

  public async down(queryRunner: QueryRunner): Promise<void> {
    await queryRunner.dropTable('card_reward_rules');
    await queryRunner.dropTable('cards');
  }
}
```

Rename to `1727520200000-CreateCards.ts`.

- [ ] **Step 8: Seed the 4 Amex cards**

Add to `src/database/seed.ts`, after the loyalty program seed block:

```typescript
  const cardRepo = dataSource.getRepository(Card);
  const ruleRepo = dataSource.getRepository(CardRewardRule);
  const flyingBlueProgram = await programRepo.findOneOrFail({
    where: { code: 'FLYING_BLUE' },
  });
  const cardSeeds = [
    { name: 'Amex KLM Silver', milesPer10Eur: 5 },
    { name: 'Amex KLM Gold', milesPer10Eur: 10 },
    { name: 'Amex KLM Platinum', milesPer10Eur: 15 },
    { name: 'Amex Platinum classique', milesPer10Eur: 10 },
  ];
  for (const c of cardSeeds) {
    let card = await cardRepo.findOne({ where: { name: c.name } });
    if (!card) {
      card = await cardRepo.save(
        cardRepo.create({
          name: c.name,
          issuer: 'American Express',
          program: flyingBlueProgram,
        }),
      );
    }
    const existingRule = await ruleRepo.findOne({
      where: { card: { id: card.id } },
    });
    if (!existingRule) {
      await ruleRepo.save(
        ruleRepo.create({
          card,
          milesPer10Eur: c.milesPer10Eur,
          validFrom: new Date('2026-01-01'),
        }),
      );
    }
  }
```

Add the matching imports (`Card`, `CardRewardRule`) at the top of
`seed.ts`.

**Note on seeded rates:** these `milesPer10Eur` values are placeholders
pending the real Amex KLM/Platinum published rate card — the spec requires
offers be "stored in base and modifiable", not that this plan know the
exact current published rate. Update via `PATCH` on `card_reward_rules` (or
re-run a corrected seed) once you have the real rate card in hand; nothing
in the code hardcodes these numbers outside this seed file.

- [ ] **Step 9: Write the failing e2e test**

```typescript
// test/cards.e2e-spec.ts
import { Test } from '@nestjs/testing';
import { INestApplication } from '@nestjs/common';
import * as request from 'supertest';
import { AppModule } from '../src/app.module';
import { CardsService } from '../src/cards/cards.service';
import { LoyaltyProgram } from '../src/loyalty-programs/loyalty-program.entity';
import { getRepositoryToken } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import { Card } from '../src/cards/card.entity';
import { CardRewardRule } from '../src/cards/card-reward-rule.entity';

describe('Cards (e2e)', () => {
  let app: INestApplication;
  let cardsService: CardsService;
  let programRepo: Repository<LoyaltyProgram>;
  let cardRepo: Repository<Card>;
  let ruleRepo: Repository<CardRewardRule>;

  beforeAll(async () => {
    process.env.NODE_ENV = 'test';
    process.env.DB_NAME = 'milespilot_test';
    const moduleRef = await Test.createTestingModule({
      imports: [AppModule],
    }).compile();
    app = moduleRef.createNestApplication();
    cardsService = moduleRef.get(CardsService);
    programRepo = moduleRef.get(getRepositoryToken(LoyaltyProgram));
    cardRepo = moduleRef.get(getRepositoryToken(Card));
    ruleRepo = moduleRef.get(getRepositoryToken(CardRewardRule));
    await app.init();
  });

  afterAll(async () => {
    await app.close();
  });

  it('GET /cards lists seeded cards and findActiveRuleForCard returns the current rate', async () => {
    const program = await programRepo.save(
      programRepo.create({ code: 'FLYING_BLUE', name: 'Flying Blue' }),
    );
    const card = await cardRepo.save(
      cardRepo.create({ name: 'Amex KLM Gold', issuer: 'American Express', program }),
    );
    await ruleRepo.save(
      ruleRepo.create({
        card,
        milesPer10Eur: 10,
        validFrom: new Date('2026-01-01'),
      }),
    );

    const res = await request(app.getHttpServer()).get('/cards').expect(200);
    expect(res.body.some((c: any) => c.name === 'Amex KLM Gold')).toBe(true);

    const rule = await cardsService.findActiveRuleForCard(card.id);
    expect(rule?.milesPer10Eur).toBe('10.00');
  });
});
```

- [ ] **Step 10: Run it to verify it fails**

```bash
npm run test:e2e -- cards
```

Expected: FAIL (module not wired / table missing).

- [ ] **Step 11: Fix until it passes**

```bash
npm run test:e2e -- cards
```

Expected: PASS.

- [ ] **Step 12: Commit**

```bash
git add src/cards src/migrations src/database/seed.ts src/app.module.ts test/cards.e2e-spec.ts
git commit -m "feat: add Card/CardRewardRule entities, seed 4 Amex cards, GET /cards"
```

---

### Task 5: Offers, offer tiers, offer conditions (entities + repository layer)

**Files:**
- Create: `backend/src/offers/offer.entity.ts`
- Create: `backend/src/offers/offer-tier.entity.ts`
- Create: `backend/src/offers/offer-condition.entity.ts`
- Create: `backend/src/offers/offers.repository.service.ts`
- Create: `backend/src/offers/offers.module.ts`
- Modify: `backend/src/app.module.ts`
- Create: `backend/src/migrations/<timestamp>-CreateOffers.ts`
- Test: `backend/test/offers-repository.e2e-spec.ts`

**Interfaces:**
- Consumes: `Merchant` (Task 3), `LoyaltyProgram` (Task 2), `Country`
  (Task 2).
- Produces: `Offer` entity (`id`, `merchant`, `loyaltyProgram`, `country`,
  `baseMilesPer10Eur`, `trackingUrl`, `source`, `scrapedAt`, `validFrom`,
  `validUntil`, `isActive`, `rawSnapshot`), `OfferTier` entity (`id`,
  `offer`, `minSpendEur`, `maxSpendEur`, `bonusMiles`, `bonusMultiplier`,
  `description`), `OfferCondition` entity (`id`, `offer`, `label`, `type:
  'exclusion'|'condition'`), `OffersRepositoryService.findActiveOffer(params:
  { merchantSlug: string; countryCode: string; programCode: string }):
  Promise<Offer | null>` (used by Task 7), `OffersRepositoryService
  .replaceActiveOffer(input: {...}): Promise<Offer>` (used by Task 9's
  sync — inserts a new offer row and flips the previous one's `isActive` to
  false, in one transaction).

- [ ] **Step 1: `Offer` entity**

```typescript
// src/offers/offer.entity.ts
import {
  Column,
  Entity,
  ManyToOne,
  PrimaryGeneratedColumn,
} from 'typeorm';
import { Merchant } from '../merchants/merchant.entity';
import { LoyaltyProgram } from '../loyalty-programs/loyalty-program.entity';
import { Country } from '../countries/country.entity';

@Entity('offers')
export class Offer {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @ManyToOne(() => Merchant, { onDelete: 'CASCADE' })
  merchant: Merchant;

  @ManyToOne(() => LoyaltyProgram)
  loyaltyProgram: LoyaltyProgram;

  @ManyToOne(() => Country)
  country: Country;

  @Column({ type: 'numeric', precision: 10, scale: 2 })
  baseMilesPer10Eur: number;

  @Column()
  trackingUrl: string;

  @Column()
  source: string;

  @Column({ type: 'timestamptz' })
  scrapedAt: Date;

  @Column({ type: 'timestamptz' })
  validFrom: Date;

  @Column({ type: 'timestamptz', nullable: true })
  validUntil?: Date;

  @Column({ default: true })
  isActive: boolean;

  @Column({ type: 'jsonb', nullable: true })
  rawSnapshot?: Record<string, unknown>;
}
```

- [ ] **Step 2: `OfferTier` entity**

```typescript
// src/offers/offer-tier.entity.ts
import { Column, Entity, ManyToOne, PrimaryGeneratedColumn } from 'typeorm';
import { Offer } from './offer.entity';

@Entity('offer_tiers')
export class OfferTier {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @ManyToOne(() => Offer, { onDelete: 'CASCADE' })
  offer: Offer;

  @Column({ type: 'numeric', precision: 10, scale: 2, nullable: true })
  minSpendEur?: number;

  @Column({ type: 'numeric', precision: 10, scale: 2, nullable: true })
  maxSpendEur?: number;

  @Column({ type: 'numeric', precision: 10, scale: 2, nullable: true })
  bonusMiles?: number;

  @Column({ type: 'numeric', precision: 10, scale: 4, nullable: true })
  bonusMultiplier?: number;

  @Column({ nullable: true })
  description?: string;
}
```

- [ ] **Step 3: `OfferCondition` entity**

```typescript
// src/offers/offer-condition.entity.ts
import { Column, Entity, ManyToOne, PrimaryGeneratedColumn } from 'typeorm';
import { Offer } from './offer.entity';

export type OfferConditionType = 'exclusion' | 'condition';

@Entity('offer_conditions')
export class OfferCondition {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @ManyToOne(() => Offer, { onDelete: 'CASCADE' })
  offer: Offer;

  @Column()
  label: string;

  @Column({ type: 'varchar', default: 'condition' })
  type: OfferConditionType;
}
```

- [ ] **Step 4: `OffersRepositoryService`**

```typescript
// src/offers/offers.repository.service.ts
import { Injectable } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { DataSource, Repository } from 'typeorm';
import { Offer } from './offer.entity';
import { OfferTier } from './offer-tier.entity';
import { OfferCondition } from './offer-condition.entity';
import { Merchant } from '../merchants/merchant.entity';
import { LoyaltyProgram } from '../loyalty-programs/loyalty-program.entity';
import { Country } from '../countries/country.entity';

export interface FindActiveOfferParams {
  merchantSlug: string;
  countryCode: string;
  programCode: string;
}

export interface ReplaceActiveOfferInput {
  merchant: Merchant;
  loyaltyProgram: LoyaltyProgram;
  country: Country;
  baseMilesPer10Eur: number;
  trackingUrl: string;
  source: string;
  scrapedAt: Date;
  validFrom: Date;
  validUntil?: Date;
  rawSnapshot?: Record<string, unknown>;
  conditions: string[];
}

@Injectable()
export class OffersRepositoryService {
  constructor(
    @InjectRepository(Offer) private readonly offerRepo: Repository<Offer>,
    @InjectRepository(OfferTier)
    private readonly tierRepo: Repository<OfferTier>,
    @InjectRepository(OfferCondition)
    private readonly conditionRepo: Repository<OfferCondition>,
    private readonly dataSource: DataSource,
  ) {}

  async findActiveOffer(params: FindActiveOfferParams): Promise<Offer | null> {
    return this.offerRepo.findOne({
      where: {
        isActive: true,
        merchant: { slug: params.merchantSlug },
        country: { code: params.countryCode },
        loyaltyProgram: { code: params.programCode },
      },
      relations: ['merchant', 'loyaltyProgram', 'country'],
    });
  }

  async findTiers(offerId: string): Promise<OfferTier[]> {
    return this.tierRepo.find({ where: { offer: { id: offerId } } });
  }

  async findConditions(offerId: string): Promise<OfferCondition[]> {
    return this.conditionRepo.find({ where: { offer: { id: offerId } } });
  }

  async replaceActiveOffer(input: ReplaceActiveOfferInput): Promise<Offer> {
    return this.dataSource.transaction(async (manager) => {
      const previous = await manager.findOne(Offer, {
        where: {
          isActive: true,
          merchant: { id: input.merchant.id },
          country: { id: input.country.id },
          loyaltyProgram: { id: input.loyaltyProgram.id },
        },
      });
      if (previous) {
        previous.isActive = false;
        await manager.save(previous);
      }

      const created = manager.create(Offer, {
        merchant: input.merchant,
        loyaltyProgram: input.loyaltyProgram,
        country: input.country,
        baseMilesPer10Eur: input.baseMilesPer10Eur,
        trackingUrl: input.trackingUrl,
        source: input.source,
        scrapedAt: input.scrapedAt,
        validFrom: input.validFrom,
        validUntil: input.validUntil,
        isActive: true,
        rawSnapshot: input.rawSnapshot,
      });
      const saved = await manager.save(created);

      for (const label of input.conditions) {
        await manager.save(
          manager.create(OfferCondition, {
            offer: saved,
            label,
            type: 'condition',
          }),
        );
      }

      return saved;
    });
  }
}
```

- [ ] **Step 5: `OffersModule`**

```typescript
// src/offers/offers.module.ts
import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';
import { Offer } from './offer.entity';
import { OfferTier } from './offer-tier.entity';
import { OfferCondition } from './offer-condition.entity';
import { OffersRepositoryService } from './offers.repository.service';

@Module({
  imports: [TypeOrmModule.forFeature([Offer, OfferTier, OfferCondition])],
  providers: [OffersRepositoryService],
  exports: [OffersRepositoryService],
})
export class OffersModule {}
```

- [ ] **Step 6: Wire into `AppModule`**

Add `OffersModule` import and to `imports`.

- [ ] **Step 7: Migration**

```bash
npx typeorm-ts-node-commonjs migration:create src/migrations/CreateOffers
```

```typescript
import { MigrationInterface, QueryRunner, Table, TableForeignKey } from 'typeorm';

export class CreateOffers1727520300000 implements MigrationInterface {
  public async up(queryRunner: QueryRunner): Promise<void> {
    await queryRunner.createTable(
      new Table({
        name: 'offers',
        columns: [
          { name: 'id', type: 'uuid', isPrimary: true, default: 'gen_random_uuid()' },
          { name: 'merchant_id', type: 'uuid' },
          { name: 'loyalty_program_id', type: 'uuid' },
          { name: 'country_id', type: 'uuid' },
          { name: 'base_miles_per_10_eur', type: 'numeric', precision: 10, scale: 2 },
          { name: 'tracking_url', type: 'varchar' },
          { name: 'source', type: 'varchar' },
          { name: 'scraped_at', type: 'timestamptz' },
          { name: 'valid_from', type: 'timestamptz' },
          { name: 'valid_until', type: 'timestamptz', isNullable: true },
          { name: 'is_active', type: 'boolean', default: true },
          { name: 'raw_snapshot', type: 'jsonb', isNullable: true },
        ],
      }),
    );
    await queryRunner.createForeignKeys('offers', [
      new TableForeignKey({ columnNames: ['merchant_id'], referencedTableName: 'merchants', referencedColumnNames: ['id'], onDelete: 'CASCADE' }),
      new TableForeignKey({ columnNames: ['loyalty_program_id'], referencedTableName: 'loyalty_programs', referencedColumnNames: ['id'] }),
      new TableForeignKey({ columnNames: ['country_id'], referencedTableName: 'countries', referencedColumnNames: ['id'] }),
    ]);

    await queryRunner.createTable(
      new Table({
        name: 'offer_tiers',
        columns: [
          { name: 'id', type: 'uuid', isPrimary: true, default: 'gen_random_uuid()' },
          { name: 'offer_id', type: 'uuid' },
          { name: 'min_spend_eur', type: 'numeric', precision: 10, scale: 2, isNullable: true },
          { name: 'max_spend_eur', type: 'numeric', precision: 10, scale: 2, isNullable: true },
          { name: 'bonus_miles', type: 'numeric', precision: 10, scale: 2, isNullable: true },
          { name: 'bonus_multiplier', type: 'numeric', precision: 10, scale: 4, isNullable: true },
          { name: 'description', type: 'varchar', isNullable: true },
        ],
      }),
    );
    await queryRunner.createForeignKey(
      'offer_tiers',
      new TableForeignKey({ columnNames: ['offer_id'], referencedTableName: 'offers', referencedColumnNames: ['id'], onDelete: 'CASCADE' }),
    );

    await queryRunner.createTable(
      new Table({
        name: 'offer_conditions',
        columns: [
          { name: 'id', type: 'uuid', isPrimary: true, default: 'gen_random_uuid()' },
          { name: 'offer_id', type: 'uuid' },
          { name: 'label', type: 'varchar' },
          { name: 'type', type: 'varchar', default: "'condition'" },
        ],
      }),
    );
    await queryRunner.createForeignKey(
      'offer_conditions',
      new TableForeignKey({ columnNames: ['offer_id'], referencedTableName: 'offers', referencedColumnNames: ['id'], onDelete: 'CASCADE' }),
    );
  }

  public async down(queryRunner: QueryRunner): Promise<void> {
    await queryRunner.dropTable('offer_conditions');
    await queryRunner.dropTable('offer_tiers');
    await queryRunner.dropTable('offers');
  }
}
```

Rename to `1727520300000-CreateOffers.ts`.

- [ ] **Step 8: Write the failing e2e test**

```typescript
// test/offers-repository.e2e-spec.ts
import { Test } from '@nestjs/testing';
import { INestApplication } from '@nestjs/common';
import { AppModule } from '../src/app.module';
import { OffersRepositoryService } from '../src/offers/offers.repository.service';
import { getRepositoryToken } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import { Merchant } from '../src/merchants/merchant.entity';
import { LoyaltyProgram } from '../src/loyalty-programs/loyalty-program.entity';
import { Country } from '../src/countries/country.entity';

describe('OffersRepositoryService (e2e)', () => {
  let app: INestApplication;
  let offersRepo: OffersRepositoryService;
  let merchantRepo: Repository<Merchant>;
  let programRepo: Repository<LoyaltyProgram>;
  let countryRepo: Repository<Country>;

  beforeAll(async () => {
    process.env.NODE_ENV = 'test';
    process.env.DB_NAME = 'milespilot_test';
    const moduleRef = await Test.createTestingModule({
      imports: [AppModule],
    }).compile();
    app = moduleRef.createNestApplication();
    offersRepo = moduleRef.get(OffersRepositoryService);
    merchantRepo = moduleRef.get(getRepositoryToken(Merchant));
    programRepo = moduleRef.get(getRepositoryToken(LoyaltyProgram));
    countryRepo = moduleRef.get(getRepositoryToken(Country));
    await app.init();
  });

  afterAll(async () => {
    await app.close();
  });

  it('replaceActiveOffer deactivates the previous offer and finds the new one as active', async () => {
    const merchant = await merchantRepo.save(
      merchantRepo.create({ name: 'Belvilla', slug: 'belvilla-repo-test' }),
    );
    const program = await programRepo.save(
      programRepo.create({ code: 'FLYING_BLUE_REPO_TEST', name: 'Flying Blue' }),
    );
    const country = await countryRepo.save(
      countryRepo.create({ code: 'ZZ_REPO_TEST', name: 'Testland' }),
    );

    const first = await offersRepo.replaceActiveOffer({
      merchant,
      loyaltyProgram: program,
      country,
      baseMilesPer10Eur: 30,
      trackingUrl: 'https://belvilla.com',
      source: 'flyingblue_sync',
      scrapedAt: new Date('2026-09-01'),
      validFrom: new Date('2026-09-01'),
      conditions: ['Excludes Holiday Parks'],
    });
    expect(first.isActive).toBe(true);

    const second = await offersRepo.replaceActiveOffer({
      merchant,
      loyaltyProgram: program,
      country,
      baseMilesPer10Eur: 36,
      trackingUrl: 'https://belvilla.com',
      source: 'flyingblue_sync',
      scrapedAt: new Date('2026-09-08'),
      validFrom: new Date('2026-09-08'),
      conditions: [],
    });

    const active = await offersRepo.findActiveOffer({
      merchantSlug: 'belvilla-repo-test',
      countryCode: 'ZZ_REPO_TEST',
      programCode: 'FLYING_BLUE_REPO_TEST',
    });
    expect(active?.id).toBe(second.id);
    expect(Number(active?.baseMilesPer10Eur)).toBe(36);

    const conditions = await offersRepo.findConditions(first.id);
    expect(conditions[0].label).toBe('Excludes Holiday Parks');
  });
});
```

- [ ] **Step 9: Run it to verify it fails**

```bash
npm run test:e2e -- offers-repository
```

Expected: FAIL (tables don't exist / module not wired).

- [ ] **Step 10: Fix until it passes**

```bash
npm run test:e2e -- offers-repository
```

Expected: PASS.

- [ ] **Step 11: Commit**

```bash
git add src/offers src/migrations src/app.module.ts test/offers-repository.e2e-spec.ts
git commit -m "feat: add Offer/OfferTier/OfferCondition entities and repository service"
```

---

### Task 6: Reward calculator (pure, TDD)

**Files:**
- Create: `backend/src/offers/reward-calculator.service.ts`
- Test: `backend/src/offers/reward-calculator.service.spec.ts`

**Interfaces:**
- Consumes: nothing (pure function, takes plain data, not entities, so it
  has zero DB/HTTP dependency and is fast to test).
- Produces:
  `RewardCalculatorService.calculate(input: CalculateRewardInput): CalculateRewardResult`
  used by Task 7's `OffersController`.

```typescript
export interface CalculateRewardInput {
  priceEur: number;
  now: Date;
  offer: {
    baseMilesPer10Eur: number;
    validFrom: Date;
    validUntil?: Date;
    isActive: boolean;
    tiers: Array<{
      minSpendEur?: number;
      maxSpendEur?: number;
      bonusMiles?: number;
      bonusMultiplier?: number;
    }>;
  } | null;
  cardRule: { milesPer10Eur: number } | null;
}

export type CalculateRewardResult =
  | {
      status: 'VERIFIED';
      flyingBlueMiles: number;
      cardMiles: number;
      totalMiles: number;
    }
  | { status: 'UNVERIFIED' };
```

- [ ] **Step 1: Write the failing unit tests**

```typescript
// src/offers/reward-calculator.service.spec.ts
import { RewardCalculatorService } from './reward-calculator.service';

describe('RewardCalculatorService', () => {
  const service = new RewardCalculatorService();
  const now = new Date('2026-09-28T00:00:00Z');

  it('computes flat-rate Flying Blue miles + card miles for a simple purchase', () => {
    const result = service.calculate({
      priceEur: 1629,
      now,
      offer: {
        baseMilesPer10Eur: 15,
        validFrom: new Date('2026-01-01'),
        isActive: true,
        tiers: [],
      },
      cardRule: { milesPer10Eur: 10 },
    });
    expect(result.status).toBe('VERIFIED');
    if (result.status === 'VERIFIED') {
      expect(result.flyingBlueMiles).toBeCloseTo(2443.5);
      expect(result.cardMiles).toBeCloseTo(1629);
      expect(result.totalMiles).toBeCloseTo(4072.5);
    }
  });

  it('adds a matching tier bonus on top of the base rate', () => {
    const result = service.calculate({
      priceEur: 600,
      now,
      offer: {
        baseMilesPer10Eur: 20,
        validFrom: new Date('2026-01-01'),
        isActive: true,
        tiers: [{ minSpendEur: 500, bonusMiles: 2500 }],
      },
      cardRule: null,
    });
    expect(result.status).toBe('VERIFIED');
    if (result.status === 'VERIFIED') {
      // base: 600 * 20 / 10 = 1200, + 2500 bonus tier = 3700
      expect(result.flyingBlueMiles).toBeCloseTo(3700);
      expect(result.cardMiles).toBe(0);
    }
  });

  it('does not apply a tier whose minSpendEur is not reached', () => {
    const result = service.calculate({
      priceEur: 100,
      now,
      offer: {
        baseMilesPer10Eur: 20,
        validFrom: new Date('2026-01-01'),
        isActive: true,
        tiers: [{ minSpendEur: 500, bonusMiles: 2500 }],
      },
      cardRule: null,
    });
    expect(result.status).toBe('VERIFIED');
    if (result.status === 'VERIFIED') {
      expect(result.flyingBlueMiles).toBeCloseTo(200);
    }
  });

  it('returns UNVERIFIED when there is no offer', () => {
    const result = service.calculate({
      priceEur: 100,
      now,
      offer: null,
      cardRule: null,
    });
    expect(result.status).toBe('UNVERIFIED');
  });

  it('returns UNVERIFIED when the offer is past its validUntil date, even if isActive is true', () => {
    const result = service.calculate({
      priceEur: 100,
      now,
      offer: {
        baseMilesPer10Eur: 20,
        validFrom: new Date('2026-01-01'),
        validUntil: new Date('2026-09-01'),
        isActive: true,
        tiers: [],
      },
      cardRule: null,
    });
    expect(result.status).toBe('UNVERIFIED');
  });

  it('returns UNVERIFIED when the offer is flagged inactive', () => {
    const result = service.calculate({
      priceEur: 100,
      now,
      offer: {
        baseMilesPer10Eur: 20,
        validFrom: new Date('2026-01-01'),
        isActive: false,
        tiers: [],
      },
      cardRule: null,
    });
    expect(result.status).toBe('UNVERIFIED');
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
npm test -- reward-calculator
```

Expected: FAIL — `RewardCalculatorService` doesn't exist yet.

- [ ] **Step 3: Implement**

```typescript
// src/offers/reward-calculator.service.ts
import { Injectable } from '@nestjs/common';

export interface CalculateRewardInput {
  priceEur: number;
  now: Date;
  offer: {
    baseMilesPer10Eur: number;
    validFrom: Date;
    validUntil?: Date;
    isActive: boolean;
    tiers: Array<{
      minSpendEur?: number;
      maxSpendEur?: number;
      bonusMiles?: number;
      bonusMultiplier?: number;
    }>;
  } | null;
  cardRule: { milesPer10Eur: number } | null;
}

export type CalculateRewardResult =
  | {
      status: 'VERIFIED';
      flyingBlueMiles: number;
      cardMiles: number;
      totalMiles: number;
    }
  | { status: 'UNVERIFIED' };

@Injectable()
export class RewardCalculatorService {
  calculate(input: CalculateRewardInput): CalculateRewardResult {
    const { priceEur, now, offer, cardRule } = input;

    if (!offer || !offer.isActive) {
      return { status: 'UNVERIFIED' };
    }
    if (offer.validUntil && offer.validUntil <= now) {
      return { status: 'UNVERIFIED' };
    }
    if (offer.validFrom > now) {
      return { status: 'UNVERIFIED' };
    }

    let flyingBlueMiles = (priceEur * offer.baseMilesPer10Eur) / 10;
    for (const tier of offer.tiers) {
      const min = tier.minSpendEur ?? -Infinity;
      const max = tier.maxSpendEur ?? Infinity;
      if (priceEur >= min && priceEur <= max) {
        if (tier.bonusMiles) flyingBlueMiles += tier.bonusMiles;
        if (tier.bonusMultiplier) flyingBlueMiles *= tier.bonusMultiplier;
      }
    }

    const cardMiles = cardRule ? (priceEur * cardRule.milesPer10Eur) / 10 : 0;

    return {
      status: 'VERIFIED',
      flyingBlueMiles,
      cardMiles,
      totalMiles: flyingBlueMiles + cardMiles,
    };
  }
}
```

- [ ] **Step 4: Run tests to verify they pass**

```bash
npm test -- reward-calculator
```

Expected: PASS, all 6 cases.

- [ ] **Step 5: Commit**

```bash
git add src/offers/reward-calculator.service.ts src/offers/reward-calculator.service.spec.ts
git commit -m "feat: add pure RewardCalculatorService with tier and expiry handling"
```

---

### Task 7: `GET /offers` endpoint

**Files:**
- Modify: `backend/src/offers/offers.module.ts` (add controller, provider)
- Create: `backend/src/offers/offers.controller.ts`
- Modify: `backend/src/offers/offers.repository.service.ts` (no change
  needed — already exposes what's required)
- Test: `backend/test/offers.e2e-spec.ts`

**Interfaces:**
- Consumes: `OffersRepositoryService.findActiveOffer` /
  `.findTiers` / `.findConditions` (Task 5), `RewardCalculatorService
  .calculate` (Task 6), `CardsService.findActiveRuleForCard` (Task 4).
- Produces: `GET /offers?merchant=:slug&country=:code&cardId=:id?` JSON
  response, consumed by the Chrome extension (future sub-project).

- [ ] **Step 1: `OffersController`**

```typescript
// src/offers/offers.controller.ts
import { Controller, Get, Query } from '@nestjs/common';
import { OffersRepositoryService } from './offers.repository.service';
import { RewardCalculatorService } from './reward-calculator.service';
import { CardsService } from '../cards/cards.service';

@Controller('offers')
export class OffersController {
  constructor(
    private readonly offersRepo: OffersRepositoryService,
    private readonly calculator: RewardCalculatorService,
    private readonly cardsService: CardsService,
  ) {}

  @Get()
  async getOffer(
    @Query('merchant') merchant: string,
    @Query('country') country: string,
    @Query('cardId') cardId?: string,
    @Query('priceEur') priceEurRaw?: string,
  ) {
    const offer = await this.offersRepo.findActiveOffer({
      merchantSlug: merchant,
      countryCode: country,
      programCode: 'FLYING_BLUE',
    });

    if (!offer) {
      return { status: 'UNVERIFIED' as const };
    }

    const [tiers, conditions] = await Promise.all([
      this.offersRepo.findTiers(offer.id),
      this.offersRepo.findConditions(offer.id),
    ]);

    const cardRule = cardId
      ? await this.cardsService.findActiveRuleForCard(cardId)
      : null;

    const priceEur = priceEurRaw ? parseFloat(priceEurRaw) : 0;

    const calc = this.calculator.calculate({
      priceEur,
      now: new Date(),
      offer: {
        baseMilesPer10Eur: Number(offer.baseMilesPer10Eur),
        validFrom: offer.validFrom,
        validUntil: offer.validUntil,
        isActive: offer.isActive,
        tiers: tiers.map((t) => ({
          minSpendEur: t.minSpendEur != null ? Number(t.minSpendEur) : undefined,
          maxSpendEur: t.maxSpendEur != null ? Number(t.maxSpendEur) : undefined,
          bonusMiles: t.bonusMiles != null ? Number(t.bonusMiles) : undefined,
          bonusMultiplier:
            t.bonusMultiplier != null ? Number(t.bonusMultiplier) : undefined,
        })),
      },
      cardRule: cardRule
        ? { milesPer10Eur: Number(cardRule.milesPer10Eur) }
        : null,
    });

    if (calc.status === 'UNVERIFIED') {
      return { status: 'UNVERIFIED' as const };
    }

    return {
      status: 'VERIFIED' as const,
      merchant: offer.merchant.name,
      program: offer.loyaltyProgram.code,
      country: offer.country.code,
      trackingUrl: offer.trackingUrl,
      flyingBlueMiles: calc.flyingBlueMiles,
      cardMiles: calc.cardMiles,
      totalMiles: calc.totalMiles,
      conditions: [
        ...conditions.map((c) => c.label),
        'Cumuler les Miles Flying Blue Store et les Miles carte sur un même achat n’est pas garanti par un justificatif vérifié — hypothèse non confirmée.',
      ],
      source: offer.source,
      scrapedAt: offer.scrapedAt,
      validUntil: offer.validUntil ?? null,
    };
  }
}
```

- [ ] **Step 2: Wire `OffersController` and `CardsModule` into `OffersModule`**

```typescript
// src/offers/offers.module.ts
import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';
import { Offer } from './offer.entity';
import { OfferTier } from './offer-tier.entity';
import { OfferCondition } from './offer-condition.entity';
import { OffersRepositoryService } from './offers.repository.service';
import { RewardCalculatorService } from './reward-calculator.service';
import { OffersController } from './offers.controller';
import { CardsModule } from '../cards/cards.module';

@Module({
  imports: [
    TypeOrmModule.forFeature([Offer, OfferTier, OfferCondition]),
    CardsModule,
  ],
  providers: [OffersRepositoryService, RewardCalculatorService],
  controllers: [OffersController],
  exports: [OffersRepositoryService],
})
export class OffersModule {}
```

- [ ] **Step 3: Write the failing e2e test**

```typescript
// test/offers.e2e-spec.ts
import { Test } from '@nestjs/testing';
import { INestApplication } from '@nestjs/common';
import * as request from 'supertest';
import { getRepositoryToken } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import { AppModule } from '../src/app.module';
import { OffersRepositoryService } from '../src/offers/offers.repository.service';
import { Merchant } from '../src/merchants/merchant.entity';
import { LoyaltyProgram } from '../src/loyalty-programs/loyalty-program.entity';
import { Country } from '../src/countries/country.entity';
import { Card } from '../src/cards/card.entity';
import { CardRewardRule } from '../src/cards/card-reward-rule.entity';

describe('GET /offers (e2e)', () => {
  let app: INestApplication;
  let offersRepo: OffersRepositoryService;
  let merchantRepo: Repository<Merchant>;
  let programRepo: Repository<LoyaltyProgram>;
  let countryRepo: Repository<Country>;
  let cardRepo: Repository<Card>;
  let ruleRepo: Repository<CardRewardRule>;

  beforeAll(async () => {
    process.env.NODE_ENV = 'test';
    process.env.DB_NAME = 'milespilot_test';
    const moduleRef = await Test.createTestingModule({
      imports: [AppModule],
    }).compile();
    app = moduleRef.createNestApplication();
    offersRepo = moduleRef.get(OffersRepositoryService);
    merchantRepo = moduleRef.get(getRepositoryToken(Merchant));
    programRepo = moduleRef.get(getRepositoryToken(LoyaltyProgram));
    countryRepo = moduleRef.get(getRepositoryToken(Country));
    cardRepo = moduleRef.get(getRepositoryToken(Card));
    ruleRepo = moduleRef.get(getRepositoryToken(CardRewardRule));
    await app.init();
  });

  afterAll(async () => {
    await app.close();
  });

  it('returns a VERIFIED reward breakdown for a merchant with an active offer and a card', async () => {
    const merchant = await merchantRepo.save(
      merchantRepo.create({ name: 'Apple', slug: 'apple-offers-test' }),
    );
    const program = await programRepo.save(
      programRepo.create({ code: 'FLYING_BLUE', name: 'Flying Blue' }),
    );
    const country = await countryRepo.save(
      countryRepo.create({ code: 'FR_OFFERS_TEST', name: 'France' }),
    );
    const card = await cardRepo.save(
      cardRepo.create({ name: 'Amex KLM Gold Offers Test', issuer: 'American Express', program }),
    );
    const rule = await ruleRepo.save(
      ruleRepo.create({ card, milesPer10Eur: 10, validFrom: new Date('2026-01-01') }),
    );
    await offersRepo.replaceActiveOffer({
      merchant,
      loyaltyProgram: program,
      country,
      baseMilesPer10Eur: 15,
      trackingUrl: 'https://apple.com',
      source: 'flyingblue_sync',
      scrapedAt: new Date('2026-09-20'),
      validFrom: new Date('2026-09-01'),
      conditions: [],
    });

    const res = await request(app.getHttpServer())
      .get(
        `/offers?merchant=apple-offers-test&country=FR_OFFERS_TEST&cardId=${card.id}&priceEur=1629`,
      )
      .expect(200);

    expect(res.body.status).toBe('VERIFIED');
    expect(res.body.flyingBlueMiles).toBeCloseTo(2443.5);
    expect(res.body.cardMiles).toBeCloseTo(1629);
    expect(res.body.totalMiles).toBeCloseTo(4072.5);
  });

  it('returns UNVERIFIED when no offer exists for the merchant/country', async () => {
    const res = await request(app.getHttpServer())
      .get('/offers?merchant=never-synced-merchant&country=FR_OFFERS_TEST&priceEur=100')
      .expect(200);
    expect(res.body.status).toBe('UNVERIFIED');
  });
});
```

- [ ] **Step 4: Run it to verify it fails**

```bash
npm run test:e2e -- 'offers\.'
```

Expected: FAIL — `OffersController` doesn't exist yet.

- [ ] **Step 5: Fix until it passes**

```bash
npm run test:e2e -- 'offers\.'
```

Expected: PASS.

- [ ] **Step 6: Commit**

```bash
git add src/offers/offers.controller.ts src/offers/offers.module.ts test/offers.e2e-spec.ts
git commit -m "feat: add GET /offers reward endpoint"
```

---

### Task 8: Users (anonymous profile)

**Files:**
- Create: `backend/src/users/user.entity.ts`
- Create: `backend/src/users/users.service.ts`
- Create: `backend/src/users/users.controller.ts`
- Create: `backend/src/users/users.module.ts`
- Modify: `backend/src/app.module.ts`
- Create: `backend/src/migrations/<timestamp>-CreateUsers.ts`
- Test: `backend/test/users.e2e-spec.ts`

**Interfaces:**
- Consumes: `Country` (Task 2), `Card` (Task 4).
- Produces: `User` entity (`id: string (client-supplied uuid)`, `email?`,
  `country`, `selectedCard`, `flyingBlueStatus`), `PATCH /users/:id`
  (upsert), `GET /users/:id`.

- [ ] **Step 1: `User` entity**

```typescript
// src/users/user.entity.ts
import { Column, Entity, ManyToOne, PrimaryColumn } from 'typeorm';
import { Country } from '../countries/country.entity';
import { Card } from '../cards/card.entity';

export type FlyingBlueStatus = 'EXPLORER' | 'SILVER' | 'GOLD' | 'PLATINUM';

@Entity('users')
export class User {
  @PrimaryColumn('uuid')
  id: string; // client-generated anonymous id

  @Column({ nullable: true })
  email?: string;

  @ManyToOne(() => Country, { nullable: true })
  country?: Country;

  @ManyToOne(() => Card, { nullable: true })
  selectedCard?: Card;

  @Column({ type: 'varchar', nullable: true })
  flyingBlueStatus?: FlyingBlueStatus;
}
```

- [ ] **Step 2: `UsersService`**

```typescript
// src/users/users.service.ts
import { Injectable } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import { User, FlyingBlueStatus } from './user.entity';

export interface UpsertUserInput {
  id: string;
  countryId?: string;
  selectedCardId?: string;
  flyingBlueStatus?: FlyingBlueStatus;
}

@Injectable()
export class UsersService {
  constructor(
    @InjectRepository(User) private readonly userRepo: Repository<User>,
  ) {}

  findById(id: string): Promise<User | null> {
    return this.userRepo.findOne({
      where: { id },
      relations: ['country', 'selectedCard'],
    });
  }

  async upsert(input: UpsertUserInput): Promise<User> {
    let user = await this.userRepo.findOne({ where: { id: input.id } });
    if (!user) {
      user = this.userRepo.create({ id: input.id });
    }
    if (input.countryId) user.country = { id: input.countryId } as any;
    if (input.selectedCardId)
      user.selectedCard = { id: input.selectedCardId } as any;
    if (input.flyingBlueStatus)
      user.flyingBlueStatus = input.flyingBlueStatus;
    return this.userRepo.save(user);
  }
}
```

- [ ] **Step 3: `UsersController`**

```typescript
// src/users/users.controller.ts
import {
  Body,
  Controller,
  Get,
  NotFoundException,
  Param,
  Patch,
} from '@nestjs/common';
import { UsersService } from './users.service';
import { FlyingBlueStatus } from './user.entity';

interface PatchUserBody {
  countryId?: string;
  selectedCardId?: string;
  flyingBlueStatus?: FlyingBlueStatus;
}

@Controller('users')
export class UsersController {
  constructor(private readonly usersService: UsersService) {}

  @Get(':id')
  async get(@Param('id') id: string) {
    const user = await this.usersService.findById(id);
    if (!user) throw new NotFoundException('user not found');
    return user;
  }

  @Patch(':id')
  async patch(@Param('id') id: string, @Body() body: PatchUserBody) {
    return this.usersService.upsert({ id, ...body });
  }
}
```

- [ ] **Step 4: `UsersModule`**

```typescript
// src/users/users.module.ts
import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';
import { User } from './user.entity';
import { UsersService } from './users.service';
import { UsersController } from './users.controller';

@Module({
  imports: [TypeOrmModule.forFeature([User])],
  providers: [UsersService],
  controllers: [UsersController],
  exports: [UsersService],
})
export class UsersModule {}
```

- [ ] **Step 5: Wire into `AppModule`**

Add `UsersModule` import and to `imports`.

- [ ] **Step 6: Migration**

```bash
npx typeorm-ts-node-commonjs migration:create src/migrations/CreateUsers
```

```typescript
import { MigrationInterface, QueryRunner, Table, TableForeignKey } from 'typeorm';

export class CreateUsers1727520400000 implements MigrationInterface {
  public async up(queryRunner: QueryRunner): Promise<void> {
    await queryRunner.createTable(
      new Table({
        name: 'users',
        columns: [
          { name: 'id', type: 'uuid', isPrimary: true },
          { name: 'email', type: 'varchar', isNullable: true },
          { name: 'country_id', type: 'uuid', isNullable: true },
          { name: 'selected_card_id', type: 'uuid', isNullable: true },
          { name: 'flying_blue_status', type: 'varchar', isNullable: true },
        ],
      }),
    );
    await queryRunner.createForeignKeys('users', [
      new TableForeignKey({ columnNames: ['country_id'], referencedTableName: 'countries', referencedColumnNames: ['id'] }),
      new TableForeignKey({ columnNames: ['selected_card_id'], referencedTableName: 'cards', referencedColumnNames: ['id'] }),
    ]);
  }

  public async down(queryRunner: QueryRunner): Promise<void> {
    await queryRunner.dropTable('users');
  }
}
```

Rename to `1727520400000-CreateUsers.ts`.

- [ ] **Step 7: Write the failing e2e test**

```typescript
// test/users.e2e-spec.ts
import { Test } from '@nestjs/testing';
import { INestApplication } from '@nestjs/common';
import * as request from 'supertest';
import { randomUUID } from 'crypto';
import { AppModule } from '../src/app.module';

describe('Users (e2e)', () => {
  let app: INestApplication;

  beforeAll(async () => {
    process.env.NODE_ENV = 'test';
    process.env.DB_NAME = 'milespilot_test';
    const moduleRef = await Test.createTestingModule({
      imports: [AppModule],
    }).compile();
    app = moduleRef.createNestApplication();
    await app.init();
  });

  afterAll(async () => {
    await app.close();
  });

  it('creates a user lazily on first PATCH and reflects fields on GET', async () => {
    const id = randomUUID();

    await request(app.getHttpServer())
      .patch(`/users/${id}`)
      .send({ flyingBlueStatus: 'GOLD' })
      .expect(200);

    const res = await request(app.getHttpServer())
      .get(`/users/${id}`)
      .expect(200);
    expect(res.body.id).toBe(id);
    expect(res.body.flyingBlueStatus).toBe('GOLD');
  });

  it('404s for an unknown user id', () => {
    return request(app.getHttpServer())
      .get(`/users/${randomUUID()}`)
      .expect(404);
  });
});
```

- [ ] **Step 8: Run it to verify it fails**

```bash
npm run test:e2e -- users
```

Expected: FAIL.

- [ ] **Step 9: Fix until it passes**

```bash
npm run test:e2e -- users
```

Expected: PASS.

- [ ] **Step 10: Commit**

```bash
git add src/users src/migrations src/app.module.ts test/users.e2e-spec.ts
git commit -m "feat: add anonymous User entity and PATCH/GET /users/:id"
```

---

### Task 9: Flying Blue+ catalog sync (weekly cron + manual trigger)

**Files:**
- Create: `backend/src/flying-blue-sync/dto/catalog-response.dto.ts`
- Create: `backend/src/flying-blue-sync/flying-blue-catalog-client.ts`
- Create: `backend/src/flying-blue-sync/flying-blue-sync.service.ts`
- Create: `backend/src/flying-blue-sync/flying-blue-sync.controller.ts`
- Create: `backend/src/flying-blue-sync/flying-blue-sync.module.ts`
- Modify: `backend/src/app.module.ts`
- Create: `backend/test/fixtures/catalog-page-1.json`
- Create: `backend/test/fixtures/catalog-page-2.json`
- Create: `backend/test/fixtures/catalog-page-3.json`
- Test: `backend/test/flying-blue-sync.service.spec.ts`

**Interfaces:**
- Consumes: `MerchantsService.upsertFromSync` (Task 3),
  `OffersRepositoryService.replaceActiveOffer` (Task 5),
  `LoyaltyProgram`/`Country` repositories (Task 2).
- Produces: `FlyingBlueSyncService.runFullSync(): Promise<{ merchantsUpserted:
  number; merchantsFailed: number }>`, `POST /admin/scrape/run`.

- [ ] **Step 1: DTO for one catalog page response**

```typescript
// src/flying-blue-sync/dto/catalog-response.dto.ts
export interface CatalogOfferDto {
  Id: string;
  Description: string;
  Expired: boolean;
  ValidUntil?: string;
}

export interface CatalogMerchantDto {
  MerchantId: string;
  Name: string;
  Code: string;
  CountryCode: string;
  LogoUrl: string;
  ShopUrl: string;
  DefaultEarnAmount: number;
  DefaultEarnMiles: number;
  DefaultEarnCurrency: string;
  Offers: CatalogOfferDto[];
}

export interface CatalogPageDto {
  TotalPages: number;
  TotalItems: number;
  PageNumber: number;
  Data: CatalogMerchantDto[];
}
```

- [ ] **Step 2: Catalog HTTP client**

```typescript
// src/flying-blue-sync/flying-blue-catalog-client.ts
import { Injectable, Logger } from '@nestjs/common';
import { HttpService } from '@nestjs/axios';
import { firstValueFrom } from 'rxjs';
import { CatalogMerchantDto, CatalogPageDto } from './dto/catalog-response.dto';

const BASE_URL = 'https://flyingblueplus-shopping.flyingblue.com';
const PAGE_SIZE = 20;

@Injectable()
export class FlyingBlueCatalogClient {
  private readonly logger = new Logger(FlyingBlueCatalogClient.name);

  constructor(private readonly httpService: HttpService) {}

  async fetchAllMerchants(): Promise<CatalogMerchantDto[]> {
    const merchants: CatalogMerchantDto[] = [];
    let page = 1;
    let totalPages = 1;

    do {
      const url =
        `${BASE_URL}/website/merchants/exploration` +
        `?q=all&sort=popularity&sortDirection=asc&allowedPayFilters=2` +
        `&pageSize=${PAGE_SIZE}&pageNumber=${page}&lang=en`;
      const response = await firstValueFrom(
        this.httpService.get<CatalogPageDto>(url),
      );
      const data = response.data;
      merchants.push(...data.Data);
      totalPages = data.TotalPages;
      this.logger.log(`Fetched catalog page ${page}/${totalPages}`);
      page += 1;
    } while (page <= totalPages);

    return merchants;
  }
}
```

- [ ] **Step 3: Sync service (upsert logic)**

```typescript
// src/flying-blue-sync/flying-blue-sync.service.ts
import { Injectable, Logger } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import { FlyingBlueCatalogClient } from './flying-blue-catalog-client';
import { MerchantsService } from '../merchants/merchants.service';
import { OffersRepositoryService } from '../offers/offers.repository.service';
import { LoyaltyProgram } from '../loyalty-programs/loyalty-program.entity';
import { Country } from '../countries/country.entity';
import { CatalogMerchantDto } from './dto/catalog-response.dto';

export interface SyncRunSummary {
  merchantsUpserted: number;
  merchantsFailed: number;
}

@Injectable()
export class FlyingBlueSyncService {
  private readonly logger = new Logger(FlyingBlueSyncService.name);
  private running = false;

  constructor(
    private readonly catalogClient: FlyingBlueCatalogClient,
    private readonly merchantsService: MerchantsService,
    private readonly offersRepo: OffersRepositoryService,
    @InjectRepository(LoyaltyProgram)
    private readonly programRepo: Repository<LoyaltyProgram>,
    @InjectRepository(Country)
    private readonly countryRepo: Repository<Country>,
  ) {}

  async runFullSync(): Promise<SyncRunSummary> {
    if (this.running) {
      this.logger.warn('Sync already running, skipping this trigger');
      return { merchantsUpserted: 0, merchantsFailed: 0 };
    }
    this.running = true;
    try {
      const program = await this.programRepo.findOneOrFail({
        where: { code: 'FLYING_BLUE' },
      });
      const merchants = await this.catalogClient.fetchAllMerchants();

      let upserted = 0;
      let failed = 0;
      const now = new Date();

      for (const m of merchants) {
        try {
          await this.syncOneMerchant(m, program, now);
          upserted += 1;
        } catch (err) {
          failed += 1;
          this.logger.error(
            `Failed to sync merchant ${m.Code}: ${(err as Error).message}`,
          );
        }
      }

      this.logger.log(
        `Sync complete: ${upserted} upserted, ${failed} failed, ${merchants.length} total`,
      );
      return { merchantsUpserted: upserted, merchantsFailed: failed };
    } finally {
      this.running = false;
    }
  }

  private async syncOneMerchant(
    m: CatalogMerchantDto,
    program: LoyaltyProgram,
    now: Date,
  ): Promise<void> {
    const country = await this.countryRepo.findOne({
      where: { code: m.CountryCode },
    });
    if (!country) {
      throw new Error(
        `Unknown country code "${m.CountryCode}" for merchant ${m.Code} — seed it in countries first`,
      );
    }

    const domain = new URL(m.ShopUrl).hostname.replace(/^www\./, '');
    const merchant = await this.merchantsService.upsertFromSync({
      slug: m.Code,
      name: m.Name,
      logoUrl: m.LogoUrl,
      domain,
    });

    const baseMilesPer10Eur =
      m.DefaultEarnAmount > 0
        ? (m.DefaultEarnMiles * 10) / m.DefaultEarnAmount
        : m.DefaultEarnMiles;

    await this.offersRepo.replaceActiveOffer({
      merchant,
      loyaltyProgram: program,
      country,
      baseMilesPer10Eur,
      trackingUrl: m.ShopUrl,
      source: 'flyingblue_sync',
      scrapedAt: now,
      validFrom: now,
      conditions: m.Offers.filter((o) => !o.Expired).map((o) => o.Description),
    });
  }
}
```

- [ ] **Step 4: Cron wrapper + controller**

```typescript
// src/flying-blue-sync/flying-blue-sync.controller.ts
import { Controller, Post } from '@nestjs/common';
import { Cron, CronExpression } from '@nestjs/schedule';
import { FlyingBlueSyncService } from './flying-blue-sync.service';

@Controller('admin/scrape')
export class FlyingBlueSyncController {
  constructor(private readonly syncService: FlyingBlueSyncService) {}

  @Cron(CronExpression.EVERY_WEEK, { name: 'flying-blue-weekly-sync' })
  async handleWeeklyCron() {
    await this.syncService.runFullSync();
  }

  @Post('run')
  async runManually() {
    return this.syncService.runFullSync();
  }
}
```

`CronExpression.EVERY_WEEK` fires Sunday at midnight server time — matches
the "weekly" requirement; adjust later via a config value if a specific
day/time is wanted, no code restructuring needed since it's one decorator
argument.

- [ ] **Step 5: `FlyingBlueSyncModule`**

```typescript
// src/flying-blue-sync/flying-blue-sync.module.ts
import { Module } from '@nestjs/common';
import { HttpModule } from '@nestjs/axios';
import { TypeOrmModule } from '@nestjs/typeorm';
import { FlyingBlueCatalogClient } from './flying-blue-catalog-client';
import { FlyingBlueSyncService } from './flying-blue-sync.service';
import { FlyingBlueSyncController } from './flying-blue-sync.controller';
import { MerchantsModule } from '../merchants/merchants.module';
import { OffersModule } from '../offers/offers.module';
import { LoyaltyProgram } from '../loyalty-programs/loyalty-program.entity';
import { Country } from '../countries/country.entity';

@Module({
  imports: [
    HttpModule,
    TypeOrmModule.forFeature([LoyaltyProgram, Country]),
    MerchantsModule,
    OffersModule,
  ],
  providers: [FlyingBlueCatalogClient, FlyingBlueSyncService],
  controllers: [FlyingBlueSyncController],
})
export class FlyingBlueSyncModule {}
```

- [ ] **Step 6: Wire into `AppModule`**

Add `FlyingBlueSyncModule` import and to `imports`.

- [ ] **Step 7: Build the 3-page fixture set**

`backend/test/fixtures/catalog-page-1.json` (20 merchants, `TotalPages: 3`),
`catalog-page-2.json`, `catalog-page-3.json` (last page with fewer than 20
to prove partial pages are handled). Use this shape for every merchant
entry, varying `MerchantId`/`Name`/`Code`/`ShopUrl`/`DefaultEarnMiles`:

```json
{
  "TotalPages": 3,
  "TotalItems": 45,
  "PageNumber": 1,
  "Data": [
    {
      "MerchantId": "11111111-1111-1111-1111-111111111111",
      "Name": "Fixture Merchant 1",
      "Code": "fixture-merchant-1",
      "CountryCode": "FR",
      "LogoUrl": "https://example.com/logo1.png",
      "ShopUrl": "https://www.fixture-merchant-1.com/",
      "DefaultEarnAmount": 10,
      "DefaultEarnMiles": 20,
      "DefaultEarnCurrency": "EUR",
      "Offers": [
        {
          "Id": "aaaa1111-aaaa-1111-aaaa-111111111111",
          "Description": "10% off first order",
          "Expired": false,
          "ValidUntil": "2026-12-31T23:59:00Z"
        }
      ]
    }
  ]
}
```

Generate 20 such entries for page 1 (`fixture-merchant-1` through
`fixture-merchant-20`, incrementing the UUID's last segment and the
`DefaultEarnMiles` value), 20 for page 2 (`fixture-merchant-21` through
`-40`, `TotalPages: 3`, `PageNumber: 2`), and 5 for page 3
(`fixture-merchant-41` through `-45`, `PageNumber: 3`). Every entry uses
`"CountryCode": "FR"`.

- [ ] **Step 8: Write the failing integration test**

```typescript
// test/flying-blue-sync.service.spec.ts
import * as nock from 'nock';
import * as fs from 'fs';
import * as path from 'path';
import { Test } from '@nestjs/testing';
import { AppModule } from '../src/app.module';
import { FlyingBlueSyncService } from '../src/flying-blue-sync/flying-blue-sync.service';
import { getRepositoryToken } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import { Country } from '../src/countries/country.entity';
import { LoyaltyProgram } from '../src/loyalty-programs/loyalty-program.entity';
import { Merchant } from '../src/merchants/merchant.entity';
import { Offer } from '../src/offers/offer.entity';

function loadFixture(name: string) {
  return JSON.parse(
    fs.readFileSync(path.join(__dirname, 'fixtures', name), 'utf-8'),
  );
}

describe('FlyingBlueSyncService', () => {
  let syncService: FlyingBlueSyncService;
  let countryRepo: Repository<Country>;
  let programRepo: Repository<LoyaltyProgram>;
  let merchantRepo: Repository<Merchant>;
  let offerRepo: Repository<Offer>;

  beforeAll(async () => {
    process.env.NODE_ENV = 'test';
    process.env.DB_NAME = 'milespilot_test';
    const moduleRef = await Test.createTestingModule({
      imports: [AppModule],
    }).compile();
    syncService = moduleRef.get(FlyingBlueSyncService);
    countryRepo = moduleRef.get(getRepositoryToken(Country));
    programRepo = moduleRef.get(getRepositoryToken(LoyaltyProgram));
    merchantRepo = moduleRef.get(getRepositoryToken(Merchant));
    offerRepo = moduleRef.get(getRepositoryToken(Offer));

    await countryRepo.save(countryRepo.create({ code: 'FR', name: 'France' }));
    await programRepo.save(
      programRepo.create({ code: 'FLYING_BLUE', name: 'Flying Blue' }),
    );
  });

  afterEach(() => {
    nock.cleanAll();
  });

  function mockThreePages() {
    const base = 'https://flyingblueplus-shopping.flyingblue.com';
    const path1 =
      '/website/merchants/exploration?q=all&sort=popularity&sortDirection=asc&allowedPayFilters=2&pageSize=20&pageNumber=1&lang=en';
    const path2 = path1.replace('pageNumber=1', 'pageNumber=2');
    const path3 = path1.replace('pageNumber=1', 'pageNumber=3');
    nock(base).get(path1).reply(200, loadFixture('catalog-page-1.json'));
    nock(base).get(path2).reply(200, loadFixture('catalog-page-2.json'));
    nock(base).get(path3).reply(200, loadFixture('catalog-page-3.json'));
  }

  it('walks all 3 pages and upserts all 45 merchants with an active offer each', async () => {
    mockThreePages();

    const summary = await syncService.runFullSync();

    expect(summary.merchantsUpserted).toBe(45);
    expect(summary.merchantsFailed).toBe(0);

    const merchantCount = await merchantRepo.count({
      where: { slug: 'fixture-merchant-45' },
    });
    expect(merchantCount).toBe(1);

    const activeOffers = await offerRepo.find({
      where: { isActive: true, merchant: { slug: 'fixture-merchant-1' } },
      relations: ['merchant'],
    });
    expect(activeOffers.length).toBe(1);
  });

  it('re-running the sync updates the existing merchant instead of duplicating it', async () => {
    mockThreePages();
    await syncService.runFullSync();
    mockThreePages();
    await syncService.runFullSync();

    const merchantCount = await merchantRepo.count({
      where: { slug: 'fixture-merchant-1' },
    });
    expect(merchantCount).toBe(1);

    const offerCount = await offerRepo.count({
      where: { merchant: { slug: 'fixture-merchant-1' } },
    });
    expect(offerCount).toBe(2); // one from each run, only the latest active

    const activeOffers = await offerRepo.find({
      where: { isActive: true, merchant: { slug: 'fixture-merchant-1' } },
    });
    expect(activeOffers.length).toBe(1);
  });

  it('does not double-run when called concurrently', async () => {
    mockThreePages();
    const [first, second] = await Promise.all([
      syncService.runFullSync(),
      syncService.runFullSync(),
    ]);
    const totalUpserted = first.merchantsUpserted + second.merchantsUpserted;
    // exactly one of the two concurrent calls should have done the work
    expect(totalUpserted).toBe(45);
  });
});
```

- [ ] **Step 9: Run it to verify it fails**

```bash
npm run test:e2e -- flying-blue-sync
```

Expected: FAIL — module/service don't exist yet.

- [ ] **Step 10: Fix until it passes**

```bash
npm run test:e2e -- flying-blue-sync
```

Expected: PASS, all 3 cases. If the concurrency test is flaky because both
calls start before either sets `running = true`, that's a real bug in Step
3 — the guard read/write isn't atomic against interleaved async calls
within a single Node process; fix by setting `this.running = true`
synchronously as the very first line of `runFullSync` (already the case
above) — Node's single-threaded event loop guarantees no other call can
interleave between the `if (this.running)` check and the assignment as
long as there's no `await` between them, which this code respects.

- [ ] **Step 11: Commit**

```bash
git add src/flying-blue-sync test/fixtures test/flying-blue-sync.service.spec.ts src/app.module.ts
git commit -m "feat: add Flying Blue+ catalog sync with weekly cron and manual trigger"
```

---

### Task 10: Full-flow E2E and seed wiring

**Files:**
- Modify: `backend/src/database/seed.ts` (idempotency check already in
  place from Task 2/4 — no changes needed unless Step 1 finds a gap)
- Modify: `backend/README.md` (create if absent — setup instructions)
- Test: `backend/test/full-flow.e2e-spec.ts`

**Interfaces:**
- Consumes: everything from Tasks 1–9.
- Produces: nothing new — this task is a verification pass proving the
  whole system works together against a real (test) Postgres, end to end,
  the way the Chrome extension will use it.

- [ ] **Step 1: Write the failing full-flow e2e test**

```typescript
// test/full-flow.e2e-spec.ts
import { Test } from '@nestjs/testing';
import { INestApplication } from '@nestjs/common';
import * as request from 'supertest';
import { getRepositoryToken } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import { randomUUID } from 'crypto';
import { AppModule } from '../src/app.module';
import { Country } from '../src/countries/country.entity';
import { LoyaltyProgram } from '../src/loyalty-programs/loyalty-program.entity';
import { Card } from '../src/cards/card.entity';
import { CardRewardRule } from '../src/cards/card-reward-rule.entity';
import { OffersRepositoryService } from '../src/offers/offers.repository.service';
import { Merchant } from '../src/merchants/merchant.entity';

describe('Full purchase flow (e2e)', () => {
  let app: INestApplication;
  let countryRepo: Repository<Country>;
  let programRepo: Repository<LoyaltyProgram>;
  let cardRepo: Repository<Card>;
  let ruleRepo: Repository<CardRewardRule>;
  let merchantRepo: Repository<Merchant>;
  let offersRepo: OffersRepositoryService;

  beforeAll(async () => {
    process.env.NODE_ENV = 'test';
    process.env.DB_NAME = 'milespilot_test';
    const moduleRef = await Test.createTestingModule({
      imports: [AppModule],
    }).compile();
    app = moduleRef.createNestApplication();
    countryRepo = moduleRef.get(getRepositoryToken(Country));
    programRepo = moduleRef.get(getRepositoryToken(LoyaltyProgram));
    cardRepo = moduleRef.get(getRepositoryToken(Card));
    ruleRepo = moduleRef.get(getRepositoryToken(CardRewardRule));
    merchantRepo = moduleRef.get(getRepositoryToken(Merchant));
    offersRepo = moduleRef.get(OffersRepositoryService);
    await app.init();
  });

  afterAll(async () => {
    await app.close();
  });

  it('extension flow: resolve domain -> list cards -> save profile -> get reward', async () => {
    const country = await countryRepo.save(
      countryRepo.create({ code: 'FR_FULLFLOW', name: 'France' }),
    );
    const program = await programRepo.save(
      programRepo.create({ code: 'FLYING_BLUE_FULLFLOW', name: 'Flying Blue' }),
    );
    const card = await cardRepo.save(
      cardRepo.create({ name: 'Amex KLM Gold Fullflow', issuer: 'American Express', program }),
    );
    await ruleRepo.save(
      ruleRepo.create({ card, milesPer10Eur: 10, validFrom: new Date('2026-01-01') }),
    );
    const merchant = await merchantRepo.save(
      merchantRepo.create({ name: 'Apple', slug: 'apple-fullflow' }),
    );
    await offersRepo.replaceActiveOffer({
      merchant,
      loyaltyProgram: program,
      country,
      baseMilesPer10Eur: 15,
      trackingUrl: 'https://apple.com',
      source: 'flyingblue_sync',
      scrapedAt: new Date(),
      validFrom: new Date('2026-01-01'),
      conditions: [],
    });
    const domainRepoResolveUrl = '/merchants/resolve?domain=apple.com';
    // Note: this merchant wasn't registered with a domain via upsertFromSync
    // in this test, so resolve is exercised separately in merchants.e2e-spec.ts.
    // This test focuses on the cards -> profile -> offers path.
    void domainRepoResolveUrl;

    const cardsRes = await request(app.getHttpServer()).get('/cards').expect(200);
    expect(cardsRes.body.some((c: any) => c.id === card.id)).toBe(true);

    const userId = randomUUID();
    await request(app.getHttpServer())
      .patch(`/users/${userId}`)
      .send({ selectedCardId: card.id, countryId: country.id, flyingBlueStatus: 'GOLD' })
      .expect(200);

    const offerRes = await request(app.getHttpServer())
      .get(`/offers?merchant=apple-fullflow&country=FR_FULLFLOW&cardId=${card.id}&priceEur=1629`)
      .expect(200);

    expect(offerRes.body.status).toBe('VERIFIED');
    expect(offerRes.body.totalMiles).toBeCloseTo(4072.5);
    expect(
      offerRes.body.conditions.some((c: string) => c.includes('non confirmée') || c.includes('non confirmee') || c.includes('hypothèse') || c.includes('hypothese')),
    ).toBe(true);
  });
});
```

- [ ] **Step 2: Run it to verify it fails or passes cleanly**

```bash
npm run test:e2e -- full-flow
```

Expected: this should PASS if Tasks 1–8 are correctly wired (it's an
integration check, not new functionality) — if it fails, that's a real
integration bug between modules (e.g. a missing export in a module) to fix
before considering the backend done.

- [ ] **Step 3: Run the entire test suite**

```bash
npm test && npm run test:e2e
```

Expected: PASS across unit and e2e suites.

- [ ] **Step 4: Write `backend/README.md`**

```markdown
# MilesPilot backend

## Setup

    cp .env.example .env
    docker compose up -d
    docker compose exec postgres psql -U milespilot -c "CREATE DATABASE milespilot_test;"
    npm install
    npm run typeorm migration:run
    npm run seed

## Test

    npm test        # unit tests
    npm run test:e2e  # e2e tests (needs milespilot_test DB, see above)

## API

- `GET /offers?merchant=:slug&country=:code&cardId=:id?&priceEur=:amount`
- `GET /merchants/resolve?domain=:domain`
- `GET /cards`
- `PATCH /users/:id`, `GET /users/:id`
- `POST /admin/scrape/run` (manual catalog sync trigger)

Weekly automatic sync runs via cron (`@nestjs/schedule`), no action needed.
```

- [ ] **Step 5: Commit**

```bash
git add test/full-flow.e2e-spec.ts backend/README.md
git commit -m "test: add full purchase-flow e2e test and backend README"
```

---

## Definition of done

- `npm test` and `npm run test:e2e` both pass.
- `docker compose up` brings up Postgres + the app cleanly from a fresh
  clone.
- `POST /admin/scrape/run` against the real Flying Blue+ site (manual,
  one-time check, not part of the automated test suite) populates
  `merchants`/`offers` with real data.
- `GET /offers?merchant=belvilla&country=FR&cardId=<amex-klm-gold-id>&priceEur=100`
  against that real data returns a `VERIFIED` result with a sane Miles
  number, or `UNVERIFIED` if the real Belvilla offer changed since this
  plan was written — either is correct; a fabricated number is not.
