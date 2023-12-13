# 서브쿼리  
<!-- TOC -->
* [서브쿼리](#서브쿼리-)
  * [비상관 서브쿼리](#비상관-서브쿼리-)
  * [상관 서브쿼리](#상관-서브쿼리)
  * [서브쿼리 함수 지원](#서브쿼리-함수-지원)
    * [하이버네이트 6을 사용하지 못하는 경우](#하이버네이트-6을-사용하지-못하는-경우)
* [JPQL 타입 표현](#jpql-타입-표현-)
    * [PQL 타입 표현](#pql-타입-표현)
      * [함수 예제](#함수-예제)
  * [사용자 정의 함수](#사용자-정의-함수-)
  * [경로 표현식](#경로-표현식)
    * [경로 표현식의 용어정리](#경로-표현식의-용어정리)
    * [경로 표현식과 특징](#경로-표현식과-특징-)
    * [묵시적 조인 사용하기](#묵시적-조인-사용하기)
      * [1. 선수의 집 주소의 별명을 조회해보기](#1-선수의-집-주소의-별명을-조회해보기)
<!-- TOC -->

최신 하이버네이트 6 버전은 FROM 절도 서브쿼리를 지원합니다.  
공식 문서 예제:  
```jpaql
select stuff.id, stuff.total
from (
    select ord.id as id, sum(item.book.price * item.quantity) as total
    from Order as ord
        join Item as item
    group by ord
) as stuff
where total > 100.0
```  
FROM 절 서브쿼리는 비상관 서브쿼리여야 합니다.  

## 비상관 서브쿼리 
+ 나이가 평균보다 많은 회원
    ```jpaql
    select m from Member m
    where m.age > (select avg(m2.age) from Member m2) 
    ```
## 상관 서브쿼리
+ 한 건이라도 주문한 고객
    ```jpaql
    select m from Member m
    where (select count(o) from Order o where m = o.member) > 0
    ```  

## 서브쿼리 함수 지원
[하이버네이트 공식 함수 링크](https://docs.jboss.org/hibernate/orm/6.4/querylanguage/html_single/Hibernate_Query_Language.html#conditional-expressions)  
+  [NOT] EXISTS (subquery): 서브쿼리에 결과가 존재하면 참  
+  {ALL | ANY | SOME} (subquery)  
+  ALL 모두 만족하면 참  
+  ANY, SOME: 같은 의미, 조건을 하나라도 만족하면 참  
+ [NOT] IN (subquery): 서브쿼리의 결과 중 하나라도 같은 것이 있으면 참  
  
### 하이버네이트 6을 사용하지 못하는 경우
1. FROM 서브쿼리
    FROM 절에 서브쿼리를 넣는 이유는 데이터를 먼저 필터링 하려는 목적이 큽니다. 
    네이티브 SQL을 작성할 수 있지만 어플리케이션에서 조작하거나 
    쿼리를 두번 날리는 방법도 있습니다.  
  
제일 좋은 방법은 조인으로 풀어서 해결하는 방법을 추천합니다. 
그게 안되면 위 방법처럼 두번 쿼리를 날리거나 어플리케이션에서 조작후 해결합니다.  
  
만약 FROM절에서 데이터를 조인하고 줄인다음 외부 쿼리에서 화면에 보여주기 위해 
뷰 로직이 들어있는 경우에는 어플리케이션으로 빼서 해결할 수 있습니다.  
  
# JPQL 타입 표현  
[공식문서 타입 캐스팅](https://docs.jboss.org/hibernate/orm/6.4/querylanguage/html_single/Hibernate_Query_Language.html#functions-typecasts)  

+ ENUM: jpabook.MemberType.Admin (패키지명 포함)  
    하이버네이트 6 부터는 패키지명 포함하지 않아도 됩니다.  
+ 엔티티 타입: TYPE(m) = Member (상속 관계에서 사용)   
### PQL 타입 표현
+ 문자: ‘HELLO’, ‘She’’s’
+ 숫자: 10L(Long), 10D(Double), 10F(Float)
+ Boolean: TRUE, FALSE
 
#### 함수 예제
| 기능 (Function)                | 목적 (Purpose)             | 문법 (Syntax)                                                                                                | JPA 표준 / ANSI SQL 표준 |
|------------------------------|--------------------------|------------------------------------------------------------------------------------------------------------|----------------------|
| `upper()`                    | 소문자를 대문자로 변환한 문자열        | `upper(str)`                                                                                               | ✔ / ✔                |
| `lower()`                    | 대문자가 소문자로 변환된 문자열        | `lower(str)`                                                                                               | ✔ / ✔                |
| `length()`                   | 문자열의 길이                  | `length(str)`                                                                                              | ✔ / ✖                |
| `concat()`                   | 문자열 연결                   | `concat(x, y, z)`                                                                                          | ✔ / ✖                |
| `locate()`                   | 문자열 내의 문자열 위치            | `locate(patt, str), locate(patt, str, start)`                                                              | ✔ / ✖                |
| `position()`                 | 비슷하다locate()             | `position(patt in str)`                                                                                    | ✖ / ✔                |
| `substring()` (JPQL 스타일)     | 문자열의 하위 문자열              | `substring(str, start), substring(str, start, len)`                                                        | ✔ / ✖                |
| `substring()` (ANSI SQL 스타일) | 문자열의 하위 문자열              | `substring(str from start), substring(str from start for len)`                                             | ✖ / ✔                |
| `trim()`                     | 문자열에서 문자 자르기             | 아래를 참조하세요                                                                                                  | ✔ / ✔                |
| `overlay()`                  | 하위 문자열을 교체하려면            | `overlay(str placing rep from start), overlay(str placing rep from start for len)`                         | ✖ / ✔                |
| `pad()`                      | 공백이나 지정된 문자로 문자열을 채웁니다.  | `pad(str with len), pad(str with len leading), pad(str with len trailing), pad(str with len leading char)` | ✖ / ✖                |
| `left()`                     | 문자열의 가장 왼쪽 문자            | `left(str, len)`                                                                                           | ✖ / ✖                |
| `right()`                    | 문자열의 가장 오른쪽 문자           | `right(str, len)`                                                                                          | ✖ / ✖                |
| `replace()`                  | 문자열에서 패턴이 나타나는 모든 항목 바꾸기 | `replace(str, patt, rep)`                                                                                  | ✖ / ✖                |
| `repeat()`                   | 문자열을 자신과 여러 번 연결         | `replace(str, times)`                                                                                      | ✖ / ✖                |
| `collate()`                  | 데이터 정렬 선택                | `collate(p.name as collation)`                                                                             | ✖ / ✖                |

## 사용자 정의 함수  

자신이 사용하는 방언(Dialect)을 찾아가서 함수 등록방법을 보고 방언을 상속후 사용하면 됩니다.
생성자에 추가하시면 됩니다.
```java
registerFunction( "decrypt", new StandardSQLFunction( "decrypt", StandardBasicTypes.BINARY ) );
registerFunction( "degrees", new StandardSQLFunction( "degrees", StandardBasicTypes.DOUBLE ) );
registerFunction( "encrypt", new StandardSQLFunction( "encrypt", StandardBasicTypes.BINARY ) );
```
```java
public class MyDialect extends H2Dialect {
    public MyDialect() {
        registerFunction("group_concat",new StandardSQLFunction("group_concat", StandardBasicTypes.STRING));
    } 
}
```  
하이버네이트 6버전 공식문서에서도 크게 설명이 없습니다 [링크](https://docs.jboss.org/hibernate/orm/6.4/querylanguage/html_single/Hibernate_Query_Language.html#user-defined-functions)  
방언을 설정할 때 상속한 방언을 등록하면 됩니다.
```xml
<property name="hibernate.dialect" value="Dialect.MyDialect"/>
```  
```java
@DisplayName("group_concat 사용자 정의 함수 사용하기")
@Test
void t5(){
    Team t1 = new Team();
    t1.setName("T1");

    Member faker = new Member();
    faker.setUsername("이상혁");
    faker.resigned(t1);
    Member keria = new Member();
    keria.setUsername("캐리아");
    keria.resigned(t1);

    em.persist(t1);
    em.persist(faker);
    em.persist(keria);
    //초기화
    em.flush();
    em.clear();
    //JPA 사용 버전
    String query = em.createQuery("select function('group_concat',m.username) from Member as m", String.class).getSingleResult();
    //HQL(하이버네이트 JPQL) 사용시
    String query = em.createQuery("select group_concat(m.username) from Member as m", String.class).getSingleResult();
    System.out.println("query = " + query);;
    //이상혁,캐리아
}
```

## 경로 표현식
JPQL에서 사용하는 경로 표현식(`pathExpresstion`)과 경로 표현식을 사용하면 발생하는 묵시적 조인도 알아보겠습니다.  
  
```Java
select m.username // 1 상태 필드
from Member as m
    join m.team as t // 2 단일 값 연관 필드
    join m.orders as o // 3 컬렉션 값 연관 필드
where t.name = 'T1'
```   
### 경로 표현식의 용어정리
+ 상태 필드(state ﬁeld): 단순히 값을 저장하기 위한 필드 (ex: m.username)
+ 연관 필드(association ﬁeld): 연관관계를 위한 필드
+  단일 값 연관 필드: 대상이 엔티티(ex: m.team)  
`@ManyToOne`, `@OneToOne`
+  컬렉션 값 연관 필드: 대상이 컬렉션(ex: m.orders)  
`@OneToMany`, `@ManyToMany`,   
```Java
@Getter @Entity @Setter
public class Member {
    @Id @GeneratedValue(strategy = GenerationType.SEQUENCE)
    private Long id;         // 상태필드 (프로퍼티)
    private String username; // 상태필드 (프로퍼티)

    @OneToMany(mappedBy = "member") // 컬렉션 값 연관필드
    private List<MemberProduct> memberProducts = new ArrayList<>();
    
    @JoinColumn(name = "team_id")
    @ManyToOne(fetch = FetchType.LAZY)
    private Team team;      // 단일 값 연관필드
}
```
**_경로 표현식에 따라서 내부 동작 방식이 다릅니다._**  
**_세가지를 구분해서 사용할 줄 알아야합니다._**  
  
### 경로 표현식과 특징   
+ 상태 필드(state ﬁeld): 경로 탐색의 끝, 탐색할 수 없다.
+ 단일 값 연관 경로: 묵시적 내부 조인(inner join) 발생, 단일 값 연관 경로는 계속 탐색할 수 있다.  
    예) `select m.address.addressAlias.addressName from Member as m`
+ 컬렉션 값 연관 경로: 묵시적 내부 조인 발생, 더이상 탐색할 수 없습니다.
  + FROM 절에서 명시적 조인을 통해 별칭을 얻으면 별칭을 통해 탐색 가능  
`하이버네이트 6에서는 element(),key(),index()등으로 탐색 가능하다고 합니다.`  
  
### 묵시적 조인 사용하기
+ 엔티티 전체 코드입니다.
```java
@Entity
public class AddressAlias {
    @Id
    @GeneratedValue
    private Long id;
    private String addressName;
}
@Embeddable @Getter @Setter
public class Address {

    private String city;
    private String street;
    private String zipcode;

    @ManyToOne
    private AddressAlias addressAlias;
}  
Entity @Getter @Setter
public class Member {

    @Id @GeneratedValue
    private Long id;
    
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "team_id")
    private Team team;

    @Embedded
    private Address address;
}
```

#### 1. 선수의 집 주소의 별명을 조회해보기
+ 테스트 코드
```java
@DisplayName("경로 탐색은 어떤 쿼리가 실행될까")
@Test
void t1(){

    Team t1 = new Team();
    t1.setName("T1");
    em.persist(t1);

    AddressAlias addressAlias = new AddressAlias();
    addressAlias.setAddressName("우리집");
    em.persist(addressAlias);
    //별칭 저장
    Address address = new Address("서울시", "마포구", "노고산동");
    address.setAddressAlias(addressAlias);
    //선수 저장
    Member member = new Member();
    member.setUsername("이상혁");
    member.resigned(t1);

    member.setAddress(address);
    em.persist(member);
    //초기화
    em.flush();
    em.clear();
        
    String sql = "select m.address.addressAlias.addressName from Member as m";
    String memberAddress = em.createQuery(sql, String.class).getSingleResult();
    System.out.println("memberAddress = " + memberAddress);
}
```   
임베디드 타입 내에 엔티티 연관관계도 탐색해서 조회가 가능합니다. 
그러면 이 테스트를 실행하면 JPA는 내부 조인을 사용할까요 ?
```sql
select
    al.addressName as `name`
from Member m
cross join AddressAlias al
where m.addressAlias_id=al.id
```  
동작 방식은 크로스 조인이 동작합니다.  
그러면 만약 명시적 조인으로 변경했다면 어떻게 될까요?  

```java
//           "select m.address.addressAlias.addressName from Member as m"
String sql = "select al.addressName from Member as m join m.address.addressAlias as al";
String memberAddress = em.createQuery(sql, String.class).getSingleResult();
```
```sql
select
    al.addressName as `name`
from Member m 
inner join AddressAlias al on m.addressAlias_id=al.id
```  
쿼리의 성능의 여부를 떠나서 실행해보기 전까지는 묵시적 조인은 어떤 SQL이 실행될지 모른다는게 단점입니다.  
  
조인이 성능상 차지하는 부분이 크기때문에 묵시적 조인은 일어나는 상황을 한눈에 파악하기 어렵습니다. 
따라서 단순한 경우에는 상관이 없을 수 있지만, 성능이 중요하다면 분석하기 쉽도록 묵시적 조인보다는 
명시적인 조인을 활용해야합니다.  
