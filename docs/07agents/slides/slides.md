<!-- .slide: class="title-slide" -->

# Implementing AI Agents

## Workshop · Section 2 · Step 01

Seu primeiro agente autônomo com Quarkus LangChain4j

<p class="small">Pressione <strong>F</strong> para tela cheia · <strong>ESC</strong> para visão geral · <strong>S</strong> para notas</p>

[quarkus.io/quarkus-workshop-langchain4j](https://quarkus.io/quarkus-workshop-langchain4j/section-2/step-01/)

---

## Agenda do Step 01

1. 🔄 **Transição:** AI Services → AI Agents
2. 🚗 **Cenário:** gestão de frota da Miles of Smiles
3. 🤖 **CleaningAgent:** `@Agent`, `@SystemMessage`, `@ToolBox`
4. 🔧 **CleaningTool:** function calling em agentes
5. 🎬 **Demo:** dois cenários de devolução de carro

<div class="destaque">
<strong>Objetivo:</strong> construir um agente autônomo que analisa feedback e decide se um carro precisa de limpeza — com ou sem invocar a tool.
</div>

Note: Este é o primeiro step da Section 2. Você já domina AI Services; agora o LLM toma decisões e ações, não apenas responde perguntas.

---

<!-- .slide: class="section-slide" -->

# Parte 1

## AI Services vs. AI Agents

---

## Uma mudança de paradigma

Na **Section 1**, você construiu um chatbot reativo:

```
Usuário pergunta → AI Service responde
```

Na **Section 2**, você constrói agentes autônomos:

```
Evento de negócio → Agente decide → Agente age (ou não)
```

<div class="destaque">
Agentes <strong>tomam decisões</strong> e <strong>executam ações</strong> com base em contexto — sem prompt explícito do usuário final.
</div>

---

## Comparativo

| Aspecto | AI Services | AI Agents |
|---|---|---|
| **Propósito** | Responder perguntas | Executar tarefas autônomas |
| **Interação** | Reativa | Reativa + proativa |
| **Anotação** | `@SystemMessage` + `@UserMessage` | `@Agent` (1 método) |
| **Workflows** | Single-agent | Multi-agente |
| **Casos de uso** | Chatbots, Q&A | Automação, orquestração |

---

## Dependência

```xml
<dependency>
    <groupId>io.quarkiverse.langchain4j</groupId>
    <artifactId>quarkus-langchain4j-agentic</artifactId>
</dependency>
```

O módulo `quarkus-langchain4j-agentic` habilita a anotação `@Agent` e a composição de workflows (Steps futuros).

---

<!-- .slide: class="section-slide" -->

# Parte 2

## Cenário e UI

---

## Gestão de frota

A Miles of Smiles precisa automatizar a devolução de carros:

1. Equipe registra **feedback** sobre o estado do carro
2. Sistema **decide** se precisa de limpeza
3. Após limpeza, carro volta ao pool **disponível**

<div class="dica">
Você construirá o <strong>CleaningAgent</strong> — especialista em decidir se um carro precisa de limpeza com base no feedback.
</div>

---

## Executando

```bash
cd section-2/step-01
./mvnw quarkus:dev
```

Abra `http://localhost:8080`:

* **Fleet Status** — grade com todos os carros e status
* **Action** — formulário de feedback para carros alugados ou em limpeza

---

## Demo: dois testes

<div class="two-col">
<div class="col">
<h3>Teste 1 — precisa limpeza</h3>
<pre><code>Car has dog hair all over
the back seat</code></pre>
<p>→ Status <code>AT_CLEANING</code></p>
<p>→ Tool invocada ✅</p>
</div>
<div class="col">
<h3>Teste 2 — carro limpo</h3>
<pre><code>Car looks good</code></pre>
<p>→ Status <code>AVAILABLE</code></p>
<p>→ <code>CLEANING_NOT_REQUIRED</code></p>
<p>→ Tool <strong>não</strong> invocada ✅</p>
</div>
</div>

Note: O segundo teste demonstra raciocínio — o agente decide NÃO agir, sem chamar a tool.

---

<!-- .slide: class="section-slide" -->

# Parte 3

## CleaningAgent

---

## A interface do agente

```java
public interface CleaningAgent {

    @SystemMessage("""
        You handle intake for the cleaning department...
        If no cleaning is needed, respond with
        "CLEANING_NOT_REQUIRED".
        """)
    @UserMessage("""
        Car Information:
        Make: {carInfo.make}
        Model: {carInfo.model}
        Year: {carInfo.year}
        Car Number: {carNumber}
        Feedback: {feedback}
        """)
    @Agent("Cleaning specialist...")
    @ToolBox(CleaningTool.class)
    String processCleaning(
            CarInfo carInfo,
            Integer carNumber,
            String feedback);
}
```

---

## Anotações-chave

<div class="two-col">
<div class="col">
<h3>@Agent</h3>
<ul>
<li>Marca o método como <strong>ponto de entrada</strong></li>
<li><strong>Apenas um</strong> por interface</li>
<li>Descrição usada por outros agentes/sistemas</li>
</ul>
</div>
<div class="col">
<h3>@SystemMessage</h3>
<ul>
<li><strong>Quem</strong> é o agente</li>
<li><strong>O que</strong> fazer</li>
<li><strong>Quando</strong> agir</li>
<li><strong>Como</strong> responder</li>
</ul>
</div>
</div>

<div class="alerta">
Sem implementação manual! LangChain4j gera o código: envia mensagens → LLM decide → invoca tool se necessário → retorna resposta.
</div>

---

## Integração no serviço

```java
@Inject
CleaningAgent cleaningAgent;

@Transactional
public String processCarReturn(
        Integer carNumber, String feedback) {

    String result = cleaningAgent.processCleaning(
            carInfo, carNumber, feedback);

    if (result.toUpperCase()
            .contains("CLEANING_NOT_REQUIRED")) {
        carInfo.status = CarStatus.AVAILABLE;
        carInfo.persist();
    }
    return result;
}
```

O agente é um bean CDI — injete e invoque como qualquer serviço Quarkus.

---

<!-- .slide: class="section-slide" -->

# Parte 4

## CleaningTool e fluxo

---

## A tool de limpeza

```java
@ApplicationScoped
public class CleaningTool {

    @Tool("Requests a cleaning with the specified options")
    @Transactional
    public String requestCleaning(
            Integer carNumber,
            String carMake, String carModel,
            Integer carYear,
            boolean exteriorWash,
            boolean interiorCleaning,
            boolean detailing, boolean waxing,
            String requestText) {

        // Atualiza status para AT_CLEANING
        CarInfo carInfo = CarInfo.findById(carNumber);
        if (carInfo != null) {
            carInfo.status = CarStatus.AT_CLEANING;
            carInfo.persist();
        }
        return generateCleaningSummary(...);
    }
}
```

Tools em agentes funcionam **igual** à Section 1 — `@Tool` + `@ToolBox`.

---

## Fluxo completo

```
Usuário → REST → Service → CleaningAgent → LLM
                                              │
                                    ┌─────────┴─────────┐
                                    ▼                   ▼
                              Chama Tool         CLEANING_NOT_REQUIRED
                              (limpeza)            (sem ação)
                                    │                   │
                                    ▼                   ▼
                              AT_CLEANING          AVAILABLE
```

---

<!-- .slide: class="section-slide" -->

# Parte 5

## Takeaways e próximos passos

---

## O que aprendemos

* **AI Agents** tomam decisões autônomas com base em contexto
* `@Agent` marca o ponto de entrada — **um método por interface**
* Reutiliza `@SystemMessage`, `@UserMessage`, `@ToolBox` da Section 1
* O agente pode **decidir não agir** — raciocínio sem tool call
* Integração CDI transparente — `@Inject CleaningAgent`
* Próximo: **workflows multi-agente** (Step 02)

---

## Experimentos

1. Edge cases: `"The trunk smells like fish"`, `"Spotless condition"`
2. Altere a system message para um especialista mais exigente
3. Adicione parâmetro `tireCleaning` — o agente aprende?

---

## Recursos & links

**Tutorial oficial**

* 📖 [Section 2, Step 01 — Implementing AI Agents](https://quarkus.io/quarkus-workshop-langchain4j/section-2/step-01/)

**Documentação**

* 📖 [LangChain4j — Agents](https://docs.langchain4j.dev/tutorials/agents/)
* 📖 [Quarkus LangChain4j — Agentic](https://docs.quarkiverse.io/quarkus-langchain4j/dev/agentic.html)

**Próximo step**

* ➡️ Step 02 — Creating simple agentic workflows
