# 6. 권한 부여와 인증을 통해 REST 엔드포인트 보호하기

## 6.1 스프링 시큐리티 및 JWT를 사용한 인증 구현

- **스프링 시큐리티**는 보일러플레이트 코드로 작성하지 않아도 엔터프라이즈 애플리케이션 레벨의 보안 기능을 쉽게 구현해주는 라이브러리로 구성된 프레임워크임.

- JWT 토큰을 사용하면 다양한 권한인증 플로우를 통해 보호된 HTTP 엔드포인트와 리소스들을 상태 없는(stateless) 방식으로 호출할 수 있음.

- 스프링 시큐리티는 요청이 DispatcherServlet에 도달하기 전 필터 수준에서 인증 로직을 수행함. 클라이언트 요청이 REST 컨트롤러에 도달하기 전 거치는 주요 보안 필터 순서는 다음과 같음.
  1. WebAsyncManagerIntegrationFilter
  2. SecurityContextPersistenceFilter
  3. HeaderWriterFilter
  4. CorsFilter
  5. CsrfFilter
  6. LogoutFilter
  7. **BearerTokenAuthenticationFilter** (베어러 토큰 인증 로직 포함)
  8. RequestCacheAwareFilter
  9. SecurityContextHolderAwareRequestFilter
  10. AnonymousAuthenticationFilter
  11. SessionManagementFilter
  12. ExceptionTranslationFilter
  13. FilterSecurityInterceptor

### 6.1.1 Gradle에 필요한 의존성 추가하기

JWT 기반 인증을 구현하기 위해 다음 의존성을 추가함.
```gradle
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-oauth2-resource-server'
    implementation 'com.auth0:java-jwt:4.3.0'
}
```

### 6.1.2 OAuth 2.0 리소스 서버를 사용한 인증 방법

OAuth 2.0 리소스 서버를 사용한 토큰 인증의 흐름은 다음과 같음.

1. 클라이언트가 Authorization 헤더에 Bearer 토큰을 포함하여 HTTP 요청을 보냄.
2. BearerTokenAuthenticationFilter가 HTTP 요청의 Authorization 헤더에서 토큰을 추출함.
3. 추출된 토큰으로 BearerTokenAuthenticationToken 인스턴스를 생성하고, AuthenticationManager로 전달하여 토큰을 검증함.
4. 인증이 성공하면 SecurityContext 인스턴스에 Authentication 객체가 설정되고 SecurityContextHolder에 저장됨. 이후 요청은 컨트롤러로 라우팅됨.
5. SecurityContextHolder.clearContext()가 호출되고, ExceptionTranslationFilter가 작동하여 401 Unauthorized 상태 코드와 함께 적절한 에러 메시지를 응답함.

### 6.1.3 JWT의 구조

JWT는 점(.)을 구분자로 하여 세 부분(`aaa.bbb.ccc`)으로 구성됨.

**1) 헤더(Header)**

토큰의 유형(typ)과 서명 알고리즘(alg) 정보를 담고 있음.
```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```

**2) 페이로드(Payload)**

토큰의 클레임(Claim, 정보의 한 조각)을 포함함.

- **등록된(Registered) 클레임**: `iss` (발급자), `sub` (주제/사용자 ID), `exp` (만료 시간), `iat` (발급 시간), `jti` (JWT ID), `aud` (수신자), `nbf` (Not Before)
- **공개(public) 클레임**: JWT 발급자가 정의함.
- **비공개 클레임(private)**: 발급자와 수신자가 정의함.
```json
{
  "sub": "scott2",
  "roles": ["ADMIN"],
  "iss": "Modern API Development with Spring and Spring Boot",
  "exp": 1676526792,
  "iat": 1676525892
}
```

**3) 서명(Signature)**

헤더와 페이로드를 인코딩한 값에 비밀 키 또는 공개/개인 키를 사용하여 생성함. 토큰이 전송 과정에서 수정되지 않았음을 보장함.

## 6.2 JWT로 REST API에 보안 적용하기

REST API를 보호하기 위해서는 다음 기능이 필요함.

- 비밀번호는 bcrypt의 해싱 기능을 사용해 인코딩한 후 저장해야 함.
- JWT는 **RSA(Rivest, Shamir, Adleman)** 알고리즘을 사용하여 생성한 키로 서명해야 함.
- 페이로드의 클레임에 민감한 정보나 보안 정보를 직접 저장해서는 안 되며, 필요 시 암호화해야 함.
- 특정 역할(Role)의 사용자에게만 API 접근 권한을 부여할 수 있어야 함.

### 6.2.1 새로운 API 추가하기

인증 시스템 구축을 위해 가입, 로그인, 로그아웃, 토큰 갱신 API를 `openapi.yaml`에 정의함.

- **Sign-up (가입)**: `/api/v1/users` (POST) 호출 시 accessToken, refreshToken, username, userId를 포함한 SignedInUser 모델을 반환함.
- **Sign-in (로그인)**: `/api/v1/auth/token` (POST)을 통해 username과 password를 전달받아 JWT와 리프레시 토큰을 생성함.
- **Sign-out (로그아웃)**: `/api/v1/auth/token` (DELETE) 호출 시 리프레시 토큰을 서버에서 제거하여 무효화함.
- **Refresh Token (토큰 갱신)**: `/api/v1/auth/token/refresh` (POST)를 통해 유효한 리프레시 토큰을 확인하고 새로운 JWT를 발급함.

### 6.2.2 데이터베이스 테이블에 리프레시 토큰 저장하기

리프레시 토큰을 관리하기 위한 테이블을 생성함.
```sql
create TABLE IF NOT EXISTS ecomm.user_token (
    id uuid NOT NULL DEFAULT random_uuid(),
    refresh_token varchar (128),
    user_id uuid NOT NULL,
    PRIMARY KEY (id),
    FOREIGN KEY (user_id) REFERENCES ecomm."user" (id)
);
```

### 6.2.3 JWT 관리자 구현하기

**보안 상수 관리**
```java
public class Constants {
    public static final String ENCODER_ID = "bcrypt";
    public static final long EXPIRATION_TIME = 900_000; // 15분
    public static final String TOKEN_PREFIX = "Bearer ";
    public static final String ROLE_CLAIM = "roles";
}
```

JwtManager는 `java-jwt` 라이브러리를 사용하여 토큰을 생성하는 JWT 관리자 클래스임.
```java
@Component
public class JwtManager {
    private final RSAPrivateKey privateKey;
    private final RSAPublicKey publicKey;
    
    public JwtManager(RSAPrivateKey privateKey, RSAPublicKey publicKey) {
        this.privateKey = privateKey;
        this.publicKey = publicKey;
    }
    
    public String create(UserDetails principal) {
        final long now = System.currentTimeMillis();
        return JWT.create()
            .withIssuer("Modern API Development with Spring and Spring Boot")
            .withSubject(principal.getUsername())
            .withClaim(ROLE_CLAIM, principal.getAuthorities().stream()
                .map(GrantedAuthority::getAuthority)
                .collect(toList()))
            .withIssuedAt(new Date(now))
            .withExpiresAt(new Date(now + EXPIRATION_TIME))
            .sign(Algorithm.RSA256(publicKey, privateKey));
    }
}
```

### 6.2.4 공개 키/개인 키 생성하기

JDK의 `keytool`을 사용하여 4096비트 RSA 키 쌍을 생성함.
```bash
$ keytool -genkey -alias "jwt-sign-key" -keyalg RSA -keystore jwt-keystore.jks -keysize 4096
```

**키 저장소 설정 (application.properties)**
```properties
app.security.jwt.keystore-location=classpath:jwt-keystore.jks
app.security.jwt.keystore-password=changeit
app.security.jwt.key-alias=jwt-sign-key
app.security.jwt.private-key-passphrase=password
```

## 6.3 새로운 API 구현

### 6.3.1 사용자 토큰 기능 구현하기

user_token 테이블을 토대로 UserTokenEntity를 생성함.
```java
@Entity
@Table(name = "user_token")
public class UserTokenEntity {
    
    @Id
    @GeneratedValue
    @Column(name = "ID", updatable = false, nullable = false)
    private UUID id;
    
    @NotNull(message = "Refresh token is required.")
    @Basic(optional = false)
    @Column(name = "refresh_token")
    private String refreshToken;
    
    @ManyToOne(fetch = FetchType.LAZY)
    private UserEntity user;
    
    // getter, setter
}
```

UserTokenEntity를 생성하고, 읽고, 업데이트 하고, 삭제하는 Repository를 생성함.
```java
public interface UserTokenRepository extends CrudRepository<UserTokenEntity, UUID> {
    Optional<UserTokenEntity> findByRefreshToken(String refreshToken);
    Optional<UserTokenEntity> deleteByUserId(UUID userId);
}
```

### 6.3.2 UserService 메소드 구현

**findUserByUsername()**

아규먼트로 전달받은 사용자 이름으로 사용자 정보를 찾아 반환함.
```java
@Override
public UserEntity findUserByUsername(String username) {
    if (Strings.isBlank(username)) {
        throw new UsernameNotFoundException("Invalid user.");
    }
    final String uname = username.trim();
    Optional<UserEntity> oUserEntity = repository.findByUsername(uname);
    UserEntity userEntity = oUserEntity.orElseThrow(() ->
        new UsernameNotFoundException(String.format("Given user(%s) not found.", uname)));
    return userEntity;
}
```

**createUser()**

새로 가입한 사용자를 데이터베이스에 추가함.
```java
@Override
@Transactional
public Optional<SignedInUser> createUser(User user) {
    Integer count = repository.findByUsernameOrEmail(user.getUsername(), user.getEmail());
    if (count > 0) {
        throw new GenericAlreadyExistsException("Use different username and email.");
    }
    UserEntity userEntity = repository.save(toEntity(user));
    return Optional.of(createSignedUserWithRefreshToken(userEntity));
}

private SignedInUser createSignedUserWithRefreshToken(UserEntity userEntity) {
    return createSignedInUser(userEntity)
        .refreshToken(createRefreshToken(userEntity));
}

private SignedInUser createSignedInUser(UserEntity userEntity) {
    String token = tokenManager.create(org.springframework.security.core.userdetails.User.builder()
        .username(userEntity.getUsername())
        .password(userEntity.getPassword())
        .authorities(Objects.nonNull(userEntity.getRole()) ? userEntity.getRole().name() : ""))
        .build());
    return new SignedInUser().username(userEntity.getUsername()).accessToken(token)
        .userId(userEntity.getId().toString());
}

private String createRefreshToken(UserEntity user) {
    String token = RandomHolder.randomKey(128);
    userTokenRepository.save(newUserTokenEntity()
        .setRefreshToken(token).setUser(user));
    return token;
}

private static class RandomHolder {
    static final Random random = new SecureRandom();
    
    public static String randomKey(int length) {
        return String.format("%"+length+"s", new BigInteger(length*5, random)
            .toString(32)).replace('\u0020', '0');
    }
}
```

**getSignedInUser()**

리프레시 토큰, 액세스 토큰(JWT), 사용자 ID 및 사용자 이름을 담고 있는 새 SignedInUser 인스턴스를 만듦.
```java
@Override
@Transactional
public SignedInUser getSignedInUser(UserEntity userEntity) {
    userTokenRepository.deleteByUserId(userEntity.getId());
    return createSignedUserWithRefreshToken(userEntity);
}
```

**getAccessToken()**

아규먼트로 전달된 유효한 리프레시 토큰을 가지고 새 액세스 토큰(JWT)을 생성하여 반환함.
```java
@Override
public Optional<SignedInUser> getAccessToken(RefreshToken refreshToken) {
    return userTokenRepository
        .findByRefreshToken(refreshToken.getRefreshToken())
        .map(ut -> Optional.of(createSignedInUser(ut.getUser()))
        .refreshToken(refreshToken.getRefreshToken()))
        .orElseThrow(() -> new InvalidRefreshTokenException("Invalid token."));
}
```

**removeRefreshToken()**

사용자가 로그아웃할 때 호출되어 데이터베이스에서 리프레시 토큰을 제거함.
```java
@Override
public void removeRefreshToken(RefreshToken refreshToken) {
    userTokenRepository
        .findByRefreshToken(refreshToken.getRefreshToken())
        .ifPresentOrElse(
            userTokenRepository::delete,
            () -> {
                throw new InvalidRefreshTokenException("Invalid token.");
            });
}
```

### 6.3.3 UserRepository 클래스의 개선
```java
@Repository
public interface UserRepository extends CrudRepository<UserEntity, UUID> {
    Optional<UserEntity> findByUsername(String username);
    
    @Query("select count(u) from UserEntity u where u.username = :username or u.email = :email")
    Integer findByUsernameOrEmail(String username, String email);
}
```

### 6.3.4 PasswordEncoder용 빈 추가
```java
@Bean
public PasswordEncoder passwordEncoder() {
    Map<String, PasswordEncoder> encoders = Map.of(
        ENCODER_ID, new BCryptPasswordEncoder(),
        "pbkdf2", Pbkdf2PasswordEncoder.defaultsForSpringSecurity_v5_8(),
        "scrypt", SCryptPasswordEncoder.defaultsForSpringSecurity_v5_8());
    return new DelegatingPasswordEncoder(ENCODER_ID, encoders);
}
```

DelegatingPasswordEncoder를 사용하면 기존 암호를 지원할 뿐 아니라 새로운 인코더를 쉽게 추가할 수 있음. 인코딩된 비밀번호에는 {bcrypt}와 같은 해싱 알고리즘 접두사를 추가해야 함.

### 6.3.5 컨트롤러 클래스 구현
```java
@RestController
public class AuthController implements UserApi {
    
    private final UserService service;
    private final PasswordEncoder passwordEncoder;
    
    public AuthController(UserService service, PasswordEncoder passwordEncoder) {
        this.service = service;
        this.passwordEncoder = passwordEncoder;
    }
    
    @Override
    public ResponseEntity<SignedInUser> signIn(@Valid SignInReq signInReq) {
        UserEntity userEntity = service.findUserByUsername(signInReq.getUsername());
        if (passwordEncoder.matches(signInReq.getPassword(), userEntity.getPassword())) {
            return ok(service.getSignedInUser(userEntity));
        }
        throw new InsufficientAuthenticationException("Unauthorized.");
    }
    
    @Override
    public ResponseEntity<Void> signOut(@Valid RefreshToken refreshToken) {
        service.removeRefreshToken(refreshToken);
        return accepted().build();
    }
    
    @Override
    public ResponseEntity<SignedInUser> signUp(@Valid User user) {
        return status(HttpStatus.CREATED).body(service.createUser(user).get());
    }
    
    @Override
    public ResponseEntity<SignedInUser> getAccessToken(@Valid RefreshToken refreshToken) {
        return ok(service.getAccessToken(refreshToken).orElseThrow(InvalidRefreshTokenException::new));
    }
}
```

### 6.3.6 웹 기반 보안 설정
```java
@Bean
protected SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    http.httpBasic().disable()
        .formLogin().disable()
        .csrf().ignoringRequestMatchers(API_URL_PREFIX).ignoringRequestMatchers(toH2Console())
        .and()
        .headers().frameOptions().sameOrigin()
        .and()
        .cors()
        .and()
        .authorizeHttpRequests(req -> req.requestMatchers(toH2Console()).permitAll()
            .requestMatchers(new AntPathRequestMatcher(TOKEN_URL, HttpMethod.POST.name())).permitAll()
            .requestMatchers(new AntPathRequestMatcher(TOKEN_URL, HttpMethod.DELETE.name())).permitAll()
            .requestMatchers(new AntPathRequestMatcher(SIGNUP_URL, HttpMethod.POST.name())).permitAll()
            .requestMatchers(new AntPathRequestMatcher(REFRESH_URL, HttpMethod.POST.name())).permitAll()
            .requestMatchers(new AntPathRequestMatcher(PRODUCTS_URL, HttpMethod.GET.name())).permitAll()
            .requestMatchers("/api/v1/addresses/**").hasAuthority(RoleEnum.ADMIN.getAuthority())
            .anyRequest().authenticated())
        .oauth2ResourceServer(oauth2ResourceServer ->
            oauth2ResourceServer.jwt(jwt -> jwt.jwtAuthenticationConverter(getJwtAuthenticationConverter())))
        .sessionManagement().sessionCreationPolicy(SessionCreationPolicy.STATELESS);
    return http.build();
}
```

설정 내용

- HTTP 기본 인증과 폼 로그인 비활성화
- H2 콘솔에 대한 CSRF 무시 설정
- H2 콘솔 프레임 표시를 위한 헤더 설정
- CORS 활성화
- URL 패턴별 접근 권한 설정
- OAuth 2.0 리소스 서버 JWT 지원
- 세션 생성 정책을 STATELESS로 설정

## 6.4 CORS와 CSRF의 구성

### 6.4.1 CORS(Cross-Origin Resource Sharing) 설정

브라우저는 보안상 출처(origin)가 다른 스크립트의 타 도메인 URL 호출을 제한함. CORS 설정을 통해 다른 출처 간의 요청을 허용할 수 있음.
```java
@Bean
CorsConfigurationSource corsConfigurationSource() {
    CorsConfiguration configuration = new CorsConfiguration();
    configuration.setAllowedOrigins(Arrays.asList("*"));
    configuration.setAllowedMethods(Arrays.asList("HEAD", "GET", "PUT", "POST", "DELETE", "PATCH"));
    configuration.addAllowedOrigin("*");
    configuration.addAllowedHeader("*");
    configuration.addAllowedMethod("*");
    
    UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
    source.registerCorsConfiguration("/**", configuration);
    return source;
}
```

**CORS 요청 흐름**

1. 브라우저가 OPTIONS 메소드로 사전 요청을 보냄.
2. 서버는 Access-Control-Allow-Origin, Access-Control-Allow-Methods, Access-Control-Allow-Headers 헤더를 응답에 포함함.
3. 브라우저가 허용 여부를 확인하고 실제 요청을 실행함.

### 6.4.2 CSRF(Cross-Site Request Forgery)

- CSRF는 사용자가 인증된 상태에서 악의적인 웹사이트를 통해 원치 않는 요청을 보내는 공격임.
- 예) 사용자가 은행 웹사이트에 로그인한 상태에서 악성 스크립트가 포함된 이메일 링크를 클릭하면, 해당 스크립트가 사용자 모르게 자금 이체 요청을 보낼 수 있음.
- REST 엔드포인트만 제공하는 경우 csrf().disable()을 사용하여 CSRF 보호를 비활성화할 수 있음. 단, 세션 기반 인증을 사용하는 경우에는 CSRF 보호를 활성화해야 함.

## 6.5 권한부여(authorization)에 대한 이해

권한 부여는 인증된 사용자가 특정 리소스에 접근할 수 있는 권한이 있는지 확인하는 과정임. 스프링 시큐리티는 역할(Role)과 권한(Authority) 기반으로 접근 제어를 제공함.
```java
public enum RoleEnum implements GrantedAuthority {
    USER(Const.USER),
    ADMIN(Const.ADMIN),
    CSR(Const.CSR);
    
    private String authority;
    
    RoleEnum(String authority) {
        this.authority = authority;
    }
    
    @Override
    @JsonValue
    public String getAuthority() {
        return authority;
    }
    
    public class Const {
        public static final String ADMIN = "ROLE_ADMIN";
        public static final String USER = "ROLE_USER";
        public static final String CSR = "ROLE_CSR";
    }
}
```

### 6.5.1 역할과 권한

- **hasRole('ADMIN')**: 스프링 시큐리티가 내부적으로 ROLE_ 접두사를 추가하여 검사함.
- **hasAuthority('ADMIN')**: 접두사 없이 문자열 그대로 권한을 검사함.

**JWT 권한 변환기**
```java
private Converter<Jwt, AbstractAuthenticationToken> getJwtAuthenticationConverter() {
    JwtGrantedAuthoritiesConverter authorityConverter = new JwtGrantedAuthoritiesConverter();
    authorityConverter.setAuthorityPrefix(AUTHORITY_PREFIX);
    authorityConverter.setAuthoritiesClaimName(ROLE_CLAIM);
    JwtAuthenticationConverter converter = new JwtAuthenticationConverter();
    converter.setJwtGrantedAuthoritiesConverter(authorityConverter);
    return converter;
}
```

OAuth 2.0 리소스 서버는 기본적으로 scope 클레임을 사용하지만, 위 변환기를 통해 roles 클레임을 사용하도록 변경할 수 있음.

### 6.5.2 메소드 수준의 역할 기반 접근 제어
```java
@Configuration
@EnableWebSecurity
@EnableGlobalMethodSecurity(prePostEnabled = true)
public class SecurityConfig {
    // ...
}
```
```java
@PreAuthorize("hasRole('" + Const.ADMIN + "')")
@Override
public ResponseEntity<Void> deleteAddressesById(String id) {
    service.deleteAddressesById(id);
    return accepted().build();
}
```

### 6.5.3 SpEL 표현식

- **hasRole(String role)**: 특정 역할을 가진 경우 true 반환
- **hasAnyRole(String... roles)**: 여러 역할 중 하나를 가진 경우 true 반환
- **hasAuthority(String authority)**: 특정 권한을 가진 경우 true 반환
- **hasAnyAuthority(String... authorities)**: 여러 권한 중 하나를 가진 경우 true 반환
- **permitAll**: true 반환
- **denyAll**: false 반환
- **isAnonymous()**: 현재 사용자가 익명인 경우 true 반환
- **isAuthenticated()**: 현재 사용자가 익명이 아닌 경우 true 반환
