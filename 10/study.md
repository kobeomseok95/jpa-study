객체지향 쿼리 언어(JPQL)
==========================

## JPQL 특징
- 테이블이 아닌 객체를 대상으로 검색하는 객체지향 쿼리
- SQL을 추상화해서 특정 DB SQL에 의존하지 않음
- 대소문자 구분(JPQL 키워드는 구분하지 않음)
- 별칭 필수!
- JPQL에서 사용하는 from 뒤의 대상은 엔티티명이다! > 클래스명이 아니다!

### TypedQuery, Query
```java
em.createQuery("select m from Member m", Member.class); // TypedQuery > 두번째 인자에 클래스를 설정해주었기 때문

em.createQuery("select m.age, m.username from Member m"); // Query > 첫번째 처럼 타입을 지정하지 않았다.
// Query에서 getResultList() 호출 시 결과가 하나일 경우 Object, 둘 이상일 경우 Object[] 반환
```

### getResultList(), getSingleResult()
- getResultList : 결과를 컬렉션으로 반환한다.(결과가 없더라도 빈 컬렉션 반환)
- getSingleResult : 결과가 하나여야만 한다! 없거나 둘 이상일 경우 예외 발생(NoResultException, NonUniqueResultException)

### 파라미터 바인딩
- 이름 기준으로 사용하자!
```java
String usernameParameter = "하이";

List<Member> members = em.createQuery("select m from Member m where m.username = :username", Member.class)
  .setParameter("username", usernameParameter)  // 여기!
  .getResultList();
```

### 프로젝션
- Select 절에 조회할 대상을 지정하는 것

- 엔티티 프로젝션
  - 엔티티를 프로젝션 대상으로 사용하는 경우 대상이 된 엔티티는 영속성 컨텍스트에서 관리가 된다.
  ```java
    select m from Member m
    select m.team from Member m
  ```
  
- 임베디드 타입 프로젝션
  - 엔티티와 비슷하게 사용되지만 조회의 시작점이 될 수 없다.
  - 엔티티가 아닌 값 타입이라 영속성 컨텍스트에서 관리되지 않는다.
  ```java
    select o.address from Order o
    // select a from Address a > 불가능!
  ```

- 스칼라 타입 프로젝션
  - 숫자, 날짜, 문자와 같은 기본 데이터 타입들을 조회한다.
  - 프로젝션에 여러 값을 조회할 경우 Query를 사용해야한다.
  ```java
    List<String> usernames = em.createQuery("select m.username from Member m", String.class)
      .getResultList();
    
    ...
    
    List<Object[]> list = em.createQuery("select o.member, o.product, o.orderAmount from Order o")
      .getResultList();
      
    for (Object[] row : list){
      Member member = (Member)row[0];
      Product product = (Product)row[1];
      int orderAmount = (Integer)row[2];
    }
  ```

- new 명령어 
  - DTO를 활용하자!
  - new 키워드 옆에 패키지명을 기재해야 하고, DTO에 생성자가 있어야 한다.
  ```java
    List<UserDTO> list = em.createQuery("select new jpql.UserDTO(m.username, m.age) from Member m", UserDTO.class)
                            .getResultList();
  ```

### 페이징 API
```java
List<UserDTO> list = em.createQuery("select new jpql.UserDTO(m.username, m.age) from Member m", UserDTO.class)
                            .setFirstResult(0)  //조회 시작 위치
                            .setMaxResults(10)  //조회할 데이터 수
                            .getResultList();
```

### 집합과 정렬
- count : 결과 수를 반환한다. Return : Long
- max, min : 최대, 최소 값을 구한다. 문자, 숫자, 날짜 등에 활용
- avg : 평균값을 구하며 숫자타입만 사용 가능하다. Return : Double
- sum : 합을 구하며 숫자타입만 사용 가능하다. Return : (정수합, Long), (소수합, Double), (BigInteger합, BigInteger), (BigDemical합, BigDemical)

- 참고사항
  - NULL값은 무시한다.(통계에 잡히지 않음!)
  - distinct를 집합 함수 안에 사용하여 중복된 값을 제거하고 나서 집합을 구할 수 있다.
  - 만약 값이 없는 경우 count : 0 반환, 나머지 함수는 null 반환
  - distinct를 count에서 사용할 때 임베디드 타입은 지원하지 않음
- 정렬
  - 기존 SQL과 마찬가지!

### JPQL JOIN
- 내부 조인
```java
select m from Member m [inner] join m.team t
```
- 기존의 SQL과 차이점 : JPQL은 연관 필드를 사용한다.
- **연관 필드 : 다른 엔티티와 연관관계를 가지기 위해 사용하는 필드**, 해당 예시에서는 m.team

- 외부 조인
```java
select m from Member m left [outer] join m.team t
```

- 컬렉션 조인
  - 일대다, 다대다 관계처럼 컬렉션을 사용하는 곳에 조인하는 것!
  - 위의 예시에서 '회원 -> 팀' 으로의 조인은 **단일 값 연관 필드**를 사용한다.(m.team)
  - '팀 -> 회원'처럼 반대의 경우에는 **컬렉션 값 연관 필드**를 사용한다.(t.members)
  ```java
  select t, m from Team t left join t.members m
  ```

- 세타 조인과 JOIN ON절
  - 세타 조인은 inner join, JOIN ON절은 외부 조인에서 사용
  - 세타 조인을 사용할 경우 연관 관계가 전혀 없는 엔티티도 조인할 수 있다는 점!
  - 왜 JOIN ON절은 외부 조인에서만 사용하는가?
    - 내부 조인일 경우 외래키 값이 NULL인 경우는 제외하기 때문에 내부 조인에서 JOIN ON절을 사용할 경우 어차피 세타조인과 결과가 같다.

### Fetch Join
- JPQL에서 성능 최적화를 위해 제공하는 기능
  #### 1. Entity Fetch Join
    - 해당 페치 조인을 통해 조회하는 엔티티와 연관된 엔티티도 함께 조회한다.
    ```java
      select m from Member m join fetch m.team
    ```
    - 만약 위의 예제 코드에서 지연로딩을 설정했다고 가정하면, 연관된 팀 엔티티는 프록시 객체!
    - 하지만 fetch join을 통해 프록시가 아닌 실제 객체를 가져왔으므로 영속성 컨텍스트에서도 관리된다.
  #### 2. Collection Fetch Join
    - 일대다 관계의 컬렉션을 조회할 경우 '다'에 해당하는 데이터의 크기에 영향을 받아 결과가 증가하여 '일'에 해당하는 데이터가 중복하여 나오게 된다.
    ```java
      select t from Team t join fetch t.members
    ```
    - DISTINCT 키워드를 이용하여 중복된 '일'에 해당하는 데이터들을 제거하여 결과를 얻을 수 있다.
  
  #### Join VS Fetch Join
    - **JPQL은 결과 반환시 연관관계를 신경쓰지 않는다!!!**
    ```java
      1. select t from Team t join t.members
      2. select t from Team t join fetch t.members
    ```
    - 1. 해당 JPQL에서 members를 전혀 조회하지 않고 select 절에 프로젝션 한 Team만 반환한다.
      - 만약 지연로딩으로 설정할 경우 해당 members는 컬렉션 래퍼(다대일의 경우 프록시)로 반환한다.
      - 즉시로딩으로 설정할 경우 해당 members를 찾기 위해 쿼리를 한번 더 실행한다.(N+1 문제)
    - 2. Fetch Join을 사용하여 연관된 엔티티까지 함께 조회했다.
  
  #### 한계
    - 글로벌 로딩 전략(@XXXToMany(fetch = FetchType.LAZY))보다 Fetch Join이 우선권을 가진다.
    - 가급적 글로벌 로딩 전략은 지연로딩으로 설정하고 최적화가 필요할 때 Fetch Join 전략을 적용하자!
    - Fetch Join은 별칭을 줄 수 없다.
    - 둘 이상의 컬렉션을 Fetch할 수 없다.
    - **페이징 API를 사용할 수 없다.**
      - 페이징을 처리하기 전의 모든 데이터가 메모리에 들어가고 해당 페이징을 처리하기 때문에 경고 발생!
      
  #### 그럼 언제 써야하나?
    - 객체 그래프를 유지할 때 사용하면 효과적!
    
### 경로 표현식
- 상태 필드 : 단순히 값을 저장하기 위한 필드
- 연관 필드 : 연관관계를 위한 필드, 임베디드 타입 포함
  - 단일 값 연관필드 : 대상이 엔티티(@ManyToOne, @OneToOne)
  - 컬렉션 값 연관필드 : 대상이 컬렉션(@OneToMany~~, @ManyToMany~~)
- 특징
  - 상태 필드 : 경로 탐색의 끝.
  - 단일 값 연관 필드 : 묵시적 내부 조인 발생, 계속 탐색 가능
  - 컬렉션 값 연관 필드 : 묵시적 내부 조인 발생, 계속 탐색 불가
    - 컬렉션을 탐색하고 싶은 경우에는 명시적으로 join으로 컬렉션 객체를 명시하고 별칭을 줘서 사용하자
- 주의사항
  - **항상 내부조인이다.**
  - **묵시적 내부 조인이 일어난다는건 From절에 영향을 미친다. 성능에 이슈가 발생할 경우 찾기도 어려우므로 명시적 조인을 꼭 사용하자.**

### 다형성 쿼리
- JPQL로 부모 엔티티를 조회하면 그 자식 엔티티도 함께 조회된다.(상속)
  #### Type, Treat
  ```java
    select i from Item i where type(i) IN (Book, Movie) // Item을 상속받은 Book, Movie 타입의 Item만 조회한다.
    
    select i from Item i where treat(i as Book).author = '고범석'; // Book 타입의 author 필드 값이 '고범석'의 경우만 조회
  ```
### 엔티티 직접 적용
- 객체 인스턴스는 참조 값으로, 테이블 로우는 기본키 값으로 식별한다.
```java
  select m from Member m where m == :member // 실제 쿼리는 where m.id = ? 으로 나간다.

  select p from Product p where p.category = :category  // (Category : Product) = (1 : 多)의 경우에서 외래키도 where category.id = ? 적용
```

### Named Query
- 동적 쿼리 : em.createQuery
- 정적 쿼리 : 미리 정의한 쿼리에 이름을 부여해 필요할 때 사용할 수 있는 쿼리(Named Query)
- 애플리케이션 로딩 시점에 JPQL 문법을 체크하고 미리 파싱하여 오류를 빨리 체크할 수 있고 성능상의 이점도 있다.
```java
@Entity
@NamedQueries({
  @NamedQuery(
  name = "Product.findByProductName",
  query = "select p from Product p where p.name = :name"),
  @NamedQuery(
  name = "Product.findByProductPrice",
  query = "select p from Product p where p.price = :price")
})
public class Product{...}

...

List<Product> products = em.createNamedQuery("Product.findByProductName", Product.class)
  .setParameter("name", "상품명")
  .getResultList();
```
