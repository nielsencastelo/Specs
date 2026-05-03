# Engine de IA e Lógica de RAG: Detalhes de Implementação

Esta especificação detalha a implementação das camadas de Inteligência Artificial e os processos de Large Language Model (LLM) utilizados no pipeline de RAG Multinível.

## 1. Processos de Enriquecimento por LLM

O sistema utiliza LLMs em diferentes momentos do ciclo de vida do documento para extrair significado e contexto.

### A. Extração Estruturada (Doc Enrichment)
Para cada documento, o LLM é provocado a retornar um JSON estrito contendo:
- **Resumo Executivo:** Uma frase objetiva (até 200 caracteres) capturando o efeito prático do documento.
- **Tópicos/Assuntos:** Tags semânticas que descrevem o domínio do conhecimento (ex: "Saneamento", "Recursos Humanos").
- **Atores Atingidos:** Quem é o público-alvo ou entidade afetada.
- **Relações de Artigos:** Mapeamento granular de como seções específicas interagem com outros documentos (ex: "Artigo 5 altera Documento X").

### B. Fallbacks e Heurísticas
Para garantir robustez, o sistema implementa:
- **Validadores de Resumo:** Se a resposta do LLM for muito curta ou contiver apenas repetição do título, o sistema aplica um resumo baseado em heurísticas (primeira frase do texto ou preâmbulo).
- **Dicionário Controlado:** Uso de regex para extrair palavras-chave caso o LLM falhe ou não esteja disponível.

## 2. Orquestração da Consulta (Conversational RAG)

A camada de consulta não é um simples "pergunta-resposta", mas uma orquestra de três fases de LLM:

### Fase 1: Contextualização da Pergunta (Re-writer)
- **Objetivo:** Tornar a pergunta do usuário autossuficiente.
- **Lógica:** O LLM recebe o histórico da conversa e a nova pergunta. Ele deve gerar uma query que não dependa de pronomes ou referências passadas (ex: "Qual o prazo?" -> "Qual o prazo de validade da licença ambiental X no Documento Y?").

### Fase 2: Extração de Evidências (CoT Privado)
- **Objetivo:** Filtrar ruído antes da resposta final.
- **Lógica:** O LLM analisa os trechos recuperados do banco vetorial e extrai apenas "bullets" de evidências concretas, cada um obrigatoriamente referenciando a fonte (ex: `[chunk 3]`).

### Fase 3: Resposta Final com Grounding
- **Objetivo:** Gerar a resposta em linguagem natural com alta fidelidade.
- **Restrição Estrita:** Proibição de inferências por analogia ou conhecimento geral. A resposta deve ser baseada **exclusivamente** nos trechos fornecidos.

## 3. Hard Grounding: Garantia de Veracidade

Para evitar alucinações em domínios críticos, o sistema implementa "portões" (gates) de validação:

### I. Âncora Lexical (Lexical Anchor Gate)
Se o usuário pergunta sobre um documento específico (ex: "O que diz o Memorando 45/2023?"):
- O sistema gera variantes do nome do documento.
- Ele executa um filtro `ILIKE` no banco de dados.
- **Regra:** Se o documento específico não for encontrado nos trechos recuperados, a IA é instruída a declarar que não encontrou o documento, em vez de tentar adivinhar com base no conteúdo semanticamente similar de outros documentos.

### II. Validação de Citações
- Após gerar a resposta, o sistema verifica se os identificadores de citação (ex: `[chunk X]`) utilizados pelo LLM correspondem aos chunks que foram realmente enviados como contexto.
- Se houver citação de um chunk inexistente, a resposta é rejeitada ou marcada como insegura.

## 4. Estratégia de Embedding (Dense Retrieval)

- **Modelo:** Modelos multi-idioma de alta performance (ex: BAAI/bge-m3).
- **Prefixos de Query:** Adição de prefixos como `"Consulta: "` para alinhar a intenção da busca com o espaço vetorial de documentos.
- **Chunk Composition:** O texto enviado para embedding não é apenas o conteúdo do documento, mas um "super-texto" que inclui:
  `[Cabeçalho + Resumo + Tópicos + Palavras-Chave + Conteúdo]`
  Isso garante que metadados ricos ajudem na recuperação semântica.

---
> [!IMPORTANT]
> A filosofia do sistema é "Abstenção em vez de Alucinação". É preferível informar a falta de evidência do que fornecer uma resposta plausível mas incorreta.
