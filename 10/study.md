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





