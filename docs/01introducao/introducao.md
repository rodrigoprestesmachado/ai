---
layout: default
title: Introdução
nav_order: 2
---

# Introdução

<center>
<iframe src="https://ai.rpmhub.dev/01introducao/slides/index.html#/" title="Introdução" width="90%" height="500" style="border:none;"></iframe>
</center>

## Tarefa para casa

**Objetivo:** executar localmente o **Step 01** do workshop oficial, ou seja, subir o projeto Quarkus + LangChain4j e conversar com o chatbot, como no tutorial. O foco do exercício é passar pelo fluxo completo de ambiente e credencial (o mesmo tipo de passo que você repetirá em projetos reais).
{: .fs-3 }

**Referência:** [Quarkus LangChain4j Workshop — Section 1, Step 01](https://quarkus.io/quarkus-workshop-langchain4j/section-1/step-01/).
{: .fs-3 }

### O que fazer

1. Clonar ou baixar o código indicado no Step 01 e abrir o projeto no seu ambiente.
2. Obter uma **API key da OpenAI**: criar conta em [OpenAI](https://platform.openai.com/) se necessário, gerar uma chave em [API keys](https://platform.openai.com/api-keys) e guardá-la **somente** como variável de ambiente (por exemplo `export OPENAI_API_KEY=sk-...` no macOS/Linux). **Não** commite a chave nem coloque em arquivo versionado.
3. Configurar o projeto conforme o tutorial (dependências e `application.properties` com `${OPENAI_API_KEY}`, como nos slides da introdução).
4. Subir a aplicação com `./mvnw quarkus:dev` (ou equivalente no seu SO) e testar o endpoint/interface do chatbot enviando pelo menos uma mensagem.
{: .fs-3 }

# Referência

[Quarkus LangChain4j Workshop](https://quarkus.io/quarkus-workshop-langchain4j/)

<center>
<a href="https://rpmhub.dev" target="blanck"><img src="../imgs/logo.png" alt="Rodrigo Prestes Machado" width="3%" height="3%" border=0 style="border:0; text-decoration:none; outline:none"></a><br/>
<a rel="license" href="http://creativecommons.org/licenses/by/4.0/">CC BY 4.0 DEED</a>
</center>
