---
layout: default
title: AI Services
nav_order: 3
has_children: true
---

# AI Services

**AI Services** são interfaces Java anotadas com `@RegisterAiService` cujo comportamento é gerado automaticamente pelo Quarkus LangChain4j. Elas representam o padrão reativo da **Section 1** do workshop: o LLM responde a prompts do usuário, pode receber contexto via RAG, invocar ferramentas locais ou remotas (MCP) e ser protegido por guardrails.
{: .fs-3 }

Este trilho cobre os *Steps* 02 a 09 do [Quarkus LangChain4j Workshop — Section 1](https://quarkus.io/quarkus-workshop-langchain4j/section-1/step-02/), sempre usando o cenário da locadora **Miles of Smiles**.
{: .fs-3 }

## Capítulos

| Ordem | Capítulo | O que você aprende |
|---|---|---|
| 1 | [Configuration and Streaming](02streaming/streaming.html) | Parâmetros do modelo, streaming de respostas e system messages |
| 2 | [Retrieval Augmented Generation](03rag/rag.html) | Ingestão de documentos e recuperação de contexto (RAG) |
| 3 | [Function Calling and Tools](04tools/tools.html) | Function calling e tools locais com `@Tool` e `@ToolBox` |
| 4 | [Model Context Protocol (MCP)](05mcp/mcp.html) | Integração com servidores MCP remotos via `@McpToolBox` |
| 5 | [Guardrails](06guardrails/guardrails.html) | Proteção contra prompt injection com input guardrails |

## Ordem sugerida

Leia os capítulos na ordem acima. Cada um pressupõe o conhecimento dos anteriores — especialmente Function Calling (Step 07) e MCP (Step 08) antes de Guardrails (Step 09).
{: .fs-3 }

<center>
<a href="https://rpmhub.dev" target="blanck"><img src="../imgs/logo.png" alt="Rodrigo Prestes Machado" width="3%" height="3%" border=0 style="border:0; text-decoration:none; outline:none"></a><br/>
<a rel="license" href="http://creativecommons.org/licenses/by/4.0/">CC BY 4.0 DEED</a>
</center>
