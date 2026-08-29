# 🤖 Instruções Modulares para Agentes de Código

Este documento contém prompts específicos para cada tipo de agente (OpenCode, Antigravity) que irá executar as tarefas.
Copie e cole o bloco relevante para cada agente conforme a fase de implementação.

---

## 📋 Prompt Global de Contexto (Incluir em todas as requisições)

```
Você é um agente de código especializado em desenvolvimento backend Node.js/TypeScript de alta qualidade.
Seu objetivo é implementar funcionalidades seguindo ESTRITAMENTE os contratos definidos em /workspace/specs/data_contracts.md.

REGRAS GERAIS:
1. NUNCA invente campos, endpoints ou comportamentos não especificados nos contratos.
2. SEMPRE escreva testes ANTES do código de produção (TDD).
3. USE apenas as bibliotecas definidas em /workspace/specs/tech_stack.md.
4. VALIDAÇÃO é obrigatória em todos os inputs usando Zod.
5. TRATE erros graciosamente - nunca deixe o servidor crashar.
6. DOCUMENTE cada endpoint com annotations Swagger/JSDoc.
7. SIGA o padrão RESTful estrito com códigos HTTP corretos.
8. LOGUE ações importantes no formato JSON estruturado.

CONTRATO DE DADOS:
- Consulte /workspace/specs/data_contracts.md para schemas exatos de request/response.
- Consulte /workspace/specs/data_contracts.md para SQL migrations exatas.
- Qualquer desvio do contrato será considerado falha crítica.

TESTES:
- Cobertura mínima de 90% nas rotas críticas.
- Use dados fictícios realistas.
- Limpe o banco após cada teste de integração.
- Testes devem rodar em ambiente isolado (Docker/Testcontainers).

ENTREGÁVEIS POR TAREFA:
1. Migration SQL (se aplicável)
2. Schema Zod de validação
3. Controller/Service com lógica de negócio
4. Rotas API com documentação Swagger
5. Testes unitários e de integração
6. Atualização do .env.example (se novas variáveis)
```

---

## 🗄️ Prompt para Agente de Banco de Dados

```
TAREFA: Implementar Migrations do Banco de Dados

CONTEXTO:
Precisamos criar as tabelas necessárias para suportar API Keys, Webhooks e Audit Logs.

INSTRUÇÕES ESPECÍFICAS:
1. Crie arquivos de migration SQL na pasta /database/migrations/
2. Use migrações IDEMPOTENTES (pode rodar múltiplas vezes sem erro)
3. NUNCA use DROP TABLE ou DELETE em produções de migration
4. Adicione ÍNDICES em todas as foreign keys e colunas de busca
5. Garanta que created_at e updated_at sejam automáticos (DEFAULT NOW())
6. Use VARCHAR(36) com gen_random_uuid() para IDs primários
7. Use JSONB para campos flexíveis (permissions, events, metadata)
8. Adicione comentários nas colunas para documentação

TABELAS A CRIAR (na ordem):
1. api_keys - Ver schema exato em /workspace/specs/data_contracts.md seção 2.1
2. webhooks - Ver schema exato em /workspace/specs/data_contracts.md seção 2.2
3. webhook_logs - Ver schema exato em /workspace/specs/data_contracts.md seção 2.3
4. audit_logs - Ver schema exato em /workspace/specs/data_contracts.md seção 2.4

CRITÉRIOS DE ACEITE:
- [ ] Todas as migrations rodam sem erro com `npm run db:migrate`
- [ ] Índices criados conforme especificação
- [ ] Foreign keys com ON DELETE CASCADE apropriado
- [ ] Teste de rollback funciona (down migration)
- [ ] Arquivo de seed criado para desenvolvimento (opcional)

COMANDO DE TESTE:
npm run db:migrate && npm run db:migrate:undo && npm run db:migrate

SAÍDA ESPERADA:
Arquivos SQL em /database/migrations/YYYYMMDDHHMMSS_create_<table>.sql
```

---

## 🔌 Prompt para Agente de API/Rotas

```
TAREFA: Implementar Endpoints da API REST

CONTEXTO:
Precisamos criar endpoints RESTful para gerenciamento de API Keys e Webhooks.

INSTRUÇÕES ESPECÍFICAS:
1. Siga estrutura de pastas: /src/routes/, /src/controllers/, /src/services/
2. Valide TODOS os inputs com Zod schemas antes de processar
3. Use códigos HTTP corretos:
   - 200: Sucesso com dados
   - 201: Recurso criado (retornar Location header)
   - 204: Sucesso sem conteúdo (DELETE bem sucedido)
   - 400: Validation Error
   - 401: Não autenticado
   - 403: Sem permissão
   - 404: Recurso não encontrado
   - 409: Conflito (ex: nome duplicado)
   - 429: Rate limit excedido
   - 500: Erro interno
4. Formato de erro PADRÃO: { success: false, error: { code, message, details, timestamp, path } }
5. Formato de sucesso PADRÃO: { success: true, data: {...}, meta: {...} }
6. Adicione tags Swagger em cada rota (@tags, @summary, @response)
7. Middleware de autenticação deve verificar API Key no header Authorization
8. Paginação padrão: ?page=1&limit=20, retornar meta { total, page, limit, totalPages }

ENDPOINTS A CRIAR:

### API Keys
POST   /api/v1/keys           - Criar nova API Key
GET    /api/v1/keys           - Listar API Keys (paginado)
GET    /api/v1/keys/:id       - Obter detalhes de uma Key
DELETE /api/v1/keys/:id       - Revogar/Deletar uma Key
POST   /api/v1/keys/:id/revoke - Revogar explicitamente

### Webhooks
POST   /api/v1/webhooks       - Registrar novo webhook
GET    /api/v1/webhooks       - Listar webhooks (paginado)
GET    /api/v1/webhooks/:id   - Obter detalhes de um webhook
PUT    /api/v1/webhooks/:id   - Atualizar webhook
DELETE /api/v1/webhooks/:id   - Remover webhook
POST   /api/v1/webhooks/:id/test - Disparar evento de teste

### Health & Status
GET    /healthz               - Health check básico (sem auth)
GET    /readyz                - Ready check (verifica DB, Redis)
GET    /api/v1/info           - Informações do servidor (versão, features)

REFERÊNCIAS:
- Contratos de Request/Response: /workspace/specs/data_contracts.md seções 1.1 a 1.5
- Variáveis de Ambiente: /workspace/specs/data_contracts.md seção 3
- Eventos Disponíveis: /workspace/specs/data_contracts.md seção 5

CRITÉRIOS DE ACEITE:
- [ ] Todos os endpoints respondem conforme contrato
- [ ] Validação Zod retorna erros padronizados
- [ ] Swagger UI mostra todos os endpoints corretamente
- [ ] Testes de integração cobrem happy path e error paths
- [ ] Rate limiting aplicado conforme ENV
- [ ] CORS configurado dinamicamente

COMANDO DE TESTE:
npm run test:routes

SAÍDA ESPERADA:
- Controllers em /src/controllers/
- Services em /src/services/
- Routes em /src/routes/
- Schemas em /src/schemas/
```

---

## 🎣 Prompt para Agente de Webhooks/Eventos

```
TAREFA: Implementar Sistema de Webhooks Assíncrono

CONTEXTO:
Precisamos de um sistema robusto de disparo de webhooks que não bloqueie a API principal.

INSTRUÇÕES ESPECÍFICAS:
1. Use BullMQ + Redis para fila de processamento
2. Implemente retry exponencial (5 tentativas máximas)
3. Logue TODAS as tentativas na tabela webhook_logs
4. Assine payloads com HMAC-SHA256 usando o secret do webhook
5. Timeout de 10 segundos por disparo (configurável via ENV)
6. NÃO bloqueie a resposta da API - enqueue e retorne 202 Accepted
7. Headers obrigatórios no disparo:
   - X-Ferdium-Signature: sha256=<hash>
   - X-Ferdium-Event: <event_name>
   - X-Ferdium-Delivery-ID: <delivery_id>
   - Content-Type: application/json
8. Worker separado para processar fila (arquivo worker.ts)

FLUXO DE IMPLEMENTAÇÃO:
1. Event Bus (Emitter) - Centraliza disparo de eventos
2. Queue Producer - Adiciona jobs na fila do Redis
3. Queue Consumer (Worker) - Processa jobs e faz HTTP request
4. Retry Logic - Reagenda falhas com backoff exponencial
5. Logging - Salva resultado em webhook_logs

EVENTOS PARA SUPORTAR:
- user.created, user.updated, user.deleted
- service.created, service.updated, service.deleted
- token.revoked
- api_key.created, api_key.revoked
- webhook.created, webhook.failed

PAYLOAD FORMAT:
Ver contrato exato em /workspace/specs/data_contracts.md seção 1.4

CRITÉRIOS DE ACEITE:
- [ ] Evento é enfileirado em < 10ms
- [ ] Worker processa jobs concorrentemente
- [ ] Retry funciona com delay exponencial (5s, 10s, 20s, 40s, 80s)
- [ ] Assinatura HMAC é válida (testar com webhook.site)
- [ ] Logs são gravados para sucesso e falha
- [ ] Fila não perde jobs em restart do Redis (persistência)

COMANDO DE TESTE:
npm run worker -- --queue=webhooks
npm run test:webhooks

SAÍDA ESPERADA:
- Event Bus em /src/events/EventBus.ts
- Queue config em /src/queues/webhookQueue.ts
- Worker em /src/workers/webhookWorker.ts
- Service de disparo em /src/services/WebhookService.ts
```

---

## 🧪 Prompt para Agente de Testes

```
TAREFA: Escrever Suite de Testes Completa (TDD)

CONTEXTO:
Precisamos garantir 90%+ de cobertura de código com testes confiáveis.

INSTRUÇÕES ESPECÍFICAS:
1. Escreva o teste ANTES do código de produção
2. Separe testes unitários (rápidos, mocks) de integração (lentos, DB real)
3. Use Jest como runner de testes
4. Para testes de integração:
   - Suba container Docker efêmero com DB e Redis
   - Rode migrations antes dos testes
   - Limpe banco após CADA teste (truncate cascade)
   - Derrube containers após suite
5. Mocks devem ser realistas (use factories, não hardcode)
6. Teste happy path E todos os error paths
7. Afirme não só o status code, mas também o formato da resposta
8. Teste concorrência (ex: duas requisições simultâneas)

ESTRUTURA DE PASTAS:
/tests
  /unit          - Testes unitários rápidos
  /integration   - Testes com DB/Redis reais
  /e2e          - Testes de fluxo completo
  /factories    - Factories para dados fictícios
  /helpers      - Helpers de teste (setup, teardown)

SUÍTES A CRIAR:

### Unit Tests
- validators.test.ts - Validação Zod dos schemas
- utils.test.ts - Funções utilitárias (hash, signature)
- eventBus.test.ts - Emissor de eventos (mocked)

### Integration Tests
- apiKeys.test.ts - CRUD completo de API Keys
- webhooks.test.ts - CRUD e disparos de Webhooks
- auth.test.ts - Autenticação com API Key
- rateLimit.test.ts - Limitação de requisições

### E2E Tests
- flow.test.ts - Fluxo completo: criar key -> registrar webhook -> trigger event -> verificar log

FACTORIES NECESSÁRIAS:
- createApiKey() - Gera API Key válida
- createWebhook() - Gera Webhook registrado
- createUser() - Gera usuário fictício
- generateSignature() - Gera HMAC válido

CRITÉRIOS DE ACEITE:
- [ ] Todos testes passam (verde)
- [ ] Cobertura >= 90% nas rotas críticas
- [ ] Nenhum teste é flaky (intermitente)
- [ ] Tempo total da suite < 5 minutos
- [ ] Testes rodam em CI/CD sem configuração manual

COMANDOS:
npm run test           - Roda tudo
npm run test:unit      - Só unitários
npm run test:integration - Só integração (sobe Docker)
npm run test:coverage  - Gera relatório de cobertura

SAÍDA ESPERADA:
Arquivos .test.ts em /tests/ com descrições claras (describe/it)
```

---

## 📚 Prompt para Agente de Documentação

```
TAREFA: Gerar Documentação Automática e Exemplos

CONTEXTO:
Precisamos de documentação viva que sincronize com o código.

INSTRUÇÕES ESPECÍFICAS:
1. Configure swagger-autogen para ler JSDoc das rotas
2. Gere swagger.json automaticamente no build
3. Habilite Swagger UI em /docs (feature flag ENABLE_SWAGGER_UI)
4. Crie exemplos de uso em múltiplas linguagens:
   - cURL
   - JavaScript (fetch)
   - Python (requests)
   - Postman Collection
5. Documente todos os códigos de erro possíveis
6. Inclua exemplos de payload de webhook

ARQUIVOS A GERAR:
- /docs/swagger.json - Auto-gerado
- /docs/examples/curl.md - Exemplos cURL
- /docs/examples/javascript.md - Exemplos JS/TS
- /docs/examples/python.md - Exemplos Python
- /docs/postman_collection.json - Importável no Postman
- README.md atualizado com quickstart

CONTEÚDO DO README:
- Badges (version, tests, coverage)
- Quick Start (3 passos para rodar)
- Variáveis de Ambiente (tabela completa)
- Exemplo de uso básico
- Link para Swagger UI
- Link para Integration Guide

CRITÉRIOS DE ACEITE:
- [ ] Swagger UI acessível em /docs
- [ ] Todos endpoints documentados com exemplos
- [ ] Exemplos de código funcionam (copiar e colar)
- [ ] Postman Collection importável e funcional
- [ ] README claro para novos desenvolvedores

COMANDO:
npm run docs:generate
npm run docs:serve

SAÍDA ESPERADA:
Documentação viva e exemplos prontos para uso
```

---

## 🐳 Prompt para Agente de DevOps/Docker

```
TAREFA: Configurar Ambiente Docker e CI/CD

CONTEXTO:
Precisamos de containers otimizados e pipeline de deploy.

INSTRUÇÕES ESPECÍFICAS:
1. Dockerfile multi-stage (build + runtime)
2. docker-compose.yml para desenvolvimento completo
3. .dockerignore otimizado
4. Health checks no container
5. Volumes persistentes para DB e Redis (dev)
6. Scripts de entrypoint para migrations automáticas

ARQUIVOS A CRIAR:
- Dockerfile (produção)
- Dockerfile.dev (desenvolvimento)
- docker-compose.yml (stack completa)
- docker-compose.test.yml (para testes)
- .github/workflows/ci.yml (GitHub Actions)

STACK DOCKER COMPOSE:
- ferdium-server (app)
- postgres (DB)
- redis (filas)
- pgadmin (opcional, dev only)

CI/CD PIPELINE:
1. Lint (eslint)
2. Type Check (tsc)
3. Test (jest)
4. Build (tsc)
5. Docker Build & Push (se main)

CRITÉRIOS DE ACEITE:
- [ ] docker-compose up roda sem erro
- [ ] Health check passa em < 30s
- [ ] Imagem produção < 200MB
- [ ] CI pipeline verde
- [ ] Migrations rodam automaticamente no start

COMANDOS:
docker-compose up -d
docker-compose run test
docker build -t ferdium-server .

SAÍDA ESPERADA:
Containers funcionando e pipeline automatizado
```

---

## ✅ Checklist de Validação Final (Para Todos Agentes)

Antes de considerar uma tarefa completa, verifique:

- [ ] O código segue os contratos em `/workspace/specs/data_contracts.md`?
- [ ] Os testes estão passando localmente?
- [ ] A documentação Swagger foi atualizada?
- [ ] As variáveis de ambiente foram adicionadas ao `.env.example`?
- [ ] O tratamento de erros segue o padrão `{ success: false, error: {...} }`?
- [ ] Não há hardcoded de valores (tudo via ENV)?
- [ ] Logs estão em formato JSON estruturado?
- [ ] Não há segredos expostos no código ou logs?

Se qualquer item acima for "NÃO", corrija antes de submeter.
