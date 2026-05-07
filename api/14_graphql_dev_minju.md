# 14. GraphQL API 개발 및 테스트

## 14.1 GraphQL 워크플로우와 도구

- GraphQL은 데이터 그래프라는 사고방식을 바탕으로 데이터가 객체의 그래프로 구성되는 API를 이용해서 노출됨.
- 데이터를 포함한 객체들은 서로 관계를 맺으며 연결되고, GraphQL은 **단일 API 엔드포인트**만 노출함.
- **OneGraph 원칙**에 따라 데이터 그래프는 단일 소스 또는 여러 소스(데이터베이스, 레거시 시스템, REST/gRPC/SOAP 서비스 등)로부터 데이터를 가져올 수 있음.

**GraphQL 서버 구현 방식**

| 방식 | 설명 |
|---|---|
| **독립형(Standalone)** | 단일 데이터 그래프 포함. 모놀리식 앱 또는 마이크로서비스 아키텍처 기반 |
| **연합(Federated)** | 게이트웨이를 통해 노출되는 단일 분포 그래프 포함. 클라이언트는 항상 동일한 단일 엔드포인트에 쿼리함 |

**넷플릭스 DGS 프레임워크**

- 넷플릭스가 2021년 2월 프로덕션에서 오픈 소스로 공개한 프레임워크
- `graphql-java` 기반의 GraphQL용 스프링 부트 스타터 프로젝트
- DGS가 제공하는 주요 기능
   - 스프링 부트 스타터 및 스프링 시큐리티와의 통합 제공
   - 완전한 웹플럭스(WebFlux) 지원
   - GraphQL 스키마로부터 코드를 생성하기 위한 그래들 플러그인
   - 인터페이스, 유니온 타입 지원 및 사용자 지정 스칼라 타입 제공
   - 웹소켓 및 서버 전송 이벤트를 이용한 GraphQL 서브스크립션 지원
   - GraphQL 연합과 쉽게 통합 가능한 연합 서비스
   - 핫 리로딩 스키마가 있는 동적 스키마
   - 오퍼레이션 캐싱
   - 파일 업로드
   - GraphQL 자바 클라이언트
   - GraphQL 테스트 프레임워크

## 14.2 GraphQL 서버 구현

### 14.2.1 GraphQL DGS 의존성 추가

```groovy
plugins {
    id 'org.springframework.boot' version '3.0.6'
    id 'io.spring.dependency-management' version '1.1.0'
    id 'java'
    id 'com.netflix.dgs.codegen' version '5.7.1' // DGS Codegen 플러그인
}

def dgsVersion = '6.0.5'
dependencies {
    implementation platform("com.netflix.graphql.dgs:graphql-dgs-platform-dependencies:${dgsVersion}") // BOM
    implementation 'com.netflix.graphql.dgs:graphql-dgs-spring-boot-starter'     // DGS 스프링 부트 스타터
    implementation 'com.netflix.graphql.dgs:graphql-dgs-extended-scalars'        // 커스텀 스칼라 타입
    implementation 'com.netflix.graphql.dgs:graphql-dgs-spring-boot-micrometer'  // Micrometer 메트릭
    runtimeOnly 'com.netflix.graphql.dgs:graphql-dgs-subscriptions-websockets-autoconfigure' // WebSocket 서브스크립션
    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation 'org.springframework.boot:spring-boot-starter-actuator'
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
    implementation 'net.datafaker:datafaker:1.9.0' // 시드 데이터 생성용
}
```

**DGS Codegen 플러그인 설정**

```groovy
generateJava {
    generateClient = true                               // 클라이언트 생성 여부
    packageName = "com.packt.modern.api.generated"      // 생성된 클래스의 패키지명
}
```

- DGS Codegen 플러그인은 기본적으로 `src/main/resources/schema` 디렉터리에서 GraphQL 스키마 파일을 탐색함.

### 14.2.2 GraphQL 스키마 추가

- DGS는 코드 우선 방식과 설계 우선 방식을 모두 지원함. 책에서는 **설계 우선 방식**을 사용함.
- `src/main/resources/schema/schema.graphqls` 파일에 스키마를 정의함.

```graphql
# 루트 타입 정의
type Query {
    products(filter: ProductCriteria): [Product]! # 필터 기반 제품 목록 조회
    product(id: ID!): Product                     # 단일 제품 조회
}
type Mutation {
    addTag(productId: ID!, tags: [TagInput!]!): Product   # 태그 추가
    addQuantity(productId: ID!, quantity: Int!): Product  # 수량 증가 (서브스크립션 이벤트 트리거)
}
type Subscription {
    quantityChanged: Product # 수량 변경 이벤트 구독
}

# 객체 타입
type Product {
    id: String
    name: String
    description: String
    imageUrl: String
    price: BigDecimal # 커스텀 스칼라 타입
    count: Int
    tags: [Tag]
}
type Tag {
    id: String
    name: String
}

# 인풋 타입
input ProductCriteria {
    tags: [TagInput] = []
    name: String = ""
    page: Int = 1
    size: Int = 10
}
input TagInput {
    name: String
}

# BigDecimal은 GraphQL 기본 제공 타입이 아니므로 커스텀 스칼라로 선언해야 함
scalar BigDecimal
```

### 14.2.3 커스텀 스칼라 타입 추가

- GraphQL이 기본 제공하지 않는 타입(예: `BigDecimal`)은 **커스텀 스칼라**로 직접 등록해야 함.
- 커스텀 스칼라를 추가하는 방법
  1. `Coercing` 인터페이스 구현: `serialize()`, `parseValue()`, `parseLiteral()` 메소드를 재정의해야 함.
  2. `graphql-dgs-extended-scalars` 라이브러리 사용

```java
// graphql-dgs-extended-scalars 라이브러리 방식
@DgsComponent // DGS 컴포넌트이자 일반 스프링 컴포넌트로 등록됨
public class BigDecimalScalar {

    @DgsRuntimeWiring // 런타임 연결(wiring) 시점에 스칼라를 등록함
    public RuntimeWiring.Builder addScalar(RuntimeWiring.Builder builder) {
        return builder.scalar(GraphQLBigDecimal);
    }
}
```

## 14.3 GraphQL 쿼리 구현

### 14.3.1 GraphQL 쿼리용 데이터 페처 작성

- **데이터 페처(fetcher)**: GraphQL 요청을 처리하는 중요한 DGS 컴포넌트
- `@DgsComponent`로 표시하며 `@DgsData` 애노테이션으로 처리할 필드를 지정함.

```java
// ProductDatafetcher.java
@DgsComponent
public class ProductDatafetcher {
    private final ProductService productService;

    public ProductDatafetcher(ProductService productService) {
        this.productService = productService;
    }

    @DgsData(parentType = DgsConstants.QUERY_TYPE, field = QUERY.Product)
    public Product getProduct(@InputArgument("id") String id) {
        if (Strings.isBlank(id)) {
            new RuntimeException("유효하지 않은 제품 아이디입니다.");
        }
        return productService.getProduct(id);
    }
}
```

**DGS 프레임워크 애노테이션**

| 애노테이션 | 설명 |
|---|---|
| `@DgsData` | 해당 메소드가 데이터 페처임을 표시함. `parentType`은 타입, `field`는 타입의 필드 |
| `@InputArgument` | GraphQL 요청에 의해 전달되는 아규먼트를 캡처함 |

### 14.3.2 N+1 문제 해결을 위한 데이터 로더 작성

- Product 목록 조회 시 각 제품의 Tag를 개별 조회하면 **N+1 문제**가 발생함.
- DGS에서는 `DataFetchingEnvironment`와 `DataLoader`를 사용하여 N+1 문제를 해결함.
- **DataLoader**는 단일 쿼리로 모든 ID를 일괄 처리하여 N번의 쿼리를 1번으로 줄임.

**컨텍스트가 없는 데이터 로더 클래스**

```java
// ProductsDatafetcher.java
@DgsData(
    parentType = PRODUCT.TYPE_NAME,
    field = PRODUCT.Tags // Query 루트 타입이 아닌 Product 객체 타입으로 설정됨
)
public CompletableFuture<List<Tags>> tags(DgsDataFetchingEnvironment env) {
    // DataLoader를 클래스명으로 로딩
    DataLoader<String, List<Tags>> tagsDataLoader =
            env.getDataLoader(TagsDataloaderWithContext.class);
    Product product = env.getSource(); // @DgsData의 parentType인 Product 객체를 소스로 가져옴
    return tagsDataLoader.load(product.getId()); // 제품 ID를 DataLoader에 전달 → 배치 처리
}
```

**컨텍스트가 있는 데이터 로더 클래스**

```java
// TagsDataloaderWithContext.java
@DgsDataLoader(name = "tagsWithContext")
public class TagsDataloaderWithContext implements
        MappedBatchLoaderWithContext<String, List<Tag>> {

    private final TagService tagService;

    public TagsDataloaderWithContext(TagService tagService) {
        this.tagService = tagService;
    }

    @Override
    public CompletionStage<Map<String, List<Tag>>> load(Set<String> keys,
            BatchLoaderEnvironment environment) {
        // 여러 제품 ID를 한 번에 일괄 조회 → N+1 해결
        return CompletableFuture.supplyAsync(
                () -> tagService.getTags(new ArrayList<>(keys)));
    }
}
```

## 14.4 GraphQL 뮤테이션 구현

- 뮤테이션은 `@DgsMutation` 애노테이션을 사용하여 구현함.
- `@DgsMutation`: Mutation을 기본값으로 하는 `parentType` 속성을 가지는 `@DgsData`의 타입 중 하나

```java
// ProductDatafetcher.java
@DgsMutation(field = MUTATION.AddTag)
public Product addTags(
        @InputArgument("productId") String productId,
        // 인풋 타입이 스칼라가 아닌 경우 collectionType 속성을 반드시 명시해야 에러를 방지할 수 있음
        @InputArgument(value = "tags", collectionType = TagInput.class) List<TagInput> tags) {
    return tagService.addTags(productId, tags);
}

@DgsMutation(field = MUTATION.AddQuantity)
public Product addQuantity(
        @InputArgument("productId") String productId,
        @InputArgument("quantity") int qty) {
    return productService.addQuantity(productId, qty);
}
```

## 14.5 GraphQL 서브스크립션 구현 및 테스트

- **서브스크립션**: 특정 이벤트가 발생할 때 객체를 구독자(클라이언트)에게 보내는 GraphQL 루트 타입
- 책에서는 리액티브 스트림(`Flux`)과 WebSocket을 사용하여 구현함.

**서브스크립션 데이터 페처 추가**

```java
// ProductDatafetcher.java
@DgsSubscription(field = SUBSCRIPTION.QuantityChanged)
public Publisher<Product> quantityChanged(
        @InputArgument("productId") String productId) {
    // Subscription 메소드는 여러 구독자에게 무제한의 객체를 보낼 수 있는 Publisher를 리턴함
    return productService.getProductPublisher();
}
```

**리포지토리에서 Publisher 구성**

```java
// FluxSink와 ConnectableFlux 선언
private FluxSink<Product> productsStream;
private ConnectableFlux<Product> productPublisher;

// 생성자에서 초기화
Flux<Product> publisher = Flux.create(emitter -> {
    productsStream = emitter;
});
productPublisher = publisher.publish();
productPublisher.connect();

// addQuantity() 메소드에서 수량 변경 시 이벤트 발행
product.setCount(product.getCount() + qty);
productEntities.put(product.getId(), product);
productsStream.next(product);
return product;
```

**CORS 및 WebSocket 설정**

```properties
management.endpoints.web.exposure.include=health,metrics
graphql.servlet.actuator-metrics=true
graphql.servlet.tracing-enabled=false
graphql.servlet.corsEnabled=true
```

### 14.5.1 GraphQL용 WebSocket 서브-프로토콜 이해

- WebSocket 기반 GraphQL 서브스크립션은 `graphql-transport-ws` 서브-프로토콜을 사용함.
- 서버와 클라이언트 모두 아래 메시지 유형을 따라야 함.

| 메시지 유형 | 방향 | 설명 |
|---|---|---|
| `CONNECTION_INIT` | 클라이언트 → 서버 | 통신 시작 요청. `payload`는 선택 필드 |
| `CONNECTION_ACK` | 서버 → 클라이언트 | 연결 승인. 구독 요청 가능 상태를 의미함 |
| `SUBSCRIBE` | 클라이언트 → 서버 | 구독 요청. 고유 `id`와 `query` 필드 포함 필수 |
| `NEXT` | 서버 → 클라이언트 | 이벤트 발생 시 데이터 전송. `payload`에 실행 결과 포함 |
| `COMPLETE` | 서버 ↔ 클라이언트 | 클라이언트 또는 서버 모두 전송 가능한 종료 메시지 |
| `ERROR` | 서버 → 클라이언트 | 오퍼레이션 실행 에러 발생 시 전송 |
| `PING / PONG` | 서버 ↔ 클라이언트 | 네트워크 문제 및 대기 시간 감지용 |

## 14.6 GraphQL API 인스트루멘테이션

- GraphQL 자바 라이브러리는 메트릭, 트레이싱, 로깅을 지원하는 **인스트루멘테이션**을 제공함.
- `SimpleInstrumentation` 클래스를 상속해서 메소드를 오버라이드하여 커스텀 구현을 할 수 있음.

### 14.6.1 커스텀 헤더 추가

```java
// DemoInstrumentation.java
@Component
public class DemoInstrumentation extends SimpleInstrumentation {

    @NotNull
    @Override
    public CompletableFuture<ExecutionResult> instrumentExecutionResult(
            ExecutionResult exeResult,
            InstrumentationExecutionParameters params,
            InstrumentationState state) {
        HttpHeaders responseHeaders = new HttpHeaders();
        responseHeaders.add("myHeader", "hello"); // 커스텀 응답 헤더 추가
        return super.instrumentExecutionResult(DgsExecutionResult
                .builder().executionResult(execResult)
                .headers(responseHeaders).build(), params, state);
    }
}
```

- `TracingInstrumentation` 빈을 등록하면 응답의 `extensions` 필드에 트레이싱 메트릭이 포함됨.
- GraphQL 구현과 벤치마킹을 미세 조정하기 위해 개발 환경에서만 활성화하고 프로덕션 환경에서는 비활성화하도록 함.

### 14.6.2 Micrometer와 통합

- `graphql-dgs-spring-boot-micrometer` 의존성을 추가하면 GraphQL 메트릭을 즉시 사용할 수 있음.
- `http://localhost:8080/actuator/metrics/gql.query` 엔드포인트로 메트릭 정보를 확인할 수 있음.

| 메트릭 | 설명 |
|---|---|
| `gql.query` | GraphQL 쿼리 또는 뮤테이션에 소요되는 시간을 수집함 |
| `gql.resolver` | 각 데이터 페처 호출 시 소요되는 시간을 수집함 |
| `gql.error` | GraphQL 요청 실행 중에 발생한 에러를 수집함. 실행 중 에러가 발생한 경우에만 사용 가능함 |
| `gql.dataloader` | 배치 쿼리에서 데이터 로더 호출 시 소요한 시간을 수집함 |

## 14.7 테스트 자동화

- DGS 프레임워크는 GraphQL API 테스트 자동화를 용이하게 하는 클래스와 유틸리티를 제공함.

### 14.7.1 테스트 환경 구성

```java
// ProductDatafetcherTest.java
@SpringBootTest(classes = {
    DgsAutoConfiguration.class,
    ProductDatafetcher.class,
    BigDecimalScalar.class  // 필요한 클래스만 제한하여 스프링 컨텍스트를 최소화함
})
public class ProductDatafetcherTest {
    private final InMemRepository repo = new InMemRepository();

    @Autowired
    private DgsQueryExecutor dgsQueryExecutor; // GraphQL 쿼리 실행 유틸리티

    @MockBean
    private ProductService productService;

    @MockBean
    private TagService tagService;

    @BeforeEach
    public void beforeEach() {
        // Mockito를 사용하여 서비스 메소드의 스텁을 정의함
        given(productService.getProduct("any")).willReturn(product);
        given(tagService.addTags("any", List.of(TagInput.newBuilder().name("addTags").build())))
                .willAnswer(invocation -> product);
    }
}
```

### 14.7.2 GraphQL 쿼리 테스트

```java
@Test
@DisplayName("'product' 쿼리가 리턴한 JSON 검증")
public void product() {
    // executeAndExtractJsonPath — 쿼리를 실행하고 JSON 경로로 값을 추출함
    String name = dgsQueryExecutor.executeAndExtractJsonPath(
            "{product(id: \"any\"){ name }}", "data.product.name");
    assertThat(name).contains("mock title");
}

@Test
@DisplayName("GraphQLQueryRequest를 사용한 JSON 유효성 검증")
void productsWithQueryApi() {
    GraphQLQueryRequest gqlRequest = new GraphQLQueryRequest(
            ProductGraphQLQuery.newRequest().id("any").build(),
            ProductProjectionRoot().id().name());
    String name = dgsQueryExecutor.executeAndExtractJsonPath(
            gqlRequest.serialize(), "data.product.name");
    assertThat(name).contains("mock title");
}

@Test
@DisplayName("'product' 쿼리가 리턴한 태그 검증")
void productsWithTags() {
    GraphQLQueryRequest gqlRequest = new GraphQLQueryRequest(
            ProductGraphQLQuery.newRequest().id("any").build(),
            new ProductProjectionRoot().id().name().tags().id().name());
    Product product = dgsQueryExecutor
            .executeAndExtractJsonPathAsObject(gqlRequest.serialize(),
                    "data.product", new TypeRef<>() {});
    assertThat(product.getId()).isEqualTo("any");
    assertThat(product.getTags().size()).isEqualTo(2);
    // 하위 필드 조회 시 executeAndExtractJsonPathAsObject의 세 번째 아규먼트(TypeRef) 사용 필수
}
```

### 14.7.3 GraphQL 뮤테이션 테스트

```java
@Test
@DisplayName("'addTags' 뮤테이션 검증")
void addTagsMutation() {
    GraphQLQueryRequest gqlRequest = new GraphQLQueryRequest(
            AddTagGraphQLQuery.newRequest().productId("any")
                    .tags(List.of(TagInput.newBuilder().name("addTags").build())).build(),
            new AddTagProjectionRoot().name().count());
    ExecutionResult exeResult = dgsQueryExecutor.execute(gqlRequest.serialize());
    assertThat(exeResult.getErrors()).isEmpty();
    verify(tagService).addTags("any", List.of(TagInput.newBuilder().name("addTags").build()));
}
```

### 14.7.4 자동화된 테스트 코드를 이용한 GraphQL 서브스크립션 테스트

```java
@Test
@DisplayName("'quantityChanged' 서브스크립션 검증")
void reviewSubscription() {
    given(productService.getProductPublisher()).willReturn(repo.getProductPublisher());
    ExecutionResult exeResult = dgsQueryExecutor.execute(
            "subscription {quantityChanged{id name price count}}");
    Publisher<ExecutionResult> pub = exeResult.getData();
    List<Product> products = new CopyOnWriteArrayList<>();

    pub.subscribe(new Subscriber<>() {
        @Override
        public void onSubscribe(Subscription s) { s.request(2); }

        @Override
        public void onNext(ExecutionResult result) {
            // 서버로부터 수신한 Product 객체를 리스트에 저장
            Map<String, Object> data = result.getData();
            products.add(new ObjectMapper().convertValue(
                    data.get(SUBSCRIPTION.QuantityChanged), Product.class));
        }

        @Override public void onError(Throwable t) {}
        @Override public void onComplete() {}
    });

    addQuantityMutation(); // 서브스크립션 게시자를 트리거함
    Integer count = products.get(0).getCount();
    addQuantityMutation();
    assertThat(products.get(0).getId()).isEqualTo(products.get(1).getId());
    assertThat(products.get(1).getCount()).isEqualTo(count + TEN); // 수량 증가 검증
}
```
