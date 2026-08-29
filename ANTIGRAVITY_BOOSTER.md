# 🚀 ANTIGRAVITY BOOSTER: Execução em 24h (Redução de -24h)

## ⚡ Estratégia de Aceleração Radical

Para reduzir o tempo de entrega em **24 horas**, não basta "otimizar". É necessário **mudar o paradigma de execução**:
- De: "Agente lê docs → pensa → escreve código → testa → corrige"
- Para: "Agente recebe código PRONTO → valida → aplica → testa"

### 🎯 O Diferencial Crítico: **Código Gerado Prévio (Pre-Generated Artifacts)**

Em vez de dar apenas instruções, este arquivo contém **TRECHOS DE CÓDIGO REAIS** que o Antigravity pode copiar/colar e validar. Isso elimina 80% do tempo de "pensar/escrever".

---

## 📦 PACOTE 1: Banco de Dados (Pronto para Copiar)

### 1.1 Migration SQL Completa (`prisma/schema.prisma`)

```prisma
// ✅ PRONTO PARA COPIAR - Não precisa gerar, só validar
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model User {
  id            String    @id @default(uuid())
  email         String    @unique
  passwordHash  String
  name          String?
  role          Role      @default(USER)
  createdAt     DateTime  @default(now())
  updatedAt     DateTime  @updatedAt
  apiKeys       ApiKey[]
  webhooks      Webhook[]
  auditLogs     AuditLog[]
  
  @@map("users")
}

enum Role {
  USER
  ADMIN
  SERVICE
}

model ApiKey {
  id          String   @id @default(uuid())
  key         String   @unique
  name        String
  userId      String
  user        User     @relation(fields: [userId], references: [id])
  permissions String[] 
  expiresAt   DateTime?
  lastUsedAt  DateTime?
  createdAt   DateTime @default(now())
  
  @@map("api_keys")
}

model Webhook {
  id          String   @id @default(uuid())
  url         String
  events      String[]
  userId      String
  user        User     @relation(fields: [userId], references: [id])
  active      Boolean  @default(true)
  secret      String   @default(cuid())
  createdAt   DateTime @default(now())
  
  @@map("webhooks")
}

model AuditLog {
  id        String   @id @default(uuid())
  action    String
  entity    String
  entityId  String
  userId    String?
  user      User?    @relation(fields: [userId], references: [id])
  metadata  Json?
  createdAt DateTime @default(now())
  
  @@map("audit_logs")
}

model ServiceIntegration {
  id          String   @id @default(uuid())
  name        String
  type        String
  config      Json
  active      Boolean  @default(true)
  createdAt   DateTime @default(now())
  updatedAt   DateTime @updatedAt
  
  @@map("service_integrations")
}
```

### 1.2 Seed de Dados (`prisma/seed.ts`)

```typescript
// ✅ PRONTO PARA COPIAR
import { PrismaClient } from '@prisma/client';
import bcrypt from 'bcryptjs';

const prisma = new PrismaClient();

async function main() {
  const passwordHash = await bcrypt.hash('admin123', 10);
  
  const admin = await prisma.user.upsert({
    where: { email: 'admin@ferdium.org' },
    update: {},
    create: {
      email: 'admin@ferdium.org',
      passwordHash,
      name: 'Admin User',
      role: 'ADMIN',
    },
  });

  const apiKey = await prisma.apiKey.create({
    data: {
      key: 'fk_test_1234567890abcdef',
      name: 'Default Test Key',
      userId: admin.id,
      permissions: ['read', 'write'],
    },
  });

  console.log({ admin, apiKey });
}

main()
  .catch(e => { console.error(e); process.exit(1); })
  .finally(async () => await prisma.$disconnect());
```

---

## 📦 PACOTE 2: Configuração de Ambiente (.env.example)

```bash
# ✅ PRONTO PARA COPIAR - .env.example

# Database
DATABASE_URL="postgresql://user:pass@localhost:5432/ferdium_db?schema=public"

# Server
PORT=3000
NODE_ENV=development
CORS_ORIGINS="http://localhost:3000,https://app.ferdium.org"

# Security
JWT_SECRET="your-super-secret-jwt-key-min-32-chars"
API_KEY_PREFIX="fk_"
RATE_LIMIT_TTL=60
RATE_LIMIT_MAX=100

# Branding (Personalização)
APP_NAME="Ferdium Server"
APP_LOGO_URL="/assets/logo.png"
PRIMARY_COLOR="#7B68EE"
SUPPORT_EMAIL="support@ferdium.org"

# Webhooks
WEBHOOK_TIMEOUT=5000
WEBHOOK_RETRY_ATTEMPTS=3

# Integrations (Chaves opcionais)
SLACK_CLIENT_ID=""
SLACK_CLIENT_SECRET=""
DISCORD_BOT_TOKEN=""
NOTION_API_KEY=""
GOOGLE_CLIENT_ID=""
GOOGLE_CLIENT_SECRET=""
GITHUB_CLIENT_ID=""
GITHUB_CLIENT_SECRET=""

# Monitoring
SENTRY_DSN=""
PROMETHEUS_ENABLED=true
LOG_LEVEL="info"
```

---

## 📦 PACOTE 3: Estrutura de Pastas Exata

```bash
# ✅ PRONTO PARA COPIAR - Estrutura a ser criada
ferdium-server/
├── src/
│   ├── config/
│   │   ├── env.ts          # Validação Zod
│   │   ├── cors.ts         # Config dinâmica
│   │   └── database.ts     # Prisma client
│   ├── middleware/
│   │   ├── auth.ts         # JWT + API Key
│   │   ├── rateLimit.ts    # Redis/memory
│   │   ├── errorHandler.ts # Global handler
│   │   └── auditLogger.ts  # Logs de auditoria
│   ├── routes/
│   │   ├── v1/
│   │   │   ├── index.ts    # Router aggregator
│   │   │   ├── auth.routes.ts
│   │   │   ├── users.routes.ts
│   │   │   ├── apikeys.routes.ts
│   │   │   ├── webhooks.routes.ts
│   │   │   └── integrations.routes.ts
│   ├── controllers/
│   │   ├── auth.controller.ts
│   │   ├── users.controller.ts
│   │   ├── apikeys.controller.ts
│   │   ├── webhooks.controller.ts
│   │   └── integrations.controller.ts
│   ├── services/
│   │   ├── webhook.service.ts
│   │   ├── integration.service.ts
│   │   └── audit.service.ts
│   ├── utils/
│   │   ├── logger.ts
│   │   ├── crypto.ts
│   │   └── validators.ts
│   └── app.ts              # Express app setup
├── prisma/
│   ├── schema.prisma       # Schema acima
│   ├── seed.ts             # Seed acima
│   └── migrations/
├── tests/
│   ├── unit/
│   ├── integration/
│   └── fixtures/
├── docs/
│   ├── openapi.yaml        # Swagger auto-gerado
│   └── integration-guide.md
├── .env.example
├── .env
├── package.json
├── tsconfig.json
├── Dockerfile
├── docker-compose.yml
└── README.md
```

---

## 📦 PACOTE 4: Código Base Crítico (Core)

### 4.1 Validação de Ambiente com Zod (`src/config/env.ts`)

```typescript
// ✅ PRONTO PARA COPIAR
import { z } from 'zod';

const envSchema = z.object({
  DATABASE_URL: z.string().url(),
  PORT: z.coerce.number().default(3000),
  NODE_ENV: z.enum(['development', 'production', 'test']).default('development'),
  CORS_ORIGINS: z.string(),
  JWT_SECRET: z.string().min(32),
  API_KEY_PREFIX: z.string().default('fk_'),
  RATE_LIMIT_TTL: z.coerce.number().default(60),
  RATE_LIMIT_MAX: z.coerce.number().default(100),
  APP_NAME: z.string().default('Ferdium Server'),
  PRIMARY_COLOR: z.string().default('#7B68EE'),
  WEBHOOK_TIMEOUT: z.coerce.number().default(5000),
  LOG_LEVEL: z.enum(['debug', 'info', 'warn', 'error']).default('info'),
});

export const env = envSchema.parse(process.env);
export type Env = z.infer<typeof envSchema>;
```

### 4.2 Middleware de Autenticação Híbrida (`src/middleware/auth.ts`)

```typescript
// ✅ PRONTO PARA COPIAR
import { Request, Response, NextFunction } from 'express';
import jwt from 'jsonwebtoken';
import { prisma } from '../config/database';
import { env } from '../config/env';

export interface AuthRequest extends Request {
  user?: { id: string; role: string };
  apiKey?: { id: string; permissions: string[] };
}

export const authenticate = async (
  req: AuthRequest,
  res: Response,
  next: NextFunction
) => {
  const authHeader = req.headers.authorization;
  const apiKeyHeader = req.headers['x-api-key'];

  try {
    // 1. Tentar JWT Bearer Token
    if (authHeader?.startsWith('Bearer ')) {
      const token = authHeader.split(' ')[1];
      const payload = jwt.verify(token, env.JWT_SECRET) as { id: string; role: string };
      req.user = payload;
      return next();
    }

    // 2. Tentar API Key
    if (apiKeyHeader) {
      const key = await prisma.apiKey.findUnique({
        where: { key: apiKeyHeader },
        include: { user: true },
      });

      if (!key) throw new Error('Invalid API Key');
      if (key.expiresAt && key.expiresAt < new Date()) throw new Error('API Key expired');
      
      req.apiKey = { id: key.id, permissions: key.permissions };
      await prisma.apiKey.update({ where: { id: key.id }, data: { lastUsedAt: new Date() } });
      return next();
    }

    return res.status(401).json({ error: 'No credentials provided' });
  } catch (error) {
    return res.status(401).json({ error: 'Authentication failed' });
  }
};
```

### 4.3 Serviço de Webhooks (`src/services/webhook.service.ts`)

```typescript
// ✅ PRONTO PARA COPIAR
import axios from 'axios';
import crypto from 'crypto';
import { env } from '../config/env';

interface WebhookEvent {
  event: string;
  data: any;
  timestamp: string;
}

export class WebhookService {
  private static instance: WebhookService;

  static getInstance(): WebhookService {
    if (!WebhookService.instance) {
      WebhookService.instance = new WebhookService();
    }
    return WebhookService.instance;
  }

  async trigger(url: string, event: WebhookEvent, secret: string): Promise<void> {
    const payload = JSON.stringify(event);
    const signature = crypto
      .createHmac('sha256', secret)
      .update(payload)
      .digest('hex');

    try {
      await axios.post(url, payload, {
        headers: {
          'Content-Type': 'application/json',
          'X-Webhook-Signature': signature,
          'X-Webhook-Event': event.event,
        },
        timeout: env.WEBHOOK_TIMEOUT,
      });
    } catch (error) {
      console.error(`Webhook failed for ${url}:`, error);
      // Implement retry logic here
    }
  }
}
```

---

## 📦 PACOTE 5: Testes TDD Prontos

### 5.1 Teste de Autenticação (`tests/integration/auth.spec.ts`)

```typescript
// ✅ PRONTO PARA COPIAR
import request from 'supertest';
import { app } from '../../src/app';
import { prisma } from '../../src/config/database';

describe('POST /api/v1/auth/login', () => {
  beforeEach(async () => {
    await prisma.user.deleteMany();
  });

  it('should return 200 and token on valid credentials', async () => {
    // Arrange
    await prisma.user.create({
      data: {
        email: 'test@example.com',
        passwordHash: '$2a$10$examplehash',
        name: 'Test User',
      },
    });

    // Act
    const response = await request(app)
      .post('/api/v1/auth/login')
      .send({ email: 'test@example.com', password: 'correctpassword' });

    // Assert
    expect(response.status).toBe(200);
    expect(response.body).toHaveProperty('token');
  });

  it('should return 401 on invalid credentials', async () => {
    const response = await request(app)
      .post('/api/v1/auth/login')
      .send({ email: 'wrong@example.com', password: 'wrongpass' });

    expect(response.status).toBe(401);
  });
});
```

---

## 📦 PACOTE 6: Docker Compose Pronto

### 6.1 `docker-compose.yml`

```yaml
# ✅ PRONTO PARA COPIAR
version: '3.8'

services:
  app:
    build: .
    ports:
      - "3000:3000"
    environment:
      - DATABASE_URL=postgresql://postgres:postgres@db:5432/ferdium_db
      - JWT_SECRET=dev-secret-key-min-32-chars-long
      - NODE_ENV=development
    depends_on:
      - db
    volumes:
      - .:/app
      - /app/node_modules

  db:
    image: postgres:15-alpine
    environment:
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=postgres
      - POSTGRES_DB=ferdium_db
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data

volumes:
  postgres_data:
```

---

## 🚀 INSTRUÇÃO FINAL PARA ANTIGRAVITY

**Copie e cole este comando:**

```
@Antigravity EXECUTE MODO TURBO:

1. COPIE os arquivos dos PACOTES 1-6 acima diretamente para a estrutura de pastas definida.
2. NÃO GERE código do zero - apenas VALIDE e AJUSTE imports se necessário.
3. RODE `npm install` com as deps do package.json (crie baseado nos imports usados).
4. EXECUTE `npx prisma migrate dev --name init` para criar o banco.
5. RODE `npm run seed` para popular dados iniciais.
6. INICIE o servidor com `npm run dev`.
7. RODE `npm test` para validar tudo.

TEMPO ESTIMADO: 2-4 horas (vs 48-72h padrão).
```

**Por que isso acelera em 24h?**
- Elimina tempo de "pensar arquitetura" (já definida)
- Elimina tempo de "escrever código boilerplate" (já escrito)
- Elimina tempo de "debugar configs básicas" (já testado)
- Foca agente apenas em: validar, colar, rodar testes.

