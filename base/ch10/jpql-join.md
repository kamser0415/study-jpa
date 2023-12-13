# 조인  
<!-- TOC -->
* [조인](#조인-)
    * [내부조인](#내부조인-)
    * [외부조인](#외부조인-)
    * [컬렉션 조인](#컬렉션-조인)
    * [세타조인](#세타조인-)
    * [JOIN ON 절](#join-on-절-)
<!-- TOC -->
JPQL도 조인을 지원합니다. SQL 조인과 기능은 같고 문법만 다릅니다.  
  
### 내부조인 
`INNER JOIN`을 사용합니다. `INNER`는 생략가능합니다.  
```java
//Member.class 연관관계 매서드 추가
public void resigned(Team team) {
    if (this.team != null) {
        this.team.getMembers().remove(this);
    }
    this.team = team;
    team.getMembers().add(this);
}
@DisplayName("내부 조인 연관관계가 있는 엔티티끼리")
@Test
void t2(){
    Team t1 = new Team();
    t1.setName("T1");

    Member faker = new Member();
    faker.setUsername("이상혁");
    faker.resigned(t1);

    em.persist(t1);
    em.persist(faker);
    //초기화
    em.flush();
    em.clear();

    Member result = em.createQuery("select m from Member as m join m.team as t", Member.class).getSingleResult();

    Assertions.assertEquals(result.getUsername(), "이상혁");
    Assertions.assertEquals(result.getTeam().getName(), "T1");
}
```
```sql
//jpql
select m from Member as m join m.team as t
    
//sql
select
    member0_.id as id1_0_,
    member0_.age as age2_0_,
    member0_.team_id as team_id4_0_,
    member0_.username as username3_0_ 
from
    Member member0_ 
inner join
    Team team1_ on member0_.team_id=team1_.id
```  
JPQL과 SQL의 내부 조인 구문이 약간 다릅니다. 
JPQL의 조인의 가장 큰 특징은 연관관계 필드를 사용합니다. 
`m.team as t`는 연관관계 필드인데 연관관계 필드는 다른 엔티티와 
연관관계를 가지기 위해 사용하는 필드를 말합니다.  
  
SQL 조인처럼 사용하는건 안됩니다. 
```sql
SELECT m FROM Member as m JOIN Team as t
```  
권장하지 않지만 SQL처럼 사용하려면 다음과 같이 작성해야합니다.  
```sql
select m from Member as m join Team as t on m.team.id = t.id
```  

### 외부조인  
```java
String qlString = "select m from Member as m left join m.team as t";
Member result = em.createQuery(qlString, Member.class).getSingleResult();
```
```sql
select
    member0_.id as id1_0_,
    member0_.age as age2_0_,
    member0_.team_id as team_id4_0_,
    member0_.username as username3_0_ 
from
    Member member0_ 
left outer join
    Team team1_ 
        on member0_.team_id=team1_.id
```  
지연로딩으로 설정해서 Member엔티티만 읽어오는것을 확인할 수 있습니다.  
   
### 컬렉션 조인
일대다 관계나 다대다 관계처럼 컬렉션을 사용하는 곳에 조인을 하는 것을 컬렉션 조인이라 합니다.  
  
+ 선수 -> 팀 으로 선수가 기준인 경우 다대일 조인이면서 단일 값 연관 필드(m.team)을 사용합니다.
+ 팀-> 선수는 일대다 조인이면서 컬렉션 값 연관 필드(m.members)를 사용합니다.  
```java
String qlString = "select t from Team as t join t.members m";
List<Team> result = em.createQuery(qlString, Team.class).getResultList();
```  
```sql
select
    team0_.id as id1_3_,
    team0_.name as name2_3_
from
    Team team0_
        inner join
    Member members1_
    on team0_.id=members1_.team_id
```  

### 세타조인  
WHERE 절을 이용해서 세타 조인을 할 수 있습니다. 
세타 조인은 내부 지원합니다. 전혀 관계없는 엔티티도 조인할 수 있습니다.  
최신 하이버네이트에서는 외부 조인도 지원합니다.
예제는 선수의 이름과 팀의 이름을 조인했습니다.
```jpaql
select book.title, publisher.name
from Book book, Publisher publisher
where book.publisher = publisher
  and book.title like :titlePattern  
```  
```java
String qlString = "select t from Team as t , Member as m where t.name = m.username";
```
```sql
select
    team0_.id as id1_3_,
    team0_.name as name2_3_ 
from
    Team team0_ cross 
join
    Member member1_ 
where
    team0_.name=member1_.username

```  

### JOIN ON 절  
INNER JOIN, OUTER JOIN시 대상 테이블이나 주 테이블에 필터링을 하고 조인할 수 있습니다. 
내부조인의 ON절은 WHERE 절을 사용할 때와 결과가 같으므로 보통 ON절은 외부 조인에서만 사용합니다. 
```java
String qlString = "select m,t from Member as m left join m.team as t on t.name = 'T1'";
```
```sql
select
    member0_.*,
    team1_.*
from
    Member member0_ 
left outer join
    Team team1_ 
        on member0_.team_id=team1_.id 
        and ( team1_.name='T1' )
```  
이렇게 조인을 할 경우는 
선수 테이블은 유지하고 팀 이름이 `T1`인 팀 테이블만 가지고 조인을 하는 겁니다.  
  
엔티티를 여러개 읽어올 경우 아래와 같이 처리도 가능합니다.
```java
List<Object[]> result = em.createQuery(query).getResultList();
for(Object[] row: result){
    Member member = (Member) row[0];
    Team member = (Team) row[0];
}
```  
만약 Object로 꺼낸 엔티티도 영속성 컨택스트에서 관리를 합니다.
```java
List<Object[]> resultList = 
        em.createQuery("select m,t from Member as m left join m.team as t", Object[].class).getResultList();
Object[] objects = resultList.get(0);
Object object = objects[0];

Member result = 
        em.createQuery("select m from Member as m left join m.team as t", Member.class).getSingleResult();
System.out.println("Object 와 Entity 클래스도 동일할까 " +(object == result));
//true 가 나옵니다.
```  
