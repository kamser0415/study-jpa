# Named 쿼리   
<!-- TOC -->
* [Named 쿼리](#named-쿼리-)
      * [예제](#예제)
      * [테스트 코드](#테스트-코드)
    * [참고](#참고-)
    * [NamedQuery를 XML에 정의](#namedquery를-xml에-정의-)
      * [xml 참고](#xml-참고)
      * [환경에 따른 설정](#환경에-따른-설정)
<!-- TOC -->
공식문서
```text
It’s even better to specify the query string using 
the @NamedQuery annotation, 
so that Hibernate can validate the query it at startup time, 
that is, when the SessionFactory is created, 
instead of when the query is first executed.
```  

쿼리 문자열을 **@NamedQuery 어노테이션**을 사용하여 지정하는 것이 더 좋습니다. 
이렇게 하면 Hibernate이 쿼리를 처음 실행하는 시점이 아니라 SessionFactory가 생성될 때, 
즉 시작 시간에 쿼리를 유효성 검사할 수 있습니다.   

[공식문서링크](https://docs.jboss.org/hibernate/orm/6.4/introduction/html_single/Hibernate_Introduction.html#named-queries)
+ 미리 정의해서 이름을 부여해두고 사용하는 JPQL
+ 정적 쿼리
+ 어노테이션, XML에 정의
+ 애플리케이션 로딩 시점에 초기화 후 재사용
+ 애플리케이션 로딩 시점에 쿼리를 검증  

#### 예제
```java
@NamedQuery(name="10BooksByTitle",
            query="from Book where title like :titlePattern order by title fetch first 10 rows only")
class BookQueries {}
```
#### 테스트 코드
네임드 쿼리 만들기
```java
@NamedQuery( name="Member.findByIdForUpdate"
        ,query = "select m from Member as m where m = :member",
        lockMode = LockModeType.PESSIMISTIC_WRITE )
public class Member {}
```  
테스트 코드
```java
@DisplayName("네임드 쿼리를 써보자")
@Test
void t5(){
    Member member = new Member();
    member.setAge(17);
    member.setUsername("이상혁");
    em.persist(member);

    em.flush();
    em.clear();

    Member result = em.createNamedQuery("Member.findByIdForUpdate", Member.class)
            .setParameter("member", member)
            .getSingleResult();
    System.out.println("result = " + result);
}
```  
**SQL**
```sql
select
    member0_.*
from
    Member member0_ 
where
    member0_.id=? for update
```  

### 참고  
NamedQuery는 영속성 유닛 단위로 관리됩니다. 
충돌을 방지하기 위해 엔티티 이름을 앞에 두는 방식이 관리하기가 쉽습니다.  
예) `Member.findByUserName`
  
여러개를 등록하려면 `@NamedQueries`를 사용하면 됩니다.  

### NamedQuery를 XML에 정의  
JPA 어노테이션을 사용할 수 있지만, 자바에서 멀티라인을 관리하는건 까다롭기 때문에 
XML로 작성하는 것이 편리할 수 있습니다.  
+ xml 설정 및 등록
    ```xml
    //namedQueryForTest.xml
    <?xml version="1.0" encoding="UTF-8" ?>
    <entity-mappings xmlns="http://xmlns.jcp.org/xml/ns/persistence/orm" version="2.2">
        <named-query name="Team.findByIdForUpdate">
            <query><![CDATA[
                select t
                from Team as t
                where t = :team
            ]]></query>
            <lock-mode>PESSIMISTIC_WRITE</lock-mode>
        </named-query>
        <named-query name="Member.count">
            <query>select count(m) from Member m</query>
        </named-query>
    </entity-mappings>
    
    //persistence.xml
    <persistence-unit name="hello">
        <mapping-file>META-INF/namedQueryForTest.xml</mapping-file>
        <!--  .. 이하생략-->
    ```  
+ 테스트코드
    ```java
    @DisplayName("XML로 네임드 쿼리 사용해보기")
    @Test
    void t6(){
        Team team = new Team();
        team.setName("T1");
        em.persist(team);
    
        em.flush();
        em.clear();
    
        List<Team> result = em.createNamedQuery("Team.findByIdForUpdate", Team.class)
                .setParameter("team", team)
                .getResultList();
    
        System.out.println("result = " + result);
    }
    ```
+ **SQL**
    ```sql
    select
        team0_.*
    from
        Team team0_ 
    where
        team0_.id=? for update
    ```  
  
#### xml 참고
`META-INF/orm.xml`은 JPA가 기본 매핑 파일로 인식해서 별도의 설정을 하지 않아도 됩니다. 
이름이나 위치가 다르면 설정을 추가해야합니다. 현재 매핑 파일 이름이 `namedQueryForTest.xml`이므로 
위치 정보를 추가했습니다.  
  
#### 환경에 따른 설정
+ XML이 항상 우선권을 가진다.
+ 애플리케이션 운영 환경에 따라 다른 XML을 배포할 수 있다.