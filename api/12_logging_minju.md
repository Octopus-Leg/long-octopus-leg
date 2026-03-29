# 12장. 서비스에 로깅 및 트레이싱 추가

## ELK 스택을 활용한 로깅과 트레이싱

마이크로서비스 환경에서 여러 서비스에 걸친 요청을 트레이싱하려면 **분산 및 중앙집중식 로깅**이 필요함.

- **트레이싱 식별자(traceId)**: 하나의 API 호출에 할당되는 고유 문자열로, 후속 API 호출에 전파됨
- **상관 식별자(correlationId)**: traceId의 다른 명칭

### ELK 스택의 이해

**ELK 스택**: Elasticsearch, Logstash, Kibana 세 가지로 구성된 로그 수집·분석·시각화 플랫폼

| 구성 요소 | 역할 |
|---|---|
| Elasticsearch | 아파치 루센 기반 분산 검색 엔진. JSON 기반 스키마 없는 스토리지, 약 1초 지연의 실시간 검색 제공 |
| Logstash | 오픈 소스 데이터 수집 엔진. 데이터 수집 → 구문 분석 → 필터링 → 저장소 전송 |
| Kibana | 오픈 소스 웹 기반 시각화 도구. Elasticsearch 인덱스의 로그를 검색·표시·분석 |

**로그 흐름**

```
서비스 1/2/n 로그  →  [브로커: Redis/Kafka/RabbitMQ]  →  Logstash  →  Elasticsearch  →  Kibana
```

> 프로덕션 환경에서는 데이터 손실 방지 및 데이터 급증 대비를 위해 서비스 로그와 Logstash 사이에 브로커 계층을 배치함.

### ELK 스택 설치

도커 컴포즈를 사용하여 설치함. 도커 컴포즈 파일의 4가지 최상위 키는 다음과 같음.

| 키 | 설명 |
|---|---|
| `version` | 도커 컴포즈 파일 형식의 버전 |
| `services` | 컨테이너 이름, 이미지, 환경 변수, 포트, 명령어, 네트워크, 볼륨, 의존 서비스 정의 |
| `networks` | 서비스 간 통신 채널 설정 (이 책에서는 `bridge` 사용) |
| `volumes` | 호스트 경로를 마운트하는 명명된 볼륨 |

```yaml
# /Chapter12/docker-compose.yaml
version: "3.2"
services:
  elasticsearch:
    container_name: es-container
    image: docker.elastic.co/elasticsearch/elasticsearch:8.7.0
    environment:
      - xpack.security.enabled=false
      - "discovery.type=single-node"
    networks:
      - elk-net
    ports:
      - 19200:9200   # 외부:내부

  logstash:
    container_name: ls-container
    image: docker.elastic.co/logstash/logstash:8.7.0
    environment:
      - xpack.security.enabled=false
    command: logstash -e 'input { tcp { port => 5001 codec => "json" }}
             output { elasticsearch { hosts => "elasticsearch:9200" index => "modern-api" }}'
    networks:
      - elk-net
    depends_on:
      - elasticsearch
    ports:
      - 5002:5001

  kibana:
    container_name: kb-container
    image: docker.elastic.co/kibana/kibana:8.7.0
    environment:
      - ELASTICSEARCH_HOSTS=http://es-container:9200
    networks:
      - elk-net
    depends_on:
      - elasticsearch
    ports:
      - 5600:5601

networks:
  elk-net:
    driver: bridge
```

Logstash `command`에는 세 가지 중요한 구성이 포함됨.

| 구성 | 설명 |
|---|---|
| `input` | TCP 입력 채널 (포트 5001). gRPC 서버/클라이언트가 JSON 형식으로 로그를 푸시 |
| `filter` | grok 등 필터 표현식 지정. 필터링하지 않으려면 생략 |
| `output` | 포트 9200의 Elasticsearch로 푸시, `modern-api` 인덱스 사용 |

```bash
$ docker-compose up -d                            # 백그라운드 실행
$ docker-compose logs --tail="10" elasticsearch   # 로그 확인
$ docker-compose logs --tail="10" kibana
$ docker-compose down                             # 중지 및 제거
```

모든 컨테이너가 기동되면 다음 URL로 동작을 확인할 수 있음.
- Elasticsearch: `http://localhost:19200/` → JSON 응답 확인
- Kibana: `http://localhost:5600` → 홈페이지 확인

## gRPC 코드에서 로깅 및 트레이싱 구현

- Spring Boot 3부터는 Spring Cloud Sleuth 대신 **Spring Micrometer**가 트레이싱을 지원함
- Zipkin 연동에는 **Brave** 라이브러리를 사용함
- 로그를 ELK 스택에 전송하기 위해 **logback-spring.xml**을 변경하여 Logstash로 전송함

**gRPC 서버·클라이언트 공통 변경 파일**

| 파일 | 변경 내용 |
|---|---|
| `build.gradle` | `logstash-logback-encoder`, `micrometer-tracing-bridge-brave`, `spring-boot-starter-actuator` 의존성 추가 |
| `logback-spring.xml` | STASH 어펜더 추가 — TCP 소켓으로 Logstash에 로그 전송. 로그 패턴에 `traceId`, `spanId` 포함 |
| `application.properties` | `logstash.destination`, `management.tracing.sampling.probability=1.0` 속성 추가 |

**gRPC 서버·클라이언트 개별 변경 파일**

| 파일 | 역할 |
|---|---|
| `Config.java` (서버) | `ObservationGrpcServerInterceptor` Bean 등록 |
| `GrpcServer.java` | `ServerBuilder`에 `ObservationGrpcServerInterceptor` 인터셉터 추가 |
| `Config.java` (클라이언트) | `ObservationGrpcClientInterceptor` Bean 등록 |
| `GrpcClient.java` | `ManagedChannelBuilder`에 `ObservationGrpcClientInterceptor` 인터셉터 추가 |

## 로깅 및 트레이싱 변경사항 테스트

**테스트 순서**: ELK 스택 실행 → gRPC 서버 시작 → gRPC 클라이언트 서비스 시작

```bash
$ curl http://localhost:8081/charges
```

**traceId vs spanId**

| 항목 | 범위 | 값 |
|---|---|---|
| traceId | 분산 트랜잭션 전체 | 클라이언트 ↔ 서버 **동일** |
| spanId | 개별 서비스 내 작업 단위 | 서비스마다 **다름** |

**Kibana에서 로그 검색하기**

1. `http://localhost:5600` 접속 → 햄버거 메뉴 → **Discover** 클릭
2. **Create data view** 클릭 → 인덱스 패턴에 `modern-api` 입력 → **Save data view to Kibana**
3. 검색창에 `traceId: {값}` 입력 → 해당 트랜잭션에 참여한 모든 서비스의 로그 확인

쿼리 기준은 **KQL(Kibana Query Language)** 을 사용하며, 비교 및 논리 연산자로 필터링 가능함.

## Zipkin과 Micrometer로 분산 트레이싱 하기

- **Spring Micrometer**: 스프링 부트 애플리케이션에서 생성된 메트릭을 수집하는 유틸리티 라이브러리. ELK 같은 시스템으로 메트릭을 내보낼 수 있는 벤더 중립적인 API를 제공함
- **Zipkin**: 각 서비스가 소요한 응답 시간을 수집하고 시각화하여 성능 병목 위치를 파악하는 데 도움이 됨

**Spring Micrometer가 수집하는 메트릭 종류**

- JVM, CPU, 캐시 관련 메트릭
- Spring MVC, WebFlux, REST 클라이언트의 레이턴시
- 데이터소스와 HikariCP 관련 메트릭
- 기동 시간과 톰캣 사용 정보
- Logback에 기록된 이벤트 정보

**Zipkin 연동 설정** (gRPC 서버·클라이언트 공통)

| 항목 | 내용 |
|---|---|
| 추가 의존성 | `io.zipkin.reporter2:zipkin-reporter-brave` |
| 추가 속성 | `management.zipkin.tracing.endpoint=http://localhost:9411/api/v2/spans` |
| 실행 방법 | `java -jar zipkin-server-{version}-exec.jar` (기본 포트: 9411) |
| UI 접속 | `http://localhost:9411` → Search by trace ID에 traceId 입력 |
