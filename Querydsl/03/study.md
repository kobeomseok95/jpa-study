중급 문법
====================

- 프로젝션과 결과 반환

  Projection : select의 대상을 지정하는 것

  ```java
  // Projection을 하나만 지정, 반환 타입을 명확하게 지정할 수 있다.
  List<String> result = queryFactory
      .select(product.username)
      .from(product)
      .fetch();
  
  // Projection을 둘 이상 지정, Tuple에 감싸서 반환된다.
  List<Tuple> result = queryFactory
      .select(product.name, product.price)
      .from(product)
      .fetch();
  ```

- 프로젝션을 DTO로 조회

  기존의 JPQL로 작성 시 프로젝션할 때 new 키워드를 사용하여 작성했다. 하지만 querydsl에서는 다음 네가지의 기능을 제공한다.

  > 1. 프로퍼티 접근(setter)
  > 2. 필드 직접 접근
  > 3. 생성자 사용
  > 4. @QueryProjection (DTO 만든 후, compileQuerydsl 꼭 해주기!)

  ```java
  // 먼저 해당 DTO 생성
  @Data
  @NoArgsConstructor	// QueryProjection 사용시 있어야 한다.
  public class ProductDto{
      private String name;
      private int price;
      
      @QueryProjection // QueryProjection 사용시 있어야 하는 생성자
      public ProductDto(String name, int price){
          this.name = name;
          this.price = price;
      }
  }
  
  // setter 접근
  List<ProductDto> result = queryFactory
      .select(Projections.bean(ProductDto.class,
                              product.name,
                              product.price))
      .from(product)
      .fetch();
  
  // field 접근
  List<ProductDto> result = queryFactory
      .select(Projections.fields(ProductDto.class,
                              product.name,
                              product.price))
      .from(product)
      .fetch();
  
  // 생성자 사용
  List<ProductDto> result = queryFactory
      .select(Projections.constructor(ProductDto.class,
                              product.name,
                              product.price))
      .from(product)
      .fetch();
  
  // QueryProjection
  List<ProductDto> result = queryFactory
      .select(new QProductDto(product.name, product.price))
      .from(product)
      .fetch();
  ```

  **만약 별칭이나 필드 접근 생성방식에서 이름이 다른 경우**

  ```java
  @Data
  public class ProdDto{
      private String itemName;
      private int avgCost;
  }
  
  // 별칭을 위한 별도의 subQ객체 생성
  // 상품들 이름과 평균가격 조회, 평균 가격은 모든 상품의 평균가격
  QProduct productSub = new QProduct("productSub");
  List<ProdDto> result = queryFactory
      .select(Projections.fields(ProdDto.class,
                                 product.name.as("itemName"),
                                 Expressions.as(
                                     JPAExpressions
                                     .select(productSub.price.avg())
                                     .from(productSub), "avgCost")))
      .from(product)
      .fetch();
  ```

  - Projection시 as메서드를 이용해 별칭을 적어준다.
  - Expressions.as(subQuery, alias)로 표현

- 동적 쿼리

  > 1. BooleanBuilder
  >
  > 2. Where 다중 Parameter

  ```java
  //BooleanBuilder
  public void dynamicQuery_BooleanBuilder(){
      String productName = "상품명";
      Integer productPrice = 10000;
      
      BooleanBuilder builder = new BooleanBuilder();
      
      if (productName != null){
          builder.and(product.name.eq(productName))
      }
      
      if (productPrice != null){
          builder.and(product.price.eq(productPrice))
      }
      
      List<Product> result = queryFactory
          .selectFrom(product)
          .where(builder)
          .fetch();
  }
  
  //Where Parameter
  public void dynamicQuery_WhereParam(){
      String productName = "상품명";
      Integer productPrice = 10000;
      
      List<Product> result = queryFactory
          .selectFrom(product)
          .where(nameEq(productName), priceEq(productPrice))
          .fetch();
  }
  
  public BooleanBuilder nameEq(String param){
      return param != null ? new BooleanBuilder(product.name.eq(param)) : new BooleanBuilder();
  }
  
  public BooleanBuilder priceEq(Integer param){
      return param != null ? new BooleanBuilder(product.price.eq(param)) : new BooleanBuilder();
  }
  ```

  > 두 가지 방식의 차이점
  >
  > BooleanBuilder 방식은 해당 파라미터에 null이 들어와도 예외가 발생하지 않는다. 하지만 builder를 만들어야 하고 재사용성이 떨어진다는 단점이 있다.
  >
  >  
  >
  > Where 다중 파라미터 방식은 설계에 따라 재사용성을 끌어올릴 수 있다. 또한 한번에 표현되어 가독성이 좋다.
  >
  >  
  >
  > 두 방식을 모두 혼합하여 사용한다면 null에도 안전하고 가독 및 유지보수에도 용이한 동적쿼리를 작성할 수 있다!

- 벌크 연산

  update, delete를 한번에 처리할 수 있다. 결과는 영향을 받은 row 수 만큼 long 타입으로 반환된다.

  ```java
  // 재고가 없는 상품의 상품명을 판매완료 했을 때
  queryFactory
      .update(product)
      .set(product.name, "판매완료")
      .where(product.stock.eq(0))
      .execute();
  
  // 모든 상품의 가격을 10% 인하할 때
  queryFactory
      .update(product)
      .set(product.price, product.price.multiply(0.9))
      .execute();
  
  // 재고가 없는 상품들을 삭제할 때
  queryFactory
      .delete(product)
      .where(product.stock.eq(0))
      .execute();
  ```

  > **주의사항**
  >
  > 벌크 연산을 수행할 경우 DB에는 값이 반영되지만 영속성 컨텍스트에는 바뀌지 않은 엔티티들이 존재한다. 벌크 연산 후 조회할 일이 있을 때는 반드시 영속성 컨텍스트를 깨끗하게 초기화 하고 조회하자!

- SQL Function

  SQL Function은 JPA와 같이 Dialect에 등록된 내용만 호출할 수 있다.

  ```java
  // replace 함수 사용 > 상품명중에 product라는 문자열을 P로 변환
  queryFactory
      .select(Expressions.stringTemplate("function('replace', {0}, {1}, {2})",
                                        product.name, "product", "P"))
      .from(product)
      .fetch();
  // 소문자로 변경해서 비교
  queryFactory
      .select(product.name)
      .from(product)
      .where(product.name.eq(
      	Expressions.stringTemplate("function('lower', {0})",
                                    product.name)
      ))
      .fetch();
  ```

  
