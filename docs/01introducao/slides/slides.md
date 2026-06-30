<!-- .slide: class="title-slide" -->

# Introdução

## Workshop · Section 1 · Step 01

Introdução à integração com modelos de IA em Java

<p class="small">Pressione <strong>F</strong> para tela cheia · <strong>ESC</strong> para visão geral · <strong>S</strong> para notas</p>

[quarkus.io/quarkus-workshop-langchain4j](https://quarkus.io/quarkus-workshop-langchain4j/section-1/step-01/)

---

## Agenda do Step 01

1. 🧠 **Conceitos** O que é Quarkus, LLM, LangChain4j
2. 🏗️ **Anatomia do projeto** Estrutura, dependências, configuração
3. 💻 **O código** As duas classes que fazem tudo acontecer
4. 🔄 **Memória & Statelessness** Como o chatbot lembra do contexto
5. 🚀 **Executando** `./mvnw quarkus:dev` e testando o bot
6. 🗺️ **Próximos passos** Para onde o workshop nos leva

<div class="destaque">
<strong>Objetivo:</strong> entender cada peça da aplicação, não apenas executar o tutorial.
</div>

---

<!-- .slide: class="section-slide" -->

# Parte 1

## Conceitos fundamentais

---

## O que é Quarkus?

**Quarkus** é um framework Java moderno, criado pela Red Hat, otimizado para **cloud-native** e **GraalVM**.

- ⚡ Inicia em **milissegundos** (vs. segundos do Spring/JEE clássico)
- 💾 Consome **pouca memória** (RSS frequentemente abaixo de 100 MB)
- 🔧 Suporte a **compilação nativa** com GraalVM (executável binário)
- 🔥 **Live coding** no `dev mode`: salvou o arquivo, recarregou a aplicação
- ↔️ Programação tradicional (imperativa) **e** reativa (Mutiny, Vert.x)

<div class="dica">
<strong>Por que importa para LLMs?</strong> Aplicações de IA fazem muitas chamadas HTTP a APIs externas (OpenAI). Quarkus gerencia isso de forma reativa e eficiente.
</div>

---

## O que é um LLM?

**LLM** = *Large Language Model* um modelo de IA treinado com bilhões de palavras para prever texto.

Exemplos populares:

- 🤖 **GPT-4o** (OpenAI) o que vamos usar neste workshop
- 🤖 **Claude** (Anthropic)
- 🦙 **Llama 3** (Meta) open source, roda local

**Características importantes:**

- 📝 Recebe texto, retorna texto (basicamente)
- 🔁 É **stateless**: cada chamada é independente, ele não lembra de nada
- 💰 Cobra por **tokens** (≈ pedaços de palavras) consumidos

---

## Limitação: LLMs são stateless

Um LLM **não tem memória própria**. Cada requisição é isolada:

```
Chamada 1:  "Meu nome é Clement"  →  LLM: "Olá, Clement!"
Chamada 2:  "Qual é meu nome?"    →  LLM: "Não sei, você não disse"
```

**Solução universal:** *reenviar todo o histórico* da conversa em cada chamada.

```
Chamada 2 (correta):
  [
    {role: "user",      content: "Meu nome é Clement"},
    {role: "assistant", content: "Olá, Clement!"},
    {role: "user",      content: "Qual é meu nome?"}
  ]
  →  LLM: "Seu nome é Clement"
```

<div class="destaque">
<strong>Quem mantém esse histórico?</strong> A sua aplicação. Vamos ver como Quarkus + LangChain4j fazem isso para você.
</div>

---

## O que é LangChain4j?

**LangChain4j** é uma biblioteca Java (port do LangChain de Python) que **abstrai** a complexidade de trabalhar com LLMs.

<div class="two-col">
<div class="col">
<h3>Sem LangChain4j:</h3>
<ul>
<li>🔧 Construir o JSON do request manualmente para uma API de um LLM</li>
<li>🔐 Gerenciar headers de autenticação</li>
<li>📋 Tratar o array <code>messages</code> com roles <code>user</code>/<code>assistant</code>/<code>system</code></li>
<li>🔄 Implementar retry, streaming, function calling, embeddings...</li>
</ul>
</div>
<div class="col">
<h3>Com LangChain4j:</h3>
<p>Você <strong>declara uma interface Java</strong> e a biblioteca cuida do resto. ✅</p>
</div>
</div>

---

## O que é Quarkus LangChain4j?

É a **extensão Quarkus** que integra o LangChain4j ao ecossistema CDI/Quarkus.

```xml
<dependency>
    <groupId>io.quarkiverse.langchain4j</groupId>
    <artifactId>quarkus-langchain4j-openai</artifactId>
</dependency>
```

**O que essa extensão faz por você:**

- ⚙️ Lê configuração de `application.properties` (chave de API, modelo, etc.)
- 🏗️ Gera **em build time** uma implementação concreta para cada interface anotada com `@RegisterAiService`
- 💉 Integra com CDI (`@Inject`, `@SessionScoped`, `@ApplicationScoped`)
- 📊 Fornece logging automático das requisições HTTP para a OpenAI
- 🔌 Funciona com **qualquer endpoint compatível com a API OpenAI** (vLLM, Podman AI Lab, LM Studio, Ollama com gateway)

---

<!-- .slide: class="section-slide" -->

# Parte 2

## Anatomia do projeto (step-01)

---

## Estrutura de diretórios

```
step-01/
├── pom.xml                       (dependências Maven)
├── mvnw, mvnw.cmd                (Maven Wrapper - já vem no projeto)
├── src/
│   ├── main/
│   │   ├── java/dev/langchain4j/quarkus/workshop/
│   │   │   ├── CustomerSupportAgent.java          (interface AI)
│   │   │   └── CustomerSupportAgentWebSocket.java (endpoint WebSocket)
│   │   └── resources/
│   │       ├── application.properties             (config Quarkus)
│   │       └── META-INF/resources/                (UI estática do chat)
│   └── test/
└── README.md
```

<div class="dica">
<strong>Apenas duas classes Java</strong> compõem todo o backend. O resto é configuração e UI estática.
</div>

---

## As duas dependências essenciais

No `pom.xml`:

```xml
<dependency>
    <groupId>io.quarkiverse.langchain4j</groupId>
    <artifactId>quarkus-langchain4j-openai</artifactId>
</dependency>

<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-websockets-next</artifactId>
</dependency>
```

**Por que duas?**

- 🤖 `quarkus-langchain4j-openai` fala com a OpenAI (ou qualquer endpoint compatível)
- 🌐 `quarkus-websockets-next` expõe o backend ao navegador via WebSocket (comunicação em tempo real, bidirecional)

---

## quarkus-websockets-next vs Jakarta WebSocket

Existem **duas** APIs WebSocket no ecossistema Java/Quarkus:

| Aspecto | Jakarta WebSocket (clássico) | quarkus-websockets-next |
|---|---|---|
| Pacote | `jakarta.websocket.*` | `io.quarkus.websockets.next.*` |
| Anotação de classe | `@ServerEndpoint("/path")` | `@WebSocket(path = "/path")` |
| Mensagem texto | `@OnMessage` | `@OnTextMessage` |
| Resposta | `session.sendText(msg)` via AsyncRemote | retornar `String` direto |
| Reativo | Não | Sim (Mutiny nativo) |

<div class="destaque">
O tutorial usa <strong>quarkus-websockets-next</strong>. É a API moderna, mais idiomática e integrada ao CDI.
</div>

---

## Configuração: application.properties

```properties
quarkus.langchain4j.openai.api-key=${OPENAI_API_KEY}
quarkus.langchain4j.openai.chat-model.model-name=gpt-4o
quarkus.langchain4j.openai.timeout=60s
quarkus.langchain4j.openai.log-requests=true
quarkus.langchain4j.openai.log-responses=true
```

**Pontos importantes:**

- 🔒 `${OPENAI_API_KEY}` substituição de variável de ambiente (NUNCA hardcode a chave)
- 📋 `log-requests`/`log-responses` habilitam o log do JSON enviado/recebido (essencial para entender o que está acontecendo)
- 🤖 `chat-model.model-name` qual modelo usar (gpt-4o, gpt-4o-mini, etc.)

---

## A variável de ambiente OPENAI_API_KEY

Antes de rodar a aplicação, exporte a chave:

```bash
export OPENAI_API_KEY=sk-proj-XXXXXXXXXXXXXXXXXXXXXXXX
```

Se esquecer, o Quarkus avisa de forma muito específica:

```
java.util.NoSuchElementException: SRCFG00011: Could not expand value
OPENAI_API_KEY in property quarkus.langchain4j.openai.api-key
```

<div class="dica">
<strong>Boa prática:</strong> use um <code>.envrc</code> (direnv) ou um arquivo <code>.env</code> fora do repositório Git. Nunca commite a chave.
</div>

---

## A UI estática (já vem pronta)

O diretório `src/main/resources/META-INF/resources/` contém uma página HTML pronta:

- 🌐 Servida automaticamente em `http://localhost:8080`
- 🤖 Mostra um ícone de **robô vermelho** no canto inferior direito
- 💬 Ao clicar, abre uma janela de chat
- 🔌 Conecta-se ao WebSocket `ws://localhost:8080/customer-support-agent`

**Você não precisa criar HTML para testar.** Basta subir a aplicação.

---

<!-- .slide: class="section-slide" -->

# Parte 3

## O código

---

## CustomerSupportAgent.java

A peça central da aplicação uma **interface**, não uma classe:

```java
package dev.langchain4j.quarkus.workshop;

import io.quarkiverse.langchain4j.RegisterAiService;
import jakarta.enterprise.context.SessionScoped;

@SessionScoped
@RegisterAiService
public interface CustomerSupportAgent {

    String chat(String userMessage);
}
```

**É isso. Não há classe de implementação.** Quarkus gera uma para você em build time.

---

## Decifrando a interface `@RegisterAiService`

```java
@SessionScoped
@RegisterAiService
public interface CustomerSupportAgent {
    String chat(String userMessage);
}
```

**`@RegisterAiService`** (do quarkus-langchain4j)

- 🏷️ Declara que esta interface é um *AI Service* uma abstração para conversar com um LLM
- 🏗️ O Quarkus **gera a implementação em build time** (compilação)
- ⚙️ A implementação gerada: monta o JSON, chama a API, parseia a resposta, gerencia tokens

---

## Decifrando a interface `@SessionScoped`

**`@SessionScoped`** (CDI / Jakarta EE)

- 📦 Define o ciclo de vida do bean: **vive enquanto durar a sessão**
- 🔌 Aqui, "sessão" = conexão WebSocket aberta
- 👤 Cada usuário conectado tem sua **própria instância** com seu **próprio histórico**
- 🗑️ Quando o usuário desconecta, o bean é destruído (e a memória, liberada)

---

## E o método chat?

```java
String chat(String userMessage);
```

**Convenções implícitas do quarkus-langchain4j:**

- 📥 O **único parâmetro** é tratado como a mensagem do usuário (role `user`)
- 📤 O **retorno** é a resposta gerada pelo LLM
- ✏️ O nome do método é **livre** poderia ser `responder()`, `falar()`, etc.

---

## E o método chat? o que não precisamos escrever

**O que NÃO precisamos escrever:**

- ❌ JSON do request
- ❌ Headers de autenticação
- ❌ Loop de mensagens com roles
- ❌ Tratamento de erros HTTP
- ❌ Retry / timeout

<div class="destaque">
Tudo isso é gerado em build time pela extensão. <strong>Magia? Não, geração automática de código.</strong>
</div>

---

## CustomerSupportAgentWebSocket.java

A "porta de entrada": recebe mensagens do navegador via WebSocket:

```java
package dev.langchain4j.quarkus.workshop;

import io.quarkus.websockets.next.OnOpen;
import io.quarkus.websockets.next.OnTextMessage;
import io.quarkus.websockets.next.WebSocket;

@WebSocket(path = "/customer-support-agent")
public class CustomerSupportAgentWebSocket {

    private final CustomerSupportAgent customerSupportAgent;

    public CustomerSupportAgentWebSocket(CustomerSupportAgent customerSupportAgent) {
        this.customerSupportAgent = customerSupportAgent;
    }

    @OnOpen
    public String onOpen() {
        return "Welcome to Miles of Smiles! How can I help you today?";
    }

    @OnTextMessage
    public String onTextMessage(String message) {
        return customerSupportAgent.chat(message);
    }
}
```

---

## Decifrando o WebSocket anotação e construtor

**`@WebSocket(path = "/customer-support-agent")`**

Expõe a classe como endpoint WebSocket em `ws://localhost:8080/customer-support-agent`.

**Construtor com parâmetro** (injeção via construtor)

- 💉 Quarkus injeta automaticamente o `CustomerSupportAgent` (que é `@SessionScoped`)
- 👤 Cada conexão WebSocket recebe **sua própria instância** do agente

---

## Decifrando o WebSocket mensagens

**`@OnOpen`** chamado quando o cliente conecta

- 📤 Retornar uma `String` envia essa string como **primeira mensagem** ao cliente
- 💬 Aqui: `"Welcome to Miles of Smiles! How can I help you today?"`

**`@OnTextMessage`** chamado a cada mensagem de texto recebida

- 📥 Recebe a `String` enviada pelo cliente
- 📤 Retornar uma `String` envia automaticamente como resposta

---

<!-- .slide: class="section-slide" -->

# Parte 4

## Memória e Statelessness

---

## Como nasce a memória?

O LLM **não** tem memória. Quem tem memória é o **bean `@SessionScoped`** gerado pela extensão.

A cada chamada de `chat(...)`:

1. 📥 A extensão **anexa** a nova mensagem do usuário ao histórico do bean
2. 📤 **Envia o histórico inteiro** para a OpenAI
3. 🤖 Recebe a resposta do LLM
4. 📋 **Anexa** essa resposta ao histórico (como role `assistant`)
5. ↩️ Retorna apenas a resposta para o seu código

```
Chamada 1:  [user: "M"]                                    → assistant: "A"
Chamada 2:  [user: "M", assistant: "A", user: "M2"]        → assistant: "A2"
Chamada 3:  [user: "M", assistant: "A", user: "M2",
             assistant: "A2", user: "M3"]                  → assistant: "A3"
```

A memória **cresce** com a conversa. (Vamos ver implicações disso em steps futuros.)

---

## Vendo a memória em ação

Conversa real do tutorial:

```
User: My name is Clement.
AI:   Hi Clement, nice to meet you.

User: What is my name?
AI:   Your name is Clement.
```

**O LLM "lembrou" do nome porque a aplicação reenviou todo o histórico** na segunda chamada.

Sem `@SessionScoped`, cada chamada criaria um histórico novo, e o bot esqueceria tudo a cada mensagem.

---

## Roles na conversa

| Role | Quem produz | Significado |
|---|---|---|
| `system` | Você (desenvolvedor) | Instrução geral ("seja um assistente educado em português") |
| `user` | Usuário final | Pergunta / mensagem do usuário |
| `assistant` | LLM | Resposta gerada pelo modelo |
| `tool` | Sistema | Resultado de uma function call (steps avançados) |

Neste Step 01, só vemos **`user`** e **`assistant`**. Mensagens `system` aparecem no Step 04.

---

<!-- .slide: class="section-slide" -->

# Parte 5

## Executando a aplicação

---

## Subindo o servidor com dev mode

Na raiz do projeto:

```bash
./mvnw quarkus:dev
```

**O que acontece:**

1. 📦 Maven baixa as dependências (primeira vez)
2. ⚡ Quarkus compila e inicia (em ~2s)
3. 🔥 Entra em **live coding**: salvar `.java` recarrega instantaneamente
4. 🛠️ Abre o **Dev UI** em `http://localhost:8080/q/dev`

---

## Subindo o servidor erro comum

**Erro comum:**

```
Permission denied: ./mvnw
```

<div class="alerta">
<strong>Solução:</strong> execute <code>chmod +x mvnw</code> para dar permissão de execução ao Maven Wrapper antes de rodar o comando novamente.
</div>

---

## Conversando com o bot

1. 🌐 Abra `http://localhost:8080`
2. 🤖 Clique no **ícone de robô vermelho** no canto inferior direito
3. ⌨️ Comece a digitar

```
Você:  Meu nome é Clement
Bot:   Hi Clement, nice to meet you.
Você:  Qual é o meu nome?
Bot:   Your name is Clement.
```

No terminal, observe os logs HTTP request/response em tempo real.

<div class="dica">
<strong>Live coding:</strong> mude o nome do bot no <code>@OnOpen</code>, salve, e a próxima conexão já mostra o novo texto. Sem rebuild!
</div>

---

## Recapitulando o que foi construído

<div class="flow">
  <div class="flow-node">
    <span class="node-icon">🌐</span>
    <span class="node-label">Browser</span>
    <span class="node-desc">UI estática<br>JS WebSocket</span>
  </div>
  <div class="flow-arrow">⟶</div>
  <div class="flow-node">
    <span class="node-icon">🔌</span>
    <span class="node-label">WebSocket</span>
    <span class="node-desc">@WebSocket<br>@OnTextMessage</span>
  </div>
  <div class="flow-arrow">⟶</div>
  <div class="flow-node">
    <span class="node-icon">🤖</span>
    <span class="node-label">AI Service</span>
    <span class="node-desc">@RegisterAiService<br>@SessionScoped</span>
  </div>
  <div class="flow-arrow">⟶</div>
  <div class="flow-node">
    <span class="node-icon">☁️</span>
    <span class="node-label">OpenAI API</span>
    <span class="node-desc">gpt-4o<br>api.openai.com</span>
  </div>
</div>

<div class="destaque" style="text-align:center;">
<strong>Total de código Java escrito por você: ~30 linhas.</strong>
</div>

---

<!-- .slide: class="section-slide" -->

# Parte 6

## Próximos passos

---

## O que vem depois? Section 1

O workshop completo tem **2 seções com 10 steps cada**. *Section 1 AI Apps:*

- 🌡️ Step 02 Parâmetros do modelo (temperature, top_p, ...)
- ⚡ Step 03 Streaming de respostas
- 💬 Step 04 System messages (instruções ao bot)
- 📚 Step 05 RAG (Retrieval Augmented Generation)
- 🔬 Step 06 Desconstruindo o RAG

---

## O que vem depois? Section 2

*Section 1 AI Apps (cont.):*

- 🔧 Step 07 Function calling e tools
- 🔗 Step 08 Model Context Protocol (MCP)
- 🛡️ Step 09 Guardrails
- 📊 Step 10 Observabilidade e tolerância a falhas

**Section 2 Agentic Workflows** agentes, supervisor, human-in-the-loop, agentes remotos (A2A), multimodalidade.

---

## Recursos & links

**Tutorial oficial deste step**

[quarkus.io/quarkus-workshop-langchain4j/section-1/step-01](https://quarkus.io/quarkus-workshop-langchain4j/section-1/step-01/)

**Documentação**

- 📖 [Quarkus](https://quarkus.io/)
- 📖 [Quarkus LangChain4j](https://docs.quarkiverse.io/quarkus-langchain4j/dev/)
- 📖 [LangChain4j (core)](https://docs.langchain4j.dev/)
- 📖 [Quarkus WebSockets Next](https://quarkus.io/guides/websockets-next-tutorial)

**Conceitos relacionados**

- 🔗 [Jakarta CDI - Session Scope](https://jakarta.ee/specifications/cdi/)
- 🔗 [OpenAI Chat Completions API](https://platform.openai.com/docs/api-reference/chat)
