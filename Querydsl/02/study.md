기본 문법
============================

1. JPQL vs Querydsl

   ```java
   // jpql
   public Product findByJpql(){
   	return em.createQuery("select p from Product p where p.name = :name")
           .setParameter("name", "상품명")
           .getSingleResult();
   }
   
   // querydsl
   public Product findByQuerydsl(){
       JPAQueryFactory queryFactory = new JPAQueryFactory(em);
       QProduct p = new QProduct("p");
       
       return queryFactory
           .select(p)
           .from(p)
           .where(p.name.eq("상품명"))
           .fetchOne();
   }
   ```

   기존 JQPL 작성시에는 엔티티 매니저에서 JPQL을 작성했지만 Querydsl을 사용할 경우 JPAQueryfactory와 Q타입의 객체를 사용한다. Querydsl은 JPQL의 빌더라고 생각하면 된다.

2. Q-Type 인스턴스를 사용하는 2가지 방법

   ```java
   QMember qMember = new QMember("m");
   QMember qMember = QMember.member;
   ```

   static import하여 더 간편하게 사용할 수 있다.

   ```java
   import static data.querydsl.entity.QMember.*;
   
   // querydsl
   public Product findByQuerydsl(){
       JPAQueryFactory queryFactory = new JPAQueryFactory(em);
       
       return queryFactory
           .select(product)
           .from(product)
           .where(product.name.eq("상품명"))
           .fetchOne();
   }
   ```

   

3. And, Or 조회

   ```java
   public void searchAndParam() {
       Product findProduct = queryFactory
           .selectFrom(product)
           .where(product.name.eq("상품명"),	//and는 쉼표로 가능하다.
                  product.price.eq(1000))
           .fetchOne();
   }
   
   public void searchAndParam2() {
       Product findProduct = queryFactory
           .selectFrom(product)
           .where(product.name.eq("상품명")
                  	.or(product.price.eq(1000)))
           .fetchOne();
   }
   ```

   메서드 체이닝 방식으로 할 수 있다. between, in, startWith 등등 해당 메서드들은 ide에 다 나와있으니 참고하면서 작성해보기

4. 결과 조회
   - fetch() : 리스트 조회, 데이터가 없을 경우 빈 리스트 반환
   - fetchOne() : 단건 조회, 없으면 null, 둘 이상일 경우 NonUniqueResultException 예외 발생
   - fetchFirst() : limit(1).fetchOne();
   - fetchResults() : 페이징 정보 포함, count 쿼리가 추가로 나간다.
   - fetchCount() : count 쿼리

5. 정렬

   - orderBy메서드를 활용하여 정렬이 가능하다

   ```java
   public void sort(){
       queryFactory
           .selectFrom(product)
           .where(product.price.eq(1000))
           .orderBy(product.name.asc().nullsLast())
           // 상품 가격이 1000원인 상품을 조회하고 이름에 대해 오름차순, 이름이 null이라면 마지막에 출력
   }
   ```

6. 페이징

   ```java
   public void paging2(){
       QueryResults<Product> queryResults = queryFactory
           .selectFrom(product)
           .orderBy(product.price.desc())
           .offset(1)
           .limit(2)
           .fetchResults();
       
       System.out.println(queryResults.getTotal());			//총 컨텐츠양(count)
       System.out.println(queryResults.getLimit());			
       System.out.println(queryResults.getOffset());			
       System.out.println(queryResults.getResults().size());	 //페이징한 컨텐츠 사이즈
   }
   ```

   이 상황에서 페이징 쿼리가 나가게 되는데 만약 조인이 되었을 경우 join이 적용된 count 쿼리가 나가게 된다. 성능 최적화를 위해서는 별도의 join이 발생하지 않는 count 쿼리를 작성해야한다.

7. 집합과 groupby

   - count, sum, avg, max, min 등등 집합 함수를 제공한다.
   - groupby, having 메서드를 통해 그룹, 그룹 제한을 제공한다

   ```java
   public void test1(){	// 상품카테고리당 재고가 있는 상품들의 평균값 조회
       List<Tuple> result = queryFactory
           .select(category.name, product.price.avg())
           .from(product)
           .join(product.category, category)
           .gropuBy(category.name)
           .having(item.stockQuantity.gt(0))
           .fetch();
   }
   ```

8. 조인

   - 기본 조인

     join(조인 대상, 별칭으로 사용할 Q타입)

     ```java
     List<Product> result = queryFactory
         .selectFrom(product)
         .join(product.category, category)
         .where(product.name.eq("상품명"))
         .fetch();
     ```

     leftJoin, rightJoin도 이와 동일하게 사용하면 된다.

   - 세타 조인

     연관관계가 없는 필드로 조인하기

     ```java
     // 상품카테고리명과 상품명이 같은 상품 조회
     public void test(){
         List<Product> result = queryFactory
             .select(product)
             .from(product, category)
             .where(product.name.eq(category.name))
             .fetch();
     }
     ```

   - Join on

     1. 조인 대상을 필터링할 경우 사용
     2. 연관관계가 없는 엔티티를 외부 조인시 사용

     ```java
     // 1
     public void test1(){
         List<Tuple> result = queryFactory
             .select(product, category)
             .from(product)
             .leftJoin(product.category, category).on(category.name.eq("카테고리명"))
             .fetch();
     }
     
     // 2
     public void test2(){
         List<Tuple> result = queryFactory
             .select(product, category)
             .from(product)
             .leftJoin(category).on(category.name.eq(product.name))
             .fetch();
     }
     ```

     - 1번 동작 원리

       Left Join에 의해 Product의 Category 외래키 == Category id가 맞는 Product를 찾는다.

       다음 on메서드에 기입한 category.name.eq()의 파라미터 값에 일치하는 Category 명을 찾아 Product의 카테고리명과 일치하면 Product와 Category를 묶어서 Tuple로 리턴, 카테고리명과 일치하지 않는 카테고리는 null로 묶어서 리턴

     - 2번 동작원리

       연관되어있지 않기 때문에 Product와 Category의 카디션 곱 발생

       on절에서 Category명과 Product명이 일치하는 경우는 Product, Category 이렇게 Tuple로 묶여서 반환, 일치하는 경우가 없으면 Product, null로 묶어 반환

   - fetch join

     해당 연관관계에서 fetchType이 LAZY로 설정되어 있고 연관된 객체들을 한번에 조회하고 싶을 경우

     ```java
     public void test2(){
         Product product = queryFactory
             .select(product)
             .from(product)
             .join(product.category, category).fetchJoin()
             .where(product.name.eq("상품명"))
             .fetch();
     }
     ```

9. 서브 쿼리

   먼저 JPA, JPQL 에서는 인라인뷰(from 절의 서브쿼리)를 지원하지 않는다. 따라서 Querydsl에서도 지원하지 않으며 이에 해결책은 다음과 같다

   - 서브쿼리를 join으로 변경
   - 쿼리를 2번 분할하여 실행
   - nativeSQL 실행

   ```java
   // JPAExpressions를 사용하자. static import 가능
   
   // 상품값이 평균 이상인 상품들 조회
   public void test1(){
       // 별칭을 따로 생성해주어야 한다.
       QProduct productSub = new QProduct("productSub");
       
       queryFactory
           .selectFrom(product)
           .where(product.price.goe(
           	JPAExpressions
               	.select(productSub.price.avg())
               	.from(productSub)
           ))
           .fetch();
   }
   
   // 상품가격이 20000원 초과의 모든 상품들을 조회하라
   public void test1(){
       QProduct productSub = new QProduct("productSub");
       
       queryFactory
           .selectFrom(product)
           .where(product.price.in(
           	JPAExpressions
               	.select(productSub.price)
               	.from(productSub)
               	.where(productSub.price.gt(20000))
           ))
           .fetch();
   }
   // 상품명과 상품의 평균가격을 조회 평균가격은 다 같게 나온다.
   public void test1(){
       QProduct productSub = new QProduct("productSub");
       
       queryFactory
           .select(product.name,
                  JPAExpressions
                  	.select(productSub.price.avg())
                  	.from(productSub))
           .from(product)
           .fetch();
   }
   ```

10. Case문과 상수 더하기

    - Case 활용

    ```java
    //CaseBuilder 활용하기
    queryFactory
        .select(new CaseBuilder()
               	.when(product.price.between(0, 10000)).then("쌈")
                .otherwise("비쌈"))
        .from(product)
        .fetch();
    
    //10001~20000원 상품을 1번째, 20001~30000원 상품을 2번째, 그외를 세번째로 출력
    //NumberExpression을 활용해 select, orderBy절에서 활용가능
    NumberExpression<Integer> rank = queryFactory
        .select(new CaseBuilder()
               .when(product.price.between(10001, 20000)).then(1)
               .when(product.price.between(20001, 30000)).then(2)
               .otherwise(3));
    
    List<Tuple> result = queryFactory
        .select(product.name, category.name, rank)
        .from(product)
        .join(product.category, category)
        .orderBy(rank.asc())
        .fetch();
    ```

    - 상수, 문자더하기

    ```java
    //상수
    queryFactory
        .select(product.name, Expressions.constant("A+"))
        .from(product)
        .fetch();
    
    //문자
    queryFactory
        .select(product.name.concat("_").concat(product.price.stringValue()))
        .from(product)
        .fetch();
    ```

    
