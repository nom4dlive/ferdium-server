# 🧪 Dados de Teste Determinísticos e Fixtures

**Objetivo:** Garantir que testes rodem de forma idêntica em qualquer ambiente (local, CI, agentes IA) sem dependência de estado externo ou dados aleatórios.

---

## 1. Seeds de Banco de Dados (Fixtures)

Todos os testes de integração devem partir de um estado conhecido do banco.

### Script de Seed Padrão (`tests/fixtures/seed.sql`)
```sql
-- Limpeza prévia
TRUNCATE TABLE users, workspaces, services, api_keys CASCADE;

-- Usuários Fixos
INSERT INTO users (id, email, password_hash, name, role, created_at) VALUES
('usr_001', 'admin@example.com', '$argon2id$v=19$m=65536...', 'Admin User', 'admin', NOW()),
('usr_002', 'user@example.com', '$argon2id$v=19$m=65536...', 'Normal User', 'user', NOW()),
('usr_003', 'banned@example.com', '$argon2id$v=19$m=65536...', 'Banned User', 'user', NOW());

-- Workspaces
INSERT INTO workspaces (id, name, owner_id, slug, created_at) VALUES
('ws_001', 'Default Workspace', 'usr_001', 'default', NOW()),
('ws_002', 'Test Workspace', 'usr_002', 'test-ws', NOW());

-- Serviços Ativos
INSERT INTO services (id, workspace_id, name, type, is_active, settings) VALUES
('svc_001', 'ws_001', 'Slack Bot', 'slack', true, '{"channel": "#general"}'),
('svc_002', 'ws_001', 'GitHub Sync', 'github', false, '{"repo": "org/repo"}');

-- API Keys
INSERT INTO api_keys (id, user_id, name, key_hash, permissions, expires_at) VALUES
('key_001', 'usr_001', 'Integration Key', 'hash_abc123', 'read:services,write:webhooks', NOW() + INTERVAL '30 days');
```

### Regras para Seeds
- **IDs Fixos:** Usar UUIDs ou IDs legíveis (`usr_001`) em vez de `gen_random_uuid()` para facilitar asserções.
- **Senhas Conhecidas:** Hash de senhas fixas (ex: `password123`) pré-calculado.
- **Datas Relativas:** Usar `NOW()` no SQL, mas assumir tempo relativo nos testes.
- **Isolamento:** Cada suite de teste deve rodar o seed em transação e fazer rollback ao final.

---

## 2. Mocks de APIs Externas (WireMock / MSW)

Para testes unitários e de integração, **NUNCA** chamar APIs reais (Slack, Google, Stripe).

### Estratégia
Usar **Mock Service Worker (MSW)** para interceptar requisições HTTP em Node.js.

### Exemplo de Handler (`tests/mocks/handlers/slack.ts`)
```typescript
import { http, HttpResponse } from 'msw';

export const slackHandlers = [
  // Simula envio de mensagem com sucesso
  http.post('https://slack.com/api/chat.postMessage', async ({ request }) => {
    const body = await request.json();
    
    // Validação estrita do payload esperado
    if (!body.channel || !body.text) {
      return HttpResponse.json({ error: 'invalid_payload' }, { status: 400 });
    }

    return HttpResponse.json({
      ok: true,
      ts: '1234567890.123456',
      channel: body.channel
    });
  }),

  // Simula erro de rate limit (para testar retry/circuit breaker)
  http.post('https://slack.com/api/chat.postMessage', () => {
    return HttpResponse.json({ error: 'rate_limited' }, { status: 429 });
  }, { times: 1 }) // Apenas na primeira chamada
];
```

### Cenários Obrigatórios de Mock
1. **Sucesso (200):** Resposta padrão válida.
2. **Erro de Cliente (400/401):** Token inválido ou payload errado.
3. **Erro de Servidor (500/503):** Simular indisponibilidade.
4. **Timeout:** Demorar > 5s para responder (testar timeout da aplicação).
5. **Rate Limit (429):** Testar lógica de backoff.

---

## 3. Variáveis de Ambiente de Teste (.env.test)

Arquivo `.env.test` **obrigatório** na raiz, nunca usar `.env` local nos testes.

```bash
# Database (Container Docker isolado)
DATABASE_URL=postgresql://test_user:test_pass@localhost:5432/ferdium_test?schema=public

# Redis (Para rate limiting e cache em testes)
REDIS_URL=redis://localhost:6379/1

# JWT Secrets (Fixos para testes)
JWT_SECRET=test_secret_key_do_not_use_in_prod
JWT_EXPIRATION=15m

# External APIs (URLs dos mocks)
SLACK_API_BASE_URL=http://localhost:3001/__msw_slack__
GOOGLE_API_BASE_URL=http://localhost:3001/__msw_google__

# Feature Flags
ENABLE_WEBHOOKS=true
ENABLE_OAUTH=false # Desativado em testes unitários rápidos
LOG_LEVEL=silent # Não poluir output do teste
```

---

## 4. Fábricas de Dados (Data Factories)

Para gerar variações de dados sem repetir código de setup.

### Implementação (`tests/factories/user.factory.ts`)
```typescript
import { faker } from '@faker-js/faker';

export function createUser(overrides = {}) {
  return {
    id: `usr_${faker.string.uuid()}`,
    email: faker.internet.email(),
    password: 'password123',
    name: faker.person.fullName(),
    role: 'user',
    ...overrides, // Permite sobrescrever campos específicos
  };
}

export function createAdmin() {
  return createUser({ role: 'admin', email: 'admin@test.com' });
}

export function createBannedUser() {
  return createUser({ role: 'banned' });
}
```

---

## 5. Estado Global de Teste

Gerenciar estado compartilhado entre testes para evitar poluição cruzada.

### Setup Global (`tests/setup/global.ts`)
```typescript
import { db } from '../../src/database';
import { seedDatabase } from './seed';

export async function setupTestEnvironment() {
  // 1. Iniciar transação
  await db.transaction(async (trx) => {
    // 2. Rodar Seed
    await seedDatabase(trx);
    
    // 3. Retornar contexto para o teste
    return { trx };
  });
}

export async function teardownTestEnvironment() {
  // 1. Rollback da transação (limpa tudo)
  // Ou TRUNCATE tables se não usar transação por teste
}
```

---

## 6. Checklist de Validação de Testes

Antes de considerar um teste "pronto", verificar:

- [ ] **Determinístico:** Roda 10 vezes seguidas com mesmo resultado?
- [ ] **Isolado:** Não depende de ordem de execução ou dados de outro teste?
- [ ] **Rápido:** Completa em < 500ms (unitário) ou < 5s (integração)?
- [ ] **Sem Side Effects:** Não envia emails reais, não cobra cartão, não posta no Slack real?
- [ ] **Cobre Edge Cases:** Testa null, undefined, arrays vazios, strings gigantes?
- [ ] **Falha Corretamente:** O teste falha se a lógica estiver errada? (Evitar falsos positivos)

---

## 7. Comandos de Execução

```bash
# Rodar todos os testes (Unitários + Integração)
npm run test

# Rodar apenas testes unitários (rápido, mocks)
npm run test:unit

# Rodar testes de integração (sobe Docker, seed DB)
npm run test:integration

# Rodar testes com coverage mínimo exigido (80%)
npm run test:coverage

# Rodar testes em modo watch (desenvolvimento)
npm run test:watch
```

---

## 8. Tratamento de Falhas em CI

Se um teste falhar no pipeline:
1. **Logs Detalhados:** Imprimir corpo da requisição e resposta do erro.
2. **Screenshot (se E2E):** Salvar screenshot do estado da tela.
3. **Estado do DB:** Exportar dump do banco no momento da falha para debug.
4. **Não Retry Automático:** Se falhou, falhou. Retry mascara problemas intermitentes reais.
