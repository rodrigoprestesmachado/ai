---
layout: default
title: Function Calling and Tools
nav_order: 5
---

# Function Calling e Tools

<center>
<iframe src="https://ai.rpmhub.dev/04tools/slides/index.html#/" title="Function Calling and Tools" width="90%" height="500" style="border:none;"></iframe>
</center>

**Function Calling** (ou chamada de função) é um mecanismo oferecido por alguns LLMs — como os modelos GPT e Llama — que permite ao modelo **invocar funções definidas pela aplicação** durante uma conversa. Em vez de apenas gerar texto, o modelo pode decidir, com base no contexto da conversa, chamar uma função específica com os parâmetros que julgar adequados, aguardar o resultado e então formular a resposta final ao usuário.
{: .fs-3 }

Enquanto o RAG amplia o conhecimento do LLM injetando contexto no prompt, o Function Calling vai além: ele permite que o LLM **execute ações reais** — consultar um banco de dados, cancelar uma reserva, chamar uma API externa ou qualquer lógica que a aplicação exponha como ferramenta.
{: .fs-3 }

## O ciclo de Function Calling

O ciclo completo envolve quatro participantes: o usuário, a aplicação, o LLM e as funções (tools) expostas pela aplicação.
{: .fs-3 }

O fluxo acontece da seguinte forma:
{: .fs-3 }

1. **Envio da lista de tools:** quando a aplicação envia a mensagem do usuário ao LLM, ela também inclui a lista de funções disponíveis, com nome, descrição e parâmetros de cada uma.
2. **Decisão do LLM:** o modelo analisa a conversa e decide se deve, ou não, invocar alguma função. Caso decida, retorna uma *tool call request* — um objeto estruturado indicando qual função chamar e com quais argumentos.
3. **Execução pela aplicação:** a aplicação recebe a solicitação, executa a função localmente (com acesso ao banco de dados, APIs, etc.) e envia o resultado de volta ao LLM.
4. **Resposta final:** o LLM usa o resultado da função para compor a resposta em linguagem natural para o usuário.
{: .fs-3 }

```
Usuário → Aplicação → LLM
                   ←  tool call request (qual função + parâmetros)
         Aplicação executa a função
         Aplicação → LLM (resultado da função)
                   ←  resposta final em linguagem natural
         Aplicação → Usuário
```
{: .fs-3 }

> **Atenção:** como o LLM define os parâmetros que serão passados para as funções, é fundamental **validar e sanitizar** as entradas antes de executar qualquer operação sensível, como exclusões ou gravações no banco de dados.
{: .fs-3 }

## Implementação com Quarkus LangChain4j

### Dependências

Para o exemplo do workshop, que persiste dados em um banco PostgreSQL via Hibernate ORM, são necessárias duas dependências adicionais no `pom.xml`:
{: .fs-3 }

```xml
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-hibernate-orm-panache</artifactId>
</dependency>
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-jdbc-postgresql</artifactId>
</dependency>
```
{: .fs-3 }

[Panache](https://quarkus.io/guides/hibernate-orm-panache) é uma camada sobre o Hibernate ORM que simplifica o acesso ao banco de dados no Quarkus, eliminando grande parte do código *boilerplate*.
{: .fs-3 }

### Entidades e exceções

O cenário do workshop modela um sistema de locação de veículos. As entidades `Customer` e `Booking` representam clientes e reservas, respectivamente:
{: .fs-3 }

```java
@Entity
public class Customer extends PanacheEntity {
    String firstName;
    String lastName;

    public static Optional<Customer> findByFirstAndLastName(String firstName, String lastName) {
        return find("LOWER(firstName) = LOWER(?1) and LOWER(lastName) = LOWER(?2)",
                    firstName, lastName).firstResultOptional();
    }
}
```
{: .fs-3 }

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
{: .fs-3 }

Além das entidades, são definidas exceções específicas para comunicar ao LLM situações como cliente não encontrado, reserva não encontrada ou reserva que não pode ser cancelada segundo as regras de negócio:
{: .fs-3 }

```java
public class Exceptions {
    public static class CustomerNotFoundException extends RuntimeException { ... }
    public static class BookingCannotBeCancelledException extends RuntimeException { ... }
    public static class BookingNotFoundException extends RuntimeException { ... }
}
```
{: .fs-3 }

Quando uma dessas exceções é lançada durante a execução de uma tool, o LLM recebe a mensagem de erro e pode comunicá-la ao usuário de forma natural.
{: .fs-3 }

### Definindo Tools com `@Tool`

Uma *tool* é simplesmente um **método Java anotado com `@Tool`**. A anotação aceita uma descrição textual que ajuda o LLM a entender para que serve a função e quando deve chamá-la.
{: .fs-3 }

No workshop, as tools são agrupadas em um repositório (`BookingRepository`) que implementa as operações de negócio:
{: .fs-3 }

```java
@ApplicationScoped
public class BookingRepository implements PanacheRepository<Booking> {

    @Tool("Cancel a booking")
    @Transactional
    public void cancelBooking(long bookingId,
                              String customerFirstName,
                              String customerLastName) {
        var booking = getBookingDetails(bookingId, customerFirstName, customerLastName);
        if (booking.dateFrom.minusDays(11).isBefore(LocalDate.now())) {
            throw new BookingCannotBeCancelledException(bookingId,
                "booking from date is 11 days before today");
        }
        if (booking.dateTo.minusDays(4).isBefore(booking.dateFrom)) {
            throw new BookingCannotBeCancelledException(bookingId,
                "booking period is less than four days");
        }
        delete(booking);
    }

    @Tool("List booking for a customer")
    @Transactional
    public List<Booking> listBookingsForCustomer(String customerName,
                                                  String customerSurname) {
        var found = Customer.findByFirstAndLastName(customerName, customerSurname);
        return found
            .map(customer -> list("customer", customer))
            .orElseThrow(() -> new CustomerNotFoundException(customerName, customerSurname));
    }

    @Tool("Get booking details")
    @Transactional
    public Booking getBookingDetails(long bookingId,
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
}
```
{: .fs-3 }

As três tools expostas são:
{: .fs-3 }

- **`cancelBooking`** — cancela uma reserva, verificando as regras de negócio (prazo mínimo de 11 dias e duração mínima de 4 dias).
- **`listBookingsForCustomer`** — lista todas as reservas de um cliente pelo nome.
- **`getBookingDetails`** — retorna os detalhes de uma reserva específica, validando que ela pertence ao cliente informado.
{: .fs-3 }

### Configurando o AI Service com `@ToolBox`

Para entregar a caixa de ferramentas ao LLM, basta anotar o método do AI Service com `@ToolBox`, passando a classe (ou classes) que contêm as tools:
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

            Today is {current_date}.
            """)
    @ToolBox(BookingRepository.class)
    String chat(String userMessage);
}
```
{: .fs-3 }

Dois pontos importantes nessa configuração:
{: .fs-3 }

1. **`@ToolBox(BookingRepository.class)`** — informa ao framework quais classes contêm os métodos anotados com `@Tool`. O Quarkus LangChain4j serializa automaticamente as assinaturas dessas funções e as inclui na requisição enviada ao LLM.
2. **`{current_date}`** — *placeholder* no system message que é substituído pela data atual em tempo de execução. Isso é essencial para que o LLM possa aplicar corretamente as regras de negócio baseadas em datas (como a política de cancelamento com 11 dias de antecedência).
{: .fs-3 }

## RAG vs. Function Calling

As duas técnicas são complementares e atendem a necessidades diferentes:
{: .fs-3 }

| Técnica | Quando usar | Como funciona |
|---|---|---|
| **RAG** | Responder perguntas com base em documentos ou bases de conhecimento | Recupera trechos relevantes e os injeta no prompt |
| **Function Calling** | Executar ações ou consultar dados em tempo real | O LLM solicita a execução de funções da aplicação |

Elas também podem ser **combinadas**: o LLM pode usar RAG para responder perguntas sobre a política da empresa e, ao mesmo tempo, usar tools para verificar e manipular dados de reservas.
{: .fs-3 }

# Referência

[Quarkus LangChain4j Workshop — Step 07](https://quarkus.io/quarkus-workshop-langchain4j/section-1/step-07/)
{: .fs-3 }

<center>
<a href="https://rpmhub.dev" target="blanck"><img src="../imgs/logo.png" alt="Rodrigo Prestes Machado" width="3%" height="3%" border=0 style="border:0; text-decoration:none; outline:none"></a><br/>
<a rel="license" href="http://creativecommons.org/licenses/by/4.0/">CC BY 4.0 DEED</a>
</center>
