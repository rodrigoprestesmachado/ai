<!-- .slide: class="title-slide" -->

# LLM Configuration and Streaming

## Workshop · Section 1 · Step 02, 03 e 04

<p class="small">Pressione <strong>F</strong> para tela cheia · <strong>ESC</strong> para visão geral · <strong>S</strong> para notas</p>

[quarkus.io/quarkus-workshop-langchain4j](https://quarkus.io/quarkus-workshop-langchain4j)

---

<!-- .slide: class="section-slide" -->

## STEP 2 — Parâmetros do Modelo

> Como configurar o comportamento do LLM via `application.properties`

### A Configuração do Modelo

A aplicação usa o arquivo `application.properties` para configurar o modelo de linguagem (LLM). As principais chaves de configuração são:

- **`api-key`** — Chave de acesso à API (via variável de ambiente)
- **`model-name`** — Qual modelo será utilizado (ex: `gpt-4o`)
- **`log-requests` / `log-responses`** — Ativar logs no terminal para debug

```properties
quarkus.langchain4j.openai.api-key=${OPENAI_API_KEY}
quarkus.langchain4j.openai.chat-model.model-name=gpt-4o
quarkus.langchain4j.openai.chat-model.log-requests=true
quarkus.langchain4j.openai.chat-model.log-responses=true
quarkus.langchain4j.timeout=1m
```

---

<!-- .slide: class="section-slide" -->

### Temperature — Criatividade do Modelo

**Propriedade:** `quarkus.langchain4j.openai.chat-model.temperature`

Controla o quanto o modelo é "criativo" ou "conservador" nas respostas.

| Valor | Comportamento |
|-------|--------------|
| `0.0` | Determinístico (respostas previsíveis) |
| `1.0` | Balanceado (recomendado para chatbots) |
| `2.0` | Caótico (pode gerar lixo!) |

> **Dica para o workshop:** Teste `temperature=0.1` perguntando sobre um pôr do sol, depois `temperature=1.5`. Observe a diferença no estilo das respostas!

---

<!-- .slide: class="section-slide" -->

### Max Tokens — Limite de Resposta

**Propriedade:** `quarkus.langchain4j.openai.chat-model.max-completion-tokens`

Define o número máximo de tokens que o modelo pode gerar na resposta.

**O que é um token?**  
Tokens são as menores unidades de texto — não são palavras exatas! Cada modelo tem sua tokenização. Exemplo: `"Hello, world!"` = 4 tokens.

```properties
# Limitar a 20 tokens:
max-completion-tokens=20

# Para o workshop:
max-completion-tokens=1000
```

> **Atenção:** Defina `max-completion-tokens=20` e veja a resposta ser cortada no meio! Depois ajuste para `1000` para continuar o workshop.

---

<!-- .slide: class="section-slide" -->

### Frequency Penalty — Evitando Repetições

**Propriedade:** `quarkus.langchain4j.openai.chat-model.frequency-penalty`

Define o quanto o modelo deve evitar repetir as mesmas palavras e frases.

| Valor | Comportamento | Resultado |
|-------|--------------|-----------|
| `0`   | Sem penalidade — o modelo pode repetir livremente | `hedgehog hedgehog hedgehog...` |
| `2`   | Penalidade máxima — evita fortemente repetições (com penalidade muito alta pode gerar texto sem sentido!) | `hedgehog... porcupine? spiky creature...` |

> **Experimento:** Peça ao modelo "Repeat the word hedgehog 50 times" com `penalty=2` e depois com `penalty=0`.

---

<!-- .slide: class="section-slide" -->

### Configuração Final do Step 2

Valores recomendados para o workshop:

| Parâmetro | Valor | Descrição |
|-----------|-------|-----------|
| `temperature` | `1.0` | Balanceado: criativo mas coerente |
| `max-completion-tokens` | `1000` | Permite respostas mais longas |
| `frequency-penalty` | `0` | Sem penalidade de repetição |

```properties
quarkus.langchain4j.openai.api-key=${OPENAI_API_KEY}
quarkus.langchain4j.openai.chat-model.model-name=gpt-4o
quarkus.langchain4j.openai.chat-model.log-requests=true
quarkus.langchain4j.openai.chat-model.log-responses=true
quarkus.langchain4j.openai.chat-model.temperature=1.0
quarkus.langchain4j.openai.chat-model.max-completion-tokens=1000
quarkus.langchain4j.openai.chat-model.frequency-penalty=0
```

---

<!-- .slide: class="section-slide" -->

## STEP 3 — Respostas em Streaming

> Enviando a resposta do LLM em tempo real, token por token

### O Problema: Espera pela Resposta Completa

**Fluxo sem streaming:**

```
Usuário envia mensagem  →  Servidor aguarda o LLM  →  LLM gera TUDO  →  Usuário recebe tudo
```

**Problemas desta abordagem:**

- O cliente fica esperando sem ver nada (só o "..." piscando)
- O servidor precisa manter toda a resposta em memória antes de enviar
- Para respostas longas (500 palavras), a espera pode ser frustrante

**Solução:** Streaming — enviar a resposta token por token, conforme o LLM vai gerando!

---

<!-- .slide: class="section-slide" -->

### Mudança 1: Retorno `Multi<String>` no AI Service

`Multi` é um tipo da biblioteca Mutiny que representa um fluxo (stream) de itens.

- **Stream de strings** — Cada item é um fragmento da resposta (token)
- **Assíncrono** — Os itens chegam conforme o LLM gera
- **Finito** — O stream termina quando a resposta acaba
- **Back-pressure** — Controla o fluxo, evitando sobrecarga

**ANTES:**
```java
@SessionScoped
@RegisterAiService
public interface CustomerSupportAgent {
    String chat(String msg);
}
```

**DEPOIS** (`String` → `Multi<String>`):
```java
@SessionScoped
@RegisterAiService
public interface CustomerSupportAgent {
    Multi<String> chat(String msg);
}
```

---

<!-- .slide: class="section-slide" -->

### Mudança 2: Atualizando o WebSocket

O endpoint WebSocket precisa retornar o `Multi<String>` ao invés de aguardar a `String` completa.

**ANTES — retorna String (bloqueia até terminar):**
```java
@OnTextMessage
public String onTextMessage(String message) {
    return customerSupportAgent.chat(message);
}
```

**DEPOIS — retorna Multi<String> (streaming!):**
```java
@OnTextMessage
public Multi<String> onTextMessage(String message) {
    return customerSupportAgent.chat(message);
}
```

> **Por que funciona sem mais mudanças?** O Quarkus WebSockets entende nativamente o tipo `Multi<String>`! Ele envia cada token ao cliente assim que chega do LLM — sem configuração extra.

---

<!-- .slide: class="section-slide" -->

### Testando o Streaming

Prompt para testar:
```
Tell me a story containing 500 words
```

| SEM Streaming (`String`) | COM Streaming (`Multi<String>`) |
|--------------------------|--------------------------------|
| 1. Usuário envia mensagem | 1. Usuário envia mensagem |
| 2. Servidor chama LLM e aguarda... | 2. Servidor chama LLM (`Multi<String>`) |
| 3. LLM gera 500 palavras (demora!) | 3. LLM envia token 1 → cliente vê! |
| 4. Servidor recebe tudo e envia | 4. LLM envia token 2 → cliente vê! |
| 5. Usuário vê a resposta de uma vez | 5. LLM continua... resposta em tempo real |

---

<!-- .slide: class="section-slide" -->

## STEP 4 — System Messages

> Definindo o contexto, tom e escopo da conversa com o LLM

### Tipos de Mensagens no LLM

Em aplicações LLM existem diferentes tipos de mensagens, cada uma com um papel distinto:

| Tipo | Descrição | Exemplo |
|------|-----------|---------|
| **USER** | Mensagem enviada pelo usuário final | `"Quero alugar um carro para o fim de semana"` |
| **ASSISTANT** | Resposta gerada pelo LLM. O histórico dessas mensagens forma a memória da conversa | `"Claro! Para qual cidade e por quantos dias?"` |
| **SYSTEM** | Instrução oculta que define contexto, papel e limites do modelo. O usuário não vê! | `"Você é um agente de suporte de aluguel de carros..."` |

---

<!-- .slide: class="section-slide" -->

### O que é uma System Message?

System Message é uma diretiva que guia o comportamento e o tom do modelo durante toda a interação. Ela define o contexto, o papel e os limites do LLM — e é invisível para o usuário final.

- **Define o papel do modelo** — Ex: "Você é um agente de suporte de aluguel de carros da Miles of Smiles"
- **Controla o tom** — Ex: "Seja amigável, educado e conciso"
- **Estabelece limites** — Ex: "Se a pergunta não for sobre aluguel, redirecione educadamente"
- **Nunca é removida** — Mesmo quando mensagens antigas são descartadas por limite de contexto

---

<!-- .slide: class="section-slide" -->

### Implementando: Anotação `@SystemMessage`

Adicione a anotação `@SystemMessage` ao método `chat()` da interface `CustomerSupportAgent`:

```java
package dev.langchain4j.quarkus.workshop;

import dev.langchain4j.service.SystemMessage;
import io.quarkiverse.langchain4j.RegisterAiService;
import io.smallrye.mutiny.Multi;
import jakarta.enterprise.context.SessionScoped;

@SessionScoped
@RegisterAiService
public interface CustomerSupportAgent {

    @SystemMessage("""
        You are a customer support agent
        of a car rental company 'Miles of Smiles'.
        You are friendly, polite and concise.
        If the question is unrelated to car rental,
        redirect the customer to the right department.
    """)
    Multi<String> chat(String userMessage);
}
```

> **Nota:** O `@SystemMessage` fica no método, não na classe — assim diferentes métodos podem ter contextos distintos!

---

<!-- .slide: class="section-slide" -->

### System Message e Memória da Conversa

O LLM recebe todo o histórico da conversa a cada mensagem — mas quando fica muito longo, mensagens antigas são removidas.

| Mensagem | Status |
|----------|--------|
| `[SYSTEM]` Você é um agente da Miles of Smiles... | ✅ mantida |
| `[USER]` Olá, quero alugar um carro | ✅ mantida |
| `[ASSISTANT]` Olá! Para qual cidade? | ✅ mantida |
| `[USER]` São Paulo, 3 dias | ✅ mantida |
| `[ASSISTANT]` Temos ótimas opções! ... | ❌ removida |
| `[USER]` [mensagem mais antiga] | ❌ removida |

> **A System Message NUNCA é removida da memória. O modelo sempre mantém seu contexto, mesmo em conversas muito longas!**

---

<!-- .slide: class="section-slide" -->

### Testando a System Message

Após adicionar o `@SystemMessage`, reinicie a aplicação e envie:
```
Tell me a story
```

| SEM System Message | COM System Message |
|--------------------|-------------------|
| O LLM conta qualquer história sem restrições. Foge completamente do propósito da aplicação. | O LLM recusa ou redireciona gentilmente, mantendo o foco no contexto de aluguel. |
| *"Era uma vez, numa terra distante..."* | *"Desculpe, posso ajudar apenas com aluguel de carros. Posso ajudá-lo a encontrar um veículo?"* |

---

<!-- .slide: class="section-slide" -->

## Resumo — O que aprendemos hoje?

### Step 2 — Parâmetros do Modelo
- `temperature`: criatividade vs previsibilidade
- `max-completion-tokens`: limite de tamanho
- `frequency-penalty`: controle de repetições

### Step 3 — Streaming com `Multi<String>`
- `Multi<String>` = stream de tokens em tempo real
- Mudança mínima de código, grande impacto na UX
- Quarkus suporta nativamente via WebSocket

### Step 4 — System Messages
- Define contexto, papel e limites do LLM
- `@SystemMessage` annotation na interface Java
- Nunca removida da memória: contexto garantido

---

<!-- .slide: class="section-slide" -->

> **Próximo passo:** Step 5 — Padrão RAG (Retrieval-Augmented Generation)
