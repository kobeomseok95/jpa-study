확장 기능
================================

### 1. 사용자 정의 Repository 구현

이전 장에서 스프링 데이터 JPA Repository는 인터페이스만 정의하고 구현체는 스프링이 자동으로 생성(프록시)하는 것을 확인했다. 

하지만 내가 직접 인터페이스를 정의하고 구현한다면 스프링 데이터 JPA에서 정의된 메서드 마저 구현해야해서 번거롭다. 

또한 직접 구현해야할 상황이 굉장히 다양한데, 직접 정의하는 방법을 알아보자.

[사용자 정의 Repository 구현 방법 참고](https://docs.spring.io/spring-data/jpa/docs/current/reference/html/#repositories.custom-implementations)

```java
// 내가 직접 정의한 인터페이스
public interface CustomProductRepository{
    List<Product> myFindProduct();
}

// 기존의 JpaRepository를 상속받은 인터페이스
public interface ProductRepository extends JpaRepository<Product, Long>,
CustomProductRepository{/* methods */}

// 내가 정의한 인터페이스를 직접구현
public class CustomProductRepositoryImpl implements CustomProductRepository{
    
    @Override
    public List<Product> myFindProduct(){/*로직*/}
}

// 구현한 메서드를 호출하고 싶다면?
productRepository.myFindProduct();
```

![repository구조](https://user-images.githubusercontent.com/37062337/103159728-29080480-4810-11eb-8fc4-1a24290b55c2.png)

이러한 구조이다. JpaRepository 위에는 상위 Interface들이 더 존재한다.

**명명규칙** : 참고한 Docs에서 내가 직접 정의한 인터페이스를 클래스로 구현하는 방법은 클래스명을 해당 Repository + Impl을 붙인다.

또한 직접 정의한 클래스를 스프링에서는 스프링 빈으로 등록하기 때문에 문제없이 잘 작동한다. 구현한 메서드를 호출하고 싶다면 CustomRepository가 아닌 JpaRepository를 상속받은 Repository에서 구현한 메서드를 호출하면 된다.

> 이유 : 그림을 보면 직접 정의한 Repository 인터페이스는 JpaRepository를 상속받은 인터페이스보다 상위이기 때문에!

### 2. Auditing

엔티티에서 '등록일, 수정일, 등록자, 수정자' 이 네가지를 빈번하게 쓴다. 스프링 데이터 JPA에서는 이 네가지를 아주 편하게 쓸 수 있도록 지원한다.

```java
@EnableJpaAuditing // 먼저, 스프링 부트 설정 클래스에 이 어노테이션 등록
@SpringBootApplication
public class JpaApplication {
	public static void main(String[] args){
        SpringApplication.run(JpaApplication.class, args);
    }
}

/* 
	그 다음 해당 Auditing을 적용해보자.
     참고로 등록일, 수정일과 등록자, 수정자는 분리하는게 좋다. 
     등록자 수정자가 필요하지 않는곳도 있기 때문
*/

// 등록일, 수정일
@EntityListeners(AuditingEntityListener.class)
@MappedSuperclass
@Getter
public class BaseTimeEntity {

    @CreatedDate
    @Column(updatable = false)
    private LocalDateTime createdDate;

    @LastModifiedDate
    private LocalDateTime lastModifiedDate;
}

// 등록자, 수정자
@MappedSuperclass
@Getter
public class BaseEntity extends BaseTimeEntity {

    @CreatedBy
    @Column(updatable = false)
    private String createdBy;

    @LastModifiedBy
    private String lastModifiedBy;
}
```

#### @EntityListeners(AuditingEntityListener.class)

- 등록자 및 수정자를 자동으로 처리해주는 어노테이션
- 해당 클래스에 Auditing(감시, 감사) 기능을 포함
- 해당 클래스를 상속받은 엔티티를 저장할 때, JPA가 트랜잭션 커밋 시점에 flush를 호출시 하이버네이트가 자동으로 해당 필드 값을 채워준다.

#### 등록자 및 수정자를 자동으로 처리할 때

```java
// 스프링 설정 클래스에 해당 생성자 추가, 실무에서는 다른 방식!
@Bean
public AuditorAware<String> auditorProvider() {
    return () -> Optional.of(UUID.randomUUID().toString());
}
```

### 3. Domain Class Converter

HTTP 파라미터로 넘어온 엔티티의 id로 엔티티 객체를 찾아서 바인딩할 수 있다.

```java
@RestController
@RequiredArgsConstructor
public class MemberController {

    private final MemberRepository memberRepository;

    @GetMapping("/members/{id}")
    public String findMember2(@PathVariable("id") Member findMember) {
        return findMember.getUsername();
    }
}
```

하지만 트랜잭션의 영향이 미치지 않는 Controller 계층에서 엔티티를 다루는 것은 상당히 위험하다. DTO로 변환해서 반환하자!

### 4. 페이징과 정렬

```java
@GetMapping("/members")
public String list(Pageable pageable) {
    return memberRepository.findAll(pageable);
}
```

- 매개변수로 파라미터를 받을 수 있다.
- Pageable은 인터페이스이며 실제는 PageRequest라는 객체를 생성한다.

해당 URI를 호출하여 응답받은 JSON을 보면

```json
{
    "content": [
        
    ],
    "pageable": {
        "sort": {
            "sorted": false,
            "unsorted": true,
            "empty": true
        },
        "offset": 0,
        "pageNumber": 0,
        "pageSize": 10,
        "unpaged": false,
        "paged": true
    },
    "last": false,
    "totalElements": 100,
    "totalPages": 10,
    "size": 10,
    "number": 0,
    "sort": {
        "sorted": false,
        "unsorted": true,
        "empty": true
    },
    "first": true,
    "numberOfElements": 10,
    "empty": false
}
```

이러한 형태로 JSON을 받게 된다. 

다음은 요청 파라미터를 보고 해석을 해보면

**/members?page=0&size=3&sort=id,desc&sort=username,desc**

- page : 보고싶은 페이지, 0부터 시작
- size : 한 페이지당 보고싶은 content 갯수
- sort : 정렬 조건 정의, 해당 예제는 id를 내림차순, 유저의 이름을 내림차순으로 보기 원한다.

#### 페이징의 디폴트 설정을 바꾸는 방법

- 글로벌 설정

  applicatoin.yml or properties 파일에 가서 다음 속성들을 설정하자

  - spring.data.web.pageable.default-page-size : 한 페이지의 기본 content 사이즈
  - spring.data.web.pageable.max-page-size :  한 페이지의 최대 content사이즈
  - spring.data.web.pageable.one-indexed-parameters :  page를 0이 아닌 1부터 시작할지의 여부

- 개별 설정

  ```java
  @GetMapping("/members")
  public String list(@PageableDefault(size = 10, 
                                      sort = "username",
                                      direction = Sort.Direction.DESC) Pageable pageable) {
      return memberRepository.findAll(pageable);
  }
  ```

#### 항상 엔티티를 직접 반환하지 말고 DTO를 반환해야 한다!

```java
// DTO
@Data
public class ProductDto{
    private Long id;
    private String name;
    private int price;
    
    public ProductDto(Product p){
        this.id = p.getId();
        this.name = p.getName();
        this.price = p.getPrice();
    }
}

// Controller
@GetMapping("/products")
public Page<ProductDto> list(Pageable pageable){
    return productRepository.findAll(pageable).map(ProductDto::new);
}

```
