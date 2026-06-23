2026.06.23

### JPA를 쓰는 이유

관계형 데이터베이스는 테이블과 행으로 데이터를 다루고, 자바는 객체와 클래스로 데이터를 다룹니다.

이러한 패러다임 차이를 패러다임 불일치라고 하며, 개발자가 직접 SQL을 작성해 불일치를 해소하는 것은 반복적이고 번거로운 작업이 됩니다.

이때 등장한 것이 JPA ( Java Persistence API )로, JPA는 자바 객체와 DB 테이블 간의 매핑을 자동으로 처리해주는 ORM ( Object-Relational Mapping ) 표준 인터페이스 입니다.

개발자들은 JPA를 통해 SQL대신 객체를 다루는 코드를 작성하고, JPA가 적절한 SQL을 생성해 실행하게 됩니다.

---

### 영속 컨텍스트

일반 자바 코드에서는 객체를 `new` 로 생성하면 끝이나고, 개발자가 직접 DB에 이를 저장하는 SQL을 실행해야 합니다.

하지만 JPA에서는 객체를 직접 관리하는 공간을 별도로 두고, 그 공간 안에 들어온 객체는 JPA가 책임지고 DB와 동기화하게 됩니다.

이때 객체를 관리하는 공간이 영속성 컨텍스트입니다.

---

### 영속성 컨텍스트의 생명주기

영속성 컨텍스트는 독립적으로 존재하지 않고 항상 트랜잭션과 함께 생성되고 함께 사라집니다.

- 트랜잭션 시작

→ 영속성 컨텍스트 생성

→ 엔티티 조회/저장/수정

→ 트랜잭션 커밋

→ 변경사항을 DB에 반영

→ 영속성 컨텍스트 종료

Spring에서는 `@Transactional` 이 붙은 메서드가 시작될 때 영속성 컨텍스트가 생성되며 메서드가 끝날 때 함께 닫힙니다.

---

### 엔티티의 상태

영속성 컨텍스트와의 관계에 따라 엔티티는 항상 4가지 상태 중 하나가 됩니다.

1. **비영속 (New)**

영속성 컨텍스트와 아무 관계가 없는 순수한 자바 객체 상태입니다. `new` 로 막 생성된 상태로, DB와는 아무 연결이 없습니다.

1. **영속 (Managed)**

영속성 컨텍스트 안에 들어간 상태입니다. 이 상태의 엔티티는 JPA가 관리하게 되는데, `persist()`를 호출하거나 DB에서 조회(`findById()` )등을 하게 되면 영속 상태가 됩니다.

영속 상태의 엔티티를 수정하면 개발자가 별도의 저장 명령을 내리지 않아도 트랜잭션 커밋 시점에 자동으로 UPDATE 가 실행되는데, 이를 변경 감지라고 합니다.

1. **준영속 (Detached)**

한 번 영속 상태였다가 영속성 컨텍스트에서 분리된 상태입니다. 이 상태의 엔티티를 수정해도 DB에 반영되지 않습니다.

트랜잭션이 끝난 뒤 엔티티 객체를 계속 들고 있으면 준영속 상태가 됩니다.

1. **삭제 (Removed)**

`remove()`가 호출된 상태입니다. 트랜잭션 커밋 시 DELETE 쿼리가 실행됩니다.

---

### 영속성 컨텍스트 핵심 기능

- **1차 캐시**

영속성 컨텍스트 내부에는 1차 캐시가 존재합니다.

같은 트랜잭션 안에서 동일한 엔티티를 두 번 조회하면, 두 번째 조회는 DB가 아닌 1차 캐시에서 데이터를 꺼내옵니다.

```jsx
@Transactional
public void example() {
    User user1 = userRepository.findById(1L); // DB 조회 (SELECT 발생)
    User user2 = userRepository.findById(1L); // 1차 캐시에서 반환 (SELECT 없음)

    System.out.println(user1 == user2); // true - 동일 인스턴스
}
```

- **변경 감지 (Dirty Checking)**

영속 상태의 엔티티는 최초 조회 시점의 스냅샷을 저장합니다.

트랜잭션이 커밋될 때 현재의 상태와 스냅샷을 비교해서 변경된 내용이 있다면 자동으로 UPDATE 쿼리를 실행합니다.

```jsx
@Transactional
public void update(Long id, String newName) {
    User user = userRepository.findById(id).orElseThrow();
    user.setName(newName); // 별도의 save() 호출 없이도 UPDATE 발생
}
```

여기서 `save()` 를 호출하지 않아도 변경 감지로 인해 자동으로 변경 사항이 반영되게 됩니다.

- **쓰기 지연 (Write-Behind)**

영속성 컨텍스트는 SQL을 즉시 실행하지 않고 내부 쿼리 저장소에 모아두었다가 트랜잭션 커밋 시점에 한 번에 실행합니다.

```jsx
@Transactional
public void bulkInsert() {
    em.persist(new User("A")); // INSERT 즉시 실행 X, 저장소에 보관
    em.persist(new User("B")); // INSERT 즉시 실행 X, 저장소에 보관
    // 트랜잭션 커밋 시점에 INSERT 2개 한 번에 실행
}
```

---

### Entity

엔티티는 DB 테이블과 매핑되는 자바 클래스입니다.

```jsx
@Entity
@Table(name = "users")
public class User {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "user_name", nullable = false, length = 50)
    private String name;

    @Column(name = "email", unique = true)
    private String email;
}
```

엔티티 설정 시 사용되는 주요 어노테이션들을 보면

- `@Entity` : 이 클래스가 JPA가 관리하는 엔티티임을 선언합니다.
- `@Table` : 매핑할 테이블 이름을 지정합니다. 생략 시 클래스 이름을 테이블 이름으로 사용합니다.
- `@Id` : 기본키(PK)를 지정합니다.
- `@GeneratedValue` : PK 생성 전략을 지정합니다.
- `@Column` : 컬럼 매핑 정보를 지정합니다. 생략 시 필드명을 컬럼명으로 사용합니다.

이때 `@GeneratedValue` 전략에 대해서 조금 더 자세히 살펴보자면

- `IDENTITY` : DB의 AUTO_INCREMENT에 전략 위임 (MySQL)
- `SEQUENCE` : DB 시퀀스 사용(PostgreSQL)
- `TABLE` : 키 생성용 테이블 별도로 사용
- `AUTO` : DB방언에 따라 자동 선택

IDENTITY 전략은 INSERT 후 DB가 생성한 ID를 알 수 있기 때문에 쓰기 지연이 동작하지 않고 persist() 시점에 즉시 INSERT가 실행됩니다.

---

### 연관관계 매핑

연관관계 매핑은 테이블의 외래키를 자바 객체의 참조로 다루는 것입니다.

- **단방향 vs 양방향**

```jsx
// 단방향: Order에서만 User를 참조할 수 있음
@Entity
public class Order {
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "user_id")
    private User user;
}

// 양방향: User에서도 Order 목록을 참조할 수 있음
@Entity
public class User {
    @OneToMany(mappedBy = "user")
    private List<Order> orders = new ArrayList<>();
}
```

양방향 연관관계에서 실제 외래키를 관리하는 쪽을 연관관계의 주인이라고 합니다.

`@JoinColumn`을 가진 쪽이 주인이 되고, `mappedBy`를 사용한 쪽은 읽기 전용이 됩니다.

이 때 주인이 아닌 쪽에 값을 넣어도 DB에 반영되지 않기 때문에 양방향 관계에서는 양쪽 모두에 값을 설정하는 것이 안전합니다.

```jsx
// 연관관계 편의 메서드 - User 엔티티에 작성
public void addOrder(Order order) {
    orders.add(order);
    order.setUser(this); // 주인 쪽에도 함께 설정
}
```

- **연관관계 종류**
    - `@ManyToOne` - N:1 관계
    - `@OneToMany` - 1:N 관계
    - `@OneToOne` - 1:1 관계
    - `@ManyToMany` - N:M 관계

`@ManyToMany` 관계에서는 중간 테이블을 JPA가 자동 생성하게 되는데 추가 컬럼을 넣을 수 없고 관리하기 어렵기 때문에 보통 중간 엔티티를 직접 만들어서 `@ManyToOne` + `@OneToMany` 로 풀어서 사용합니다.

---

### Spring Data JPA와 JpaRepository

Spring Data JPA의 `JpaRepository`를 상속하면 기본 CRUD 메서드를 자동으로 제공받습니다.

```jsx
public interface UserRepository extends JpaRepository<User, Long> {
    // 기본 제공: save(), findById(), findAll(), delete() 등
}
```

`JpaRepository<T, ID>`에서 T는 엔티티 타입, ID는 기본키 타입입니다.

- **save() 내부 동작**

`save()`는 단순한 INSERT가 아닙니다.

```jsx
// Spring Data JPA의 SimpleJpaRepository 내부 코드 (간략화)
public <S extends T> S save(S entity) {
    if (entityInformation.isNew(entity)) {
        em.persist(entity); // 새 엔티티 → INSERT
        return entity;
    } else {
        return em.merge(entity); // 기존 엔티티 → UPDATE
    }
}
```

`isNew()` 판단은 기본적으로 `@Id` 필드가 `null`인지로 결정됩니다. ID가 있으면 `merge()`를 호출하는데, `merge()`는 DB에서 해당 엔티티를 다시 조회한 뒤 병합하기 때문에 불필요한 SELECT가 발생할 수 있습니다.

따라서 트랜잭션 안에서 이미 영속 상태인 엔티티는 변경 감지를 활용하고, `save()`를 불필요하게 반복 호출하지 않는 것이 좋습니다.

---

### 메서드 쿼리 (Query Method)

Spring Data JPA는 메서드 이름만으로 쿼리를 자동 생성합니다.

```jsx
public interface UserRepository extends JpaRepository<User, Long> {

    // SELECT * FROM users WHERE name = ?
    List<User> findByName(String name);

    // SELECT * FROM users WHERE name = ? AND email = ?
    Optional<User> findByNameAndEmail(String name, String email);

    // SELECT * FROM users WHERE name LIKE '%?%'
    List<User> findByNameContaining(String keyword);

    // SELECT * FROM users WHERE created_at > ? ORDER BY created_at DESC
    List<User> findByCreatedAtAfterOrderByCreatedAtDesc(LocalDateTime dateTime);

    // SELECT COUNT(*) FROM users WHERE name = ?
    long countByName(String name);

    // DELETE FROM users WHERE name = ?
    void deleteByName(String name);
}
```

자주 사용하는 키워드는 아래와 같습니다.

| 키워드 | SQL | 예시 |
| --- | --- | --- |
| `findBy` | WHERE | `findByName` |
| `And`, `Or` | AND, OR | `findByNameAndEmail` |
| `Containing` | LIKE '%?%' | `findByNameContaining` |
| `StartingWith` | LIKE '?%' | `findByNameStartingWith` |
| `Between` | BETWEEN | `findByAgeBetween` |
| `LessThan`, `GreaterThan` | <, > | `findByAgeLessThan` |
| `OrderBy` | ORDER BY | `findByNameOrderByCreatedAtDesc` |
| `Exists` | EXISTS | `existsByEmail` |

메서드 이름이 너무 길어지거나 복잡한 조건이 필요한 경우 `@Query`를 사용합니다.

---

### JPQL과 @Query

메서드 쿼리로 표현하기 어려운 복잡한 쿼리는 JPQL(Java Persistence Query Language)을 직접 작성합니다.

JPQL은 SQL과 유사하지만, 테이블이 아닌 엔티티 클래스와 필드를 기준으로 작성합니다.

```jsx
public interface UserRepository extends JpaRepository<User, Long> {

    // JPQL - 엔티티명(User)과 필드명(name)을 사용
    @Query("SELECT u FROM User u WHERE u.name = :name AND u.email = :email")
    Optional<User> findByNameAndEmail(@Param("name") String name,
                                      @Param("email") String email);

    // fetch join - 연관 엔티티를 함께 조회
    @Query("SELECT u FROM User u JOIN FETCH u.orders WHERE u.id = :id")
    Optional<User> findByIdWithOrders(@Param("id") Long id);

    // 네이티브 SQL - 실제 테이블명과 컬럼명 사용 (nativeQuery = true)
    @Query(value = "SELECT * FROM users WHERE email = :email", nativeQuery = true)
    Optional<User> findByEmailNative(@Param("email") String email);
}
```

JPQL은 DB 방언(MySQL, PostgreSQL 등)에 종속되지 않습니다. Hibernate가 DB에 맞는 SQL로 변환해서 실행하기 때문입니다.

반면 `nativeQuery = true`를 사용하면 특정 DB의 SQL 문법을 그대로 쓸 수 있지만, DB가 바뀌면 쿼리도 변경해야 합니다.

---

### Pageable과 페이징 처리

Spring Data JPA는 페이징과 정렬을 `Pageable`로 간단하게 처리할 수 있습니다.

```jsx
public interface UserRepository extends JpaRepository<User, Long> {
    Page<User> findByName(String name, Pageable pageable);
}
```

```jsx
@Service
public class UserService {

    public Page<User> getUsers(String name, int page, int size) {
        Pageable pageable = PageRequest.of(page, size, Sort.by("createdAt").descending());
        return userRepository.findByName(name, pageable);
    }
}
```

`Page<T>`는 데이터 목록 외에 전체 건수, 전체 페이지 수, 현재 페이지 등의 정보를 함께 제공합니다.

`Page`는 count 쿼리를 자동으로 실행하고, 단순히 다음 페이지 여부만 필요한 경우에는 count 쿼리가 없는 `Slice<T>`를 사용하는 것이 성능상 유리합니다.

---

### 지연 로딩과 즉시 로딩

이 둘은 JPA에서 연관관계에 있는 엔티티를 언제 조회할지 결정하는 전략입니다.

- **EAGER (즉시 로딩)**

엔티티를 조회할 때 연관된 엔티티도 즉시 함께 조회합니다.

```jsx
@ManyToOne(fetch = FetchType.EAGER)
private Team team;
```

- **LAZY (지연 로딩)**

연관된 엔티티는 실제로 접근하는 시점에 조회합니다. 그 전까지는 프록시 객체로 대체하게 됩니다.

```jsx
@ManyToOne(fetch = FetchType.LAZY)
private Team team;
```

```jsx
User user = userRepository.findById(1L);
// 이 시점에는 team은 프록시 객체 (SELECT 없음)

String teamName = user.getTeam().getName();
// 이 시점에 team SELECT 발생
```

실무에서는 기본적으로 지연 로딩을 사용하고, 필요한 경우에만 `fetch join` 으로 함께 조회하는 방식이 권장됩니다.

즉시 로딩은 예상치 못한 쿼리가 발생하기 쉽고 N+1 문제의 원인이 될 수 있습니다.

---

### N+1 문제

N+1 문제는 JPA를 사용할때 가장 흔하게 발생하는 성능문제입니다.

```jsx
// User : Team = N : 1 관계

List<User> users = userRepository.findAll();
// SELECT * FROM user (1번 실행)

for (User user : users) {
    System.out.println(user.getTeam().getName());
    // 각 user마다 SELECT * FROM team WHERE id = ? 발생
    // user가 100명이면 team SELECT가 100번 실행
}
```

위의 예시처럼 1건을 조회하는 쿼리를 실행했을 때 연관된 엔티티를 조회하기 위해서 N개의 추가 쿼리가 발생하는 현상입니다.

이를 해결하기 위한 방법들을 3가지 살펴보겠습니다.

1. **fetch join**

JPQL에서 `JOIN FETCH` 를 사용해 연관 엔티티를 한 번의 쿼리로 함께 조회할 수 있습니다.

```jsx
@Query("SELECT u FROM User u JOIN FETCH u.team")
List<User> findAllWithTeam();
```

이 경우

`SELECT u.*, t.* FROM user u INNER JOIN team t ON u.team_id = t.id` 형태의 쿼리 1번으로 해결할 수 있습니다.

1. **@EntityGraph**

fetch join을 어노테이션으로 선언하는 방식입니다.

```jsx
@EntityGraph(attributePaths = {"team"})
List<User> findAll();
```

1. **@BatchSize**

지연 로딩을 유지하면서, N번의 쿼리 대신 IN 절을 활용해 한 번에 조회할 수 있습니다.

```jsx
@BatchSize(size = 100)
@OneToMany(mappedBy = "user", fetch = FetchType.LAZY)
private List<Order> orders;
```

이 경우`SELECT * FROM order WHERE user_id IN (1, 2, 3, ...)` 형태로 최대 100개씩 묶어서 조회할 수 있습니다.

---

### fetch join 주의사항

fetch join이 가장 편하게 사용되지만 주의해야할 점들이 있습니다.

- **컬렉션 fetch join과 페이징**

`@OneToMany`관계에서 fetch join과 페이징 (limit, offset)을 함께 사용하면 문제가 발생합니다.

JPA는 컬렉션 fetch join 시 페이징을 메모리에서 처리하며, 이 때 경고가 발생하고 전체 데이터를 메모리에 올린 뒤 잘라냅니다.

그래서 이 경우에는 fetch join 대신 `@BatchSize`를 사용하는 것이 적합합니다.

- **둘 이상의 컬렉션 fetch join 불가**

`@OneToMany`관계를 두 개 이상 fetch join 하면 MultipleBagFetchException이 발생합니다.

그래서 이 경우에도 `@BatchSize`를 사용해 해결합니다.