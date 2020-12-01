값 타입
==============

## 값 타입의 분류
- 기본값 타입
  - primitive type
  - wrapper class
  - String
- 임베디드 타입
- 컬렉션 타입

## 임베디드 타입
- 새로운 값의 타입을 직접 정의해서 사용하는 것
```java
@Embeddable
public class Period{
  private LocalDateTime startDate;
  private LocalDateTime endDate;
  
  public Period(){
  } //기본 생성자 필수
}
...
@Entity
public class Member{
  @Id @GeneratedValue
  private Long id;
  private String name;
  
  @Embedded
  private Period workPeriod;  //값타입
}
```
- 새로 정의한 값 타입들은 보다 객체지향적으로 사용이 가능하다.
- 해당 엔티티의 생명주기에 의존한다.
- 값 타입을 포함하거나 엔티티를 참조할 수 있다.

## @AttributeOverride : 속성 재정의
- 만약 해당 엔티티에서 같은 값 타입의 필드가 하나 더 필요하다면 ?
```java
@Entity
public class Member{
  @Id @GeneratedValue
  private Long id;
  private String name;
  
  @Embedded
  private Period workPeriod;  //값타입
  
  @Embedded
  @AttributeOverrides({
    @AttributeOverride(name = "startDate", column = @Column(name = "PREVIOUS_STARTDATE")),
    @AttributeOverride(name = "endDate", column = @Column(name = "PREVIOUS_ENDDATE"))
  })
  private Period prevWorkPeriod;  // 이전 직장에서의 근무기간
}
```
- 이 어노테이션은 무조건 Entity에 설정해야한다!

## 값 타입과 불변 객체
- 값 타입 공유 참조
```java
// 해당 샵의 주소를 변경해보자

shop.setAddress(new Address("oldCity"));
Address address = shop.getAddress();

address.setCity("newCity");
shop2.setAddress(address);
```
- 위 코드의 경우 shop, shop2는 모두 주소가 변경된다!
- 왜??
  - Address는 객체이므로 shop, shop2는 같은 Address 참조를 가지기 때문에 참조하는 객체의 값이 변경되면 참조하는 엔티티들의 값들도 모두 변경되기 때문이다!!
  - 즉 같은 인스턴스를 사용하면 안된다!!
  - 하지만 새 인스턴스를 만드는 것도 한계가 있다... 극복방법은? > **불변객체로 만들자!**

## 값 타입의 비교
- privimive 타입을 동일성(==)으로 비교하면 같다고 나오지만 인스턴스는 값이 같더라도 인스턴스(참조하는 주소)가 다르다면 동일성 비교를 해도 false가 나온다.
- 해당 값 타입 내부에 equals와 hashcode 메서드를 재정의하자!

## 값 타입 컬렉션
- 값 타입을 여러개 저장해야할 경우
```java
@Entity
public class Member{
  @Id @GeneratedValue
  private Long id;
  private String name;
  
  @Embedded
  private Period workPeriod;  //값타입
  
  @ElementCollection
  @CollectionTable(name = "FAVORITE_FOOD", joinColumns = @JoinColumn(name = "MEMBER_ID"))
  @Column(name = "FOOD_NAME")
  private Set<String> favoriteFoods = new HashSet<>();        // 값 타입을 컬렉션(Set)으로 저장
  
  @ElementCollection
  @CollectionTable(name = "ADDRESS", joinColumns = @JoinColumn(name = "MEMBER_ID"))
  private List<String> addressHistory = new ArrayList<>();    // 값 타입을 컬렉션(List)으로 저장
}
```
- 관계형 DB에서는 컬럼안에 컬렉션을 포함할 수 없으므로 별도의 테이블을 만든다.
- 값 타입 컬렉션은 영속성 전이(Cascade) + 고아 객체 제거(orphan remove) 기능을 필수로 가진다!
- 또한 조회할 때 fetch 전략이 LAZY가 기본!

## 값 타입 컬렉션의 제약사항
- 값 타입은 식별자가 존재하지 않고 단순한 값들의 모음이므로 값을 변경해버리면 DB에 저장된 원본 데이터를 찾기 어렵다!
- 이러한 문제로 JPA 구현체들은 값 타입 컬렉션에 변경 사항 발생시, 값 타입 컬렉션이 매핑된 테이블의 연관된 모든 데이터를 삭제하고 현재 값 타입 컬렉션 객체에 들어가 있는 모든 값을 DB에 다시 저장한다.
- 값 타입 컬렉션이 매핑된 테이블에 데이터가 많다면 일대다 관계를 고려하자!
> 값 타입 컬렉션을 매핑하는 테이블은 모든 컬럼을 묶어서 기본 키를 구성한다. 그래서 null값과 중복데이터를 넣을 수 없다!
