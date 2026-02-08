# 5. 비동기 API 설계

## 1️⃣ 리액티브 스트림 이해하기
- 데이터 스트림을 비동기적이고 논-블로킹으로 처리함.
- 백프레셔(back-pressure)를 지원하여 데이터의 흐름을 제어함.
- 기존 자바 스트림의 풀(pull) 모델과 달리, 리액티브 스트림은 발행자가 구독자에게 데이터를 밀어넣는 푸시(push) 모델을 사용함.

### 발행자(Publisher)
- 하나 이상의 구독자에게 데이터 스트림을 제공하는 주체임.
- 구독자가 `subscribe()` 메소드를 통해 등록된 경우에만 데이터를 푸시함.

### 구독자(Subscriber)
- 발행자가 푸시한 데이터를 소비하는 주체임.
- `onSubscribe()`를 통해 구독이 시작되면 `Subscription` 객체를 전달받음.
- `request()` 메소드를 통해 자신이 처리할 수 있는 데이터의 양을 발행자에게 알림. ➡️ **백프레셔** 또는 **흐름 제어**로 부름.
- 데이터 수신(`onNext`), 에러(`onError`), 완료(`onComplete`) 이벤트를 처리함.

### 구독(Subscription)
- 발행자와 구독자 사이의 중재자(mediator) 역할을 수행함.
- 구독자는 이 객체를 통해 데이터 요청(`request`)이나 구독 취소(`cancel`)를 제어함.
- 발행자에게 리소스 정리나 데이터 전송 중단을 요청할 수 있는 수단임.

### 프로세서(Processor)
- 발행자와 구독자 사이에서 다리 역할을 하며 프로세싱 단계(processing stage)를 나타냄.
- `Subscriber`와 `Publisher` 인터페이스를 모두 상속받아, 데이터를 소비하는 동시에 발행하는 역할을 수행함.
- 각 인터페이스에 정의된 컨트랙트(contract)를 모두 따름.


## 2️⃣ 스프링 웹플럭스 살펴보기
- 기존 서블릿(Servlet) API는 블로킹 방식이었음. 서블릿 3.1부터 논-블로킹 I/O를 제공했으나 일부 동기 방식(Filter 등)이 남아있었음.
- 스프링 웹플럭스는 완전한 논-블로킹이며 백프레셔를 지원하여 적은 하드웨어 리소스로 높은 동시성을 제공함.
- Netty, Tomcat, Jetty, Undertow 및 서블릿 3.1 컨테이너 등 다양한 서버를 지원함.

### 리액티브 API 이해
- **리액터 라이브러리 사용**
  - 스프링 웹플럭스는 `프로젝트 리액터`를 핵심 의존성으로 사용함.
  - 발행자(Publisher) 입력을 리액터 타입(`Mono`, `Flux`)으로 조정하여 처리함.

- **Mono와 Flux**
  - `Mono`: 0 또는 1개의 요소를 반환함.
  - `Flux`: 0에서 N개의 요소를 반환함.
  - 둘 다 `CorePublisher` 인터페이스를 구현하는 추상 클래스임.

- **Hot 스트림 vs Cold 스트림**
  - **Cold 스트림**: 기본값. 구독자가 등록될 때마다 소스가 다시 시작됨(재사용 X).
  - **Hot 스트림**: 구독자 존재 여부와 상관없이 데이터가 생성됨.
  - `cache()` 메소드를 사용하여 Cold 스트림을 Hot 스트림으로 변환할 수 있음.

### 리액티브 코어
- 웹 애플리케이션의 HTTP 요청 처리를 위한 기초를 제공함.

- **핵심 컴포넌트**
  - **HttpHandler**: HTTP 서버 API(Netty, Tomcat 등) 위에 요청/응답 핸들러의 추상화를 제공함.
  - **WebHandler**: 사용자 세션, 요청 로케일, 폼 데이터 등을 처리함.
  - **코덱**: 요청 및 응답의 직렬화/역직렬화를 담당함 (Encoder, Decoder 등).

- **서버 어댑터 및 동작 방식**
  - **네이티브 지원**: Netty와 같이 리액티브 스트림을 지원하는 서버는 기본적으로 구독을 수행함.
  - **서블릿 3.1 지원**: 리액티브를 지원하지 않는 서블릿 3.1 컨테이너는 `ServletHttpHandlerAdapter`를 통해 비동기 I/O 간의 조정을 처리함.
  - **응답 처리**: 컨트롤러가 반환한 `Mono`/`Flux` 스트림을 웹플럭스 내부에서 구독하고, 이를 HTTP 패킷으로 변환하여 응답함. JSON의 경우 전체 요소가 수신될 때까지 기다렸다가 직렬화함.

## 3️⃣ DispatcherHandler 이해하기
- 스프링 웹플럭스의 프런트 컨트롤러로, 스프링 MVC의 `DispatcherServlet`과 유사한 역할을 수행함.
- `webHandler`라는 bean 이름으로 식별됨.
- **요청 처리 순서**:
  1. `DispatcherHandler`가 요청을 받음.
  2. `HandlerMapping`을 통해 요청에 맞는 핸들러를 찾음.
  3. `HandlerAdapter`를 통해 핸들러를 실행하고 `HandlerResult`를 반환받음.
  4. `HandlerResultHandler`를 통해 응답을 작성하거나 뷰를 렌더링함.
  5. 요청 처리를 완료함.

### 컨트롤러
- 스프링 MVC와 동일한 애노테이션(`@RestController`, `@RequestMapping`, `@RequestBody` 등)을 사용하여 정의함.
- 동일한 애노테이션을 쓰지만 리액티브 코어 위에서 실행되며 논-블로킹 흐름을 제공함.
- 개발자는 리액티브 체인(파이프라인)이 끊기지 않도록 관리해야 하며, 체인 내에서 블로킹 호출을 피해야 할 책임이 있음.

### 함수형 엔드포인트
- 명령형 프로그래밍 대신 함수형 프로그래밍 스타일을 따름.
- `RouterFunction`: 라우팅 로직을 정의함. `RouterFunctions.route()` 빌더를 사용해 경로와 핸들러를 매핑함.
- **요청 및 응답 처리**:
  - `ServerRequest`를 받아 `ServerResponse`를 반환함.
  - `ServerRequest`: `bodyToMono()`, `pathVariable()` 등을 통해 요청 데이터를 추출함.
  - `ServerResponse`: `ok()`, `notFound()` 등 빌더 패턴으로 불변 응답 객체를 생성함.
- **로직 흐름**:
  - 리포지토리에서 반환된 `Mono` 객체에 대해 `flatMap` 등을 사용하여 비동기적으로 로직을 연결함.
  - 데이터가 없을 경우(nullX 빈 Mono) `switchIfEmpty()` 연산자를 사용하여 404 응답 등으로 분기 처리를 수행함.


## 4️⃣  전자 상거래 앱용 리액티브 API 구현

### 리액티브 API용 OpenAPI Codegen 변경
- `config.json` 설정 파일에서 `reactive` 속성을 `true`로 지정함.

### build.gradle에 리액티브 의존성 추가
- 기존의 블로킹 방식인 `spring-boot-starter-web`을 제거하고, 논-블로킹 방식인 `spring-boot-starter-webflux`를 추가함.
- 데이터베이스 연결을 위해 `r2dbc-h2`와 같은 R2DBC 기반의 비동기 드라이버를 의존성에 포함함.

### 예외 처리
- 리액티브 스트림 흐름 내에서 예외를 처리하기 위해 `Mono.error()` 나 `Flux.error()`를 사용함.
- 데이터가 비어있을 경우 `switchIfEmpty()` 연산자를 사용하여 예외를 발생시킴.
- 에러 발생 시 `onErrorReturn()` 등을 사용하여 기본값을 반환하는 방식 등으로 처리함.

### 컨트롤러에 대한 전역 예외 처리
- **에러 속성 커스터마이징**: `DefaultErrorAttributes`를 상속받아 응답에 포함될 `status`, `message` 등의 속성을 재정의함.
- **핸들러 구현**: `AbstractErrorWebExceptionHandler`를 상속받아 에러 발생 시 로직을 구현함.
- **우선순위 지정**: `@Order(-2)` 애노테이션을 사용하여 스프링 기본 에러 핸들러보다 먼저 실행되도록 설정함.

### API 응답에 하이퍼미디어 링크 추가
- **리액티브 HATEOAS 지원**: 스프링 웹플럭스에서는 `ReactiveRepresentationModelAssembler` 인터페이스를 구현하여 엔터티를 하이퍼미디어 링크가 포함된 모델로 변환함.

### 엔터티 정의
- **Spring Data R2DBC 사용**: Hibernate/JPA 대신 R2DBC를 사용하므로 `@Entity` 대신 Spring Data 애노테이션을 사용함.
- **애노테이션 매핑**:
  - `@Table("이름")`: 데이터베이스 테이블과 매핑함.
  - `@Id`: 기본 키를 지정함.
  - `@Column("col_name")`: 테이블 컬럼과 필드를 매핑함.
- JPA처럼 `@OneToMany` 같은 관계 매핑을 지원하지 않으므로, 외래 키(FK) ID 값만 필드로 가짐.

### 리포지토리 추가
- **ReactiveCrudRepository**: R2DBC는 `CrudRepository` 대신 리액티브 스트림을 반환하는 `ReactiveCrudRepository`를 상속받아 사용함.
- **@Query**: 복잡한 쿼리가 필요한 경우 `@Query("SELECT ...")` 애노테이션을 사용하여 SQL을 직접 정의하고 `Flux`나 `Mono`로 반환받음.
- R2DBC는 연관 엔터티(예: Order 안에 List<Item>)를 자동으로 가져오지 못하므로, 이를 수동으로 처리하는 로직이 필요함.

### 서비스 추가
- R2DBC는 JPA처럼 연관 관계를 자동으로 로딩(Lazy Loading 등)하지 않음. 따라서 관련된 데이터를 각각 조회하여 수동으로 합쳐야 함.
- **`zipWith()` 병합**: `flatMap` 내부에서 `Mono.just(order)`에 `zipWith()`를 호출하여, 비동기로 가져온 연관 데이터들을 `OrderEntity`에 세팅함.
- `subscribeOn()` 스케줄러: 서로 다른 소스(Cart, Item)를 병렬로 가져올 때 `subscribeOn(Schedulers.boundedElastic())`을 사용하여 별도의 스레드 풀에서 실행되도록 조정하기도 함.

### 컨트롤러 구현 추가
- **`cache()`**: 입력 스트림(`Mono`)을 여러 번 구독하거나 데이터 유실을 방지하기 위해 핫 스트림으로 변환함.
- **`zipWhen()`**: 주문 생성 후, 그 결과를 기반으로 매핑 업데이트(`updateMapping`)를 수행하고 두 결과를 묶을 때 사용함.

### 애플리케이션에 H2 콘솔 추가
- H2 콘솔은 서블릿 기반이므로 스프링 웹플럭스에서는 기본적으로 동작하지 않음. 그래서 직접 bean을 정의해서 추가함.

### 애플리케이션 설정 추가
- **Flyway 설정**: DB 마이그레이션은 JDBC URL(`jdbc:h2:...`)을 사용하여 수행하도록 설정함.
