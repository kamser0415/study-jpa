# JPQL 기초  
+ 하이버네이트 공식 HQL 문서 [링크](https://docs.jboss.org/hibernate/orm/6.4/querylanguage/html_single/Hibernate_Query_Language.html)  
  <!-- TOC -->
* [JPQL 기초](#jpql-기초-)
      * [SQL](#sql)
      * [HQL(JPQL 하이버네이트 버전)](#hqljpql-하이버네이트-버전-)
    * [TypeQuery,Query](#typequeryquery-)
    * [결과값 조회](#결과값-조회)
    * [파라미터 바인딩](#파라미터-바인딩)
      * [파라미터 바인딩 필수](#파라미터-바인딩-필수-)
    * [프로젝션](#프로젝션-)
  * [페이징 API](#페이징-api-)
<!-- TOC -->

#### SQL
```sql
select book.title, pub.name            /* projection */
from Book as book                      /* root table */
    join Publisher as pub              /* table join */
        on book.publisherId = pub.id   /* join condition */
where book.title like 'Hibernate%'     /* restriction (selection) */
order by book.title                    /* sorting */
```  
#### HQL(JPQL 하이버네이트 버전)  
```jpaql
select book.title, pub.name            /* projection */
from Book as book                      /* root entity */
    join book.publisher as pub         /* association join */
where book.title like 'Hibernate%'     /* restriction (selection) */
order by book.title                    /* sorting */
```  
+ JPQL은 객체지향 언어입니다. 따라서 테이블을 대상으로 하는게 아니라 
    엔티티 객체를 대상으로 쿼리를 실행합니다.  
+ JPQL은 SQL을 추상화해서 특정 데이터베이스 SQL에 의존하지 않습니다. 
+ JPQL도 결국 SQL로 변합니다.  
  
>  HQL에서 "Book"은 자바로 작성된 엔티티 클래스를 나타내며, book.title은 해당 클래스의 필드를 가리킵니다. HQL 또는 JPQL에서 데이터베이스 테이블과 열을 직접 참조할 수는 없습니다. 이 쿼리 언어들은 엔티티 클래스와 그 필드들을 기반으로 작동합니다.
  
JPQL에서는 엔티티를 테이블로 생각해서 조인을 하는게 아니라 내부 필드 클래스와 조인을 한다고 생각하면 됩니다.  

| 원본 (Original)                                               | 번역 (Translation)                         |
|-------------------------------------------------------------|------------------------------------------|
| `select`, `SeLeCT`, `sELEct`, and `SELECT`                  | 모두 동일, `select`는 키워드입니다.                 |
| `upper(name)` and `UPPER(name)`                             | 동일, `upper`는 함수명입니다.                     |
| `from BackPack` and `from Backpack`                         | 다름, 서로 다른 자바 클래스를 가리킵니다.                 |
| `person.nickName` and `person.nickname`                     | 다름, `nickName`은 자바에서 정의된 엔티티의 속성을 가리킵니다. |
| `person.nickName`, `Person.nickName`, and `PERSON.nickName` | 모두 다름, 경로 표현식의 첫 번째 요소가 식별 변수이기 때문입니다.   |  
엔티티와 속성은 대소문자를 구분합니다. 반면 JPQL 키워드는 대소문자 구분을 하지 않습니다. 
추가로 별칭은 필수 입니다.

+ 추가 규칙  
  JPQL 명세서는 식별 변수를 대소문자 구분 없이 정의합니다. 그래서 엄격한 JPA 준수 모드에서 Hibernate는 person.nickName, Person.nickName 및 PERSON.nickName을 동일하게 처리합니다.

+ 기본 문법과 쿼리 API

| 절 (Clause)               | 용어 (Jargon)                           | 목적 (Purpose)                           |
|--------------------------|---------------------------------------|----------------------------------------|
| with                     | 공통 테이블 표현식 (Common table expressions) | 선언된 이름있는 서브쿼리를 다음 쿼리에서 사용할 수 있도록 함     |
| from and join            | 루트와 조인 (Roots and joins)              | 쿼리에 관련된 엔티티를 지정하고 서로 어떻게 관련되어 있는지를 명시함 |
| where                    | 선택/제한 (Selection/restriction)         | 쿼리에서 반환된 데이터에 대한 제약 조건을 명시함            |
| group by                 | 집계/그룹화 (Aggregation/grouping)         | 집계를 제어함                                |
| having                   | 선택/제한 (Selection/restriction)         | 집계 이후에 적용할 조건을 명시함                     |
| select                   | 프로젝션 (Projection)                     | 쿼리에서 반환할 대상을 명시함                       |
| union, intersect, except | 집합 대수 (Set algebra)                   | 여러 서브쿼리 결과에 적용되는 집합 연산자                |
| order by                 | 정렬 (Ordering)                         | 결과를 정렬하는 방식을 명시함                       |
| limit, offset, fetch     | 제한 (Limits)                           | 결과를 제한하거나 페이지네이션하는 데 사용함               |
  
### TypeQuery,Query  
작성한 JPQL을 실행하려면 쿼리 객체를 만들어야 합니다.  
+ TypeQuery: 반환할 타입이 명확하게 지정할 수 있을 때
+ Query: 반환할 타입을 지정할 수 없을 때  
  
+ `em.persist("jpql","기본적으로 엔티티 클래스를 넣습니다")`  
```java
// 반환타입이 엔티티 일경우
TypedQuery<Member> typedQuery1 = em.createQuery("select m from Member as m", Member.class);
// 반환타입이 기본형 타입으로 매핑할 수 있는 경우
TypedQuery<String> typedQuery2 = em.createQuery("select m.username from Member as m", String.class);
// 두 가지 이상의 타입을 받을 수 없기 때문에 Query로 받습니다.
Query query = em.createQuery("select m.username,m.age from Member as m");
```    
반환 타입은 다음과 같습니다.
```java
List<Member> typedList1 = typedQuery1.getResultList();
List<String> typedList2 = typedQuery2.getResultList();
// 구조는 List<Object[]> queryList 입니다.
List queryList1 = query.getResultList();
for (Object o : queryList1) {
    Object[] result = (Object[]) o;
    System.out.println("username = "+result[0]);
    System.out.println("age = "+result[1]);
}
```
`Query`로 반환타입일 때, SELECT 절의 조회 대상이 하나이면 `List<Object>`, 
SELECT 절의 조회 대상이 여러 개라면 `List<Object[]>`입니다.  
두 쿼리를 비교해보면 `TypedQuery`를 사용하는게 좋습니다. 

Map,List도 지원하지만 이 방법도 가능합니다.  
```java
import javax.persistence.tuple;

TypedQuery<Tuple> tupleObject = em.createQuery("select m.username as name ,m.age as age from Member as m", Tuple.class);
for (Tuple tuple : tupleObject.getResultList()) {
    System.out.println("이름 : "+tuple.get("name",String.class));
    System.out.println("나이 : "+tuple.get("age",Integer.class));
}
```  
`Object[]`, `List`, `Map` 또는 `Tuple` 중 어떤 형식도 형 변환 없이 결과 튜플의 개별 항목에 접근할 수 있는 방법을 제공하지 않습니다. 
Tuple은 get() 메서드에 클래스 객체를 전달하면 형 변환을 수행해주지만, 논리적으로는 동일합니다.  

### 결과값 조회
+ `List<T>` query.getResultList() :  
    결과를 `List<T>`로 반환합니다. 결과가 없으면 빈 배열이 반환됩니다.  
+ `Object` query.getSingleResult():  
    + 결과가 없으면 javax.persistence.NoResultException 예외 발생  
    + 결과가 1개보다 많으면 javax.persistence.NonUniqueResultException 발생  
    + 결과가 하나일때 사용합니다.  

### 파라미터 바인딩
JPQL에서는 두 가지 유형의 매개변수를 사용하며, HQL은 역사적인 이유로 세 번째 유형을 지원합니다.

| Parameter Type               | Examples                 | Usage from Java                    |
|------------------------------|--------------------------|------------------------------------|
| Named parameters (이름)        | `:name`, `:title`, `:id` | `query.setParameter("name", name)` |
| Ordinal parameters (순서)      | `?1`, `?2`, `?3`         | `query.setParameter(1, name)`      |
| JDBC-style parameters (JDBC) | `?`                      | `query.setParameter(1, name)`      |  
  
+ 이름 기준 파라미터: 이름으로 구분하는 방법 파라미터 앞에` : `를 사용합니다.
    ```java
    // JPA API
    int updatedEntities = entityManager.createQuery(
    "update Person p set p.name = :newName where p.name = :oldName")
    .setParameter("oldName", oldName)
    .setParameter("newName", newName)
    .executeUpdate();
    ```
+ 위치 기준 파라미터: 파라미터에` ? `을 넣고 위치 값는 1부터  시작합니다.
    ```java
    // JPA API
    int updatedEntities = entityManager.createQuery(
    "update Person p set p.name = ? where p.name = ?")
    .setParameter(1, oldName)
    .setParameter(2, newName)
    .executeUpdate();
    ```  

**이름 기준 파라미터 바인딩 방식을 사용하는 것이 더 명확합니다.**  
  
#### 파라미터 바인딩 필수  
```java
entityManager.createQuery(
        "select m from as m where m.username = '"+ usernameParam+"'")
```  
파라미터 바인딩을 사용하지 않고 직접 JPQL을 만들면 위험합니다. 
1. SQL 인젝션 공격을 당할 수  있습니다.
2. 성능 이슈가 있습니다.
    파라미터 바인딩 방식을 사용하면 파라미터의 값이 달라도 **같은 쿼리로 인식**해서 
    JPA는 JPQL을 SQL로 **파싱한 결과를 재사용**할 수 있습니다. 그리고 데이터베이스 내부에서 실행한 
    SQL을 파싱해서 사용하는데 **같은 쿼리는 파싱한 결과를 재사용**할 수 있습니다. 
    애플리케이션과 데이터베이스 모두 해당 쿼리의 파싱 결과를 재사용할 수 있습니다.
  
### 프로젝션  
SELECT 절에 조회할 대상을 지정하는 것을 프로젝션(projection)이라 하고 
[SELECT {프로젝션 대상} FROM]으로 대상을 선택합니다.   
  
+ 엔티티 프로젝션
    ```
    SELECT m FROM Member as m // 회원
    SELECT t FROM Member as m join m.team t // 팀 
    ```  
    엔티티로 관리되는 클래스로 반환할 경우 영속성 컨텍에서 관리합니다.  
  

+ 임베디드 타입 프로젝션  
    ```java
    String query = "SELECT o.address FROM Order as o";
    List<Address> addresses = em.createQuery(query,Address.class)
                                .getResultList();
    ```  
    인베디드 타입은 엔티티 조회하는 것처럼 클래스를 지정해서 조회가 가능합니다. 
    다만 값 타입이기 때문에 영속성 컨택스트에서 관리하지 않습니다.  
  
+ 스칼라 타입 프로젝션  
    숫자,문자,날짜와 같은 기본 데이터 타입들을 스칼라 타입이라 합니다. 
    중복 제거는 DISTINCT로 가능하며 통계형 쿼리도 가능합니다.
    ```java
    record BookSummary(String title, String summary) {}

    List<BookSummary> results = entityManager.createQuery("select title, left(book.text, 200) from Book",BookSummary.class)
                                     .getResultList();
    // 이 방법은 매개변수 명을 보고 바인딩이 됩니다. 
    // 하지만 공식문서는 아래와 같은 방식을 추천합니다.
    List<BookSummary> results = entityManager.createQuery("select new BookSummary(title, left(book.text, 200)) from Book",BookSummary.class)
                                .getResultList();
    ```  
`new`를 사용할 경우 2가지를 주의해야합니다.
+ 패키지 명을 포함한 전체 클래스 명을 입력한다.
+ 순서와 타입이 일치하는 생성자가 필요하다.
    불편한 부분은 QueryDSL에서 어느정도 해소해줍니다.  

## 페이징 API  
페이징용 SQL을 작성을 JPA가 대신해주고, 데이터베이스 마다 다른 페이징 방법을 
방언을 통해 알아서 처리해줍니다. 
+ `setFirstResult(int startPosition)`: 조회 시작 위치(0 부터 시작)
+ `setMaxResult(int maxResult`: 조회할 데이터 수(size)

+ 테스트 코드
```java
for (int i = 1; i < 100; i++) {
    Member member = new Member();
    member.setUsername("게스트 "+i);
    member.setAge(i);
    em.persist(member);
}
//초기화
em.flush();
em.clear();

//테스트
List<Member> resultList = em.createQuery("select m from Member as m order by m.id desc ", Member.class)
            .setFirstResult(0)
            .setMaxResults(10)
            .getResultList();

//검증
Assertions.assertEquals(resultList.size(),10);
Assertions.assertEquals(resultList.get(0).getAge(),99);
Assertions.assertEquals(resultList.get(9).getAge(),90);
```   
**페이징 SQL을 더 최적화 하고 싶다면 네이티브 SQL을 직접 사용해야합니다.**  
  
