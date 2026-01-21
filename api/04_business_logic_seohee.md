# 4. API를 위한 비즈니스 로직 작성

## 1️⃣ 서비스 설계 개요
- **Layered Architecture (계층형 아키텍처)**
  - 이 책에서는 4개의 레이어로 구성된 아키텍처를 구현함 (DDD 스타일).
  - **프레젠테이션 레이어**: 사용자 인터페이스(UI) 담당.
  - **애플리케이션 레이어**: 전체 흐름 유지 및 조정. REST API 컨트롤러가 여기에 해당함.
  - **도메인 레이어**: **핵심 비즈니스 로직**과 상태(Entity)를 포함함. 서비스와 리포지토리로 구성됨.
  - **인프라 레이어**: 데이터베이스 등 외부 시스템과의 통신 담당.
- **구현 순서 (Bottom-up)**
  - 상향식 접근 방식을 사용함.
  - **Repository(DB 접근) ➡️ Service(로직) ➡️ Controller(요청 처리)** 순서로 구현.

## 2️⃣ Repository 컴포넌트 추가
- **`@Repository` 애노테이션**
  - 해당 클래스가 데이터 접근 계층(DAO)임을 스프링에게 알려주는 역할을 함.
  - DDD의 리포지토리와 Java EE의 DAO 패턴을 모두 나타내는 스테레오 타입임.
- **필수 의존성 라이브러리**
  - **H2 Database**: 개발 및 테스트용 인메모리 데이터베이스.
  - **Spring Data JPA (Hibernate)**: 자바 객체와 DB 테이블을 매핑해주는 ORM 기술.
  - **Flyway**: 데이터베이스 형상 관리(버전 관리) 및 마이그레이션 도구.

### 2-1. 데이터베이스 및 JPA 설정
- **application.properties 설정**
  - **Data Source**: H2 DB 접속 URL(`jdbc:h2:mem:ecomm`), 드라이버 등을 설정.
  - **H2 Console**: 웹 브라우저에서 DB를 확인하기 위해 `enabled=true` 설정. ➡️ http://localhost:8080/h2-console/ 로 접속함.
  - **JPA**: DDL 자동 생성 끄기(`generate-ddl=false`,`ddl-auto=none` - Flyway를 쓰기 때문).
  - **Flyway**: 마이그레이션 스크립트 위치 등을 설정.
### 2-2. 데이터베이스 및 시드 데이터 스크립트
  - `src/main/resources/db/migration/V1.0.0__Init.sql` 경로에 SQL 파일을 배치함.
  - `CREATE TABLE`, `INSERT INTO` 등의 SQL을 작성하여 앱 실행 시 자동으로 테이블을 만들고 초기 데이터를 넣음.

### 2-3. 엔터티 추가
- **@Entity 애노테이션**
  - 자바 클래스를 데이터베이스 테이블과 1:1로 매핑함.
- **매핑 애노테이션**
  - **@Table**: 매핑할 DB 테이블 이름을 명시함.
  - **관계 매핑**:
    - `@OneToOne`: 1:1 관계
    - `@ManyToMany`: 다대다 관계(Cart - Item). `@JoinTable`을 사용하여 중간 테이블 매핑함.
- **FetchType & OrphanRemoval**
  - `FetchType.LAZY`: 연관된 데이터를 실제로 사용할 때 조회함.
  - `orphanRemoval=true`: 부모 엔터티에서 자식 엔터티를 리스트에서 제거하면, DB에서도 해당 데이터가 삭제됨.

### 2-4. 리포지토리 추가
- **기본 리포지토리**
  - `CrudRepository<Entity, ID>`를 상속받는 인터페이스를 정의하면, 기본적인 CRUD 메소드가 자동 생성됨.
- **@Query와 JPQL**
  - `@Query` 애노테이션을 사용하여 직접 쿼리를 작성할 수 있음.
  - **JPQL**: SQL과 유사하지만 테이블명 대신 **자바 클래스명**을 사용하는 객체지향 쿼리 언어.
- **커스텀 리포지토리 구현**
  - 표준 CRUD 외에 복잡한 로직이 필요할 때 사용
  - **구조**: `OrderRepository`(인터페이스) ➡️ `OrderRepositoryExt`(커스텀 인터페이스) 상속.
  - **인터페이스(`OrderRepositoryExt`)를 구현한 구현체(`OrderRepositoryImpl`)**:
    - `@PersistenceContext`로 `EntityManager`를 주입받아 직접 DB를 제어함.
    - `@PersistenceContext`를 사용하면 수동으로 네이티브 쿼리 등을 사용하여 복잡한 SQL을 실행할 수 있음.(텍스트 블록을 사용)
    - `@Transactional` 사용해서 롤백을 보장함.

## 3️⃣ 서비스 컴포넌트 추가
- **@Service 애노테이션**
  - 스프링이 컴포넌트 스캔을 통해 자동으로 빈으로 등록하는 클래스임.(`@Component`)
  - 비즈니스 로직을 추가하는 데 사용됨.
  - **비즈니스 서비스 파사드(Business Service Façade)** 패턴을 구현하며, 컨트롤러와 리포지토리 사이의 중간 계층 역할을 함.
- **CartService 구현**
  - 책에서는 `CartService` 인터페이스를 정의하고, `CartServiceImpl`에서 이를 구현함.
  - 리포지토리를 생성자 주입으로 받음.
  - **주요 로직**:
    - `addOrReplaceItemsByCustomerId`: 고객 ID로 카트를 찾고(없으면 생성), 아이템이 이미 있으면 수량/가격을 업데이트하고 없으면 새로 추가함.

## 4️⃣ 하이퍼미디어 구현
- REST API의 핵심 원칙 중 하나로, 클라이언트에게 데이터뿐만 아니라 `다음에 할 수 있는 행동(링크)` 을 함께 제공하는 것.
- **스프링 HATEOAS 라이브러리**
  - `WebMvcLinkBuilder`: 컨트롤러의 메소드를 가리키는 링크를 동적으로 생성해줌(`linkTo`, `methodOn` 사용).
  - `RepresentationModel`: DTO가 이 클래스를 상속받으면 `add()` 메소드를 통해 링크를 추가할 수 있음.
- **Assembler 구현(RepresentationModelAssembler)**
  - 엔터티를 HATEOAS 지원 모델로 변환하는 전용 클래스.
  - `RepresentationModelAssemblerSupport` 추상 클래스를 extends해서 사용함.
  - `toModel()` 메소드를 오버라이딩함.
  - 이렇게 하면 컨트롤러 코드가 깔끔해지고 링크 생성 로직을 재사용할 수 있음.

## 5️⃣ 서비스와 HATEOAS로 컨트롤러 향상
- **컨트롤러 업그레이드**
  - 기존에는 단순히 `List<Item>`을 반환했다면, 이제는 `Service`를 호출하여 비즈니스 로직을 수행하고, `Assembler`를 통해 링크가 포함된 모델을 반환하도록 수정함.
  - `return ok(assembler.toModel(service.getCart(...)));`로 사용함.

## 6️⃣ API 응답에 ETag 추가
- **ETag ＝ Entity Tag**
  - 리소스의 특정 버전을 나타내는 해시 또는 이에 상응하는 값.
- **ShallowEtagHeaderFilter**
  - 스프링에서 제공하는 필터로, 별도 구현 없이 빈(`@Bean`)으로 등록만 하면 동작함.
  - 응답 본문 Body를 MD5로 해싱하여 `ETag` 헤더를 생성해줌.
- **동작 방식**
  1.  클라이언트가 `If-None-Match: "해시값"` 헤더를 보내 요청함.
  2.  서버는 현재 데이터의 해시값을 계산해서 비교함.
  3.  값이 같으면 **`304 Not Modified`** 응답을 보내고 응답 본문 Body는 보내지 않음. ➡️ 대역폭 절약되지만 동일한 CPU 계산은 필요
  4.  값이 다르면 **`200 OK`** 와 함께 새로운 데이터와 ETag를 보냄.
