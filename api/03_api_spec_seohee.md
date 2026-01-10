# 3. API 명세 및 구현

## 1️⃣ OAS로 API 설계
- **설계 우선(Design-First) 접근 방식**
  - 코드를 작성하기 전에 API 명세(OAS)를 먼저 작성하는 방식임.
  - 개발 도중 잦은 수정, API 관리의 어려움, 부서 간 소통 오류 등의 문제를 방지할 수 있음.
  - REST API 구현의 표준 부재 문제를 해결하기 위해 OAS(OpenAPI Specification)가 도입됨.
- **OAS(OpenAPI Specification)**
  - RESTful API의 명세 및 설명을 위한 표준 인터페이스임.
  - **YAML** 또는 **JSON** 언어로 작성하며, 이 책에서는 가독성이 좋은 YAML(3.0버전)을 사용함.
  - **YAML 작성 규칙**: 공백(Space)에 민감함. 들여쓰기는 공백을 사용하며, 키: 값(`key: value`) 쌍을 쓸 때 콜론 뒤에 반드시 공백이 있어야 함.
- **OAS 지원 도구 (Swagger)**
  - **Swagger Editor**: 브라우저 기반의 편집기로, 실시간으로 명세를 작성하고 미리볼 수 있음.
  - **Swagger Codegen**: 명세서를 바탕으로 서버 스텁 코드나 클라이언트 SDK를 생성해줌.
  - **Swagger UI**: 작성된 명세를 시각화하여 대화형 API 문서를 생성해줌.

## 2️⃣ OAS의 기본 구조 이해
- OAS 정의 파일은 `openapi`, `info`, `externalDocs`, `servers`, `tags`, `paths`, `components` 등의 루트(root) 객체들로 구성됨.
- **메타데이터 절**
  - **`openapi`**: 사용하는 OAS의 버전을 명시함(시맨틱 버전 관리: major.minor.patch).
  - **`info`**: API에 대한 필수 메타데이터를 포함함.
    - `title`: API 제목(필수).
    - `description`: API에 대한 상세 설명(마크다운 지원).
    - `termsOfService`: 서비스 약관 URL.
    - `contact`: 담당자 이름, URL, 이메일 정보.
    - `license`: 라이선스 명 및 링크.
    - `version`: API 버전 문자열(필수).
  - **`externalDocs`**: API와 관련된 추가 문서나 외부 자료의 링크를 제공함.
- **Servers와 Tags 절**
  - **`servers`**: API를 호스팅하는 서버의 URL 목록(개발, 스테이징 등). 생략 시 기본값은 `/`임.
  - **`tags`**: API 오퍼레이션(Endpoint)들을 논리적인 그룹으로 묶어 문서의 가독성을 높임. `name`은 필수이며 `description`과 `externalDocs`를 추가할 수 있음.

## 3️⃣ OAS의 컴포넌트(components) 절
- **개념**: API 전체에서 재사용 가능한 객체(모델, 보안 스키마 등)를 정의하는 곳임. 개념적으로는 모델을 먼저 정의하고, 이를 path 항목에서 참조하여 사용하는 것이 순서임.
- **데이터 타입 (Data Types)**: OAS는 6가지 기본 데이터 타입을 지원함(모두 소문자).
  - `string`, `number`, `integer`, `boolean`, `object`, `array`
- **Format 애트리뷰트**: 기본 데이터 타입을 더 구체적으로 정의할 때 사용함.
  - `integer`: `int32`, `int64`
  - `number`: `float`, `double`
  - `string`: `date`(날짜), `date-time`(날짜와 시간), `byte`(Base64), `binary`(이진 데이터)
- **참조 ($ref)**
    - 중복을 피하기 위해 `schema` 정의 시 다른 모델을 참조할 때 사용함.
    - 예: `$ref: '#/components/schemas/Item'`과 같이 내부 문서를 참조하거나 외부 파일을 참조할 수 있음.
- **객체와 배열 정의**
  - **Object**: `properties` 키워드를 사용해 필드를 구성함.
  - **Array**: `items` 키워드를 사용해 배열 요소의 타입을 정의함.


## 4️⃣ OAS의 경로(path) 절
- **개념**: API의 엔드포인트(URI)와 HTTP 메소드(GET, POST, PUT, DELETE 등)를 정의하는 곳이며, URI는 항상 `/`로 시작함.
- **오퍼레이션 필드 구성 요소**
  - **`tags`**: API를 그룹화할 때 사용하며, 코드 생성 시 클래스 단위 등으로 묶이는 기준이 됨.
  - **`summary` & `description`**: API 작업에 대한 요약과 상세 설명을 작성함 (마크다운 지원).
  - **`operationId`**: 오퍼레이션의 고유 식별자로, **코드 생성 도구(Swagger Codegen)가 API 인터페이스의 메소드 이름을 생성할 때 사용**됨.
- **파라미터(parameters)**
  - `path`, `query`, `header`, `cookie` 등 다양한 위치의 매개변수를 정의할 수 있음.
  - `in`: 매개변수의 위치를 지정함(예: `in: path`).
  - `required`: 필수 여부를 지정함 (path 파라미터는 반드시 `true`여야 함).
  - `-` (하이픈)으로 시작하는 배열 형태로 여러 파라미터를 나열함.
- **응답(responses)**
  - API 요청에 대한 응답 상태 코드(200, 404 등)와 반환 데이터 타입을 정의하는 필수 필드임.
  - `content`: `application/json`과 같은 미디어 타입을 정의하고 `schema`를 통해 반환될 객체 구조를 명시함.
- **요청 본문(requestBody)**
  - POST나 PUT 요청 시 클라이언트가 보낼 데이터(페이로드)를 정의함.
  - `responses`와 유사하게 `content`와 `schema`를 사용하여 데이터 구조를 정의함.


## 5️⃣ OAS를 스프링 코드로 변환
- **프로젝트 준비**
  - Spring Initializr를 통해 스프링 부트 기반의 프로젝트를 생성함.
  - `build.gradle`에 OpenAPI 지원에 필요한 의존성(`swagger-annotations`, `jackson-databind-nullable`, `spring-boot-starter-validation` 등)을 추가함.

- **코드 생성 7단계 프로세스**
    1.  **Gradle 플러그인 추가**: `build.gradle`에 Swagger Gradle 플러그인을 추가하여 자동화 도구를 사용할 준비를 함.
    2.  **Config 파일 정의(`config.json`)**: OpenAPI Generator가 코드를 어떻게 생성할지 옵션을 설정함.
    - `library`: `spring-boot` 지정.
    - `modelPackage`, `apiPackage`: 생성될 패키지 경로 지정.
    - `useTags`: `true`로 설정하여 태그 단위로 인터페이스를 생성.
    - `useSpringBoot3`: 스프링 부트 3 버전과 호환되는 어노테이션(`jakarta.*` 등)을 사용하도록 설정.
    3.  **Ignore 파일 정의**: `.openapi-generator-ignore` 파일을 생성하여, 생성기가 덮어쓰지 말아야 할 파일(예: 직접 구현할 Controller 등)을 지정함.
    - `**/*Controller.java` ➡️ 컨트롤러는 나중에 수동으로 추가함.
    4.  **OAS 파일 배치**: 작성한 `openapi.yaml` 파일을 프로젝트의 리소스 폴더(`src/main/resources/api/`)에 복사함.
    5.  **SwaggerSources 태스크 정의**: `build.gradle`에 입력 파일(yaml), 설정 파일(json), 출력 디렉토리를 지정함.
    6.  **빌드 의존성 설정**: 컴파일 전에 코드가 먼저 생성되도록 `compileJava` 작업이 `swaggerSources`에 의존하게 설정함.
    - `compileJava.dependsOn swaggerSources.eStore.code`, `processResources { dependsOn(generateSwaggerCode) }`
    7.  **SourceSets 추가**: 생성된 코드 경로를 Gradle이 소스 코드로 인식하도록 `sourceSets`에 추가함.

- **코드 생성 실행**
  - `gradlew clean build` 명령어를 실행하면 `/build` 디렉터리에 모델(DTO)과 API 인터페이스가 자동으로 생성됨.
  - IDE에서 처음 로드할 때는 빌드를 먼저 실행해야 생성된 클래스를 인식할 수 있음.

## 6️⃣ OAS 코드 인터페이스 구현
자동 생성된 인터페이스를 활용하여 비즈니스 로직을 구현하는 단계임.

- **생성된 인터페이스의 특징(`CartApi.java`)**
  - OAS(YAML)에 정의한 모든 내용이 반영되어 있음.
  - `@RequestMapping`, `@PathVariable`, `@RequestBody`, `@Valid` 등 요청 처리와 유효성 검사에 필요한 모든 스프링 어노테이션이 포함됨.
  - `tags`를 기준으로 인터페이스가 묶이므로, 같은 태그를 가진 오퍼레이션은 하나의 인터페이스(예: `CartApi`)에 모이게 됨.

- **컨트롤러 구현(`CartController.java`)**
  - 개발자는 생성된 인터페이스를 `implements` 하여 컨트롤러를 만들기만 하면 됨.
  - 인터페이스가 스펙을 완벽하게 정의하고 있으므로, 개발자는 오버라이드(`@Override`)한 메소드 내부의 **비즈니스 로직 구현에만 집중**할 수 있음.

  ```java
  @RestController
  public class CartController implements CartApi { // 생성된 인터페이스 상속

      private static final Logger log = LoggerFactory.getLogger(CartController.class);

      @Override
      public ResponseEntity<List<Item>> addCartItemsByCustomerId(String customerId, @Valid Item item) {
          log.info("고객 ID 요청: {}\nItem: {}", customerId, item);
          return ok(Collections.EMPTY_LIST); // 비즈니스 로직 구현
      }
      
      // ...
  }
  ```

## 7️⃣ 전역 예외 처리기 추가
- 애플리케이션 전역에서 발생하는 예외를 한곳에서 처리하여 유지 보수성을 높이고 클린 코드를 작성하는 방법임.
- **필요성**: 더 나은 유지 관리와 모듈화, 그리고 클린 코드를 위해서는 모든 에러를 처리할 중심 역할이 필요함.
- **구현 구성 요소**
  - **`Error` 모델**: 에러 응답 포맷을 정의하는 클래스 (필드: `errorCode`, `message`, `status`, `url`, `reqMethod`).
  - **`ErrorCode` Enum**: 애플리케이션에서 사용하는 모든 에러 코드와 메시지를 상수로 관리함 (예: `GENERIC_ERROR("PACKT-0001")`).
  - **`ErrorUtils`**: 에러 객체 생성을 돕는 유틸리티 클래스.
  - **`RestApiErrorHandler`클래스 (`@ControllerAdvice`)**:
    - `@ControllerAdvice`를 사용하여 모든 컨트롤러의 예외를 가로챔.
    - `@ExceptionHandler`를 사용하여 특정 예외(예: `HttpMediaTypeNotSupportedException`) 또는 모든 예외(`Exception.class`)를 처리하는 메소드를 정의함.
    - 최종적으로 `ResponseEntity`에 `Error` 객체를 담아 클라이언트에게 반환함.

## 8️⃣ API 구현 테스트
- 코드가 완성되었으면 빌드하고 실행하여 실제 요청을 보내 테스트를 수행함.
- **빌드 및 실행**
  - **빌드**: `./gradlew clean build` 명령어로 기존 빌드 폴더를 제거하고 새로운 아티팩트(JAR)를 생성함.
  - **실행**: `java -jar build/libs/Chapter03-0.0.1-SNAPSHOT.jar` 명령어로 애플리케이션을 구동함.
- **cURL을 이용한 테스트**
  - **GET 요청(예외 테스트)**:
    - 앞서 컨트롤러에서 수동으로 예외를 던지도록 설정한 경로(`/api/v1/carts/1`)를 호출함.
    - 전역 예외 처리기가 동작하여 설정한 포맷(XML 또는 JSON)대로 에러 응답이 오는지 확인함.
  - **Accept 헤더(콘텐츠 협상)**:
    - `Accept: application/xml` 헤더를 보내면 XML로 응답이 옴.
    - `Accept: application/json` 헤더를 보내면 JSON으로 응답이 옴.
    - 이를 통해 스프링이 마샬링/직렬화를 자동으로 처리함을 확인할 수 있음.