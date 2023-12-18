# @EntityGraph  
연관된 엔티티들을 SQL 한번에 조회하는 방법입니다.  
[EntityGraph 공식문서](https://docs.spring.io/spring-data/jpa/reference/jpa/query-methods.html#jpa.entity-graph)  

`Member > Team`은 다대일로 지연로딩으로 설정되어있습니다.   
`team.name`을 조회하면 `N+1 조회`가 발생합니다. 
```Java
@Test
public void findMemberLazy() throws Exception { 
    //given
    //member1 -> teamA 
    //member2 -> teamB
    Team teamA = new Team("teamA"); 
    Team teamB = new Team("teamB");
    teamRepository.save(teamA); 
    teamRepository.save(teamB);
    memberRepository.save(new Member("member1", 10, teamA)); 
    memberRepository.save(new Member("member2", 20, teamB));
    em.flush(); 
    em.clear();
    //when
    List<Member> members = memberRepository.findAll(); 
    //then
    for (Member member : members) { 
            member.getTeam().getName(); 
        }
}
```  
> 참고:  
> //Hibernate 기능으로 확인  
Hibernate.isInitialized(member.getTeam())  
//JPA 표준 방법으로 확인  
PersistenceUnitUtil util = em.getEntityManagerFactory().getPersistenceUnitUtil();  
util.isLoaded(member.getTeam());   
>

#### JPQL 페치 조인
```java
@Query("select m from Member m left join fetch m.team")List<Member> findMemberFetchJoin();
```
스프링 데이터 JPA는 JPA가 제공하는 엔티티 그래프 기능을 편리하게 사용하게 도와준다. 이 기능을 사용하면 JPQL
없이 페치 조인을 사용할 수 있다. (JPQL + 엔티티 그래프도 가능)
#### EntityGraph
```java
//공통 메서드 오버라이드
@Override
@EntityGraph(attributePaths = {"team"}) 
List<Member> findAll();
//JPQL + 엔티티 그래프
@EntityGraph(attributePaths = {"team"})
@Query("select m from Member m")
List<Member> findMemberEntityGraph();
//메서드 이름으로 쿼리에서 특히 편리하다.
@EntityGraph(attributePaths = {"team"}) 
List<Member> findByUsername(String username)
```
#### EntityGraph 정리
+ 사실상 페치 조인(FETCH JOIN)의 간편 버전  
+ LEFT OUTER JOIN 사용
#### NamedEntityGraph 사용 방법
```java
@NamedEntityGraph(name = "Member.all", attributeNodes =
@NamedAttributeNode("team"))
@Entity
public class Member {}  

@EntityGraph("Member.all")
@Query("select m from Member m")
List<Member> findMemberEntityGraph();
```  

### EntityGraphType에 따라 다른 결과가 나옵니다.  
1. Team은 `LAZY` 전략
2. Address는 `EAGER` 전략
```Java
public class Member {

    @ManyToOne(fetch = FetchType.LAZY)
    private Team team;

    @ManyToOne(fetch = FetchType.EAGER)
    private Address address;
    
}
```  
`MemberRepository`코드는 다음과 같습니다.
```Java
@EntityGraph(attributePaths = {"team"},type = EntityGraph.EntityGraphType.LOAD)
List<Member> findLOADByUsername(String username);
@EntityGraph(attributePaths = {"team"},type = EntityGraph.EntityGraphType.FETCH)
List<Member> findFETCHByUsername(String username);
```  
테스트 코드에는 각각 들어있지만 보기편하기 위해서 별도로 뺐습니다.
```Java
//== LAZY 타입
Team team1 = new Team("지구");
Team team2 = new Team("달");
em.persist(team1);
em.persist(team2);

//== EAGER 타입
Address address1 = new Address("개포동", 45);
Address address2 = new Address("아현동", 34);
em.persist(address1);
em.persist(address2);

Member member1 = new Member("둘리", 10,team1,address1);
Member member2 = new Member("둘리", 20,team2,address2);
memberRepository.saveAll(List.of(member1, member2));
```

#### EntityGraph.EntityGraphType.LOAD
테스트코드
```Java
@DisplayName("EntityGraphType.LOAD는 선택한 엔티티는 EAGER, 나머지는 엔티티 페치전략을 사용")
@TestFactory
Collection<DynamicTest> f2(){
    //when
    List<Member> findMembers = memberRepository.findLOADByUsername("둘리");
    Member member = findMembers.get(0);
    boolean teamProxy = Hibernate.isInitialized(member.getTeam());
    boolean addressEager = Hibernate.isInitialized(member.getAddress());

    //then
    return Arrays.asList(
            dynamicTest("teamProxy는 초기화 true",()->Assertions.assertThat(teamProxy).isTrue()),
            dynamicTest("addressPoxy 초기화 false",()->Assertions.assertThat(addressEager).isTrue())
    );
}
```  
**Member의 지연로딩인 Team과 EAGER인 Address 엔티티 모두 초기화되었습니다.**  

#### EntityGraph.EntityGraphType.FETCH
```Java
@DisplayName("EntityGraphType.FETCH는 선택한 엔티티는 EAGER, 나머지는 엔티티 LAZY 적용")
@TestFactory
Collection<DynamicTest> f1(){
    //when
    List<Member> findMembers = memberRepository.findFETCHByUsername("둘리");
    Member member = findMembers.get(0);
    boolean teamProxy = Hibernate.isInitialized(member.getTeam());
    boolean addressEager = Hibernate.isInitialized(member.getAddress());

    //then
    return Arrays.asList(
            dynamicTest("teamProxy는 초기화 true",()->Assertions.assertThat(teamProxy).isTrue()),
            dynamicTest("addressPoxy 초기화 false",()->Assertions.assertThat(addressEager).isFalse())
    );
}
```  
**LOAD와 차이점은 FETCH type은 선택한 엔티티만 초기화를 하고 나머지는 LAZY로 프록시객체로 초기화합니다.**  

