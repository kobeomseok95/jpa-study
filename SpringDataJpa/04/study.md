스프링 데이터 JPA 분석
================================

### 1. 스프링 데이터 JPA 구현체 분석

IDE에서 SimpleJpaRepository 클래스를 들어가보면

![image](https://user-images.githubusercontent.com/37062337/103202253-3e2a8380-4935-11eb-86b6-a078454478bc.png)

이렇게 @Repository와 @Transactional 어노테이션이 존재한다.

- @Repository
  - 컴포넌트 스캔의 대상이 된다.
  - JPA 예외를 스프링이 추상화한 예외로 변환한다. **하부 구현의 내용이 변경되어도 영향을 받지 않는다!**

- @Transactional
  - JPA의 모든 변경(insert, update, delete)는 트랜잭션 안에서 동작한다.
  - @Service에서 트랜잭션을 시작하지 않더라도 해당 계층에서 트랜잭션을 시작하며 반대로 @Service에서 트랜잭션을 시작할 경우 해당 트랜잭션을 전파 받아서 사용한다.
  - 이러한 이유로 인해 트랜잭션이 없어도 등록 및 변경, 삭제가 가능했던 것이다!
  - **readOnly = true의 이유** : 대부분의 쿼리는 조회 쿼리가 많다! 조회의 경우에는 영속성 컨텍스트에서 flush나 변경 감지가 굳이 일어나지 않아도 되므로 readOnly가 true로 설정이 되어있을 때는 flush, 변경 감지 등의 데이터 변경에 필요한 동작들이 생략된다.

### 2. 새로운 엔티티를 구별하는 방법 (save 메서드)

save 메서드를 보자

![image](https://user-images.githubusercontent.com/37062337/103203789-163d1f00-4939-11eb-88a7-68c927d99857.png)

- 메서드에 @Transactional이 걸려있다. 이 경우에는 readOnly = true가 적용되지 않는다.
- isNew 메서드를 통해 해당 객체의 id를 검사한다.
- id가 null(객체의 경우)이거나 0(primitive type의 경우)일 경우 persist
- id가 존재할 경우 merge

**이슈와 주의사항**

@GeneratedValue가 식별자 생성 전략이라면 save() 시점에 식별자가 없으므로 persist가 정상적으로 동작한다. 하지만 생성전략을 @Id만 사용하고 직접 할당할 경우 이미 식별자 값이 존재하기 때문에 merge가 실행되며 select 쿼리 및 DB에 값이 없을 경우 새로운 엔티티로 인지하기 때문에 매우 비효율적이다.

- 해당 엔티티의 id에 @GeneratedValue를 쓰지 않을 수 있다. 이럴 때는?

  - isNew 메서드를 다른 방법으로 구현하자
  - 이전 시간에 배웠던 Auditing을 활용해보자

  ```java
  @Entity
  @Getter
  @EntityListeners(AuditingEntityListener.class)	// 1
  @NoArgsConstructor(access = AccessLevel.PROTECTED)	// 2
  public class Item implements Persistable<String> {	// 3
      @Id
      private String id;
  
      @CreatedDate    // 4
      private LocalDateTime createdDate;
  
      public Item(String id) {
          this.id = id;
      }
  
      @Override
      public boolean isNew() {
          return createdDate == null;
      }
  }
  ```

  1. Auditing을 설정하기 위한 어노테이션

  2. Repository 계층에서 엔티티를 가져올 때는 반드시 기본 생성자가 필요하다. 또한, 이 엔티티가 프록시 객체에서 진짜 객체 조회를 할 때도 마찬가지로 기본 생성자가 필요한데, public 으로 할 경우는 코드 상에서 생성자를 쓸 수 있다. private로 설정한다면 엔티티가 만들어지지 않는다. protected를 활용하여 실수 및 엔티티 클래스를 생성할 수 있도록 하는 lombok 어노테이션이다.
  3. isNew를 구현하기 위해 상속받은 인터페이스 제네릭 타입은 해당 엔티티의 id클래스
  4. @CreatedDate는 우선 해당 엔티티가 persist가 될 때 해당 값이 채워진다. 그 말은 persist가 되지 않은 때는 null 값인 상태인데. 이때 isNew() 메서드에서 createdDate == null인 경우를 true로 반환하여 새로운 엔티티임을 인식하게 해준다. > select 쿼리를 안날려도 된다!

- merge의 경우 해당 id와 일치하는 엔티티를 DB에서 조회한 다음, 파라미터로 들어온 정보들로 싹 덮어버린다.

  - 이 경우에 어떤 문제가 발생하냐면 파라미터에서 null이 들어와도 무조건 null로 해당 엔티티의 정보들이 덮인다.
  - 변경을 할 경우에는 반드시 트랜잭션 내에서 변경 감지를 활용하자

