엔티티 매핑
===============

## @Entity
- 해당 어노테이션이 붙은 클래스는 JPA가 관리하는 클래스다.
- 속성
  - name : 사용할 엔티티 이름 지정 (default : 클래스명)
- 기본 생성자 필수 (생성자를 하나 이상 만들 때는 기본 생성자가 자동으로 만들어지지 않으므로 유의하자)
- final, enum, interface, inner에 사용불가
- 필드에 final 키워드 불가

## @Table
- 엔티티와 매핑할 테이블 지정             
- 속성              
  - name : 매핑 테이블 명            
  - catalog : catalog 매핑             
  - schema : schema 매핑                    
  - uniqueConstraints : DDL 생성시 유니크 제약조건을 만듬        

## @Id
- 기본키 할당 전략
  ```java
  Product product = new Product();
  product.setId("id1"); //id를 직접 할당한다.
  em.persist(product);
  ```
- 데이터베이스가 생성해주는 값을 사용하려면??
  - 직접 할당 : 기본 키를 애플리케이션에서 직접 할당
  - 자동 생성 : 대리 키를 사용한다.
    - IDENTITY : 기본 키 생성을 데이터베이스에 위임
      ```java
      @Id
      @GeneratedValue(strategy = GenerationType.IDENTITY)
      private Long id;
      ...
      
      Product product = new Product();
      em.persist(product);
      System.out.println(product.getId());  //값이 나온다.
      ```
      - IDENTITY로 전략을 사용하면 식별자 값 없이 persist가 가능하다.
      - 그리고 getId() 메서드에 값이 나온다는 말은?
      - 해당 전략은 곧 persist를 하게 되면 DB까지 갔다 온다는 얘기다.
      - 트랜잭션을 지원하는 쓰기 지연이 동작하지 않는다.
    - SEQUENCE : 데이터베이스의 시퀀스 사용
      ```java
      @Entity
      @SequenceGenerator(name = "MEMBER_SEQ_GENERATOR", //식별자 생성기 이름
              sequenceName = "MEMBER_SEQ",              //DB에 저장된 시퀀스 이름
              initialValue = 1, allocationSize = 50     //DDL 생성시에만 적용되고 처음 시작하는 수, 시퀀스 한번 호출에 증가하는 수(성능 최적화)
      )
      public class Procuct{
        @Id
        @GeneratedValue(strategy = GenerationType.SEQUENCE,
          initialValue = 1, allocationSize = 1)
        private Long id;
      }
      ```
      - persist() 호출시 시퀀스를 사용해 식별자를 조회한 다음, 엔티티에 할당하고 영속성 컨텍스트에 저장한다.
    - TABLE : 키 생성 테이블을 사용
      ```java
      @TableGenerator(                                    
        name = "MEMBER_SEQ_GENERATOR",                    //테이블 키 생성기
        table = "MY_SEQUENCES",                           //테이블 생성 이름
        pkColumnValue = "MEMBER_SEQ", allocationSize = 1  //해당 테이블 전략 테이블의 pk과 할당 크기
      )
      public class Product{
        @Id
        @GeneratedValue(strategy = GenerationType.TABLE, 
          generator = "BOARD_SEQ_GENERATOR")
        private Long id;
        ...
      }
      ```
      - 시퀀스 전략과 동작 방식은 같다.(시퀀스 대신 테이블만 쓰는 것 뿐)
      - SEQUENCE 전략보다 한번 더 DB와 통신한다 (왜?? > 테이블 전략이므로 테이블의 pk를 +1 해야하기 때문에!)
    - AUTO : DB방언에 맞게 자동으로 설정한다.

## @Column
- 속성
  - name : 테이블 컬럼 이름
  - nullable : null 값의 허용여부, default = true
  - unique : 한 컬럼에 간단히 유니크 제약조건을 걸 경우
  - columnDefinition : DB 컬럼정보를 직접 준다.
  - length : String의 경우만 사용가능, 문자 길이 제약조건이다.
  - precision : 소수점을 포함한 전체 자릿수 BigDemical, BigInteger에서 사용
  - scale : 소수의 자릿수 double, float에는 적용되지 않으며 아주 큰 숫자나 성밀한 소수에만 사용
  
## @Enumerated
- 자바의 Enum 타입을 매핑
- 속성
  - value : EnumType.STRING만 쓰자!

## @Lob
- BLOB, CLOB 타입과 매핑한다
- BLOB : byte[], java.sql.BLOB
- CLOB : String, char[], java.sql.CLOB

## @Transient
- 이 필드는 매핑하지 않고 자바에서만 사용한다.
