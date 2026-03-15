# 10. gRPC 시작하기

- **gRPC**는 네트워크를 통한 범용 **RPC(원격 프로시저 호출)** 용 오픈 소스 프레임워크임.
- 원격 상호 작용의 상세 사항을 직접 코딩할 필요 없이, 로컬 프로시저를 호출하는 것처럼 다른 시스템의 원격 프로시저를 호출할 수 있음.

## 10.1 gRPC 동작방식

- **전송 프로토콜**: HTTP/2 기반, 완전 이중 스트리밍(full-duplex streaming)을 지원함.
- **직렬화 형식**: Protocol Buffers(기본), JSON, XML, Thrift 등 미디어 형식을 지원함.
- **주요 기능**: 로드 밸런싱, 장애 조치, 캐스케이드 호출 취소를 지원하며, 대기 시간이 낮음.
- **통신 범위**: 서비스 간 통신, 모바일 앱, 웹 브라우저 ↔ gRPC 서버 통신을 모두 지원함.

## 10.2 REST 대 gRPC

| 항목 | REST | gRPC |
|---|---|---|
| 아키텍처 | 클라이언트-서버 아키텍처 기반X | 클라이언트-서버 아키텍처 기반O |
| HTTP 버전 | HTTP/1.1 (주로) | HTTP/2, 완전 이중 스트리밍 지원 |
| 페이로드 전달 | 쿼리/경로 매개변수, 요청 본문 등 다양한 방식 | 정적 경로 사용, 한 가지 방식만 사용 → 성능 우수 |
| 에러 처리 | HTTP 상태 코드에 의존 | 에러 집합을 정형화하여 API와 호환 |
| 구현 유연성 | 높음 (표준·규칙 없이 다양한 방식 가능) | 엄격한 계약 기반 |
| 추가 기능 | - | 호출 취소, 부하 분산, 장애 조치(fail-over) 지원 |

## 10.3 웹 브라우저와 모바일 앱에서 gRPC 서버를 호출할 수 있을까?

- 가능함. gRPC는 HTTP/2 의미 체계와 일치하도록 설계됐음.
- 인트라넷·인터넷을 통한 서비스 간 통신, 모바일 앱, 웹 브라우저에서 gRPC 서버 호출을 모두 지원함.
- 웹용 gRPC(gRPC-web)은 2018년 이후 점점 더 많은 인지도를 얻고 있으며, 특히 IoT 응용 프로그램에서 많이 사용됨.
- gRPC 도입 시 서비스 내부 통신에 먼저 채택한 다음 웹/모바일 서버 통신에 채택하는 순서를 권장함.

## 10.4 gRPC 아키텍처란

- gRPC는 다음 3개의 계층으로 구성된 계층화된 아키텍처임.

| 계층 | 역할 |
|---|---|
| **스텁** (최상위) | 클라이언트가 서버를 호출하는 레이어. IDL 파일에 생성되며, Protobuf 사용 시 `.proto` 확장자를 가짐. |
| **채널** (중간) | ABI(Application Binary Interfaces)를 제공하는 중간 계층. 특정 호스트·포트의 서버에 대한 연결을 제공함. |
| **전송** (최하위) | HTTP/2 프로토콜 사용. 완전 이중(full-duplex) 통신과 다중 병렬 호출(multiplex parallel call) 제공함. |

**gRPC 기반 서비스 개발 순서**

1. `.proto` 파일(Protobuf)을 사용해서 서비스 인터페이스를 정의한다.
2. 서비스 인터페이스의 구현을 작성한다.
3. gRPC 서버를 생성하고 서비스를 등록한다.
4. 서비스 스텁을 생성하고 gRPC 클라이언트와 함께 사용한다.

> **gRPC 스텁**: 서비스 인터페이스를 노출시키는 객체. gRPC 클라이언트는 스텁 메소드를 호출하고, 호출을 서버에 연결하고 응답을 다시 가져온다.

## 10.5 gRPC가 Protocol Buffer를 사용하는 방법

- Protobuf는 2001년에 만들어져서 2008년에 공개됐음.
- **gRPC의 기본 직렬화 형식**이며, JSON·YAML과 달리 사람이 읽을 수 없는 바이너리 형식으로 성능이 우수함.
- Protobuf를 사용하면 공식 계약(contract), 대역폭 최적화, 코드 생성이 가능함.

### 10.5.1 Protobuf 메시지 구조

- 메시지는 키–값 쌍으로 구성되며, 각 필드에는 타입과 일련번호(태그)를 지정함.
- 일련번호는 직렬화·파싱에 사용되며, **한 번 직렬화한 메시지 구조는 변경할 수 없음.**

```protobuf
message Employee {
    int64 id = 1;
    string firstName = 2;
}
```

### 10.5.2 와이어 타입

| 와이어 타입 | 의미 | 사용 용도 |
|---|---|---|
| 0 | Var int (가변 길이 정수) | int32, int64, uint32, uint64, sint32, sint64, bool, enum |
| 1 | 64-bit | fixed64, sfixed64, double |
| 2 | Length-delimited | string, bytes, embedded messages, packed repeated fields |
| 3 | Start group | groups (더 이상 사용되지 않음) |
| 4 | End group | groups (더 이상 사용되지 않음) |
| 5 | 32-bit | fixed32, sfixed32, float |

### 10.5.3 직원(Employee) 샘플 서비스 인터페이스

```protobuf
syntax = "proto3";                                          // (1) Protobuf 버전 정의
package com.packtpub;                                       // (2) proto 패키지명 (이름 충돌 방지)
option java_package = "com.packt.modern.api.proto";         // (3) 자바 패키지명 정의
option java_multiple_files = true;                          // (4) 메시지 타입별 별도 파일 생성

message Employee {
    int64 id = 1;
    string firstName = 2;
    string lastName = 3;
    int64 deptId = 4;
    double salary = 5;
    message Address {                                       // (5) 중첩 메시지 정의 가능
        string houseNo = 1;
        string street1 = 2;
        string city = 3;
        string state = 4;
        string country = 5;
        string pincode = 6;
    }
    enum Grade {                                            // (6) 열거형 타입
        I_GRADE = 1;
        II_GRADE = 2;
        III_GRADE = 3;
        IV_GRADE = 4;
    }
    map<string, int32> nominees = 1;                        // (7) map 타입
}

message EmployeeCreateResponse {
    int64 id = 1;
}

service EmployeeService {
    rpc Create(Employee) returns (EmployeeCreateResponse);  // (8) 서비스 정의
}
```

### 10.5.4 Protobuf 타입 ↔ 자바 타입 매핑

| Protobuf 타입 | 자바 타입 | 비고 |
|---|---|---|
| double / float | double / float | 자바와 동일 |
| int32 / int64 | int / long | 음수 포함 시 sint32 / sint64 사용 권장 |
| uint32 / uint64 | int / long | 값이 2³²/2⁶⁴보다 클 경우 fixed32 / fixed64 사용 |
| sint32 / sint64 | int / long | 음수 인코딩에 효율적, 가변 길이 인코딩 |
| fixed32 / fixed64 | int / long | 항상 4 bytes / 8 bytes |
| sfixed32 / sfixed64 | int / long | 항상 4 bytes / 8 bytes, 큰 값 인코딩에 효율적 |
| bool | boolean | 참 또는 거짓 |
| string | String | UTF-8 인코딩 문자열 (최대 2³²) |
| bytes | ByteString | 임의의 바이트 시퀀스 (최대 2³²) |

## 10.6 서비스 정의의 이해

- gRPC에서 제공하는 서비스 메소드의 타입은 4가지임.

| 타입 | 요청 | 응답 | 예시 |
|---|---|---|---|
| **단향** | 단일 | 단일 | `rpc Create(Employee) returns (EmployeeCreateResponse);` |
| **서버 스트리밍** | 단일 | 스트림 | `rpc LiveMatchScore(MatchId) returns (stream MatchScore);` |
| **클라이언트 스트리밍** | 스트림 | 단일 | `rpc AnalyzeData(stream DataInput) returns (Report);` |
| **양방향 스트리밍** | 스트림 | 스트림 | `rpc BatchProcessing(stream InputRecords) returns (stream Response);` |

## 10.7 RPC 수명 주기 살펴보기

### 10.7.1 단향 RPC 수명 주기

1. 클라이언트가 스텁 메소드를 호출한다.
2. 스텁이 서버에 메타데이터, 메소드 이름, 기한(deadline)을 전달한다.
3. 서버가 요청을 처리하고 상태정보(코드·옵션 메시지)와 응답을 보낸다.
4. 클라이언트가 응답을 수신하고 호출을 완료한다 (상태 OK = HTTP 상태 200과 유사).

### 10.7.2 스트리밍 RPC 수명 주기 비교

| 타입 | 단향과의 차이점 |
|---|---|
| **서버 스트리밍** | 서버가 모든 메시지를 스트림으로 전송 후 상태정보를 보내며 완료함. |
| **클라이언트 스트리밍** | 클라이언트가 모든 메시지를 스트림으로 전송 후 서버의 단일 응답을 기다림. |
| **양방향 스트리밍** | 두 스트림은 독립적으로 작동함. 서버와 클라이언트 모두 순서에 관계없이 읽고 쓸 수 있음. |

## 10.8 수명 주기에 영향을 주는 이벤트

- **데드라인/타임아웃**: 클라이언트는 정의된 데드라인/타임아웃까지 응답을 기다림. 초과 시 `DEADLINE_EXCEEDED` 에러 발생함. 타임아웃 설정은 언어별로 다름.
- **RPC 종료**: 클라이언트·서버가 각자 독립적으로 성공 여부를 결정하므로 결론이 불일치할 수 있음. (예: 서버는 성공, 클라이언트는 타임아웃으로 실패)
- **RPC 취소**: 서버·클라이언트 모두 언제든지 RPC를 취소 가능함. 즉시 종료되나 취소 전 변경 사항은 롤백되지 않음.

## 10.9 gRPC 서버 및 gRPC 스텁 이해

- `protoc`(Protobuf 컴파일러)와 gRPC 자바 플러그인으로 서비스 인터페이스를 컴파일하면 다음 두 가지 파일이 생성됨.
  - **모델 파일**: 요청·응답 메시지 타입의 직렬화, 역직렬화, 가져오기를 위한 Protobuf 코드 포함.
  - **gRPC 자바 파일**: 서비스 기반 인터페이스와 스텁 포함. 인터페이스는 gRPC 서버의 일부로, 스텁은 클라이언트가 서버와 통신하는 데 사용됨.

**서버 측 구현**

```java
// 1. 서비스 인터페이스 구현
public class EmployeeService extends EmployeeServiceImplBase {
    @Override
    public void create(Employee request,
            io.grpc.stub.StreamObserver<Response> responseObserver) {
        // 구현부
    }
}

// 2. gRPC 서버 실행
public class GrpcServer {
    public static void main(String[] arg) {
        try {
            Server server = ServerBuilder.forPort(8080)
                    .addService(new EmployeeService()).build();
            server.start();
            server.getServices().stream()
                    .forEach(s -> System.out.println("Service Name: "
                            + s.getServiceDescriptor().getName()));
            server.awaitTermination();
        } catch (Exception e) {
            // 에러 처리
        }
    }
}
```

**클라이언트 측 스텁 생성**

```java
public EmployeeServiceClient(ManagedChannelBuilder<?> channelBuilder) {
    channel = channelBuilder.build();
    blockingStub = EmployeeServiceGrpc.newBlockingStub(channel);  // 동기 스텁
    asyncStub = EmployeeServiceGrpc.newStub(channel);              // 비동기 스텁
}
```

## 10.10 에러 처리와 에러 상태 코드

- gRPC는 HTTP 상태 코드 대신 **에러 코드와 옵션 에러 메시지(문자열)가 포함된 상태 모델**을 사용함.
- 더 풍부한 에러 정보가 필요한 경우 아래 `Status` 모델을 활용함.

```protobuf
package google.rpc;
message Status {
    int32 code = 1;                           // 실제 에러 코드 (google.rpc.Code)
    string message = 2;                       // 개발자용 에러 메시지
    repeated google.protobuf.Any details = 3; // 추가 에러 정보 (RetryInfo, DebugInfo 등)
}
```

| HTTP 상태 코드 | gRPC 상태 코드 | 설명 |
|---|---|---|
| 400 | INVALID_ARGUMENT | 유효하지 않은 아규먼트 |
| 400 | FAILED_PRECONDITION | 잘못된 사전 조건으로 액션 실행 불가 |
| 400 | OUT_OF_RANGE | 클라이언트가 지정한 범위가 유효하지 않음 |
| 401 | UNAUTHENTICATED | 누락·만료된 토큰 또는 인증받지 않은 클라이언트 요청 |
| 403 | PERMISSION_DENIED | 클라이언트가 충분한 권한이 없음 |
| 404 | NOT_FOUND | 리소스를 찾을 수 없음 |
| 409 | ABORTED | 읽기–쓰기 작업 또는 동시성 충돌 |
| 409 | ALREADY_EXISTS | 이미 존재하는 리소스에 대한 생성 요청 |
| 429 | RESOURCE_EXHAUSTED | API 리미트에 도달해서 요청 처리 불가 |
| 499 | CANCELLED | 요청이 클라이언트에 의해서 취소됨 |
| 500 | DATA_LOSS | 복구 불가능한 데이터 손실 발생 |
| 500 | UNKNOWN | 알려지지 않은 서버 측 에러 |
| 500 | INTERNAL | 내부 서버 에러 |
| 501 | NOT_IMPLEMENTED | API가 서버 측에 구현되어 있지 않음 |
| 502 | N/A | 도달할 수 없는 네트워크 또는 잘못된 네트워크 설정으로 인한 에러 |
| 503 | UNAVAILABLE | 서버 다운 또는 다른 이유로 유효하지 않음. 클라이언트는 에러에 대해 재시도 할 수 있음 |
| 504 | DEADLINE_EXCEEDED | 요청이 데드라인 이내에 처리되지 못함 |
