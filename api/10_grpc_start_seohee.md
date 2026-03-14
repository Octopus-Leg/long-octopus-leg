# 10. gRPC 시작하기

## 1️⃣ gRPC 동작방식
- gRPC는 구글에서 개발한 범용 **RPC**(원격 프로시저 호출)용 오픈 소스 프레임워크로 네트워크를 통해 다른 시스템의 함수를 로컬 함수처럼 호출할 수 있게 해줌.
- gRPC의 `g`는 릴리스마다 변경되었고 확장성이 뛰어난 분산 시스템과 낮은 대기 시간이 필요한 환경에 적합함.
- gRPC는 `Protocol Buffers`, `JSON`, `XML`, `Thrift`와 같은 다양한 미디어 형식을 지원하고 특히 프로토콜 버퍼는 성능이 좋아 다른 미디어 형식들보다 선호됨.

### REST 대 gRPC
| 구분 | REST                  | gRPC                                       |
| :--- |:----------------------|:-------------------------------------------|
| **통신 프로토콜** | HTTP 1.1 / HTTP 2     | **HTTP/2** 전용                              |
| **데이터 형식** | 주로 **JSON**           | 주로 **Protocol Buffers**                    |
| **통신 방식** | 단방향 요청/응답 위주          | 완전한 **이중 스트리밍** 지원(음성이나 화상 통화 같은 시나리오에 유리) |
| **성능 및 지연** | 경로/매개변수 파싱으로 지연 발생 가능 | 정적 경로 및 단일 페이로드로 **성능 우수**                 |
| **에러 처리** | HTTP 상태 코드에 의존        | **정형화된 에러 상태 코드** 제공                       |
| **유연성** | 구현이 매우 유연하고 대중적임      | 엄격한 계약 기반으로 동작                             |
| **기타 기능** | 기본 기능 위주              | **호출 취소, 부하 분산, 장애 조치** 기본 지원              |

### 웹 브라우저와 모바일 앱에서 gRPC 서버를 호출할 수 있을까?
- gRPC는 분산 시스템 통신을 위해 설계되었으므로 모바일 앱이나 웹 브라우저에서도 호출이 가능함.
- 특히 웹 브라우저를 위한 **웹용 gRPC(gRPC-web)** 는 2018년에 등장하여 IoT 응용 프로그램 등에서 많이 사용되고 있음.
- 이상적인 구현을 위해서는 서비스 내부 통신에 gRPC를 먼저 채택한 뒤 외부 웹/모바일 통신으로 확장하는 방식이 권장됨.

### gRPC 아키텍처란
- gRPC 아키텍처는 크게 세 가지 계층인 **스텁**, **채널**, **전송**으로 구성됨.
- **스텁**은 최상위 레이어로 서비스 인터페이스와 메시지가 정의된 `인터페이스 정의 언어(IDL)` 파일에 의해 생성되며 클라이언트는 이를 호출하여 서버와 통신함.
    - `IDL`에서 인터페이스가 Protobuf를 사용해서 정의된 경우는 `.proto` 확장자를 가짐.
- **채널**은 `응용 프로그램 바이너리 인터페이스(ABI)`를 제공해 스텁과 서버 사이를 연결하는 중간 계층으로 특정 호스트와 포트에 대한 연결을 제공함.
- **전송**은 가장 낮은 계층으로 **HTTP/2** 프로토콜을 사용하여 데이터의 이중 통신과 다중 병렬 호출을 제공함.

#### gRPC 기반 서비스를 개발하는 방법
1. `.proto` 파일을 사용해서 서비스 인터페이스를 정의함.
2. 1단계에서 정의한 서비스 인터페이스의 구현을 작성함.
3. gRPC 서버를 생성하고 서비스를 등록함.
4. 서비스 스텁을 생성하고 gRPC 클라이언트와 함께 사용함.

### gRPC가 Protocol Buffer를 사용하는 방법
- Protocol Buffer(Protobuf)를 사용하면 공식 계약(contract), 더 나은 대역폭 최적화, 코드 생성이 가능함.
- Protobuf는 gRPC의 기본 데이터 직렬화뿐만 아니라 코드 생성에서도 사용되고, 이진 데이터로 변환되어 전송되므로 JSON보다 크기가 작고 빠름.
- Protobuf 메시지는 **키-값 쌍**의 일련번호로 구성되며 키에는 메시지 필드와 해당 타입을 지정함.
  ```protobuf
  message Employee {
    int64 id = 1;
    string firstName = 2;
  }
  ```
- 각 필드에는 고유한 **일련번호(태그)** 가 부여되어 직렬화와 파싱에 사용됨.
- `int32`, `string`, `bool` 같은 기본 타입 외에도 `enum`을 통한 열거형이나 `map` 타입을 사용하여 복잡한 데이터 구조를 정의할 수 있음.
  ```protobuf
  message Employee {
    // ...생략
    // enum 키워드를 사용한 열거형 타입 정의
    enum Grade {
      I_GRADE = 1;
      II_GRADE = 2;
      III_GRADE = 3;
      IV_GRADE = 4;
    }
    
    // map<key_type, value_type> 키워드를 사용한 맵 타입 정의
    // string 타입의 키와 int32 타입의 값을 가짐
    map<string, int32> nominees = 1;
    // ...생략
  }
  ```

#### Protobuf 타입과 자바 타입 매핑
| Protobuf 타입 | 자바 타입 | 설명 |
| :--- | :--- | :--- |
| `double` | `double` | 자바의 `double`과 유사함. |
| `float` | `float` | 자바의 `float`과 유사함. |
| `int32` | `int` | 가변 길이 인코딩을 사용함. 음수 인코딩에는 비효율적이므로 음수 포함 시 `sint32` 권장. |
| `int64` | `long` | 가변 길이 인코딩을 사용하며 음수 포함 시 `sint64` 권장. |
| `uint32` | `int` | 부호 없는 가변 길이 인코딩. 값이 $2^{28}$보다 크면 `fixed32` 사용 권장. |
| `uint64` | `long` | 부호 없는 가변 길이 인코딩. 값이 $2^{56}$보다 크면 `fixed64` 사용 권장. |
| `sint32` | `int` | 부호 있는 `int` 값을 포함할 때 음수 인코딩에 더 효율적임. |
| `sint64` | `long` | 부호 있는 `long` 값을 포함할 때 음수 인코딩에 더 효율적임. |
| `fixed32` | `int` | 항상 **4 bytes**를 차지함. |
| `fixed64` | `long` | 항상 **8 bytes**를 차지함. |
| `bool` | `boolean` | 참 또는 거짓 값을 가짐. |
| `string` | `String` | $2^{32}$보다 짧은 **UTF-8** 인코딩 문자열 또는 7-bit 아스키 문자를 포함함. |
| `bytes` | `ByteString` | $2^{32}$보다 짧은 임의의 바이트 시퀀스를 포함함. |

#### 직원 샘플 서비스 인터페이스
```protobuf
syntax = "proto3";
package com.packtpub;
option java_package = "com.packt.modern.api.proto";
option java_multiple_files = true;
// 직원 정보 메시지
message Employee {
  int64 id = 1;
  string firstName = 2;
  string lastName = 3;
  message Address {
    string houseNo = 1;
    string street1 = 2;
    string street2 = 3;
    string city = 4;
  }
}
// 응답 메시지
message EmployeeCreateResponse {
  int64 id = 1;
}
// 서비스 인터페이스
service EmployeeService {
  rpc Create(Employee) returns (EmployeeCreateResponse);
}
```
- **기본 설정**: `syntax = "proto3";`로 버전을 지정하며, `package`와 `option` 키워드로 패키지 이름과 자바 파일 생성 방식(별도 파일 생성 등)을 결정함.
- **메시지 정의**: `message` 키워드로 데이터 구조를 정의함. `Employee` 메시지 안에 이름, ID 등과 함께 `Address` 같은 중첩 메시지를 포함할 수 있음.
- **서비스 정의**: `service` 키워드로 실제 호출할 메서드를 정의하며, `rpc` 키워드를 사용하여 메서드 이름, 매개변수, 리턴 타입을 지정함.

## 2️⃣ 서비스 정의의 이해
- gRPC 서비스는 각 매개변수와 리턴 타입으로 메소드를 지정하여 정의하며 원격으로 호출할 수 있는 서버에 의해 노출됨.
  ```protobuf
  service EmployeeService {
    rpc Create(Employee) returns (EmployeeCreateResponse);
  }
  ```
  - `Create`는 `EmployeeService`에 의해 외부로 노출되는 메소드임.
  - 클라이언트가 `Employee`라는 단일 요청 객체를 보내면 서버로부터 `EmployeeCreateResponse`라는 단일 응답 객체를 받기 때문에 **단항 서비스 메소드**에 해당함.
  - 서비스 정의에 사용되는 모든 요청 및 응답 메시지는 반드시 서비스 정의의 일부로 함께 정의되어야 함.

### gRPC 서비스 메소드 타입
- **단항**: 클라이언트가 단일 요청을 보내고 서버가 단일 응답을 반환하는 가장 기본적인 방식임.
- **서버 스트리밍**: 클라이언트가 단일 객체를 보내면 서버는 일련의 메시지가 포함된 스트림을 응답함.
  - 스트림은 클라이언트가 모든 메시지를 수신할 때까지 열린 상태로 유지되며 메시지 시퀀스는 gRPC에 의해 보장됨.
  - `rpc LiveMatchScore(MatchId) returns (stream MatchScore);` ➡️ 클라이언트는 경기가 끝날 때까지 라이브 스코어 메시지를 계속 수신함.
- **클라이언트 스트리밍**: 클라이언트가 일련의 메시지 스트림을 보내면 서버가 이를 모두 수신한 후 단일 응답 객체를 반환함.
  - 메시지 시퀀스는 gRPC에 의해 보장되며 클라이언트는 모든 메시지를 보낸 후 서버 응답을 기다림.
  - `rpc AnalyzeData(stream DataInput) returns (Report);` ➡️ 모든 데이터 레코드가 전송될 때까지 서버에 데이터 메시지를 보낸 다음 Report 객체를 기다림.
- **양방향 스트리밍**: 서버와 클라이언트가 모두 읽기-쓰기 스트림을 사용하여 일련의 메시지를 보냄.
  - 두 스트림은 독립적으로 작동하여 각자 원하는 순서대로 메시지를 읽고 쓸 수 있음.
  - 서버는 하나씩 또는 한 번에 메시지를 읽고 회신하거나 이를 조합한 방식을 사용할 수 있음.
  - `rpc BatchProcessing(stream InputRecords) returns (stream Response);` ➡️ 처리된 레코드는 즉시 하나씩 보내거나 나중에 배치로 보낼 수 있음.

## 3️⃣ RPC 수명 주기 살펴보기
- 각 서비스 정의 타입에 따라 클라이언트와 서버가 상호작용하는 고유한 수명 주기가 존재함.
- **단항 RPC의 수명 주기**
  - 클라이언트가 스텁 메소드를 호출하면 수명 주기가 시작되며, 스텁은 호출 알림과 함께 클라이언트의 **메타데이터**, 메소드 이름, 지정된 기한을 서버로 전달함.
  - 서버는 초기 메타데이터를 즉시 돌려보내거나 클라이언트의 요청 메시지를 받은 후에 보낼 수 있으며, 반드시 응답 전에는 이를 돌려보내야 함.
  - 서버가 비즈니스 로직을 처리한 뒤 응답 메시지와 함께 **상태 정보(코드와 옵션 메시지)**, 옵션 후행 메타데이터를 보내면 클라이언트는 이를 수신하고 호출을 완료함.

- **서버 스트리밍 RPC의 수명 주기**
  - 단항 RPC와 거의 동일한 단계를 따르지만, 서버가 응답을 단일 메시지가 아닌 **메시지 스트림**으로 전송한다는 점이 다름.
  - 서버는 모든 메시지가 전송될 때까지 메시지를 스트림으로 보내며, 최종적으로 상태 정보와 후행 메타데이터가 포함된 응답을 보내 서버 측 처리를 완료함.
  - 클라이언트는 서버의 모든 메시지를 수신한 후 수명 주기를 완료함.

- **클라이언트 스트리밍 RPC의 수명 주기**
  - 단항 RPC와 거의 동일한 단계를 따르지만, 클라이언트가 단일 요청 대신 **메시지 스트림**을 서버로 전송한다는 점이 다름.
  - 클라이언트는 모든 메시지가 전송될 때까지 메시지를 스트림으로 보내며, 서버는 클라이언트의 메시지를 모두 받은 후에만 단일 응답 메시지, 상태 정보, 후행 메타데이터를 보냄.
  - 클라이언트는 서버로부터 최종 응답을 받으면 수명 주기를 완료함.

- **양방향 스트리밍 RPC의 수명 주기**
  - 처음 두 단계(호출 및 메타데이터 전달)는 단항 RPC와 동일하며, 이후 클라이언트와 서버가 서로 독립적으로 메시지를 읽고 씀.
  - 두 스트림이 완전히 분리되어 작동하므로 서버는 클라이언트의 메시지 스트림을 받는 중에 즉시 응답을 보낼 수도 있고, 모든 메시지를 수신할 때까지 기다렸다가 보낼 수도 있음.
  - 클라이언트는 요청 메시지를 보내고 서버는 이를 처리하는 프로세스를 계속함.
  - 클라이언트는 모든 서버 메시지를 수신하면 수명 주기를 완료함.

### 수명 주기에 영향을 주는 이벤트
- **데드라인/타임아웃**: 클라이언트는 정의된 데드라인/타임아웃까지 응답을 기다리며 이를 초과하면 `DEADLINE_EXCEEDED` 에러가 발생함. 서버 역시 해당 RPC가 타임아웃되었는지 또는 남은 시간이 얼마인지 쿼리하여 확인할 수 있음.
- **RPC 종료**: 클라이언트와 서버는 호출 성공 여부에 대한 결정을 독립적으로 내리기 때문에 양측의 종료 시나리오가 서로 일치하지 않을 수 있음.
- **RPC 취소**: 서버나 클라이언트 모두 언제든지 RPC를 취소할 수 있으나 취소 전 변경된 사항은 **롤백되지 않음**.

### gRPC 서버 및 gRPC 스텁 이해
- gRPC는 클라이언트-서버 아키텍처를 기반으로 하며, `protoc` 컴파일러를 통해 생성된 코드가 통신의 핵심 역할을 수행함.
- **`protoc` 컴파일러에서 생성되는 파일 타입**
  - **모델**: 서비스 정의 파일(`.proto`)에 기술된 모든 메시지 객체가 생성되며, 요청/응답 메시지의 직렬화/역직렬화 및 가져오기를 수행하기 위한 Protobuf 코드를 포함함.
  - **gRPC 자바 파일**: 서비스 **인터페이스**와 클라이언트가 서버와 통신할 때 사용하는 **스텁**이 포함되어 있음.
- **인터페이스 구현**
  ```java
  public class EmployeeService extends EmployeeServiceImplBase {
    @Override
    public void create(Employee request, StreamObserver<Response> responseObserver) {
      // 구현부
    }
  }
  ```
- **gRPC 서버 실행**
  - `ServerBuilder`를 사용해 포트를 지정하고 서비스를 등록한 뒤 `start()`를 호출함.
  ```java
  public class GrpcServer {
    public static void main(String[] args) {
      try {
        Server server = ServerBuilder.forPort(8080)
                .addService(new EmployeeService())
                .build();
        System.out.println("Starting gRPC Server Service...");
        server.start();
  
        System.out.println("Server has started at port:8080");
        System.out.println("Following services are available:");
  
        server.getServices().stream().forEach(
          s -> System.out.println("Service Name: " + s.getServiceDescriptor().getName())
        );
  
        server.awaitTermination();
      } catch (Exception e) {
        // 에러 처리
      }
    }
  }
  ```
- **클라이언트 스텁 생성** 
  - `ManagedChannelBuilder`를 통해 채널을 생성한 후, 이를 사용하여 동기 `BlockingStub` 또는 비동기 `Stub` 스텁을 생성하여 서버와 통신함.
  ```java
  public EmployeeServiceClient(ManagedChannelBuilder<?> channelBuilder) {
    channel = channelBuilder.build();
    blockingStub = EmployeeServiceGrpc.newBlockingStub(channel);
    asyncStub = EmployeeServiceGrpc.newStub(channel);
  }
  ```

## 4️⃣ 에러 처리와 에러 상태 코드
- HTTP 상태 코드를 사용하는 REST와 달리 gRPC는 에러 코드와 옵션 에러 메시지가 포함된 **상태 모델**을 사용함.
- HTTP 에러 코드는 제한된 정보만 포함하기 때문에 gRPC는 더 세부적인 정보를 제공하기 위해 `Error`라는 특수한 Class나 풍부한 에러 모델을 활용함.
- 풍부한 에러 모델은 Protobuf를 통해 정의되며, 클라이언트가 에러를 처리하거나 재시도하는 데 필요한 상세 정보를 포함할 수 있음.

#### gRPC 에러 모델 예시
```protobuf
package google.rpc;
message Status {
  // google.rpc.Code에 정의된 실제 에러 코드
  int32 code = 1;
  // 개발자가 읽을 수 있는 에러 메시지
  string message = 2;
  // 재시도 정보, 도움말 링크 등 추가 에러 세부 정보
  repeated google.protobuf.Any details = 3;
}
```
- **details 필드**: `RetryInfo`, `DebugInfo`, `QuotaFailure`, `BadRequest` 등 관련 정보를 포함하여 클라이언트에게 더 정확한 에러 원인을 전달할 수 있음.

#### REST 에러 코드와 gRPC 상태 코드 매핑
| HTTP 상태 코드 | gRPC 상태 코드 | 설명 |
| :--- | :--- | :--- |
| 400 | `INVALID_ARGUMENT` | 유효하지 않은 아규먼트 |
| 400 | `FAILED_PRECONDITION` | 잘못된 사전 조건으로 액션 실행 불가 |
| 400 | `OUT_OF_RANGE` | 클라이언트가 지정한 범위가 유효하지 않음 |
| 401 | `UNAUTHENTICATED` | 누락되거나 만료된 토큰 또는 인증받지 않은 클라이언트 요청 |
| 403 | `PERMISSION_DENIED` | 클라이언트에게 충분한 권한이 없음 |
| 404 | `NOT_FOUND` | 리소스를 찾을 수 없음 |
| 409 | `ABORTED` | 읽기-쓰기 작업 또는 동시성 충돌 |
| 409 | `ALREADY_EXISTS` | 이미 존재하는 리소스에 대한 생성 요청 |
| 429 | `RESOURCE_EXHAUSTED` | API 리미트에 도달하여 요청 처리 불가 |
| 499 | `CANCELLED` | 요청이 클라이언트에 의해서 취소됨 |
| 500 | `DATA_LOSS` | 복구 불가능한 데이터 손실 발생 |
| 500 | `UNKNOWN` | 알려지지 않은 서버 측 에러 |
| 500 | `INTERNAL` | 내부 서버 에러 |
| 501 | `NOT_IMPLEMENTED` | API가 서버 측에 구현되어 있지 않음 |
| 502 | `N/A` | 도달할 수 없는 네트워크 또는 잘못된 네트워크 설정으로 인한 에러 |
| 503 | `UNAVAILABLE` | 서버 다운 또는 다른 이유로 유효하지 않음. 클라이언트는 재시도 가능 |
| 504 | `DEADLINE_EXCEEDED` | 요청이 데드라인 이내에 처리되지 못함 |
- gRPC 에러 코드는 숫자로 된 상태 코드를 매핑할 필요 없이 **문자열 형태의 코드**를 사용하므로 이해하기 더 쉬움.