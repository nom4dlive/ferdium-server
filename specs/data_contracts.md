# 📐 Contratos de Dados e Especificações Técnicas

Este arquivo define **ESTRITAMENTE** os schemas de dados que TODOS os agentes devem seguir.
Violar estes contratos resultará em falha de integração.

---

## 1. Contratos de API (Request/Response)

### 1.1. Criação de API Key
**Endpoint:** `POST /api/v1/keys`

**Request Body:**
```json
{
  "name": "Minha Integração Slack",
  "description": "Chave para envio de notificações",
  "expiresInDays": 30,
  "permissions": ["webhooks.write", "users.read"]
}
```

**Response (201 Created):**
```json
{
  "success": true,
  "data": {
    "id": "key_8f7d6e5c4b3a2910",
    "name": "Minha Integração Slack",
    "key": "ferdium_sk_live_8f7d6e5c4b3a2910abcdef1234567890", 
    "keyPrefix": "ferdium_sk_live_8f7d...",
    "createdAt": "2023-10-27T10:00:00Z",
    "expiresAt": "2023-11-26T10:00:00Z",
    "permissions": ["webhooks.write", "users.read"]
  }
}
```
⚠️ **ATENÇÃO:** O campo `key` completo só é mostrado UMA VEZ na criação. Nunca armazenar em logs.

### 1.2. Listagem de API Keys
**Endpoint:** `GET /api/v1/keys`

**Response (200 OK):**
```json
{
  "success": true,
  "data": [
    {
      "id": "key_8f7d6e5c4b3a2910",
      "name": "Minha Integração Slack",
      "keyPrefix": "ferdium_sk_live_8f7d...",
      "createdAt": "2023-10-27T10:00:00Z",
      "expiresAt": "2023-11-26T10:00:00Z",
      "lastUsedAt": "2023-10-27T12:30:00Z",
      "isActive": true
    }
  ],
  "meta": {
    "total": 1,
    "page": 1,
    "limit": 20
  }
}
```

### 1.3. Registro de Webhook
**Endpoint:** `POST /api/v1/webhooks`

**Request Body:**
```json
{
  "url": "https://meu-app.com/webhooks/ferdium",
  "events": ["user.created", "service.updated"],
  "secret": "whsec_meu_segredo_super_forte",
  "isActive": true,
  "metadata": {
    "source": "slack_integration"
  }
}
```

**Response (201 Created):**
```json
{
  "success": true,
  "data": {
    "id": "wh_9a8b7c6d5e4f3210",
    "url": "https://meu-app.com/webhooks/ferdium",
    "events": ["user.created", "service.updated"],
    "createdAt": "2023-10-27T10:00:00Z",
    "isActive": true
  }
}
```

### 1.4. Payload de Disparo de Webhook
**Formato enviado para a URL registrada:**

```json
{
  "id": "evt_1a2b3c4d5e6f7890",
  "type": "user.created",
  "timestamp": "2023-10-27T10:00:00Z",
  "serverId": "srv_ferdium_001",
  "data": {
    "userId": "usr_123456",
    "email": "novo@usuario.com",
    "name": "Novo Usuário"
  },
  "metadata": {
    "attempt": 1,
    "version": "1.0.0"
  }
}
```

**Headers Obrigatórios no Disparo:**
- `X-Ferdium-Signature`: `sha256=<hash_hmac>`
- `X-Ferdium-Event`: `user.created`
- `X-Ferdium-Delivery-ID`: `del_abc123`
- `Content-Type`: `application/json`

### 1.5. Resposta de Erro Padrão
**Qualquer erro deve seguir este formato:**

```json
{
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "O campo 'url' é obrigatório.",
    "details": [
      {
        "field": "url",
        "message": "Deve ser uma URL válida começando com http ou https"
      }
    ],
    "timestamp": "2023-10-27T10:00:00Z",
    "path": "/api/v1/webhooks"
  }
}
```

**Códigos de Erro Padronizados:**
- `VALIDATION_ERROR` (400)
- `UNAUTHORIZED` (401)
- `FORBIDDEN` (403)
- `NOT_FOUND` (404)
- `CONFLICT` (409)
- `RATE_LIMIT_EXCEEDED` (429)
- `INTERNAL_SERVER_ERROR` (500)

---

## 2. Schema do Banco de Dados (Migrations)

### 2.1. Tabela: `api_keys`
```sql
CREATE TABLE api_keys (
  id VARCHAR(36) PRIMARY KEY DEFAULT gen_random_uuid(),
  name VARCHAR(255) NOT NULL,
  description TEXT,
  key_hash VARCHAR(255) NOT NULL,
  key_prefix VARCHAR(50) NOT NULL,
  user_id VARCHAR(36),
  permissions JSONB DEFAULT '[]',
  expires_at TIMESTAMP WITH TIME ZONE,
  last_used_at TIMESTAMP WITH TIME ZONE,
  is_active BOOLEAN DEFAULT TRUE,
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE INDEX idx_api_keys_prefix ON api_keys(key_prefix);
CREATE INDEX idx_api_keys_user_id ON api_keys(user_id);
CREATE INDEX idx_api_keys_expires_at ON api_keys(expires_at);
```

### 2.2. Tabela: `webhooks`
```sql
CREATE TABLE webhooks (
  id VARCHAR(36) PRIMARY KEY DEFAULT gen_random_uuid(),
  url TEXT NOT NULL,
  secret VARCHAR(255) NOT NULL,
  events JSONB NOT NULL,
  metadata JSONB DEFAULT '{}',
  is_active BOOLEAN DEFAULT TRUE,
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE INDEX idx_webhooks_active ON webhooks(is_active);
```

### 2.3. Tabela: `webhook_logs` (Auditoria de Disparos)
```sql
CREATE TABLE webhook_logs (
  id VARCHAR(36) PRIMARY KEY DEFAULT gen_random_uuid(),
  webhook_id VARCHAR(36) NOT NULL REFERENCES webhooks(id) ON DELETE CASCADE,
  event_type VARCHAR(100) NOT NULL,
  payload JSONB NOT NULL,
  response_status INTEGER,
  response_body TEXT,
  error_message TEXT,
  attempt_number INTEGER DEFAULT 1,
  next_retry_at TIMESTAMP WITH TIME ZONE,
  delivered_at TIMESTAMP WITH TIME ZONE,
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE INDEX idx_webhook_logs_webhook_id ON webhook_logs(webhook_id);
CREATE INDEX idx_webhook_logs_created_at ON webhook_logs(created_at);
CREATE INDEX idx_webhook_logs_next_retry ON webhook_logs(next_retry_at) WHERE delivered_at IS NULL;
```

### 2.4. Tabela: `audit_logs` (Ações Administrativas)
```sql
CREATE TABLE audit_logs (
  id VARCHAR(36) PRIMARY KEY DEFAULT gen_random_uuid(),
  action VARCHAR(100) NOT NULL,
  actor_type VARCHAR(50) NOT NULL,
  actor_id VARCHAR(36),
  target_type VARCHAR(50),
  target_id VARCHAR(36),
  ip_address INET,
  user_agent TEXT,
  metadata JSONB DEFAULT '{}',
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE INDEX idx_audit_logs_actor ON audit_logs(actor_type, actor_id);
CREATE INDEX idx_audit_logs_action ON audit_logs(action);
CREATE INDEX idx_audit_logs_created_at ON audit_logs(created_at);
```

---

## 3. Variáveis de Ambiente (`.env.example`)

```bash
# Server Config
NODE_ENV=production
PORT=3000
HOST=0.0.0.0

# Database
DB_HOST=localhost
DB_PORT=5432
DB_NAME=ferdium_server
DB_USER=ferdium
DB_PASSWORD=changeme
DB_SSL=false

# Redis (Required for Webhooks Queue)
REDIS_HOST=localhost
REDIS_PORT=6379
REDIS_PASSWORD=
REDIS_DB=0

# Security
JWT_SECRET=super_secret_jwt_key_change_this
JWT_EXPIRES_IN=7d
API_KEY_PREFIX=ferdium_sk

# CORS
ALLOWED_ORIGINS=*
CORS_MAX_AGE=86400

# Rate Limiting
RATE_LIMIT_WINDOW_MS=60000
RATE_LIMIT_MAX_REQUESTS=100

# Webhooks
WEBHOOK_MAX_RETRIES=5
WEBHOOK_RETRY_DELAY_MS=5000
WEBHOOK_TIMEOUT_MS=10000

# Logging
LOG_LEVEL=info
LOG_FORMAT=json

# Feature Flags
ENABLE_SWAGGER_UI=true
ENABLE_AUDIT_LOGS=true
ENABLE_RATE_LIMIT=true
```

---

## 4. Regras de Validação (Zod Schemas)

Todos os inputs devem ser validados com estes schemas antes de qualquer processamento.

```typescript
import { z } from 'zod';

// API Key Schema
export const CreateApiKeySchema = z.object({
  name: z.string().min(3).max(255),
  description: z.string().max(1000).optional(),
  expiresInDays: z.number().int().positive().max(365).nullable().optional(),
  permissions: z.array(z.enum(['webhooks.write', 'webhooks.read', 'users.read', 'admin'])).optional()
});

// Webhook Schema
export const CreateWebhookSchema = z.object({
  url: z.string().url().refine(url => url.startsWith('http')),
  events: z.array(z.enum(['user.created', 'user.updated', 'user.deleted', 'service.created', 'service.updated', 'token.revoked'])).min(1),
  secret: z.string().min(16).optional(),
  isActive: z.boolean().default(true),
  metadata: z.record(z.string()).optional()
});

// Response Error Schema
export const ErrorResponseSchema = z.object({
  success: z.literal(false),
  error: z.object({
    code: z.string(),
    message: z.string(),
    details: z.array(z.object({
      field: z.string(),
      message: z.string()
    })).optional(),
    timestamp: z.string().datetime(),
    path: z.string()
  })
});
```

---

## 5. Eventos do Sistema (Event Bus)

Lista definitiva de eventos que podem ser disparados e consumidos por webhooks.

| Evento | Payload Data | Descrição |
|--------|--------------|-----------|
| `user.created` | `{ userId, email, name }` | Novo usuário registrado |
| `user.updated` | `{ userId, changes }` | Dados de usuário alterados |
| `user.deleted` | `{ userId, email }` | Usuário removido |
| `service.created` | `{ serviceId, name, type }` | Novo serviço adicionado |
| `service.updated` | `{ serviceId, changes }` | Serviço modificado |
| `service.deleted` | `{ serviceId, name }` | Serviço removido |
| `token.revoked` | `{ tokenId, reason, userId }` | Token de sessão revogado |
| `api_key.created` | `{ keyId, name, permissions }` | Nova API Key gerada |
| `api_key.revoked` | `{ keyId, reason }` | API Key revogada |
| `webhook.created` | `{ webhookId, url, events }` | Webhook registrado |
| `webhook.failed` | `{ webhookId, url, error, attempt }` | Falha crítica no disparo |

---

## 6. Instruções de Implementação para Agentes

1. **Nunca invente campos:** Se um campo não está neste contrato, não existe.
2. **Validação primeiro:** Sempre valide o input com Zod antes de tocar no banco.
3. **Hash de segredos:** Chaves de API e secrets de webhook devem ser hasheados (SHA-256) antes de salvar.
4. **IDs UUID:** Use `gen_random_uuid()` para todos os IDs primários.
5. **Timestamps ISO:** Todas as datas devem ser ISO 8601 UTC (`2023-10-27T10:00:00Z`).
6. **Paginação:** Todas as listagens devem suportar `?page=1&limit=20` e retornar meta.
7. **Assinatura HMAC:** Webhooks devem usar HMAC-SHA256 para assinar payloads.

---

**Versão do Contrato:** 1.0.0
**Última Atualização:** 2023-10-27
**Status:** CONGELADO PARA IMPLEMENTAÇÃO
