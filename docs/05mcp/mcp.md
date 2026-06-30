---
layout: default
title: Model Context Protocol (MCP)
parent: AI Services
nav_order: 4
---

# Model Context Protocol (MCP)

<center>
<iframe src="https://ai.rpmhub.dev/05mcp/slides/index.html#/" title="Model Context Protocol" width="90%" height="500" style="border:none;"></iframe>
</center>

O **Model Context Protocol (MCP)** é um padrão aberto que padroniza a forma como aplicações de IA se comunicam com fontes de dados e ferramentas externas. Enquanto o Function Calling permite ao LLM invocar funções **locais** da aplicação, o MCP vai além: ele define um protocolo de comunicação que permite ao LLM interagir com **servidores remotos** de forma segura, bidirecional e agnóstica de linguagem.
{: .fs-3 }

Criado pela Anthropic e adotado amplamente pela comunidade, o MCP funciona como um "conector universal" entre agentes de IA e o ecossistema de ferramentas e dados ao redor. Um servidor MCP pode ser escrito em qualquer linguagem — Python, TypeScript, Java — e ser consumido por qualquer cliente MCP compatível.
{: .fs-3 }

## Arquitetura MCP

A arquitetura do MCP é composta por dois papéis principais: o **MCP Server** e o **MCP Client**.
{: .fs-3 }

<center>
<img src="mcp-agent-tools.png" alt="Arquitetura MCP: AI Agent orquestrando múltiplas ferramentas" width="80%"/>
</center>

A imagem acima ilustra um exemplo concreto de agente MCP: a partir de um prompt do usuário, o **AI Agent** orquestra três ferramentas em sequência:
{: .fs-3 }

1. **City Extractor Tool** — extrai o nome da cidade mencionada no prompt (executa localmente, usando o próprio LLM como ferramenta de extração).
2. **Geocoding Tool** — converte o nome da cidade em coordenadas geográficas (latitude/longitude), consultando uma API externa.
3. **Weather Forecast Service** — usa as coordenadas para buscar a previsão do tempo em uma API meteorológica remota.
{: .fs-3 }

O resultado de cada tool alimenta a próxima etapa, e o agente combina todas as informações para entregar a resposta final ao usuário. Esse encadeamento de ferramentas é exatamente o que o MCP padroniza: uma forma comum de expor e invocar ferramentas — locais ou remotas — de qualquer aplicação de IA.
{: .fs-3 }

```
┌─────────────────────────────────────────────────────────┐
│                    Aplicação (MCP Client)                │
│                                                         │
│   AI Service ──── @McpToolBox ──── MCP Client           │
│                                        │                │
└────────────────────────────────────────┼────────────────┘
                                         │ HTTP/SSE
                                         ▼
                             ┌───────────────────────┐
                             │   MCP Server (remoto)  │
                             │                       │
                             │  @Tool getForecast()  │
                             │  @Tool getStocks()    │
                             │  ...                  │
                             └───────────────────────┘
```
{: .fs-3 }

O fluxo de comunicação segue os mesmos princípios do Function Calling local, mas com a execução das ferramentas ocorrendo em um processo separado, potencialmente em outra máquina:
{: .fs-3 }

1. O LLM recebe a lista de tools disponíveis no servidor MCP (via `@McpToolBox`).
2. O LLM decide invocar uma tool e retorna uma *tool call request*.
3. O **MCP Client** (na aplicação) transmite a requisição ao **MCP Server** via HTTP/SSE.
4. O MCP Server executa a tool e retorna o resultado.
5. O MCP Client repassa o resultado ao LLM.
6. O LLM formula a resposta final ao usuário.
{: .fs-3 }

## Criando um MCP Server com Quarkus

### Dependências

Para criar um servidor MCP com Quarkus, adicione a extensão `quarkus-mcp-server-sse` e o REST Client para consumir APIs externas:
{: .fs-3 }

```bash
quarkus create app dev.langchain4j.quarkus.workshop:mcp-server:1.0-SNAPSHOT \
  -x quarkus-mcp-server-sse \
  -x quarkus-rest-client-jackson
```
{: .fs-3 }

Ou adicione as dependências manualmente ao `pom.xml`:
{: .fs-3 }

```xml
<dependency>
    <groupId>io.quarkiverse.mcp</groupId>
    <artifactId>quarkus-mcp-server-sse</artifactId>
</dependency>
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-rest-client-jackson</artifactId>
</dependency>
```
{: .fs-3 }

### Definindo um REST Client

O servidor MCP pode chamar APIs externas para enriquecer as respostas. No exemplo do workshop, é criado um cliente REST que consulta previsões meteorológicas:
{: .fs-3 }

```java
@Path("/v1/forecast")
@RegisterRestClient(configKey = "weatherclient")
public interface WeatherClient {

    @GET
    String getForecast(
            @RestQuery double latitude,
            @RestQuery double longitude,
            @RestQuery int forecastDays,
            @RestQuery String hourly
    );
}
```
{: .fs-3 }

### Definindo Tools no MCP Server

Assim como no Function Calling local, as tools do MCP Server são métodos Java anotados com `@Tool`. A diferença é que agora essas ferramentas ficam em um processo separado e são expostas remotamente via protocolo MCP:
{: .fs-3 }

```java
public class Weather {

    @RestClient
    WeatherClient weatherClient;

    @Tool(description = "Get weather forecast for a location.")
    String getForecast(
            @ToolArg(description = "Latitude of the location") double latitude,
            @ToolArg(description = "Longitude of the location") double longitude) {

        return weatherClient.getForecast(
                latitude,
                longitude,
                16,
                "temperature_2m,snowfall,rain,precipitation,precipitation_probability");
    }
}
```
{: .fs-3 }

Note o uso de `@ToolArg` (em vez de apenas nomear o parâmetro) para fornecer ao LLM uma descrição semântica de cada argumento.
{: .fs-3 }

### Configurando o MCP Server

No `application.properties` do servidor MCP:
{: .fs-3 }

```properties
# Porta diferente da aplicação cliente
quarkus.http.port=8081

# Identificação e logs do servidor MCP
quarkus.mcp.server.server-info.name=Weather Service
quarkus.mcp.server.traffic-logging.enabled=true
quarkus.mcp.server.traffic-logging.text-limit=100

# REST Client para a API meteorológica
quarkus.rest-client.logging.scope=request-response
quarkus.rest-client.follow-redirects=true
quarkus.rest-client.logging.body-limit=50
quarkus.rest-client."weatherclient".uri=https://api.open-meteo.com/
```
{: .fs-3 }

## Configurando o MCP Client (aplicação principal)

### Dependência

Na aplicação que consome o MCP Server, adicione a dependência:
{: .fs-3 }

```xml
<dependency>
    <groupId>io.quarkiverse.langchain4j</groupId>
    <artifactId>quarkus-langchain4j-mcp</artifactId>
</dependency>
```
{: .fs-3 }

Ou via CLI:
{: .fs-3 }

```bash
./mvnw quarkus:add-extension -Dextensions="quarkus-langchain4j-mcp"
```
{: .fs-3 }

### Configuração do transporte

No `application.properties` da aplicação cliente, aponte para o servidor MCP:
{: .fs-3 }

```properties
quarkus.langchain4j.mcp.weather.transport-type=http
quarkus.langchain4j.mcp.weather.url=http://localhost:8081/mcp/sse/
```
{: .fs-3 }

O nome `weather` é o identificador do servidor MCP. Ele será referenciado na anotação `@McpToolBox`.
{: .fs-3 }

### Configurando o AI Service com `@McpToolBox`

Para expor as tools do servidor MCP ao LLM, use a anotação `@McpToolBox` no AI Service:
{: .fs-3 }

```java
@SessionScoped
@RegisterAiService
public interface CustomerSupportAgent {

    @SystemMessage("""
            You are a customer support agent of a car rental company 'Miles of Smiles'.
            You are friendly, polite and concise.
            If the question is unrelated to car rental, you should politely redirect
            the customer to the right department.

            When calling tools or functions, strictly use JSON objects,
            do not wrap in quotes or use plain strings.

            When asked to provide details about a reservation,
            provide weather details and gently try to upsell the customer
            based on this info.

            Today is {current_date}.
            """)
    @ToolBox(BookingRepository.class)
    @McpToolBox("weather")
    String chat(String userMessage);
}
```
{: .fs-3 }

Dois pontos importantes:
{: .fs-3 }

* **`@McpToolBox("weather")`** — referencia o servidor MCP pelo nome configurado em `application.properties`. O Quarkus LangChain4j descobre automaticamente as tools disponíveis no servidor e as inclui nas requisições ao LLM.
* **`@ToolBox` e `@McpToolBox` podem coexistir** — é possível combinar tools locais (Function Calling) com tools remotas (MCP) no mesmo AI Service.
{: .fs-3 }

## Function Calling Local vs. MCP

Ambas as abordagens permitem ao LLM executar ações além de responder perguntas. A escolha depende do contexto:
{: .fs-3 }

| Aspecto | Function Calling (`@ToolBox`) | MCP (`@McpToolBox`) |
|---|---|---|
| **Localização** | Execução local, dentro da aplicação | Execução remota, em servidor separado |
| **Reusabilidade** | Limitada à aplicação | Alta — qualquer cliente MCP pode usar |
| **Linguagem** | Java | Qualquer linguagem |
| **Transporte** | In-process | HTTP com SSE |
| **Complexidade** | Baixa | Moderada |
| **Ideal para** | Lógica de negócio própria | APIs externas, ferramentas compartilhadas |

As duas técnicas são **complementares** e podem ser combinadas no mesmo AI Service.
{: .fs-3 }

# Referência

[Quarkus LangChain4j Workshop — Step 08](https://quarkus.io/quarkus-workshop-langchain4j/section-1/step-08/)
{: .fs-3 }

<center>
<a href="https://rpmhub.dev" target="blanck"><img src="../imgs/logo.png" alt="Rodrigo Prestes Machado" width="3%" height="3%" border=0 style="border:0; text-decoration:none; outline:none"></a><br/>
<a rel="license" href="http://creativecommons.org/licenses/by/4.0/">CC BY 4.0 DEED</a>
</center>
