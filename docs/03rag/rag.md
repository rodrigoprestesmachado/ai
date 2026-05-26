---
layout: default
title: Retrieval Augmented Generation
nav_order: 4
---

# RAG - Retrieval Augmented Generation

RAG é uma técnica que combina a geração de texto com a recuperação de informações relevantes. Ele é usado para melhorar a qualidade e a precisão das respostas geradas por modelos de linguagem, permitindo que eles acessem informações externas para fornecer respostas mais informadas e contextualmente relevantes.

Na prática, uma aplicação RAG é dividida em duas fases. A primeira é a **ingestão** (offline), na qual os documentos da base de conhecimento são preparados e indexados em um banco vetorial. A segunda é a **recuperação e geração** (online), executada a cada pergunta do usuário: o sistema busca os trechos mais relevantes no banco vetorial e os envia, junto com a pergunta, ao LLM como contexto adicional para compor a resposta.

A figura abaixo, extraída do [Quarkus LangChain4j Workshop](https://quarkus.io/quarkus-workshop-langchain4j/), ilustra justamente o **pipeline de ingestão**:

<center>
<img src="https://quarkus.io/quarkus-workshop-langchain4j/images/ingestion.png" alt="Pipeline de ingestão de documentos para RAG" width="80%">
</center>

O fluxo apresentado pode ser entendido em quatro etapas:

1. **Documentos de origem**: arquivos diversos (PDF, Markdown, HTML, texto puro, etc.) que formam a base de conhecimento a ser consultada pelo modelo.
2. **Text Splitter**: como LLMs e modelos de embedding têm um limite de contexto, os documentos são divididos em pedaços menores chamados *text segments* (ou *chunks*). Um bom particionamento preserva a semântica de cada trecho e melhora a qualidade da recuperação.
3. **Embedding Computation**: cada segmento é transformado em um vetor numérico (*embedding*) por um **modelo de embeddings**. Esses vetores capturam o significado do texto, de modo que trechos semanticamente próximos fiquem próximos também no espaço vetorial.
4. **Storage no Vector Store**: o segmento original e seu embedding correspondente são armazenados juntos em um **banco vetorial** (por exemplo, Chroma, pgvector, Redis, Infinispan). Posteriormente, durante a consulta, esse banco será pesquisado por similaridade para recuperar os trechos mais relevantes à pergunta do usuário.

É importante notar que esta fase é executada **uma única vez** (ou sempre que a base de conhecimento for atualizada) e é o que torna possível, em tempo de execução, responder perguntas com base em conteúdo específico do domínio sem precisar re-treinar o modelo.

<center>
<iframe src="https://ai.rpmhub.dev/03rag/slides/index.html#/" title="RAG - Retrieval Augmented Generation" width="90%" height="500" style="border:none;"></iframe>
</center>

# Referência

[Quarkus LangChain4j Workshop](https://quarkus.io/quarkus-workshop-langchain4j/)

<center>
<a href="https://rpmhub.dev" target="blanck"><img src="../imgs/logo.png" alt="Rodrigo Prestes Machado" width="3%" height="3%" border=0 style="border:0; text-decoration:none; outline:none"></a><br/>
<a rel="license" href="http://creativecommons.org/licenses/by/4.0/">CC BY 4.0 DEED</a>
</center>
