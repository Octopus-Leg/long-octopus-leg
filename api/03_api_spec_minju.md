# 3. API 명세 및 구현

## 3.1 OAS로 API 설계

- 프로그래밍 시에는 **설계 우선(design-first) 접근 방식**을 사용해야 함.
- **OAS(OpenAPI Specification)** 는 REST API의 명세와 설명을 해결하기 위해 도입되었음.
- **YAML(YAML Ain't Markup Language)** 또는 **JSON(JavaScript Object Notation)** 형식으로 REST API를 작성할 수 있음.
- OAS는 **Swagger 명세**에 많이 쓰이게 되면서 널리 알려졌음.

### Swagger 도구

- **Swagger Editor**: REST API를 설계 및 설명 작성
- **Swagger Codegen**: Spring 기반 API 인터페이스 생성
- **Swagger UI**: REST API 문서 생성

## 3.2 OAS 기본 구조 이해

### OAS의 메타데이터 절

- **openapi**: 시맨틱 버전 관리를 사용하며, 버전이 major.minor.patch 형식임.
- **info**: API에 대한 메타데이터가 포함되며, title과 version만 필수 필드임.
- **externalDocs**: 노출된 API의 확장 문서를 가리키는 선택적 필드임.

```yaml
openapi: 3.0.3
info:
  title: 샘플 전자 상거래 앱
  description: '이것은 ***샘플 전자 상거래 앱***이다.'
  termsOfService: https://github.com/LICENSE
  contact:
    email: support@packtpub.com
  license:
    name: MIT
    url: https://github.com/LICENSE
  version: 1.0.0
externalDocs:
  description: API와 함께 생성하고자 하는 문서 링크.
  url: http://swagger.io
```

### OAS의 servers와 tags 절

- **servers**: API를 호스팅하는 서버 목록을 가지는 선택적 항목임.
- **tags**: 리소스에서 수행되는 작업을 그룹화하는 데 사용됨.

### OAS의 경로(path) 절

- **paths**: 엔드포인트를 정의함.
    - 엔드포인트 경로에는 연결된 HTTP 메서드가 있으며, 각 메서드는 필드(tags, summary, description, operationId, parameters, responses, requestBody)를 가질 수 있음.

### OAS의 components 절

- **components**: 재사용 가능한 스키마, 요청 본문, 응답 등을 정의함.
    - 기본 데이터 타입(string, number, integer, boolean, object, array)을 지원함.

## 3.3 OAS를 스프링 코드로 변환

- Swagger 플러그인을 사용하여 API 정의로부터 코드를 생성함.

### 코드 생성 단계

1. Gradle 플러그인 추가
2. 코드 생성을 위한 OpenAPI config 정의
3. OpenAPI 생성기 ignore 파일 정의
4. openapi.yaml에서 OAS 파일 복사해서 /src/main/resources/api에 넣기
5. Gradle 빌드 파일에서 swaggerSources 태스크 정의
6. compileJava 작업 의존성에 swaggerSources 추가
7. 생성된 소스 코드를 Gradle sourceSets에 추가
8. 빌드 실행

## 3.4 OAS 코드 인터페이스 구현

- 생성된 API 인터페이스를 구현하여 비즈니스 로직을 작성함.
- Swagger Codegen은 제공된 각 태그에 대한 API 인터페이스를 생성함.

```java
@RestController
public class CartsController implements CartApi {

  private static final Logger log = LoggerFactory.getLogger(CartsController.class);

  @Override
  public ResponseEntity<List<Item>> addCartItemsByCustomerId(
      String customerId, @Valid Item item) {
    log.info("고객 ID 요청: {}\nItem: {}", customerId, item);
    return ok(Collections.EMPTY_LIST);
  }

  @Override
  public ResponseEntity<List<Cart>> getCartByCustomerId(String customerId) {
    throw new RuntimeException("수동 예외 발생 (Manual Exception thrown)");
  }
}
```

## 3.5 전역 예외 처리기 추가

- 여러 컨트롤러와 메서드에서 발생하는 예외를 중앙에서 처리하기 위해 전역 예외 처리기를 구현함.
- **@ControllerAdvice**: REST 컨트롤러의 모든 요청 및 응답 처리를 추적함.
- **@ExceptionHandler**: 특정 예외를 처리하는 메서드를 정의함.

### Error 모델 정의

```java
public class Error {
  private static final long serialVersionUID = 1L;
  private String errorCode;     // HTTP 오류 코드와 다른 애플리케이션 오류 코드
  private String message;       // 문제에 대한 요약
  private Integer status;       // 문제 발생 시 서버에 설정된 HTTP 상태 코드
  private String url = "Not available";        // 오류를 발생시킨 요청의 URL
  private String reqMethod = "Not available";  // 오류를 발생시킨 요청의 메서드
  // getter와 setter 생략
}
```

### 전역 예외 처리기 구현

```java
@ControllerAdvice
public class RestApiErrorHandler {
  private final MessageSource messageSource;

  @Autowired
  public RestApiErrorHandler(MessageSource messageSource) {
    this.messageSource = messageSource;
  }

  @ExceptionHandler(Exception.class)
  public ResponseEntity<Error> handleException(
      HttpServletRequest request, Exception ex, Locale locale) {
    Error error = ErrorUtils.createError(
        ErrorCode.GENERIC_ERROR.getErrMsgKey(), 
        ErrorCode.GENERIC_ERROR.getErrCode(), 
        HttpStatus.INTERNAL_SERVER_ERROR.value())
      .setUrl(request.getRequestURL().toString())
      .setReqMethod(request.getMethod());
    return new ResponseEntity<>(error, HttpStatus.INTERNAL_SERVER_ERROR);
  }

  @ExceptionHandler(HttpMediaTypeNotSupportedException.class)
  public ResponseEntity<Error> handleHttpMediaTypeNotSupportedException(
      HttpServletRequest request, 
      HttpMediaTypeNotSupportedException ex, Locale locale) {
    Error error = ErrorUtils.createError(
        ErrorCode.HTTP_MEDIATYPE_NOT_SUPPORTED.getErrMsgKey(),
        ErrorCode.HTTP_MEDIATYPE_NOT_SUPPORTED.getErrCode(),
        HttpStatus.UNSUPPORTED_MEDIA_TYPE.value())
      .setUrl(request.getRequestURL().toString())
      .setReqMethod(request.getMethod());
    return new ResponseEntity<>(error, HttpStatus.UNSUPPORTED_MEDIA_TYPE);
  }
  // 생략
}
```

## 3.6 API 구현 테스트

- 빌드 완료 후 curl 명령을 사용하여 API를 테스트할 수 있음.

```bash
# 빌드 및 실행
$ gradlew clean build
$ java -jar build/libs/Chapter03-0.0.1-SNAPSHOT.jar

# XML 응답 요청
$ curl --request GET 'http://localhost:8080/api/v1/carts/1' \
  --header 'Accept: application/xml'

# JSON 응답 요청
$ curl --request GET 'http://localhost:8080/api/v1/carts/1' \
  --header 'Accept: application/json'

# 장바구니에 아이템 추가
$ curl --request POST 'http://localhost:8080/api/v1/carts/1/items' \
  --header 'Content-Type: application/json' \
  --header 'Accept: application/json' \
  --data-raw '{
    "id": "1",
    "quantity": 1,
    "unitPrice": 2.5
  }'
```