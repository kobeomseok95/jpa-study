연관관계 매핑 기초
================

# 핵심 키워드
- ### 방향
  - 단방향 : 한 객체에서 참조하는 다른 객체를 한방향으로만 참조하는 경우
  - 양방향 : 한 객체에서 참조하는 다른 객체를 한방향으로 참조하는 것 뿐만 아니라 참조하는 객체에서 역방향으로도 참조할 수 있는 경우
- ### 다중성
  - 일대다 : 한 객체가 여러 객체에서 참조될 수 있는 경우
  - 다대일 : 한 객체 타입의 여러 객체가 다른 한 객체를 공통으로 참조할 경우
  - 다대다
  - 일대일
- ### 연관관계의 주인
  - 양방향으로 관계를 설정할 때, 연관관계의 주인을 정해야 한다.
  
## 단방향 연관관계
- 객체 연관관계 vs 테이블 연관관계
  - 해당 객체에 참조하는 객체가 있을 경우 단방향이다.
  ```java
  public class Member{
    private Long id;
    private String name;
    private Dept dept;  //해당 객체 참조, 단방향
  }
  ...


  public class Dept{
    private Long id;
    private String name;
  }
  ```
  - DB는 외래키 하나로 양방향으로 참조가 가능하다!
  - 객체 참조를 통한 연관관계에 있어 양방향은 사실 서로 다른 단방향 관계 2개이다.

## @ManyToOne
- 다대일 관계라는 매핑정보, 다중성을 나타내는 어노테이션을 필수로 써야한다
- 속성
  - optional : false로 설정할 경우 연관된 엔티티가 항상 있어야 한다.
  - fetch : 글로벌 페치전략(뒤에서!)
  - cascade : 영속성 전이 기능을 사용(뒤에서!)

## @JoinColumn
- 외래키를 매핑할 때 사용한다.
- 속성
  - name : 매핑할 외래키 이름(필드명_참조하는 테이블의 기본키 컬럼명 ex) @JoinColumn(name = "PRODUCT_ID") 즉, 해당 엔티티의 @Id의 name 속성!
  - referencedColumnName : 외래키가 참조하는 대상 테이블의 컬럼명
  - foreignKey(DDL) : 외래키 제약조건을 직접 지정할 수 있다.
  
## 연관관계 사용
- 저장
  - 주의 : JPA에서 엔티티를 저장할 때 연관된 모든 엔티티는 영속상태여야 한다
  ```java
  public static void main(String[] args) {
        EntityManagerFactory emf = Persistence.createEntityManagerFactory("hello");
        EntityManager em = emf.createEntityManager();
        EntityTransaction tx = em.getTransaction();
        tx.begin();

        try {
            Member2 member1 = new Member2();
            member1.setUsername("첫째");
            em.persist(member1);

            Member2 member2 = new Member2();
            member2.setUsername("둘째");
            em.persist(member2);

            Team team1 = new Team();
            team1.setName("팀1");
            em.persist(team1);

            member1.changeTeam(team1);
            member2.changeTeam(team1);

            tx.commit();
        } catch(Exception e) {
            tx.rollback();
        } finally {
            em.close();
        }
        emf.close();
    }
    /*
      멤버 등록쿼리 두번, 
      팀 등록쿼리 두번,
      멤버 수정쿼리 두번
    */
  ```
- 조회
  - 객체 그래프 탐색 : 객체를 통해 연관된 엔티티를 조회하는 방법 (ex) product.getCategory().getName(); > product에 속한 category도 가져온다.)
  - JPQL 사용
- 수정
  - 마찬가지로 트랜잭션 커밋시점에 flush가 일어나면서 변경 감지 기능이 작동한다.
- 연관관계 제거
  - null값으로 세팅하기 (ex) product.setCategory(null); )
- 연관관계 삭제
  - 기존에 있던 연관관계를 제거하고 삭제하자
    - 왜???? > 외래키 제약조건으로 인해 에러가 발생하니깐!!
    
## 양방향 연관관계
- @ManyToOne으로 매핑했던 반대 엔티티에 @OneToMany를 쓴 다음 해당 객체들을 컬렉션으로 받는다.
- 문제는 양쪽 엔티티에서 모두 관리하기 때문에 관리 포인트가 하나 더 늘어난다.
- 속성
  - mappedBy : 연관관계의 주인을 설정한다. 주인이 아닌 객체에서 mappedBy 속성값에 주인 필드명을 넣어준다. 참고로 주인은 mappedBy 속성을 사용하지 않는다.
  - 그렇다면 연관관계 주인을 누구로 두는게 좋은가?
    - 연관관계 주인 == 외래키 관리자
    - **테이블을 기준으로 외래키가 있는 테이블의 엔티티의 필드를 연관관계 주인으로 설정하자!**
    ```java
    //상품과 카테고리가 있다고 가정(한 카테고리에는 여러 상품이 있다)
    class Product{
      ...
      @ManyToOne
      @JoinColumn(name = "CATEGORY_ID")
      private Category category;
      ...
    }
    
    class Category{
      ...
      @OneToMany(mappedBy = "category")
      private List<Product> products = new ArrayList<Product>();
      ...
    }
    ```
    - 이 때, 주인이 아닌 경우는 읽기만 가능하다. 외래키를 변경 불가능!
- 저장
  - 단방향과 마찬가지다. 연관관계의 주인이 외래키를 관리하므로!
  - 양방향 연관관계의 주의점!
    - 만약 연관관계 주인이 아닌 엔티티에만 값을 설정하면 조회할 때 외래키 값이 null로 나온다!
    - 그렇다고 주인만 엔티티에 값을 설정하라는 것은 아님
    - 연관관계 편의 메서드를 통해 극복하자. 예를 들면
    ```java 
    public class Product{
      private Category category;
      
      public void addCategory(Category category){
        // 기존 관계 제거
        if (this.category != null){
          this.category.getProducts().remove(this);
        }
        
        this.category = category;
        category.getProducts().add(this); // 포인트! 또한 메서드 명을 setXXX로 하지말고 직접 지어서 설정하자!
      }
    }
    ```
