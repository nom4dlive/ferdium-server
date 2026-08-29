# 🛑 Estratégia de Fail-Fast e Rollback

**Objetivo:** Minimizar danos e tempo de inatividade em caso de falhas. O sistema deve falhar de forma segura, informativa e reversível.

---

## 1. Princípio Fail-Fast

**Definição:** Detectar e reportar erros o mais cedo possível, impedindo que o sistema continue em estado inconsistente.

### Aplicações Práticas

#### A. Inicialização da Aplicação
O servidor **NÃO** deve iniciar se:
- Variáveis de ambiente críticas faltarem ou forem inválidas.
- Conexão com banco de dados falhar.
- Conexão com Redis falhar (se requerido para rate limiting).
- Migrações pendentes existirem (em produção).
- Chaves de criptografia/segredo não forem carregadas.

```typescript
// Exemplo de validação no bootstrap
async function bootstrap() {
  const envValidation = validateEnv(process.env);
  if (!envValidation.valid) {
    logger.error('Environment validation failed', envValidation.errors);
    process.exit(1); // Exit code 1 indica erro crítico
  }

  try {
    await db.connect();
  } catch (error) {
    logger.error('Database connection failed', error);
    process.exit(1);
  }

  // Só iniciar servidor se tudo estiver OK
  app.listen(PORT);
}
```

#### B. Validação de Requests
Rejeitar requests malformados imediatamente (400 Bad Request) antes de processar qualquer lógica.
- Schema inválido.
- Headers obrigatórios faltando.
- Token de autenticação expirado/inválido.
- Rate limit excedido (429 Too Many Requests).

#### C. Transações de Banco de Dados
Se qualquer passo de uma transação falhar, fazer rollback automático de TODAS as operações.
```typescript
await db.transaction(async (trx) => {
  await trx.insert('users').values(newUser);
  await trx.insert('audit_logs').values(logEntry); // Se falhar aqui, usuário NÃO é criado
});
```

---

## 2. Health Checks Estratificados

Implementar 3 níveis de health check para monitoramento preciso.

### A. Health Check Básico (`/health/live`)
Verifica apenas se o processo está rodando.
- **Uso:** Kubernetes Liveness Probe (reiniciar container se falhar).
- **Resposta:** `200 OK` `{ "status": "alive" }`

### B. Health Check de Prontidão (`/health/ready`)
Verifica dependências críticas.
- **Checa:** Conexão DB, Redis, serviços externos essenciais.
- **Uso:** Kubernetes Readiness Probe (não enviar tráfego se falhar).
- **Resposta:** 
  - `200 OK` se tudo OK.
  - `503 Service Unavailable` se alguma dependência falhar.

```json
{
  "status": "unhealthy",
  "checks": {
    "database": "up",
    "redis": "down", // Causa da falha
    "external_api": "up"
  },
  "timestamp": "2024-01-15T10:30:00Z"
}
```

### C. Health Check Profundo (`/health/detailed`)
Inclui métricas detalhadas para debugging (apenas interno/admin).
- Latência do DB.
- Uso de memória/CPU.
- Tamanho das filas.
- Status de circuit breakers.

---

## 3. Estratégia de Rollback

### A. Migrações de Banco de Dados Reversíveis
Toda migração DEVE ter função `up()` e `down()`.

```typescript
// migration_001_add_users_table.ts
export async function up(db: Database) {
  await db.createTable('users')...
}

export async function down(db: Database) {
  await db.dropTable('users'); // Deve reverter exatamente
}
```

**Regra de Ouro:** Testar `down()` em staging antes de rodar `up()` em produção.

### B. Versionamento de Deploy
Manter pelo menos 3 versões anteriores disponíveis para rollback imediato.
- **Docker Tags:** `latest`, `v1.2.3`, `v1.2.2`, `v1.2.1`.
- **Kubernetes:** Usar `kubectl rollout undo deployment/ferdium-server`.

### C. Feature Flags para Rollback Rápido
Para funcionalidades críticas, usar feature flags em vez de deploy reverso.
```typescript
if (featureFlags.isEnabled('new_webhook_system')) {
  await newWebhookService.send(data);
} else {
  await legacyWebhookService.send(data); // Fallback seguro
}
```
**Vantagem:** Desativar feature em segundos sem precisar de novo deploy.

---

## 4. Circuit Breaker Pattern

Prevenir cascata de falhas quando um serviço externo está indisponível.

### Estados do Circuit Breaker
1. **Closed (Fechado):** Operação normal. Se falhar > N vezes em T tempo, abre circuito.
2. **Open (Aberto):** Falhas imediatas sem chamar serviço externo. Após T tempo, vai para Half-Open.
3. **Half-Open (Meio Aberto):** Permite 1 request de teste. Se sucesso → Closed. Se falha → Open.

### Implementação
```typescript
const slackCircuitBreaker = new CircuitBreaker({
  failureThreshold: 3,     // Abre após 3 falhas
  resetTimeout: 60000,     // Tenta novamente após 1 minuto
  halfOpenMaxCalls: 1      // Apenas 1 call de teste em half-open
});

async function sendSlackMessage(msg) {
  return slackCircuitBreaker.execute(async () => {
    return slackAPI.postMessage(msg);
  });
}
```

### Fallbacks
Quando circuito está aberto:
- Retornar resposta em cache (se disponível).
- Retornar erro gracioso: "Serviço temporariamente indisponível".
- Enfileirar mensagem para retry posterior (fila dead-letter).

---

## 5. Timeout em Todas as Operações

Nenhuma operação pode rodar indefinidamente.

| Operação | Timeout Padrão | Ação ao Exceder |
|----------|----------------|-----------------|
| HTTP Request (Incoming) | 30s | Retornar 408 Timeout |
| API Externa (Outgoing) | 5s | Circuit Breaker + Retry |
| Query SQL | 2s | Abort query + Log erro |
| Lock Redis | 10s | Release lock + Erro |
| Webhook Delivery | 10s | Marcar como falha + Retry |

### Exemplo de Timeout
```typescript
import { timeout } from 'promise-timeout';

try {
  const result = await timeout(
    externalApi.call(),
    5000 // 5 segundos
  );
} catch (error) {
  if (error.name === 'TimeoutError') {
    logger.warn('External API timed out');
    // Acionar fallback
  }
}
```

---

## 6. Plano de Contingência por Tipo de Falha

| Cenário | Detecção | Ação Automática | Ação Manual |
|---------|----------|-----------------|-------------|
| **DB Down** | Health check falha | Parar servidor (fail-fast), ativar modo leitura-only (cache) | Restaurar backup, verificar logs DB |
| **Redis Down** | Erros de conexão | Desativar rate limiting (degradar segurança), usar memória local | Reiniciar Redis, verificar rede |
| **API Externa Down** | Circuit breaker open | Usar fallback/cache, enfileirar requests | Contatar provider, verificar status page |
| **Memory Leak** | Monitoramento (Prometheus) | Reiniciar container automaticamente (K8s) | Analisar heap dump, corrigir código |
| **Disk Full** | Alerta de disco | Parar escrita de logs, limpar logs antigos | Expandir disco, investigar causa |
| **Security Breach** | IDS/WAF alert | Bloquear IPs suspeitos, revogar tokens comprometidos | Investigar forense, notificar usuários |

---

## 7. Logs de Falha para Debug

Em caso de falha crítica, garantir que os seguintes dados sejam logados:
- Timestamp exato.
- ID do request (correlation ID).
- Stack trace completo (em dev) ou resumido (prod).
- Estado das variáveis relevantes (sem dados sensíveis).
- Contexto: usuário, workspace, ação sendo executada.

### Formato de Log de Erro Crítico
```json
{
  "level": "fatal",
  "time": "2024-01-15T10:30:00Z",
  "service": "ferdium-server",
  "requestId": "abc-123-xyz",
  "userId": "usr_001",
  "action": "webhook_delivery",
  "error": {
    "type": "DatabaseConnectionError",
    "message": "Connection refused to postgres:5432",
    "code": "ECONNREFUSED"
  },
  "context": {
    "retryCount": 3,
    "lastRetryAt": "2024-01-15T10:29:55Z"
  }
}
```

---

## 8. Procedimento de Rollback de Emergência

Se uma implantação causar falhas críticas:

1. **Detectar:** Alertas de saúde, aumento de erro 5xx, latência alta.
2. **Decidir:** Tech Lead ou On-Call decide rollback em até 5 minutos.
3. **Executar:**
   ```bash
   # Kubernetes
   kubectl rollout undo deployment/ferdium-server
   
   # Docker Compose
   docker-compose up -d --force-recreate ferdium-server:v1.2.2
   ```
4. **Validar:** Confirmar health checks voltaram ao verde.
5. **Comunicar:** Avisar stakeholders sobre incidente e rollback.
6. **Investigar:** Analisar logs da versão falha em ambiente isolado.

---

## 9. Checklist de Resiliência

Antes de marcar feature como "pronta":

- [ ] Tem timeout definido?
- [ ] Tem fallback em caso de falha?
- [ ] Circuit breaker implementado (se chama externo)?
- [ ] Transações são atômicas (rollback em erro)?
- [ ] Logs de erro são informativos?
- [ ] Health check reflete estado real?
- [ ] Migração tem rollback (down())?
- [ ] Feature flag disponível para desativar rápido?
