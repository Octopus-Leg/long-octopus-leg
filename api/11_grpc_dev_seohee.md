# 11. gRPC API 개발 및 테스트

## 1️⃣ API 작성
### 프로젝트 설정
#### gRPC 서버, 클라이언트 프로젝트 생성
- **Spring Initializr**를 사용하여 기본 프로젝트 구조를 생성함.
- **Spring Web** 의존성을 추가하고, Java 17, Spring Boot 3.0 이상 버전 및 Gradle 환경으로 설정함.
- 서버와 클라이언트 디렉터리에에 해당 프로젝트를 복사해 서버, 클라이언트로 프로젝트를 명명함.
#### gRPC API 라이브러리 프로젝트 생성
- `api` 디렉터리를 생성함.
- `gradle init` 명령어를 통해 **library** 타입의 자바 프로젝트를 구성함.
#### gRPC API 라이브러리 프로젝트 구성
- **Protobuf** 및 **Maven Publish** 플러그인을 설정함.Maven Publish 플러그인은 로컬 메이븐 저장소로 게시하는 데 사용됨.
- `api/lib/build.gradle` 파일에서 `java-library` 플러그인을 **java**로 변경함.
- gRPC 통신 및 코드 생성을 위해 `grpc-stub`, `grpc-protobuf`, `grpc-netty`, `javax.annotation-api` 등의 의존성을 추가함.
- 커맨드라인 컴파일러 **'protoc'** 와 자바 플러그인(`protoc-gen-grpc-java`)의 아티팩트를 설정하여 `.proto` 파일을 기반으로 자바 코드를 생성하도록 **Protobuf 그래들 플러그인**을 구성함.
- 생성된 소스 파일을 IDE에서 인식할 수 있도록 `sourceSets`에 `src/main/grpc` 디렉터리를 추가함.
- 마지막으로 **'publishing'** 블록을 추가하여 생성된 `Jar` 아티팩트가 메이븐 저장소에 올바르게 게시되도록 설정함.
- api 프로젝트는 서비스의 인터페이스 역할을 하며 서버와 클라이언트 양쪽에 배포될 `'Jar'` 파일을 생성하는 목적을 가짐.

### 결제 게이트웨이 기능 작성
- 고객의 결제 요청을 수신하여 판매자와 은행 사이에서 중개하는 시스템을 설계함.
- 주요 서비스로 결제 실행을 담당하는 `'ChargeService'`와 결제 수단을 관리하는 `'SourceService'`를 정의함.
#### 온라인 결제 워크플로우 단계
1. 고객이 결제에 사용할 **출처(결제 수단)** 를 먼저 생성함.
2. 생성된 출처를 기반으로 **청구** 를 생성해 결제를 시작함.
3. 결제 게이트웨이는 유효성 검사 후 최종적으로 결제가 청구되도록 함.
#### 결제 게이트웨이 서비스 정의 작성
- `PaymentGatewayService.proto` 파일에 `syntax = "proto3"`를 명시하고 서비스 구조를 정의함.
- **ChargeService**: `Create`, `Retrieve`, `Update`, `Capture`, `RetrieveAll`와 같은 Charge 객체와 연관된 메소드를 포함함.
- **SourceService**: `Create`, `Attach`, `Detach` 등 결제 수단 연결 및 관리 메소드를 포함함.
- 데이터의 유연한 확장을 위해 요청과 응답 시 항상 별도의 래포 요청 및 응답 타입을 정의하여 사용하는 것이 좋음.
- **Charge**, **Source** 등 핵심 도메인과 통신에 필요한 매개변수 메시지를 정의함.
#### 결제 게이트웨이 서비스 gRPC 서버, 스텁, 모델 개시
- 파일 인코딩을 UTF-8으로 설정함.
- `gradlew clean publishToMavenLocal` 명령을 통해 기존 파일을 제거하고, **Protobuf 그래들 플러그인**이 `.proto` 파일을 기반으로 자바 코드를 자동 생성함.
    - **generateProto 그래들 태스크**로 생성된 파일
        - **Models**: Card, Address, Charge, Source 등 데이터 구조를 정의한 메시지 클래스들임. CreateChargeReq, Charge와 같은 클라이언트와 서버 간의 데이터 교환에 사용되는 요청과 응답 객체들을 포함함.
        - **gRPC classes**: 서비스 정의(ChargeServiceGrpc 등)를 바탕으로 생성된 추상 기본 클래스(ImplBase), 그리고 클라이언트가 서버를 호출할 때 사용하는 스텁 클래스들을 포함함.
- 이를 빌드한 후, 아티팩트를 로컬 메이븐 저장소에 게시함.

## 2️⃣ gRPC 서버 개발
- 서버 프로젝트의 디렉터리 구조를 설정하고 `build.gradle`에 gRPC 관련 의존성을 추가함.
- 로컬 메이븐 저장소(`mavenLocal()`)를 참조하여 앞서 생성한 **API 라이브러리**를 프로젝트에 주입함.

### gRPC 서버 구현
- 서버는 **지속성 저장소 > 리포지토리 > 서비스 > API 엔드포인트**의 계층형 아키텍처를 따름.
- 인메모리 데이터베이스인 `DbStore`를 생성하여 `ConcurrentHashMap`으로 데이터를 관리함.
- 테스트를 위해 미리 정의된 **Source**와 **Charge** 시드 객체를 두 개의 해시 맵에 각각 저장함.
- 서비스 계약에 정의된 작업에 따라 데이터베이스 저장소에 메소드를 만들며, 비즈니스 로직을 간결하게 구현함.
#### 리포지토리 클래스 작성
- `ChargeRepositoryImpl`과 `SourceRepositoryImpl` 클래스를 생성하여 데이터 접근 계층을 구현함.
- 각 리포지토리는 `@Repository` 어노테이션을 사용하여 스프링 빈으로 등록하고 `DbStore`를 주입받음.
- Source 및 Charge 저장소 클래스의 메소드는 서비스 클래스에서 사용됨.
#### 서비스 클래스 구현
- **Protobuf 플러그인**에 의해 자동 생성된 **추상 기본 클래스**(`SourceServiceImplBase` 등)를 상속받아 실제 서비스 로직을 구현함.
- `StreamObserver` 객체를 사용하여 **비동기 응답 흐름**을 제어함.
- StreamObserver의 3가지 기본 메소드:
    - `onNext()`: 처리된 결과 데이터를 클라이언트에 전달함. 여러 데이터가 클라이언트에게 전송될 때는 스트림에 대해 `onNext()`를 여러 번 호출해야 함.
    - `onError()`: 실행 중 오류가 발생했을 때 예외 정보를 보내고 통신을 강제 종료함.
    - `onCompleted()`: 모든 데이터 전달이 성공적으로 끝났음을 알리고 RPC 통신을 정상 종료함.
- `StreamObserver`는 **스레드-세이프**하지 않으므로, 멀티스레드 환경에서 동일한 관찰자에게 응답을 보낼 때는 `synchronized` 처리가 필요함.

### gRPC 서버 클래스 구현
- `application.properties`에서 `spring.main.web-application-type=none`으로 설정하여 스프링 부트의 기본 웹 서버가 실행되지 않도록 함.
- gRPC 포트는 `grpc.port=8080`으로 지정함.
- **Netty** 기반의 gRPC 서버를 실행하기 위해 `GrpcServer` 클래스를 작성함.
- 서버 빌더(`ServerBuilder`)를 사용하여 구현한 서비스들과 에러 처리를 위한 인터셉터를 등록함.
- `start()`, `stop()`, `block()` 메소드를 통해 서버의 생명주기를 관리하며, 앱 종료 시 서버가 안전하게 닫히도록 훅을 추가함.
- `'CommandLineRunner'`를 구현한 `'GrpcServerRunner'`를 작성하여 `run()` 메소드를 재정의함. 이를 통해 애플리케이션 시작과 동시에 gRPC 서버가 리스닝을 시작함.
- `@Profile("!test")` 어노테이션을 사용해서 **테스트 환경이 아닐 때만** 서버를 자동으로 실행하도록 제한함.

### gRPC 서버 테스트
- `@ActiveProfiles("test")`를 설정하여 **테스트 환경**임을 명시해 `GrpcServerRunner`를 비활성화함.
- **gRPC 테스트 라이브러리**의 `'GrpcCleanupRule'`을 사용하여 테스트 종료 시 서버와 채널을 자동으로 정리함. (JUnit `@Rule` 어노테이션을 적용해야 함.)
- `'InProcessServerBuilder'`와 `'InProcessChannelBuilder'`를 사용하여 서버와 채널을 빌드하고 독립적인 테스트 환경을 구축함.
- `setup()` 메소드에서 서버를 빌드 및 시작하고, 서버와 통신할 채널 및 블로킹 스텁을 생성함.
- **유효성 에러 테스트**를 통해 정상적인 서비스 호출 결과뿐만 아니라, 잘못된 ID 입력 시 발생하는 예외 상황이 올바르게 반환되는지 검증함.

## 3️⃣ 예외 처리 구현
- **io.grpc.ServerInterceptor**를 사용하여 인증, 로깅, 에러 처리와 같은 **횡단 관심사**를 처리하며, 이는 **스레드 세이프**한 인터페이스임.
- 예외를 가로채기 위해 `'ServerInterceptor'`를 구현한 **ExceptionInterceptor**를 작성함.
- 인터셉터는 서버 호출을 `ExceptionHandlingServerCallListener`로 전달하여 응답 과정의 예외를 감시함.
- **onHalfClose()** 와 **onReady()** 이벤트를 재정의하여 예외 발생 시 `'handleException()'`을 호출하도록 구성함.
- `'handleException()'`은 실제 예외 처리를 위해 **ExceptionUtils** 메소드를 사용함.

### ExceptionUtils 클래스
- **observerError()**: `'StreamObserver'`의 `onError()` 이벤트를 발생시켜 클라이언트에 에러를 전송함.
- **traceException()**: 발생한 `Throwable`로부터 에러를 추적하여 gRPC 표준인 **StatusRuntimeException**을 리턴함.
- `'com.google.rpc.Status'`를 사용하여 에러 상태코드, 메시지, 상세 정보(`addDetails`)를 구조화함.
- **StatusProto.toStatusRuntimeException(status)** 를 사용하여 구조화된 데이터를 **StatusRuntimeException**으로 변환함.

## 4️⃣ gRPC 클라이언트 개발
- `build.gradle`에 **grpc-stub**과 **protobuf-java-util** 등 클라이언트 구현에 필요한 의존성을 추가함.
- 로컬 메이븐 저장소(`mavenLocal()`)를 참조하여 앞서 생성한 **API 라이브러리**를 프로젝트에 주입함.
### gRPC 클라이언트 구현
- `application.properties`에서 클라이언트 포트(`8081`), 서버 호스트(`localhost`)와 gRPC 서버 포트(`8080`) 정보를 설정함.
- `'GrpcClient'` 클래스를 작성하며, **ManagedChannel**을 생성하여 gRPC 서버와의 통신 통로를 구축함.
- 서비스 호출을 위해 자동 생성된 **BlockingStub**(`SourceServiceBlockingStub`, `ChargeServiceBlockingStub`)을 사용함.
- `start()` 메소드에서 채널을 빌드하고 스텁을 초기화하며, `shutdown()`으로 채널 자원을 안전하게 해제함.
- `CommandLineRunner`를 상속받은 `GrpcClientRunner`를 통해 앱 시작 시 **스텁 인스턴스**를 생성하고 서버 연결을 확인하는 메시지를 호출함.
### REST 컨트롤러 추가
- 외부에서 gRPC 서버 기능을 호출해 볼 수 있도록 클라이언트 앱에 **REST API** 엔드포인트(`/charges`)를 추가함.
- 컨트롤러는 주입받은 `GrpcClient`의 **ChargeServiceStub**을 사용해서 charge gRPC 서버의 `retrieveAll()` 메소드를 호출함.
- 응답받은 Protobuf 메시지 객체는 **JsonFormat** 클래스를 사용하여 JSON 문자열로 변환한 뒤 반환함.
### gRPC 서비스 테스트
- 로컬 메이븐 저장소에 api 프로젝트가 게시되었는지 확인 후 `publishToMavenLocal`을 통해 라이브러리를 최신화함.
- **gRPC 서버**를 먼저 실행하고, **클라이언트**를 실행함.
- `curl` 명령어를 통해 클라이언트의 REST 엔드포인트에 접속하여 결제 정보가 정상적으로 출력되는지 확인함.

## 5️⃣ 마이크로서비스란?
- 네트워크를 통해 통신하는 **독립형 경량 프로세스**임.
- 서비스 사용자에게 특화된 API(REST, gRPC, 이벤트 등)를 제공함.
- 가시성이 좋고 사용 빈도가 늘어남에 따라 현대 애플리케이션 개발의 표준이 됨.
### 전통적인 모놀리식 디자인
- 데이터베이스 파일이나 소스가 상호 작용하는 방식에 관계없이 모든 구성 요소(프레젠테이션, 어플리케이션 로직 비즈니스 로직, 데이터 액세스)가 **동일한 아카이브**에 포함됨.
### 서비스 기반 모놀리식 디자인
- SOA(서비스 지향 아키텍처) 이후 등장한 방식으로, 애플리케이션이 서비스 기반으로 개발되기 시작함.
- 일부의 SOA 서비스가 별도로 배포되기도 하지만 전반적으로는 **여전히 모놀리식** 구조를 유지함.
- 모든 서비스와 프레젠테이션 코드가 함께 번들링되어 배포되며, 데이터베이스를 서로 공유함.
### 마이크로서비스 디자인
- 각 컴포넌트가 **자율적**이며 독립적으로 개발, 구축, 테스트, 배포될 수 있음.
- **API 게이트웨이**를 통해 서로 다른 클라이언트가 개별 서비스에 액세스할 수 있는 인터페이스를 제공함. 각 서비스 간의 통신은 노출된 API를 기반으로 이루어짐.
- 각 서비스는 별도의 프로세스로 실행되며, 도메인과 바운디드 컨텍스트를 기반으로 분할됨.
- 전자 상거래 앱의 경우 고객, 주문, 청구, 배송 등 각 도메인을 별도의 마이크로서비스로 개발하고 프로세스 간 통신을 사용해서 결합함.
