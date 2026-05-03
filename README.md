# Specs — Biblioteca de Especificações Técnicas

Repositório de especificações de implementação prontas para uso com agentes de IA (Claude Code, Cursor, Copilot, etc.) e desenvolvedores. Cada spec é um documento técnico completo que descreve **como construir** uma funcionalidade do zero — decisões arquiteturais, configurações, modelos, fluxos e checklists incluídos.

---

## Objetivo

Acelerar o desenvolvimento de projetos eliminando o tempo gasto em pesquisa e tomada de decisão repetitiva. Em vez de começar do zero toda vez que precisar de autenticação, pagamentos ou internacionalização, você abre a spec correspondente e executa — seja você ou um agente de IA.

**Princípios:**
- Specs são genéricas: não são atadas a nenhum projeto específico
- Uma IA deve conseguir ler a spec e implementar o sistema do zero
- Cada documento cobre decisões arquiteturais, código, configuração e checklist final

---


## Specs Planejadas

### Django

- **Autenticação & Autorização** — registro, login social (OAuth), 2FA, controle de permissões por grupo
- **Multi-tenancy** — isolamento de dados por organização com subdomínio ou schema separado no PostgreSQL
- **API REST com DRF** — versionamento, autenticação JWT, throttling, documentação com drf-spectacular
- **Upload de Arquivos** — imagens, documentos, armazenamento em S3/R2, processamento assíncrono com Celery
- **Notificações em Tempo Real** — Django Channels, WebSocket, notificações push via Firebase
- **Sistema de Assinatura (SaaS)** — planos, trial, cobrança recorrente via Stripe, portal do cliente
- **Auditoria & Logs de Atividade** — rastreamento de ações do usuário, Django Admin aprimorado
- **Background Jobs com Celery** — filas, retry, beat para tarefas agendadas, monitoramento com Flower

### Next.js

- **Autenticação com NextAuth.js** — providers sociais, sessão, proteção de rotas, middleware
- **SSR / SSG / ISR** — quando usar cada estratégia, caching, revalidação e fallback
- **App Router com Server Components** — layouts aninhados, streaming, suspense boundaries
- **Formulários com React Hook Form + Zod** — validação client/server, upload de arquivos, feedback de erro
- **Internacionalização com next-intl** — roteamento por locale, traduções, formatação de datas e moedas
- **Integração com API Django** — autenticação JWT no browser e no servidor, cache e stale-while-revalidate
- **Design System com Tailwind + shadcn/ui** — temas, componentes base, acessibilidade, dark mode

### Python Geral

- **CLI com Typer** — subcomandos, opções, validação, configuração via `.env`, distribuição com PyPI
- **Scraping Robusto** — Playwright, rotação de proxies, detecção de rate limit, persistência de resultados
- **ETL com Pandas + SQLAlchemy** — ingestão, transformação, carga incremental e schema migration
- **API Client Reutilizável** — retry com backoff, autenticação, cache, tipagem com Pydantic
- **Automação com Prefect / Airflow** — DAGs, dependências, observabilidade, execução em nuvem

### Ciência de Dados & Machine Learning

- **Pipeline de Dados com Pandas & Polars** — limpeza, feature engineering, versionamento de datasets
- **Treinamento e Rastreamento de Modelos** — MLflow, experimentos, registro de artefatos, comparação de runs
- **Servir Modelos com FastAPI** — endpoints de inferência, validação de input, latência e batching
- **EDA Padronizado** — análise exploratória reproduzível com notebooks estruturados e relatórios automatizados
- **Feature Store Leve** — armazenamento, versionamento e recuperação de features para treinamento e inferência

### Agentes & Inteligência Artificial

- **Agente com Claude API** — tool use, prompt caching, memory, conversação multi-turno
- **RAG (Retrieval-Augmented Generation)** — chunking, embedding, vector store (pgvector / Chroma), reranking
- **Agente com Memória Persistente** — short-term, long-term, episódica; armazenamento em banco e recuperação semântica
- **Multi-Agente com Claude Code SDK** — orquestração, sub-agentes especializados, handoff e supervisão
- **Avaliação de LLMs (LLM-as-judge)** — métricas, datasets de teste, CI para qualidade de respostas
- **Fine-tuning e RLHF Leve** — coleta de preferências, LoRA, avaliação antes/depois
- **MCP Server Customizado** — criação de ferramentas MCP para integrar sistemas internos ao Claude Code

### Infraestrutura & DevOps

- **Django em Produção** — Docker, Gunicorn, Nginx, PostgreSQL, Redis, CI/CD com GitHub Actions
- **Deploy Serverless** — Railway, Render, Fly.io — configuração, secrets, bancos gerenciados, rollback
- **Observabilidade** — Sentry para erros, Prometheus + Grafana para métricas, logging estruturado
- **Banco de Dados** — migrations seguras em produção, backups automatizados, read replicas, connection pooling

---

## Como Usar

### Com um agente de IA

```
Leia o arquivo [spec].md e implemente o sistema descrito no meu projeto.
Siga as decisões arquiteturais da seção 1 antes de começar.
```

### Como referência técnica

Abra a spec antes de iniciar uma feature. Use o **Checklist de Implementação** ao final de cada documento para garantir que nada foi esquecido.

### Contribuindo

1. Crie um arquivo com o padrão `YYYY-MM-DD-nome-da-feature-tecnologia-spec.md`
2. Siga a estrutura: Visão Geral → Decisões Arquiteturais → Implementação passo a passo → Checklist
3. A spec deve ser genérica o suficiente para funcionar em qualquer projeto da stack

---

## Estrutura de uma Spec

```
# Nome da Feature — Spec de Implementação

## 1. Visão Geral
## 2. Decisões Arquiteturais       ← escolhas com trade-offs explicados
## 3. Configuração / Settings
## 4. Modelos / Schema
## 5. Lógica de Negócio
## 6. Endpoints / Views / API
## 7. Testes
## 8. Checklist de Implementação   ← itens verificáveis ao final
```

---

## Licença

MIT — use, adapte e distribua livremente.
