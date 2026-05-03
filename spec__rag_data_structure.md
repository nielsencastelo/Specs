# Estrutura de Dados e Índices: Esquema de Informação

Esta especificação define o formato de dados utilizado para manter a interoperabilidade entre as 5 etapas do pipeline RAG Multinível.

## 1. Formato do Documento Estruturado (JSON)

Cada documento original é convertido em um arquivo JSON que serve como a "Fonte da Verdade" (Single Source of Truth) para o portal e para a IA.

### Campos Principais:
- `key`: Identificador único slugificado (ex: `relatorio-vendas-2023`).
- `tipo`, `numero`, `ano`, `data`: Metadados estruturados de identificação.
- `resumo`: Texto curto gerado por IA para visualização rápida.
- `palavras_chave`: Lista de termos técnicos extraídos.
- `assuntos`: Tópicos semânticos (enriquecimento etapa 2).
- `vigencia`: Status de validade do documento.
- `mentions`: Lista de referências a outros documentos encontradas no texto.
- `links_in` / `links_out`: Mapeamento de relacionamentos bidirecionais (resolvido na etapa 2).

### Segmentação de Conteúdo (`artigos` / `seções`):
O conteúdo é dividido em uma lista de objetos:
```json
{
  "artigos": [
    {
      "artigo": "1",
      "texto": "Conteúdo integral da seção...",
      "situacao": "vigente",
      "historico": [
        {"tipo": "alterado_por", "ato": "doc-xyz-2024", "prova": "Trecho que prova a alteração"}
      ]
    }
  ]
}
```

## 2. Índices Auxiliares (Serverless Search)

Para evitar a necessidade de um banco de dados relacional complexo para buscas básicas, o sistema gera índices em JSON:

### A. Índice de Tópicos (`topics_index.json`)
Mapeia cada assunto para os documentos que o tratam.
```json
{
  "recursos humanos": [
    {"key": "doc-001-2023"},
    {"key": "doc-045-2024"}
  ]
}
```

### B. Índice de Relações (`relations_index.json`)
Armazena a rede de impacto entre documentos (quem mexe em quem).
```json
[
  {
    "src_key": "doc-A",
    "tgt_key": "doc-B",
    "acao": "revoga",
    "src_artigo": "10",
    "tgt_artigo": "3"
  }
]
```

## 3. Esquema do Banco Vetorial (pgvector)

Para a Etapa 5 (Chat), os dados são projetados em uma estrutura vetorial:

- **Tabela `docs`**: Armazena metadados globais e caminho do JSON original.
- **Tabela `doc_chunks`**: Armazena os fragmentos de texto e seus respectivos embeddings (vetores).
- **Tabela `embed_control`**: Controle de versão para garantir que, se o modelo de IA mudar ou o JSON for atualizado, os vetores sejam recalculados.

---
> [!TIP]
> A separação entre JSONs (para portal e metadados) e Banco Vetorial (para busca semântica) garante que o sistema seja resiliente e fácil de depurar.
