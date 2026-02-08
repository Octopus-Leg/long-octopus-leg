# 5. 비동기 API 설계

- 일반적인 자바 코드는 스레드 풀을 사용하여 비동기성을 구현하지만, 블로킹 호출로 인해 JVM 리소스 사용에 제약이 생겨 수평적 확장이 필요해짐.

- 이를 극복하기 위해 **리액티브 스트림(Reactive Streams)** 을 사용함.

## 5.1 리액티브 스트림 이해하기

- **발행자-구독자 모델(Publisher-Subscriber Model)**: 데이터 소스인 발행자가 구독자에게 데이터를 푸시(Push)하는 방식
- **백프레셔(Back-pressure)**: 구독자가 자신이 안전하게 처리할 수 있는 데이터 양을 발행자에게 알려 흐름을 제어하는 핵심 기능
- 리액티브 스트림은 푸시 스타일을 사용하는 반면, 기존 자바 스트림은 소스에서 항목을 가져오는 풀 모델로 동작함.

### 5.1.1 리액티브 스트림 기본 타입

리액티브 스트림 명세에는 네 가지 기본 타입이 정의되어 있음.

```java
// 발행자(Publisher): 구독자의 요청에 따라 요소를 푸시함. 게으른(lazy) 특성이 있어 구독자가 있는 경우에만 요소를 푸시함.
public interface Publisher<T> {
    public void subscribe(Subscriber<? super T> s);
}

// 구독자(Subscriber): 발행자가 푸시한 데이터를 소비함.
public interface Subscriber<T> {
    public void onSubscribe(Subscription s); // 구독 시작 시 호출
    public void onNext(T t);                 // 다음 요소 처리
    public void onError(Throwable t);        // 에러 발생 시 호출
    public void onComplete();                // 모든 요소 푸시 완료 시 호출
}

// 구독(Subscription): 발행자와 구독자 사이의 중재자(매개체)
public interface Subscription {
    public void request(long n); // n개의 요소 요청
    public void cancel();        // 데이터 전송 중지 및 리소스 정리
}

// 프로세서(Processor): 발행자와 구독자 사이에서 다리 역할을 하며, 두 인터페이스를 모두 상속함.
public interface Processor<T, R> extends Subscriber<T>, Publisher<R> {
}
```

## 5.2 스프링 웹플럭스 살펴보기

- **스프링 웹플럭스(Spring WebFlux)** 는 완전히 논-블로킹이며 백프레셔 기능을 제공하는 리액티브 웹 프레임워크임.

- 적은 수의 스레드로 높은 동시성을 제공하며, 기본 핵심 의존성으로 **프로젝트 리액터(Project Reactor)** 를 사용함.

**발행자 타입**

- **Mono**: 0 또는 1개의 요소를 반환하는 발행자
- **Flux**: 0에서 N개의 요소를 반환하는 발행자

### 5.2.1 리포지토리 대체 예시

```java
// 기존 방식
public Product findById(UUID id);
public List<Product> getAll();

// 리액티브 방식 대체
public Mono<Product> findById(UUID id);
public Flux<Product> getAll();
```

## 5.3 DispatcherHandler 이해하기

- `DispatcherHandler`는 스프링 MVC의 `DispatcherServlet`에 대응하는 스프링 웹플럭스의 프런트 컨트롤러임.

- 웹 서버가 받은 요청을 리액티브 파이프라인으로 빌드하며, `HandlerMapping`, `HandlerAdapter`, `HandlerResultHandler`를 사용하여 처리함.

**요청 처리 순서**

1. `DispatcherHandler`가 웹 요청을 수신함.
2. `HandlerMapping`을 통해 요청에 일치하는 핸들러를 찾음.
3. `HandlerAdapter`로 요청을 처리하고 `HandlerResult`를 노출함.
4. `HandlerResultHandler`가 결과를 기반으로 응답을 작성하거나 뷰를 렌더링함.
5. 요청 처리를 완료함.

### 5.3.1 애노테이션 기반 컨트롤러

- 스프링 MVC와 동일한 애노테이션(`@RestController`, `@RequestMapping`, `@RequestBody`, `@PathVariable` 등)을 그대로 사용하며, 반환 타입으로 `Mono`나 `Flux`를 사용함.

```java
@RestController
public class OrderController {
    @RequestMapping(value = "/api/v1/orders", method = RequestMethod.POST)
    public Mono<ResponseEntity<Order>> addOrder(@RequestBody NewOrder newOrder) {
        // ...
    }
}
```

### 5.3.2 함수형 엔드포인트

- 명령형 스타일의 컨트롤러 대신 함수형 프로그래밍 스타일로 엔드포인트를 정의할 수 있음.

- 라우팅 로직과 핸들러 로직을 분리하는 방식임.

- **RouterFunction**: 요청 경로와 핸들러를 매핑함.
- **HandlerFunction**: 실제 요청을 처리하며 `ServerRequest`를 받고 `Mono<ServerResponse>`를 반환함.

```java
// RouterFunction 예시
RouterFunction<ServerResponse> route = route()
    .GET("/v1/api/orders/{id}", accept(APPLICATION_JSON), handler::getOrderById)
    .POST("/v1/api/orders", handler::addOrder)
    .build();

// HandlerFunction 예시
public Mono<ServerResponse> getOrderById(ServerRequest req) {
    String orderId = req.pathVariable("id");
    return repository.getOrderById(UUID.fromString(orderId))
        .flatMap(order -> ok().contentType(APPLICATION_JSON).bodyValue(toModel(order)))
        .switchIfEmpty(ServerResponse.notFound().build());
}
```

**주요 메서드**

- `request.bodyToMono(Class)`: 요청 본문을 Mono로 변환함.
- `request.pathVariable("id")`: 경로 변수를 추출함.
- `flatMap()`: 리포지토리에서 수신한 데이터를 처리함.
- `switchIfEmpty()`: 데이터가 없을 경우(Empty Mono) 404 NOT FOUND 등을 처리함.

## 5.4 리액티브 API 구현

### 5.4.1 리액티브 API용 OpenAPI Codegen 변경

- OpenAPI Codegen을 사용하여 리액티브 지원 코드를 생성할 수 있음.

- `config.json`에서 `"library": "spring-boot"`와 `"reactive": true` 플래그를 설정하면 생성된 인터페이스의 메서드들은 `ResponseEntity`를 래핑한 `Mono` 또는 `Flux` 타입을 반환하게 됨.

### 5.4.2 build.gradle에 리액티브 의존성 추가

리액티브 파이프라인 구축을 위해 기존의 `spring-boot-starter-web`을 제거하고 아래 항목을 추가함.

```gradle
implementation 'org.springframework.boot:spring-boot-starter-webflux'
implementation 'org.springframework.boot:spring-boot-starter-data-r2dbc'
runtimeOnly 'io.r2dbc:r2dbc-h2'
testImplementation 'io.projectreactor:reactor-test'
```

- **spring-boot-starter-webflux**: 리액티브 웹 지원
- **spring-boot-starter-data-r2dbc**: 관계형 데이터베이스(H2, MySQL, PostgreSQL 등)와 논-블로킹 통신을 하기 위한 R2DBC 지원
- **reactor-test**: 리액터 단위 테스트 지원

### 5.4.3 엔티티 정의

- `@Table`, `@Id`, `@Column` 등의 스프링 데이터 애노테이션을 사용하여 데이터베이스 테이블과 매핑함.

### 5.4.4 리포지토리 추가

- `ReactiveCrudRepository`를 확장하여 인터페이스를 생성하며, 반환 타입은 `Mono` 또는 `Flux`가 됨.

```java
@Repository
public interface OrderRepository extends ReactiveCrudRepository<OrderEntity, UUID>, OrderRepositoryExt {
    @Query("select o.* from ecomm.orders o join ecomm.user u on o.customer_id = u.id where u.id = :custId")
    Flux<OrderEntity> findByCustomerId(String custId); // Flux 반환
}
```

- **복합 쿼리 처리**: `DatabaseClient`나 `R2dbcEntityTemplate`을 사용해 SQL을 직접 처리하거나 유동적인 조작을 수행할 수 있음.

### 5.4.5 애플리케이션에 H2 콘솔 추가

- 스프링 웹플럭스는 H2 콘솔을 기본적으로 지원하지 않으므로, 별도의 `H2ConsoleComponent` 빈을 정의하여 `org.h2.tools.Server`를 직접 시작해야 함.