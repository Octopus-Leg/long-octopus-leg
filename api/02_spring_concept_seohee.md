# 2. 스프링의 개념과 REST API

## 1️⃣ 스프링 패턴과 패러다임 이해하기
- **IoC (Inversion of Control, 제어의 역전)**
  - 객체의 생성과 생명주기 관리를 개발자가 아닌 프레임워크나 컴포넌트 같은 외부 소스가 대신 담당하는 것임.
  - 객체 지향 프로그래밍 접근 방식이 등장하면서 프레임워크들에 의존성 주입을 지원하는 IoC 컨테이너 패턴 구현이 보편화됨.
- **DI (Dependency Injection, 의존성 주입)**
  - IoC를 구현하는 구체적인 패턴 중 하나임.
  - 객체가 필요로 하는 의존 객체를 내부에서 직접 생성하지 않고, 외부에서 주입받는 방식임. 이를 통해 모듈 간의 결합도를 낮춤.
- **AOP (Aspect-Oriented Programming, 관점 지향 프로그래밍)**
  - OOP에서는 한 클래스에서 하나의 책임만 다루는 것이 좋은 관행이며, 이를 단일 책임 원칙(SRP)라고 함.
  - 하지만 프로그래밍 모델에서는 종종 하나 이상의 클래스에 걸쳐 있는 기능이나 함수가 필요함.
  - 로깅, 보안, 트랜잭션 관리 등 여러 모듈에 공통적으로 나타나는 `횡단 관심사`(Cross-cutting Concerns)를 핵심 비즈니스 로직에서 분리하여 모듈화하는 프로그래밍 기법임.
  - AOP를 사용하면 횡단 관심사를 추상화하고 캡슐화할 수 있고, 코드에 여러 부분에 걸쳐 관점 동작을 추가할 수 있음.
  - 또한, 코드를 쉽게 유지하고 확장할 수 있도록 횡단 관심사에 대한 코드를 모듈화할 수 있음.
---

## 2️⃣ IoC 컨테이너 이해하기
- 스프링 프레임워크의 핵심 컴포넌트로, bean의 생성, 구성, 생명주기 관리 및 의존성 주입을 담당함.
- `ApplicationContext`나 `BeanFactory` 인터페이스가 컨테이너 역할을 수행하며, 설정 정보(XML, 자바 클래스, 애노테이션)를 읽어 객체를 관리함.
---

## 3️⃣ Bean과 그 범위 정의하기
- **Bean**: 스프링 IoC 컨테이너가 관리하는 객체를 의미함.
- **@Configuration 애노테이션**: 클래스에 설정 코드가 포함돼 있음을 보여주는 클래스 수준 애노테이션으로, @Bean과 함께 사용되어 bean을 정의함.
- 일반적으로 bean의 이름은 첫 글자가 소문자인 클래스 이름인데 name 애트리뷰트를 사용해 이름과 여러 개의 별칭을 정의할 수 있음.
- **@ComponentScan 애노테이션**: `@Component`, `@Service`, `@Repository`, `@Controller` 등이 붙은 클래스를 자동으로 스캔하여 빈으로 등록함.
- **Bean의 범위 (Scope)**
  - **Singleton (기본값)**: 컨테이너 당 하나의 인스턴스만 생성되어 공유됨. 상태를 가지지 않는(Stateless) 빈에 적합함.
  - **Prototype**: 빈을 요청할 때마다 새로운 인스턴스가 생성됨.
  - **Request**: HTTP 요청마다 생성됨(Web 환경).
  - **Session**: HTTP 세션마다 생성됨(Web 환경).
  - **Application**: 응용 프로그램 범위마다(유효한 서블릿 컨텍스트 라이프 사이클) 생성됨(Web 환경).
  - **Websocket**: WebSocket 세션에 대해 단일 인스턴스가 생성됨.
---

## 4️⃣ 자바를 사용하여 bean 설정
- XML 대신 자바 클래스에 `@Configuration`과 `@Bean`을 사용하여 빈을 정의하는 방식임.
- **@Import 애노테이션**: 다른 설정 클래스(`@Configuration`)를 현재 설정 클래스로 가져와서 병합할 때 사용함. 설정을 모듈화할 때 유용함.
스프링 부트는 자동 설정을 사용하기 떄문에 `@Import`를 사용할 필요가 없음.
- **@DependsOn 애노테이션**: 특정 빈이 다른 빈보다 먼저 생성되어야 할 때, 초기화 순서를 강제로 지정함.
---

## 5️⃣ DI 코딩 방법
- **생성자로 의존성 정의**
    ```java
    @Service
      // 롬복 사용 시: @RequiredArgsConstructor (아래 생성자 코드를 자동으로 생성해줌)
      public class OrderService {
      
          private final MemberRepository memberRepository;
          private final DiscountPolicy discountPolicy;
          
          @Autowired // 생성자가 하나라면 생략 가능
          public OrderService(MemberRepository memberRepository, DiscountPolicy discountPolicy) {
              this.memberRepository = memberRepository;
              this.discountPolicy = discountPolicy;
          }
      }
    ```
  - 생성자를 통해 의존성을 주입받음.
  - 필수 의존성을 강제할 수 있고, 불변성을 보장하며, 순수 자바 코드로 테스트하기 용이함. 롬복의 `@RequiredArgsConstructor`와 함께 많이 사용됨.
- **설정자(Setter) 메소드로 의존성 정의**
    ```java
    @Service
    public class OrderService {
    
        private MemberRepository memberRepository;
        
        @Autowired
        public void setMemberRepository(MemberRepository memberRepository) {
            this.memberRepository = memberRepository;
        }
    }
    ```
  - Setter 메소드에 `@Autowired`를 붙여 주입받음.
  - 선택적인(Optional) 의존성이나 변경 가능성이 있는 의존성에 사용함.
- **클래스 프로퍼티(Field)를 사용한 의존성 정의**
    ```java
    @Service
    public class OrderService {
    
        @Autowired
        private MemberRepository memberRepository;
    
        @Autowired
        private DiscountPolicy discountPolicy;
    }
    ```
  - 필드 위에 바로 `@Autowired`를 붙이는 방식.
  - 코드가 간결하지만, 외부에서 의존성을 주입하기 어려워 테스트가 힘들고 DI 컨테이너와 강하게 결합되는 단점이 있어 권장되지 않음.
---

## 6️⃣ 애노테이션을 사용하여 bean의 메타데이터 설정
- **`@Autowired` 사용 방법**: 스프링이 타입(Type)을 기준으로 알맞은 빈을 찾아 자동으로 주입함. 이 방식은 리플렉션을 사용해서 다른 주입 접근 방식보다 비용이 많이 듦.
  - **리플렉션**: 구체적인 클래스 타입을 알지 못해도, 런타임에 클래스의 내부 정보(필드, 메소드 등)를 동적으로 분석하고 조작할 수 있게 해주는 자바 API임.
- **타입별 일치(Match by type)**: 주입하려는 타입과 일치하는 빈이 하나면 바로 주입됨.
- **한정자별 일치(Match by qualifier)**: 같은 타입의 빈이 여러 개일 때, `@Qualifier("이름")`을 사용하여 특정 빈을 명시적으로 지정함.
- **이름으로 일치(Match by name)**: `@Resource` 등을 사용하거나, 파라미터 이름을 빈 이름과 일치시켜 주입받을 수도 있음.
  - **`@Resource`** : 실행 경로 순서가 이름, 타입, 한정자별로 지정됨.(반면에 `@Autowired`랑 `@Inject`은 타입별, 한정자별, 이름별 순서임.)
- **`@Primary`의 목적은 무엇일까?**: 같은 타입의 빈이 여러 개일 때, 우선순위를 가장 높게 설정하여 기본적으로 주입될 빈을 지정함. (`@Qualifier`보다 우선순위가 낮음)
- **`@Value는 언제 사용할까?`**: `application.properties`, `application.yml` 등의 외부 설정 파일에 있는 값을 필드에 주입할 때 사용함.
---

## 7️⃣ AOP용 코드 작성
- `@Aspect` 애노테이션을 사용하여 관점(Aspect)을 정의함.
- **Pointcut**: 부가 기능이 적용될 위치(메소드 등)를 지정함.
- **Advice**: 실제 수행될 부가 기능 코드(Log 등). 실행 시점(`@Before`, `@After`, `@Around`)을 지정할 수 있음.
- **Join Point**: 클라이언트가 호출하는 모든 비즈니스 메소드, 즉 **Advice가 적용될 수 있는 모든 지점(후보)** 을 의미함.
  - 스프링 AOP는 프록시 방식을 사용하므로 조인 포인트는 항상 **메소드 실행 지점**으로 제한됨.
  - `JoinPoint` 파라미터를 통해 실행 중인 **대상 객체(Target)**, **메소드 인수**, **프록시 객체** 등의 정보를 캡처하고 접근할 수 있음.
- 핵심 로직을 수정하지 않고도 부가 기능을 유연하게 추가/삭제할 수 있음.

```java
// 1. 커스텀 애노테이션 정의
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface TimeMonitor {}

// 2. Aspect 정의 (부가 기능 구현체)
@Aspect
@Component
public class TimeMonitorAspect {

    // @Around: 메소드 실행 전후를 모두 제어할 수 있는 강력한 Advice
    // Pointcut: @TimeMonitor 애노테이션이 붙은 메소드를 대상으로 지정
    @Around("@annotation(com.packt.modern.api.TimeMonitor)")
    public Object logTime(ProceedingJoinPoint joinPoint) throws Throwable {

        // 실행 전 (Before)
        long start = System.currentTimeMillis();
        // joinPoint.proceed(): 실제 타겟 메소드(performSomeTask)를 실행
        Object proceed = joinPoint.proceed();
        // 실행 후 (After)
        long executionTime = System.currentTimeMillis() - start;
        // joinPoint.getSignature(): 호출된 메소드의 정보(이름, 리턴타입 등)를 가져옴
        System.out.println(joinPoint.getSignature() + " takes: " + executionTime + " ms");

        return proceed; // 메소드 실행 결과 반환
    }
}

// 3. 적용 예시
class Test {
    @TimeMonitor // 이 애노테이션이 붙어 있으므로 Aspect가 적용됨 (프록시가 동작)
    public void performSomeTask() {
        // 핵심 비즈니스 로직
    }
}
```
---

## 8️⃣ 스프링 부트를 사용하는 이유
- **자동 설정**: 클래스패스에 있는 라이브러리를 감지하여 자동으로 빈을 설정해 줌으로써 복잡한 설정을 최소화함.
- **내장 서버**: Tomcat, Jetty 같은 WAS를 내장하고 있어 별도의 서버 설치 없이 `jar` 실행만으로 웹 애플리케이션을 구동할 수 있음.
- **Starter 종속성**: 호환되는 라이브러리 버전들을 묶어놓은 스타터 패키지를 제공하여 의존성 관리가 매우 편리함 (예: `spring-boot-starter-web`).
- 스프링 부트를 쓰면 설정은 스프링 이니셜라이저가 알아서 수행해줘서 개발자는 비즈니스 로직에 집중할 수 있음.
---

## 9️⃣ 서블릿 디스패처의 중요성의 이해
- **DispatcherServlet(프론트 컨트롤러 역할)**
  - 모든 HTTP 요청을 가장 먼저 받아 적절한 핸들러(Controller)에게 위임하고, 결과를 받아 응답을 생성하는 중앙 집중식 처리자임.
  - 공통적인 처리(인코딩, 에러 처리 등)를 한곳에서 관리할 수 있게 해주며, 개발자는 비즈니스 로직이 담긴 컨트롤러 구현에만 집중할 수 있게 함.
- **스프링 MVC의 사용자 요청 흐름**
  > 1. **요청 수신**: 사용자가 HTTP 요청을 보내면 `DispatcherServlet`이 이를 가장 먼저 수신함.
  > 2. **핸들러 조회**: `DispatcherServlet`은 `HandlerMapping`에게 요청 정보를 전달하여, 해당 URI를 처리할 컨트롤러(핸들러)를 찾음.
  > 3. **핸들러 어댑터 조회**: `DispatcherServlet`은 찾은 컨트롤러를 실행할 수 있는 `HandlerAdapter`를 찾음.
  > 4. **핸들러 실행**: `HandlerAdapter`가 실제 컨트롤러의 적절한 메소드를 호출함.
  > 5. **비즈니스 로직 수행**: 컨트롤러는 관련 비즈니스 로직을 실행하고 응답 데이터를 구성함.
  > 6. **변환 및 응답 (마샬링/직렬화)**: 스프링은 자바 객체와 JSON/XML 간의 변환을 위해 마샬링/언마샬링 과정을 수행함. (현대적인 REST API에서는 이를 주로 **직렬화/역직렬화**라고 표현함)