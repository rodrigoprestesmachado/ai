---
layout: default
title: Home
nav_order: 1
---

# Tópicos em Inteligência Artificial

Material didático em português para o [Quarkus LangChain4j Workshop](https://quarkus.io/quarkus-workshop-langchain4j/) — slides de apresentação e texto aprofundado com código, diagramas e exercícios práticos.
{: .fs-3 }

O objetivo deste site é ir além do tutorial oficial: explicar **por que** cada peça existe, **como** ela se encaixa na arquitetura e **o que** você deve experimentar localmente para fixar o aprendizado.
{: .fs-3 }

## Dois trilhos de aprendizado

O conteúdo segue a estrutura do workshop em duas trilhas complementares:

### [Introdução](01introducao/introducao.html)

Ponto de partida obrigatório. Você sobe o primeiro chatbot Quarkus + LangChain4j, configura a API key da OpenAI e entende a anatomia mínima de um AI Service.

### [AI Services](aiservices/)

**Section 1 — AI Apps.** Interfaces reativas (`@RegisterAiService`) que respondem a prompts do usuário. Cobre configuração do modelo, streaming, RAG, function calling, MCP e guardrails.

| Capítulo | Tema |
|---|---|
| [Configuration and Streaming](02streaming/streaming.html) | Parâmetros, streaming e system messages |
| [Retrieval Augmented Generation](03rag/rag.html) | Ingestão e recuperação de contexto (RAG) |
| [Function Calling and Tools](04tools/tools.html) | Tools locais com `@Tool` e `@ToolBox` |
| [Model Context Protocol (MCP)](05mcp/mcp.html) | Ferramentas remotas via MCP |
| [Guardrails](06guardrails/guardrails.html) | Proteção contra prompt injection |

### [AI Agents](aiagents/)

**Section 2 — Agentic Workflows.** Agentes autônomos (`@Agent`) que tomam decisões, invocam tools e colaboram em workflows multi-agente.

| Capítulo | Tema |
|---|---|
| [Implementing AI Agents](07agents/agents.html) | Primeiro agente autônomo com `@Agent` |

## Cenário unificado

Todos os exemplos usam a locadora fictícia **Miles of Smiles**: um chatbot de atendimento ao cliente que, ao longo do workshop, ganha RAG, tools, MCP, guardrails e, na Section 2, agentes autônomos para gerenciar a frota de veículos.
{: .fs-3 }

## Pré-requisitos

Para executar os exercícios localmente, você precisa de:
{: .fs-3 }

- **Java 17+** e **Maven** (ou use o `./mvnw` incluído no projeto do workshop)
- Conta na [OpenAI](https://platform.openai.com/) com uma API key
- Variável de ambiente `OPENAI_API_KEY` configurada no terminal
- Git e uma IDE de sua preferência (IntelliJ, VS Code, etc.)
{: .fs-3 }

## Como usar este site

Cada capítulo oferece duas formas de estudo:
{: .fs-3 }

1. **Slides** — apresentação em formato Reveal.js embutida no topo da página (ideal para aula ou revisão rápida)
2. **Texto** — explicação detalhada com trechos de código, diagramas e **tarefa para casa** para praticar no projeto do workshop
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
