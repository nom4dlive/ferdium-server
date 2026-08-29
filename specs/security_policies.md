# 🔒 Políticas de Segurança "Hardcoded"

**Objetivo:** Definir regras de segurança inegociáveis que devem ser implementadas diretamente no código. Os agentes de IA **NÃO** podem criar atalhos ou ignorar estas regras sob nenhuma hipótese.

---

## 1. Funções e Práticas Proibidas (Blacklist)

O uso das seguintes funções/padrões resultará em falha automática no lint/teste:

| Categoria | Proibido | Alternativa Obrigatória | Motivo |
|-----------|----------|-------------------------|--------|
| **Eval/Exec** | `eval()`, `Function()`, `child_process.exec()` | `child_process.execFile()` com args fixos, sandboxes | Prevenção de RCE (Remote Code Execution) |
| **FS Direto** | `fs.readFileSync` em rotas HTTP | Streams, `fs.promises`, limites de tamanho | Bloqueio de I/O síncrono e DoS |
| **SQL Dinâmico** | Concatenação de strings em queries | Prepared Statements, Query Builders (Kysely/Prisma) | Prevenção de SQL Injection |
| **Random Crypto** | `Math.random()` para tokens/segurança | `crypto.randomBytes()`, `crypto.randomUUID()` | Previsibilidade de tokens |
| **Logs Sensíveis** | Log de `req.body` completo, senhas, tokens | Máscaras, omitir campos sensíveis | Vazamento de dados (LGPD/GDPR) |
| **CORS Wildcard** | `origin: '*'` com credenciais | Lista branca de origens via ENV | CSRF e vazamento de dados |
| **Any Type** | `any` no TypeScript | `unknown`, tipos específicos, Zod schemas | Perda de type safety |
| **Console** | `console.log`, `console.error` em prod | Logger estruturado (`pino`) | Vazamento de info, performance |

---

## 2. Validação e Sanitização de Inputs

**Regra:** Nenhum dado externo é confiável. Validar TUDO na borda (Controller/Middleware).

### Headers de Segurança Obrigatórios
Todos as respostas HTTP devem incluir:
```http
Strict-Transport-Security: max-age=31536000; includeSubDomains
X-Content-Type-Options: nosniff
X-Frame-Options: DENY
X-XSS-Protection: 1; mode=block
Referrer-Policy: strict-origin-when-cross-origin
Content-Security-Policy: default-src 'self'
```

### Validação de Payload
- **Tamanho Máximo:** Limitar body a 1MB (JSON) e 10MB (upload).
- **Tipo de Conteúdo:** Rejeitar qualquer request sem `Content-Type: application/json` (exceto uploads).
- **Schema Estrito:** Usar Zod para definir schema exato. Campos extras devem ser removidos (`strict()` ou `strip()`).

### Exemplo de Validação (Zod)
```typescript
import { z } from 'zod';

const CreateUserSchema = z.object({
  email: z.string().email(),
  password: z.string().min(8).regex(/[A-Z]/), // Requer maiúscula
  name: z.string().min(1).max(100),
  role: z.enum(['user', 'admin']).default('user')
});

// No controller
const data = CreateUserSchema.parse(req.body); 
// Dados aqui estão 100% tipados e validados
```

---

## 3. Autenticação e Autorização

### JWT Security
- **Algoritmo:** Apenas `RS256` (Assimétrico) ou `HS256` com segredo > 32 bytes.
- **Validação:** Verificar `exp`, `iat`, `iss`, `aud` em TODOS os requests protegidos.
- **Armazenamento:** Nunca armazenar tokens no localStorage do frontend (usar httpOnly cookies).

### Senhas
- **Hashing:** Argon2id (preferencial) ou bcrypt (custo >= 12).
- **Salting:** Automático pela biblioteca de hashing.
- **Política:** Mínimo 8 caracteres, 1 maiúscula, 1 número, 1 especial.

### Rate Limiting (Obrigatório)
- **Global:** 100 req/min por IP.
- **Auth:** 5 req/min para login/signup (prevenir brute-force).
- **API:** 30 req/min por API Key/User.
- **Implementação:** Redis-based (para funcionar em cluster).

---

## 4. Proteção contra Ataques Comuns

### SQL Injection
- **Regra:** Nunca interpoler variáveis em strings SQL.
- **Exemplo Errado:** `` db.query(`SELECT * FROM users WHERE id = ${id}`) ``
- **Exemplo Correto:** `db.query('SELECT * FROM users WHERE id = $1', [id])`

### XSS (Cross-Site Scripting)
- **Output Encoding:** Escapar todos os dados renderizados no frontend.
- **CSP:** Header Content-Security-Policy restritivo.
- **HTML Input:** Sanitizar qualquer campo que aceite HTML (usar `dompurify`).

### CSRF (Cross-Site Request Forgery)
- **Tokens:** Usar tokens CSRF para requisições state-changing (POST, PUT, DELETE).
- **SameSite:** Cookies com atributo `SameSite=Strict` ou `Lax`.
- **Origins:** Validar header `Origin` e `Referer` em requests sensíveis.

### Path Traversal
- **Validação:** Checar se caminhos de arquivo não contêm `..`.
- **Restrição:** Limitar acesso a diretórios específicos (chroot virtual).
- **Bibliotecas:** Usar `path.resolve()` e validar prefixo.

---

## 5. Segurança de Dependências

### Auditoria
- Rodar `npm audit` ou `yarn audit` em todo PR.
- Bloquear merge se houver vulnerabilidade crítica/alta.
- Usar `Snyk` ou `Dependabot` para monitoramento contínuo.

### Pinning de Versões
- **Obrigatório:** Usar versões exatas (`"express": "4.18.2"`) no `package.json`.
- **Proibido:** Carets (`^`) ou tildes (`~`) para dependências de produção críticas.
- **Lockfile:** `package-lock.json` ou `yarn.lock` deve estar sempre atualizado e versionado.

---

## 6. Privacidade de Dados (LGPD/GDPR)

### Classificação de Dados
- **Públicos:** Nome, avatar (se usuário permitir).
- **Internos:** Email, ID do workspace.
- **Sensíveis:** Senhas, tokens, CPF, cartão de crédito, logs de auditoria.

### Regras de Manipulação
- **Mascaramento:** Logs nunca mostram dados sensíveis completos (ex: `***@gmail.com`, `****-1234`).
- **Criptografia em Repouso:** Dados sensíveis no DB devem ser criptografados (AES-256).
- **Direito ao Esquecimento:** Endpoint para deletar conta e todos os dados associados (soft delete + anonymization).

---

## 7. Gestão de Segredos (Secrets Management)

### Regras de ENV
- **Nunca** commitar `.env` no git.
- **Validação:** App não inicia se variáveis críticas faltarem.
- **Rotação:** Suportar rotação de chaves sem downtime (carregar novas chaves antes de invalidar antigas).

### Exemplo de Validação de ENV (Zod)
```typescript
const envSchema = z.object({
  NODE_ENV: z.enum(['development', 'production', 'test']),
  DATABASE_URL: z.string().url(),
  JWT_SECRET: z.string().min(32),
  PORT: z.string().transform(Number)
});

const env = envSchema.safeParse(process.env);
if (!env.success) {
  console.error('Invalid environment variables:', env.error.flatten());
  process.exit(1); // Falha rápida
}
```

---

## 8. Checklist de Segurança para Pull Requests

Antes de mergiar, verificar:

- [ ] Nenhuma função proibida foi usada?
- [ ] Todos os inputs são validados com Zod?
- [ ] Queries SQL usam prepared statements?
- [ ] Senhas/token são hash/criptografados?
- [ ] Headers de segurança estão presentes?
- [ ] Rate limiting está ativo nas rotas críticas?
- [ ] Logs não expõem dados sensíveis?
- [ ] Dependências estão atualizadas e sem vulnerabilities?
- [ ] Testes de segurança (ex: tentativas de injection) foram adicionados?

---

## 9. Resposta a Incidentes de Segurança

Se uma vulnerabilidade for detectada:
1. **Imediata:** Desativar funcionalidade afetada (feature flag).
2. **Investigação:** Analisar logs e escopo do impacto.
3. **Correção:** Patch de emergência seguindo fluxo normal de testes.
4. **Comunicação:** Notificar usuários afetados se houver vazamento de dados (obrigatório por lei em 72h).
5. **Post-Mortem:** Documentar causa raiz e prevenir recorrência.
