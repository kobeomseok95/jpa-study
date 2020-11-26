다양한 연관관계 매핑
====================

## 다대일 단방향
- DB를 보면항상 1:N 관계에서는 N 쪽에 외래키가 있다.
- N에서 1의 객체를 참조할 수 있지만, 1의 객체에서 N의 객체를 참조할 수 없는 관계를 뜻한다.
```java
// 한 카테고리에 여러 상품을 가질 수 있는 관계

class Product{
  @Id @GeneratedValue
  @Column(name = "PRODUCT_ID"
  private Long id;

  private String name;

  @ManyToOne
  @JoinColumn(name = "CATEGORY_ID")
  private Category category;
  ...
}

...

class Category{
  @Id @GeneratedValue
  @Column(name = "CATEGORY_ID"
  private Long id;

  private String name;

  // 여기는 products를 참조할 수 없다.
}
```
  
## 다대일 양방향
- 항상 외래키의 연관관계 주인을 생각하자!
- 또한 항상 서로를 참조해야 한다!
- 꼭 연관관계 편의 메서드도 생성하기
```java
// 한 카테고리에 여러 상품을 가질 수 있는 관계

class Product{
  @Id @GeneratedValue
  @Column(name = "PRODUCT_ID"
  private Long id;

  private String name;

  @ManyToOne
  @JoinColumn(name = "CATEGORY_ID")
  private Category category;
  ...
}

...

class Category{
  @Id @GeneratedValue
  @Column(name = "CATEGORY_ID"
  private Long id;

  private String name;

  @OneToMany(mappedBy = "category")
  private List<Product> products = new ArrayList<>();
  // 여기는 products를 참조할 수 없다.
  ...
}
```
  
## 일대다 단방향
- 이 경우는 반대쪽 테이블의 외래키를 직접 관리하는 관계다.
- 또한 @JoinColumn을 반드시 명시해주어야 한다.
- 그렇지 않으면 JoinTable 전략을 사용하기 때문!
- 단점 : 매핑한 객체가 관리하는 외래키가 다른 테이블에 있다는 점이다.
- 웬만하면 다대일 양방향을 사용하자!
```java
// 한 카테고리에 여러 상품을 가질 수 있는 관계
// 이 경우는 DB에서 외래키는 Product 테이블에 있지만, 관리는 Category가 관리한다

class Product{
  @Id @GeneratedValue
  @Column(name = "PRODUCT_ID"
  private Long id;

  private String name;
  ...
}

...

class Category{
  @Id @GeneratedValue
  @Column(name = "CATEGORY_ID"
  private Long id;

  private String name;

  @OneToMany
  @JoinColumn(name = "CATEGORY_ID") //Product 테이블의 CATEGORY_ID
  private List<Product> products = new ArrayList<>();
  ...
}
```

## 일대다 양방향
- 일대다 단방향 매핑 + 다대일 단방향 매핑을 읽기 전용으로 추가한 개념
```java
// 한 카테고리에 여러 상품을 가질 수 있는 관계
// 이 경우는 DB에서 외래키는 Product 테이블에 있지만, 관리는 Category가 관리한다

class Product{
  @Id @GeneratedValue
  @Column(name = "PRODUCT_ID"
  private Long id;

  private String name;
  
  @ManyToOne
  @JoinColumn(
    name = "CATEGORY_ID",
    insertable = false, updatable = false   // 읽기 전용이라 insert, update를 하면 안된다.
  )
  private Category category;
  ...
}

...

class Category{
  @Id @GeneratedValue
  @Column(name = "CATEGORY_ID"
  private Long id;

  private String name;

  @OneToMany
  @JoinColumn(name = "CATEGORY_ID") //Product 테이블의 CATEGORY_ID
  private List<Product> products = new ArrayList<>();
  ...
}
```

## 일대일
- 해당 관계는 대상 테이블들중 어디든지 외래키를 가질 수 있다.
  - 참고로 여기서 주 테이블이란? 조회를 많이 필요로하는 테이블을 주 테이블로 가정해보자
  - 주 테이블에 외래키를 둘 경우 : 주 테이블만 확인해도 대상 테이블과 연관관계를 확인할 수 있다.
  - 대상 테이블에 외래키를 둘 경우 : 구조를 일대다로 변경시 테이블 구조를 그대로 유지할 수 있다.
  
- 주 테이블에서의 단방향
```java
// 한 카테고리에는 한 상품만 들어갈 수 있다 가정, 또한 주 테이블은 Product!
class Product{
  @Id @GeneratedValue
  @Column(name = "PRODUCT_ID")
  private Long id;
  private String name;
  
  @OneToOne
  @JoinColumn(name = "CATEGORY_ID")
  private Category category;
}

...

class Category{
  @Id @GeneratedValue
  @Column(name = "CATEGORY_ID")
  private Long id;
  private String name;
}
```

- 주 테이블에서의 양방향
```java
// 한 카테고리에는 한 상품만 들어갈 수 있다 가정, 또한 주 테이블은 Product!
class Product{
  @Id @GeneratedValue
  @Column(name = "PRODUCT_ID")
  private Long id;
  private String name;
  
  @OneToOne
  @JoinColumn(name = "CATEGORY_ID")
  private Category category;
}

...

class Category{
  @Id @GeneratedValue
  @Column(name = "CATEGORY_ID")
  private Long id;
  private String name;
  
  @OneToOne(mappedBy = "category")
  private Product product;
}
```

- ~~대상 테이블에서의 단방향~~

- 대상 테이블에서의 양방향
```java
// 한 카테고리에는 한 상품만 들어갈 수 있다 가정, 또한 주 테이블은 Product!
class Product{
  @Id @GeneratedValue
  @Column(name = "PRODUCT_ID")
  private Long id;
  private String name;
  
  @OneToOne(mappedBy = "product")
  private Category category;
}

...

class Category{
  @Id @GeneratedValue
  @Column(name = "CATEGORY_ID")
  private Long id;
  private String name;
  
  @OneToOne
  @JoinColumn(name = "PRODUCT_ID")
  private Product product;
}
```

## 다대다
- 꼭 일대다 다대일로 풀어가자
- 연결 엔티티의 Key는 비즈니스와 아무 관련이 없는 대리 키를 
