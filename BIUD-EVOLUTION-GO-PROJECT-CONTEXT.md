# BIUD — Evolution Go Project Context

**Empresa:** BIUD Tecnologia
**Projeto:** Migração Evolution API v2 → Evolution Go
**Última Atualização:** Abril 2026
**Versão do Documento:** 1.0

---

## 1. MODELO DE NEGÓCIO

### 1.1 Estrutura Comercial

```
BIUD Tecnologia (SaaS Provider)
    ↓ [B2B - vende plataforma]
Empresas Clientes (ex: lojas, clínicas, e-commerces)
    ↓ [B2C - atende consumidores via WhatsApp]
Consumidores Finais
    ↓ [enviam mensagens WhatsApp para as empresas]
IA da BIUD analisa mensagens e dá insights/dicas
```

### 1.2 Produto BIUD

Plataforma de Atendimento Inteligente via WhatsApp:

- Empresa cliente conecta seu WhatsApp Business na plataforma BIUD
- Consumidores finais enviam mensagens para a empresa via WhatsApp
- IA da BIUD lê mensagens em tempo real
- IA dá dicas/sugestões para os atendentes da empresa
- Atendente vê insights e responde melhor/mais rápido

Exemplo de uso:

```
Cliente: "Tem esse vestido no tamanho M?"
IA BIUD sugere: "Cliente interessado em compra. Estoque: 3 unidades
tamanho M. Responder em <2min aumenta conversão em 40%"
Atendente: responde rapidamente com informação precisa
```

### 1.3 Características Técnicas

**1 Empresa Cliente = 1 Instância WhatsApp:**

- Cada empresa conecta 1 número de WhatsApp Business
- 1 instância na Evolution API = 1 empresa cliente BIUD
- Consumidores finais não acessam plataforma BIUD
- Consumidores finais nem sabem que BIUD existe (transparente)

**Volume:**

- Hoje (Evolution v2): 20-30 empresas clientes
- Fila de espera: ~100 empresas querendo contratar
- Objetivo imediato: atender 100-120 empresas
- Crescimento futuro: TBD (definir meta 6-12 meses)

---

## 2. PROBLEMA ATUAL

### 2.1 Limitações da Evolution API v2 (Node.js)

**Capacidade:**

- Máximo 20-30 empresas por instância/pod
- Após 30 instâncias: crashes, lentidão, perda de sessões
- Problema estrutural: Node.js single-threaded, consumo alto de memória

**Impacto no Negócio:**

- 100 empresas na fila sem conseguir usar o produto
- Perda de receita (não consegue onboarding)
- Risco de churn (clientes desistem de esperar)

**Tentativas de Solução (que não funcionaram):**

- Aumentar recursos do pod: não resolveu (problema de arquitetura)
- Múltiplos pods: sessões WhatsApp são stateful, não suportam balanceamento simples
- Otimização de código v2: limite estrutural da arquitetura Node.js

### 2.2 Arquitetura Atual (Evolution v2)

```yaml
Stack em Produção:
  - Evolution API v2.3.7 (Node.js)
  - PostgreSQL (compartilhado)
  - Redis (compartilhado)
  - Kubernetes (Rancher)
  - Infraestrutura: Contabo
  - Domínios: múltiplos ambientes
```

Ambientes:

- Dev: `evogo-dev.biud.com.br`
- Homologação: `evogo-hmg.biud.com.br`
- Produção: `evogo.biud.com.br`

---

## 3. SOLUÇÃO PROPOSTA: EVOLUTION GO

### 3.1 Por Que Evolution Go?

**Evolution Go (Golang):**

- Mesma equipe Evolution Foundation
- Reescrita completa da API em Go
- Promessa oficial: até 120 instâncias por pod (4x capacidade)
- Performance superior: concorrência nativa (goroutines)
- Menor consumo de memória em alta carga
- Binário compilado (vs interpretado Node.js)

**Versão:** v0.7.0 (atual do fork biudtech/evolution-biud)

- Status: Early adoption / Production-ready com caveats
- Maturidade: repositório oficial jovem
- Diferencial técnico: Usa whatsmeow (lib oficial WhatsApp em Go)

### 3.2 Trade-offs Conhecidos

**Vantagens:**
- 4x mais capacidade (20-30 → 120 instâncias)
- Menor consumo de RAM
- Performance CPU-bound melhor
- Ecosistema Evolution (suporte, atualizações)

**Desvantagens:**
- Versão nova vs v2.3.7 battle-tested
- Documentação menor que v2
- Ecossistema Go menor que Node.js
- Requer licença (heartbeat para Evolution Foundation)

**Decisão:** Trade-off aceito porque bloqueio de negócio é crítico (100 clientes esperando).

---

## 4. LIMITAÇÃO TÉCNICA CRÍTICA

### 4.1 Evolution Go NÃO Suporta Horizontal Scaling (Nativo)

**Problema Arquitetural:**

```
Sessões WhatsApp são stateful:
- Chaves criptográficas E2E armazenadas localmente
- Estado de conexão no filesystem do pod
- Path: /evolution/instances/{instanceID}/

Se Pod 1 tem Instância A:
  - Chaves criptográficas em /evolution/instances/A/
  - Load balancer roteia requisição para Pod 2
  - Pod 2 não tem as chaves → FALHA
```

**Consequência:**

- NÃO dá para fazer Horizontal Pod Autoscaler simples
- NÃO dá para balancear instâncias entre pods
- 1 pod = máximo 120 instâncias (hard limit)

### 4.2 Estratégias para Múltiplos Pods (Futuro)

**Opção 1: Session Affinity (Simples, mas limitado)**

```
- Nginx/Kong roteia cliente sempre para o mesmo pod
- Se pod crashar → cliente perde sessão → re-pair (QR code)
- Trade-off: downtime aceito, mas não é HA real
```

**Opção 2: Shared Session Storage (Complexo, HA real)**

```
- Modificar código Evolution Go
- Salvar chaves criptográficas em Redis/PostgreSQL
- Qualquer pod pode retomar qualquer sessão
- Requer fork + desenvolvimento custom
- Estimativa: 2-4 meses de trabalho (otimista)
```

**Decisão Atual:**

- Fase 1: Single pod (120 instâncias) → resolve problema imediato
- Fase 2-3: Avaliar fork + modificação se negócio crescer >120 empresas

---

## 5. INFRAESTRUTURA ATUAL (BIUD)

### 5.1 Kubernetes Environment

Provider: Contabo (bare metal K8s)

- Management: Rancher UI
- Namespace: `evolution-go` (dedicado)
- Cert Manager: cert-manager com Let's Encrypt (TLS automático)
- Ingress Controller: TBD - confirmar se Nginx ou Traefik

**Recursos Compartilhados (já existentes):**

- PostgreSQL cluster (disponível)
- Redis cluster (disponível)
- RabbitMQ (verificar se disponível ou criar)
- MinIO/S3 (verificar se necessário para mídia)

### 5.2 Domínios e Ambientes

| Ambiente | Domínio | Uso |
|---|---|---|
| Dev | `evogo-dev.biud.com.br` | Desenvolvimento e testes |
| Homologação | `evogo-hmg.biud.com.br` | Validação pré-produção |
| Produção | `evogo.biud.com.br` | Clientes reais |

TLS: Automático via cert-manager + Let's Encrypt

### 5.3 Sizing Inicial (Fase 1 - 100-120 empresas)

Evolution Go Pod:

```yaml
requests:
  cpu: 2000m (2 cores)
  memory: 4Gi
limits:
  cpu: 4000m (4 cores)
  memory: 8Gi
```

PostgreSQL (compartilhado):

- Conexão: 2 databases separados
  - `evogo_auth` (autenticação/licença)
  - `evogo_users` (dados de usuários/instâncias)
- Configuração específica Evolution Go (dual-database requirement)

Redis (compartilhado):

- Uso: Cache de sessões + rate limiting (futuro)
- Configuração específica Evolution Go

Total Estimado (apenas Evolution Go pod):

- Node K8s: ~4-8 cores disponíveis
- RAM: ~8-16Gi disponíveis
- Custo incremental: ~R$ 500-1.000/mês (se precisar node adicional)

---

## 6. ARQUITETURA TÉCNICA EVOLUTION GO

### 6.1 Estrutura de Código

```
evolution-go/
├── cmd/evolution-go/        # Entry point
├── pkg/
│   ├── core/                # Licença + middleware
│   ├── instance/            # Gerenciamento de instâncias
│   ├── message/             # Handlers de mensagens
│   ├── sendMessage/         # Send API
│   ├── routes/              # HTTP routes
│   ├── middleware/          # Auth + validação
│   ├── config/              # Configuração
│   ├── events/              # AMQP, NATS, Webhook, WS
│   └── storage/             # MinIO (mídia)
├── whatsmeow-lib/           # WhatsApp protocol (fork)
├── docs/                    # Swagger
└── Makefile
```

### 6.2 Variáveis de Ambiente Críticas

```bash
# Server
SERVER_PORT=8080
CLIENT_NAME=evolution

# Security
GLOBAL_API_KEY=<secure-random-key>

# Database (DUAL - requirement específico Go)
POSTGRES_AUTH_DB=postgresql://user:pass@postgres:5432/evogo_auth?sslmode=require
POSTGRES_USERS_DB=postgresql://user:pass@postgres:5432/evogo_users?sslmode=require

# Redis
REDIS_HOST=redis-cluster
REDIS_PORT=6379
REDIS_PASSWORD=<redis-password>

# Logging
WADEBUG=INFO
LOGTYPE=json

# Storage (Opcional)
MINIO_ENABLED=false
MINIO_ENDPOINT=minio:9000
MINIO_BUCKET=evolution-media

# Message Queue (Opcional)
AMQP_URL=amqp://user:pass@rabbitmq:5672/
RABBITMQ_ENABLED=false

# Webhooks
WEBHOOK_URL=https://biud-app/api/whatsapp/webhook
```

### 6.3 Licenciamento Evolution Go

**Sistema de Licença Obrigatório:**

```
1. Primeira inicialização → API retorna 503
2. Acessar Manager UI: http://evogo-dev.biud.com.br/manager/login
3. Inserir API URL + GLOBAL_API_KEY
4. Completar fluxo de registro (Evolution Foundation)
5. Heartbeat periódico mantém licença ativa
6. Status persiste em: runtime_configs table (PostgreSQL)
```

**Implicações:**

- Requer conectividade externa (Evolution Foundation servers)
- Licença pode ter custo (verificar com Evolution Foundation)
- Heartbeat failure → API pode desativar (verificar SLA)

Contato Licença: contato@evolution-api.com

---

## 7. INTEGRAÇÃO COM APLICAÇÃO BIUD

### 7.1 Endpoints da Evolution API Usados pela BIUD

**Gestão de Instâncias:**

- `POST /instance/create` — Criar nova instância (onboarding cliente)
- `GET /instance/{name}/qrcode` — Obter QR code para pairing
- `GET /instance/{name}/status` — Status da conexão
- `DELETE /instance/{name}` — Remover instância (offboarding)

**Mensagens:**

- `POST /message/sendText` — Enviar mensagem texto
- `POST /message/sendMedia` — Enviar mídia
- `GET /message/` — verificar se BIUD faz polling ou usa webhook

**Webhooks (se configurado):**

- `POST /webhook` — Evolution envia eventos para BIUD
  - `messages.upsert` — Nova mensagem recebida
  - `messages.update` — Mensagem atualizada
  - `connection.update` — Status conexão mudou

TODO: Documentar exatamente quais endpoints BIUD usa.

### 7.2 Fluxo de Dados (BIUD ↔ Evolution Go)

```
┌─────────────────────────────────────────────────┐
│  Consumidor Final (WhatsApp)                    │
└────────────────┬────────────────────────────────┘
                 │ Mensagem
                 ↓
┌─────────────────────────────────────────────────┐
│  Empresa Cliente (WhatsApp Business)            │
│  - Conectado via Evolution Go                   │
└────────────────┬────────────────────────────────┘
                 │ WebSocket/Webhook
                 ↓
┌─────────────────────────────────────────────────┐
│  Evolution Go API                               │
│  - Recebe mensagem                              │
│  - Armazena (PostgreSQL se configurado)         │
│  - Dispara webhook para BIUD                    │
└────────────────┬────────────────────────────────┘
                 │ HTTP POST webhook
                 ↓
┌─────────────────────────────────────────────────┐
│  Aplicação BIUD                                 │
│  - Recebe mensagem via webhook                  │
│  - IA processa e gera insights                  │
│  - Exibe sugestões para atendente               │
└─────────────────────────────────────────────────┘
                 │ (opcional)
                 ↓
┌─────────────────────────────────────────────────┐
│  Evolution Go API                               │
│  - BIUD envia resposta via API                  │
└────────────────┬────────────────────────────────┘
                 ↓
┌─────────────────────────────────────────────────┐
│  Empresa Cliente → Consumidor Final             │
└─────────────────────────────────────────────────┘
```

**Confirmar com time BIUD:**

- BIUD recebe mensagens via webhook ou faz polling?
- BIUD envia respostas via API ou apenas mostra sugestões?
- Precisa armazenar histórico de mensagens? (`DATABASE_SAVE_MESSAGES=true`)

---

## 8. PLANO DE IMPLEMENTAÇÃO

### 8.1 FASE 1: Deploy Mínimo Viável (2 semanas)

**Objetivo:** Rodar 100-120 empresas em 1 pod Evolution Go

**Entregáveis:**

1. Deployment K8s completo (PostgreSQL + Redis + Evolution Go)
2. Ingress com TLS (cert-manager + Let's Encrypt)
3. Configuração de secrets (credenciais seguras)
4. Ativação de licença Evolution Go
5. Smoke tests básicos (criar instância, obter QR code)
6. Documentação de deploy

**Recursos Necessários:**

- 1 pod Evolution Go (4 cores, 8Gi RAM)
- PostgreSQL: 2 databases (`evogo_auth`, `evogo_users`)
- Redis: conexão compartilhada
- TLS cert via cert-manager

**Riscos:**

- Licença Evolution Go pode ter custo (verificar antes)
- Compatibilidade com PostgreSQL/Redis existentes (testar)
- Versão pode ter bugs (monitorar logs ativamente)

**Critério de Sucesso:**

- 10 empresas piloto migradas da v2 → Go
- Sem crashes por 48h contínuas
- Latência API <500ms P95
- QR code pairing funciona 100%

### 8.2 FASE 2: Migração Gradual (2-4 semanas)

**Objetivo:** Migrar 100 empresas da fila + manter 20-30 atuais

**Opção A: Cold Migration (Recomendado - Simples)**

```
1. Comunicar cliente: "vamos atualizar infraestrutura"
2. Agendar janela (madrugada/fim de semana)
3. Desconectar WhatsApp na v2
4. Criar instância na Evolution Go
5. Cliente escaneia QR code novamente
6. Validar conexão
7. Deletar instância v2

Downtime: ~30min por cliente
Lotes: 10 empresas/dia
Total: ~10 dias úteis
```

**Opção B: Hot Migration (Complexo - Não Recomendado Inicialmente)**

```
1. Exportar session store da v2 (chaves criptográficas)
2. Importar na Evolution Go
3. Testar conexão
4. Fallback para Opção A se falhar

Risco: Formato de sessão pode ser incompatível (v2 Node.js vs Go)
Benefício: Zero downtime percebido
```

**Decisão:** Começar com Opção A, avaliar B apenas se SLA com clientes exigir.

**Plano de Rollback:**

- Manter Evolution v2 rodando em paralelo (standby)
- Se Evolution Go crashar → reverter tráfego para v2
- Período de overlap: 4 semanas (até migração completa validada)

### 8.3 FASE 3: Otimização e Observability (1-2 meses)

**Objetivo:** Produção estável, monitorada, otimizada

1. **Monitoring Stack:**
   - Prometheus (métricas Evolution Go)
   - Grafana (dashboards: instâncias ativas, latência, errors)
   - Loki (logs agregados)
   - AlertManager (alertas críticos)

2. **Dashboards Específicos BIUD:**
   - Instâncias ativas por empresa
   - Taxa de mensagens/hora por empresa
   - Latência P50/P95/P99 da API
   - Taxa de erro QR code pairing
   - Uptime por empresa cliente

3. **Alertas Críticos:**
   - Pod Evolution Go em CrashLoopBackOff
   - 10% instâncias desconectadas
   - Latência P95 >2s por >5min
   - Database connection pool >80%
   - Licença Evolution Go perto de expirar

4. **Tuning Performance:**
   - PostgreSQL: `max_connections`, `shared_buffers`
   - Redis: `maxmemory-policy`, cache eviction
   - Evolution Go: garbage collection Go (GOGC)

### 8.4 FASE 4: Kong API Gateway (2-3 meses - Paralelo)

**Objetivo:** Rate limiting, autenticação, observability avançada

1. **Rate Limiting por Empresa Cliente:**

```
Empresa Pequena: 1.000 req/hora
Empresa Média: 10.000 req/hora
Empresa Enterprise: Ilimitado
```

2. **API Key por Empresa:**

```
- Cada empresa BIUD tem sua API key
- Revogação fácil (offboarding)
- Auditoria de uso por cliente
```

3. **Request Routing Inteligente:**

```
/instance/* → Evolution Go
/ai/* → Aplicação BIUD (IA)
/webhook/* → Aplicação BIUD (receber eventos)
```

4. **Observability:** métricas/latência/erros por cliente.

**Decisão:** Kong é nice-to-have, não blocker.

### 8.5 FASE 5: Fork e Modificação (4-6 meses - Opcional)

**Objetivo:** Código Evolution Go otimizado para BIUD + HA real

**Quando Considerar:**

- Se crescimento passar de 120 empresas
- Se SLA com clientes exigir 99.9% uptime
- Se features específicas BIUD não existirem na Evolution Go

**Trabalho Necessário:**

1. Fork Repositório (`github.com/biudtech/evolution-biud`)
2. Remover Features Não Usadas (Typebot, Chatwoot, Dify AI/OpenAI, S3/MinIO se não usar)
3. Adicionar Session Storage Compartilhado (chaves cripto em PostgreSQL/Redis)
4. Health Checks Custom
5. Métricas Custom BIUD

Esforço Estimado: 2-4 meses (1-2 devs Go sênior) — **otimista**.
Trade-off: Manutenção própria vs dependência Evolution Foundation updates.

---

## 9. DECISÕES ARQUITETURAIS IMPORTANTES

### 9.1 Decisões Tomadas

| # | Decisão | Rationale | Data |
|---|---|---|---|
| 1 | Evolution Go vs Node.js v2 | 4x capacidade, resolve bloqueio negócio | Abr/2026 |
| 2 | Single pod (Fase 1) vs HA imediato | HA requer fork, single pod resolve urgência | Abr/2026 |
| 3 | Cold migration vs Hot migration | Simples, menor risco, downtime aceitável | Abr/2026 |
| 4 | Kong Gateway em Fase 4 | Nice-to-have, não blocker crítico | Abr/2026 |
| 5 | PostgreSQL dual-database | Requirement Evolution Go | Abr/2026 |

### 9.2 Decisões Pendentes

| # | Questão | Opções | Impacto | Prazo |
|---|---|---|---|---|
| 1 | Armazenar mídia? | MinIO vs não armazenar | Storage + custo | Fase 1 |
| 2 | Webhooks síncronos ou async? | Webhook direto vs queue | Latência + confiabilidade | Fase 1 |
| 3 | Histórico de mensagens? | `DATABASE_SAVE_MESSAGES=true/false` | Storage + custo DB | Fase 1 |
| 4 | SLA com clientes? | 99% vs 99.9% vs best-effort | HA necessity | Fase 2 |
| 5 | Crescimento 6-12 meses? | 200? 500? 1000? | Fork planning | Fase 3 |
| 6 | Custo licença Evolution Go? | Verificar pricing oficial | Budget | Imediato |

---

## 10. RISCOS E MITIGAÇÕES

### 10.1 Riscos Técnicos

| Risco | Probabilidade | Impacto | Mitigação |
|---|---|---|---|
| Evolution Go tem bugs críticos | Média | Alto | Testes rigorosos Fase 1, v2 standby |
| Licença cara/inviável | Baixa | Alto | Verificar pricing ANTES de Fase 1 |
| Incompatibilidade session v2 → Go | Alta | Médio | Assumir cold migration |
| PostgreSQL/Redis incompatíveis | Baixa | Alto | Smoke tests em dev |
| Performance <120 instâncias | Média | Alto | Load tests graduais (50, 80, 100, 120) |
| Crash em produção pós-migração | Média | Crítico | Rollback plan, v2 standby 4 semanas |

### 10.2 Riscos de Negócio

| Risco | Probabilidade | Impacto | Mitigação |
|---|---|---|---|
| Clientes na fila desistem | Alta | Alto | Comunicação proativa |
| Downtime migração afeta satisfação | Média | Médio | Janelas agendadas |
| Custo infra explode | Baixa | Alto | Sizing conservador |
| SLA breach se single pod crashar | Alta | Crítico | SLA conservador Fase 1 (99%) |

---

## 11. MÉTRICAS DE SUCESSO

### 11.1 KPIs Técnicos (Fase 1)

| Métrica | Target | Como Medir |
|---|---|---|
| Capacidade máxima | 120 empresas/pod | Teste de carga gradual |
| Uptime | >99% | `up{job="evolution-go"}` |
| Latência API P95 | <500ms | `http_request_duration_seconds` |
| Taxa de erro API | <1% | `http_requests_total{status=~"5.."}` |
| QR pairing success | >95% | `qr_pairing_success_rate` |
| Crash loop | Zero por >48h | `kube_pod_container_status_restarts_total` |

### 11.2 KPIs de Negócio

| Métrica | Target | Como Medir |
|---|---|---|
| Empresas migradas (v2 → Go) | 100% em 4 semanas | Tracking manual |
| Empresas fila onboardadas | 100 em 6 semanas | CRM/Admin Panel |
| Churn pós-migração | <5% | CRM |
| NPS pós-migração | >8 | Survey clientes |
| Tempo médio de onboarding | <30min | Admin panel analytics |

### 11.3 Critérios Go/No-Go Produção

- 10 empresas piloto em Dev por >1 semana sem crash
- Load test: 100 instâncias simultâneas OK
- Latência P95 <500ms em carga
- Rollback plan testado e documentado
- Alertas críticos configurados e validados
- Licença Evolution Go ativada e válida
- Time BIUD treinado (deploy, troubleshoot, rollback)

---

## 12. ARQUIVOS DE REFERÊNCIA

### 12.1 Repositórios

| Recurso | URL |
|---|---|
| Evolution Go (oficial) | https://github.com/EvolutionAPI/evolution-go |
| Evolution API v2 (oficial) | https://github.com/EvolutionAPI/evolution-api |
| Whatsmeow | https://github.com/tulir/whatsmeow |
| Fork BIUD | https://github.com/biudtech/evolution-biud |
| Evolution Go Docs | https://docs.evolutionfoundation.com.br |

### 12.2 Documentação Técnica

| Documento | Propósito |
|---|---|
| `BIUD-EVOLUTION-GO-PROJECT-CONTEXT.md` | Este arquivo |
| `evolution-go-architecture.md` | Arquitetura técnica detalhada |
| `deployment.yaml` (A gerar) | Kubernetes manifests |
| `RUNBOOK.md` (A gerar) | Procedimentos operacionais |
| `MIGRATION-GUIDE.md` (A gerar) | Passo-a-passo migração v2 → Go |

---

## 13. CONTATOS E RESPONSABILIDADES

### 13.1 Time BIUD

| Papel | Responsável | Contato |
|---|---|---|
| Tech Lead / Arquitetura | Guilherme (Gui) | gui@biud.tech |
| DevOps / Infra | TBD | - |
| Backend / Integração | TBD | - |
| Product Owner | TBD | - |

### 13.2 Fornecedores Externos

| Fornecedor | Produto | Contato |
|---|---|---|
| Evolution Foundation | Evolution Go licença + suporte | contato@evolution-api.com |
| Contabo | Infraestrutura K8s | Suporte via portal |

---

## 14. INFORMAÇÕES TÉCNICAS ESPECÍFICAS BIUD

### 14.1 Infraestrutura Atual

- Kubernetes Provider: Contabo (bare metal)
- Management: Rancher
- Ingress: TBD (Nginx ou Traefik)
- Cert Manager: cert-manager + Let's Encrypt

**Serviços Compartilhados (confirmar endpoint/credenciais):**

- PostgreSQL (versão?)
- Redis (versão?)
- RabbitMQ (existe ou criar?)
- MinIO (existe ou criar?)

### 14.2 Domínios e DNS

DNS: verificar provedor (Cloudflare? Route53? Contabo?)
CDN: verificar se usa.

### 14.3 Aplicação BIUD (Backend)

**Stack — TBD:** linguagem, framework, arquitetura (mono vs micro).

**Integração com Evolution API:**

- Endpoints usados: listar exatamente
- Autenticação: API Key (GLOBAL_API_KEY)
- Webhooks: confirmar URL e eventos
- Polling: frequência, se usado

**Armazenamento:** database principal, cache, object storage — TBD.

---

## 15. PERGUNTAS EM ABERTO

### 15.1 Técnicas

1. Endpoints Evolution API usados pela BIUD (listar).
2. Webhooks (recebe? URL? eventos? polling?).
3. Armazenamento de mensagens (`DATABASE_SAVE_MESSAGES=true/false`).
4. Armazenamento de mídia (MinIO/S3 ou descartar).
5. Message Queue (RabbitMQ: sim/não).
6. Endpoints/credenciais PostgreSQL e Redis compartilhados.

### 15.2 Negócio

1. Crescimento esperado em 6-12 meses.
2. SLA com clientes (uptime/manutenção).
3. Receita por empresa cliente (break-even vs custo infra).
4. Deadline real para os 100 da fila.
5. Budget infra mensal disponível.

---

## 16. PRÓXIMOS PASSOS IMEDIATOS

### 16.1 Validações Necessárias (ANTES de Deploy)

**Prioridade 1 (Blockers):**

1. Verificar pricing/custo licença Evolution Go
2. Confirmar compatibilidade PostgreSQL/Redis existentes
3. Testar Evolution Go em Dev isolado (smoke test)
4. Documentar exatamente quais endpoints BIUD usa

**Prioridade 2 (Recomendado):**

5. Mapear fluxo completo: BIUD App ↔ Evolution Go ↔ WhatsApp
6. Definir estratégia de migração (Cold vs Hot)
7. Criar runbook de deploy + rollback
8. Configurar monitoring básico (Prometheus + Grafana)

### 16.2 Artefatos a Gerar

**Fase 1:**

- [ ] `deployment.yaml`
- [ ] `secrets.yaml` (template sem credenciais reais)
- [ ] `DEPLOY.md`
- [ ] `RUNBOOK.md`
- [ ] `MIGRATION-GUIDE.md`

**Fase 2:**

- [ ] `MONITORING.md`
- [ ] `PERFORMANCE-TUNING.md`
- [ ] `TROUBLESHOOTING.md`

---

## 17. GLOSSÁRIO

| Termo | Definição |
|---|---|
| Instância | 1 conexão WhatsApp = 1 empresa cliente BIUD = 1 número |
| Evolution API | API REST para WhatsApp (v2 = Node.js, Go = Golang) |
| Whatsmeow | Lib Go para protocolo WhatsApp Web |
| Session | Estado de conexão WhatsApp (chaves E2E + metadata) |
| QR Code Pairing | Processo de vincular WhatsApp Web ao telefone |
| Stateful | Aplicação que mantém estado local |
| HA | Sistema que tolera falhas sem downtime |
| Horizontal Scaling | Adicionar mais pods |
| Session Affinity | Rotear cliente sempre para o mesmo pod |
| Cold Migration | Migração com downtime |
| Hot Migration | Migração sem downtime |
| Fork | Cópia de repositório para dev customizado |
| Dual-Database | Evolution Go requer 2 DBs PostgreSQL (auth + users) |

---

## 18. CHANGELOG DO DOCUMENTO

| Versão | Data | Autor | Mudanças |
|---|---|---|---|
| 1.0 | Abr/2026 | Claude + Gui | Documento inicial completo |

---

**FIM DO CONTEXTO** — Atualizar conforme decisões são tomadas.
