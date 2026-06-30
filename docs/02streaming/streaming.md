---
layout: default
title: Configuration and Streaming
parent: AI Services
nav_order: 1
---

# LLM configuration and Streaming

<center>
<iframe src="https://ai.rpmhub.dev/02streaming/slides/index.html#/" title="LLM Configuration and Streaming " width="90%" height="500" style="border:none;"></iframe>
</center>

Construir uma aplicação que conversa com um LLM não se resume a "chamar a API e exibir o texto". Para obter uma boa experiência é preciso **configurar o comportamento do modelo**, **entregar a resposta ao usuário em tempo real** e **delimitar o papel** que esse modelo deve assumir na conversa. Este material, baseado nos *Steps* 2, 3 e 4 do [Quarkus LangChain4j Workshop](https://quarkus.io/quarkus-workshop-langchain4j/), apresenta esses três pilares: parâmetros do modelo, *streaming* de respostas e *system messages*.
{: .fs-3 }

## Parâmetros do modelo

A configuração do LLM em uma aplicação Quarkus é feita, em geral, pelo arquivo `application.properties`. Esse arquivo concentra a chave de API, o nome do modelo, as opções de *log* e os hiperparâmetros que controlam o comportamento do modelo durante a geração.
{: .fs-3 }

```properties
quarkus.langchain4j.openai.api-key=${OPENAI_API_KEY}
quarkus.langchain4j.openai.chat-model.model-name=gpt-4o
quarkus.langchain4j.openai.chat-model.log-requests=true
quarkus.langchain4j.openai.chat-model.log-responses=true
quarkus.langchain4j.timeout=1m
```

As chaves mais importantes têm os seguintes papéis:
{: .fs-3 }

- **`api-key`**: chave de acesso à API, normalmente injetada via variável de ambiente para não ser versionada no código.
- **`model-name`**: define qual modelo será utilizado (por exemplo, `gpt-4o`).
- **`log-requests` / `log-responses`**: habilitam os *logs* das requisições e respostas no terminal, úteis para *debug*.
{: .fs-3 }

Além dessas configurações estruturais, três hiperparâmetros impactam diretamente o **estilo** e o **tamanho** das respostas: `temperature`, `max-completion-tokens` e `frequency-penalty`.
{: .fs-3 }

### Temperature: criatividade do modelo

A propriedade `quarkus.langchain4j.openai.chat-model.temperature` controla o quanto o modelo é "criativo" ou "conservador" ao escolher cada token da resposta. Valores baixos tornam o modelo previsível; valores altos introduzem mais aleatoriedade.
{: .fs-3 }

| Valor | Comportamento                                |
|-------|----------------------------------------------|
| `0.0` | Determinístico (respostas previsíveis)       |
| `1.0` | Balanceado (recomendado para *chatbots*)     |
| `2.0` | Caótico (pode gerar texto sem sentido)       |

Um experimento simples é pedir ao modelo para descrever um pôr do sol com `temperature=0.1` e em seguida com `temperature=1.5`: a diferença de estilo e de variabilidade entre as respostas evidencia o efeito desse parâmetro.
{: .fs-3 }

### Max Tokens: limite da resposta

A propriedade `quarkus.langchain4j.openai.chat-model.max-completion-tokens` define o **número máximo de tokens** que o modelo pode gerar como resposta. *Tokens* não são exatamente palavras: são as menores unidades em que o texto é segmentado pelo modelo. Como exemplo, a expressão `"Hello, world!"` corresponde a aproximadamente 4 *tokens*.
{: .fs-3 }

```properties
# Limitar a 20 tokens (resposta será cortada no meio):
quarkus.langchain4j.openai.chat-model.max-completion-tokens=20

# Valor recomendado para o workshop:
quarkus.langchain4j.openai.chat-model.max-completion-tokens=1000
```

Definir `max-completion-tokens=20` e enviar uma pergunta cuja resposta seja naturalmente longa é uma boa forma de observar o efeito do parâmetro: a resposta é interrompida abruptamente. Ajustando para `1000`, o modelo passa a ter espaço suficiente para concluir o raciocínio.
{: .fs-3 }

### Frequency Penalty: evitando repetições

A propriedade `quarkus.langchain4j.openai.chat-model.frequency-penalty` define o quanto o modelo deve **evitar repetir** as mesmas palavras e expressões ao longo da resposta.
{: .fs-3 }

| Valor | Comportamento                                              | Resultado típico                            |
|-------|------------------------------------------------------------|---------------------------------------------|
| `0`   | Sem penalidade; o modelo pode repetir livremente           | `hedgehog hedgehog hedgehog...`             |
| `2`   | Penalidade máxima; evita fortemente repetições             | `hedgehog... porcupine? spiky creature...`  |

Penalidades muito altas podem comprometer a coerência, gerando texto sem sentido. Um experimento didático é pedir "Repeat the word hedgehog 50 times" com `frequency-penalty=0` e depois com `frequency-penalty=2` e comparar os resultados.
{: .fs-3 }

### Configuração final

Os valores recomendados para o *workshop* combinam coerência, espaço para respostas longas e ausência de penalidade de repetição:
{: .fs-3 }

| Parâmetro                | Valor   | Descrição                                  |
|--------------------------|---------|--------------------------------------------|
| `temperature`            | `1.0`   | Balanceado: criativo, porém coerente       |
| `max-completion-tokens`  | `1000`  | Permite respostas mais longas              |
| `frequency-penalty`      | `0`     | Sem penalidade de repetição                |

```properties
quarkus.langchain4j.openai.api-key=${OPENAI_API_KEY}
quarkus.langchain4j.openai.chat-model.model-name=gpt-4o
quarkus.langchain4j.openai.chat-model.log-requests=true
quarkus.langchain4j.openai.chat-model.log-responses=true
quarkus.langchain4j.openai.chat-model.temperature=1.0
quarkus.langchain4j.openai.chat-model.max-completion-tokens=1000
quarkus.langchain4j.openai.chat-model.frequency-penalty=0
```

## Respostas em streaming

Mesmo com o modelo bem configurado, ainda há um problema de **experiência de uso**: por padrão, o servidor aguarda o LLM gerar a resposta **completa** antes de devolvê-la ao cliente. Em respostas longas, o usuário fica olhando para um "…" piscando até que tudo apareça de uma só vez.
{: .fs-3 }

```
Usuário envia mensagem  →  Servidor aguarda o LLM  →  LLM gera TUDO  →  Usuário recebe tudo
```

Esse fluxo tem dois inconvenientes principais:
{: .fs-3 }

- O cliente **não recebe nenhum feedback** enquanto a resposta é gerada.
- O servidor precisa **manter toda a resposta em memória** antes de enviá-la.
{: .fs-3 }

A solução é **streaming**: enviar a resposta ao cliente *token por token*, à medida que o LLM a gera.
{: .fs-3 }

### Mudança 1: retorno `Multi<String>` no AI Service

O tipo `Multi`, da biblioteca **Mutiny**, representa um *fluxo* (stream) de itens. No contexto do *chat*, cada item é um fragmento da resposta:
{: .fs-3 }

- **Stream de strings**: cada item é um pedaço (token) da resposta.
- **Assíncrono**: os itens chegam conforme o LLM gera.
- **Finito**: o stream termina quando a resposta acaba.
- **Back-pressure**: o fluxo é controlado para evitar sobrecarga.
{: .fs-3 }

Basta alterar o tipo de retorno do método `chat()` na interface do *AI Service*:
{: .fs-3 }

**Antes** (resposta completa):

```java
@SessionScoped
@RegisterAiService
public interface CustomerSupportAgent {
    String chat(String msg);
}
```

**Depois** (resposta em streaming):

```java
@SessionScoped
@RegisterAiService
public interface CustomerSupportAgent {
    Multi<String> chat(String msg);
}
```

### Mudança 2: atualizando o WebSocket

O *endpoint* WebSocket também precisa devolver um `Multi<String>`, em vez de aguardar uma `String` inteira:
{: .fs-3 }

**Antes** (retorna `String`, bloqueia até terminar):

```java
@OnTextMessage
public String onTextMessage(String message) {
    return customerSupportAgent.chat(message);
}
```

**Depois** (retorna `Multi<String>`, streaming):

```java
@OnTextMessage
public Multi<String> onTextMessage(String message) {
    return customerSupportAgent.chat(message);
}
```

O suporte a `Multi<String>` é **nativo** no Quarkus WebSockets: cada *token* recebido do LLM é encaminhado imediatamente ao cliente, sem necessidade de configuração adicional.
{: .fs-3 }

### Testando o streaming

Uma forma eficiente de testar é enviar um prompt cuja resposta seja naturalmente longa, por exemplo:
{: .fs-3 }

```
Tell me a story containing 500 words
```

A diferença entre as duas abordagens é nítida:
{: .fs-3 }

| Sem streaming (`String`)                       | Com streaming (`Multi<String>`)                   |
|------------------------------------------------|---------------------------------------------------|
| Usuário envia mensagem                         | Usuário envia mensagem                            |
| Servidor chama o LLM e aguarda…                | Servidor chama o LLM (`Multi<String>`)            |
| LLM gera 500 palavras (demora)                 | LLM envia o token 1 → cliente já vê!              |
| Servidor recebe tudo e envia                   | LLM envia o token 2 → cliente já vê!              |
| Usuário vê a resposta de uma vez               | LLM continua… resposta aparece em tempo real      |

## System Messages

O último passo para tornar a aplicação útil em um cenário real é **definir o papel** que o modelo deve assumir. Para isso, é importante entender que, em uma conversa com um LLM, existem três tipos de mensagens:
{: .fs-3 }

| Tipo          | Descrição                                                                                  | Exemplo                                                     |
|---------------|--------------------------------------------------------------------------------------------|-------------------------------------------------------------|
| **USER**      | Mensagem enviada pelo usuário final                                                        | `"Quero alugar um carro para o fim de semana"`              |
| **ASSISTANT** | Resposta gerada pelo LLM; o histórico dessas mensagens forma a memória da conversa         | `"Claro! Para qual cidade e por quantos dias?"`             |
| **SYSTEM**    | Instrução oculta que define contexto, papel e limites do modelo (o usuário não a vê)       | `"Você é um agente de suporte de aluguel de carros..."`     |

### O que é uma System Message?

Uma **System Message** é uma diretiva que **guia o comportamento e o tom** do modelo durante toda a interação. Ela é invisível para o usuário final e cumpre quatro funções principais:
{: .fs-3 }

- **Define o papel do modelo**: por exemplo, "Você é um agente de suporte de aluguel de carros da Miles of Smiles".
- **Controla o tom**: por exemplo, "Seja amigável, educado e conciso".
- **Estabelece limites**: por exemplo, "Se a pergunta não for sobre aluguel, redirecione educadamente".
- **Nunca é removida**: mesmo quando mensagens antigas são descartadas por limite de contexto.
{: .fs-3 }

### Implementação: anotação `@SystemMessage`

No Quarkus LangChain4j, a *system message* é declarada por meio da anotação `@SystemMessage`, aplicada ao **método** do *AI Service* (e não à classe), o que permite que diferentes métodos tenham contextos distintos:
{: .fs-3 }

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

### System Message e memória da conversa

A cada nova mensagem, o LLM recebe **todo o histórico da conversa**. Quando esse histórico se torna grande demais para o limite de contexto do modelo, mensagens antigas começam a ser descartadas. A *system message*, porém, **nunca é removida**:
{: .fs-3 }

| Mensagem                                                   | Status     |
|------------------------------------------------------------|------------|
| `[SYSTEM]` Você é um agente da Miles of Smiles…            | mantida    |
| `[USER]` Olá, quero alugar um carro                        | mantida    |
| `[ASSISTANT]` Olá! Para qual cidade?                       | mantida    |
| `[USER]` São Paulo, 3 dias                                 | mantida    |
| `[ASSISTANT]` Temos ótimas opções! …                       | removida   |
| `[USER]` *(mensagem mais antiga)*                          | removida   |

Esse comportamento garante que, mesmo em conversas muito longas, o modelo continua sabendo **quem é** e **qual é seu escopo**.
{: .fs-3 }

### Testando a System Message

Para perceber o efeito, basta reiniciar a aplicação após adicionar a anotação `@SystemMessage` e enviar uma mensagem fora do escopo do agente, por exemplo:
{: .fs-3 }

```
Tell me a story
```

| Sem System Message                                                              | Com System Message                                                                       |
|---------------------------------------------------------------------------------|------------------------------------------------------------------------------------------|
| O LLM conta qualquer história, fugindo completamente do propósito da aplicação. | O LLM recusa ou redireciona gentilmente, mantendo o foco no contexto de aluguel.         |
| *"Era uma vez, numa terra distante..."*                                         | *"Desculpe, posso ajudar apenas com aluguel de carros. Posso ajudá-lo a encontrar um veículo?"* |

## Síntese

- **Step 2 (Parâmetros do modelo)**: `temperature` regula criatividade vs. previsibilidade, `max-completion-tokens` limita o tamanho da resposta e `frequency-penalty` controla a tendência do modelo a repetir palavras.
- **Step 3 (Streaming com `Multi<String>`)**: trocar o tipo de retorno do *AI Service* e do *endpoint* WebSocket de `String` para `Multi<String>` é uma mudança mínima de código com grande impacto na experiência de uso, entregando a resposta token a token.
- **Step 4 (System Messages)**: a anotação `@SystemMessage` define papel, tom e limites do modelo, é invisível ao usuário e nunca é descartada da memória, garantindo coerência ao longo de toda a conversa.
{: .fs-3 }

Esses três passos formam a base para construir *chatbots* corporativos com Quarkus + LangChain4j que sejam, ao mesmo tempo, **bem comportados**, **responsivos** e **alinhados ao domínio** da aplicação.
{: .fs-3 }

# Referência

[Quarkus LangChain4j Workshop](https://quarkus.io/quarkus-workshop-langchain4j/)
{: .fs-3 }

<center>
<a href="https://rpmhub.dev" target="blanck"><img src="../imgs/logo.png" alt="Rodrigo Prestes Machado" width="3%" height="3%" border=0 style="border:0; text-decoration:none; outline:none"></a><br/>
<a rel="license" href="http://creativecommons.org/licenses/by/4.0/">CC BY 4.0 DEED</a>
</center>
