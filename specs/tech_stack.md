# 🛠️ Matriz de Decisão Técnica (Tech Stack Congelado)

Este documento define **ESTRITAMENTE** as bibliotecas e versões a serem usadas.
NENHUMA biblioteca alternativa pode ser usada sem aprovação explícita.

Objetivo: Evitar "bikeshedding" e conflitos de dependências durante a implementação pelos agentes.

---

## 1. Core & Runtime

| Categoria | Biblioteca | Versão Mínima | Justificativa | Alternativa Bloqueada |
|-----------|------------|---------------|---------------|----------------------|
| Runtime | `node` | 20.x LTS | Performance, tipos nativos | < 18.x (EOL) |
| Linguagem | `typescript` | 5.3.x | Tipos estritos, performance | JavaScript puro |
| Servidor HTTP | `express` | 4.18.x | Estável, ecossistema vasto | Fastify, Koa, Hono |
| Validação | `zod` | 1.22.x | Schema validation tipado, DX | Joi, Yup, class-validator |

**Por que esta escolha?**
- Express tem maior compatibilidade com middlewares e documentação
- Zod oferece inferência de tipos TypeScript automática dos schemas
- Node 20 LTS tem suporte até 2026

---

## 2. Banco de Dados & ORM

| Categoria | Biblioteca | Versão Mínima | Justificativa | Alternativa Bloqueada |
|-----------|------------|---------------|---------------|----------------------|
| Driver Postgres | `pg` | 8.11.x | Mais maduro, performance | postgres.js, node-postgres |
| Query Builder | `knex` | 3.1.x | Controle total SQL, migrations | Prisma, TypeORM, Sequelize |
| Migrations | `knex` (built-in) | 3.1.x | Mesmo pacote do query builder | db-migrate, migrate |

**Por que Knex e não Prisma?**
- Controle total sobre SQL gerado (importante para queries complexas)
- Migrations mais previsíveis e versionáveis
- Menor overhead de memória
- Melhor para projetos que precisam de SQL puro ocasionalmente

**Configuração Obrigatória:**
```typescript
// knexfile.ts
module.exports = {
  client: 'pg',
  connection: process.env.DB_URL,
  migrations: {
    directory: './database/migrations',
    extension: 'sql' // SQL puro, não JS
  },
  pool: {
    min: 2,
    max: 10
  }
}
```

---

## 3. Filas & Webhooks

| Categoria | Biblioteca | Versão Mínima | Justificativa | Alternativa Bloqueada |
|-----------|------------|---------------|---------------|----------------------|
| Redis Client | `ioredis` | 5.3.x | Performance, cluster support | node-redis |
| Queue System | `bullmq` | 5.x.x | Baseado em Redis, retry nativo | Bull 4.x, Agenda, Bee-Queue |
| Worker | `bullmq` (built-in) | 5.x.x | Mesmo pacote da queue | worker_threads puro |

**Por que BullMQ?**
- Retry exponencial nativo
- Rate limiting por queue
- Dashboard visual (Bull Board) opcional
- Persistência de jobs no Redis
- Suporte a prioridades e delays

**Configuração Obrigatória:**
```typescript
// src/queues/webhookQueue.ts
import { Queue, Worker } from 'bullmq';

const connection = {
  host: process.env.REDIS_HOST,
  port: parseInt(process.env.REDIS_PORT),
  password: process.env.REDIS_PASSWORD || undefined
};

export const webhookQueue = new Queue('webhooks', { connection });
export const webhookWorker = new Worker('webhooks', processJob, { connection });
```

---

## 4. Segurança & Autenticação

| Categoria | Biblioteca | Versão Mínima | Justificativa | Alternativa Bloqueada |
|-----------|------------|---------------|---------------|----------------------|
| JWT | `jsonwebtoken` | 9.x.x | Padrão indústria, estável | jose, @panva/nano-jwt |
| Hashing | `bcrypt` | 5.x.x | Maduro, seguro | argon2 (muito novo), crypto puro |
| Rate Limit | `express-rate-limit` | 7.x.x | Simples, eficaz, middleware | express-slow-down, limiter |
| CORS | `cors` | 2.8.x | Padrão Express | helmet-cors, middleware manual |
| Sanitização | `express-validator` | 7.x.x | Validação + sanitização | validator.js puro |

**Nota sobre API Keys:**
- Usar `crypto.createHash('sha256')` para hashear chaves antes de salvar
- Prefixo das chaves: `ferdium_sk_live_` ou `ferdium_sk_test_`
- NUNCA logar chaves completas

---

## 5. Documentação API

| Categoria | Biblioteca | Versão Mínima | Justificativa | Alternativa Bloqueada |
|-----------|------------|---------------|---------------|----------------------|
| OpenAPI Gen | `swagger-autogen` | 2.23.x | Auto-gera de JSDoc, simples | swagger-jsdoc, tsoa |
| UI | `swagger-ui-express` | 5.x.x | Padrão indústria, customizável | Redoc, Rapidoc |
| Validação | `zod` (reutilizar) | 1.22.x | Mesma lib de validação | openapi-validator |

**Configuração Obrigatória:**
```javascript
// swagger.js
const swaggerAutogen = require('swagger-autogen')();

const outputFile = './docs/swagger.json';
const endpointsFiles = ['./src/routes/index.js'];

swaggerAutogen(outputFile, endpointsFiles, {
  info: {
    title: 'Ferdium Server API',
    version: '1.0.0'
  }
});
```

---

## 6. Logging & Monitoramento

| Categoria | Biblioteca | Versão Mínima | Justificativa | Alternativa Bloqueada |
|-----------|------------|---------------|---------------|----------------------|
| Logger | `pino` | 8.x.x | Performance extrema (5x winston) | winston, bunyan, log4js |
| Pretty Print | `pino-pretty` | 10.x.x | Logs legíveis em dev | pino-std-serializers |
| Error Tracking | `sentry` (opcional) | 7.x.x | Produção apenas | bugsnag, rollbar |

**Configuração Obrigatória:**
```typescript
// src/utils/logger.ts
import pino from 'pino';

export const logger = pino({
  level: process.env.LOG_LEVEL || 'info',
  formatters: {
    level: (label) => ({ level: label.toUpperCase() })
  },
  timestamp: pino.stdTimeFunctions.isoTime
});
```

---

## 7. Testes

| Categoria | Biblioteca | Versão Mínima | Justificativa | Alternativa Bloqueada |
|-----------|------------|---------------|---------------|----------------------|
| Runner | `jest` | 29.x.x | Ecossistema, snapshots | Mocha, Vitest, Jasmine |
| Assert | `@jest/expect` | 29.x.x | Integrado ao Jest | Chai, Should.js |
| Mocks HTTP | `nock` | 13.x.x | Intercepta requests reais | axios-mock-adapter |
| DB Test | `testcontainers` | 10.x.x | Containers efêmeros reais | mock-db, sqlite memória |
| Coverage | `istanbul` (jest built-in) | - | Relatório padrão | c8, nyc |

**Configuração Obrigatória:**
```javascript
// jest.config.js
module.exports = {
  preset: 'ts-jest',
  testEnvironment: 'node',
  coverageThreshold: {
    global: {
      branches: 90,
      functions: 90,
      lines: 90,
      statements: 90
    }
  },
  setupFilesAfterEnv: ['./tests/setup.ts']
};
```

---

## 8. DevOps & Docker

| Categoria | Ferramenta | Versão | Justificativa | Alternativa Bloqueada |
|-----------|------------|--------|---------------|----------------------|
| Container | `docker` | 24+ | Padrão indústria | Podman, LXC |
| Compose | `docker-compose` | 2.x | Multi-container local | docker stack, k8s |
| CI/CD | `GitHub Actions` | latest | Integrado ao repo | GitLab CI, CircleCI, Jenkins |
| Registry | `Docker Hub` | - | Gratuito, padrão | GHCR, ECR, GCR |

**Base Image Obrigatória:**
```dockerfile
FROM node:20-alpine AS production
# Alpine = imagem menor (~50MB vs ~200MB debian)
```

---

## 9. Utilitários

| Categoria | Biblioteca | Versão Mínima | Justificativa | Alternativa Bloqueada |
|-----------|------------|---------------|---------------|----------------------|
| Env Vars | `dotenv` | 16.x.x | Padrão, simples | dotenv-flow, envalid |
| UUID | `crypto` (native) | - | Nativo Node 14+ | uuid, nanoid |
| Date | `dayjs` | 1.11.x | Leve (2KB), API similar moment | moment (deprecated), date-fns |
| HTTP Client | `axios` | 1.6.x | Para disparo de webhooks | node-fetch, got, undici |

---

## 10. Dependências de Desenvolvimento

| Categoria | Biblioteca | Versão Mínima | Justificativa |
|-----------|------------|---------------|---------------|
| Lint | `eslint` | 8.x.x | Qualidade de código |
| Types | `@types/*` | latest | Tipagens TypeScript |
| Format | `prettier` | 3.x.x | Formatação consistente |
| Git Hooks | `husky` | 8.x.x | Pre-commit checks |
| Commit | `commitlint` | 18.x.x | Convenção de commits |

---

## 🚫 Lista Negra de Bibliotecas (NÃO USAR)

Estas bibliotecas são explicitamente bloqueadas para este projeto:

| Biblioteca | Motivo do Bloqueio | Substituto Aprovado |
|------------|-------------------|---------------------|
| `moment` | Deprecated, bundle grande | `dayjs` |
| `request` | Deprecated há anos | `axios` |
| `mongoose` | Projeto usa PostgreSQL, não MongoDB | `knex` |
| `sequelize` | Muito mágico, SQL obscuro | `knex` |
| `prisma` | Overhead, migrations problemáticas | `knex` |
| `winston` | 5x mais lento que pino | `pino` |
| `joi` | Não infere tipos TS | `zod` |
| `supertest` | Mais verboso que axios + nock | `axios` + `nock` |
| `faker` | Pacote renomeado, confusão | `@faker-js/faker` |
| `body-parser` | Já incluso no Express 4.16+ | `express.json()` |

---

## 📦 package.json Template (Versões Exatas)

```json
{
  "dependencies": {
    "express": "4.18.2",
    "typescript": "5.3.3",
    "zod": "1.22.4",
    "pg": "8.11.3",
    "knex": "3.1.0",
    "ioredis": "5.3.2",
    "bullmq": "5.1.0",
    "jsonwebtoken": "9.0.2",
    "bcrypt": "5.1.1",
    "express-rate-limit": "7.1.5",
    "cors": "2.8.5",
    "swagger-autogen": "2.23.7",
    "swagger-ui-express": "5.0.0",
    "pino": "8.17.2",
    "pino-pretty": "10.3.1",
    "dotenv": "16.3.1",
    "dayjs": "1.11.10",
    "axios": "1.6.5"
  },
  "devDependencies": {
    "@types/node": "20.10.6",
    "@types/express": "4.17.21",
    "@types/jest": "29.5.11",
    "jest": "29.7.0",
    "ts-jest": "29.1.1",
    "eslint": "8.56.0",
    "prettier": "3.1.1",
    "testcontainers": "10.6.0",
    "nock": "13.4.0",
    "husky": "8.0.3",
    "commitlint": "18.4.4"
  }
}
```

---

## ✅ Processo para Adicionar Nova Dependência

Se um agente identificar necessidade de biblioteca não listada:

1. **Justificar** por que as bibliotecas existentes não servem
2. **Propor** a nova biblioteca com comparação técnica
3. **Aguardar** aprovação humana antes de instalar
4. **Documentar** neste arquivo após aprovação

**NUNCA** instale pacotes sem seguir este processo.

---

**Versão deste documento:** 1.0.0  
**Data de congelamento:** 2023-10-27  
**Próxima revisão:** Após entrega de segunda-feira (se necessário)
