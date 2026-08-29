# 🏛️ Arquitetural Decision Records (ADRs)

**Objetivo:** Eliminar ambiguidades arquiteturais. Os agentes de IA **NÃO** devem decidir padrões; devem apenas implementar o que está definido aqui. Qualquer desvio é considerado erro.

---

## ADR-001: Padrão de Injeção de Dependência
**Status:** Aprovado | **Impacto:** Alto

### Decisão
Utilizar **Injeção de Dependência Explícita via Construtor** para todas as classes de serviço, repositório e controller.
- **Proibido:** Singleton global (`global.db`), importação direta de instâncias em arquivos de lógica de negócio.
- **Obrigatório:** Passar dependências como argumentos no construtor.

### Exemplo Correto
```typescript
// ✅ CORRETO
class UserService {
  constructor(private readonly db: Database, private readonly logger: Logger) {}
}
const userService = new UserService(databaseInstance, loggerInstance);
```

### Exemplo Proibido
```typescript
// ❌ PROIBIDO - Gera acoplamento forte e dificulta testes
import { db } from '../database'; 
class UserService {
  async getUser(id) { return db.query(...); }
}
```

---

## ADR-002: Estratégia de Tratamento de Erros
**Status:** Aprovado | **Impacto:** Crítico

### Decisão
Adotar padrão **"Fail-Fast" com Error Objects Tipados**.
1. **Nunca** usar `try/catch` genérico sem logar o erro específico.
2. **Nunca** retornar status 200 com erro na body (`{ success: false, error: ... }`). Usar códigos HTTP apropriados (400, 401, 403, 404, 500).
3. Criar classe base `AppError` estendendo `Error`.

### Hierarquia de Erros
- `BadRequestError` (400)
- `UnauthorizedError` (401)
- `ForbiddenError` (403)
- `NotFoundError` (404)
- `ConflictError` (409)
- `InternalServerError` (500)

### Middleware de Erros
Todos os erros não tratados devem passar por um middleware central que:
1. Loga o stack trace (em dev) ou mensagem genérica (em prod).
2. Retorna JSON padronizado: `{ error: { code: "ERR_CODE", message: "Human readable", details: {} } }`.

---

## ADR-003: Estrutura de Camadas (Layered Architecture)
**Status:** Aprovado | **Impacto:** Alto

### Decisão
Seguir estritamente a separação: **Controller → Service → Repository → Database**.

| Camada | Responsabilidade | Pode importar | Não pode importar |
|--------|------------------|---------------|-------------------|
| **Controller** | Receber HTTP, validar input (Zod), chamar Service | DTOs, Services | Repositories, DB |
| **Service** | Regra de negócio, transações, orquestração | Repositories, Utils | Controllers, HTTP |
| **Repository** | Queries SQL/NoSQL, mapeamento de dados | DB Client, Entities | Services, Business Logic |
| **Database** | Conexão, Migrations, Seeds | - | Tudo acima |

**Regra de Ouro:** Dependências apontam apenas para baixo. Services não sabem que existem HTTP requests.

---

## ADR-004: Política de Logs e Observabilidade
**Status:** Aprovado | **Impacto:** Médio

### Decisão
Usar biblioteca `pino` (performance) com estrutura JSON.
- **Nível de Log:** `info` (produção), `debug` (dev).
- **Contexto Obrigatório:** Todo log deve incluir `requestId`, `userId` (se auth), e `timestamp`.
- **Dados Sensíveis:** **NUNCA** logar senhas, tokens completos, PII (CPF, Email) sem máscara.

### Formato do Log
```json
{
  "level": "error",
  "time": 1715623423123,
  "pid": 1234,
  "hostname": "server-01",
  "reqId": "abc-123",
  "msg": "Falha ao criar usuário",
  "err": { "type": "ConflictError", "message": "Email já existe" },
  "context": { "email": "u***@example.com" }
}
```

---

## ADR-005: Validação de Dados (Input/Output)
**Status:** Aprovado | **Impacto:** Crítico

### Decisão
Usar **Zod** para toda validação de entrada e saída.
1. **Request:** Validar `req.body`, `req.query`, `req.params` antes de entrar no Service.
2. **Response:** Validar dados retornados pelo Service antes de enviar ao cliente (garante contrato da API).
3. **Environment:** Validar variáveis de ambiente na inicialização do app. Se faltar algo crítico, o app **não sobe**.

---

## ADR-006: Gerenciamento de Estado e Cache
**Status:** Aprovado | **Impacto:** Médio

### Decisão
- **Estado da Aplicação:** Stateless. Nenhuma variável global mutável compartilhada entre requests.
- **Cache:** Usar Redis apenas para:
  - Sessões (se não usar JWT stateless)
  - Rate Limiting
  - Resultados de queries pesadas (TTL máximo 5min)
- **Proibido:** Cache em memória (variáveis globais) para dados de negócio, pois quebra em escalabilidade horizontal.

---

## ADR-007: Segurança de Banco de Dados
**Status:** Aprovado | **Impacto:** Crítico

### Decisão
1. **ORM vs Query Builder:** Usar **Kysely** ou **Prisma** (definido em tech_stack.md). Raw SQL apenas para migrações complexas.
2. **Prepared Statements:** Obrigatório para prevenir SQL Injection. Nunca concatenar strings em queries.
3. **Migrações:** Todas as mudanças de schema devem ser arquivos de migração versionados. **Nunca** alterar schema manualmente no banco de produção.
4. **Soft Delete:** Tabelas principais (`users`, `workspaces`) devem ter coluna `deleted_at`. Deletes físicos são proibidos via API.

---

## ADR-008: Testes e Qualidade de Código
**Status:** Aprovado | **Impacto:** Alto

### Decisão
1. **TDD Obrigatório:** O teste deve existir e falhar antes da implementação.
2. **Cobertura Mínima:** 80% de linhas, 100% de caminhos críticos (auth, payments).
3. **Mocks:** Mockar todas as dependências externas (APIs de terceiros, DB) nos testes de unidade. Testes de integração sobem container Docker real.
4. **Linting:** ESLint + Prettier com regras estritas (sem `any`, sem `console.log` em produção).

---

## ADR-009: Versionamento de API
**Status:** Aprovado | **Impacto:** Médio

### Decisão
- **URL Versioning:** `/api/v1/resource`.
- **Depreciação:** Versões antigas mantidas por 6 meses após lançamento de nova versão major.
- **Breaking Changes:** Proibidos em versões minors/patches. Apenas em `v2`, `v3`, etc.

---

## ADR-010: Lidando com Integrações Externas
**Status:** Aprovado | **Impacto:** Alto

### Decisão
1. **Adapter Pattern:** Cada serviço externo (Slack, Google, Stripe) deve ter uma interface comum e uma classe adaptadora.
2. **Timeouts:** Toda chamada externa deve ter timeout explícito (padrão 5s).
3. **Circuit Breaker:** Implementar lógica de "abrir circuito" se um serviço externo falhar > 3 vezes consecutivas em 1 minuto.
4. **Fallbacks:** Se possível, fornecer resposta degradada em vez de erro 500 quando integração falhar.
