Spring Data Jpa란?
===========================

Spring Data 안에 있는 모듈로, 개발자가 JPA를 좀 더 쉽고 편하게 사용할 수 있도록 도와준다.
JPA를 한단계 추상화시킨 Repository라는 인터페이스를 제공함으로써 이루이진다.

개발자가 단순한 쿼리 및 애플리케이션의 Data Access Layer를 구현하는것을 Spring Data Jpa가 규칙에 맞게 제공한다는 것!

기존의 JPA + Hibernate를 사용하여 기능을 구현했을 때는 EntityManager를 직접 사용했지만 Spring Data Jpa에서는 다음과 같이 Repository 계층을 생성 및 메서드 명명 규칙에 맞는 쿼리를 만들어 실행 시켜준다.

```java
//example
public interface MemberRepository extends JpaRepository<Member, Long> {
    List<Member> findByUsernameAndAgeGreaterThan(String username, int age);
}
```

기존의 JPA + Hibernate 와 Spring Data Jpa의 차이점은

- 기존의 Repository를 클래스로 작성했는데, Spring Data Jpa를 사용하면 JpaRepository<T, ID>를 상속받은 인터페이스로 작성한다.
- JpaRepository에서 T : Entity Class, ID = 해당 Entity의 Id 타입

> 인터페이스인데 동작하는 이유??

```java
    @Test
    public void memberRepositoryTest() {
        System.out.println("memberRepository = " + memberRepository);
        System.out.println("memberRepository = " + memberRepository.getClass());
    }

//memberRepository = springData.jpa.repository.MemberRepositoryImpl@2ec23ec3
//memberRepository = class com.sun.proxy.$Proxy104
```

테스트를 돌려보았다. Spring Data Jpa가 memberRepositoryImpl 이라는 클래스를 구현하여 클래스를 만들어주었고, memberRepository.getClass()를 출력했을 때, 해당 객체는 프록시 객체라는 것을 알 수 있다.

또한, @Repository를 생략해도 된다. Spring Data Jpa가 구현한 XXXImpl 클래스들을 컴포넌트 스캔의 대상이 되도록한다!

구조는 JpaRepository > PagingAndSortingRepository > CrudRepository > Repository 구조이며, JpaRepository는 Spring Data Jpa, PagindAndSortingRepository부터 상위 영역은 Spring Data 영역이다.

> JpaRepository는 Jpa를 사용하는 개발자들에게 필요로 하지만 페이징, 정렬, CRUD는 어느 DB를 쓰던간에 필요한 기능들이기 때문에 이렇게 분리한 것이다.

