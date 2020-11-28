프록시와 연관관계 관리
====================

## 프록시
- 지연 로딩 : 엔티티가 실제 사용될 때 까지 조회를 지연하는 방법
- 프록시 객체가 지연 로딩을 도와준다!
- EntityManager.getReferences()
  - 해당 메서드 호출 시 DB 조회도 하지 않고, 실제 엔티티 객체도 생성하지 않는다.
  - 대신, DB접근을 위임한 프록시 객체를 반환!
- getReferences()로 찾은 프록시 객체의 메서드를 불러 실제 데이터를 조회한다.

## 프록시 초기화 과정
```java
Member m = em.getReference(Member.class, member.getId()); //Proxy
m.getName();  //Proxy 상태에서 진짜 객체 메서드 호출, 실제 데이터 조회
```
- 1. 프록시 객체의 엔티티 메서드 호출
- 2. 프록시 객체는 실제 엔티티가 생성되어 있지 않으면 영속성 컨텍스트에 실제 엔티티 생성을 요청한다. 이 때 초기화가 발생!
- 3. 영속성 컨텍스트는 DB를 조회하여 실제 엔티티 객체 생성
- 4. 이후 프록시 객체는 실제 엔티티 객체의 참조를 멤버변수에 보관
- 5. m.getName()의 결과 반환

## 프록시 객체의 특징
- 1. 프록시 객체는 한번만 초기화
- 2. **초기화가 되었다고 해서 프록시 객체가 실제 엔티티가 되는 것은 아니다! 단지, 프록시 객체를 통해 실제 엔티티에 접근할 수 있는 것**
- 3. 타입 체크에 유의하자. 동일성(==) 타입 체크 하지 말것.
- 4. **만약 영속성 컨텍스트에 찾는 엔티티가 존재한다면? getReferences를 호출해도 실제 엔티티 반환**
- 5. 프록시 객체가 초기화를 할 때 영속성 컨텍스트의 도움을 받는다. 준영속 상태라면 초기화할 수 없다.
- 6. **엔티티를 프록시로 조회할 때, 식별자 값을 파라미터로 전달하는데, 프록시 객체는 이 식별자 값을 보관한다.**
- 7. **연관관계를 설정할 때, 식별자 값만 사용하기 때문에 프록시 객체를 활용해서 DB 쿼리를 줄일 수 있다.**
```java
Member member = em.find(Member.class, 1L);
Team team = em.getReferences(Team.class, "team1");  //식별자 값 보관 및 DB 쿼리가 나가지 않음
member.setTeam(team); //쿼리가 안나가고도 연관관계를 설정할 수 있다.
```

## 즉시 로딩과 지연 로딩
- 즉시 로딩 : 엔티티 조회 시 연관된 엔티티도 함께 조회한다.
- 지연 로딩 : 엔티티 조회 시 연관된 엔티티를 실제 사용할 때 조회한다.
```java
// 한 카테고리에는 여러 상품이 있다고 가정

// 1. 즉시로딩시
class Product{
  ...
  @ManyToOne(fetch = FetchType.EAGER)
  @JoinColumn(name = "CATEGORY_ID")
  private Category category;    // 이 때 Product를 조회할 경우 연관된 카테고리 까지 가져온다.
  ...
}

// 2. 지연로딩시
class Product{
  ...
  @ManyToOne(fetch = FetchType.LAZY)
  @JoinColumn(name = "CATEGORY_ID")
  private Category category;    // 이 때 Product를 조회할 경우 연관된 카테고리 까지 가져온다.
  ...
}

// 3. 지연로딩 실행시
Member member = em.find(Member.class, 1L);
Team team = member.getTeam(); // 이때 Team은 프록시객체!
team.getName(); // 팀 객체 실제 사용 이때 쿼리가 나감!
```
#### Tip
- 모든 연관관계를 설정할 때 우선은 지연 로딩으로 사용하기!
- 상황을 보고 나서 즉시 로딩 전략을 쓸지 고려해보기
- EAGER 전략 사용시, 컬렉션을 하나 이상 즉시 로딩하는 것은 권장하지 않음
  - 해당 컬렉션(다)를 즉시 로딩할 경우 SQL 실행 결과가 N*M이 된다.
- 컬렉션 즉시 로딩은 항상 OUTER JOIN을 사용한다.
  - 예를 들어 다대일의 기준에서 외래키 제약조건이 not null이면 항상 inner join을 사용해도 된다.
  - 하지만, 일대다의 경우 일의 엔티티에 속한 엔티티들이 하나도 없을경우 inner join을 사용하면 일의 엔티티까지 조회가 되지 않는다.
  - 정리
  - @ManyToOne, @OneToOne
    - (optional = false, 외래키가 NULL이 없을 경우)    : INNER JOIN
    - (optional = true, 외래키가 NULL을 허용할 경우)   : OUTER JOIN
  - @OneToMany, @ManyToMany
    - (optional = false)  : OUTER JOIN
    - (optional = true)   : OUTER JOIN

## 영속성 전이(CASCADE)
- 특정 엔티티를 영속상태로 만들때 연관된 엔티티도 함께 영속 상태, 엔티티를 삭제할 경우 연관된 엔티티도 연쇄적으로 지울 때 사용
- 참고로 REMOVE, PERSIST의 경우는 flush를 호출할 때 전이가 발생한다.
- REMOVE를 설정하지 않고 부모의 엔티티를 삭제시 에러가 발생한다
  - 당연하지! 외래키 제약조건에 의해 자식들을 먼저 삭제해야하기 때문이걸랑
```java
// Persist
class Parent{
  @OneToMany(mappedBy = "parent", cascade = CascadeType.PERSIST)  //연관된 엔티티들도 함께 영속화
  private List<Child> children = new ArrayList<>();  
}
...

Child c1 = new Child();
Child c2 = new Child();

Parent p = new Parent();

c1.setParent(p);
c2.setParent(p);

p.getChildren().add(c1);
p.getChildren().add(c2);

em.persist(p);  // Parent 엔티티 뿐만 아니라 Child 엔티티 두개도 함께 영속화
```

```java
// Remove
class Parent{
  @OneToMany(mappedBy = "parent", cascade = CascadeType.REMOVE)  //연관된 엔티티들도 함께 삭제
  private List<Child> children = new ArrayList<>();  
}
...

Parent findP = em.find(Parent.class, 1L);
em.remove(findP); // Parent 엔티티에 속한 Child들 까지 모두 제거
```

## 고아 객체(orphan)
- 부모 엔티티와 연관관계가 끊어진 자식 엔티티들을 자동으로 삭제하는 기능을 제공한다.
```java
// Persist
class Parent{
  @OneToMany(mappedBy = "parent", orphanRemoval = true)  //연관관계가 끊어진 자식 엔티티 자동 삭제
  private List<Child> children = new ArrayList<>();  
}
...

Parent findP = em.find(Parent.class, 1L);
findP.getChildren().remove(0);  
// 해당 컬렉션 내부의 0번째 객체는 부모와 연관관계가 끊어져서 orphanRemoval 옵션에 의해 자동으로 삭제
```

#### Tip
- CascadeType.ALL + orphanRemoval = True의 경우?
  - 자식엔티티들이 부모 엔티티에 의해 생명주기를 관리한다는 의미다.
  - 왜??
    - CascadeType.ALL로 인해 해당 엔티티가 영속화, 삭제될 경우 자식 엔티티도 그에 따라 연쇄적으로 영속화, 삭제가 진행되기 때문
    - orphanRemoval = True로 인해 부모 엔티티에서 한 자식 엔티티의 연관관계를 제거할 경우 연관관계가 끊어진 자식 엔티티를 제거하기 때문
