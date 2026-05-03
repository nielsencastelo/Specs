# Spec: Pipeline de Multi-Agentes com RAG e Banco Vetorial

## Objetivo

Este documento especifica como implementar um sistema de chatbot com pipeline sequencial de agentes de IA, base de conhecimento vetorial (RAG), banco de dados PostgreSQL com pgvector, e integrações de canais via webhook. A arquitetura é genérica o suficiente para ser reaproveitada em qualquer domínio.

---

## Visão geral da arquitetura

```
Mensagem inbound (canal externo)
        ↓
    Routes (FastAPI)
        ↓
  MessageService
  ├── Verifica pausa / handoff ativo
  ├── Identifica ou cria usuário
  └── Chama AgentPipeline
            ↓
    ┌──────────────────┐
    │  SecurityGuard   │ → bloqueia → resposta de recusa
    └──────────────────┘
            ↓ seguro
    ┌──────────────────┐
    │   Classifier     │ → needs_human → cria HandoffRequest
    └──────────────────┘
            ↓ normal
    ┌──────────────────┐
    │    Response      │ → KnowledgeService (RAG + LLM)
    └──────────────────┘
            ↓
    Resposta ao usuário via canal
```

---

## 1. Banco de dados

### Stack

- **PostgreSQL** com extensão **pgvector** para armazenamento de embeddings.
- SQLAlchemy ORM com `SessionLocal` factory e `pool_pre_ping=True`.
- pgvector habilitado na inicialização: `CREATE EXTENSION IF NOT EXISTS vector`.

### Schema multi-tenant

Toda tabela de dados pertence a um `client_id`. O cliente representa um negócio/persona isolada com sua própria base de conhecimento e histórico.

#### Tabela: `clients`

| Coluna | Tipo | Descrição |
|---|---|---|
| `id` | UUID/serial | PK |
| `name` | text | Nome do cliente/negócio |
| `external_ref` | text | Referência externa opcional |
| `system_prompt` | text | Prompt de personalidade do bot |
| `created_at` | timestamp | Data de criação |

- Exclusão em cascade para todas as tabelas dependentes.

#### Tabela: `client_channel_identities`

| Coluna | Tipo | Descrição |
|---|---|---|
| `id` | serial | PK |
| `client_id` | FK → clients | Tenant ao qual pertence |
| `channel` | text | `telegram`, `whatsapp`, `whatsapp_evolution` |
| `external_user_id` | text | ID do usuário no canal externo |
| `display_name` | text | Nome capturado do canal |

- Unique constraint: `(channel, external_user_id)`.
- Auto-criação no primeiro contato do usuário.

#### Tabela: `client_documents`

| Coluna | Tipo | Descrição |
|---|---|---|
| `id` | serial | PK |
| `client_id` | FK → clients | Tenant |
| `filename` | text | Nome original do arquivo |
| `raw_text` | text | Conteúdo bruto extraído |
| `content_type` | text | `txt`, `md`, `pdf`, `docx`, `json` |
| `summary` | text | Resumo gerado por LLM |

#### Tabela: `client_document_chunks`

| Coluna | Tipo | Descrição |
|---|---|---|
| `id` | serial | PK |
| `client_id` | FK → clients | Tenant |
| `document_id` | FK → documents | Documento pai |
| `chunk_text` | text | Trecho do documento |
| `embedding` | vector(N) | Embedding do chunk |

- Chunks com sobreposição (ex: 600 chars, overlap de 100).
- Usados para recuperação verbatim de trechos específicos.

#### Tabela: `client_facts`

| Coluna | Tipo | Descrição |
|---|---|---|
| `id` | serial | PK |
| `client_id` | FK → clients | Tenant |
| `fact_text` | text | Fato extraído do documento |
| `embedding` | vector(N) | Embedding do fato |
| `source_document_id` | FK opcional | Documento de origem |

- Fatos: informações estáveis (preços, produtos, serviços, contatos).
- Gerados por LLM durante ingestão de documentos.

#### Tabela: `client_rules`

| Coluna | Tipo | Descrição |
|---|---|---|
| `id` | serial | PK |
| `client_id` | FK → clients | Tenant |
| `rule_text` | text | Regra de comportamento |
| `priority` | int | Menor número = maior prioridade |
| `active` | boolean | Ativa ou suspensa |
| `embedding` | vector(N) | Embedding da regra |

- Regras: diretrizes de tom, compliance, condições de escalada.

#### Tabela: `conversation_messages`

| Coluna | Tipo | Descrição |
|---|---|---|
| `id` | serial | PK |
| `client_id` | FK → clients | Tenant |
| `external_user_id` | text | Usuário no canal |
| `role` | text | `user` ou `assistant` |
| `content` | text | Texto da mensagem |
| `embedding` | vector(N) | Embedding (background) |
| `created_at` | timestamp | Timestamp |

#### Tabela: `handoff_requests`

| Coluna | Tipo | Descrição |
|---|---|---|
| `id` | serial | PK |
| `client_id` | FK → clients | Tenant |
| `external_user_id` | text | Usuário solicitante |
| `channel` | text | Canal de origem |
| `trigger_message` | text | Mensagem que ativou o handoff |
| `status` | text | `pending`, `active`, `resolved` |
| `operator_id` | text | Operador atribuído |
| `created_at` | timestamp | Timestamp |

#### Tabela: `agent_configs`

| Coluna | Tipo | Descrição |
|---|---|---|
| `id` | serial | PK |
| `agent_name` | text | `security`, `classifier`, `response` |
| `provider` | text | `ollama`, `openai`, `anthropic`, `gemini` |
| `model` | text | Nome do modelo |
| `enabled` | boolean | Agente ativo ou ignorado |
| `blocked_response` | text | Mensagem de recusa (security) |
| `use_gpu` | boolean | GPU affinity para Ollama |

- Seed automático no startup com defaults por agente.
- `ON CONFLICT DO NOTHING` para não sobrescrever configurações salvas.

#### Tabela: `agent_logs`

| Coluna | Tipo | Descrição |
|---|---|---|
| `id` | serial | PK |
| `agent_name` | text | Nome do agente |
| `input_preview` | text | Trecho do input |
| `output_preview` | text | Trecho do output |
| `duration_ms` | int | Tempo de execução |
| `safe` | boolean | Flag de segurança (security agent) |
| `intent` | text | Intent classificado |
| `blocked` | boolean | Se foi bloqueado |
| `created_at` | timestamp | Timestamp |

#### Tabela: `system_config`

| Coluna | Tipo | Descrição |
|---|---|---|
| `key` | text | PK, chave de configuração |
| `value` | text | Valor da configuração |

- Configuração dinâmica sem restart.
- Exemplos de chaves: `default_client_id`, `llm_provider`, `llm_model`, `embedding_provider`, `openai_api_key`, `bot_paused`.

---

## 2. Pipeline de agentes

### Estrutura

O pipeline é composto por três agentes executados sequencialmente. Security e Classifier podem ser paralelizados (via `ThreadPoolExecutor`).

```python
class AgentPipeline:
    def run(self, message: str, context: dict) -> AgentResult:
        # Stage 1: Segurança
        security_result = self.security_agent.run(message)
        if security_result.blocked:
            return AgentResult(blocked=True, response=security_result.blocked_response)

        # Stage 2: Classificação
        classifier_result = self.classifier_agent.run(message, context)
        if classifier_result.needs_human:
            self._create_handoff(context)
            return AgentResult(handoff=True, response=HANDOFF_CONFIRMATION_MESSAGE)

        # Stage 3: Resposta
        response = self.response_agent.run(message, context, classifier_result)
        return AgentResult(response=response)
```

### Agente 1: SecurityGuard

**Responsabilidade:** Filtrar mensagens maliciosas ou inadequadas antes de qualquer processamento.

**Input:** Texto da mensagem do usuário.

**Output JSON:**
```json
{
  "safe": true,
  "reason": "mensagem normal"
}
```

**Categorias detectadas:**
- SQL injection, XSS, command injection
- Prompt injection / jailbreak
- Conteúdo ilícito, assédio, ameaças

**Comportamento:**
- Modelo pequeno e rápido (menor custo por requisição).
- Retorna mensagem de recusa configurável por cliente.
- Loga tentativa bloqueada com motivo.

**Configuração:**
```python
agent_configs["security"] = {
    "provider": "ollama",
    "model": "phi4-mini",
    "enabled": True,
    "blocked_response": "Desculpe, não posso responder a isso."
}
```

### Agente 2: Classifier

**Responsabilidade:** Entender a intenção e o contexto da mensagem.

**Input:** Texto + histórico recente.

**Output JSON:**
```json
{
  "intent": "question",
  "language": "pt",
  "tone": "formal",
  "needs_human": false
}
```

**Intents possíveis:**
`question`, `complaint`, `sales`, `support`, `greeting`, `feedback`, `off_topic`, `human_handoff`

**Detecção de handoff:**
- Primário: LLM classifica `intent = "human_handoff"` ou `needs_human = true`.
- Fallback: Keyword matching em lista de frases configuráveis (ex: "quero falar com atendente", "preciso de ajuda humana").

**Configuração:**
```python
agent_configs["classifier"] = {
    "provider": "ollama",
    "model": "phi4-mini",
    "enabled": True
}
```

### Agente 3: Response

**Responsabilidade:** Gerar a resposta final com contexto RAG.

**Input:** Mensagem + resultado do Classifier + contexto RAG montado por KnowledgeService.

**Comportamento:**
- Usa o modelo principal do sistema (configurável).
- Monta contexto com: system_prompt, fatos RAG, regras RAG, chunks RAG, memória de conversa, histórico recente.
- Detecta sinal de auto-handoff `[CHAMAR_HUMANO]` no output → cria HandoffRequest automaticamente.

**Configuração:**
```python
agent_configs["response"] = {
    "provider": "ollama",   # ou "openai", "anthropic", "gemini"
    "model": "phi4-mini",   # modelo principal
    "enabled": True
}
```

---

## 3. RAG (Retrieval-Augmented Generation)

### Três camadas de recuperação

```
Mensagem do usuário
        ↓
┌──────────────────────────────────────────────┐
│  Camada 1: Fatos e Regras                    │
│  - cosine similarity sobre embeddings        │
│  - threshold mínimo configurável             │
│  - fallback: últimos N fatos/regras          │
└──────────────────────────────────────────────┘
        ↓
┌──────────────────────────────────────────────┐
│  Camada 2: Chunks de Documentos              │
│  - recuperação verbatim de trechos           │
│  - chunks sobrepostos para contexto contínuo │
└──────────────────────────────────────────────┘
        ↓
┌──────────────────────────────────────────────┐
│  Camada 3: Memória de Conversa               │
│  - histórico semântico do próprio usuário    │
│  - "Usuário disse: ..." / "Você respondeu:"  │
└──────────────────────────────────────────────┘
        ↓
  Contexto final → LLM
```

### Query de similaridade (pgvector)

```sql
SELECT fact_text, embedding <=> :query_embedding AS distance
FROM client_facts
WHERE client_id = :client_id
ORDER BY distance ASC
LIMIT :top_k;
```

- Operador `<=>` = distância cosseno.
- Threshold mínimo: ex. `distance < 0.28` (equivalente a similarity > 0.72).

### Pipeline de ingestão de documentos

```
Arquivo (TXT / MD / JSON / PDF / DOCX)
        ↓
    DocumentParser
        ↓ raw_text
    Chunking (4000 chars para LLM)
        ↓
    LLM extrai: summary + facts[] + rules[]
        ↓
    Deduplicação (dict.fromkeys)
        ↓
    EmbeddingService → vetores para cada item
        ↓
    Persistência em DB
    (client_facts, client_rules, client_document_chunks)
```

### EmbeddingService

Suporte a dois provedores intercambiáveis:

| Provider | Endpoint | Modelo padrão |
|---|---|---|
| `ollama` | `POST /api/embeddings` | `nomic-embed-text` |
| `openai` | `POST /v1/embeddings` | `text-embedding-3-small` |

- Troca de provider dispara reindexação de todos os embeddings.
- Embedding em background de mensagens do usuário (sem bloquear resposta).

### Montagem do contexto final (KnowledgeService)

```python
system_parts = [
    client.system_prompt,
    document_summaries,
    f"Usuário: {user_display_name}",
    rag_facts,           # Camada 1
    rag_rules,           # Camada 1
    rag_chunks,          # Camada 2
    rag_conversation,    # Camada 3
]
```

---

## 4. LLMService (multi-provider)

### Provedores suportados

| Provider | API Style | Auth |
|---|---|---|
| `ollama` | REST local | Nenhuma |
| `openai` | Messages API | `OPENAI_API_KEY` |
| `anthropic` | Messages API | `ANTHROPIC_API_KEY` |
| `gemini` | SDK genai | `GEMINI_API_KEY` |

### Métodos principais

```python
class LLMService:
    def complete(self, system_prompt: str, user_prompt: str) -> str: ...
    def complete_with_history(self, system_prompt: str, messages: list) -> str: ...
    def complete_json(self, system_prompt: str, user_prompt: str) -> dict: ...
```

### `complete_json` — tratamento robusto de JSON

1. Remove blocos markdown (` ```json ... ``` `).
2. Extrai primeiro objeto JSON se houver texto ao redor.
3. Remove blocos `<think>` (modelos de raciocínio).
4. Em caso de `JSONDecodeError`, envia prompt de reparo e tenta novamente.

### Retry e timeout

```python
RETRIES = 3
BACKOFF = [1, 2, 4]          # segundos
RETRYABLE = [429, 500, 502, 503, 504]
TIMEOUT_OLLAMA = 600         # segundos
TIMEOUT_CLOUD  = 30          # segundos
```

---

## 5. Camada de rede (FastAPI)

### Estrutura de routers

```
POST /api/chat/test              # Teste com cliente específico
GET  /health                     # Status do banco

POST /webhooks/telegram          # Inbound Telegram
GET  /webhooks/whatsapp          # Verificação Meta
POST /webhooks/whatsapp          # Inbound WhatsApp Cloud
POST /webhooks/evolution         # Inbound Evolution API

GET  /api/admin/agents           # Lista agentes + config
PATCH /api/admin/agents/{name}   # Atualiza config do agente
GET  /api/admin/agents/logs/recent

GET  /api/operator/handoffs      # Lista handoffs
PATCH /api/operator/handoffs/{id}
POST /api/operator/handoffs/{id}/reply
POST /api/operator/handoffs/{id}/resolve

GET  /api/clients                # CRUD clientes
POST /api/clients/{id}/documents # Upload de documento
DELETE /api/clients/{id}

PATCH /api/config/telegram
PATCH /api/config/whatsapp
PATCH /api/config/evolution
PATCH /api/config/llm
PATCH /api/config/embeddings
```

### Padrão de webhook (resposta imediata + background task)

```python
@router.post("/webhooks/telegram")
async def telegram_webhook(request: Request, background: BackgroundTasks):
    payload = await request.json()
    background.add_task(process_message, payload)
    return {"ok": True}   # 200 imediato → evita retentativas do canal
```

### Deduplicação de eventos (Telegram)

```python
processed_ids: set[int] = set()   # in-memory, max 1000 entradas

def is_duplicate(update_id: int) -> bool:
    if update_id in processed_ids:
        return True
    processed_ids.add(update_id)
    if len(processed_ids) > 1000:
        processed_ids.pop()
    return False
```

### CORS

```python
app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://localhost:3000"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)
```

---

## 6. Channel Provider (abstração de canais)

### Interface base

```python
class BaseChannelProvider(ABC):
    @abstractmethod
    async def parse_inbound(self, payload: dict) -> InboundMessage | None: ...

    @abstractmethod
    async def send_text(self, message: OutboundMessage) -> DeliveryResult: ...
```

### InboundMessage

```python
@dataclass
class InboundMessage:
    channel: str              # "telegram" | "whatsapp" | "whatsapp_evolution"
    external_user_id: str     # ID do usuário no canal
    display_name: str | None  # Nome capturado
    text: str                 # Texto da mensagem
    raw_payload: dict         # Payload original para debug
```

### Implementações

| Provider | Canal | Notas |
|---|---|---|
| `TelegramOfficialProvider` | Telegram Bot API | Secret token header |
| `WhatsAppCloudProvider` | Meta Cloud API v23+ | verify_token + bearer |
| `EvolutionWhatsAppProvider` | Evolution API (self-hosted) | messages.upsert event |

### Seleção dinâmica de provider

```python
def get_provider(channel: str) -> BaseChannelProvider:
    providers = {
        "telegram": TelegramOfficialProvider,
        "whatsapp": WhatsAppCloudProvider,
        "whatsapp_evolution": EvolutionWhatsAppProvider,
    }
    return providers[channel]()
```

---

## 7. Configuração dinâmica

### Hierarquia de configuração (menor nível sobrescreve maior)

```
.env (fallback técnico)
    ↓
system_config table (painel de admin)
    ↓
agent_configs table (por agente)
    ↓
client.system_prompt (por cliente/tenant)
```

### Leitura de configuração runtime

```python
def get_system_config(db: Session, key: str, default=None) -> str:
    row = db.query(SystemConfig).filter_by(key=key).first()
    return row.value if row else default
```

### Configurações em `system_config`

| Chave | Descrição |
|---|---|
| `default_client_id` | Cliente usado quando não identificado |
| `llm_provider` | Provider padrão do sistema |
| `llm_model` | Modelo padrão do sistema |
| `embedding_provider` | `ollama` ou `openai` |
| `openai_api_key` | Chave OpenAI |
| `anthropic_api_key` | Chave Anthropic |
| `gemini_api_key` | Chave Gemini |
| `bot_paused` | `true` para pausar o bot globalmente |
| `telegram_token` | Token do bot Telegram |
| `whatsapp_access_token` | Bearer token Meta |
| `whatsapp_phone_number_id` | ID do número WhatsApp |
| `evolution_base_url` | URL da instância Evolution |
| `evolution_api_key` | Chave da instância Evolution |

---

## 8. Human Handoff

### Fluxo completo

```
Usuário: "quero falar com um humano"
    ↓
Classifier: needs_human = true
    ↓
MessageService:
    - Cria HandoffRequest (status: pending)
    - Retorna mensagem de confirmação
    - Marca usuário como em handoff ativo
    ↓
IA pausa para este external_user_id
    ↓
Operador abre painel:
    - PATCH /handoffs/{id} → status: active
    - POST /handoffs/{id}/reply → envia via canal
    - POST /handoffs/{id}/resolve → status: resolved
    ↓
IA retoma normalmente
```

### Verificação de handoff ativo em MessageService

```python
active_handoff = db.query(HandoffRequest).filter_by(
    external_user_id=user_id,
    client_id=client_id,
    status="active"
).first()

if active_handoff:
    return  # Não processa pelo pipeline de IA
```

### Auto-handoff por sinal no response

```python
if "[CHAMAR_HUMANO]" in response_text:
    create_handoff_request(db, context)
    response_text = response_text.replace("[CHAMAR_HUMANO]", "")
```

Permite que o próprio LLM decida escalar com base em regras de negócio (ex: "se o usuário relatar bug crítico, acionar humano").

---

## 9. Infraestrutura (Docker Compose)

### Serviços base

```yaml
services:
  backend:
    build: .
    ports: ["8080:8080"]
    depends_on: [postgres, redis]
    environment:
      DATABASE_URL: postgresql://...
      REDIS_URL: redis://redis:6379

  postgres:
    image: pgvector/pgvector:pg16
    environment:
      POSTGRES_DB: appdb
      POSTGRES_USER: appuser
      POSTGRES_PASSWORD: apppass
    volumes:
      - pg_data:/var/lib/postgresql/data

  frontend:
    build: ./frontend
    ports: ["3000:3000"]

  redis:
    image: redis:7-alpine
```

### Serviços opcionais (profiles)

```yaml
  ollama:
    image: ollama/ollama
    profiles: ["ollama"]
    volumes:
      - ollama_data:/root/.ollama

  ollama-gpu:
    image: ollama/ollama
    profiles: ["ollama-gpu"]
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]
```

### Tunnel público (Cloudflare Tunnel)

```yaml
  cloudflared:
    image: cloudflare/cloudflared
    command: tunnel --no-autoupdate run --token ${CLOUDFLARE_TUNNEL_TOKEN}
    depends_on: [backend]
```

Usado para expor o webhook de forma pública sem abrir portas no roteador.

---

## 10. Checklist de implementação

### Banco de dados
- [ ] Habilitar extensão `pgvector`
- [ ] Criar tabelas na ordem: `clients` → dependentes
- [ ] Configurar `CASCADE DELETE` em todas as FKs de `client_id`
- [ ] Adicionar índices vetoriais: `CREATE INDEX ON table USING ivfflat (embedding vector_cosine_ops)`
- [ ] Seed de `agent_configs` no startup (ON CONFLICT DO NOTHING)

### Pipeline de agentes
- [ ] Implementar `SecurityGuard` com saída JSON `{safe, reason}`
- [ ] Implementar `Classifier` com saída JSON `{intent, language, tone, needs_human}`
- [ ] Implementar `Response` integrado ao KnowledgeService
- [ ] Configurar fallback de keyword para detecção de handoff
- [ ] Logar execução de cada agente em `agent_logs`

### RAG
- [ ] Parser de documentos (TXT, MD, PDF, DOCX)
- [ ] Chunking com sobreposição
- [ ] Extração de fatos e regras via LLM
- [ ] EmbeddingService com dois provedores
- [ ] `retrieve_rag_context()` unificando as 3 camadas
- [ ] Threshold de similaridade configurável

### Rede
- [ ] Webhook retorna 200 imediato + background task
- [ ] Deduplicação de eventos por ID
- [ ] Rota de health check para dependências externas
- [ ] CORS configurado para o frontend

### Configuração
- [ ] `system_config` com leitura fallback para `.env`
- [ ] Seed de chaves padrão sem sobrescrever valores existentes
- [ ] Endpoint para trocar provider/modelo sem restart
