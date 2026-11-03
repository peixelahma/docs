# Java + Spring: boas práticas para código legível por IA

Nov 1, 2026 · @Julio

Assumi **Java 21 + Spring Boot 3.3+** (Maven ou Gradle indiferente). A IA acerta mais quando o código é **explícito, local e previsível**; o custo de tokens vem de ter que abrir muitos arquivos para entender uma única mudança.

## 1. Premissas: como a IA lê código

| Princípio | Por que importa para a IA | Regra prática |
| --- | --- | --- |
| Localidade | Tudo que muda junto fica no mesmo pacote; menos arquivos lidos por tarefa | Package-by-feature, não package-by-layer |
| Explícito > mágico | Reflection, AOP e geração de código não aparecem no texto que a IA lê | Preferir código visível; usar mágica só onde o Spring já é padrão |
| Uma convenção por problema | A IA copia o padrão que vê; dois padrões geram um terceiro, inconsistente | Um único jeito de fazer query, DTO, erro, client |
| Tipos fortes e nomes descritivos | O nome substitui leitura da implementação | Records, enums, nomes de método que descrevem a intenção |
| Arquivos pequenos e coesos | Cabem inteiros no contexto e geram diffs pequenos | Classe até \~250 linhas, método até \~30 |
| Pouca indireção | Cada salto (interface, evento, base class) custa uma leitura | Sem interface com uma só implementação; sem herança profunda |
| Feedback automático | Teste, compilação e regra de arquitetura corrigem a IA sem você | Testes rápidos + ArchUnit rodando no build |
| Convenções por escrito | A IA não adivinha decisões do time | Arquivo AGENTS.md / CLAUDE.md curto na raiz do projeto |

## 2. Estrutura do projeto e convenções

Organize por **funcionalidade (feature)**, com sufixos fixos nos nomes: a IA localiza e cria arquivos sem explorar o projeto inteiro.

```text
com.empresa.app
├── pedido/                    # uma feature = um pacote
│   ├── PedidoController.java
│   ├── PedidoService.java
│   ├── PedidoRepository.java
│   ├── Pedido.java            # entity
│   ├── CriarPedidoRequest.java    # record
│   ├── PedidoResponse.java        # record
│   └── PedidoNaoEncontradoException.java
├── pagamento/
│   ├── PagamentoClient.java   # integração externa
│   └── ...
└── shared/
    ├── config/                # SecurityConfig, ClientConfig, AsyncConfig
    ├── error/                 # DomainException, GlobalExceptionHandler
    └── messaging/             # envelope de eventos, helpers
```

| Tipo | Sufixo / forma | Regra |
| --- | --- | --- |
| Entrada HTTP | `XxxController` | Só traduz HTTP; zero regra de negócio |
| Regra de negócio | `XxxService` (classe concreta) | Dono da transação (`@Transactional`) |
| Acesso a dados | `XxxRepository` | Interface Spring Data, sem implementação customizada |
| DTO | `XxxRequest` / `XxxResponse` / `XxxEvent` | `record`, imutável |
| Integração externa | `XxxClient` | Uma classe por sistema externo |
| Configuração | `XxxConfig`, `XxxProperties` | `@ConfigurationProperties` com `record` |
| Visibilidade | package-private por padrão | `public` só o que outra feature realmente usa |

- **Lombok:** evitar em código novo. Records cobrem DTOs; getters explícitos em entities são curtos e visíveis. Se usar, limitar a `@Getter` e `@Slf4j` (nunca `@Data` ou `@Builder` em entity).
- **Java moderno ajuda a IA:** `record`, `sealed`, `switch` com pattern matching e `var` reduzem boilerplate sem esconder lógica.
- **Configuração:** `application.yml` único por perfil; nada de XML; nada de `@Value` espalhado (usar `@ConfigurationProperties`).

## 3. Injeção de dependência

**Injeção por construtor é benéfica para a IA** (o grafo de dependências fica escrito em um lugar só); o que prejudica é o **excesso** e a **indireção desnecessária**.

| Prática | Efeito para a IA | Veredito |
| --- | --- | --- |
| Construtor único, campos `final` | Dependências visíveis na assinatura, fácil de testar e de mockar | Usar sempre |
| `@Autowired` em campo | Dependência escondida, teste exige reflection | Proibir |
| Até 5 dependências por classe | Cabe na cabeça (e no contexto) | Limite; acima disso, dividir a classe |
| Interface com uma única implementação | Dobra os arquivos lidos, zero ganho | Evitar; criar interface só com 2+ implementações reais |
| `@Qualifier` / `@Primary` espalhados | Resolução implícita por nome de bean | Evitar; preferir tipos distintos |
| Cadeia Service → Service → Service | A IA precisa abrir cada salto | Máximo 1 nível; compartilhar via Repository ou componente de domínio |
| `ApplicationContext.getBean()`, `@PostConstruct` com lógica | Fluxo invisível | Proibir |
| `ApplicationEvent` interno para fluxo de negócio | Quem escuta não aparece no código que publica | Só para efeitos colaterais desacoplados (ex.: auditoria) |

**Boilerplate (service):**

```java
@Service
class PedidoService {

    private final PedidoRepository repository;
    private final PagamentoClient pagamentoClient;

    PedidoService(PedidoRepository repository, PagamentoClient pagamentoClient) {
        this.repository = repository;
        this.pagamentoClient = pagamentoClient;
    }

    @Transactional
    public PedidoResponse criar(CriarPedidoRequest request) {
        // 1. validar regra  2. persistir  3. integrar  4. responder
    }
}
```

## 4. Persistência com banco de dados

**Recomendação:** usar Spring Data JPA em escada: *derived query* para casos simples, **`@Query` com JPQL** para o resto e **Specification apenas para filtros dinâmicos**. A JPQL escrita no repositório é a opção mais legível para a IA: o texto da consulta está visível, é validado na subida da aplicação e custa poucos tokens.

| Abordagem | Legibilidade para IA | Tokens | Erro aparece | Quando usar |
| --- | --- | --- | --- | --- |
| Derived query (`findByStatusAndClienteId`) | Alta se até 2 condições; o nome é a especificação | Mínimo (1 linha) | Na subida | Consultas simples, até 2 condições |
| `@Query` JPQL com parâmetros nomeados | Alta; a IA lê como SQL | Baixo | Na subida (JPQL é validada) | 3+ condições, joins, ordenação, projeções. **Padrão recomendado** |
| `@Query(nativeQuery = true)` | Alta, mas acoplada ao banco | Baixo | Só em execução | Relatórios, funções específicas do banco |
| `Specification` | Média; o fluxo de composição é espalhado | Alto | Em execução | Busca com muitos filtros opcionais combináveis |
| QueryDSL / jOOQ | Alta no uso, mas exige geração de código | Alto (classes `Q*` geradas) | Na compilação | Só se o projeto já adota |
| Criteria API manual | Baixa; verbosa | Muito alto | Em execução | Evitar |
| Spring Data JDBC | Alta; sem proxies, lazy loading ou cache de sessão | Baixo | Na subida | Alternativa mais previsível ao JPA para domínios simples |

**Regras de persistência:**

- Schema versionado com **Flyway em SQL puro**: é a fonte de verdade que a IA lê; `spring.jpa.hibernate.ddl-auto=validate`.
- **Nunca expor entity** na API; leituras usam **projeção em `record`**.
- `@Transactional` no Service; `readOnly = true` nas consultas.
- Relações `LAZY` por padrão; resolver N+1 com `@EntityGraph` ou `join fetch` na própria `@Query`.
- Sem relação bidirecional, salvo necessidade real.
- Sem `RepositoryImpl` customizado nem `EntityManager` solto no Service.
- Proibido: filtrar com `findAll()` em memória; concatenar string em query; nomes derivados com mais de 3 propriedades.
- Controle de concorrência com `@Version` (lock otimista) nas entities que sofrem atualização simultânea.

**Boilerplate (repository):**

```java
interface PedidoRepository extends JpaRepository<Pedido, Long> {

    // Simples: derived query
    List<Pedido> findByClienteIdAndStatus(Long clienteId, StatusPedido status);

    // Padrão: JPQL + projeção em record
    @Query("""
        select new com.empresa.app.pedido.PedidoResumo(p.id, p.status, p.total, c.nome)
        from Pedido p
        join p.cliente c
        where p.criadoEm >= :desde
          and (:status is null or p.status = :status)
        order by p.criadoEm desc
        """)
    Page<PedidoResumo> buscarResumos(@Param("desde") Instant desde,
                                     @Param("status") StatusPedido status,
                                     Pageable pageable);
}

record PedidoResumo(Long id, StatusPedido status, BigDecimal total, String cliente) {}
```

**Boilerplate (Specification, só para filtro dinâmico):** uma classe `XxxSpecs` com métodos estáticos nomeados, sem lógica no Service.

```java
final class PedidoSpecs {
    private PedidoSpecs() {}

    static Specification<Pedido> comStatus(StatusPedido status) {
        return (root, q, cb) -> status == null ? null : cb.equal(root.get("status"), status);
    }

    static Specification<Pedido> criadoApos(Instant desde) {
        return (root, q, cb) -> desde == null ? null : cb.greaterThanOrEqualTo(root.get("criadoEm"), desde);
    }
}
// uso: repository.findAll(where(comStatus(s)).and(criadoApos(d)), pageable);
```

## 5. DTOs, mapeamento e validação

**Recomendação:** `record` imutável por caso de uso, validação declarativa no próprio record e **mapeamento manual** dentro do record de resposta. Para a IA, um método `from(...)` visível vale mais do que um mapper gerado em tempo de compilação.

| Decisão | Recomendado | Motivo |
| --- | --- | --- |
| Tipo do DTO | `record` | Sem getters, sem Lombok, menos tokens |
| Request e Response | Separados, um par por caso de uso (`CriarPedidoRequest`, `PedidoResponse`) | Evita campo nulo "às vezes" e DTO genérico gigante |
| Validação | Bean Validation (`@NotNull`, `@Size`, `@Valid`) no record | Regra de entrada visível ao lado do campo |
| Mapeamento | Método estático `from(Entity)` no Response; construção explícita da entity no Service | Código visível, sem etapa de geração |
| MapStruct | Só com mais de \~15 campos por mapeamento ou muitos mapeamentos repetidos | Gera código fora do contexto da IA |
| DTO de integração externa | Record separado do DTO da API própria (`PagamentoApiRequest`) | Mudança do terceiro não vaza para o seu contrato |
| Valores monetários e datas | `BigDecimal`, `Instant` / `LocalDate`; nunca `double` ou `Date` | Evita bugs que a IA reproduz por imitação |

**Boilerplate:**

```java
public record CriarPedidoRequest(
        @NotNull Long clienteId,
        @NotEmpty @Valid List<ItemRequest> itens
) {
    public record ItemRequest(@NotNull Long produtoId, @Positive int quantidade) {}
}

public record PedidoResponse(Long id, StatusPedido status, BigDecimal total, Instant criadoEm) {
    static PedidoResponse from(Pedido p) {
        return new PedidoResponse(p.getId(), p.getStatus(), p.getTotal(), p.getCriadoEm());
    }
}

@RestController
@RequestMapping("/pedidos")
class PedidoController {
    private final PedidoService service;

    PedidoController(PedidoService service) { this.service = service; }

    @PostMapping
    ResponseEntity<PedidoResponse> criar(@Valid @RequestBody CriarPedidoRequest request) {
        PedidoResponse response = service.criar(request);
        return ResponseEntity.status(HttpStatus.CREATED).body(response);
    }
}
```

## 6. Tratamento de exceções

**Recomendação:** hierarquia curta de exceções de domínio (*unchecked*), **um único** `@RestControllerAdvice` e resposta no formato `ProblemDetail` (RFC 9457, nativo no Spring 6). A IA aprende o fluxo uma vez e o repete: lançar a exceção certa no Service, nunca montar resposta de erro manualmente.

| Regra | Detalhe |
| --- | --- |
| Exceções *unchecked* | Nada de `throws` em cadeia; a IA não precisa propagar assinaturas |
| Hierarquia de 4 tipos | `NotFoundException` (404), `ConflictException` (409), `BusinessRuleException` (422), `ExternalServiceException` (502) |
| Código de erro estável | Campo `code` (ex.: `PEDIDO_NAO_ENCONTRADO`) para o cliente da API e para os testes |
| Um único handler | `GlobalExceptionHandler` em `shared/error`; nenhum `try/catch` de negócio no Controller |
| Sem `catch (Exception)` genérico | Só no handler (fallback 500) e em fronteiras de integração, sempre re-lançando exceção própria com a causa |
| Log uma vez | Logar no handler, não em cada camada (evita log duplicado) |
| Mensagem sem dado sensível | Nada de CPF, token ou SQL na resposta |

**Boilerplate:**

```java
public abstract class DomainException extends RuntimeException {
    private final String code;
    protected DomainException(String code, String message) { this(code, message, null); }
    protected DomainException(String code, String message, Throwable cause) { super(message, cause); this.code = code; }
    public String code() { return code; }
}

public class NotFoundException extends DomainException {
    public NotFoundException(String code, String message) { super(code, message); }
}
// Uso no Service:
// repository.findById(id).orElseThrow(() -> new NotFoundException("PEDIDO_NAO_ENCONTRADO", "Pedido " + id));

@RestControllerAdvice
class GlobalExceptionHandler {

    private static final Logger log = LoggerFactory.getLogger(GlobalExceptionHandler.class);

    @ExceptionHandler(NotFoundException.class)
    ProblemDetail notFound(NotFoundException ex) {
        return problem(HttpStatus.NOT_FOUND, ex);
    }

    @ExceptionHandler(ConflictException.class)
    ProblemDetail conflict(ConflictException ex) {
        return problem(HttpStatus.CONFLICT, ex);
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    ProblemDetail invalid(MethodArgumentNotValidException ex) {
        ProblemDetail pd = ProblemDetail.forStatusAndDetail(HttpStatus.BAD_REQUEST, "Dados inválidos");
        pd.setProperty("code", "VALIDACAO");
        pd.setProperty("campos", ex.getFieldErrors().stream()
                .collect(Collectors.toMap(FieldError::getField, FieldError::getDefaultMessage, (a, b) -> a)));
        return pd;
    }

    @ExceptionHandler(Exception.class)
    ProblemDetail unexpected(Exception ex) {
        log.error("Erro não tratado", ex);
        ProblemDetail pd = ProblemDetail.forStatusAndDetail(HttpStatus.INTERNAL_SERVER_ERROR, "Erro interno");
        pd.setProperty("code", "ERRO_INTERNO");
        return pd;
    }

    private ProblemDetail problem(HttpStatus status, DomainException ex) {
        ProblemDetail pd = ProblemDetail.forStatusAndDetail(status, ex.getMessage());
        pd.setProperty("code", ex.code());
        return pd;
    }
}
```

## 7. Autenticação e autorização

**Recomendação:** API *stateless* com **Spring Security como OAuth2 Resource Server (JWT)**, uma única classe `SecurityConfig` com a DSL em lambda e autorização por método com `@PreAuthorize`. Quem emite o token (Keycloak, Cognito, Auth0) fica fora da aplicação; a IA não precisa gerar código de parsing ou assinatura de JWT.

| Decisão | Recomendado | Evitar |
| --- | --- | --- |
| Sessão | Stateless (`SessionCreationPolicy.STATELESS`) | Sessão em memória do servidor |
| Validação do token | `spring-boot-starter-oauth2-resource-server` com `issuer-uri` no `application.yml` | Filtro próprio que decodifica JWT |
| Configuração | Um bean `SecurityFilterChain` (DSL em lambda, Spring Security 6) | `WebSecurityConfigurerAdapter` (removido) |
| Autorização por rota | Regras gerais na `SecurityConfig` (`/actuator/health`, `/publico/**`) | Regras espalhadas em vários filtros |
| Autorização por regra de negócio | `@PreAuthorize("hasAuthority('pedido:escrever')")` no método do Controller/Service | `if (user.isAdmin())` dentro do código de negócio |
| Usuário atual | `@AuthenticationPrincipal Jwt jwt` ou um componente `CurrentUser` | Ler `SecurityContextHolder` em qualquer lugar |
| Autoridades | Constantes/escopos nomeados, convenção `recurso:acao` | Strings soltas e papéis implícitos |
| Segredos | Variáveis de ambiente / secret manager | Chaves no repositório |
| Testes | `@WithMockUser` / `jwt()` do `SecurityMockMvcRequestPostProcessors` | Desligar a segurança nos testes |

**Boilerplate:**

```yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: ${AUTH_ISSUER_URI}
```

```java
@Configuration
@EnableMethodSecurity
class SecurityConfig {

    @Bean
    SecurityFilterChain api(HttpSecurity http) throws Exception {
        return http
                .csrf(csrf -> csrf.disable())   // API stateless com token no header
                .sessionManagement(s -> s.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
                .authorizeHttpRequests(auth -> auth
                        .requestMatchers("/actuator/health", "/publico/**").permitAll()
                        .anyRequest().authenticated())
                .oauth2ResourceServer(oauth -> oauth.jwt(Customizer.withDefaults()))
                .build();
    }
}

// Uso:
@PreAuthorize("hasAuthority('SCOPE_pedido:escrever')")
@PostMapping
ResponseEntity<PedidoResponse> criar(@Valid @RequestBody CriarPedidoRequest request,
                                     @AuthenticationPrincipal Jwt jwt) { ... }
```

Se a aplicação for um sistema interno com login próprio (sem provedor externo), a decisão muda; ver pergunta em aberto na seção 13.

## 8. Integração com outras APIs

**Recomendação:** **HTTP Interface declarativa (`@HttpExchange`) sobre `RestClient`** (Spring 6.1 / Boot 3.2+). A interface é o contrato inteiro da integração em poucas linhas, sem implementação para a IA ler. Uma classe `XxxClient` por sistema externo, com timeout, DTOs próprios e tradução de erro para exceção de domínio.

| Opção | Legibilidade para IA | Observação |
| --- | --- | --- |
| `@HttpExchange` + `RestClient` | Alta; contrato declarativo, sem dependência extra | **Padrão recomendado** |
| `RestClient` direto | Alta; fluxo explícito | Bom quando é 1 ou 2 chamadas com lógica própria |
| OpenFeign (Spring Cloud) | Alta, mas puxa Spring Cloud inteiro | Só se o projeto já usa |
| `WebClient` | Média; API reativa (`Mono`/`Flux`) | Só com stack reativa ou streaming |
| `RestTemplate` | Média; modo manutenção | Evitar em código novo |

**Regras:**

- **Timeouts sempre definidos** (conexão e leitura); sem timeout, uma API lenta derruba a sua.
- URL, credenciais e timeouts em `@ConfigurationProperties` (`record`), nunca no código.
- Resiliência com **Resilience4j** (`@Retry`, `@CircuitBreaker`), configurada no `application.yml`; *retry* só em chamadas idempotentes.
- DTOs da API externa ficam no pacote do client, separados do domínio.
- O client traduz falhas HTTP em `ExternalServiceException`; o Service nunca vê `HttpClientErrorException`.
- Teste com **WireMock** ou `MockRestServiceServer`; nunca chamar API real no teste.

**Boilerplate:**

```java
@ConfigurationProperties("integracao.pagamento")
record PagamentoProperties(String baseUrl, Duration connectTimeout, Duration readTimeout) {}

interface PagamentoApi {
    @PostExchange("/v1/cobrancas")
    CobrancaApiResponse cobrar(@RequestBody CobrancaApiRequest request);
}

@Configuration
@EnableConfigurationProperties(PagamentoProperties.class)
class PagamentoClientConfig {

    @Bean
    PagamentoApi pagamentoApi(PagamentoProperties props) {
        var factory = new JdkClientHttpRequestFactory(
                HttpClient.newBuilder().connectTimeout(props.connectTimeout()).build());
        factory.setReadTimeout(props.readTimeout());

        RestClient restClient = RestClient.builder()
                .baseUrl(props.baseUrl())
                .requestFactory(factory)
                .build();

        return HttpServiceProxyFactory
                .builderFor(RestClientAdapter.create(restClient))
                .build()
                .createClient(PagamentoApi.class);
    }
}

@Component
class PagamentoClient {
    private final PagamentoApi api;
    PagamentoClient(PagamentoApi api) { this.api = api; }

    @CircuitBreaker(name = "pagamento")
    @Retry(name = "pagamento")
    public Cobranca cobrar(Pedido pedido) {
        try {
            return Cobranca.from(api.cobrar(CobrancaApiRequest.from(pedido)));
        } catch (RestClientException ex) {
            throw new ExternalServiceException("PAGAMENTO_INDISPONIVEL", "Falha ao cobrar", ex);
        }
    }
}
```

## 9. Mensageria: tópicos e filas

**Recomendação:** usar o **cliente nativo do broker** (`spring-kafka`, `spring-amqp` ou Spring Cloud AWS SQS) com um *listener* fino que só desserializa e delega ao Service. Evitar **Spring Cloud Stream**: as ligações por nome de bean e convenção são invisíveis no código que a IA lê.

| Broker | Modelo | Anotação | Quando |
| --- | --- | --- | --- |
| Kafka | Tópico com log persistente, vários consumidores por grupo | `@KafkaListener` | Eventos de domínio, alto volume, reprocessamento |
| RabbitMQ | Fila com exchange e roteamento | `@RabbitListener` | Tarefas e comandos com roteamento flexível |
| SQS / SNS | Fila gerenciada (SNS = tópico) | `@SqsListener` | Ambiente AWS, operação sem servidor para manter |

**Regras (valem para qualquer broker):**

- **Envelope padrão** em todo evento: `eventId`, `tipo`, `versao`, `ocorridoEm`, `payload` (record). A IA reaproveita o mesmo formato.
- **Listener fino:** um listener por tópico/fila, sem regra de negócio; chama `service.processar(evento)`.
- **Idempotência obrigatória:** consumidor guarda `eventId` processado (tabela `eventos_processados` com chave única) ou usa operação naturalmente idempotente. Entrega *at-least-once* é a regra.
- **Retry com backoff + DLQ** configurados no `application.yml`/bean de erro; nunca `catch` silencioso dentro do listener.
- **Publicação atômica com o banco:** padrão **Outbox** (grava o evento na mesma transação e um publicador o envia). Evita "salvou no banco mas não publicou".
- Nomes de tópicos/filas em `@ConfigurationProperties` ou constantes, nunca strings repetidas.
- Serialização JSON com records e campo de versão; mudança de contrato só com campos opcionais novos.
- Testes com **Testcontainers** (Kafka/RabbitMQ/LocalStack).

**Boilerplate (Kafka; RabbitMQ e SQS seguem o mesmo desenho):**

```java
public record EventoEnvelope<T>(UUID eventId, String tipo, int versao, Instant ocorridoEm, T payload) {}

public record PedidoCriadoEvent(Long pedidoId, Long clienteId, BigDecimal total) {}

@Component
class PedidoEventosListener {

    private final PedidoEventosService service;

    PedidoEventosListener(PedidoEventosService service) { this.service = service; }

    @KafkaListener(topics = "${mensageria.topicos.pedido-criado}", groupId = "${spring.application.name}")
    void onPedidoCriado(EventoEnvelope<PedidoCriadoEvent> evento) {
        service.processar(evento);   // idempotência e regra ficam no Service
    }
}

@Service
class PedidoEventosService {

    private final EventosProcessadosRepository processados;

    PedidoEventosService(EventosProcessadosRepository processados) { this.processados = processados; }

    @Transactional
    public void processar(EventoEnvelope<PedidoCriadoEvent> evento) {
        if (processados.existsById(evento.eventId())) {
            return;                                   // já tratado
        }
        // ... regra de negócio ...
        processados.save(new EventoProcessado(evento.eventId(), evento.ocorridoEm()));
    }
}
```

```java
// Publicação via Outbox: grava na mesma transação do negócio
@Transactional
public PedidoResponse criar(CriarPedidoRequest request) {
    Pedido pedido = repository.save(Pedido.novo(request));
    outbox.registrar("pedido-criado", new PedidoCriadoEvent(pedido.getId(), pedido.getClienteId(), pedido.getTotal()));
    return PedidoResponse.from(pedido);
}
// Um @Scheduled lê a tabela outbox e publica no broker, marcando como enviado.
```

## 10. Concorrência: mono thread vs multi thread

**Recomendação:** código **imperativo e sequencial** (uma thread por requisição) com **virtual threads** habilitadas (`spring.threads.virtual.enabled=true`, Java 21 + Boot 3.2+). Você ganha escala de I/O sem trocar o estilo de código. Evite WebFlux/reativo, salvo streaming real: `Mono`/`Flux` aumentam o custo de leitura e a chance de erro da IA.

| Cenário | Recomendado | Evitar |
| --- | --- | --- |
| Requisição HTTP normal | Fluxo sequencial no Service | Paralelizar "por precaução" |
| Muitas chamadas de I/O (APIs, banco) | Virtual threads ligadas no `application.yml` | Pool de threads gigante e manual |
| Duas ou três chamadas independentes na mesma requisição | `ExecutorService` de virtual threads + `CompletableFuture` | `parallelStream()` com I/O |
| Tarefa em segundo plano | `@Async` com executor nomeado, retornando `CompletableFuture` | `@Async` sem executor definido; chamar método `@Async` da própria classe (não funciona) |
| Job periódico | `@Scheduled` + **ShedLock** (lock no banco) se houver mais de uma instância | `@Scheduled` sem lock em ambiente com réplicas |
| Processamento em lote | Chunk sequencial; paralelismo só no nível do job | Estado mutável compartilhado entre threads |
| Estado compartilhado | Objetos imutáveis (`record`); `ConcurrentHashMap` só quando inevitável | Singleton Spring com campo mutável |
| Transações | Uma transação por thread; nunca passar entity entre threads | `@Transactional` em método `@Async` esperando herdar a transação |
| `StructuredTaskScope` | Ainda *preview* no Java 21; evitar em produção |  |

**Boilerplate:**

```yaml
spring:
  threads:
    virtual:
      enabled: true
```

```java
@Configuration
@EnableAsync
class AsyncConfig {
    @Bean("tarefasExecutor")
    Executor tarefasExecutor() {
        return Executors.newVirtualThreadPerTaskExecutor();
    }
}

@Service
class RelatorioService {

    @Async("tarefasExecutor")
    public CompletableFuture<Relatorio> gerar(Long id) {
        return CompletableFuture.completedFuture(/* ... */ null);
    }
}

// Chamadas independentes na mesma requisição:
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    var cliente = executor.submit(() -> clienteClient.buscar(id));
    var estoque = executor.submit(() -> estoqueClient.consultar(id));
    return montar(cliente.get(), estoque.get());
}
```

## 11. Testes como especificação para a IA

Os testes são o **ciclo de feedback** da IA: ela altera o código, roda o teste e se corrige sozinha. Eles também documentam o comportamento esperado melhor do que comentários.

| Camada | Ferramenta | Regra |
| --- | --- | --- |
| Regra de negócio (Service) | JUnit 5 + Mockito, sem Spring | Rápido (milissegundos); um cenário por método |
| Controller | `@WebMvcTest` + `MockMvc` | Valida status, JSON e `ProblemDetail` de erro |
| Repository | `@DataJpaTest` + **Testcontainers** (mesmo banco da produção) | Evitar H2 se o SQL/JPQL usa recursos do banco real |
| Integração com API externa | WireMock | Cobrir sucesso, 4xx, 5xx e timeout |
| Mensageria | Testcontainers + Awaitility | Cobrir evento duplicado (idempotência) |
| Arquitetura | **ArchUnit** | Transforma as convenções deste documento em teste que falha no build |

- **Nome do teste descreve o comportamento:** `deveRetornar404QuandoPedidoNaoExiste`.
- Padrão **Given / When / Then** em todo teste, para a IA copiar a forma.
- Um comando único e rápido (`./mvnw test` ou `./gradlew test`) documentado no `AGENTS.md`.
- Dados de teste com *factory methods* (`PedidoFixtures.pedidoPago()`), não JSON gigante inline.

**Boilerplate (regras de arquitetura com ArchUnit):**

```java
@AnalyzeClasses(packages = "com.empresa.app")
class ArquiteturaTest {

    @ArchTest
    static final ArchRule controllers_nao_acessam_repositories =
            noClasses().that().haveSimpleNameEndingWith("Controller")
                    .should().dependOnClassesThat().haveSimpleNameEndingWith("Repository");

    @ArchTest
    static final ArchRule sem_field_injection =
            noFields().should().beAnnotatedWith(Autowired.class);

    @ArchTest
    static final ArchRule entities_nao_saem_do_controller =
            noClasses().that().haveSimpleNameEndingWith("Controller")
                    .should().dependOnClassesThat().areAnnotatedWith(Entity.class);
}
```

## 12. Economia de tokens: regras consolidadas

Os tokens se gastam em três lugares: **ler** código para entender, **escrever** código novo e **corrigir** erros. As regras abaixo atacam os três.

| Regra | Reduz | Como aplicar |
| --- | --- | --- |
| Package-by-feature | Leitura | A IA abre 1 pacote em vez de 4 camadas |
| Records no lugar de classes com getters/Lombok | Leitura e escrita | DTOs de 1 linha |
| Sem interface com implementação única | Leitura | Menos arquivos por feature |
| Sem código gerado versionado ou no caminho de leitura (QueryDSL, MapStruct) | Leitura | Mapeamento manual; geração fica em `target/` e fora do contexto |
| JPQL na `@Query` em vez de Specification/Criteria | Escrita | A consulta cabe em 5 a 10 linhas |
| Herança rasa (no máximo 1 nível, só `DomainException`) | Leitura | Composição em vez de classe base genérica |
| Um padrão por problema | Correção | Menos tentativa e erro; a IA copia o exemplo certo |
| Classes até \~250 linhas, métodos até \~30 | Leitura e diff | Dividir por responsabilidade |
| Comentários só para o *porquê* (nunca repetir o código) | Leitura | Javadoc em decisões de negócio e contratos externos |
| `AGENTS.md` com até \~150 linhas | Leitura | Comandos de build/teste, convenções, proibições; detalhes ficam nas skills |
| Skills carregadas sob demanda | Leitura | Uma skill por tema (persistência, mensageria etc.), não um documento único |
| Testes rápidos e mensagens de erro claras | Correção | Ciclo editar-testar em segundos |
| Pedir à IA o diff, não o arquivo inteiro | Escrita | Instrução padrão nos prompts/skills |
| Evitar anotações empilhadas e meta-programação | Leitura e correção | Se a anotação precisa de explicação, escrever o código explícito |

## 13. Como transformar isto em skills

**Proposta:** uma skill por tema, cada uma com `SKILL.md` curto (quando usar, regras, boilerplate, checklist, anti-padrões) e **um** `AGENTS.md` na raiz com o mínimo comum a todas.

| Skill | Conteúdo | Seções deste documento |
| --- | --- | --- |
| `java-spring-base` | Estrutura de pacotes, nomes, DI, Lombok, regras gerais | 1, 2, 3, 12 |
| `spring-persistencia` | Escada derived → JPQL → Specification, Flyway, projeções, transações | 4 |
| `spring-api-rest` | Controller, DTOs, validação, `ProblemDetail`, handler de exceções | 5, 6 |
| `spring-seguranca` | Resource Server JWT, `@PreAuthorize`, testes de segurança | 7 |
| `spring-integracao-http` | `@HttpExchange`, timeouts, Resilience4j, WireMock | 8 |
| `spring-mensageria` | Envelope, listener fino, idempotência, DLQ, Outbox | 9 |
| `spring-concorrencia` | Virtual threads, `@Async`, `@Scheduled` + ShedLock | 10 |
| `spring-testes-arquitetura` | Testes por camada, Testcontainers, ArchUnit | 11 |

**Estrutura sugerida de cada `SKILL.md`:** descrição de quando acionar (1 frase) → regras *Faça / Não faça* (tabela) → boilerplate mínimo copiável → checklist de revisão → anti-padrões com a correção ao lado.

### Perguntas em aberto

Responda as que importam; as demais ficam com o padrão assumido.

- [ ] **Versões e build:** confirma Java 21 + Spring Boot 3.3+? Maven ou Gradle?
- [ ] **Banco:** qual (PostgreSQL, MySQL, Oracle, SQL Server)? Isso decide `nativeQuery`, tipos e Testcontainers.
- [ ] **Mensageria:** qual broker vai ser usado de fato (Kafka, RabbitMQ, SQS)? Posso aprofundar só o que você usa.
- [ ] **Autenticação:** haverá provedor externo (Keycloak, Cognito, Auth0) ou login próprio com usuário e senha na aplicação?
- [ ] **Arquitetura:** monolito modular ou microsserviços? Isso muda comunicação entre features e contratos.
- [ ] **API:** só REST/JSON ou também GraphQL/gRPC?
- [ ] **IA consumidora:** quais modelos/ferramentas vão ler as skills (modelos grandes, ou modelos menores com contexto curto)? Modelos menores pedem skills ainda mais enxutas e exemplos mais completos.
- [ ] **JPA ou Spring Data JDBC:** topa avaliar JDBC para domínios simples, por ser mais previsível para a IA?
