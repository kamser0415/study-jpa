# Fetch 조인  
페치 조인은 관계형 데이터베이스에서 쿼리 최적화 기법 중 하나로, 
즉시로딩과 지연로딩의 **단점을 극복하기 위해 등장**했습니다. 
**즉시로딩**은 연관된 모든 데이터를 한 번에 가져오지만, 
필요하지 않은 데이터를 불러올 수 있고, 
**지연로딩**은 필요할 때만 데이터를 가져오지만 **N+1 쿼리 문제**로 성능 저하를 초래할 수 있습니다.

페치 조인은 쿼리를 실행하여 한 번에 필요한 데이터를 가져오는 방법으로, 
객체 그래프를 통해 연관된 엔티티를 즉시로딩처럼 한 번에 가져올 수 있습니다. 
이를 통해 `N+1 조회 문제`를 해결하고, 
**_원하는 시점에 필요한 데이터를 적절히 로딩하여 성능을 향상_** 시킬 수 있습니다.

#### 페치 조인(`fetch join`)은 주어진 연관 관계의 **지연로딩을 무시**합니다.  

페치 조인은 내부 조인(inner join) 또는 외부 조인(outer join)일 수 있습니다.

조인 페치(`fetch`) 또는 명시적으로 말하자면 `inner join fetch`는 **연관된 엔티티를 가진 기본 엔티티만 반환**합니다.

`left join fetch` 또는 더 상세히 말하자면 `left outer join fetch`는 연관된 조인된 엔티티가 없는 **기본 엔티티를 포함하여 모든 기본 엔티티를 반환**합니다.

+ 하이버네이트의 중요한 기능이며 `N+1 selects`문제를 해결해줍니다.
+ 공식문서 **Synax**
  + Book과 Publisher는 다대일 연관관계입니다.
  + Book과 Authors는 일대다 연관관계입니다.
      ```jpaql
      select book
      from Book as book
          left join fetch book.publisher
          join fetch book.authors
      ```  
    이 예시에서는 book.publisher에 대해 `left outer join`을 사용했습니다.   
    왜냐하면 **발행사 정보가 없는 책들도 함께 가져오고자 했기 때문**입니다.   
    반면 `book.authors`에 대해서는 모든 책이 적어도 하나의 저자를 가지기 때문에 
    일반적인 `inner join`을 사용했습니다.  

## Fetch 조인 주의사항(공식문서)
+ 단일 쿼리에서 연속적으로 또는 병렬로 여러 `@xxxToOne` 연관을 가져오는 것은 안전합니다.  
+ 단일 시리즈의 중첩된 `fetch join`도 괜찮습니다.
+ 병렬로 `여러 컬렉션` 또는 `@xxxToMany` 연관을 가져오면 데이터베이스 수준에서 **카테시안 곱**이 발생하며 성능이 매우 나빠질 수 있습니다.  
    예) `1 -> N -> N =>(  1 * N * N ) record`
+ HQL에서는 조인된 엔티티에 제약을 적용하는 것은 금지되지 않지만,일반적으로 좋은 방법은 아닙니다.  
왜냐하면 가져온 **컬렉션의 요소가 불완전할 수 있기 때문**입니다.
+ 실제로, 중첩된 fetch join을 명시하는 목적 이외에는 fetch된 조인된 엔티티에 식별 변수를 할당하는 것조차도 피하는 것이 좋습니다.

+ **페이징이나 제한된 쿼리에서는 보통 페치 조인을 피해야 합니다.**  
  + Query의 `setFirstResult()` 및 `setMaxResults()` 메서드로 지정된 제한이 있는 쿼리
  + HQL에서 설명하는 제한 또는 오프셋이 있는 쿼리 (아래의 "Limits and offsets"에서 설명)
  + Query 인터페이스의 scroll() 및 stream() 메서드와 함께 사용해서는 안 됩니다.
  + 페치 조인은 의미가 없는 하위 쿼리(subquery)에서 금지됩니다.  
  
## 사용방법  

### 1. N+1이 발생하는 이유

연관관계를 **지연로딩**으로 설정한 상태에서, 
일대다 관계에서 **프록시 객체의 특정 필드를 조회**할 때마다 해당 필드의 데이터를 가져오기 위해 
데이터베이스에 **추가적인 쿼리를 실행**합니다. 이로 인해 N+1 쿼리 문제가 발생하는데, 
이는 프록시 객체에서 필드를 접근할 때마다 데이터베이스에서 데이터를 가져오기 때문입니다.

+ 테스트코드
```java
@DisplayName("N+1 발생시키기 일대다 관계")
@Test
void t4(){

    Team t1 = new Team();
    t1.setName("T1");
    em.persist(t1);
    Team gen = new Team();
    gen.setName("GEN.G");
    em.persist(gen);

    Member faker = new Member();
    faker.setUsername("페이커");
    faker.resigned(t1);
    Member keria = new Member();
    keria.setUsername("케리아");
    keria.resigned(t1);
    Member choby = new Member();
    choby.setUsername("쵸비");
    choby.resigned(gen);

    em.persist(faker);
    em.persist(keria);
    em.persist(choby);

    em.flush();
    em.clear();

    List<Team> lck = em.createQuery("select t from Team as t join t.members", Team.class).getResultList();

    for (Team team : lck) {
        for (Member member : team.getMembers()) {
            System.out.println("member = " + member);
        }
    }
}
```  
```sql
Hibernate: 
    select
        team0_.id as id1_4_,
        team0_.name as name2_4_ 
    from
        Team team0_ 
    inner join
        Member members1_ 
            on team0_.id=members1_.team_id
Hibernate: 
    select
        members0_.team_id as team_id8_1_0_,
        members0_.id as id1_1_0_,
        members0_.id as id1_1_1_,
        members0_.addressAlias_id as addressa7_1_1_,
        members0_.city as city2_1_1_,
        members0_.street as street3_1_1_,
        members0_.zipcode as zipcode4_1_1_,
        members0_.age as age5_1_1_,
        members0_.team_id as team_id8_1_1_,
        members0_.username as username6_1_1_ 
    from
        Member members0_ 
    where
        members0_.team_id=?
# member =  Member {id=3, username='페이커}
# member =  Member {id=4, username='케리아}
# member =  Member {id=3, username='페이커}
# member =  Member {id=4, username='케리아}
Hibernate: 
    select
        members0_.team_id as team_id8_1_0_,
        members0_.id as id1_1_0_,
        members0_.id as id1_1_1_,
        members0_.addressAlias_id as addressa7_1_1_,
        members0_.city as city2_1_1_,
        members0_.street as street3_1_1_,
        members0_.zipcode as zipcode4_1_1_,
        members0_.age as age5_1_1_,
        members0_.team_id as team_id8_1_1_,
        members0_.username as username6_1_1_ 
    from
        Member members0_ 
    where
        members0_.team_id=?
# member =  Member {id=5, username='쵸비}
```
분명 팀을 참조하는 선수의 엔티티를 개별로 조회를 했기 때문에 총 2번(페이커,케리아) 선수의 조회쿼리가 발생한다고 생각하지만 
실제 쿼리를 보면 팀을 참조하는 엔티티를 한 번에 불러옵니다. 

일대다 관계에서 JPA에서 List 타입의 엔티티를 조회할 때,
각각의 **member.id 별로 쿼리**를 실행하는 것이 아니라 **한 번에 모든 데이터를 한꺼번에 읽어옵니다.**
다시 말해, 일대다 관계에서는 연관된 여러 엔티티(Members)를 조회할 때,
하나의 조회 쿼리로 해당 일대다 관계에 속하는 모든 엔티티(Members)를 가져온다는 것을 의미합니다.
따라서 각각의 엔티티를 참조할 때마다 별도의 조회 쿼리가 발생하는 것이 아니라,
**_한 번의 쿼리로 모든 데이터를 읽어온 뒤 해당 리스트에서 필요한 정보를 추출할 수 있습니다._**
  
+ 테스트코드
   ```java
  @DisplayName("일대다 관계 Bag 정체")
  @Test
  void t6(){

      Team t1 = new Team();
      t1.setName("T1");
      em.persist(t1);
      Team gen = new Team();
      gen.setName("GEN.G");
      em.persist(gen);

      Member faker = new Member();
      faker.setUsername("페이커");
      faker.resigned(t1);
      Member keria = new Member();
      keria.setUsername("케리아");
      keria.resigned(t1);
      Member choby = new Member();
      choby.setUsername("쵸비");
      choby.resigned(gen);

      em.persist(faker);
      em.persist(keria);
      em.persist(choby);

      em.flush();
      em.clear();
      
      Team findT1 = em.createQuery("select t from Team as t join t.members where t.id = 1", Team.class).getSingleResult();

      List<Member> members = findT1.getMembers();
      System.out.println("members.getClass() = " + members.getClass());
      for (Member member : members) { // 회원들은 프록시 객체일까요?
          System.out.println("member.getClass() = " + member.getClass());
      }
   }
    //members.getClass() = class org.hibernate.collection.internal.PersistentBag
    //member.getClass() = class hello.Member
    //member.getClass() = class hello.Member
   ```
  만약 팀마다 다 다른 선수라면 N+1 쿼리가 동작하게 됩니다.  

### 2. 페치조인 사용하기  
이제 선수와 팀을 모두 조회해서 불필요한 쿼리를 줄여보겠습니다.
```jpaql
select t from Team as t join fetch t.members
```  
```sql
select
    members1_.id as id1_1_1_,
    team0_.name as name2_4_0_,
    members1_.team_id as team_id8_1_1_,
    members1_.username as username6_1_1_,
from
    Team team0_ 
inner join
    Member members1_ 
        on team0_.id=members1_.team_id
```
지연 로딩으로 발생하는 불필요한 쿼리 대신에 한번에 엔티티를 매핑할 수 있습니다.   
**페치 조인은 즉시로딩과 쿼리가 동일합니다. 다만 동적으로 원하는 시점에 조회가 가능합니다.**  
  
**페치 조인은 지연로딩이 아니기 때문에 영속성 컨텍스트에 분리되어 준영속 상태가 되어도 
연관된 선수를 조회할 수 있습니다.**  
  
### 중복 객체 
데이터를 조회해서 매핑된 엔티티를 조회해보면 2개가 아니라 3개가 됩니다.  
실제 SQL을 조회하면 다음과 같습니다.  

| TEAM_ID | NAME  | ID | USERNAME | Entity 매핑     |
|---------|-------|----|----------|---------------|
| 1       | T1    | 3  | 페이커      | Entity 1번     |
| 1       | T1    | 4  | 케리아      | Entity 2번(중복) |
| 2       | GEN.G | 5  | 쵸비       | Entity 3번     |
  
JPA는 데이터베이스가 반환한 데이터를 줄마다 매핑을 하고 연관관계 엔티티의 참조를 넣어서 반환합니다.  
그래서 조회 결과 row마다 하나의 엔티티가 만들어지고 이미 존재하는 엔티티라면 영속성 컨텍스트에서 그대로 반환합니다.  
  
```java
for (Team team : lck) {
  //팀의 members 메모리 주소
  System.out.printf(
          "team address = %s, " +
          "team.members.hashCode = %s\n",team,team.getMembers().hashCode());
}
//team address = hello.Team@6a9344f5, team.members.hashCode = 372261610
//team address = hello.Team@6a9344f5, team.members.hashCode = 372261610
//team address = hello.Team@2fe74516, team.members.hashCode = 1492897838
```   
JPA도 데이터베이스가 반환한 결과를 엔티티에 매핑해서 사용자에게 전달하는 역할이기 때문에 중복된 객체가 발생하게 됩니다. 

#### 중복 제거 방법
1. DISTINCT
    ```jpaql
    select distinct t from Team as t join t.members
    ```
    ```sql
    select
      distinct t.id,
               t.name
    from Team t
        inner join Member m on t.id = m.team_id
    where t.id = 1
    ```  
    실질적으로 SQL 결과를 줄이지는 못하고 JPA 내부에서 중복을 제거합니다.  
    만약 SQL에 DISTINCT 쿼리가 실행되는게 싫을 수 있습니다.
2. HashSet()  
   결과를 hashSet<>();에 담으면 중복이 제거가 됩니다.
    ```java
    List<Team> skt = em.createQuery("select t from Team as t join fetch t.members", Team.class).getResultList();
    Set<Team> result = new HashSet<>(skt);
    ```

## 페치 조인의 특징과 한계  
페치 조인은 WHERE에 특정 조건을 만족하는 엔티티만 읽어 올 수 없습니다. 
1. 페치 조인은 객체 그래프를 통해서 모든 엔티티에 접근할 수있어야 합니다.
2. 데이터 정합성이 깨집니다.  
3. JPA는 데이터 베이스가 반환된 정보만 가지고 매핑을 하기 때문에 불안전합니다.
  
만약 1:N을 페치 조인과 페이징을 사용할 경우 메모리 부족으로 예외가 발생할 수 있습니다.  
페이징으로 일부만 가져올 경우 JPA는 그 엔티티 연관관계가 전체인지 일부인지 알 수 없습니다. 
데이터베이스가 반환된 결과를 매핑하기 때문에 내부 메모리에서 모두 읽어 저장한 다음 
페이징시 메모리에 있는 데이터를 제공하는 방식으로 동작합니다.  
  
만약 페이징을 원할 경우 일대다 관계에서 다대일 관계로 주 엔티티를 변경해서 읽어오는 방법을 사용합니다.

### Batch로 N+1을 해결해보기(지연로딩)  
batch는 읽어올 데이터를 WHERE XXXX IN (...) 방식으로 읽어옵니다. 
여기서 (...)에 몇개의 조건을 Max으로 설정할 값을 넣어주면 됩니다(1000 이하).  
  
+ 애노테이션 활용 (`@BatchSize`)
  ```java
  @Entity
  public class Team {
    @Id @GeneratedValue
    private Long id;
    private String name;
    
    @OneToMany(mappedBy = "team",fetch = FetchType.LAZY)
    @BatchSize(size = 2)
    private List<Member> members = new ArrayList<>();
  }
  ```
+ 설정 정보(xml,properties) 작성 
  ```xml
  <property name="hibernate.jdbc.batch_size" value="5"/>
   ```  
+ 테스트 코드
  ```java
  List<Team> findT1 = em.createQuery("select distinct t from Team as t join t.members", Team.class).getResultList();
   ```
  ```sql
  select
        m.id,
        m.team_id,
        m.username
    from
        Member m 
    where
        m.team_id in ( ?, ? )
  ```
페치조인과 Batch의 차이점은 페치 조인은 연관된 엔티티를 한번에 조인으로 가져오는 방식이라면  

Batch는 지연로딩으로 프록시 객체가 데이터베이스 접근할 때 매번 조회하는 방식에서 
`@BatchSize(value=5)`라면 5개씩 모아서 조회 쿼리를 실행하는 방식입니다.  
```sql
select * from member where id = 1;
select * from member where id = 2;
select * from member where id = 3;
-- 이렇게 하나씩 조회쿼리를 실행하지 않고 BatchSize만큼 모아서 실행합니다.
select * from member where id in (1,2,3);
```
  
### 페치조인과 DTO  
패치 조인 쿼리를 보면 아래와 같습니다.
```jpaql
select
    T.* , M.*
from
    Team T 
inner join
    Member M on T.id=M.team_id
```  
쿼리를 보면 Team을 참조하는 Member의 엔티티를 모두 읽어옵니다.    
여러 테이블을 조인해서 엔티티가 가진 모양이 아닌 **전혀 다른 결과**를 내야 하면, 
페치 조인 보다는 일반 조인을 사용하고 **필요한 데이터들만 조회해서 DTO로 반환하는 것이 효과적**입니다.  
  
[하이버 네이트 반환 타입 공식 문서 설명](https://docs.jboss.org/hibernate/orm/6.4/querylanguage/html_single/Hibernate_Query_Language.html#returning-to-java)
+ 자바에서 데이터를 다루는 가장 불편한 측면 중 하나는 테이블을 표현하는 좋은 방법이 없다는 것입니다.  
  자바의 근본적인 문제는 튜플 유형(tuple types)이 없다는 것입니다.

#### 제일 안전한 방법은 인스턴스화 하여 반환하는 방법을 사용합니다.  
```java
record BookSummary(String title, String summary) {}

List<BookSummary> results =
        entityManager.createQuery("select new BookSummary(title, left(book.text, 200)) from Book",
                                  BookSummary.class)
            .getResultList();
for (var result : results) {
    String title = result.title();
    String preamble = result.summary();
}
```
```text
selection
    : (expression | instantiation) alias?
instantiation
    : "NEW" instantiationTarget "(" selection ("," selection)* ")"
alias
    : "AS"? identifier
```  
  
+ 테스트 코드  
  ```java
  // TeamInfo DTO
  public class TeamInfo {
    private String members;
    private Double averageAge;
    public TeamInfo(String members, Double averageAge) {
        this.members = members;
        this.averageAge = averageAge;
    }
    @Override
    public String toString() {
        return "TeamInfo{" + 
                "members='" + members + '\'' +
                ", averageAge=" + averageAge +
                '}';
    }
  }
  ```
  ```java
  //테스트용 메서드
  private Member createMember(String memberName, Team team, int age) {
    Member member = new Member();
    member.setUsername(memberName);
    member.setAge(age);
    member.resigned(team);
    return member;
  }
  private Team createTeam(String name) {
        Team t1 = new Team();
        t1.setName(name);
        return t1;
  }
  //테스트 코드
  @DisplayName("인스턴스화하여 반환하기")
  @Test
  void t8(){
      Team t1 = createTeam("T1");
      em.persist(t1);

      Member faker = createMember("페이커", t1, 20);
      Member keria = createMember("케리아", t1, 27);
      Member guma = createMember("구마유시", t1, 22);
      Member oner = createMember("오너", t1, 22);
      Member zeus = createMember("제우스", t1, 20);

      em.persist(faker);
      em.persist(keria);
      em.persist(guma);
      em.persist(oner);
      em.persist(zeus);

      em.flush();
      em.clear();

      String qlString = "select new hello.TeamInfo(group_concat(m.username),avg(m.age)) " +
              "from Member as m " +
              "join m.team " +
              "where m.team.name = :teamName";
      List<TeamInfo> sqlResult = em.createQuery(qlString, TeamInfo.class)
              .setParameter("teamName", "T1")
              .getResultList();
      for (TeamInfo teamInfo : sqlResult) {
          System.out.println("teamInfo = " + teamInfo);
          // teamInfo = TeamInfo{members='페이커,케리아,구마유시,오너,제우스', averageAge=22.2}
      }
  }
  ```  
```jpaql
select new hello.TeamInfo(group_concat(m.username),avg(m.age))
from Member as m
join m.team
where m.team.name = :teamName
```
#### 정리  
페치조인은 동적으로 연관관계 엔티티를 JPQL로 작성하여 한꺼번에 읽어올 수 있습니다. 
객체 그래프를 활용하기 위한 방법이기 때문에 필터링이나 페이징은 권장하지 않습니다.  
  
엔티티를 그대로 사용하는게 아니라 가공하거나 필요한 데이터만 가져올 경우에는 
DTO를 활용하는 방식을 사용합니다.  
