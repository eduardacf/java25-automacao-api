# Java 25 — Framework de Automação de Testes de API

> Exercício prático em português brasileiro, com **Java 25, REST Assured, TestNG e Gradle**.

**Status:** projeto planejado. Os exemplos são atividades a implementar e validar.

## 1. Objetivo e escopo

Construir um framework reutilizável de testes funcionais para a [Restful Booker](https://restful-booker.herokuapp.com/apidoc/index.html), explorando recursos estáveis do Java 25.

**Tecnologias:** Java 25 LTS, Gradle, REST Assured, TestNG, Jackson, JSON Schema e GitHub Actions.

**Fora do escopo:** Maven, Allure, WireMock, testes de performance, comparações com Java 21 e recursos em preview.

**Regras:** classes, pacotes, métodos, variáveis, mensagens e documentação em português; nomes de campos exigidos pelo JSON da API permanecem como definidos no contrato. Cada teste deve preparar e limpar seus dados. Não publicar senhas ou tokens.

## 2. Endpoints

| Método | Rota | Finalidade |
|---|---|---|
| GET | `/ping` | Disponibilidade |
| POST | `/auth` | Autenticação |
| GET | `/booking` | Listagem e filtros |
| GET | `/booking/{id}` | Consulta |
| POST | `/booking` | Criação |
| PUT | `/booking/{id}` | Atualização completa |
| PATCH | `/booking/{id}` | Atualização parcial |
| DELETE | `/booking/{id}` | Exclusão |

A API é pública e compartilhada: não depender de reservas preexistentes ou de uma ordem fixa de resultados.

## 3. Estrutura planejada

```text
java25-automacao-api/
├── .github/workflows/testes-api.yml
├── gradle/wrapper/
├── src/main/java/br/com/automacao/
│   ├── configuracao/ConfiguracaoApi.java
│   ├── cliente/ClienteReserva.java
│   ├── cliente/ClienteAutenticacao.java
│   ├── contexto/ContextoTeste.java
│   ├── modelo/Reserva.java
│   ├── modelo/PeriodoReserva.java
│   ├── modelo/ResultadoOperacao.java
│   ├── fabrica/FabricaReservas.java
│   ├── validacao/ClassificadorResposta.java
│   └── utilitario/OrganizadorMassas.java
├── src/test/java/br/com/automacao/
│   ├── autenticacao/
│   ├── reserva/
│   ├── contrato/
│   ├── cenariosnegativos/
│   ├── concorrencia/
│   └── recursosjava25/
├── src/test/resources/esquemas/
├── build.gradle
├── settings.gradle
├── gradlew
├── gradlew.bat
└── README.md
```

## 4. Preparação e execução

Usar JDK 25 e Gradle **9.1 ou superior** com Wrapper versionado. Configurar a toolchain Java 25 e `useTestNG()` no `build.gradle`; fixar versões compatíveis de REST Assured, TestNG, Jackson e JSON Schema Validator.

```bash
./gradlew clean test
./gradlew test --tests "*CriacaoReservaTeste"
```

Relatório HTML nativo: `build/reports/tests/test/index.html`.

## 5. Recursos Java: explicação, código, exercício e aceite

Os códigos a seguir são exemplos isolados. Os imports e as integrações completas serão adicionados durante a implementação. Alguns recursos são anteriores ao Java 25, mas fazem parte do Java moderno utilizado no projeto.

### 5.1 Records

**O que é:** declaração concisa de objetos de dados imutáveis. **Por que usar:** representar requisições e respostas.

```java
public record PeriodoReserva(String entrada, String saida) {}

public record Reserva(
    String firstname,
    String lastname,
    int totalprice,
    boolean depositpaid,
    PeriodoReserva bookingdates,
    String additionalneeds
) {}
```

**Aplicação:** Jackson serializa `Reserva` em `POST /booking` e desserializa respostas. Os componentes em inglês correspondem aos campos obrigatórios do contrato JSON.

**Exercício:** criar reserva válida, enviar e conferir os campos retornados.

**Teste:** `ModeloReservaTeste.devePreservarDadosDaReserva()`.

**Aceite:** serialização correta e dados preservados. **Aprendizado:** modelos de dados e imutabilidade.

### 5.2 Construtores compactos

**O que é:** validação de invariantes dentro de records.

```java
public record PeriodoReserva(String entrada, String saida) {
    public PeriodoReserva {
        if (entrada == null || saida == null) {
            throw new IllegalArgumentException("Datas obrigatórias");
        }
    }
}
```

**Aplicação:** massas válidas. Testes negativos de campos ausentes devem usar payloads brutos ou modelos permissivos.

**Exercício:** tentar criar um período sem data.

**Teste:** `ModeloReservaTeste.deveRecusarPeriodoSemData()`.

**Aceite:** exceção esperada. **Aprendizado:** validação na construção.

### 5.3 Sealed Interfaces

**O que é:** hierarquia com implementações permitidas explicitamente.

```java
sealed interface ResultadoOperacao
    permits Sucesso, FalhaCliente, FalhaServidor {}

record Sucesso(int codigo) implements ResultadoOperacao {}
record FalhaCliente(int codigo) implements ResultadoOperacao {}
record FalhaServidor(int codigo) implements ResultadoOperacao {}
```

**Aplicação:** modelar resultados HTTP já classificados.

**Exercício:** instanciar e verificar os três tipos.

**Teste:** `ResultadoOperacaoTeste.deveRepresentarResultadosPermitidos()`.

**Aceite:** hierarquia fechada e testes aprovados. **Aprendizado:** modelagem de tipos.

### 5.4 Pattern Matching para switch

**O que é:** tratar valores de acordo com seus tipos.

```java
String descrever(ResultadoOperacao resultado) {
    return switch (resultado) {
        case Sucesso s -> "Sucesso: " + s.codigo();
        case FalhaCliente f -> "Erro do cliente: " + f.codigo();
        case FalhaServidor f -> "Erro do servidor: " + f.codigo();
    };
}
```

**Aplicação:** mensagens de resultados de operações.

**Exercício:** validar cada ramo.

**Teste:** `ClassificadorRespostaTeste.deveDescreverResultados()`.

**Aceite:** tratamento exaustivo. **Aprendizado:** pattern matching.

### 5.5 Record Patterns

**O que é:** extrair componentes de records em padrões.

```java
String resumir(Object objeto) {
    return switch (objeto) {
        case PeriodoReserva(String entrada, String saida) ->
            entrada + " até " + saida;
        default -> "Formato desconhecido";
    };
}
```

**Aplicação:** resumos de períodos nos diagnósticos.

**Exercício:** formatar datas de um record.

**Teste:** `PeriodoReservaTeste.deveResumirPeriodo()`.

**Aceite:** texto correto. **Aprendizado:** desestruturação.

### 5.6 ScopedValue

**O que é:** contexto imutável limitado a um escopo.

```java
public final class ContextoTeste {
    public static final ScopedValue<String> CENARIO =
        ScopedValue.newInstance();

    public static void executar(String nome, Runnable acao) {
        ScopedValue.where(CENARIO, nome).run(acao);
    }
}
```

```java
@Test
public void deveDisponibilizarContexto() {
    ContextoTeste.executar("CONSULTA", () ->
        Assert.assertEquals(ContextoTeste.CENARIO.get(), "CONSULTA")
    );
    Assert.assertFalse(ContextoTeste.CENARIO.isBound());
}
```

**Aplicação:** identificação de cenários em utilitários sem variável global mutável.

**Exercício:** executar dois contextos independentes.

**Teste:** `ContextoTesteTeste.deveIsolarCenarios()`.

**Aceite:** valor disponível apenas no escopo. **Aprendizado:** isolamento de contexto.

### 5.7 Stream Gatherers

**O que é:** operações avançadas de transformação de Streams.

```java
var nomes = List.of("Ana", "Bruna", "Carla", "Diana", "Eva");
var lotes = nomes.stream()
    .gather(Gatherers.windowFixed(2))
    .toList();
```

**Aplicação:** preparar massas agrupadas para DataProviders.

**Exercício:** dividir cinco nomes em lotes de dois.

**Teste:** `OrganizadorMassasTeste.devePreservarUltimoLote()`.

**Aceite:** três lotes, cinco nomes preservados. **Aprendizado:** janelas de dados.

### 5.8 Virtual Threads

**O que é:** threads leves para tarefas com espera de I/O.

```java
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    var tarefas = List.of(101, 102, 103).stream()
        .map(id -> executor.submit(() ->
            given()
                .baseUri("https://restful-booker.herokuapp.com")
            .when()
                .get("/booking/" + id)
                .statusCode()
        ))
        .toList();

    for (var tarefa : tarefas) {
        int codigo = tarefa.get();
        Assert.assertTrue(codigo > 0);
    }
}
```

**Aplicação:** consultas funcionais concorrentes. Na implementação final, criar os IDs no próprio teste e validar HTTP 200 e o corpo de cada resposta.

**Exercício:** consultar três reservas independentes.

**Teste:** `ConsultasConcorrentesTeste.deveConsultarReservas()`.

**Aceite:** todos os `Future` verificados, falhas propagadas. **Aprendizado:** concorrência segura, sem teste de carga.

### 5.9 Flexible Constructor Bodies

**O que é:** executar validações antes de `super()`.

```java
class ClienteBase {
    ClienteBase(String endereco) {}
}

class ClienteReserva extends ClienteBase {
    ClienteReserva(String endereco) {
        if (endereco == null || endereco.isBlank()) {
            throw new IllegalArgumentException("Endereço obrigatório");
        }
        super(endereco);
    }
}
```

**Aplicação:** validar configurações de clientes HTTP.

**Exercício:** testar endereço vazio e válido.

**Teste:** `ConfiguracaoClienteTeste.deveRecusarEnderecoVazio()`.

**Aceite:** mensagem correta e inicialização segura. **Aprendizado:** construtores.

### 5.10 Module Import Declarations

**O que é:** importar tipos públicos exportados por um módulo.

```java
import module java.base;

class UtilitarioColecoes {
    List<String> nomes() {
        return List.of("Ana", "Bruna");
    }
}
```

**Aplicação:** utilitários com muitos tipos de `java.base`.

**Exercício:** compilar e executar o utilitário.

**Teste:** `UtilitarioColecoesTeste.deveCriarLista()`.

**Aceite:** compilação correta. **Aprendizado:** sistema de módulos.

### 5.11 Sequenced Collections

**O que é:** métodos uniformes para coleções ordenadas.

```java
var nomes = List.of("Ana", "Bruna", "Carla");
var primeiro = nomes.getFirst();
var ultimo = nomes.getLast();
var invertidos = nomes.reversed();
```

**Aplicação:** verificar ordem de massas e resultados quando relevante.

**Exercício:** validar início, fim e ordem invertida.

**Teste:** `OrganizadorResultadosTeste.deveInverterSequencia()`.

**Aceite:** ordem correta e tratamento de lista vazia. **Aprendizado:** coleções ordenadas.

### 5.12 Switch Expressions

**O que é:** produzir resultados com uma expressão `switch`.

```java
String categoria(int codigo) {
    return switch (codigo / 100) {
        case 2 -> "Sucesso";
        case 4 -> "Erro do cliente";
        case 5 -> "Erro do servidor";
        default -> "Outra resposta";
    };
}
```

**Aplicação:** categorização de códigos HTTP.

**Exercício:** testar códigos 200, 404, 500 e 302.

**Teste:** `ClassificadorStatusTeste.deveClassificarCodigos()`.

**Aceite:** categorias corretas. **Aprendizado:** expressões condicionais.

### 5.13 Text Blocks

**O que é:** texto multilinha legível.

```java
String corpo = """
    {
      "firstname": "%s",
      "lastname": "Silva",
      "totalprice": 150
    }
    """.formatted("Maria");
```

**Aplicação:** payloads específicos de testes negativos; usar Jackson nos fluxos regulares.

**Exercício:** validar o JSON produzido.

**Teste:** `FabricaPayloadsTeste.deveGerarJsonValido()`.

**Aceite:** estrutura JSON válida. **Aprendizado:** templates de texto.

### 5.14 Optional e Streams

**O que é:** processamento declarativo e representação de ausência.

```java
Optional<String> nome = List.of("Ana", "Bruna").stream()
    .filter(valor -> valor.startsWith("B"))
    .findFirst();
```

**Aplicação:** filtrar massas de testes.

**Exercício:** testar busca existente e inexistente.

**Teste:** `FiltroMassasTeste.deveTratarResultadoAusente()`.

**Aceite:** ausência tratada sem `get()` inseguro. **Aprendizado:** programação funcional.

### 5.15 Compact Source Files

**O que é:** programa pequeno sem declaração explícita de classe principal.

Arquivo `VerificadorAmbiente.java`:

```java
void main() {
    IO.println("Verificação de ambiente Java 25");
    IO.println(System.getProperty("java.version"));
}
```

**Aplicação:** utilitário independente para verificar a instalação local.

**Exercício:** `java VerificadorAmbiente.java`.

**Aceite:** exibir versão do Java. **Aprendizado:** programas compactos.

### 5.16 Class-File API (opcional)

**O que é:** API estável para leitura, geração e transformação de arquivos `.class`.

**Aplicação:** utilitário educacional `InspetorClasses` para ler metadados de uma classe compilada do projeto com `java.lang.classfile.ClassFile`.

**Exercício:** identificar estrutura e métodos de uma classe do próprio repositório.

**Teste:** `InspetorClassesTeste.deveLerClasseCompilada()`.

**Aceite:** leitura correta, sem alterar bytecode. **Aprendizado:** estrutura de classes Java.

## 6. Matriz de rastreabilidade

| Recurso | Implementação | Teste | Situação |
|---|---|---|---|
| Records | `Reserva` | `ModeloReservaTeste` | Não iniciado |
| Construtor compacto | `PeriodoReserva` | `ModeloReservaTeste` | Não iniciado |
| Sealed Interfaces | `ResultadoOperacao` | `ResultadoOperacaoTeste` | Não iniciado |
| Pattern Matching | `ClassificadorResposta` | `ClassificadorRespostaTeste` | Não iniciado |
| Record Patterns | `PeriodoReserva` | `PeriodoReservaTeste` | Não iniciado |
| ScopedValue | `ContextoTeste` | `ContextoTesteTeste` | Não iniciado |
| Stream Gatherers | `OrganizadorMassas` | `OrganizadorMassasTeste` | Não iniciado |
| Virtual Threads | `ExecutorConsultas` | `ConsultasConcorrentesTeste` | Não iniciado |
| Flexible Constructors | `ClienteReserva` | `ConfiguracaoClienteTeste` | Não iniciado |
| Module Imports | `UtilitarioColecoes` | `UtilitarioColecoesTeste` | Não iniciado |
| Sequenced Collections | `OrganizadorResultados` | `OrganizadorResultadosTeste` | Não iniciado |
| Switch Expressions | `ClassificadorStatus` | `ClassificadorStatusTeste` | Não iniciado |
| Text Blocks | `FabricaPayloads` | `FabricaPayloadsTeste` | Não iniciado |
| Optional e Streams | `FiltroMassas` | `FiltroMassasTeste` | Não iniciado |
| Compact Source Files | `VerificadorAmbiente` | Execução independente | Não iniciado |
| Class-File API | `InspetorClasses` | `InspetorClassesTeste` | Opcional |

## 7. Cenários funcionais

| ID | Cenário | Resultado esperado |
|---|---|---|
| CT-001 | Verificar disponibilidade | Resposta conforme documentação |
| CT-002 | Autenticar com credenciais válidas | Token retornado |
| CT-003 | Autenticar com credenciais inválidas | Autenticação recusada |
| CT-004 | Criar reserva válida | Identificador retornado |
| CT-005 | Criar reserva sem depósito | Campo `depositpaid` preservado |
| CT-006 | Criar reserva sem campo obrigatório | Comportamento documentado |
| CT-007 | Criar reserva com JSON inválido | Requisição rejeitada |
| CT-008 | Consultar reserva criada | HTTP 200 e dados corretos |
| CT-009 | Consultar reserva inexistente | HTTP 404 |
| CT-010 | Listar reservas | Estrutura válida |
| CT-011 | Filtrar por nome | Filtro aplicado |
| CT-012 | Atualizar reserva completa | Dados atualizados |
| CT-013 | Atualizar parcialmente | Campos alterados corretamente |
| CT-014 | Atualizar sem autorização | Operação recusada |
| CT-015 | Excluir reserva | Exclusão confirmada |
| CT-016 | Consultar reserva excluída | Não encontrada |
| CT-017 | Validar JSON Schema da consulta | Contrato válido |
| CT-018 | Validar JSON Schema da criação | Contrato válido |
| CT-019 | Executar DataProvider | Todas as massas processadas |
| CT-020 | Isolar contextos | Nenhum vazamento de contexto |
| CT-021 | Consultar reservas simultaneamente | Respostas verificadas |
| CT-022 | Propagar falhas concorrentes | Falhas reportadas no TestNG |

**Nota:** resultados de entradas inválidas devem ser fundamentados no comportamento documentado e observado da API, sem pressupor regras que ela não implemente.

## 8. Etapas e critérios de aceite

### Etapa 1 — Preparação
- [ ] Instalar JDK 25 e configurar IntelliJ.
- [ ] Criar Gradle Wrapper compatível.
- [ ] Adicionar dependências.
- [ ] Executar primeiro teste.

### Etapa 2 — Arquitetura
- [ ] Criar configuração e clientes HTTP.
- [ ] Criar records e fábricas de massa.
- [ ] Centralizar validações.
- [ ] Garantir limpeza das reservas criadas.

### Etapa 3 — Testes funcionais
- [ ] Implementar autenticação e CRUD.
- [ ] Implementar cenários negativos.
- [ ] Adicionar JSON Schema.
- [ ] Implementar DataProviders.

### Etapa 4 — Recursos Java
- [ ] Implementar recursos da matriz.
- [ ] Criar testes associados.
- [ ] Documentar aprendizados.
- [ ] Validar concorrência funcional e isolamento.

### Etapa 5 — Publicação
- [ ] Configurar GitHub Actions.
- [ ] Disponibilizar relatórios HTML nativos.
- [ ] Usar segredos para credenciais.
- [ ] Atualizar a matriz com resultados reais.
- [ ] Publicar o repositório.

**Conclusão:** build Java 25 funcionando, testes independentes, cenários principais implementados, recursos escolhidos validados, documentação em português e pipeline executando com sucesso.

## 9. Referências

- [Documentação Java 25](https://docs.oracle.com/en/java/javase/25/)
- [OpenJDK 25](https://openjdk.org/projects/jdk/25/)
- [REST Assured](https://rest-assured.io/)
- [TestNG](https://testng.org/)
- [Gradle](https://docs.gradle.org/)
- [Restful Booker](https://restful-booker.herokuapp.com/apidoc/index.html)
- [GitHub Actions](https://docs.github.com/pt/actions)
