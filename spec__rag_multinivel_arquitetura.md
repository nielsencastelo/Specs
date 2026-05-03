# Arquitetura RAG Multinível: Document Intelligence Pipeline

Esta especificação descreve um processo de RAG (Retrieval-Augmented Generation) de múltiplos níveis para ingestão, enriquecimento, indexação e consulta de grandes volumes de documentos. O objetivo é transformar documentos brutos (ex: PDFs) em uma base de conhecimento estruturada e semanticamente interconectada.

## Visão Geral do Pipeline

O processo é dividido em 5 etapas principais, garantindo que a informação seja refinada progressivamente antes de chegar ao usuário final.

### 1. Camada de Estruturação (Decomposition)
O primeiro nível foca na transformação de arquivos binários/não estruturados em dados granulares.
- **Extração de Texto:** Leitura integral com preservação de ordem.
- **Identificação de Metadados:** Extração automática de atributos básicos (ID, data, tipo, autor).
- **Segmentação (Chunking Estruturado):** Em vez de cortes fixos por caracteres, o sistema identifica a estrutura lógica do documento (ex: seções, capítulos, itens) para criar chunks que mantenham a unidade de sentido.
- **Enriquecimento Inicial (IA):** Geração de resumos curtos e palavras-chave para cada documento.
- **Idempotência:** Uso de registros (hashing/CSV) para evitar reprocessamento de arquivos inalterados.

### 2. Camada de Inteligência Contextual (Enrichment)
Este nível opera sobre os dados estruturados da Etapa 1 para criar uma "rede de conhecimento".
- **Resolução de Entidades:** Identificação de menções a outros documentos dentro do texto.
- **Mapeamento de Relações:** Criação de links bidirecionais (*backlinks*) entre documentos (ex: Documento A cita Documento B).
- **Consolidação de Status:** Propagação de efeitos baseada em relações (ex: se A altera ou revoga B, o status de B é atualizado automaticamente no sistema).
- **Indexação Temática:** Agrupamento de documentos por tópicos semânticos extraídos por LLM, permitindo navegação por assuntos sem depender de busca vetorial.

### 3. Camada de Distribuição e Apresentação
Foca na acessibilidade dos dados para humanos e máquinas.
- **Geração de Portal Estático:** Transformação dos JSONs enriquecidos em páginas navegáveis (HTML) com links interconectados.
- **Catálogo Moderno:** Interface de busca com filtros por metadados e tópicos gerados.

### 4. Camada de Busca Semântica (Vector Indexing)
Preparação técnica para a IA de conversação.
- **Embedding Denso:** Uso de modelos de alta performance (ex: BGE-M3) para converter chunks em vetores.
- **Metadata-Aware Chunks:** Cada vetor é armazenado com metadados ricos (título, resumo, tópicos, link original).
- **Banco Vetorial:** Armazenamento em bancos otimizados (ex: pgvector) com índices de proximidade (ex: IVFFlat).

### 5. Camada Conversacional (Orchestrated RAG)
A interface final de interação com o conhecimento.
- **Contextualização de Query:** Reescrita da pergunta do usuário com base no histórico da conversa para torná-la autossuficiente.
- **Recuperação Híbrida (Hybrid Retrieval):** Combinação de busca léxica (âncoras de metadados) com busca semântica (vetores).
- **Hard Grounding:** Portões de validação que garantem que a IA responda apenas com base nos trechos recuperados, exigindo citações diretas de fontes.

---
> [!NOTE]
> Este pipeline permite que o sistema não apenas "leia" documentos, mas entenda como eles se relacionam, permitindo consultas complexas sobre o impacto de um documento em outro.
