# 9장. 웹서비스 배포하기

## 9.1 컨테이너화란 무엇일까?

- 서로 다른 시스템 간 **종속성(Java 버전, OS, 웹 서버 등), 구성 또는 파일의 불일치**로 인해 발생하는 문제를 해결하기 위해 애플리케이션을 컨테이너화함.
- 컨테이너화를 하면 애플리케이션은 필요한 모든 의존성과 함께 번들링되어 모든 환경에서 동일하게 동작함.
- 컨테이너는 호스트 운영체제의 라이브러리, 바이너리뿐만 아니라 **커널도 공유**하므로 아주 가벼움.

### 9.1.1 가상머신 vs 컨테이너

- **가상화(virtualization)** 는 하드웨어를 분할하여 **가상 머신(virtual machine)** 을 만드는 프로세스로, 컨테이너와는 다름.

| 항목 | 가상머신 | 컨테이너 |
|---|---|---|
| 실행 방식 | 가상화 플랫폼(hypervisor) 위에서 실행 | OS 위에서 격리된 프로세스로 실행 |
| 커널 공유 | 각 VM이 별도 OS 보유 | 호스트 OS 커널 공유 |
| 크기 | 무거움 (수 GB) | 가벼움 (몇 MB ~ 1GB) |
| 이식성 | 낮음 | 높음 |

## 9.2 도커(Docker) 이미지 빌드하기

### 9.2.1 도커란 무엇인가?

- 2013년에 런칭한 시장을 리딩하는 **컨테이너 플랫폼이자 오픈 소스 프로젝트**임.
- 리눅스 커널의 **cgroup**과 **네임스페이스**를 사용해 의존성을 애플리케이션과 함께 패키징할 수 있도록 제공함.
- 각 컨테이너는 고유한 사용자 네임스페이스를 가짐.

**컨테이너 네임스페이스 종류**

| 네임스페이스 | 역할 |
|---|---|
| **PID** | 프로세스 격리 |
| **NET** | 네트워크 인터페이스 관리 |
| **IPC** | 프로세스 간 통신 리소스에 대한 액세스 관리 |
| **MNT** | 파일시스템 마운트 포인트 관리 |
| **UTS** | 커널과 버전 식별자 격리 |

### 9.2.2 도커 아키텍처에 대한 이해

- 도커는 **클라이언트-서버 아키텍처**를 채택함.
- 클라이언트와 데몬은 **소켓이나 RESTful API**를 통해 통신함.

**도커 구성 요소**

| 구성 요소 | 설명 |
|---|---|
| **도커 클라이언트** | 최종 사용자가 사용하는 CLI. 소켓 또는 RESTful API로 데몬과 통신 |
| **도커 데몬** | 컨테이너의 빌드, 실행, 배포 등 복잡한 작업을 처리 |
| **도커 이미지** | 읽기 전용 템플릿. 컨테이너를 만드는 데 사용됨 |
| **도커 컨테이너** | 이미지로부터 생성. 자체 프로세스, 파일시스템, 네트워킹 스택을 가짐 |
| **도커 레지스트리** | 이미지를 업로드/다운로드하는 리포지토리. 예: Docker Hub |

### 9.2.3 도커 컨테이너의 생명주기

| 단계 | 명령어 | 설명 |
|---|---|---|
| 컨테이너 생성 | `docker create` | 컨테이너 이미지로부터 컨테이너 생성 |
| 컨테이너 실행 | `docker run` | 생성된 컨테이너를 실행 |
| 컨테이너 일시 중지 (선택) | `docker pause` | 실행 중인 프로세스를 일시 중지 |
| 컨테이너 일시 중지 해제 (선택) | `docker unpause` | 일시 중지된 프로세스를 다시 실행 |
| 컨테이너 시작 | `docker start` | 중지된 컨테이너를 시작 |
| 컨테이너 중지 | `docker stop` | 컨테이너와 내부 프로세스를 중지 |
| 컨테이너 재시작 | `docker restart` | 컨테이너와 내부 프로세스를 재시작 |
| 컨테이너 강제 종료 | `docker kill` | 실행 중인 컨테이너를 강제 종료 |
| 컨테이너 제거 | `docker rm` | 중지된 컨테이너를 삭제 |

## 9.3 액추에이터(Actuator) 의존성 추가를 통해 이미지 빌드하기

- 도커 이미지 생성을 위한 추가 라이브러리는 필요 없음.
- 프로덕션 수준의 기능을 제공하는 **스프링 부트 액추에이터(spring boot actuator)** 의존성만 추가함.
- 이 장에서는 `/actuator/health` 엔드포인트만 사용해 컨테이너 내부 서비스의 상태를 파악함.

**1. build.gradle에 의존성 추가**
```gradle
runtimeOnly 'org.springframework.boot:spring-boot-starter-actuator'
```

**2. Constants.java에 URL 상수 추가**
```java
public static final String ACTUATOR_URL_PREFIX = "/actuator/**";
```

**3. SecurityConfig.java — 액추에이터 엔드포인트 인증 제외**
```java
// 앞부분 생략
req.requestMatchers(toH2Console()).permitAll()
    .requestMatchers(new AntPathRequestMatcher(
        ACTUATOR_URL_PREFIX)).permitAll()
    .requestMatchers(new AntPathRequestMatcher(
        TOKEN_URL, HttpMethod.POST.name())).permitAll()
// 뒷부분 생략
```

## 9.4 스프링 부트 플러그인 태스크 설정하기

- 스프링 부트 그래들 플러그인의 `bootBuildImage` 명령어로 도커 이미지를 빌드함.
- **`.jar` 파일 빌드에만 사용 가능**함(`.war` 파일 불가).
- **Paketo 빌드팩**을 사용해 이미지를 빌드하며, LTS 버전의 자바 릴리스만 지원함.
- Java 17은 LTS이므로 **지원이 중단되지 않음**.

> 플러그인이 대문자 여부를 체크하기 때문에 **도커 이미지 이름은 항상 소문자로만 지정**해야 함.

```groovy
// build.gradle
bootBuildImage {
    imageName = "192.168.1.2:5000/${project.name}:${project.version}"
}
environment = ["BP_JVM_VERSION" : "17"]

// settings.gradle
rootProject.name = 'packt-modern-api-development-chapter09'
```

## 9.5 도커 레지스트리의 설정

- 개발 단계에서는 **로컬 도커 레지스트리**를 구성하는 것이 이상적임.
- 로컬 레지스트리는 **TLS(전송 계층 보안)** 를 구성하지 않아 안전하지 않으므로 도커 설정 변경이 필요함.

**daemon.json에 안전하지 않은 레지스트리 추가하기**

1. `Docker app > Settings > Docker Engine` 메뉴를 찾는다.
2. JSON 파일에 `insecure-registries` 항목을 추가한다.

```json
{
    "features": { "buildkit": true },
    "insecure-registries": [ "192.168.1.2:5000" ]
}
```

3. 도커를 재실행한다.

## 9.6 이미지를 빌드하는 그래들 태스크 실행

```bash
# 1. 로컬 도커 레지스트리 구동
$ docker run -d -p 5000:5000 -e REGISTRY_STORAGE_DELETE_ENABLED=true \
--restart=always --name registry registry:2

# 2. 이미지 빌드
$ ./gradlew clean build
$ ./gradlew bootBuildImage

# 3. 로컬 컨테이너 실행 (테스트)
$ docker run -p 8080:8080 \
192.168.1.2:5000/packt-modern-api-development-chapter09:0.0.1-SNAPSHOT

# 4. 이미지 태그 설정 및 레지스트리 푸시
$ docker tag 192.168.1.2:5000/packt-modern-api-development-chapter09:0.0.1-SNAPSHOT \
192.168.1.2:5000/packt-modern-api-development-chapter09:0.0.1-SNAPSHOT
$ docker push 192.168.1.2:5000/packt-modern-api-development-chapter09:0.0.1-SNAPSHOT

# 5. 레지스트리 이미지 확인
$ curl -X GET http://192.168.80.1:5000/v2/_catalog
{"repositories":["packt-modern-api-development-chapter09"]}

$ curl -X GET http://192.168.80.1:5000/v2/packt-modern-api-development-chapter09/tags/list
{"name":"packt-modern-api-development-chapter09","tags":["0.0.1-SNAPSHOT"]}
```

## 9.7 쿠버네티스에 애플리케이션 배포하기

- 도커 컴포즈도 여러 컨테이너를 관리하지만, **동적 확장**이 필요한 경우 쿠버네티스를 사용하는 것이 좋음.
- **미니큐브(Minikube)**: 로컬에서 학습 및 개발 목적으로 **단일 노드의 쿠버네티스 클러스터**를 실행하는 도구.
- **네임스페이스(namespace)**: 쿠버네티스 리소스를 사용자나 프로젝트 간에 나누는 특수 객체. 기본값은 `default`.

### 9.7.1 미니큐브 설정 (로컬 레지스트리 연동)

- 미니큐브는 기본적으로 원격 도커 허브를 사용하므로 로컬 레지스트리 연동을 위해 설정 변경이 필요함.
- `~/.minikube/machines/minikube/config.json` 의 `InsecureRegistry` 항목에 로컬 레지스트리를 추가함.

```json
"InsecureRegistry": [
    "10.96.0.0/12",
    "192.168.1.2:5000"
],
```

```bash
# 미니큐브 재시작
$ minikube start --insecure-registry="192.168.80.1:5000"

# 쿠버네티스 동작 확인
$ kubectl get po -A

# 미니큐브 대시보드 실행
$ minikube dashboard
```

### 9.7.2 deployment.yaml 생성

**쿠버네티스 핵심 객체**

| 객체 | 역할 |
|---|---|
| **Deployment** | 애플리케이션 컨테이너를 실행할 파드를 클러스터에 생성. replica 수로 규모 정의 |
| **Service** | 파드의 IP를 외부에 노출하거나 맵핑 정보를 관리하는 추상화 계층 제공 |
| **Pod** | 쿠버네티스의 가장 작은 배포 단위. 하나 이상의 컨테이너를 포함하며 단일 인스턴스를 의미 |

```bash
# deployment.yaml 생성
$ kubectl create deployment chapter09 \
--image=192.168.1.2:5000/packt-modern-api-development-chapter09:0.0.1-SNAPSHOT \
--dry-run=client -o=yaml > deployment.yaml

$ echo --- >> deployment.yaml

$ kubectl create service clusterip chapter09 --tcp=8080:8080 \
--dry-run=client -o=yaml >> deployment.yaml
```

```yaml
# /Chapter09/k8s/deployment.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: chapter09
  name: chapter09
spec:
  replicas: 1
  selector:
    matchLabels:
      app: chapter09
  strategy: {}
  template:
    metadata:
      labels:
        app: chapter09
    spec:
      containers:
      - image: 192.168.1.2:5000/packt-modern-api-development-chapter09:0.0.1-SNAPSHOT
        name: packt-modern-api-development-chapter09
        resources: {}
status: {}
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: chapter09
  name: chapter09
spec:
  ports:
  - name: 8080-8080
    port: 8080
    protocol: TCP
    targetPort: 8080
  selector:
    app: chapter09
  type: ClusterIP
status:
  loadBalancer: {}
```

### 9.7.3 쿠버네티스에 배포 및 확인

```bash
# 배포 실행
$ kubectl apply -f k8s/deployment.yaml
deployment.apps/chapter09 created
service/chapter09 created

# 파드 및 서비스 상태 확인
$ kubectl get all
```

### 9.7.4 SSH 터널링으로 외부에서 접근하기

- 쿠버네티스 내에서 실행 중인 애플리케이션에 직접 접근할 방법이 없음.
- **포트 포워딩(SSH 터널링)** 을 사용해 외부에서 접근함.

```bash
# 포트 포워딩
$ kubectl port-forward service/chapter09 8080:8080
Forwarding from 127.0.0.1:8080 -> 8080

# 새 터미널에서 헬스 체크
$ curl localhost:8080/actuator/health
{"status":"UP","groups":["liveness","readiness"]}
```
