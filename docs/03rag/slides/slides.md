<!-- .slide: class="title-slide" -->

# RAG com Quarkus LangChain4j

## Workshop · Section 1 · Steps 05 e 06

Retrieval Augmented Generation — do EasyRAG às entranhas do padrão

<p class="small">Pressione <strong>F</strong> para tela cheia · <strong>ESC</strong> para visão geral · <strong>S</strong> para notas</p>

[quarkus.io/quarkus-workshop-langchain4j](https://quarkus.io/quarkus-workshop-langchain4j/section-1/step-05/)

---

## Agenda dos Steps 05 e 06

1. 🧠 **Por que RAG?** — O problema que o padrão resolve
2. 🏗️ **Arquitetura RAG** — Ingestão e Augmentação
3. ⚡ **Step 05 — EasyRAG** — Setup rápido com mínimo de código
4. 🔬 **Step 06 — Desconstruindo o RAG** — Embedding model, Vector Store, Ingestor e Retriever
5. 🔧 **RAG Avançado** — Customizando o Content Injector
6. 🗺️ **Próximos passos** — Para onde o workshop nos leva

<div class="destaque">
<strong>Objetivo:</strong> entender cada camada do RAG — primeiro via abstração (EasyRAG), depois desmontando peça por peça.
</div>

---

<!-- .slide: class="section-slide" -->

# Parte 1

## Por que RAG?

---

## O problema: LLMs têm conhecimento limitado

LLMs são treinados com enormes volumes de texto público — mas esse conhecimento tem **duas limitações críticas**:

- 📅 **Corte temporal** — o modelo não sabe de eventos recentes
- 🏢 **Conhecimento específico** — o modelo não conhece *seus* documentos, políticas, dados de negócio

**Exemplo real:**

```
Usuário:  Qual é a nossa política de cancelamento?
LLM:      Desculpe, não tenho informações sobre isso.
```

O modelo simplesmente **não tem** essa informação — ela não estava no treinamento.

---

## A solução: injetar contexto relevante no prompt

**RAG = Retrieval Augmented Generation**

A ideia central é simples:

1. 📥 **Buscar** os trechos mais relevantes da sua base de conhecimento
2. 📎 **Injetar** esses trechos diretamente no prompt enviado ao LLM
3. 🤖 O LLM **responde com base no contexto fornecido**

```
Sem RAG:
  "Qual é a política de cancelamento?" → LLM não sabe

Com RAG:
  "Qual é a política de cancelamento?
   Responda usando: reservas canceláveis 11 dias antes,
   reservas < 4 dias não permitem cancelamento" → LLM responde com precisão
```

---

## As duas fases do RAG

O padrão RAG é composto de **duas fases distintas**:

<div class="two-col">
<div class="col">
<h3>📥 Ingestão (Ingestion)</h3>
<ul>
<li>Lê os documentos</li>
<li>Divide em segmentos (<em>chunks</em>)</li>
<li>Transforma em vetores (<em>embeddings</em>)</li>
<li>Armazena na base vetorial (<em>vector store</em>)</li>
</ul>
</div>
<div class="col">
<h3>📤 Augmentação (Augmentation)</h3>
<ul>
<li>Recebe a pergunta do usuário</li>
<li>Transforma em vetor (embedding)</li>
<li>Busca os segmentos mais similares</li>
<li>Injeta no prompt enviado ao LLM</li>
</ul>
</div>
</div>

<div class="destaque">
<strong>Ingestão</strong> acontece uma vez (ou quando os documentos mudam). <strong>Augmentação</strong> acontece em cada pergunta.
</div>

---

<!-- .slide: class="section-slide" -->

# Parte 2

## Arquitetura RAG

---

## Fase 1: Ingestão em detalhes

```
Documentos (PDF, TXT, DOCX...)
        ↓
   Document Loader
        ↓
   Document Splitter   ←  max-segment-size, max-overlap-size
        ↓
  Embedding Model      ←  transforma texto → vetor numérico
        ↓
   Vector Store        ←  armazena (texto + embedding)
```

- 📄 **Document Loader** — lê arquivos do disco, web, banco de dados...
- ✂️ **Document Splitter** — divide em pedaços menores com sobreposição
- 🔢 **Embedding Model** — converte texto em vetor numérico de alta dimensão
- 🗄️ **Vector Store** — banco de dados especializado em vetores

---

## O que é um Embedding?

**Embedding** é uma representação numérica de texto em forma de vetor.

Textos **semanticamente similares** geram vetores **geometricamente próximos**:

```
"política de cancelamento"  → [0.12, -0.45, 0.87, ..., 0.33]  (384 números)
"posso cancelar minha reserva?" → [0.14, -0.41, 0.89, ..., 0.31]  ← próximo!
"clima em São Paulo"        → [-0.72, 0.18, -0.05, ..., 0.61]  ← distante
```

**Por que isso importa?**

A busca na base vetorial usa **similaridade coseno** — encontra os textos
numericamente mais próximos do vetor da pergunta, independente das palavras exatas usadas.

---

## O que é Segmentação (Chunking)?

Documentos longos precisam ser **divididos em pedaços menores** antes de virar embeddings.

**Por que não gerar um embedding do documento inteiro?**

- 📏 Modelos de embedding têm **limite de tokens**
- 🎯 Segmentos menores têm **mais precisão semântica**
- 💰 Quanto menor o segmento, **menos tokens** consomem o contexto do LLM

**Parâmetros importantes:**

| Parâmetro | Significado |
|---|---|
| `max-segment-size` | Máximo de tokens por segmento |
| `max-overlap-size` | Tokens de sobreposição entre segmentos consecutivos |

A **sobreposição** garante que informações no limite entre dois segmentos não se percam.

---

## Fase 2: Augmentação em detalhes

```
Pergunta do usuário
        ↓
   Embedding Model    ←  mesma pergunta → vetor
        ↓
   Vector Store       ←  busca os N mais similares (cosine similarity)
        ↓
   Content Injector   ←  monta o prompt aumentado
        ↓
      LLM             ←  recebe pergunta + contexto
        ↓
   Resposta final
```

<div class="dica">
<strong>Regra crítica:</strong> o embedding model usado na ingestão e na augmentação <em>deve ser o mesmo</em>. Modelos diferentes geram vetores em espaços incompatíveis — a busca não funcionaria.
</div>

---

<!-- .slide: class="section-slide" -->

# Parte 3

## Step 05 — EasyRAG

---

## O que é EasyRAG?

**EasyRAG** é uma abstração de alto nível que esconde toda a complexidade do RAG.

Basicamente: você coloca seus dados em um diretório configurado e *voilà* — o Quarkus cuida de todo o pipeline.

```properties
quarkus.langchain4j.easy-rag.path=src/main/resources/rag
quarkus.langchain4j.easy-rag.max-segment-size=100
quarkus.langchain4j.easy-rag.max-overlap-size=25
quarkus.langchain4j.easy-rag.max-results=3
```

**O que o EasyRAG faz automaticamente:**

- 📄 Lê os arquivos do diretório configurado
- ✂️ Divide em segmentos (chunking)
- 🔢 Gera embeddings (via OpenAI por padrão)
- 🗄️ Armazena em vector store in-memory
- 🔎 Configura o retriever e o augmentor

**Não há classe Java para escrever.** EasyRAG implementa todo o pipeline.

---

## Adicionando a dependência EasyRAG

No `pom.xml`:

```xml
<dependency>
    <groupId>io.quarkiverse.langchain4j</groupId>
    <artifactId>quarkus-langchain4j-easy-rag</artifactId>
</dependency>
```

Ou via terminal:

```bash
./mvnw quarkus:add-extension -Dextension=easy-rag
```

<div class="dica">
<strong>Dev mode:</strong> ao adicionar a dependência com a aplicação rodando, ela reinicia automaticamente — mas ainda não funciona até que você adicione as propriedades de configuração.
</div>

---

## Adicionando os dados: o arquivo de termos

Crie o diretório e o arquivo de conhecimento:

```
src/main/resources/rag/miles-of-smiles-terms-of-use.txt
```

Conteúdo (resumo):

```
Miles of Smiles Car Rental Services Terms of Use

4. Cancellation Policy
4.1 Reservations can be cancelled up to 11 days prior
    to the start of the booking period.
4.2 If the booking period is less than 4 days,
    cancellations are not permitted.
...
```

<div class="destaque">
EasyRAG suporta TXT, PDF, Word e outros formatos. Basta colocar os arquivos no diretório configurado.
</div>

---

## Configurando o EasyRAG

Adicione ao `src/main/resources/application.properties`:

```properties
quarkus.langchain4j.easy-rag.path=src/main/resources/rag
quarkus.langchain4j.easy-rag.max-segment-size=100
quarkus.langchain4j.easy-rag.max-overlap-size=25
quarkus.langchain4j.easy-rag.max-results=3
```

**Significado de cada parâmetro:**

| Propriedade | Valor | Significado |
|---|---|---|
| `path` | `src/main/resources/rag` | Onde estão os documentos |
| `max-segment-size` | `100` | Máx. de tokens por segmento |
| `max-overlap-size` | `25` | Sobreposição entre segmentos |
| `max-results` | `3` | Qtde de segmentos recuperados por busca |

---

## Vendo a ingestão nos logs

Ao iniciar a aplicação, você verá no console:

```
INFO  [io.qua.lan.eas.run.EasyRagIngestor] Ingesting documents from path:
      src/main/resources/rag, path matcher = glob:**, recursive = true
INFO  [io.qua.lan.eas.run.EasyRagIngestor] Ingested 1 files as 8 documents
```

- 📄 `1 file` — o arquivo TXT que criamos
- 📋 `8 documents` — o arquivo foi dividido em 8 segmentos de até 100 tokens

O EasyRAG usou automaticamente o **embedding model da OpenAI** para vetorizar os segmentos e os armazenou em um **store in-memory**.

---

## Inspecionando a Vector Store no Dev UI

1. 🌐 Abra `http://localhost:8080/q/dev-ui`
2. 🔍 Localize o tile **LangChain4j Core**
3. 🗄️ Clique em **Embedding store**
4. 🔎 Na seção *Search for relevant embeddings*, digite `Cancellation` e clique em **Search**

Você verá os segmentos mais similares à palavra buscada, junto com o **score de similaridade** (quanto maior, mais relevante).

<div class="dica">
<strong>Ferramenta valiosa para depuração:</strong> use o Dev UI para verificar se seus documentos foram ingeridos corretamente antes de testar o chatbot.
</div>

---

## O RAG em ação: testando o chatbot

Abra `http://localhost:8080` e pergunte:

```
What can you tell me about your cancellation policy?
```

Observe nos logs o prompt **aumentado** que chega ao LLM:

```json
{
  "role": "user",
  "content": "What can you tell me about your cancellation policy?
  
  Answer using the following information:
  4. Cancellation Policy
  4.1 Reservations can be cancelled up to 11 days prior...
  4.2 If the booking period is less than 4 days,
      cancellations are not permitted.
  ..."
}
```

O LLM recebeu a pergunta **mais o contexto relevante** extraído do nosso documento.

---

<!-- .slide: class="section-slide" -->

# Parte 4

## Step 06 — Desconstruindo o RAG

---

## Por que desconstruir?

EasyRAG é ótimo para começar, mas em produção você vai precisar de:

- 🧩 **Embedding model próprio** — rodar localmente, sem enviar dados a APIs externas
- 🗄️ **Vector store persistente** — não perder os índices a cada restart
- 🎛️ **Controle fino** sobre ingestão, retrieval e augmentação
- 📐 **Customizações** — filtros, múltiplos retrievers, prompt personalizado

O Step 06 remove o EasyRAG e implementa cada peça manualmente.

---

## Limpeza: removendo o EasyRAG

Remova do `pom.xml`:

```xml
<!-- REMOVER -->
<dependency>
    <groupId>io.quarkiverse.langchain4j</groupId>
    <artifactId>quarkus-langchain4j-easy-rag</artifactId>
</dependency>
```

Remova do `application.properties`:

```properties
# REMOVER
quarkus.langchain4j.easy-rag.path=src/main/resources/rag
quarkus.langchain4j.easy-rag.max-segment-size=100
quarkus.langchain4j.easy-rag.max-overlap-size=25
quarkus.langchain4j.easy-rag.max-results=3
```

Ou via terminal: `./mvnw quarkus:remove-extension -Dextension=easy-rag`

---

## Peça 1: Embedding Model local (BGE-Small-EN)

Em vez de usar o modelo de embedding da OpenAI (que envia dados para a nuvem), usamos um modelo **que roda localmente**:

```xml
<dependency>
    <groupId>dev.langchain4j</groupId>
    <artifactId>langchain4j-embeddings-bge-small-en-q</artifactId>
</dependency>
```

E no `application.properties`:

```properties
quarkus.langchain4j.embedding-model.provider=\
  dev.langchain4j.model.embedding.onnx.bgesmallenq.BgeSmallEnQuantizedEmbeddingModel
```

**Características do BGE-Small-EN:**

- 🏠 Roda localmente (ONNX Runtime) — **zero dados enviados à nuvem**
- 📐 Gera vetores de **384 dimensões**
- ⚡ Modelo pequeno e rápido — ideal para desenvolvimento e edge computing

---

## Peça 2: Vector Store com PostgreSQL pgVector

Em vez do store in-memory do EasyRAG, usamos **PostgreSQL com a extensão pgVector**:

```xml
<dependency>
    <groupId>io.quarkiverse.langchain4j</groupId>
    <artifactId>quarkus-langchain4j-pgvector</artifactId>
</dependency>
```

E no `application.properties`:

```properties
quarkus.langchain4j.pgvector.dimension=384
```

**Por que `dimension=384`?** É o tamanho do vetor gerado pelo BGE-Small-EN. O banco precisa saber o tamanho antecipadamente para criar as colunas corretas.

<div class="dica">
<strong>Dev Services:</strong> Quarkus sobe automaticamente um container PostgreSQL em dev mode. Certifique-se de ter Docker ou Podman instalado.
</div>

---

## Peça 3: O Ingestor

Crie a classe `RagIngestion.java`:

```java
@ApplicationScoped
public class RagIngestion {

    public void ingest(@Observes StartupEvent ev,
                       EmbeddingStore store,
                       EmbeddingModel embeddingModel,
                       @ConfigProperty(name = "rag.location") Path documents) {

        store.removeAll(); // limpa o store a cada restart (só em demo)

        List<Document> list =
            FileSystemDocumentLoader.loadDocumentsRecursively(documents);

        EmbeddingStoreIngestor ingestor = EmbeddingStoreIngestor.builder()
            .embeddingStore(store)
            .embeddingModel(embeddingModel)
            .documentSplitter(recursive(100, 25,
                new HuggingFaceTokenCountEstimator()))
            .build();

        ingestor.ingest(list);
        Log.info("Documents ingested successfully");
    }
}
```

---

## Decifrando o Ingestor

```java
@Observes StartupEvent ev
```
→ O método roda **automaticamente ao iniciar a aplicação** (evento CDI de lifecycle do Quarkus)

```java
EmbeddingStore store, EmbeddingModel embeddingModel
```
→ Quarkus **injeta automaticamente** os beans — o `PgVectorEmbeddingStore` e o `BgeSmallEnQuantizedEmbeddingModel`

```java
@ConfigProperty(name = "rag.location") Path documents
```
→ Lê o caminho dos documentos da propriedade `rag.location` definida no `application.properties`

```java
recursive(100, 25, new HuggingFaceTokenCountEstimator())
```
→ Splitter recursivo: 100 tokens por segmento, 25 de sobreposição, contagem via HuggingFace

<div class="alerta">
<strong>Importante (do tutorial):</strong> splitter, segment size e overlap são cruciais para a precisão do RAG. Não há solução única — é preciso experimentar para cada caso de uso.
</div>

---

## Alternativa: In-Memory Store (sem Docker/Podman)

Se não for possível rodar Dev Services com Docker, o tutorial oferece um fallback:

**1.** Remova a dependência `quarkus-langchain4j-pgvector` do `pom.xml`

**2.** Crie a classe `InMemoryEmbeddingStoreProvider`:

```java
@ApplicationScoped
public class InMemoryEmbeddingStoreProvider {

    @Produces
    @ApplicationScoped
    EmbeddingStore embeddingStore() {
        return new InMemoryEmbeddingStore<>();
    }
}
```

O `RagIngestion` funciona normalmente — ele recebe qualquer implementação de `EmbeddingStore` via CDI.

<div class="alerta">
<strong>Atenção (do tutorial):</strong> solução de emergência apenas. Os dados são perdidos a cada restart. Use pgVector sempre que possível.
</div>

---

## Peça 4: O Retriever e o Augmentor

Crie a classe `RagRetriever.java` — note: **sem** anotação de escopo na classe, apenas no método producer:

```java
public class RagRetriever {   // sem @ApplicationScoped na classe!

    @Produces
    @ApplicationScoped
    public RetrievalAugmentor create(EmbeddingStore store,
                                    EmbeddingModel model) {

        var contentRetriever = EmbeddingStoreContentRetriever.builder()
            .embeddingModel(model)
            .embeddingStore(store)
            .maxResults(3)
            .build();

        return DefaultRetrievalAugmentor.builder()
            .contentRetriever(contentRetriever)
            .build();
    }
}
```

---

## Decifrando o Retriever

**`EmbeddingStoreContentRetriever`**

- Recebe a pergunta do usuário
- Gera o embedding da pergunta (usando o **mesmo modelo** do ingestor)
- Busca no vector store os `maxResults=3` segmentos mais similares

**`DefaultRetrievalAugmentor`**

- Recebe os segmentos recuperados pelo retriever
- Injeta esses segmentos no prompt do usuário antes de chamar o LLM
- É produzido como bean CDI (`@Produces @ApplicationScoped`) — o Quarkus LangChain4j o detecta e o aplica automaticamente ao AI Service

<div class="destaque">
<strong>Regra de ouro:</strong> use o <em>exato mesmo</em> embedding model no ingestor e no retriever. Modelos diferentes geram espaços vetoriais incompatíveis.
</div>

---

## Resumo: EasyRAG vs. RAG Manual

| Aspecto | EasyRAG (Step 05) | RAG Manual (Step 06) |
|---|---|---|
| Embedding model | OpenAI (nuvem) | BGE-Small-EN (local) |
| Vector store | In-memory | PostgreSQL pgVector |
| Ingestor | Automático | `RagIngestion.java` |
| Retriever/Augmentor | Automático | `RagRetriever.java` |
| Configuração | ~4 propriedades | ~6 propriedades + 2 classes |
| Controle | Baixo | Alto |
| Privacidade dos dados | Dados vão à nuvem | Dados ficam locais |

---

<!-- .slide: class="section-slide" -->

# Parte 5

## RAG Avançado — Customizando o Content Injector

---

## O prompt padrão do RAG

Por padrão, o `DefaultRetrievalAugmentor` injeta os segmentos assim:

```
<pergunta do usuário>
Answer using the following information:
<segmento 1>
<segmento 2>
<segmento 3>
```

Esse formato funciona bem — mas e se você quiser algo diferente?

<div class="dica">
<strong>Por que customizar?</strong> Diferentes LLMs respondem melhor a diferentes formatos de prompt. Ajustar o injector pode melhorar a qualidade das respostas.
</div>

---

## Criando um Content Injector customizado

Edite o método `create` em `RagRetriever.java`:

```java
return DefaultRetrievalAugmentor.builder()
    .contentRetriever(contentRetriever)
    .contentInjector(new ContentInjector() {
        @Override
        public UserMessage inject(List<Content> list,
                                  ChatMessage chatMessage) {
            StringBuffer prompt = new StringBuffer(
                ((UserMessage) chatMessage).singleText());

            prompt.append("\nPlease, only use the following information:\n");

            list.forEach(content ->
                prompt.append("- ")
                      .append(content.textSegment().text())
                      .append("\n")
            );

            return new UserMessage(prompt.toString());
        }
    })
    .build();
```

---

## Resultado do injector customizado

O log do tutorial mostra exatamente o prompt que chega à OpenAI:

```json
{
  "role": "user",
  "content": "What can you tell me about your cancellation policy?
Please, only use the following information:
- 4. Cancellation Policy
- 4. Cancellation Policy 4.1 Reservations can be cancelled up to 11 days
  prior to the start of the booking period.
- booking period. 4.2 If the booking period is less than 4 days,
  cancellations are not permitted."
}
```

**O que mudou em relação ao padrão:**

- Instrução mais diretiva: `"Please, only use the following information"`
- Segmentos listados com `- ` em vez de texto corrido
- O tutorial ressalta: este injector não muda o *comportamento*, mas mostra como **customizar** o padrão para qualquer necessidade

---

## Outras possibilidades de customização do RAG

O tutorial destaca que o RAG pode ser extendido muito além do que vimos:

- 🔢 **Diferentes embedding models** — trocar o BGE-Small-EN por outro modelo local ou remoto
- 🗄️ **Diferentes vector stores** — Redis, Infinispan, Chroma, e muitos outros além do pgVector
- 🔎 **Múltiplos retrievers** — combinar resultados de fontes diferentes
- 🎯 **Min Score Filter** — só injetar segmentos com similaridade acima de um limiar
- 🏷️ **Metadata Filter** — filtrar segmentos por categoria, data, departamento, etc.
- ✍️ **Content Injector customizado** — como acabamos de ver, controle total sobre o formato do prompt

<div class="dica">
O tutorial apresenta um diagrama chamado <em>Advanced Augmentation</em> mostrando que o pipeline pode ter múltiplas etapas de retrieval, re-ranking e injeção configuráveis.
</div>

---

<!-- .slide: class="section-slide" -->

# Parte 6

## Próximos passos

---

## O que foi construído nos Steps 05 e 06

<div class="flow">
  <div class="flow-node">
    <span class="node-icon">📄</span>
    <span class="node-label">Documentos</span>
    <span class="node-desc">TXT, PDF, DOCX<br>no diretório rag/</span>
  </div>
  <div class="flow-arrow">⟶</div>
  <div class="flow-node">
    <span class="node-icon">🔢</span>
    <span class="node-label">Embeddings</span>
    <span class="node-desc">BGE-Small-EN<br>local/OpenAI</span>
  </div>
  <div class="flow-arrow">⟶</div>
  <div class="flow-node">
    <span class="node-icon">🗄️</span>
    <span class="node-label">Vector Store</span>
    <span class="node-desc">pgVector<br>ou in-memory</span>
  </div>
  <div class="flow-arrow">⟶</div>
  <div class="flow-node">
    <span class="node-icon">🤖</span>
    <span class="node-label">LLM + RAG</span>
    <span class="node-desc">Prompt aumentado<br>resposta precisa</span>
  </div>
</div>

<div class="destaque" style="text-align:center;">
<strong>RAG é o padrão mais adotado em aplicações de IA empresariais hoje.</strong>
</div>

---

## O que vem depois?

Os próximos steps da **Section 1 — AI Apps**:

- 🔧 **Step 07** — Function calling e tools — o LLM chama funções Java da sua aplicação
- 🔗 **Step 08** — Model Context Protocol (MCP) — protocolo padronizado para tools
- 🛡️ **Step 09** — Guardrails — validar e filtrar entradas e saídas do LLM
- 📊 **Step 10** — Observabilidade e tolerância a falhas — métricas, traces, circuit breaker

E depois, a **Section 2 — Agentic Workflows**: agentes autônomos, supervisor pattern, human-in-the-loop, agentes remotos (A2A), multimodalidade.

---

## Conceitos-chave para fixar

| Conceito | Definição curta |
|---|---|
| **RAG** | Injetar contexto relevante no prompt para guiar o LLM |
| **Embedding** | Representação numérica vetorial de texto |
| **Chunking** | Divisão de documentos em segmentos menores |
| **Vector Store** | Banco de dados otimizado para busca por similaridade |
| **Cosine Similarity** | Métrica para comparar quão "próximos" dois vetores são |
| **EasyRAG** | Abstração de alto nível que automatiza todo o pipeline RAG |
| **Content Injector** | Componente que formata como os segmentos entram no prompt |

---

## Recursos & links

**Tutorial oficial**

- 📖 [Step 05 — EasyRAG](https://quarkus.io/quarkus-workshop-langchain4j/section-1/step-05/)
- 📖 [Step 06 — Deconstructing RAG](https://quarkus.io/quarkus-workshop-langchain4j/section-1/step-06/)

**Documentação**

- 📖 [Quarkus LangChain4j — EasyRAG](https://docs.quarkiverse.io/quarkus-langchain4j/dev/rag-easy-rag.html)
- 📖 [Quarkus LangChain4j — pgVector Store](https://docs.quarkiverse.io/quarkus-langchain4j/dev/rag-pgvector-store.html)
- 📖 [BGE-Small-EN no HuggingFace](https://huggingface.co/neuralmagic/bge-small-en-v1.5-quant)

**Conceitos**

- 🔗 [IBM — O que é RAG](https://research.ibm.com/blog/retrieval-augmented-generation-RAG)
- 🔗 [Cosine Similarity — Wikipedia](https://en.wikipedia.org/wiki/Cosine_similarity)
