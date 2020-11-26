고급 매핑
====================

## 상속 관계 매핑
- 객체의 상속구조와 데이터베이스의 슈퍼타입, 서브타입을 매핑하는 것이다.
1. 각각의 테이블로 변환
2. 통합 테이블로 변환
3. 서브타입 테이블로 변환

### 1. 조인전략(각각의 테이블로 변환)
- 엔티티 각각을 모두 테이블로 만들고 자식테이블이 부모테이블의 기본키를 받아 기본키 + 외래키로 사용하는 전략
- Dtype 컬럼을 구분 컬럼으로 사용하자 (@DiscriminatorColumn(name = "DTYPE"))
- 장점
  - 정규화
  - 외래키 참조 무결성 제약조건 활용
  - 저장공간의 효율화
- 단점
  - 조회할 때 성능저하
  - 조회쿼리의 복잡도
  - INSERT SQL이 두번 실행
```java
@Entity
@Inheritance(strategy = InheritanceType.JOINED)   //상속 매핑
@DiscriminatorColumn(name = "DTYPE")
public abstract class Item{   //슈퍼클래스는 추상으로!
  @Id @GeneratedValue
  @Column(name = "ITEM_ID")
  private Long id;
  
  private String name;
  private int price;
  private String brand;
}

@Entity
@DiscriminatorValue(name = "T")   // 엔티티 저장 시 구분 컬럼에 입력할 값
public class Top extends Item{  //윗도리
  private int armLength;
}

@Entity
@DiscriminatorValue(name = "B")
public class Bottom extends Item{ //아랫도리
  private int length;
}
```

### 2. 단일 테이블 전략(테이블을 하나로만 사용)
- 자식 엔티티가 매핑한 컬럼은 모두 null을 허용해야 한다.
- 구분 컬럼을 필수로 사용해야 한다.
- 장점
  - 조회 성능이 빠르고 단순하다
- 단점
  - 자식 엔티티가 매핑한 컬럼은 모두 null 허용하기
  - 테이블이 커질 수 있다
```java
@Entity
@Inheritance(strategy = InheritanceType.SINGLE_TABLE)
@DiscriminatorColumn(name = "DTYPE")
public abstract class Item{
  @Id @GeneratedValue
  @Column(name = "ITEM_ID")
  private Long id;
  
  private String name;
  private int price;
  private String brand;
}

@Entity
@DiscriminatorValue(name = "T")   //단일 테이블 전략에서는 필수! default = 엔티티명
public class Top extends Item{  //윗도리
  private int armLength;
}

@Entity
@DiscriminatorValue(name = "B")   //단일 테이블 전략에서는 필수! default = 엔티티명
public class Bottom extends Item{ //아랫도리
  private int length;
}
```

## @MappedSuperclass
- 테이블을 매핑하지 않고 매핑정보만 제공할 때 사용한다.
- ex) 등록날짜, 수정날짜 등등..
- 객체들이 주로 사용하는 공통 매핑 정보 정의
- 추상 클래스로 만들자!
- JPQL에서 사용할 수 없다.
