# 4. API를 위한 비즈니스 로직 작성

## 4.1 서비스 설계 개요

이 책에서는 **DDD(Domain-Driven Design)** 아키텍처 스타일에 기반한 멀티레이어 아키텍처를 구현함.

### 멀티레이어 아키텍처 구성

- **프레젠테이션 레이어**: 사용자 인터페이스(UI)를 담당함.
- **애플리케이션 레이어**: 비즈니스 로직이 아닌 애플리케이션 로직(전체 흐름 유지 및 조정)을 포함함. RESTful 웹 서비스, gRPC, GraphQL API 등이 이 레이어에 속함.
- **도메인 레이어**: 비즈니스 로직과 도메인 정보(주문, 제품 등)를 포함함. 서비스와 리포지토리로 구성됨.
- **인프라 레이어**: 데이터베이스, 메시지 브로커 등 외부 및 내부 시스템과의 통신 및 상호 작용을 지원함.

개발 방법론 중 **상향식 접근 방식(bottom-to-top approach)** 을 사용하여 도메인 레이어부터 구현을 시작함.

## 4.2 Repository 컴포넌트 추가

### 의존성 추가

데이터베이스 및 JPA 사용을 위해 필요한 라이브러리를 build.gradle에 추가함.

```gradle
implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
implementation 'org.flywaydb:flyway-core'
runtimeOnly 'com.h2database:h2'
```

### 데이터베이스 및 JPA 설정

application.properties에서 H2 인메모리 데이터베이스와 JPA/Hibernate 설정을 구성함.

```properties
# 데이터 소스 설정
spring.datasource.name=ecomm
spring.datasource.url=jdbc:h2:mem:ecomm;DB_CLOSE_DELAY=-1;IGNORECASE=TRUE;DATABASE_TO_UPPER=false
spring.datasource.driverClassName=org.h2.Driver
spring.datasource.username=sa
spring.datasource.password=

# H2 콘솔 설정
spring.h2.console.enabled=true
spring.h2.console.settings.web-allow-others=false

# JPA/Hibernate 설정
spring.jpa.properties.hibernate.default_schema=ecomm
spring.jpa.database-platform=org.hibernate.dialect.H2Dialect
spring.jpa.show-sql=true
spring.jpa.format-sql=true
spring.jpa.generate-ddl=false
spring.jpa.hibernate.ddl-auto=none

# Flyway 설정
spring.flyway.url=jdbc:h2:mem:ecomm
spring.flyway.schemas=ecomm
spring.flyway.user=sa
spring.flyway.password=
```

### 데이터베이스 마이그레이션

- Flyway를 사용하여 데이터베이스 스키마를 관리함.
- 스크립트 파일은 `src/main/resources/db/migration`에 `V<version>__<name>.sql` 형식으로 작성함.

### 엔티티 추가

JPA 엔티티는 `@Entity` 애노테이션을 사용하여 데이터베이스 테이블과 매핑됨.

```java
@Entity
@Table(name = "cart")
public class CartEntity {
    @Id
    @GeneratedValue
    @Column(name = "ID", updatable = false, nullable = false)
    private UUID id;

    @OneToOne
    @JoinColumn(name = "USER_ID", referencedColumnName = "ID")
    private UserEntity user;

    @ManyToMany(cascade = CascadeType.ALL)
    @JoinTable(
        name = "CART_ITEM",
        joinColumns = @JoinColumn(name = "CART_ID"),
        inverseJoinColumns = @JoinColumn(name = "ITEM_ID")
    )
    private List<ItemEntity> items = Collections.emptyList();
}
```

- `@OneToOne`, `@ManyToMany`, `@JoinTable` 등을 사용하여 엔티티 간의 관계를 정의함.

### 리포지토리 추가

스프링 데이터 JPA의 `CrudRepository`를 확장하여 데이터 액세스 메서드를 제공함.

```java
public interface CartRepository extends CrudRepository<CartEntity, UUID> {
    @Query("select c from CartEntity c join c.user u where u.id = :customerId")
    Optional<CartEntity> findByCustomerId(@Param("customerId") UUID customerId);
}
```

- **JPQL(Java Persistence Query Language)** 은 SQL과 유사하지만 테이블 이름 대신 자바 클래스 이름을 사용함.
- `@Query` 애노테이션을 통해 커스텀 쿼리를 정의할 수 있음.

### 사용자 지정 리포지토리 추가

- 기본 CRUD 외에 별도의 구현이 필요한 경우 사용자 정의 인터페이스를 생성하고 구현체를 작성함.
- `@PersistenceContext`를 통해 `EntityManager`를 주입받아 복잡한 쿼리나 네이티브 SQL을 실행할 수 있음.
- Java 15의 **텍스트 블록(text blocks)** 기능을 사용하면 쿼리의 가독성을 높일 수 있음.

## 4.3 서비스 컴포넌트 추가

`@Service` 컴포넌트는 컨트롤러와 리포지토리 사이에서 비즈니스 로직을 처리하는 레이어임.

### 서비스 구현

`CartService` 인터페이스를 생성하고 필요한 모든 작업을 정의한 뒤, `CartServiceImpl` 클래스에서 이를 구현함.

```java
@Service
public class CartServiceImpl implements CartService {
    private final CartRepository cartRepository;
    
    @Override
    public List<Item> addOrReplaceItemsByCustomerId(String customerId, @Valid Item item) {
        // 1. customerId를 기반으로 데이터베이스에서 CartEntity를 가져옴
        CartEntity entity = getCartByCustomerId(customerId);
        List<ItemEntity> items = Objects.nonNull(entity.getItems()) ?
                entity.getItems() : Collections.emptyList();
        AtomicBoolean itemExists = new AtomicBoolean(false);

        // 2. CartEntity 객체에서 검색된 항목을 반복함
        items.forEach(i -> {
            // 주어진 아이템이 이미 있다면 수량과 가격을 변경함
            if (i.getProduct().getId().equals(UUID.fromString(item.getId()))) {
                i.setQuantity(item.getQuantity()).setPrice(i.getPrice());
                itemExists.set(true);
            }
        });

        // 주어진 아이템이 없다면 새 Item 엔티티를 만들어 CartEntity 객체에 저장함
        if (!itemExists.get()) {
            items.add(itemService.toEntity(item));
        }

        // 3. 업데이트된 CartEntity를 저장하고 최신 모델 컬렉션으로 변환해 반환함
        return itemService.toModelList(repository.save(entity).getItems());
    }
}
```

## 4.4 하이퍼미디어 구현

`spring-boot-starter-hateoas` 의존성을 사용하여 하이퍼미디어와 HATEOAS 지원을 추가함.

### 주요 클래스

| 클래스 | 설명 |
|---|---|
| RepresentationModel | 모델/DTO가 이 클래스를 확장하여 링크를 수집할 수 있음 |
| EntityModel | 도메인 객체를 content 필드로 래핑하고 링크를 포함함 |
| CollectionModel | 모델 컬렉션을 래핑하고 링크를 관리하는 방법을 제공함 |
| PageModel | 페이지 메타데이터를 통해 반복하는 방법을 제공함 |

### Swagger Codegen 설정

config.json에서 `"hateoas": true` 설정을 통해 하이퍼미디어를 지원하는 모델을 자동 생성할 수 있음.

### 어셈블러 구현

`RepresentationModelAssemblerSupport`를 확장하여 엔티티를 모델로 변환할 때 링크를 자동으로 생성함.

```java
@Component
public class CartRepresentationModelAssembler 
    extends RepresentationModelAssemblerSupport<CartEntity, CartModel> {
    
    @Override
    public CartModel toModel(CartEntity entity) {
        CartModel model = new CartModel();
        // 엔티티를 모델로 변환
        
        // 자가 참조 링크 추가
        model.add(linkTo(methodOn(CartController.class)
            .getCartByCustomerId(entity.getUser().getId().toString()))
            .withSelfRel());
        
        // 관련 리소스 링크 추가
        return model;
    }
}
```

## 4.5 서비스와 HATEOAS로 컨트롤러 향상

컨트롤러 클래스에 `CartService`와 `CartRepresentationModelAssembler`를 생성자 주입 방식으로 추가함.

```java
@Component
public class CartRepresentationModelAssembler extends 
    RepresentationModelAssemblerSupport<CartEntity, Cart> {

    private final ItemService itemService;

    public CartRepresentationModelAssembler(ItemService itemService) {
        super(CartsController.class, Cart.class);
        this.itemService = itemService;
    }

    @Override
    public Cart toModel(CartEntity entity) {
        // 엔티티에서 사용자 ID와 카트 ID를 추출함
        String uid = Objects.nonNull(entity.getUser()) ? 
            entity.getUser().getId().toString() : null;
        String cid = Objects.nonNull(entity.getId()) ? 
            entity.getId().toString() : null;
        
        Cart resource = new Cart();
        
        // 엔티티의 속성을 모델 객체로 복사함
        BeanUtils.copyProperties(entity, resource);
        
        // 모델의 ID, 고객 ID 및 아이템 목록을 설정함
        resource.id(cid).customerId(uid)
                .items(itemService.toModelList(entity.getItems()));
        
        // 1. 자체 참조(Self) 링크를 추가함
        resource.add(linkTo(methodOn(CartsController.class)
                .getCartByCustomerId(uid)).withSelfRel());
        
        // 2. 카트 아이템 목록에 대한 링크를 추가함
        resource.add(linkTo(methodOn(CartsController.class)
                .getCartItemsByCustomerId(uid)).withRel("cart-items"));
        
        return resource;
    }

    public List<Cart> toListModel(Iterable<CartEntity> entities) {
        if (Objects.isNull(entities)) return Collections.emptyList();
        
        return StreamSupport.stream(entities.spliterator(), false)
                .map(this::toModel)
                .collect(toList());
    }
}
```

## 4.6 API 응답에 ETag 추가

**ETag**는 응답 엔티티의 계산된 해시값이며, 엔티티의 작은 변경에도 값이 변경되는 HTTP 응답 헤더임.

### 조건부 요청의 작동 방식

1. 클라이언트는 `If-None-Match` 헤더에 ETag 값을 넣어 요청을 보냄.
2. 서버는 현재 데이터의 해시값과 클라이언트가 보낸 ETag를 비교함.
3. 값이 동일하면 변경 사항이 없음을 의미하는 `304 NOT MODIFIED` 응답을 보냄.
4. 값이 다르면 새로운 데이터와 함께 새로운 ETag를 응답함.

### ShallowEtagHeaderFilter 구현

스프링의 `ShallowEtagHeaderFilter`를 빈으로 등록하여 가장 쉽고 간단하게 구현할 수 있음.

```java
@Configuration
public class WebConfig {
    @Bean
    public ShallowEtagHeaderFilter shallowEtagHeaderFilter() {
        return new ShallowEtagHeaderFilter();
    }
}
```

- 이 필터는 응답 콘텐츠의 MD5 해시를 계산하여 요청 헤더의 해시와 비교함.
- 대역폭 절약과 서버 부하 감소에 도움을 줌.
- 불필요한 CPU 계산을 피하고 더 나은 ETag를 처리하기 위해 HTTP 캐시 제어(`CacheControl`) 클래스를 사용하거나 각 변경에 대해 업데이트되는 버전 속성을 사용함.