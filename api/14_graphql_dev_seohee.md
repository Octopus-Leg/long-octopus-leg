# 14.GraphQL API 개발 및 테스트

## 1️⃣ GraphQL용 워크플로우와 도구
- **GraphQL 사고방식**: 데이터를 객체의 **그래프(Graph)** 구조로 구성하여 노출하며, 단일 API 엔드포인트만 사용하는 것이 특징임.
- **구현 방식 분류**:
  - **독립형(Standalone) GraphQL 서비스**: 단일 소스에서 데이터를 가져오는 모놀리식 또는 마이크로서비스 구조임.
  - **연합(Federated) GraphQL 서비스**: 여러 서비스의 데이터 그래프를 **게이트웨이**를 통해 단일 분포 그래프로 통합하여 제공하는 방식임.
- **Netflix DGS(Domain Graph Service) 프레임워크**: 스프링 부트 기반의 오픈소스 프레임워크로, 스키마 우선 개발, **WebFlux** 지원, 코드 생성 플러그인 등 풍부한 기능을 제공함.

## 2️⃣ GraphQL 서버 구현
### gRPC 서버 프로젝트 생성
- **Spring Initializr**를 사용하여 **Gradle-Groovy**, **Spring 3.0.x** 이상 기반의 프로젝트를 생성함.

### GraphQL DGS 의존성 추가
- `graphql-dgs-spring-boot-starter`, `graphql-dgs-extended-scalars` 등 라이브러리를 `build.gradle`에 추가함.
- **DGS Codegen** 플러그인을 설정하여 `src/main/resources/schema`의 스키마 파일로부터 자바 클래스를 자동 생성함.
  - `generateJava` 태스크를 통해 `packageName`과 클라이언트 생성 여부를 지정함.

### GraphQL 스키마 추가
- `schema.graphqls` 파일에 **Query**, **Mutation**, **Subscription** 루트 타입을 정의함.
    ```java
    type Query {
        products(filter: ProductCriteria): [Product]
        product(id: ID!): Product
    }
    
    type Mutation {
        addTag(productId: ID!, tags: [TagInput!]!): Product
        addQuantity(productId: ID!, quantity: Int!): Product
    }
    
    type Subscription {
        quantityChanged: Product
    }
    ```
    - **Query**: `products`(목록 조회)와 `product`(단일 조회) 필드를 포함함.
- **Type 개체 타입 & Input 타입**: `Product`, `Tag` 객체 타입과 검색 조건을 위한 `ProductCriteria` 인풋 타입을 정의함.
    ```java
    type Product {
        id: String
        name: String
        description: String
        imageUrl: String
        price: BigDecimal
        count: Int
        tags: [Tag]
    }
    
    type Tag {
        id: String
        name: String
    }
    
    input ProductCriteria {
        tags: [TagInput] = []
        name: String = ""
        page: Int = 1
        size: Int = 10
    }
    
    input TagInput {
        name: String
    }
    ```

### 커스텀 스칼라 타입 추가
- 기본 타입 외에 `BigDecimal`, `DateTime` 등의 타입을 처리하기 위해 커스텀 스칼라를 등록함.
- **방법 1**: `Coercing` 인터페이스를 직접 구현하고 `@DgsScalar`를 추가함.
- **방법 2(책에서 사용한 방법)**: `graphql-dgs-extended-scalars` 라이브러리의 타입을 `RuntimeWiring`에 연결하여 사용함.

### API 문서화
- **GraphiQL**: 브라우저에서 `http://localhost:8080/graphiql`에 접속하여 스키마 탐색기와 문서 확인이 가능함.

## 3️⃣ GraphQL 쿼리 구현
- **리포지토리 및 서비스**: 데이터를 관리할 `Repository` 인터페이스와 `ConcurrentHashMap` 기반의 구현체를 작성하고, 이를 호출하는 `ProductService`를 구현함.
- **필터링 로직**: `ProductCriteria`를 기반으로 `Predicate`를 생성하여 제품 목록을 필터링하는 기능을 포함함.

## 4️⃣ GraphQL 쿼리용 페처 작성

### Product용 데이터 페처 작성
- **@DgsComponent**: DGS가 스캔하여 사용할 수 있도록 해당 클래스를 스프링 빈으로 등록함.
- **@DgsData**: 특정 필드의 데이터를 가져오는 메소드임을 명시함.
  - `parentType = DgsConstants.QUERY_TYPE`, `field = QUERY.Product`와 같이 설정하여 진입점을 지정함.
- **@InputArgument**: 쿼리 요청 시 전달된 인자(예: `id`)를 메소드 파라미터로 직접 매핑하여 사용함.
- **동작**: 주입된 `ProductService`를 호출하여 실제 데이터를 검색하고 결과를 반환함.

### Product 컬렉션용 데이터 페처 작성
- `@DgsData(parentType = DgsConstants.QUERY_TYPE, field = QUERY.Products)`를 사용하여 제품 목록 조회 기능을 구현함.
- `@InputArgument("filter")`를 통해 클라이언트가 전달한 `ProductCriteria` 객체를 받아 서비스 계층의 `getProducts(criteria)`를 호출함.
- 빌드 후 `http://localhost:8080/graphiql` 엔드포인트에서 쿼리를 실행하여 필터링된 제품 목록이 정상적으로 반환되는지 확인함.

### 데이터 페처 메소드를 사용한 필드 해석기 작성
- 특정 객체의 하위 필드 `Product`의 `tags` 같이 별도로 로드해야 할 때 사용함.
- `@DgsData(parentType = PRODUCT.TYPE_NAME, field = PRODUCT.Tags)`와 같이 부모 타입을 `Query`가 아닌 특정 객체 타입으로 지정함.
- 이 방식은 각 제품마다 태그를 조회하는 쿼리가 개별적으로 실행되어 성능 저하를 유발하는 **N+1 문제**의 원인이 됨.

### N+1 문제를 해결하기 위한 데이터 로더 작성
- **데이터 로더**: 여러 개의 개별 요청을 하나의 배치 요청으로 묶어 처리하고 결과를 캐싱하여 효율을 높임.
- **DgsDataFetchingEnvironment**: `env.getDataLoader(TagsDataloaderWithContext.class)`를 통해 등록된 데이터 로더를 가져옴.
- **비동기 처리**: `CompletableFuture` 또는 `CompletionStage`를 반환하여 논블로킹 방식으로 데이터를 로드함.
  - `tagsDataLoader.load(product.getId())`를 호출하면 개별 ID가 큐에 쌓이고, 최종적으로 배치 함수가 실행됨.
- **MappedBatchLoaderWithContext**: 컨텍스트 정보를 포함할 수 있는 인터페이스를 구현하여 `load(Set<K> keys, BatchLoaderEnvironment env)` 메소드 내에서 `TagService`를 통해 한꺼번에 데이터를 조회함.

## 5️⃣ GraphQL 뮤테이션 구현
- **뮤테이션**: 데이터의 생성, 수정, 삭제와 같이 시스템의 상태를 변경하는 작업임. ➡️`addTag`, `addQuantity` 뮤테이션
  - `addTag`: 특정 제품(`productId`)에 새로운 태그 목록을 추가하고 업데이트된 `Product` 객체를 반환함.
  - `addQuantity`: 제품의 재고 수량을 변경하는 로직을 수행함.
- **@DgsMutation**: **DGS** 프레임워크에서 뮤테이션 핸들러를 정의하기 위해 사용하는 어노테이션임.
- `@InputArgument`를 통해 스키마에 정의된 인자를 자바 객체나 기본 타입으로 매핑하여 처리함.

## 6️⃣ GraphQL 서브스크립션 구현 및 테스트
- **서브스크립션**: 서버 측에서 데이터 변경 이벤트가 발생했을 때 클라이언트에 실시간으로 데이터를 푸시하는 기능임.
- **@DgsSubscription**: 서브스크립션 스트림을 생성하는 메소드에 사용함.
- **리액티브 스트림 반환**: 서브스크립션 메소드는 지속적인 데이터 흐름을 위해 `Publisher` **Project Reactor**의 `Flux` 타입을 반환해야 함.
- **구현 방식**:
  - `ProductService`에서 `Sinks.Many`를 활용하여 이벤트를 발행하고, 데이터 페처에서 이를 `asFlux()`로 변환하여 클라이언트에 전달함.
  - 새로운 제품이 추가되거나 재고가 변경될 때마다 등록된 구독자에게 알림을 보낼 수 있음.

### GraphQL용 WebSocket 서브-프로토콜 이해
- **서브-프로토콜 사양**: **DGS** 프레임워크는 `graphql-transport-ws` 사양을 사용하며, 모든 메시지는 **JSON** 형식의 문자열로 교환됨.
- **메시지 유형(MessageType)**:
  - `연결 초기화 connection_init`: 클라이언트가 통신 시작을 위해 보내는 초기화 메시지임.
  - `연결 승인 connection_ack`: 서버가 연결을 승인했음을 알리는 응답임.
  - `구독 subscribe`: 클라이언트가 특정 쿼리나 이벤트를 구독하기 위해 보내는 요청임. 고유한 `id`를 포함해야 함.
  - `다음 next`: 구독 중인 이벤트가 발생했을 때 서버가 클라이언트에 데이터를 전달하는 메시지임.
  - `완료 complete`: 클라이언트나 서버가 구독 작업을 종료할 때 사용하는 메시지임.
  - `오류 error`: 서버에서 오퍼레이션 실행 에러가 발생하면 보내는 에러 메시지임.
  - `ping` & `pong`: 연결 유지 상태를 확인하기 위한 양방향 메시지임.

### Insomnia 웹소켓을 이용한 GraphQL 서브스크립션 테스트
- **연결 설정**: **Insomnia**에서 `ws://localhost:8080/subscriptions` 주소로 웹소켓 요청을 생성하고, 필요한 헤더(`Sec-WebSocket-Protocol` 등)를 설정함.
- **테스트 워크플로우**:
  1. `connection_init` 메시지를 전송하여 서버와 연결을 확인함.
  2. `subscribe` 메시지에 서브스크립션 쿼리를 담아 전송하여 구독을 시작함.
  3. 별도의 창에서 `addQuantity` 뮤테이션을 실행하여 데이터를 변경함.
  4. 웹소켓 창에서 서버로부터 실시간으로 전달되는 `next` 타입의 **JSON** 데이터를 확인하여 정상 동작을 검증함.

## 7️⃣ GraphQL API 인스트루먼테이션
- **인스트루먼테이션**: **GraphQL** 실행 과정(쿼리 분석, 실행, 결과 반환 등)에 개입하여 로깅, 메트릭 수집, 트레이싱 등을 수행하는 기능임.
- **SimpleInstrumentation**: `graphql.execution.instrumentation.Instrumentation` 인터페이스를 직접 구현하는 대신, 이를 상속받아 필요한 메소드만 오버라이드하여 간편하게 구현 가능함.
- **목적**: 실행 시간이 오래 걸리는 필드를 식별하거나 응답 헤더를 조작하는 등 API 성능 미세 조정에 활용함.

### 커스텀 헤더 추가
- **SimpleInstrumentation 상속**: `instrumentExecutionResult` 메소드를 오버라이드하여 쿼리 실행 결과에 개입함.
- **헤더 조작**: `HttpHeaders` 객체를 생성하고 `responseHeaders.add("myHeader", "hello")`와 같이 커스텀 헤더를 추가함.
- **결과 반환**: 변경된 헤더 정보를 포함하여 `ExecutionResult`를 빌드하고 반환함으로써 클라이언트 응답에 특정 정보를 실어 보낼 수 있음.

### Micrometer와 통합
- **의존성 추가**: `graphql-dgs-spring-boot-micrometer` 라이브러리를 사용하여 **Spring Actuator** 메트릭과 통합함.
- **추적 활성화**: `InstrumentationConfig` 설정 클래스에서 `TracingInstrumentation` 빈을 등록하여 **GraphQL Tracing** 정보를 활성화함.
- **메트릭 확인**:
  - `application.properties`에서 `management.endpoints.web.exposure.include=health,metrics` 설정을 통해 엔드포인트를 노출함.
  - `http://localhost:8080/actuator/metrics/gql.query` 등의 경로에서 쿼리 실행 횟수, 총 소요 시간 등을 모니터링할 수 있음.
- **제공되는 메트릭 타입**:
  - `gql.query`: 쿼리 및 뮤테이션 전체 실행 시간 수집함.
  - `gql.resolver`: 각 데이터 페처 Resolver 호출 시 소요되는 시간 측정함.
  - `gql.error`: 실행 중 발생한 에러 정보를 수집함.
  - `gql.dataLoader`: 배치 쿼리에서 데이터 로더 호출 성능을 기록함.
- **확장 필드**: 응답 데이터의 `extensions` 필드 내 `tracing` 항목을 통해 파싱, 유효성 검사, 각 필드별 해석기 실행 시간을 나노초 단위로 정밀하게 확인할 수 있음.

## 8️⃣ 테스트 자동화
- **DGS 테스트 프레임워크**: **DGS**는 **GraphQL** API 테스트를 용이하게 하는 `DgsQueryExecutor` 등의 유틸리티를 제공함.
- **테스트 설정**: `@SpringBootTest`를 사용하며, `classes` 속성에 테스트에 필요한 최소한의 구성 요소(`DgsAutoConfiguration`, 데이터 페처, 스칼라 등)만 지정하여 컨텍스트를 가볍게 유지함.
- **Mock 활용**: 외부 서비스 호출을 배제하기 위해 `@MockBean`을 사용하여 `ProductService`와 `TagService`를 모킹함.

### GraphQL 쿼리 테스트
- **실행 및 검증**: `dgsQueryExecutor.executeAndExtractJsonPath()`를 사용하여 쿼리를 실행하고 결과값의 특정 경로를 **JsonPath**로 추출함.
- **예외 테스트**: 존재하지 않는 ID 조회 시 발생하는 예외 상황을 스텁으로 정의하고, `ExecutionResult` 내의 에러 메시지가 예상과 일치하는지 `assertThat`으로 검증함.
- **유형 안전성**: **DGS** 플러그인에 의해 자동 생성된 `GraphQLQueryRequest`와 `ProjectionRoot`를 사용하면 문자열 쿼리 작성 시 발생할 수 있는 오타를 방지하고 유형 안전하게 테스트 코드를 작성할 수 있음.

### GraphQL 뮤테이션 테스트
- **상태 변경 검증**: 쿼리 테스트와 동일한 방식으로 진행되나, 데이터 변경 후 반환된 객체의 필드값이나 상태가 올바르게 업데이트되었는지 확인함.
- **코드 생성 활용**: 자동 생성된 뮤테이션 클래스`AddTagGraphQLQuery`를 활용하여 요청을 직렬화하고 실행함.

### 자동화된 테스트 코드를 이용한 GraphQL 서브스크립션 테스트
- **Publisher 구독**: 서브스크립션 실행 결과인 `Publisher<ExecutionResult>`를 확보한 뒤, `Subscriber` 객체를 직접 구현하여 스트림을 구독함.
- **실시간 검증**:
  - `onNext()` 메소드 내에서 수신된 데이터를 리스트에 저장하거나 즉시 검증함.
  - 테스트 시나리오 내에서 뮤테이션을 호출하여 이벤트를 트리거하고, 구독자가 해당 변경 사항을 정상적으로 수신했는지 확인함.
  - `CopyOnWriteArrayList`와 같은 스레드 안전한 컬렉션을 사용하여 비동기적으로 들어오는 데이터를 관리함.