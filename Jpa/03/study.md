영속성 관리
===============

## EntityManagerFactory
- EntityManager를 만드는 공장이라고 생각하면 된다.
- 한 개만 만들어서 애플리케이션 전체에서 공유하는 것이 좋다. EntityManagerFactory를 만드는 비용은 상당히 크기 때문!
```java
EntityManagerFactory emf = Persistence.createEntityManagerFactory("persistUnitName");
```
- 참고로 createEntityManagerFactory에 넣는 인자는 META-INF 안에서 설정한 persistence-unit 태그의 name속성에 설정된 값을 넣어준다.


## EntityManager
- 필요한 곳에서 엔티티 매니저 팩토리로부터 불러오면 된다.
```java
EntityManager em = emf.createEntityManger();
```
**주의할 점**
- EntityManagerFactory는 서로 다른 스레드 간에 공유해도 되지만, EntityManager는 스레드 간에 절대 공유하면 안됨
  - 동시성 이슈가 발생하기 때문이다.
  
## Persistence Context(영속성 컨텍스트)
- Entity를 영구 저장하는 환경
- persist()의 의미 : EntityManager를 사용해서 해당 메소드 인자의 엔티티를 Persistence Context에 저장한다는 의미

## Entity Life-Cycle
1. 비영속
  - Entity 객체를 생성했고, 영속성 컨텍스트에 저장하지 않은 상태이다
  ```java
  Product product = new Product();
  product.setId(1L);
  product.setName("상품명");
  ```

2. 영속
  - EntityManager를 통하여 영속성 컨텍스트에 저장된 상태!
  ```java
  em.persist(product);
  ```
  - 영속성 컨텍스트에 의해 해당 엔티티가 관리된다.
  
3. 준영속
  - 영속상태였던 엔티티가 영속성 컨텍스트에서 관리하지 않는 상태를 준영속 상태라고 한다.
  - detach() : 준영속 상태로 만든다.
  - clear() : 영속성 컨텍스트 초기화
  - close() : 영속성 컨텍스트 닫기
  - 비영속 상태에 가깝다.
  - 식별자 값을 가지고 있다. > 한번 영속 상태를 거쳤기 때문!
  - 지연 로딩이 불가능 하다.

4. 삭제
  - remove() : 영속성 컨텍스트에서 해당 엔티티를 삭제함과 동시에 DB에서도 해당 엔티티를 삭제한다.
  
## 영속성 컨텍스트의 특징
**영속 상태에는 반드시 식별자 값이 있어야 한다.**
- 1차 캐시
- 동일성 보장
- 쓰기 지연
- 변경 감지
- 지연 로딩

1. 1차 캐시
  ```java
  Product product = new Product();
  product.setId(1L);
  product.setName("상품명");
  
  em.persist(product);
  ```
  - 1차 캐시는 영속상태의 엔티티를 저장하는 장소이다.
  - 여기서, 1차 캐시의 키는 해당 엔티티의 @Id, 밸류는 Entity가 되겠다!
  ```java
  em.find(product);
  ```
  - 해당 엔티티를 찾을 경우 DB로 바로가지 않고 1차 캐시를 먼저 찾는다.
  - 만약 1차 캐시에 없을 경우 DB에 갔다 와서 1차 캐시에 저장 후 반환한다.

2. 동일성 보장
  ```java
  //만약 순수한 자바였다면?
  Product a = new Product(1L, "a");
  Product b = new Product(1L, "a");
  
  System.out.println(a == b); //false
  ```
  - 원래 순수한 객체를 '=='로 동일성을 체크할 경우 false가 나온다 하지만
   ```java
  //영속성 컨텍스트에서 가져와보자
  Product a = em.find(Product.class, "a");
  Product b = em.find(Product.class, "a");
  
  System.out.println(a == b); //true
  ```
  - 동일성을 보장한다! 키 값이 같은 객체는 동일하다고 판단하기 때문

3. 쓰기 지연
  - 트랜잭션의 commit()이 가기 전에는 절대로 DB에 쿼리를 날리지 않는다.
  - 해당 쿼리들을 모두 쓰기 지연 SQL 저장소에 저장한 다음 commit()이 될 경우 한번에 쿼리가 반영된다.

4. 변경 감지
  - 수정된 엔티티를 자동으로 반영하는 기능
  - EntityManager 내부에서 flush()가 호출이 되는데, 이 때 엔티티와 스냅샷을 비교하여 값이 다를 경우 수정 쿼리를 생성한다.
  - 그리고 해당 수정 쿼리를 쓰기 지연 SQL 저장소에 보내고 DB에 보낸다.
  - commit()
  - 이 변경 감지는 꼭 영속 상태인 엔티티만 변경이 적용된다!
  
## flush()
- 영속성 컨텍스트의 변경 내용을 DB에 반영하는 메서드!
- 변경 감지 작동
- 쓰기 지연 SQL 저장소의 쿼리를 DB에 전송
- 직접 호출하거나, 트랜잭션 커밋 시, JPQL 쿼리 실행 시 호출
- JPQL 실행시?
```java
em.persist(productA);
em.persist(productB);
em.persist(productC);

List<Product> products = em.createQuery("select p from Product p", Product.class).getResultList();
```
- 이 때, em.createQuery("select p from Product p", Product.class).getResultList()를 실행하기 직전에 영속성 컨텍스트를 flush하여 products에 포함되게 만든다.

## merge()
- 준영속 상태의 엔티티를 받아 그 정보로 새로운 영속 상태의 엔티티를 반환한다.
- merge()를 실행한다.
- 파라미터로 넘어온 준영속 엔티티 식별자값을 1차 캐시, DB에서 엔티티를 조회한다.
- 만약, DB에도 없을 경우 새로운 엔티티 생성
- 해당 식별자의 엔티티가 있을 경우, 기존의 파라미터로 넘어온 엔티티를 영속 상태로 변경한 다음에 수정된 값들을 DB에 반영한다.
