2026.06.20

### 자바 코드는 어떻게 실행될까

**자바 코드**( .java)는 **컴파일러**(javac)를 통해 **바이트코드**(.class)로 변환됩니다.

이때 변환된 바이트코드를 **JVM**( Java Virtual Machine )이 읽어 운영체제와 상관없이 실행할 수 있도록 해줍니다.

“Write Once, Run Anywhere”

---

### JVM의 메모리 구조

JVM은 실행 중 사용하는 메모리를 영역별로 나누어 관리합니다.

1. **Method Area** (메서드 영역)
    - 클래스 정보
    - static 변수
    - 상수 풀
    
    등이 저장되는데, JVM 시작 시 클래스가 로드되면서 함께 생성되고, 모든 스레드가 공유합니다.
    
2. **Heap** (힙 영역)
    - new로 생성된 객체와 배열
    
    이 저장되는 공간으로, GC(Garbage Collection)의 관리 대상이 되는 영역이며, 모든 스레드가 공유합니다.
    
3. **Stack** (스택 영역)
    - 지역 변수
    - 매개변수
    - 연산 중간값
    
    등이 저장되는 공간으로 메서드 호출 시마다 생성되는 Stack Frame이 쌓이는 공간입니다. 스택 영역은 스레드마다 독립적으로 가집니다.
    
4. PC Register
    
    현재 스레드가 실행 중인 JVM 명령어의 주소를 저장합니다.
    
5. Native Method Stack
    
    자바가 아닌 네이티브 코드(C/C++) 를 실행하기 위한 스택입니다.
    

---

### Heap 영역의 세부 구조

Heap은 객체의 생존 기간에 따라 다시 영역을 나누어 관리합니다.

- Young Generation : 새로 생성된 객체가 위치합니다. Eden, Sruvivor 0, Survivor 1 영역으로 나뉩니다.
- Old Generation : Young 영역에서 오래 살아남은 객체가 이동합니다.

대부분의 객체는 생성된 후 금방 사용되지 않게 되는 특성 (Weak Generational Hypothesis)이 있어 이 구조를 활용해 GC의 효율을 높입니다.

---

### Garbage Collection (GC)

GC는 더 이상 참조되지 않는 객체를 자동으로 메모리에서 제거하는 과정입니다.

- **Minor GC**
    
    Young Generation에서 발생하는 GC입니다.
    
    Eden 영역이 가득 차면 살아있는 객체를 Survivor 영역으로 이동시키고, 일정 횟수 이상 살아남은 객체는 Old 영역으로 이동합니다.
    
- **Major GC (Full GC)**
    
    Old Generation에서 발생하는 GC입니다.
    
    Minor GC 보다 대상 영역이 넓어 시간이 오래 걸리고, 이 동안 어플리케이션이 멈추게 되는 Stop-The-World가 발생합니다.
    
- **Stop-The-World**
    
    GC를 수행하기 위해 GC 스레드를 제외한 모든 스레드가 일시 정지되는 현상입니다.
    
    Full GC가 잦거나 오래 걸리면 응답 지연으로 직결되기 때문에, GC 튜닝의 핵심은 이 Stop-The-World를 최소화 하는 것입니다.
    

---

### GC 알고리즘

JVM은 여러 GC 알고리즘을 제공하며, 자바 버전에 따라 기본값이 달라집니다.

대부분의 Spring Boot 애플리케이션은 G1 GC를 기본으로 사용하는데, G1 GC는 힙을 Region 단위로 나누어 관리합니다.

---

### Spring - IoC와 DI

Spring의 핵심 개념은 IoC(Inversion of Control, 제어의 역전)입니다.

기존에는 개발자가 직접 객체를 생성하고 의존성을 주입했다면, Spring에서는 객체의 생성과 생명주기 관리를 Spring 컨테이너가 대신 담당합니다.

이렇게 컨테이너가 필요한 의존성을 외부에서 주입해주는 방식을 DI(Dependency Injection, 의존성 주입)이라고 입니다.

```jsx
@Service
public class OrderService {

    private final PaymentService paymentService;

    // 생성자 주입
    public OrderService(PaymentService paymentService) {
        this.paymentService = paymentService;
    }
}
```

예시 코드는 생성자 주입 방식으로, DI 방식에는 생성자 주입, 필드 주입, 수정자 주입 방식이 있지만 불변성 보장과 순환참조 방지를 위해 생성자 주입 방식이 권장됩니다.

---

### Spring Bean의 생명주기

Spring 컨테이너가 관리하는 객체를 **Bean**이라고 합니다.

Bean의 생명주기는 다음과 같이 진행됩니다.

- 객체 생성 → 의존성 주입 → 초기화 콜백(@PostConstruct) → 사용 → 소멸 전 콜백(@PreDestry) → 소멸

```jsx
@Component
public class CacheManager {

    @PostConstruct
    public void init() {
        // 빈 생성과 의존성 주입이 끝난 후 호출
    }

    @PreDestroy
    public void destroy() {
        // 컨테이너 종료 직전 호출
    }
}
```

---

### Bean Scope

Bean이 생성되는 범위를 Scope라고 하며 기본적으로 Singleton 패턴을 사용합니다.

- Singleton : 컨테이너 내에 단 하나의 인스턴스만 생성됩니다
- Prototype : 요청할 때마다 새로운 인스턴스를 생성합니다
- Request : HTTP 요청마다 새로운 인스턴스를 생성합니다 (웹 환경에서만 사용)

Singleton Bean은 여러 요청에서 공유되기 때문에, 상태를 가지는 필드를 두면 동시성 문제가 발생할 수 있어 주의가 필요합니다.

---

### AOP(관점 지향 프로그래밍)

AOP는 핵심 비즈니스 로직과 부가 기능(로깅, 트랜잭션, 보안 등)을 분리하는 프로그래밍 방식입니다.

`@Transactional` 이 대표적인 AOP의 사례로, 실제로는 트랜잭션 처리 코드가 비즈니스 로직 메서드를 감싸는 형태로 동작합니다.

---

### AOP와 프록시(Proxy)

Spring은 AOP를 구현하기 위해 프록시 객체를 사용합니다.

`@Transactional` 이 붙은 Bean을 실제로 사용할 때, Spring은 원본 객체 대신 프록시 객체를 대신 등록합니다.

클라이언트 → 프록시 객체 → 부가기능 수행 → 실제 객체 → 비즈니스 로직 실행

프록시는 메서드 호출 전후로 트랜잭션 시작/커밋/롤백 같은 부가 기능을 수행한 뒤, 실제 객체의 메서드를 호출합니다.

---

### AOP 미적용

같은 클래스 안에서 `this.메서드()` 형태로 `@Transactional` 이 붙은 메서드를 호출하면 프록시를 거치지 않고 직접 호출되기 때문에 AOP가 적용되지 않게됩니다.

```jsx
@Service
public class OrderService {

    public void process() {
        this.save(); // 프록시를 거치지 않아 @Transactional 적용 안 됨
    }

    @Transactional
    public void save() {
        // ...
    }
}
```

이를 해결하기 위해서는 메서드를 별도의 클래스로 분리하거나, AopContext를 통해 프록시 객체를 직접 호출하는 방법을 사용합니다.

---

### 트랜잭션과 프록시

프록시가 메서드 호출 시마다 트랜잭션 시작 여부를 판단하기 때문에 같은 클래스 내부 호출에서는 새로운 트랜잭션이 시작되지 않습니다.

또한 Singleton Bean 안에서 Connection 같은 상태를 직접 필드로 들고 있으면 동시성 문제가 발생할 수 있어서 Connection Pool에서 매 요청마다 Connection을 빌려 쓰는 구조를 사용합니다.