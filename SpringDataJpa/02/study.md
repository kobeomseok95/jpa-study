쿼리 메서드
=====================

### 1장에서 살펴보았듯이 스프링 데이터 jpa에서는 JpaRepository를 상속받은 인터페이스다. 그렇다는 얘기는 구현을 직접하지 않고 쿼리를 날릴 수 있다는건데, 어떻게 하는걸까?

1. 메소드 이름으로 쿼리 생성

   - 스프링 데이터 Jpa는 메소드 이름을 분석하여 JPQL을 생성하고 실행한다.

   - 단순 조회 쿼리의 경우는 이 메소드 이름으로 쿼리를 생성하는 방법이 효과적이다.

   - 하지만 쿼리가 복잡할 경우 메소드 이름이 너무 길어질 수 있어서 이 경우는 사용을 자제하고 다른 방법을 사용하자.

   - 예제

     ```java
     public interface SampleRepository extends JpaRepository<Member, Long>{
         // 10살부터 20살 사이 회원 조회
         public List<Member> findByAgeBetween(int low, int high);
         
         // 10살부터 20살 사이 회원들을 이름 역순으로 조회
         List<Member> findByAgeBetweenOrderByUsernameDesc(int low, int high);
         
         // count, update, delete는 반환타입이 Long, EXISTS는 boolean
     }
     ```

   - 메소드 이름으로 작성하는 가이드라인은 [여기](https://docs.spring.io/spring-data/jpa/docs/current/reference/html/#jpa.query-methods)를 참조하자

   - 이 기능은 엔티티의 필드명이 변경되면 꼭 인터페이스 내에 메서드 이름도 변경해주어야 한다.

2. 메소드 이름으로 Jpa NamedQuery 호출

   - 엔티티에서 @NamedQuery 를 이용해 쿼리를 정의하고 메서드에서 호출하는 방식

   - 애플리케이션 실행 시점에 오류를 찾을 수 있다!

   - 예제

     ```java
     @Entity
     @NamedQuery(
     	name = "Member.findByUsername",	// '엔티티명.쿼리명'이 관례
         query = "select m from Member m where m.username = :username"
     )
     public class Member{...}
     
     ...
         
     public interface SampleRepository extends JpaRepository<Member, Long>{
         
         @Query(name = "Member.findByUsername")	// 생략가능
         List<Member> findByUsername(@Param("username") String username);
     }
     ```

   - @Query를 생략할 수 있는데 여기서 알 수 있는 것은 스프링 데이터 JPA는 선언한 '도메인 클래스 + . + 메서드명'으로 쿼리를 찾아 실행한다. 없을 경우 메서드 이름으로 쿼리 생성!

3. @Query 어노테이션 사용

   - 방금 위에서 봤던 @Query 에 JPQL을 작성하는 방법

   - NamedQuery와 마찬가지로 애플리케이션 실행 시점에 오류를 찾을 수 있다.

   - 예제

     ```java
     @Query("select m from Member m")
     List<Member> findAllMembers();
     ```

<hr></hr>

### 데이터 조회해보기

- 단순한 조회는 @Query를 통해 예를 들었다. 이 방식은 @Embedded도 조회할 수 있다!

- DTO로 직접 조회

  ```java
  @Query("select new com.example.repository.UserDto(m.id, m.name, m.phone) from Member m")
  List<UserDto> findUserDto();	// 꼭 select 절에 new 키워드 사용하기!
  ```

<hr></hr>

### 파라미터 바인딩

- 항상 이름 기반으로 사용하자.

- 참고로 컬렉션을 파라미터로 쓰게 되면 in절 query를 쓸 수 있다.

  ```java
  @Query("select p from Product p where p.name in :names")
  List<Member> findByNames(@Param("names") List<String> names);
  ```

<hr></hr>

### 반환 타입

- 스프링 데이터 JPA는 유연하게 반환타입을 지정할 수 있다.

  ```java
  List<Product> findByNames(String name);
  Product findByNames(String name);
  Optional<Product> findByNames(String name);
  ```

- 반환 타입이 컬렉션인 경우 getResultList() 메서드를 실행하며 조회된 데이터가 없더라도 사이즈가 0인 컬렉션이 반환된다. null이 아님에 주의하자

- 반환 타입이 단건(해당 엔티티, Optional)인 경우 getSingleResult() 메서드를 실행하는데, 이 경우 결과가 없다면 null을 반환하고, 결과가 복수일 경우 NonUniqueResultException 예외가 발생한다.

- 원래 getSingleResult()에서 결과가 없다면 NoResultException 예외가 터져야 하지만 개발자 편의를 위해 예외를 다 무시하고 null로 반환한다.

<hr></hr>

### 스프링 데이터 JPA에서 페이징과 정렬

먼저 반환 타입을 살펴보자

- Page<T>

  해당 엔티티를 페이징하여 반환 받았다. 이 Page 인터페이스는 Slice 인터페이스를 상속받았는데, Page 인터페이스에서 getTotalPages(), getTotalElements(), Slice 인터페이스에서 getSize(), getContent(), hasContent() 등등 쿼리로 직접 구현해야하는 페이징 및 정렬을 스프링 데이터 JPA가 도와준다.

- Slice<T> 

  다음 페이지만 확인이 가능한 인터페이스다. Page 인터페이스의 부모다.

- 예제

  ```java
  Page<Member> findByUsername(String name, Pageable pageable);
  Slice<Member> findByUsername(String name, Pageable pageable);
  
  ...
      
  // 상품가격이 10000원 이상, 상품명으로 내림차순, 첫번째 페이지를 조회해보자. 한페이지당 다섯개의 상품을 볼 수 있다.
  Page<Product> findByPrice(int price, Pageable pageable);
  
  ...
      
  // 구현 of(보고싶은 페이지 + 1, 한 페이지당 보여줄 데이터 수, 정렬기준)
  PageRequest pageRequest = 
      PageRequest.of(0, 5, Sort.by(Sort.Direction.DESC, "name"));
  Page<Product> page = productRepository.findByPrice(10000, pageRequest);
  ```

- 참고로 반환 타입이 Page인 경우는 getTotalCount()에 의해 count 쿼리를 날리게 된다.  이 count 쿼리는 성능을 굉장히 잡아먹는다. 이 점을 주의해서 다음과 같이 count 쿼리를 분리할 수 있다. 

  > count 쿼리를 분리해야할 때 -> 복잡한 sql, 조회하는 데이터가 left join 쿼리를 날릴 경우
  >
  > 실제 count 쿼리는 left join을 하지 않아도 되기 때문이다!

  ```java
  @Query(value = "select p from Product p",
        countQuery = "select count(p.name) from Product p")
  Page<Product> findByPrice(int price, Pageable pageable);
  ```

- 또한 페이지 및 정렬을 유지하여 DTO로 반환할 수 있다

  ```java
  Page<Product> page = productRepository.findByPrice(10000, pageRequest);
  Page<ProductDto> dtoPage = page.map(p -> new ProductDto());
  //map 메서드의 매개변수는 Function 인터페이스라서 람다 표현식을 활용하자.
  ```

<hr></hr>

### 벌크성 수정 쿼리

- 벌크성 수정, 삭제 쿼리는 @Modyfying을 함께 사용하자. 안 쓸 경우 QueryExecutionRequestException 예외 발생

- 벌크 연산은 JPA의 원리를 잘 알고 있어야 한다. 예제를 통해 같이 보자.

  ```java
  // 상품 생성
  productRepository.save(new Product("p1", 1000));
  productRepository.save(new Product("p2", 2000));
  productRepository.save(new Product("p3", 3000));
  productRepository.save(new Product("p4", 4000));
  productRepository.save(new Product("p5", 5000));
  
  // 1000원 이상의 상품 가격을 10% 인상하자.
  int price = 1000;
  int result = productRepository.bulkPricePlus(price);	
  ```

  이 때 p1~p5 상품 객체들은 모두 영속성 컨텍스트에 존재한다. bulkPricePlus() 메서드에 작성된 JPQL에 의해 쓰기 지연 SQL 저장소에 저장된 모든 쿼리(insert *5 + update)들이 DB에 반영된다. 이 상황에서내가 p1의 가격을 조회하게 되면!!

  영속성 컨텍스트 1차 캐시에 아직 상품 정보가 저장되어 있기에 DB에는 인상된 값이 반영되었지만, 막상 조회를 하면 인상되지 않은 가격이 조회된다!

  > 해결책?

  벌크성 쿼리를 실행하고 나서 반드시 영속성 컨텍스트를 초기화 하자.

  @Modifying(clearAutomatically = true) 설정을 반드시 해줄것!

  > Best Practice

  1. 영속성 컨텍스트에 엔티티가 없는 상태에서 벌크 연산 먼저 실행
  2. 영속성 컨텍스트에 엔티티가 있을 경우 반드시 초기화!

<hr></hr>

### @EntityGraph

다대일 연관관계에서는 항상 페치 타입을 Lazy로 설정해야 한다. Eager로 해둘 경우 예측할 수 없는 쿼리들이 발생하기 때문이다.

하지만 데이터를 조회할 때 join을 자주 사용하게 되는데, 이 때 Lazy의 경우에는 join된 엔티티는 프록시 객체로 되어있고, 해당 프록시 객체를 초기화할 때 진짜 객체를 찾는 쿼리를 날린다. 이를 N + 1 문제라 하며 이런 문제들을 해결하는 방법은 fetch join을 JPQL에서 사용하는 것이다.

스프링 데이터 JPA에서는 이 fetch join(Entity Graph) 기능을 편하게 사용할 수 있도록 도와준다.

- 예제

  ```java
  @Override
  @EntityGraph(attributePaths = {"category"})	// 연관된 엔티티
  List<Product> findAll();
  
  // JPQL과 함께 사용할 수 있다.
  @EntityGraph(attributePaths = {"category"})
  @Query("select p from Product p")
  List<Product> findProductEntityGraph();
  
  // 메서드 이름 쿼리에서 특히 편리하다
  @EntityGraph(attributePaths = {"category"})
  List<Product> findByName(@Param("name") String name);
  ```

NamedEntityGraph도 있다!

- 예제

  ```java
  @NamedEntityGraph(name = "Product.all",
  	attributeNodes = @NamedAttributeNode("category"))
  @Entity
  public class Product{}
  
  ...
      
  @EntityGraph("Product.all")
  @Query("select p from Product p")
  List<Product> findProductNamedEntityGraph();
  ```
  
