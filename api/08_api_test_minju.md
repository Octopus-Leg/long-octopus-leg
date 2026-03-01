# 8. API 테스트

## 8.1 API와 코드를 수동으로 테스트하기

API 테스트의 주요 유형은 다음과 같음.

- **단위 테스트**: 클래스 메소드 단위를 개발자가 테스트함.
- **통합 테스트**: 서로 다른 레이어 간 컴포넌트 통합을 개발자가 테스트함.
- **컨트랙트 테스트**: API 변경이 클라이언트 코드에 오류를 유발하지 않는지 확인함. 주로 마이크로서비스 기반 개발에 필요함.
- **종단 간(E2E) 테스트**: QA 팀이 UI에서 백엔드까지의 시나리오를 테스트함.
- **사용자 수락 테스트(UAT)**: 비즈니스 사용자가 수행하며 E2E 테스트와 중복될 수 있음.

## 8.2 테스트 자동화

- 모든 테스트는 빌드 과정의 일부로 자동화할 수 있으며, 모든 테스트를 통과한 경우에만 빌드가 성공함.
- **CI/CD 파이프라인**에 통합하면 테스트 시간을 단축할 수 있음.
  - **CI(지속적인 통합)**: 코드 저장소와 통합되어 빌드 → 테스트 → 병합하는 과정
  - **CD(지속적 전달 / 지속적 배포)**: 코드가 자동으로 테스트되고 아티팩트 리포지토리나 컨테이너 레지스트리에 릴리즈됨. 지속적 배포는 모든 단계를 완전 자동화함.

## 8.3 단위 테스트

- 단위 테스트는 작은 코드 단위의 예상 결과를 검증함.
- 코드 커버리지 **90% 이상**, 브랜치 커버리지 **80% 이상**을 목표로 함.
- **코드 커버리지**: 테스트 실행 시 검사되는 코드 라인과 분기(if-else 등)의 수를 나타내는 메트릭

별도 의존성 추가 없이 `build.gradle`에 포함된 아래 스타터로 단위·통합 테스트에 필요한 모든 라이브러리를 사용할 수 있음.

```gradle
test Implementation('org.springframework.boot:spring-boot-starter-test')
```

**주요 테스트 라이브러리**

- **JUnit 5**: JUnit Platform, JUnit Jupiter, JUnit Vintage 세 모듈로 구성됨.
- **AssertJ**: 플루언트 API로 어서션 작성을 단순화하는 라이브러리
- **햄크레스트(Hamcrest)**: 매처(matchers) 기반 어서션 라이브러리. AssertJ보다 플루언트 API가 부족함.
- **모키토(Mockito)**: 모의객체 추가 및 메소드 스텁(stub)을 지원하는 프레임워크

컨트롤러 단위 테스트에는 스프링이 제공하는 **MockMvc**를 사용함. **MockitoExtension**과 함께 사용하면 모의 객체 추가와 메소드 스터빙을 지원받을 수 있음.

### 8.3.1 AssertJ 어서선을 사용해 테스트하기

```java
@ExtendWith(MockitoExtension.class)
public class ShipmentControllerTest {
  private static final String id = "a1b9b31d-e73c-4112-af7c-b68530f38222";
  private MockMvc mockMvc;

  @Mock private ShipmentService service;
  @Mock private ShipmentRepresentationModelAssembler assembler;
  @Mock private MessageSource msgSource;

  @InjectMocks
  private ShipmentController controller;
  private ShipmentEntity entity;
  private Shipment model = new Shipment();
  private JacksonTester<List<Shipment>> shipmentTester;
}
```

- **`@ExtendWith(MockitoExtension.class)`**: Mockito 확장 모듈을 등록해 모의객체 삽입과 메소드 스터빙을 지원함.
- **`@InjectMocks`**: 테스트 클래스에 필요한 모든 모의 객체를 자동으로 찾아 주입함.
- **`@BeforeAll`**: 테스트 클래스 당 한 번만 실행되며 퍼블릭 정적 메소드에 사용함.
- **`@BeforeEach`**: 각 테스트 메소드 실행 전 매번 실행되며 퍼블릭 비정적 메소드에 적용함.

`@BeforeEach`에서 MockMvc를 설정하고, 테스트는 **Given > When > Then**(BDD 스타일)으로 작성함.

```java
@BeforeEach
public void setup() {
  ObjectMapper mapper = new AppConfig().objectMapper();
  JacksonTester.initFields(this, mapper);
  MappingJackson2HttpMessageConverter mappingConverter =
    new MappingJackson2HttpMessageConverter();
  mappingConverter.setObjectMapper(mapper);
  mockMvc = MockMvcBuilders.standaloneSetup(controller)
              .setControllerAdvice(new RestApiErrorHandler(msgSource))
              .setMessageConverters(mappingConverter)
              .build();
}

@Test
@DisplayName("returns shipments by given order ID")
public void testGetShipmentByOrderId() throws Exception {
  // given
  given(service.getShipmentByOrderId(id)).willReturn(List.of(entity));
  given(assembler.toListModel(List.of(entity))).willReturn(List.of(model));
  // when
  MockHttpServletResponse response = mockMvc.perform(
    get("/api/v1/shipping/" + id)
      .contentType(MediaType.APPLICATION_JSON)
      .accept(MediaType.APPLICATION_JSON))
      .andDo(print())
      .andReturn().getResponse();
  // then
  assertThat(response.getStatus()).isEqualTo(HttpStatus.OK.value());
  assertThat(response.getContentAsString())
    .isEqualTo(shipmentTester.write(List.of(model)).getJson());
}
```

### 8.3.2 스프링과 햄크레스트 어서선을 사용한 테스팅

- **`MockMvcResultMatchers.jsonPath()`**: JSON 경로 표현식과 매처를 아규먼트로 받음.
- **`Is.is()`** 햄크레스트 매처를 통해 `Is.is(equalsTo(...))`를 간단히 표현할 수 있음.

```java
@Test
@DisplayName("returns address by given existing ID")
public void getAddressByOrderIdWhenExists() throws Exception {
  given(service.getAddressesById(id)).willReturn(Optional.of(entity));
  // when
  ResultActions result = mockMvc.perform(
    get("/api/v1/addresses/a1b9b31d-e73c-4112-af7c-b68530f38222")
      .contentType(MediaType.APPLICATION_JSON)
      .accept(MediaType.APPLICATION_JSON));
  // then
  result.andExpect(status().isOk());
  verifyJson(result);
}

private void verifyJson(final ResultActions result) throws Exception {
  final String BASE_PATH = "http://localhost";
  result
    .andExpect(jsonPath("id", is(entity.getId().toString())))
    .andExpect(jsonPath("city", is(entity.getCity())))
    .andExpect(jsonPath("links[0].rel", is("self")))
    .andExpect(jsonPath("links[0].href", is(BASE_PATH + "/" + entity.getId())));
}
```

### 8.3.3 프라이빗 메소드 테스트

- **`ReflectionTestUtils.invokeMethod()`**: 대상 객체, 메소드 이름, 아규먼트(가변 인수) 세 가지를 받아 프라이빗 메소드를 호출함.

```java
@Test
@DisplayName("returns an AddressEntity when private method toEntity() is called with Address model")
public void convertModelToEntity() {
  // given
  AddressServiceImpl srvc = new AddressServiceImpl(repository);
  // when
  AddressEntity e = ReflectionTestUtils.invokeMethod(srvc, "toEntity", addAddressReq);
  // then
  then(e).as("Check address entity is returned and not null").isNotNull();
  then(e.getNumber()).as("Check house/flat no is set").isEqualTo(entity.getNumber());
}
```

- **`BDDAssertions.then()`**: 검증 대상 값을 받음.
- **`as()`**: 어서션 설명을 추가하며 어서션 수행 전에 호출해야 함.

### 8.3.4 void 메소드 테스트하기

- 반환 값이 없는 메소드는 Mockito의 **`doNothing()`** / BDDMockito의 **`willDoNothing()`** 으로 스텁함.
- 실제 객체의 특정 메소드만 스텁하려면 **`spy()`** 를 사용함.

```java
List linkedList = new LinkedList();
List spyLinkedList = spy(linkedList);
doNothing().when(spyLinkedList).clear();
```

### 8.3.5 예외에 대한 테스트 작성

- 예외를 발생시키는 메소드는 **`thenThrow()`** / BDDMockito의 **`willThrow()`** 로 스텁함.
- **`verify()`** 와 **`times()`** 로 메소드 호출 횟수를 검증함.

```java
@Test
@DisplayName("delete address by given non-existing id, should throw ResourceNotFoundException")
public void deleteAddressesByNonExistId() throws Exception {
  given(repository.findById(UUID.fromString(nonExistId))).willReturn(Optional.empty())
      .willThrow(new ResourceNotFoundException(
          String.format("No Address found with id %s.", nonExistId)));
  // when
  try {
    service.deleteAddressesById(nonExistId);
  } catch (Exception ex) {
    // then
    assertThat(ex).isInstanceOf(ResourceNotFoundException.class);
    assertThat(ex.getMessage()).contains("No Address found with id " + nonExistId);
  }
  verify(repository, times(1)).findById(UUID.fromString(nonExistId));
  verify(repository, times(0)).deleteById(UUID.fromString(nonExistId));
}
```

### 8.3.6 단위 테스트 실행하기

```bash
$ ./gradlew clean test
```

실행 결과는 `Chapter08/build/reports/tests/test/index.html`에서 확인할 수 있음.

## 8.4 코드 커버리지

- **JaCoCo** 툴로 라인·브랜치 커버리지를 측정함.
- `build.gradle`에 jacoco 플러그인을 추가함.

```gradle
plugins {
  id 'org.springframework.boot' version '3.0.4'
  id 'io.spring.dependency-management' version '1.1.0'
  id 'java'
  id 'org.hidetake.swagger.generator' version '2.19.2'
  id 'jacoco'
}

jacoco {
  toolVersion = "0.8.8"
  reportsDir = file("$buildDir/jacoco")
}
```

- 자동 생성 코드를 제외한 **`jacocoTestReport`** 태스크와 최소 커버리지 **90%** 를 강제하는 **`jacocoTestCoverageVerification`** 을 설정함.

```gradle
jacocoTestReport {
  dependsOn test
  afterEvaluate {
    classDirectories.setFrom(files(classDirectories.files.collect {
      fileTree(
        dir: it,
        exclude: [
          'com/packt/modern/api/model/*',
          'com/packt/modern/api/*Api.*',
          'com/packt/modern/api/security/UNUSED/*',
        ]
      )
    }))
  }
}

jacocoTestCoverageVerification {
  violationRules {
    rule {
      limit {
        minimum = 0.9
      }
    }
  }
}

test {
  useJUnitPlatform()
  finalizedBy(jacocoTestReport)
}
```

```bash
$ ./gradlew clean build
```

커버리지 리포트는 `Chapter08/build/jacoco/test/htm/index.html`에서 확인할 수 있음.

## 8.5 통합 테스트하기

- 스프링 테스트 라이브러리가 통합 테스트에 필요한 모든 기능을 제공하므로 별도 플러그인·라이브러리 추가는 불필요함.

### 8.5.1 통합 테스트를 위한 설정하기

별도 소스셋과 태스크를 `build.gradle`에 설정함.

```gradle
sourceSets {
  integrationTest {
    java {
      compileClasspath += main.output + test.output
      runtimeClasspath += main.output + test.output
      srcDir file('src/integration/java')
    }
    resources.srcDir file('src/integration/resources')
  }
}

configurations {
  integration TestImplementation.extendsFrom testImplementation
  integration TestRuntime.extendsFrom testRuntime
}

tasks.register('integrationTest', Test) {
  useJUnitPlatform()
  description = 'Runs the integration tests.'
  group = 'verification'
  testClassesDirs = sourceSets.integrationTest.output.classesDirs
  classpath = sourceSets.integrationTest.runtimeClasspath
}

check.dependsOn integrationTest
integrationTest.mustRunAfter test
```

### 8.5.2 통합 테스트를 위한 지원 클래스 작성

- 통합 테스트는 모의 객체 없이 실제 애플리케이션처럼 동작함.
- Flyway 스크립트로 H2 인메모리 DB를 구성하며 자체 `application.properties`를 사용함.
- **`AuthClient`**: JWT 토큰 발급을 위해 **`TestRestTemplate`** 을 사용하는 헬퍼 클래스

```java
public class AuthClient {
  private TestRestTemplate restTemplate;
  private ObjectMapper objectMapper;

  public AuthClient(TestRestTemplate restTemplate, ObjectMapper objectMapper) {
    this.restTemplate = restTemplate;
    this.objectMapper = objectMapper;
  }

  public SignedInUser login(String username, String password) {
    SignInReq signInReq = new SignInReq().username(username).password(password);
    return restTemplate.execute("/api/v1/auth/token", HttpMethod.POST,
      request -> {
        objectMapper.writeValue(request.getBody(), signInReq);
        request.getHeaders().add(HttpHeaders.CONTENT_TYPE, MediaType.APPLICATION_JSON_VALUE);
        request.getHeaders().add(HttpHeaders.ACCEPT, MediaType.APPLICATION_JSON_VALUE);
      },
      response -> objectMapper.readValue(response.getBody(), SignedInUser.class));
  }
}
```

AddressControllerIT 통합 테스트 클래스의 주요 설정은 다음과 같음.

```java
@ExtendWith(SpringExtension.class)
@SpringBootTest(webEnvironment = WebEnvironment.RANDOM_PORT,
    properties = "spring.flyway.clean-disabled=false")
@TestPropertySource(locations = "classpath:application-it.properties")
@TestMethodOrder(OrderAnnotation.class)
@TestInstance(TestInstance.Lifecycle.PER_CLASS)
public class AddressControllerIT {
  @Autowired private AddressRepository repository;
  @Autowired private TestRestTemplate restTemplate;

  @BeforeAll
  public static void init(@Autowired Flyway flyway) {
    objectMapper = TestUtils.objectMapper();
    flyway.clean();
    flyway.migrate();
  }

  @BeforeEach
  public void setup(TestInfo info) throws JsonProcessingException {
    if (Objects.isNull(signedInUser) || isTokenExpired(signedInUser.getAccessToken())) {
      authClient = new AuthClient(restTemplate, objectMapper);
      signedInUser = info.getTags().contains("NonAdminUser")
        ? authClient.login("scott", "tiger")
        : authClient.login("scott2", "tiger");
    }
  }
}
```

- **`@SpringBootTest`**: 모든 의존성과 컨텍스트를 제공하며 랜덤 포트로 테스트 서버를 실행함.
- **`@TestMethodOrder + @Order`**: POST가 DELETE보다 먼저 실행되도록 순서를 지정함.
- **`@TestInstance(PER_CLASS)`**: 테스트 인스턴스 수명 주기를 클래스 단위로 설정함.

실제 API 호출은 **`TestRestTemplate.exchange()`** 로 수행하며, URI·HTTP 메소드·HttpEntity·반환 타입 네 가지를 아규먼트로 받음.

```java
MultiValueMap<String, String> headers = new LinkedMultiValueMap<>();
headers.add(HttpHeaders.CONTENT_TYPE, MediaType.APPLICATION_JSON_VALUE);
headers.add(HttpHeaders.ACCEPT, MediaType.APPLICATION_JSON_VALUE);
headers.add("Authorization", "Bearer " + signedInUser.getAccessToken());

ResponseEntity<JsonNode> addressResponseEntity = restTemplate
    .exchange("/api/v1/addresses", HttpMethod.GET,
        new HttpEntity<>(headers), JsonNode.class);

assertThat(addressResponseEntity.getStatusCode()).isEqualTo(HttpStatus.OK);
List<Address> addressFromResponse = objectMapper
    .convertValue(addressResponseEntity.getBody(), new TypeReference<ArrayList<Address>>() {});
assertThat(addressFromResponse).hasSizeGreaterThan(0);
assertThat(addressFromResponse.get(0)).hasFieldOrProperty("links");
assertThat(addressFromResponse.get(0)).isInstanceOf(Address.class);
```

```bash
$ gradlew clean integrationTest
# 또는
$ gradlew clean build
```

리포트는 `Chapter08/build/reports/tests/integrationTest`에서 확인할 수 있음.
