# 11장. gRPC API 개발 및 테스트

## API 작성

결제 서비스에 대해 **프로토콜 버퍼(Protobuf)** 를 이용해서 API를 작성함.

### 프로젝트 설정

Chapter11 디렉터리 아래에는 세 개의 프로젝트가 포함됨.

| 프로젝트 | 설명 |
|---------|------|
| **API** | `.proto` 파일과 생성된 자바 클래스를 포함하는 라이브러리. `payment-gatewayapi-0.0.1.jar`를 생성해서 로컬 리포지토리에 게시. 서버·클라이언트 모두에서 사용. |
| **Server** | gRPC 서비스를 구현하고 gRPC 요청을 처리하는 서버. |
| **Client** | gRPC 서버를 호출할 클라이언트. REST 호출을 구현하고, HTTP 요청을 처리하기 위해 내부적으로 gRPC 서버를 호출. |

### gRPC 서버, 클라이언트 프로젝트 생성

Spring Initialzr(https://start.spring.io/ )에서 다음 옵션으로 프로젝트를 생성함.

| 항목 | 값 | 항목 | 값 |
|------|---|------|---|
| Project | Gradle - Groovy | Packaging | Jar |
| Language | Java | Java | 17 |
| Spring Boot | 3.0.8 | ADD DEPENDENCIES | Spring Web |
| Group | com.packt.modern.api | Artifact | chapter11 |

### gRPC API 라이브러리 프로젝트 생성

Chapter11 디렉터리에 `api` 디렉터리를 만들고 그래들로 초기화함.

```bash
$ mkdir api && cd api
$ ../server/gradlew init
```

터미널 선택 옵션 (강조 표시된 항목 선택):

| 항목 | 선택 | 항목 | 선택 |
|------|------|------|------|
| project type | **3: library** | Project name | api |
| language | **3: Java** | Source package | com.packt.modern.api |
| build script DSL | **1: Groovy** | new APIs | no |
| test framework | **4: JUnit Jupiter** | | |

### gRPC API 라이브러리 프로젝트 구성

Protobuf와 Maven Publish 플러그인을 사용해서 `api/lib/build.gradle`을 7단계로 설정함.

```groovy
// 1. settings.gradle
rootProject.name = 'payment-gateway-api'

// 2. plugins
plugins {
    id 'java'
    id 'maven-publish'                              // Jar 아티팩트를 로컬 저장소에 게시
    id "com.google.protobuf" version "0.9.2"
}

// 3. 그룹, 버전, 소스 호환성
group = 'com.packt.modern.api'
version = '0.0.1'
sourceCompatibility = JavaVersion.VERSION_17

// 4. 의존성
def grpcVersion = '1.54.0'
dependencies {
    implementation "io.grpc:grpc-protobuf:${grpcVersion}"
    implementation "io.grpc:grpc-stub:${grpcVersion}"
    implementation "io.grpc:grpc-netty:${grpcVersion}"
    implementation 'javax.annotation:javax.annotation-api:1.3.2'
}

// 5. Protobuf 그래들 플러그인 구성
protobuf {
    protoc { artifact = "com.google.protobuf:protoc:3.22.2" }
    plugins { grpc { artifact = "io.grpc:protoc-gen-grpc-java:1.54.0" } }
    generateProtoTasks { all()*.plugins { grpc {} } }
}

// 6. sourceSets (IDE 컴파일 에러 방지)
sourcesets {
    main { proto { srcDir "src/main/grpc" } }
}
task sourcesJar(type: Jar, dependsOn: classes) {
    archiveClassifier = "sources"
    from sourceSets.main.allSource
}

// 7. Maven Publish 구성
publishing {
    publications {
        mavenJava(MavenPublication) {
            artifactId = 'payment-gateway-api'
            from components.java
        }
    }
}
```

## 결제 게이트웨이 기능 작성

### 온라인 결제 워크플로우 단계

이 장에서는 **ChargeService** 와 **SourceService** 두 개의 gRPC 서비스를 구현함.

1. 고객은 결제를 시작하기 전에 결제 출처(**Source**)를 생성함.
2. 결제 출처에 대한 청구(**Charge**)를 생성하면 결제가 시작됨.
3. 결제 게이트웨이는 유효성 검사와 확인 단계를 수행한 후, 발행 은행에서 판매자 계정으로 자금 이체를 트리거함.

### 결제 게이트웨이 서비스 정의 작성

`api/lib/src/main/proto/PaymentGatewayService.proto` 파일을 생성하고 작성함.

```protobuf
// 1. 파일 메타데이터
syntax = "proto3";                                          // proto3 버전 사용
package com.packtpub.v1;                                    // 메시지 형식에 네임스페이스 연결
option java_package = "com.packt.modern.api.grpc.v1";      // 생성될 자바 파일의 패키지
option java_multiple_files = true;                          // 타입별 별도 자바 파일 생성

// 2. ChargeService 정의
service ChargeService {
    rpc Create(CreateChargeReq) returns (CreateChargeReq.Response);
    rpc Retrieve(ChargeId) returns (ChargeId.Response);
    rpc Update(UpdateChargeReq) returns (UpdateChargeReq.Response);
    rpc Capture(CaptureChargeReq) returns (CaptureChargeReq.Response);  // 청구되지 않은 과금 결제
    rpc RetrieveAll(CustomerId) returns (CustomerId.Response);
}

// 3. SourceService 정의
service SourceService {
    rpc Create(CreateSourceReq) returns (CreateSourceReq.Response);
    rpc Retrieve(SourceId) returns (SourceId.Response);
    rpc Update(UpdateSourceReq) returns (UpdateSourceReq.Response);
    rpc Attach(AttachOrDetachReq) returns (AttachOrDetachReq.Response); // Source를 고객에게 연결
    rpc Detach(AttachOrDetachReq) returns (AttachOrDetachReq.Response); // Source를 고객으로부터 분리
}

// 4. enum 정의
enum Flow { REDIRECT = 0; RECEIVER = 1; CODEVERIFICATION = 2; NONE = 3; }
enum Usage { REUSABLE = 0; SINGLEUSE = 1; }

// 5. Source 메시지
message Source {
    string id = 1; uint32 amount = 2; string clientSecret = 3;
    uint64 created = 4; string currency = 5;
    Flow flow = 6; Owner owner = 7; Receiver receiver = 8;
    string statementDescriptor = 9;
    enum Status { CANCELLED = 0; CHARGEABLE = 1; CONSUMNED = 2; FAILED = 3; PENDING = 4; }
    Status status = 10; string type = 11; Usage usage = 12;
}

// 6. ChargeService 메시지 타입
message CreateChargeReq {
    uint32 amount = 1; string currency = 2; string customerId = 3;
    string description = 4; string receiptEmail = 5;
    Source sourceId = 6; string statementDescriptor = 7;
    message Response { Charge charge = 1; }
}
message CustomerId {
    string id = 1;
    message Response { repeated Charge charge = 1; }  // repeated: 리스트 반환
}

// 7. SourceService 메시지 타입
message CreateSourceReq {
    string type = 1; uint32 amount = 2; string currency = 3;
    Owner owner = 4; Flow flow = 6; Receiver receiver = 7; Usage usage = 8;
    message Response { Source source = 1; }
}
message AttachOrDetachReq {
    string sourceId = 1; string customerId = 2;
    message Response { Source source = 1; }
}
```

> **빈 요청/응답 타입**: `google.protobuf.Empty`를 사용할 수 있음. 사용 전 `import "google/protobuf/timestamp.proto";`를 파일 상단에 추가함.

### 결제 게이트웨이 서비스 gRPC 서버, 스텁, 모델 게시

```bash
# api 프로젝트 루트 디렉터리에서 실행
$ export JAVA_TOOL_OPTIONS="-Dfile.encoding=UTF8"   # 자바 파일 인코딩 UTF-8 설정
$ gradlew clean publishToMavenLocal
```

`generateProto` 태스크가 생성하는 두 가지 타입의 자바 클래스:

| 타입 | 생성 경로 | 포함 클래스 |
|------|----------|------------|
| **Models** | `.../generated/source/proto/main/java` | 요청·응답 모델 클래스 (`CreateChargeReq`, `Charge.java` 등) |
| **gRPC classes** | `.../generated/source/proto/main/grpc` | `ChargeServiceGrpc.java`, `SourceServiceGrpc.java`<br>각 파일에 ImplBase(추상 기본 클래스) + BlockingStub / Stub / FutureStub 포함 |

---

## gRPC 서버 개발

### server/build.gradle 설정

```groovy
// settings.gradle
rootProject.name = 'chapter11-server'

// build.gradle
def grpcVersion = '1.54.1'
dependencies {
    implementation 'com.packt.modern.api:payment-gateway-api:0.0.1'
    implementation "io.grpc:grpc-protobuf:${grpcVersion}"
    implementation "io.grpc:grpc-stub:${grpcVersion}"
    implementation "io.grpc:grpc-netty:${grpcVersion}"
    implementation 'com.google.api.grpc:googleapis-common-protos:0.0.3'
    implementation 'org.springframework.boot:spring-boot-starter-web'
    testImplementation "io.grpc:grpcTesting:${grpcVersion}"
}
repositories {
    mavenCentral()
    mavenLocal()   // payment-gateway-api는 로컬 저장소에 게시되므로 반드시 추가
}
```

### gRPC 서버 구현

REST 구현과 동일한 계층형 아키텍처를 사용함: **저장소(DbStore) → 리포지토리 → 서비스 → GrpcServer**

#### ① DbStore.java — 인-메모리 지속성 저장소

```java
@Component
public class DbStore {
    // Thread-safe한 인-메모리 저장소 (실제 운영 시 외부 DB로 교체 가능)
    private static final Map<String, Source> sourceEntities = new ConcurrentHashMap<>();
    private static final Map<String, Charge> chargeEntities = new ConcurrentHashMap<>();

    public CreateSourceReq.Response createSource(CreateSourceReq req) {
        Source source = Source.newBuilder()
            .setId(RandomHolder.randomKey())
            .setType(req.getType()).setAmount(req.getAmount())
            .setCurrency(req.getCurrency()).setFlow(req.getFlow()).setUsage(req.getUsage())
            .setCreated(Instant.now().getEpochSecond()).build();
        sourceEntities.put(source.getId(), source);
        return CreateSourceReq.Response.newBuilder().setSource(source).build();
    }

    public SourceId.Response retrieveSource(String sourceId) {
        if (Strings.isBlank(sourceId)) {            // 유효성 검사
            com.google.rpc.Status status = com.google.rpc.Status.newBuilder()
                .setCode(Code.INVALID_ARGUMENT.getNumber())
                .setMessage("Invalid Source ID is passed.")
                .addDetails(Any.pack(SourceId.Response.getDefaultInstance())).build();
            throw StatusProto.toStatusRuntimeException(status);
        }
        Source source = sourceEntities.get(sourceId);
        if (Objects.isNull(source)) {
            com.google.rpc.Status status = com.google.rpc.Status.newBuilder()
                .setCode(Code.INVALID_ARGUMENT.getNumber())
                .setMessage("Requested source is not available").build();
            throw StatusProto.toStatusRuntimeException(status);
        }
        return SourceId.Response.newBuilder().setSource(source).build();
    }
}
```

#### ② 리포지토리 클래스 작성

```java
@Repository
public class ChargeRepositoryImpl implements ChargeRepository {
    private DbStore dbStore;
    public ChargeRepositoryImpl(DbStore dbStore) { this.dbStore = dbStore; }

    @Override
    public CreateChargeReq.Response create(CreateChargeReq req) {
        return dbStore.createCharge(req);
    }
}

@Repository
public class SourceRepositoryImpl implements SourceRepository {
    private DbStore dbStore;
    public SourceRepositoryImpl(DbStore dbStore) { this.dbStore = dbStore; }

    @Override
    public UpdateSourceReq.Response update(UpdateSourceReq req) {
        return dbStore.updateSource(req);
    }
}
```

#### ③ 서비스 클래스 구현

서비스 클래스는 gRPC가 생성한 `SourceServiceImplBase` 또는 `ChargeServiceImplBase`를 상속하여 구현함.

메서드 파라미터로 전달되는 **StreamObserver** 의 3가지 메서드:

| 메서드 | 설명 | 호출 횟수 |
|--------|------|----------|
| `onNext()` | 스트림에서 값을 전송. `onCompleted()` / `onError()` 이후 호출 불가. | 여러 번 가능 |
| `onCompleted()` | 스트림 완료 표시. 완료 이후 메서드 호출 불가. | 한 번만 |
| `onError()` | 스트림에서 종료 에러를 수신. | 한 번만 |

> `StreamObserver` 아규먼트는 스레드-세이프하지 않으므로 멀티스레딩 처리 시 `synchronized` 호출을 사용해야 함.

```java
@Service
public class SourceService extends SourceServiceImplBase {
    private SourceRepository repository;
    public SourceService(SourceRepository repository) { this.repository = repository; }

    @Override
    public void create(CreateSourceReq req,
            StreamObserver<CreateSourceReq.Response> resObserver) {
        CreateSourceReq.Response resp = repository.create(req);
        resObserver.onNext(resp);       // 응답 전송
        resObserver.onCompleted();      // 스트림 완료 표시
    }

    @Override
    public void retrieve(SourceId sourceId,
            StreamObserver<SourceId.Response> resObserver) {
        try {
            SourceId.Response resp = repository.retrieve(sourceId.getId());
            resObserver.onNext(resp);
            resObserver.onCompleted();
        } catch (Exception e) {         // 예외 발생 시 StreamObserver에 에러 전파
            ExceptionUtils.observeError(resObserver, e, SourceId.Response.getDefaultInstance());
        }
    }
}
```

### gRPC 서버 클래스 구현

내부적으로 Netty 웹 서버를 사용하므로, 스프링 부트의 자체 웹 서버를 비활성화함.

```properties
# application.properties
spring.main.web-application-type=none   # 자체 웹 서버 비활성화
grpc.port=8080
```

```java
@Component
public class GrpcServer {
    @Value("${grpc.port:8080}")
    private int port;
    private Server server;

    public void start() throws IOException, InterruptedException {
        server = ServerBuilder.forPort(port)
            .addService(sourceService)
            .addService(chargeService)
            .intercept(exceptionInterceptor)    // 인터셉터 등록 (에러 처리)
            .build().start();
        Runtime.getRuntime().addShutdownHook(   // 종료 훅 등록
            new Thread(() -> GrpcServer.this.stop()));
    }
    private void stop() { if (server != null) { server.shutdown(); } }
    public void block() throws InterruptedException {
        if (server != null) { server.awaitTermination(); }  // 애플리케이션 종료시까지 요청 수신
    }
}

// @Profile("!test"): 테스트 프로파일에서는 로드되지 않아 서버가 실행되지 않음
@Profile("!test")
@Component
public class GrpcServerRunner implements CommandLineRunner {
    private GrpcServer grpcServer;
    @Override
    public void run(String... args) throws Exception {
        grpcServer.start();
        grpcServer.block();
    }
}
```

### gRPC 서버 테스트

gRPC 테스트 라이브러리가 제공하는 핵심 클래스:

- **GrpcCleanupRule**: 등록된 서버와 채널의 종료를 정상적으로 관리함. JUnit `@Rule` 애노테이션 적용 필요.
- **InProcessServerBuilder**: 서버를 빌드하는 빌더 클래스.
- **InProcessChannelBuilder**: 채널을 빌드하는 빌더 클래스.

```java
@ActiveProfiles("test")     // GrpcServerRunner 비활성화
@SpringBootTest
@TestMethodOrder(OrderAnnotation.class)
class ServerAppTests {
    @Rule
    public final GrpcCleanupRule grpcCleanup = new GrpcCleanupRule();
    private static SourceServiceGrpc.SourceServiceBlockingStub blockingStub;
    @Autowired
    private static String newlyCreatedSourceId = null;

    @BeforeAll
    public static void setup(@Autowired SourceService srcSrvc,
            @Autowired ExceptionInterceptor exceptionInterceptor) throws IOException {
        String sName = InProcessServerBuilder.generateName();           // 1. 고유 서버명 생성
        grpcCleanup.register(InProcessServerBuilder
            .forName(sName).directExecutor().addService(srcSrvc)
            .intercept(exceptionInterceptor).build().start());          // 2. 서버 등록 및 시작
        blockingStub = SourceServiceGrpc.newBlockingStub(
            grpcCleanup.register(InProcessChannelBuilder
                .forName(sName).directExecutor().build()));             // 3. 블로킹 스텁 생성
    }

    @Test @Order(2) @DisplayName("RPC create를 호출해서 source 객체 생성")
    public void SourceService_Create() {
        CreateSourceReq.Response response =
            blockingStub.create(CreateSourceReq.newBuilder().setAmount(100).setCurrency("USD").build());
        assertNotNull(response.getSource());
        newlyCreatedSourceId = response.getSource().getId();
        assertEquals(100, response.getSource().getAmount());
    }

    @Test @Order(3) @DisplayName("유효하지 않은 source id로 RPC 호출 시 예외 발생")
    public void SourceService_RetrieveForInvalidId() {
        Throwable throwable = assertThrows(StatusRuntimeException.class,
            () -> blockingStub.retrieve(SourceId.newBuilder().setId("").build()));
        assertEquals("INVALID_ARGUMENT: Invalid Source ID is passed.", throwable.getMessage());
    }

    @Test @Order(4) @DisplayName("RPC 호출로 source 객체 조회")
    public void SourceService_Retrieve() {
        SourceId.Response response =
            blockingStub.retrieve(SourceId.newBuilder().setId(newlyCreatedSourceId).build());
        assertNotNull(response.getSource());
        assertEquals(100, response.getSource().getAmount());
    }
}
```

### 에러 처리 구현

**io.grpc.ServerInterceptor** 를 구현한 `ExceptionInterceptor`로 횡단(cross-cutting) 예외 처리를 담당함.

| 클래스 | 역할 |
|--------|------|
| `ExceptionInterceptor` | `ServerInterceptor` 구현. 호출을 `ExceptionHandlingServerCallListener`로 전달. |
| `ExceptionHandlingServerCallListener` | `ExceptionInterceptor`의 내부 프라이빗 클래스. `onHalfClose()`, `onReady()`를 재정의하여 예외를 포착하고 `handleException()`을 호출. |
| `ExceptionUtils` | `observeError()`: StreamObserver에 에러 전파. `traceException()`: Throwable을 `StatusRuntimeException`으로 변환. |

```java
@Component
public class ExceptionInterceptor implements ServerInterceptor {
    @Override
    public <RQT, RST> ServerCall.Listener<RQT> interceptCall(
            ServerCall<RQT, RST> serverCall, Metadata metadata,
            ServerCallHandler<RQT, RST> serverCallHandler) {
        ServerCall.Listener<RQT> listener = serverCallHandler.startCall(serverCall, metadata);
        return new ExceptionHandlingServerCallListener<>(listener, serverCall, metadata);
    }

    private class ExceptionHandlingServerCallListener<RQT, RST> extends
            ForwardingServerCallListener.SimpleForwardingServerCallListener<RQT> {
        @Override
        public void onHalfClose() {
            try { super.onHalfClose(); }
            catch (RuntimeException e) { handleException(e, serverCall, metadata); throw e; }
        }
        @Override
        public void onReady() {
            try { super.onReady(); }
            catch (RuntimeException e) { handleException(e, serverCall, metadata); throw e; }
        }
        private void handleException(RuntimeException e, ...) {
            StatusRuntimeException status = ExceptionUtils.traceException(e);
            serverCall.close(status.getStatus(), metadata);
        }
    }
}

@Component
public class ExceptionUtils {
    public static <T extends GeneratedMessageV3> void observeError(
            StreamObserver<T> responseObserver, Exception e, T defaultInstance) {
        responseObserver.onError(traceException(e, defaultInstance));
    }

    public static <T extends GeneratedMessageV3> StatusRuntimeException traceException(
            Throwable e, T defaultInstance) {
        if (e instanceof StatusRuntimeException) return (StatusRuntimeException) e;
        // SocketException → UNAVAILABLE, 그 외 → INTERNAL SERVER ERROR
        com.google.rpc.Status status = com.google.rpc.Status.newBuilder()
            .setCode(com.google.rpc.Code.INTERNAL_VALUE)
            .setMessage("Internal server error")
            .addDetails(Any.pack(defaultInstance)).build();
        return StatusProto.toStatusRuntimeException(status);
    }
}
```

## gRPC 클라이언트 개발

### client/build.gradle 설정

```groovy
// settings.gradle
rootProject.name = 'chapter11-client'

// build.gradle
def grpcVersion = '1.54.1'
dependencies {
    implementation 'com.packt.modern.api:payment-gateway-api:0.0.1'
    implementation "io.grpc:grpc-stub:${grpcVersion}"                   // 스텁 관련 API
    implementation "com.google.protobuf:protobuf-java-util:3.22.2"     // Protobuf ↔ JSON 변환
    implementation 'org.springframework.boot:spring-boot-starter-web'
}
repositories { mavenCentral(); mavenLocal() }
```

### gRPC 클라이언트 구현

클라이언트의 애플리케이션 포트는 gRPC 서버 포트와 달라야 함.

```properties
# application.properties
server.port=8081           # 클라이언트 REST 포트
grpc.server.host=localhost
grpc.server.port=8080      # gRPC 서버 포트
```

```java
@Component
public class GrpcClient {
    @Value("${grpc.server.host:localhost}") private String host;
    @Value("${grpc.server.port:8080}") private int port;
    private ManagedChannel channel;
    private SourceServiceBlockingStub sourceServiceStub;
    private ChargeServiceBlockingStub chargeServiceStub;

    public void start() {
        // ManagedChannel: gRPC 호출을 위해 개념적인 엔드포인트에 가상 연결 제공
        channel = ManagedChannelBuilder.forAddress(host, port).usePlaintext().build();
        sourceServiceStub = SourceServiceGrpc.newBlockingStub(channel);
        chargeServiceStub = ChargeServiceGrpc.newBlockingStub(channel);
    }
    public void shutdown() throws InterruptedException {
        channel.shutdown().awaitTermination(1, TimeUnit.SECONDS);
    }
}

// @Profile("!test"): 테스트 프로파일에서는 실행되지 않음
@Profile("!test")
@Component
public class GrpcClientRunner implements CommandLineRunner {
    @Autowired GrpcClient client;
    @Override
    public void run(String... args) {
        client.start();
        Runtime.getRuntime().addShutdownHook(new Thread(() -> {
            try { client.shutdown(); }
            catch (InterruptedException e) { System.out.println("에러: {}", e.getMessage()); }
        }));
    }
}

// REST 엔드포인트: gRPC 스텁을 사용해 Charge 목록을 JSON으로 반환
@RestController
public class ChargeController {
    private GrpcClient client;
    public ChargeController(GrpcClient client) { this.client = client; }

    @GetMapping("/charges")
    public String getSources(@RequestParam(defaultValue = "ab1ab2ab3ab4ab5") String customerId)
            throws InvalidProtocolBufferException {
        var req = CustomerId.newBuilder().setId(customerId).build();
        CustomerID.Response resp = client.getChargeServiceStub().retrieveAll(req);
        // JsonFormat.printer(): Protobuf 응답을 JSON 문자열로 변환 (기본값 포함)
        return JsonFormat.printer().includingDefaultValueFields().print(resp);
    }
}
```

## gRPC 서비스 테스트

```bash
# 1. api 라이브러리 로컬 저장소 게시 (api 루트 디렉터리)
$ export JAVA_TOOL_OPTIONS="-Dfile.encoding=UTF8"
$ ./gradlew clean publishToMavenLocal

# 2. 서버 시작 (server 루트 디렉터리)
$ ./gradlew clean build
$ java -jar build/libs/chapter11-server-0.0.1-SNAPSHOT.jar
# → gRPC server started and listening on port: 8080

# 3. 클라이언트 시작 (client 루트 디렉터리)
$ ./gradlew clean build
$ java -jar build/libs/chapter11-client-0.0.1-SNAPSHOT.jar
# → Tomcat initialized with port(s): 8081 / gRPC client connected to localhost:8080

# 4. /charges 엔드포인트 호출
$ curl http://localhost:8081/charges
```

```json
{
  "charge": [{
    "id": "cle9e9oam6gajkkeivjof5pploq89ncp",
    "amount": 1000,
    "currency": "USD",
    "customerId": "ab1ab2ab3ab4ab5",
    "description": "Charge Description",
    "receiptEmail": "receipt@email.com",
    "statementDescriptor": "Statement Descriptor",
    "status": "SUCCEEDED",
    "sourceId": "0ovjn4l6crgp9apr79bhpefme4dok3qf"
  }]
}
```

## 마이크로서비스란?

- **마이크로서비스**: 네트워크를 통해 통신하는 독립형 경량 프로세스
- API를 통해 서비스를 사용하는 소비자에게 특화된 API(REST, gRPC, 이벤트)를 제공함.
- 이 장에서 개발한 gRPC 서버는 마이크로서비스라고 할 수 있음.

아키텍처 설계 방식의 발전 과정:

| 아키텍처 | 특징 | 한계 |
|---------|------|------|
| **전통적인 모놀리식** | 프레젠테이션, 비즈니스 로직, DAO 등 모든 것이 단일 아카이브(EAR/WAR)에 포함됨. | 배포 단위가 크고, 일부 변경에도 전체 재배포가 필요함. |
| **서비스 기반 모놀리식 (SOA)** | 각 컴포넌트가 서비스를 제공하는 방식. EAR 형태로 함께 번들로 제공됨. | 전반적으로 모놀리식이며 데이터베이스를 서로 공유함. |
| **마이크로서비스** | 각 컴포넌트가 독립적으로 개발, 구축, 테스트, 배포됨. API 게이트웨이로 클라이언트 요청을 각 서비스에 라우팅함. | 서비스 간 통신과 운영 복잡도가 증가함. |

샘플 전자 상거래 앱을 마이크로서비스로 분리한 예시:

- 고객 / 주문 / 청구 / 배송 / 인보이스 발행 / 인벤토리 / 결제대금 청구

각 서비스를 개별적으로 개발하고 프로세스 간(서비스 간) 통신을 사용해서 솔루션을 결합할 수 있음.
