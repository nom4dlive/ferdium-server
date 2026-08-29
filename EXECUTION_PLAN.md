# 🚀 MASTER PLAN: Execução Automática até Segunda-Feira

**Objetivo:** Transformar o Ferdium Server em uma plataforma universalmente integrável, com personalização total via ENV e SDKs oficiais.
**Prazo:** Domingo 23:59 (Entrega Final).
**Executores:** Agentes de Código (OpenCode, Antigravity).
**Metodologia:** TDD Estrito + Contratos de Dados + Prompts Modulares.

---

## ⚠️ Pontos de Atenção Críticos (Gargalos Potenciais)

Para garantir que a execução seja automática e sem retrabalho, os seguintes pontos foram mapeados e blindados nas especificações:

### 1. Conflito de Dependências
*   **Risco:** Instalação de pacotes incompatíveis (ex: `pg` vs `postgres.js`, versões do Express).
*   **Mitigação:** Arquivo `package.json` congelado com versões exatas no início da Fase 0.
*   **Ação:** Rodar `npm ci` em vez de `npm install`.

### 2. Variáveis de Ambiente Não Definidas
*   **Risco:** O servidor falha ao iniciar por falta de chaves críticas (ex: `JWT_SECRET`, `DB_URL`).
*   **Mitigação:** Arquivo `.env.example` completo gerado na Fase 0 com validação de schema no bootstrap (`config/validator.ts`).
*   **Ação:** O servidor não inicia se as variáveis críticas faltarem, com mensagem de erro clara.

### 3. Schema do Banco Desatualizado
*   **Risco:** Migrações falham ou tabelas novas (webhooks, api_keys) não são criadas.
*   **Mitigação:** Scripts de migração idempotentes e sequenciais.
*   **Ação:** Script `npm run migrate:latest` executado automaticamente no `prestart`.

### 4. Webhooks Falhando Silenciosamente
*   **Risco:** Disparos de webhook falham e o sistema ignora, perdendo dados de auditoria.
*   **Mitigação:** Sistema de retry exponencial e fila de falhos (tabela `webhook_logs`).
*   **Ação:** Implementar fila de processamento assíncrono para webhooks (não bloquear a resposta da API).

### 5. Documentação OpenAPI Dessincronizada
*   **Risco:** O Swagger UI mostra rotas que não existem ou parâmetros errados.
*   **Mitigação:** Geração automática do `swagger.json` a partir dos decorators/anotações do código fonte em tempo de build.
*   **Ação:** Integrar `swagger-autogen` ou similar no pipeline de build.

### 6. CORS Bloqueando Integrações Locais
*   **Risco:** SDKs locais não conseguem conectar devido a políticas de origem cruzada.
*   **Mitigação:** Configuração dinâmica de CORS via ENV (`ALLOWED_ORIGINS=*` para dev).
*   **Ação:** Middleware de CORS configurado antes de qualquer outra rota.

### 7. Testes Falsos Positivos
*   **Risco:** Testes passam mas a integração real falha (mocks excessivos).
*   **Mitigação:** Testes de integração reais usando banco de dados efêmero (Docker/Testcontainers).
*   **Ação:** Separar testes unitários (rápidos) de testes de integração (validação de fluxo).

---

## 📂 Estrutura de Arquivos de Especificação

Os agentes devem consumir estritamente estes arquivos. Nenhuma lógica deve ser inventada.

### 1. Contratos de Dados (`specs/data_contracts.ts`)
Define exatamente o formato JSON de entrada/saída.

```typescript
// Exemplo de Contrato Rigoroso
export interface WebhookPayload {
  event: 'user.created' | 'service.updated' | 'token.revoked';
  timestamp: string; // ISO 8601
  data: Record<string, any>;
  metadata: {
    serverId: string;
    version: string;
  };
}

export interface ApiKeyResponse {
  id: string;
  name: string;
  keyPrefix: string; // Apenas os primeiros 8 chars
  createdAt: string;
  expiresAt?: string;
  // NUNCA retornar a chave completa aqui
}
```

### 2. Matriz de Decisão Técnica (`specs/tech_stack.md`)
Define as bibliotecas exatas a serem usadas para evitar "bikeshedding".

| Funcionalidade | Biblioteca Escolhida | Versão Mínima | Justificativa |
|----------------|----------------------|---------------|---------------|
| Validação ENV | `zod` | 1.22.0 | Schema validation tipado |
| Webhooks | `bullmq` + `redis` | 5.0.0 | Filas robustas e retry |
| Docs API | `swagger-jsdoc` + `ui` | Latest | Padrão indústria |
| Auth Extra | `passport` + strategies | Latest | Flexibilidade OAuth/SAML |
| Rate Limit | `express-rate-limit` | 7.0.0 | Simples e eficaz |
| DB Migration | `knex` ou `prisma` | Latest | Controle total de schema |
| Logging | `pino` | 8.0.0 | Performance alta |

### 3. Instruções para Agentes (`AGENT_INSTRUCTIONS.md`)
Prompt modularizado para cada tipo de tarefa.

#### Para Agente de Banco de Dados:
> "Crie migrações SQL idempotentes. Nunca use `DROP TABLE` em produção. Adicione índices em todas as colunas de busca (foreign keys, status). Garanta que `created_at` e `updated_at` sejam automáticos."

#### Para Agente de API:
> "Siga o padrão RESTful estrito. Use códigos HTTP corretos (201 para criação, 204 para sucesso sem conteúdo). Todas as respostas de erro devem seguir o formato `{ error: { code, message, details } }`. Adicione tags Swagger em cada rota."

#### Para Agente de Testes:
> "Escreva o teste ANTES do código (TDD). Use dados fictícios realistas. Garanta que testes de integração limpem o banco após cada execução. Cobertura mínima de 90% nas rotas críticas."

---

## 🗓️ Cronograma de Execução (Sprint de 3 Dias)

### Dia 1: Fundação & Configuração (Hoje)
*   **Foco:** Estrutura, ENV, DB Schema, Validações.
*   **Entregáveis:**
    *   [ ] `.env.example` completo e validador `zod`.
    *   [ ] Migrations: `api_keys`, `webhooks`, `webhook_logs`, `audit_trail`.
    *   [ ] Configuração de CORS dinâmica.
    *   [ ] Health Check avançado (`/healthz`, `/readyz`).
    *   [ ] Setup do Redis (para filas de webhook).

### Dia 2: Core API & Webhooks (Amanhã)
*   **Foco:** Lógica de Negócio, Eventos, Disparos.
*   **Entregáveis:**
    *   [ ] CRUD de API Keys (criar, listar, revogar).
    *   [ ] CRUD de Webhooks (registrar URLs, selecionar eventos).
    *   [ ] Event Bus interno (Emitter de eventos).
    *   [ ] Worker de Webhooks (processamento assíncrono com retry).
    *   [ ] Logs de Auditoria (quem fez o quê e quando).

### Dia 3: Documentação, SDKs & Polimento (Domingo)
*   **Foco:** DX (Developer Experience), Testes Finais, Docs.
*   **Entregáveis:**
    *   [ ] Geração automática Swagger UI (`/docs`).
    *   [ ] SDK JavaScript/TypeScript (publicável no npm).
    *   [ ] Exemplos de integração (Python, cURL, Postman Collection).
    *   [ ] Suite de testes E2E passando.
    *   [ ] Dockerfile otimizado e `docker-compose.yml` completo.

---

## ✅ Critérios de Aceite (Definition of Done)

Para considerar uma tarefa "Pronta", ela deve satisfazer:
1.  **Código:** Segue linting e padrões definidos.
2.  **Testes:** Testes unitários e de integração passando (verde).
3.  **Contrato:** Respeita estritamente os schemas em `data_contracts.ts`.
4.  **Docs:** Endpoint documentado no Swagger com exemplo de request/response.
5.  **ENV:** Funciona com variáveis de ambiente padrão sem hardcode.
6.  **Resiliência:** Trata erros de rede/banco graciosamente (não crasha o servidor).

---

## 🛠️ Comandos de Controle para Agentes

Use estes comandos para orquestrar a construção:

```bash
# 1. Preparação do Ambiente
npm ci
npm run db:migrate:latest
npm run seed:dev # Apenas se necessário

# 2. Desenvolvimento Guiado por Testes
npm run test:watch # Mantém rodando enquanto o agente codifica

# 3. Validação de Contratos
npm run validate:schemas # Verifica se os tipos TS batem com os contratos

# 4. Build e Docs
npm run build
npm run docs:generate # Gera o swagger.json

# 5. Teste Final de Integração
npm run test:e2e # Sobe docker, roda testes, derruba docker
```

## 🚨 Plano de Contingência

Se algum agente travar ou gerar código inválido:
1.  **Reverter:** `git reset --hard HEAD` para o último commit estável.
2.  **Isolar:** Identificar o módulo falho via logs de teste.
3.  **Simplificar:** Remover complexidade (ex: remover Redis e usar memória temporária) apenas para destravar, criando um ticket de débito técnico para restaurar depois da segunda-feira.
4.  **Fallback:** Se o SDK complexo falhar, entregar apenas a especificação OpenAPI e exemplos cURL funcionais.

---

**Status:** Pronto para Início da Execução Automática.
**Próximo Passo:** Acionar Agente Antigravity com o comando de inicialização da Fase 1.
