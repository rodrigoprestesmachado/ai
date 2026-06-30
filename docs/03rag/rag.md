---
layout: default
title: Retrieval Augmented Generation
parent: AI Services
nav_order: 2
---

# RAG: Retrieval Augmented Generation

<center>
<iframe src="https://ai.rpmhub.dev/03rag/slides/index.html#/" title="RAG - Retrieval Augmented Generation" width="90%" height="500" style="border:none;"></iframe>
</center>

RAG é uma técnica que combina a geração de texto com a recuperação de informações relevantes. Ele é usado para melhorar a qualidade e a precisão das respostas geradas por modelos de linguagem, permitindo que eles acessem informações externas para fornecer respostas mais informadas e contextualmente relevantes.
{: .fs-3 }

Na prática, uma aplicação RAG é dividida em duas fases. A primeira é a **ingestão** (offline), na qual os documentos da base de conhecimento são preparados e indexados em um banco vetorial. A segunda é a **recuperação e geração** (online), executada a cada pergunta do usuário: o sistema busca os trechos mais relevantes no banco vetorial e os envia, junto com a pergunta, ao LLM como contexto adicional para compor a resposta.
{: .fs-3 }

## Fase 1: ingestão (offline)

A figura abaixo, extraída do [Quarkus LangChain4j Workshop](https://quarkus.io/quarkus-workshop-langchain4j/), ilustra o **pipeline de ingestão**:
{: .fs-3 }

<center>
<img src="https://quarkus.io/quarkus-workshop-langchain4j/images/ingestion.png" alt="Pipeline de ingestão de documentos para RAG" width="80%">
</center>

O fluxo apresentado pode ser entendido em quatro etapas:
{: .fs-3 }

1. **Documentos de origem**: arquivos diversos (PDF, Markdown, HTML, texto puro, etc.) que formam a base de conhecimento a ser consultada pelo modelo.
2. **Text Splitter**: como LLMs e modelos de embedding têm um limite de contexto, os documentos são divididos em pedaços menores chamados *text segments* (ou *chunks*). Um bom particionamento preserva a semântica de cada trecho e melhora a qualidade da recuperação.
3. **Embedding Computation**: cada segmento é transformado em um vetor numérico (*embedding*) por um **modelo de embeddings**. Esses vetores capturam o significado do texto, de modo que trechos semanticamente próximos fiquem próximos também no espaço vetorial.
4. **Storage no Vector Store**: o segmento original e seu embedding correspondente são armazenados juntos em um **banco vetorial** (por exemplo, Chroma, pgvector, Redis, Infinispan). Posteriormente, durante a consulta, esse banco será pesquisado por similaridade para recuperar os trechos mais relevantes à pergunta do usuário.
{: .fs-3 }

Esta fase é executada **uma única vez** (ou sempre que a base de conhecimento for atualizada). É justamente esse índice vetorial pré-construído que viabiliza, em tempo de execução, a segunda fase do RAG.
{: .fs-3 }

## Fase 2: recuperação e geração (online)

A segunda fase, ilustrada na figura abaixo (também extraída do [Quarkus LangChain4j Workshop](https://quarkus.io/quarkus-workshop-langchain4j/)), é executada **a cada pergunta** feita pelo usuário:
{: .fs-3 }

<center>
<img src="https://quarkus.io/quarkus-workshop-langchain4j/images/augmentation.png" alt="Pipeline de recuperação e aumento do prompt para RAG" width="80%">
</center>

O fluxo pode ser entendido em cinco etapas:
{: .fs-3 }

1. **Query**: a pergunta do usuário chega à aplicação em formato de texto.
2. **Embedding Computation**: a *query* é convertida em vetor pelo **mesmo modelo de embeddings** utilizado na ingestão. Esse detalhe é fundamental, pois vetores gerados por modelos diferentes não são comparáveis e a recuperação retornaria resultados irrelevantes.
3. **Find relevant segments**: o vetor da pergunta é comparado, por similaridade (tipicamente *cosseno*), com os vetores armazenados no *vector store*. São selecionados os *k* segmentos mais próximos, isto é, os candidatos mais prováveis de conter a resposta.
4. **Append the query and the segments**: o prompt enviado ao LLM é **aumentado** anexando os trechos recuperados como contexto adicional à pergunta original. É exatamente daí que vem o nome **Retrieval *Augmented* Generation**.
5. **LLM**: o modelo de linguagem gera a resposta final apoiando-se tanto no seu conhecimento pré-treinado quanto, sobretudo, no contexto recém-fornecido. Isso permite respostas mais atualizadas, ancoradas em dados específicos do domínio e com menor probabilidade de *alucinações*.
{: .fs-3 }

## Síntese

- A fase de **ingestão** prepara o conhecimento *uma vez*, transformando documentos em vetores e armazenando-os.
- A fase de **recuperação e geração** consulta esse conhecimento *a cada interação*, montando dinamicamente o contexto que o LLM utilizará.
{: .fs-3 }

Essa separação é o que diferencia RAG do *fine-tuning*: atualizar o conhecimento do sistema exige apenas reingerir documentos no banco vetorial, sem o custo de retreinar o modelo de linguagem.
{: .fs-3 }

> **Nota sobre *fine-tuning***
>
> *Fine-tuning* (ou **ajuste fino**) é o processo de continuar o treinamento de um modelo de linguagem já pré-treinado utilizando um conjunto de dados específico, com o objetivo de adaptar seus pesos a um domínio, estilo ou tarefa. É uma técnica poderosa, mas custosa: exige dados rotulados de qualidade, recursos computacionais (GPUs) e tempo, além de produzir um **novo modelo** que precisa ser versionado, hospedado e mantido.
>
> Diferença prática para o RAG:
>
> - O *fine-tuning* embute o conhecimento **nos pesos** do modelo. Toda atualização significativa do conteúdo exige um novo ciclo de treino.
> - O RAG mantém o conhecimento **fora do modelo**, em um banco vetorial. Para atualizar, basta reingerir os documentos.
>
> Por isso, RAG costuma ser a escolha natural quando o foco é trazer **informação factual e atualizada** (manuais, documentação, base de FAQs), enquanto o *fine-tuning* é mais adequado para ensinar ao modelo um **comportamento ou formato** específico (estilo de resposta, terminologia de uma área, formato estruturado de saída). As duas técnicas também podem ser combinadas.
{: .fs-3 }

# Referência

[Quarkus LangChain4j Workshop](https://quarkus.io/quarkus-workshop-langchain4j/)
{: .fs-3 }

<center>
<a href="https://rpmhub.dev" target="blanck"><img src="../imgs/logo.png" alt="Rodrigo Prestes Machado" width="3%" height="3%" border=0 style="border:0; text-decoration:none; outline:none"></a><br/>
<a rel="license" href="http://creativecommons.org/licenses/by/4.0/">CC BY 4.0 DEED</a>
</center>
