# 6. 권한부여와 인증을 통해 REST 엔드포인트 보호하기

## 1️⃣ 스프링 시큐리티 및 JWT를 사용한 인증 구현
### Gradle에 필요한 의존성 추가하기
- **OAuth 2.0 리소스 서버 라이브러리**: `spring-boot-starter-oauth2-resource-server`를 추가함.
- **JWT 라이브러리**: `com.auth0:java-jwt`를 추가하여 JWT 생성 및 파싱 기능을 사용함.
### OAuth 2.0 리소스 서버를 사용한 인증 방법
- **필터 체인 동작**: 스프링 시큐리티는 요청이 `DispatcherServlet`에 도달하기 전에 **서블릿 필터 체인**을 통해 보안 로직을 수행함.
- `BearerTokenAuthenticationFilter`: OAuth 2.0 리소스 서버의 필터로, 요청 헤더에서 **Bearer 토큰**을 추출하고 인증을 시도함.
  - 토큰 부재 시: `FilterSecurityInterceptor`가 `AccessDeniedException`을 발생시키고, `ExceptionTranslationFilter`가 이를 잡아 401 Unauthorized 응답을 반환함.
  - 토큰 존재 시: 토큰을 추출하여 `BearerTokenAuthenticationToken`을 생성하고 `AuthenticationManager`에게 검증을 위임함.
  - 인증 성공: 인증 객체를 `SecurityContextHolder`에 저장하고 다음 필터로 진행함.
  - 인증 실패: 컨텍스트를 비우고 에러 메시지와 함께 401 에러를 반환함.
  
### JWT의 구조
- JWT는 aaa.bbb.ccc와 같은 형태로 Header.Payload.Signature 세 부분으로 구성되며, 각 부분은 점(.)으로 구분됨.
- **헤더**: 토큰의 유형(typ)과 해싱 알고리즘(alg) 정보를 포함함.
- **페이로드**: **클레임(Claim)** 이라 불리는 실제 데이터를 담고 있음.
  - **등록된 클레임**: iss(발급자), sub(주제), exp(만료 시간), iat(발급 시간) 등 표준화된 정보.
  - **공개/비공개 클레임**: 사용자 정의 데이터.
- **서명**: 헤더와 페이로드를 Base64로 인코딩한 값과 비밀키를 사용하여 생성함.

## 2️⃣ JWT로 REST API에 보안 적용하기
### 새로운 API 추가하기
- 기존 API를 개선하기 위해 **가입, 로그인, 로그아웃, 토큰 갱신** 등 4가지 API를 추가함.
- **Sign-up 엔드포인트**:
    - 새로운 사용자를 등록하고 `SignedInUser` 모델(액세스 토큰, 리프레시 토큰 포함)을 반환함.
- **Sign-in 엔드포인트**:
    - 사용자 이름과 비밀번호를 받아 인증하고 `SignedInUser` 모델을 반환함.
- **Sign-out 엔드포인트**:
    - 리프레시 토큰을 요청 본문에 담아 보내면, 서버에서 해당 토큰을 삭제하여 로그아웃 처리함.
- **토큰을 리프레시하는 엔드포인트**:
    - 유효한 리프레시 토큰을 보내면 새로운 액세스 토큰을 발급해 줌.

### 데이터베이스 테이블에 리프레시 토큰 저장하기
- 리프레시 토큰을 영구적으로 관리하기 위해 데이터베이스에 `user_token` 테이블을 생성함.

### JWT 관리자 구현하기
- **`Constants` 클래스**: 토큰 만료 시간(15분), 서명 키, API URL 패턴 등 보안 관련 상수들을 한곳에서 관리함.
- **`JwtManager` 클래스**: `java-jwt` 라이브러리를 사용하여 실제 토큰을 생성하는 역할을 담당함.
    - **공개 키/개인 키 사용**: RSA 알고리즘(SHA256withRSA)을 사용하여 토큰에 서명함.
    - **클레임 추가**: 발급자(iss), 주제(sub), 만료일(exp) 등의 표준 클레임 외에 `ROLE_CLAIM`과 같은 커스텀 클레임(권한 정보)을 포함함.
- **키 생성**: JDK의 `keytool`을 사용하여 (공개 키/개인 키를 생성하고 `jwt-keystore.jks` 파일로 저장함.
- **`SecurityConfig` 설정**:
    - **`KeyStore` 빈**: 저장된 jks 파일을 로드하여 `KeyStore` 인스턴스를 생성함.
    - **`RSAPrivateKey` 빈**, **`RSAPublicKey` 빈** 생성함.
- **`JwtDecoder` 빈 등록**: 스프링 시큐리티 리소스 서버가 토큰을 검증할 수 있도록 `NimbusJwtDecoder`에 공개 키를 주입하여 빈으로 등록함.


## 3️⃣ 새로운 API 구현

### findUserByUsername() 메소드 구현하기
- **`UserDetailsService` 인터페이스 활용**:
    - 스프링 시큐리티에서 사용자 정보를 로드하는 핵심 표준 인터페이스인 `UserDetailsService`를 구현하여 인증 로직을 작성함.
    - 이 인터페이스는 로그인 시 사용자 이름(username)을 기반으로 사용자 데이터를 조회하는 역할을 수행함.
- **`loadUserByUsername()`의 역할**:
    - `Authentication Manager`가 인증을 시도할 때 호출하는 메소드임.
    - 데이터베이스에서 조회한 사용자 엔터티(`UserEntity`)를 스프링 시큐리티가 이해할 수 있는 `UserDetails` 객체(사용자 이름, 암호화된 비밀번호, 권한 목록 등 포함)로 변환하여 반환함.
- **예외 처리**:
    - 만약 제공된 사용자 이름에 해당하는 사용자가 데이터베이스에 없을 경우, 반드시 `UsernameNotFoundException`을 발생시켜야 함.
### REST 컨트롤러 구현
- **`AuthController` 작성**: `UserApi` 인터페이스를 구현하며, `UserService`와 `PasswordEncoder` 빈을 주입받아 인증 로직을 수행함.
- **`PasswordEncoder` 설정**: `AppConfig`에 `BCryptPasswordEncoder`를 사용하는 빈을 등록하여 비밀번호 암호화를 함.
### 웹 기반 보안 설정
- **`SecurityFilterChain` 빈 등록**: 보안 설정을 담은 필터 체인을 빈으로 등록하는 방식을 사용함.
- **HTTP 보안 구성 (Fluent API)**:
    - JWT를 사용하므로 `httpBasic`, `formLogin`을 비활성화`disable`함.
    - REST API는 상태가 없어야 하므로 세션 정책을 `STATELESS`로 설정함.
    - 로그인, 회원가입, H2 콘솔, Swagger UI 등은 `permitAll()`로 접근 허용.
    - 그 외의 모든 요청은 `authenticated()`로 인증된 사용자만 접근하도록 제한함.
    - `oauth2ResourceServer().jwt()`를 설정하여 앞서 등록한 `JwtDecoder`를 통해 들어오는 요청의 JWT 유효성을 검증하도록 함.

## 4️⃣ CORS와 CSRF의 구성
### CORS
- 브라우저는 보안상의 이유로 스크립트가 다른 도메인(Origin)의 리소스를 호출하는 것을 기본적으로 차단함. 이를 허용하기 위해 설정이 필요함.
- **`CorsConfigurationSource` 빈 구현**:
    - `allowedOrigins("*")`: 모든 도메인에서의 요청을 허용함.
    - `allowedMethods(...)`: GET, POST, PUT, DELETE, PATCH 등 허용할 HTTP 메소드를 명시함.
    - `allowedHeaders("*")`: 모든 헤더를 허용함.
    - 이 설정을 `UrlBasedCorsConfigurationSource`에 등록하여 필터 체인에 적용함.

### CSRF
- 사용자가 의도하지 않은 악성 요청을 서버로 전송하게 만드는 공격 기법임.
- CSRF는 주로 세션(쿠키) 기반 인증에서 발생함.
- 헤더에 JWT 토큰을 담아 보내는 무상태 REST API 방식에서는 CSRF 공격으로부터 상대적으로 안전함.
- 따라서 `csrf().disable()`을 통해 기능을 비활성화함.

## 5️⃣ 권한부여(authorization)에 대한 이해

### 역할과 권한
- 애플리케이션에서 사용할 세 가지 역할(사용자, 관리자, 고객 지원)을 `RoleEnum`으로 정의하고 `GrantedAuthority` 인터페이스를 구현함.
- 스프링 시큐리티에서 역할은 일반적으로 `ROLE_` 접두사를 가짐.
  - `hasRole('ADMIN')`: 내부적으로 `ROLE_ADMIN` 권한을 확인함.
  - `hasAuthority('ROLE_ADMIN')`: 입력된 문자열 그대로 권한을 확인함.
- 스프링 시큐리티 OAuth2 리소스 서버는 기본적으로 범위 `scope` 또는 `scp` 클레임을 권한으로 매핑함.

- **메소드 수준 보안**
  - URL 패턴 매칭(`SecurityConfig`)만으로는 부족한 세밀한 접근 제어가 필요할 때 사용함.
  - `SecurityConfig` 클래스에 `@EnableGlobalMethodSecurity(prePostEnabled = true)` 애노테이션을 추가하여 활성화함.
- **@PreAuthorize 사용**:
    - 컨트롤러 메소드 위에 `@PreAuthorize("hasRole('ADMIN')")`와 같이 선언하여, 해당 메소드 실행 전에 권한을 검증함.
    - SpEL(Spring Expression Language)을 지원하므로 유연한 표현식 사용이 가능함.