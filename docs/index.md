---
layout: default
title: Home
nav_order: 1
---

# Tópicos em Inteligência Artificial

Material didático em português para o [Quarkus LangChain4j Workshop](https://quarkus.io/quarkus-workshop-langchain4j/), com slides de apresentação e texto aprofundado contendo código, diagramas e exercícios práticos.
{: .fs-3 }

## Sobre o workshop

O workshop usa **Quarkus** e **LangChain4j** para ensinar, passo a passo, como construir aplicações Java que integram modelos de linguagem (LLMs). Todos os exemplos giram em torno da locadora fictícia **Miles of Smiles**: um chatbot de atendimento ao cliente que, ao longo das seções, ganha RAG, tools, MCP, guardrails e, na Section 2, agentes autônomos para gerenciar a frota de veículos.
{: .fs-3 }

## Jornada de aprendizado

O conteúdo está organizado em duas partes sequenciais:
{: .fs-3 }

### 1. [Introdução](01introducao/introducao.html)

Configure o ambiente, obtenha a API key da OpenAI e execute o primeiro chatbot Quarkus localmente.

### 2. [AI Services](aiservices/)

**Section 1 (AI Apps).** Construa um AI Service completo, partindo de interfaces `@RegisterAiService` até proteção por guardrails.

| Capítulo | O que você aprende |
|---|---|
| [Configuration and Streaming](02streaming/streaming.html) | Parâmetros do modelo, streaming e system messages |
| [Retrieval Augmented Generation](03rag/rag.html) | Ingestão de documentos e recuperação de contexto |
| [Function Calling and Tools](04tools/tools.html) | Invocação de funções locais com `@Tool` e `@ToolBox` |
| [Model Context Protocol (MCP)](05mcp/mcp.html) | Ferramentas remotas via servidores MCP |
| [Guardrails](06guardrails/guardrails.html) | Proteção contra prompt injection |

### 3. [AI Agents](aiagents/)

**Section 2 (Agentic Workflows).** Construa agentes autônomos com `@Agent` que tomam decisões, invocam tools e colaboram em workflows multi-agente.

| Capítulo | O que você aprende |
|---|---|
| [Implementing AI Agents](07agents/agents.html) | Primeiro agente autônomo com `@Agent` e decisão baseada em contexto |

## Pré-requisitos

Para executar os exercícios localmente:
{: .fs-3 }

| Requisito | Detalhe |
|---|---|
| Java 17+ | Verifique com `java -version` |
| Maven | Ou use o `./mvnw` incluído no projeto do workshop |
| Conta OpenAI | Gere uma API key em [platform.openai.com](https://platform.openai.com/api-keys) |
| `OPENAI_API_KEY` | Variável de ambiente configurada no terminal |
| Git e IDE | IntelliJ, VS Code ou outra de sua preferência |

## Como usar este site

Cada capítulo oferece duas formas de estudo:
{: .fs-3 }

1. **Slides:** apresentação em formato Reveal.js embutida no topo da página, ideal para aula ou revisão rápida.
2. **Texto:** explicação detalhada com código, diagramas e tarefa para casa para praticar no projeto do workshop.
{: .fs-3 }

Comece pela [Introdução](01introducao/introducao.html), execute o Step 01 localmente e siga a ordem sugerida dentro de cada trilho.
{: .fs-3 }

<center>
  <a href="https://rpmhub.dev" target="blanck">
    <img src="imgs/logo.png" alt="Rodrigo Prestes Machado" width="3%"
    height="3%" border=0 style="border:0; text-decoration:none; outline:none">
  </a>
  <br/>
  <a rel="license" href="http://creativecommons.org/licenses/by/4.0/">
        CC BY 4.0 DEED
  </a>
</center>
