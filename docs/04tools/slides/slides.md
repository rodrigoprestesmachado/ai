<!-- .slide: class="title-slide" -->

# Function Calling e Tools

## Workshop · Section 1 · Step 07

Do RAG às ações reais: como dar superpoderes ao LLM

<p class="small">Pressione <strong>F</strong> para tela cheia · <strong>ESC</strong> para visão geral · <strong>S</strong> para notas</p>

[quarkus.io/quarkus-workshop-langchain4j](https://quarkus.io/quarkus-workshop-langchain4j/section-1/step-07/)

---

## Agenda do Step 07

1. 🤔 **Por que Function Calling?** O que o RAG não consegue fazer
2. 🔄 **O ciclo completo:** como o LLM chama funções
3. 🏗️ **Dependências e entidades:** preparando o terreno
4. 🔧 **Definindo Tools:** a anotação `@Tool`
5. 🎁 **Configurando o AI Service:** a anotação `@ToolBox`
6. 🎬 **Demo:** testando na prática

<div class="destaque">
<strong>Objetivo:</strong> entender como expor lógica de negócio ao LLM de forma segura, para que ele possa não só responder, mas também <em>agir</em>.
</div>

Note: Este step continua de onde o RAG parou. A ideia é mostrar que RAG e Function Calling são complementares.

---

<!-- .slide: class="section-slide" -->

# Parte 1

## Por que Function Calling?

---

## O que o RAG não resolve

RAG é excelente para **responder perguntas** com base em documentos.

Mas e quando o usuário quer **agir**?

```
Usuário: Quais são as minhas reservas?
RAG:     ❌ Não acessa o banco de dados em tempo real

Usuário: Pode cancelar minha reserva de número 3?
RAG:     ❌ Não executa operações no sistema

Usuário: Qual é o preço da diária hoje?
RAG:     ❌ Não consulta dados dinâmicos/atualizados
```

<div class="destaque">
RAG injeta <strong>conhecimento</strong> estático no prompt. Function Calling permite ao LLM executar <strong>ações reais</strong> na aplicação.
</div>

Note: O RAG foi projetado para busca de conhecimento. Para operações transacionais, precisamos de outra abordagem.

---

## A limitação fundamental dos LLMs

LLMs são, por natureza, **stateless e passivos**:

* Recebem um prompt → geram texto
* Não têm acesso ao banco de dados da sua aplicação
* Não conhecem dados criados após o treinamento
* Não podem executar nenhuma ação por conta própria

**Function Calling** é o mecanismo que quebra essa limitação:

> O LLM **solicita** a execução de uma função → a aplicação **executa** → o LLM **usa o resultado** para responder

Note: Importante frisar: o LLM nunca executa código diretamente. Ele apenas solicita. A aplicação mantém o controle total da execução.

---

## O que Function Calling resolve

<div class="two-col">
<div class="col">
<h3>✅ Casos de uso</h3>
<ul>
<li>Consultar dados em tempo real</li>
<li>Criar, atualizar ou deletar registros</li>
<li>Chamar APIs externas</li>
<li>Executar cálculos especializados</li>
<li>Verificar estoque, preços, agenda</li>
</ul>
</div>
<div class="col">
<h3>⚠️ Cuidados essenciais</h3>
<ul>
<li>Validar e sanitizar entradas</li>
<li>Limitar o escopo das funções expostas</li>
<li>Nunca expor operações destrutivas sem guardrails</li>
<li>Logar todas as chamadas para auditoria</li>
</ul>
</div>
</div>

Note: O LLM define os parâmetros das chamadas. Por isso, validação é obrigatória — nunca confie cegamente nos argumentos gerados pelo modelo.

---

<!-- .slide: class="section-slide" -->

# Parte 2

## O Ciclo de Function Calling

---

## O fluxo completo

```
1. Usuário envia mensagem
        ↓
2. Aplicação envia ao LLM:
   - Mensagem do usuário
   - Lista de tools disponíveis (nome + descrição + parâmetros)
        ↓
3. LLM decide chamar uma função
   → Retorna: { tool: "cancelBooking", args: { id: 3, ... } }
        ↓
4. Aplicação executa a função localmente
        ↓
5. Aplicação envia resultado de volta ao LLM
        ↓
6. LLM gera resposta final em linguagem natural
        ↓
7. Usuário recebe a resposta
```

Note: O LLM pode encadear múltiplas chamadas de função antes de dar a resposta final. Por exemplo: primeiro lista as reservas, depois cancela a selecionada.

---

## Passo a passo detalhado

| Etapa | Quem age | O que acontece |
|---|---|---|
| **1** | Usuário | Envia mensagem ("cancele minha reserva") |
| **2** | Aplicação | Serializa as tools e envia tudo ao LLM |
| **3** | LLM | Analisa o contexto e escolhe qual tool invocar |
| **4** | LLM | Retorna *tool call request* com função + argumentos |
| **5** | Aplicação | Executa a função, valida dados, acessa o banco |
| **6** | Aplicação | Envia o resultado da função de volta ao LLM |
| **7** | LLM | Formula resposta em linguagem natural |
| **8** | Aplicação | Entrega a resposta ao usuário |

<div class="destaque">
A aplicação <strong>nunca perde o controle</strong>. O LLM apenas solicita — a execução é sempre da aplicação.
</div>

---

## RAG vs. Function Calling

<div class="two-col">
<div class="col">
<h3>📚 RAG</h3>
<ul>
<li>Busca em documentos estáticos</li>
<li>Injeção de contexto no prompt</li>
<li>Somente leitura</li>
<li>Ideal para: FAQs, políticas, manuais</li>
<li>Conhecimento atualizado por reingestão</li>
</ul>
</div>
<div class="col">
<h3>🔧 Function Calling</h3>
<ul>
<li>Acesso a dados dinâmicos em tempo real</li>
<li>Execução de lógica de negócio</li>
<li>Leitura e escrita</li>
<li>Ideal para: reservas, pedidos, contas</li>
<li>Dados sempre atualizados via banco/API</li>
</ul>
</div>
</div>

> As duas técnicas são **complementares** e podem ser usadas juntas no mesmo AI Service.

Note: Exemplo de uso combinado: RAG para responder sobre a política de cancelamento; Function Calling para executar o cancelamento em si.

---

<!-- .slide: class="section-slide" -->

# Parte 3

## Dependências e Entidades

---

## Dependências necessárias

Para o cenário do workshop (banco PostgreSQL via Hibernate ORM):

```xml
<!-- Panache: camada simplificada sobre Hibernate ORM -->
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-hibernate-orm-panache</artifactId>
</dependency>

<!-- Driver JDBC para PostgreSQL -->
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-jdbc-postgresql</artifactId>
</dependency>
```

Ou via CLI:

```bash
./mvnw quarkus:add-extension \
  -Dextensions="hibernate-orm-panache,jdbc-postgresql"
```

Note: Panache elimina muito código boilerplate do Hibernate. Com PanacheEntity, o ID já está incluído e operações básicas de CRUD ficam disponíveis diretamente na classe.

---

## Entidade Customer

```java
@Entity
public class Customer extends PanacheEntity {

    String firstName;
    String lastName;

    public static Optional<Customer> findByFirstAndLastName(
            String firstName, String lastName) {
        return find(
            "LOWER(firstName) = LOWER(?1) and LOWER(lastName) = LOWER(?2)",
            firstName, lastName
        ).firstResultOptional();
    }
}
```

* `PanacheEntity` fornece o campo `id` e os métodos CRUD básicos
* O método de busca usa `LOWER()` para pesquisa sem distinção de maiúsculas
* O retorno `Optional<Customer>` força o tratamento de "não encontrado"

Note: A busca case-insensitive é importante porque o LLM pode capitalizar nomes de formas diferentes a cada interação.

---

## Entidade Booking

```java
@Entity
public class Booking extends PanacheEntity {

    @ManyToOne
    Customer customer;

    LocalDate dateFrom;
    LocalDate dateTo;
    String location;
}
```

* Relacionamento `@ManyToOne` com `Customer`
* Datas em `LocalDate` (tipo mais seguro que `Date` para lógica de calendário)
* Um cliente pode ter **múltiplas** reservas

---

## Exceções de negócio

```java
public class Exceptions {

    public static class CustomerNotFoundException
            extends RuntimeException {
        public CustomerNotFoundException(String firstName, String lastName) {
            super("Customer not found: %s %s".formatted(firstName, lastName));
        }
    }

    public static class BookingNotFoundException
            extends RuntimeException {
        public BookingNotFoundException(long bookingId) {
            super("Booking %d not found".formatted(bookingId));
        }
    }

    public static class BookingCannotBeCancelledException
            extends RuntimeException {
        public BookingCannotBeCancelledException(long id, String reason) {
            super("Booking %d cannot be cancelled because %s"
                    .formatted(id, reason));
        }
    }
}
```

<div class="destaque">
Quando uma exceção é lançada dentro de uma tool, o LLM recebe a mensagem de erro e a comunica ao usuário de forma natural.
</div>

Note: Mensagens de exceção claras e em inglês ajudam o LLM a construir respostas mais precisas para o usuário.

---

<!-- .slide: class="section-slide" -->

# Parte 4

## Definindo Tools com `@Tool`

---

## A anotação `@Tool`

Para expor um método ao LLM, basta anotá-lo com `@Tool`:

```java
@Tool("Descrição do que a função faz")
public TipoRetorno nomeDaFuncao(Parametro1 p1, Parametro2 p2) {
    // lógica de negócio
}
```

O framework:
1. Lê o nome do método, a descrição e os tipos dos parâmetros
2. Serializa tudo em JSON Schema
3. Envia esse schema ao LLM junto com cada mensagem
4. O LLM usa essa informação para decidir se e como chamar a função

<div class="destaque">
A <strong>descrição</strong> é crucial: quanto mais clara e específica, melhor o LLM saberá quando e como usar a tool.
</div>

Note: O nome dos parâmetros também importa — o LLM usa nomes semânticos para inferir o significado. "customerFirstName" é melhor do que "param1".

---

## Tool: listar reservas

```java
@Tool("List booking for a customer")
@Transactional
public List<Booking> listBookingsForCustomer(
        String customerName,
        String customerSurname) {

    var found = Customer.findByFirstAndLastName(
                    customerName, customerSurname);

    return found
        .map(customer -> list("customer", customer))
        .orElseThrow(() ->
            new CustomerNotFoundException(customerName, customerSurname));
}
```

* O LLM chama essa tool quando o usuário quer ver suas reservas
* Se o cliente não for encontrado, a exceção é propagada ao LLM
* `@Transactional` garante consistência no acesso ao banco

---

## Tool: detalhes de uma reserva

```java
@Tool("Get booking details")
@Transactional
public Booking getBookingDetails(
        long bookingId,
        String customerFirstName,
        String customerLastName) {

    var found = findByIdOptional(bookingId)
        .orElseThrow(() -> new BookingNotFoundException(bookingId));

    if (!found.customer.firstName.equals(customerFirstName)
            || !found.customer.lastName.equals(customerLastName)) {
        throw new BookingNotFoundException(bookingId);
    }
    return found;
}
```

* Validação dupla: a reserva existe **e** pertence ao cliente
* Evita que um cliente acesse dados de outro — **segurança**

Note: Esta é uma validação de autorização simples. Em produção, use tokens de sessão e nunca confie apenas nos dados fornecidos pelo LLM.

---

## Tool: cancelar reserva

```java
@Tool("Cancel a booking")
@Transactional
public void cancelBooking(long bookingId,
                           String customerFirstName,
                           String customerLastName) {

    var booking = getBookingDetails(
                    bookingId, customerFirstName, customerLastName);

    // Regra 1: cancelamento com mínimo 11 dias de antecedência
    if (booking.dateFrom.minusDays(11).isBefore(LocalDate.now())) {
        throw new BookingCannotBeCancelledException(bookingId,
            "booking from date is 11 days before today");
    }

    // Regra 2: reservas com menos de 4 dias não podem ser canceladas
    if (booking.dateTo.minusDays(4).isBefore(booking.dateFrom)) {
        throw new BookingCannotBeCancelledException(bookingId,
            "booking period is less than four days");
    }

    delete(booking);
}
```

Note: As regras de negócio ficam no código Java, não no prompt. Isso é intencional: regras críticas não devem depender da interpretação do LLM.

---

<!-- .slide: class="section-slide" -->

# Parte 5

## Configurando o AI Service com `@ToolBox`

---

## A anotação `@ToolBox`

```java
@SessionScoped
@RegisterAiService
public interface CustomerSupportAgent {

    @SystemMessage("""
            You are a customer support agent of a car rental company
            'Miles of Smiles'. You are friendly, polite and concise.
            If the question is unrelated to car rental, politely redirect
            the customer to the right department.

            When calling tools or functions, strictly use JSON objects,
            do not wrap in quotes or use plain strings.

            Today is {current_date}.
            """)
    @ToolBox(BookingRepository.class)        // ← expõe as tools ao LLM
    String chat(String userMessage);
}
```

Note: @SessionScoped garante que o histórico da conversa seja mantido por sessão de usuário. Isso é essencial para que o LLM lembre o nome do cliente entre turnos da conversa.

---

## Dois detalhes importantes

<div class="two-col">
<div class="col">
<h3>@ToolBox</h3>
<ul>
<li>Recebe uma ou mais classes como argumento</li>
<li>O framework escaneia os métodos <code>@Tool</code> automaticamente</li>
<li>Serializa as assinaturas em JSON Schema</li>
<li>Inclui o schema em cada requisição ao LLM</li>
</ul>
</div>
<div class="col">
<h3>{current_date}</h3>
<ul>
<li>Placeholder resolvido em tempo de execução</li>
<li>Usa o motor de templates Qute</li>
<li>Essencial para regras baseadas em data</li>
<li>Sem isso, o LLM não sabe se 11 dias já passaram</li>
</ul>
</div>
</div>

<div class="destaque">
Placeholders no system message permitem injetar contexto dinâmico — data atual, nome do usuário logado, configurações do tenant, etc.
</div>

---

<!-- .slide: class="section-slide" -->

# Parte 6

## Demo: testando na prática

---

## Conversa 1: listando e cancelando

<div class="two-col">
<div class="col">
<h3>💬 Conversa</h3>
<p><strong>Usuário:</strong> Olá, gostaria de cancelar uma reserva.</p>
<p><strong>AI:</strong> Claro! Pode me informar seu nome e o ID da reserva?</p>
<p><strong>Usuário:</strong> Sou Speedy McWheels. Não lembro o ID. Pode listar?</p>
<p><strong>AI:</strong> Suas reservas: #1 Verbier (amanhã), #2 São Paulo (+14 dias), #3 Antuérpia (+30 dias). Qual deseja cancelar?</p>
<p><strong>Usuário:</strong> A última.</p>
<p><strong>AI:</strong> Reserva #3 cancelada com sucesso!</p>
</div>
<div class="col">
<h3>⚙️ Nos bastidores</h3>
<ol>
<li>LLM chama <code>listBookingsForCustomer</code></li>
<li>Aplicação consulta o banco</li>
<li>LLM recebe a lista e apresenta ao usuário</li>
<li>LLM infere "última" = reserva #3</li>
<li>LLM chama <code>cancelBooking(3, ...)</code></li>
<li>Aplicação valida as regras e deleta</li>
<li>LLM confirma ao usuário</li>
</ol>
</div>
</div>

Note: O LLM encadeou duas chamadas de função para completar a tarefa. Primeiro listou, depois cancelou. Isso é o comportamento esperado de um agente.

---

## Conversa 2: regra de negócio aplicada

**Usuário:** Pode cancelar a reserva que começa amanhã?

**AI:** Infelizmente não é possível cancelar essa reserva. De acordo com nossa política, cancelamentos devem ser solicitados com pelo menos 11 dias de antecedência. Se precisar de ajuda com outra reserva, estou à disposição!

---

**O que aconteceu:**

1. LLM chamou `cancelBooking` com os dados da reserva
2. A função verificou: `dateFrom - 11 dias < hoje` → lançou `BookingCannotBeCancelledException`
3. O LLM recebeu a mensagem de erro e a transformou em resposta amigável

<div class="destaque">
As <strong>regras de negócio vivem no código Java</strong>, não no prompt. O LLM apenas comunica o resultado ao usuário.
</div>

---

## O que aprendemos

* **Function Calling** complementa o RAG, permitindo ao LLM **executar ações** além de responder perguntas
* **`@Tool`** marca um método Java como função chamável pelo LLM
* **`@ToolBox`** conecta as tools ao AI Service
* **Exceções** são o mecanismo para comunicar erros e regras de negócio ao LLM
* O LLM pode **encadear múltiplas chamadas** de função para completar uma tarefa
* A aplicação **mantém o controle**: o LLM solicita, a aplicação executa

---

## Recursos & links

**Tutorial oficial**

* 📖 [Step 07, Function calling and Tools](https://quarkus.io/quarkus-workshop-langchain4j/section-1/step-07/)

**Documentação**

* 📖 [Quarkus LangChain4j, Tools](https://docs.quarkiverse.io/quarkus-langchain4j/dev/agent-and-tools.html)
* 📖 [Quarkus Hibernate ORM com Panache](https://quarkus.io/guides/hibernate-orm-panache)

**Conceitos**

* 🔗 [OpenAI, Function Calling](https://platform.openai.com/docs/guides/function-calling)
* 🔗 [LangChain4j, Tools & Agents](https://docs.langchain4j.dev/tutorials/tools)
