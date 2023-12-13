# 엔티티 매핑  
<!-- TOC -->
* [엔티티 매핑](#엔티티-매핑-)
    * [목차](#목차)
  * [객체와 테이블 매핑](#객체와-테이블-매핑-)
    * [엔티티 매핑 소개](#엔티티-매핑-소개-)
    * [@Entity](#entity-)
      * [@Entity 속성 정리](#entity-속성-정리-)
    * [데이터베이스 스키마 자동 생성](#데이터베이스-스키마-자동-생성-)
      * [스키마 자동 생성 - 주의](#스키마-자동-생성---주의-)
      * [DDL 생성 기능](#ddl-생성-기능)
  * [필드와 컬럼 매핑](#필드와-컬럼-매핑-)
      * [insertable,updateable](#insertableupdateable-)
      * [nullable(중요)](#nullable중요-)
      * [unique(사용 안함)](#unique사용-안함-)
      * [length](#length-)
      * [columnDefinition](#columndefinition-)
      * [enum 타입사용시 옵션 주의사항](#enum-타입사용시-옵션-주의사항-)
      * [Temporal](#temporal-)
      * [Lob](#lob-)
<!-- TOC -->

JPA에서 제일 중요하게 봐야하는 두가지는
1. **영속성 컨택스트**  
   어떤 JPA의 내부 동작 방식과 JPA가 내부적으로 어떤 메커니즘으로 동작하는지 알아야한다.  
2. **객체와 관계형 데이터베이스의 매핑**  
    실제 설계적인 측면에서 오브젝트와 RDBMS를 매핑해서 사용하는지 알아야합니다.  

### 목차
+ 객체와 테이블 매핑
+ 데이터베이스 스키마 자동 생성
+ 필드와 컬럼 매핑
+ 기본 키 매핑
+ 실정 예제 -1. 요구사항 분석과 기본 매핑  
  
## 객체와 테이블 매핑  
### 엔티티 매핑 소개  
+ 객체와 테이블 매핑: `@Entity`,`@Table`  
+ 필드와 칼럼 매핑: `@Column`  
+ 기본 키 매핑: `@Id`  
+ 연관관계 매핑: `@ManyToOne`,`@JoinColumn`  
    + 1:1,1:N,N:M 관계를 어떻게 객체와 테이블을 매핑할 때 사용  

### @Entity  
> 클래스와 테이블을 매핑할 때 사용하는 애노테이션  
  
+ `@Entity`가 붙은 클래스는 이제 JPA가 관리하는 엔티티입니다.  
    해당 애노테이션이 붙지 않으면 JPA와 전혀 관련이 없는 사용자 정의 클래스라고 보면 됩니다.  
    그래서 JPA를 사용해서 테이블과 매핑할 클래스는 `@Entity`가 필수 입니다.
+ **주의**  
  + 기본 생성자 필수(파라미터가 없는 public 또는 protected 생성자)  
    JPA 라이브러리를 구현할 때 리플렉션이나 다양한 기술을 사용할 때 기본생성자가 필요하다.
  + final 클래스,enum,interface,inner 클래스는 사용할 수 없다.  
    + 해당 클래스들은 `@Entity`를 붙여서 매핑이 되지 않습니다.  
  + 저장할 필드에 final 사용을 하지 않는다.  
    
#### @Entity 속성 정리  
+ 속성 : `name`  
  + 기본 값 : 클래스 이름 그대로 사용 ex) Member.class => Member  
  + 다른 패키지 내에 동일한 엔티티 이름이 있을 경우에 사용한다.  
  + JPA가 내부적으로 구분하는 이름과 같다고 생각하면 된다.  
```java
@Table(name="MBR")
```  
그러면 데이터베이스의 테이블 `MBR`과 해당 클래스는 매핑됩니다.  

### 데이터베이스 스키마 자동 생성  
> 운영 단계에서 사용하지 않고,  
> 애플리케이션 로딩 시점에 DB테이블을 생성하는 기능도 지원합니다.  
> 개발 단계나 로컬pc에서 개발할 때 이럴 때 도움이 됩니다.  
  
+ DDL을 애플리케이션 실행 시점에 자동 생성  
    장점은 개발을 할때 테이블을 만들고 객체를 만들어서 개발하지만,  
    JPA는 객체 생성하고 매핑을 하면 애플리케이션을 로딩할 때 테이블을 만들어줍니다.  
    **데이터베이스 방언을 활용해 데이터베이스에 맞는 DDL을 생성 및 실행합니다.**  
+ **_생성된 DDL은 개발 환경에서만 사용_**  
+ 생성된 DDL은 운영 서버에서는 사용하지 않거나, 적절히 다듬은 후 사용합니다.  
  
```xml
<persistence-unit name="hello">
    <properties>
        <!-- 기타 설정 생략 -->
        <!-- application 실행 시 ddl 전략 -->
        <property name="hibernate.hbm2ddl.auto" value="none"/>
    </properties>
</persistence-unit>
```  
`hibernate.hbm2ddl.auto` 옵션에 따라 애플리케이션 로딩 시점에 `@Entity`가 매핑된 엔티티의 정보를 보고 테이블을 만듭니다.  
  
데이터베이스에 가서 `ALTER`로 칼럼을 추가하고 오브젝트에 필드 추가하는 작업이 필요없어진다.  
그 안에서 DDL 옵션설정만 하면 DB에 직접 DDL을 먼저 할 필요가 없다.  

+ 주의  
`UPDATE` 설정은 기존 테이블에서 변경된 부분만 DDL을 생성해주지만,  
필드를 제거하는 건 DDL을 생성하지 않는다. 추가하는 것만 됩니다.  

+ 현재 설정이 `H2Dialect`
    ```sql
    create table Member (
       id bigint not null,
        name varchar(255),
        primary key (id)
    )
    ```  
+ 방언 설정을 `Oracle`  
    ```sql
    create table Member (
       id number(19,0) not null,
        name varchar2(255 char),
        primary key (id)
    )
    ```  
적절한 데이터베이스의 문법으로 변형 후 적용됩니다.  

#### 스키마 자동 생성 - 주의  
+ **_운영 장비에는 절대 `create`,`create-drop`,`update`를 사용하면 안된다._**  
+ 개발 초기 단계 (로컬서버)에는 `create`,`update`를 사용  
+ 테스트 서버(여러명 사용)에는 `update`또는 `validate`사용  
+ 스테이징과 운영 서버에는 `validate` 또는 `none` 사용  
  
테스트 서버부터 `create`나 `update`도 사용하지 않는 것을 권장합니다.  
DDL은 직접 스크립트를 작성해서 DB에 적용할 때 문제가 없으면  
DBA나 검수를 받고 적용하는 것을 권장합니다.  
  
웹 애플리케이션에서 사용하는 DB 계정을 분리해서 사용하는 것을 권장합니다.  
  
#### DDL 생성 기능
```java
@Entity
public class Member {
    @Id
    private Long id;
    @Column(unique = true, length = 10)
    private String name;
}
```  
```sql
create table Member (
    id bigint not null,
    name varchar(10),
    primary key (id)
)
alter table Member
    add constraint UK_ektea7vp6e3low620iewuxhlq unique (name)
```  
  
`@Table`과 같은 애노테이션의 `name`이라는 옵션이 변경되면  
해당 테이블에 `DML`이 실행되기 때문에 런타임시에 영향을 주지만,

`@Column(unique,length)`은 DDL 생성 기능은 DDL을 자동 생성할 때만 사용되고  
JPA의 실행 로직에는 영향을 주지 않습니다.  

## 필드와 컬럼 매핑  
```java
@Id
private Long id;

@Column(name = "name")
private String username;

// Integer와 적절한 숫자 타입으로 만들어진다.
private Integer age;

@Enumerated(EnumType.STRING)
private RoleType roleType;

//생성일
@Temporal(TemporalType.TIMESTAMP)
private Date createDate;

//수정일
@Temporal(TemporalType.TIMESTAMP)
private Date lastModifiedDate;

@Lob
private String description;
```
자바의 `Date`타입은 날짜,시간을 같이 사용하지만 데이터베이스는 분리해서 사용하기 때문에  
`Date`과 같이 혼합된 클래스를 사용하려면 맵핑 정보를 애노테이션으로 줘야합니다.  

```sql
create table Member (
   id bigint not null,
    age integer,
    createDate timestamp,
    description clob,
    lastModifiedDate timestamp,
    roleType varchar(255),
    name varchar(255),
    primary key (id)
)
```
  
| 어노테이션       | 설명                        |
|-------------|---------------------------|
| @Column     | 칼럼 매핑                     |
| @Temporal   | 날짜 타입 매핑                  |
| @Enumerated | enum 타입 매핑                |
| @Lob        | BLOB,CLOB 매핑              |
| @Transient  | 특정 필드를 컬럼에 매핑하지 않음(매핑 무시) |  

`@Lob` 같은 경우는 문자 타입이면 `CLOB`으로 매핑이 됩니다.  
`@Transient`는 DB와 관련없이 메모리에서 사용하고 싶은 필드일 경우에 사용합니다.  
```java
@Transient
private String temp;
```  
DDL에 반영이 되지 않습니다.   

| 속성               | 기능                                                                                  | 기본값                                     |
|------------------|-------------------------------------------------------------------------------------|-----------------------------------------|
| name             | 필드와 매핑할 테이블의 컬럼 이름을 지정한다.                                                           | 객체의 필드 이름                               |
| insertable       | 엔티티 저장 시 이 필드도 같이 저장한다. false로 설정하면 이 필드는 데이터베이스에 저장하지 않는다. false 옵션은 읽기 전용일 때 사용한다 | true                                    |
| updateable       | 엔티티 수정 시 이 필드도 같이 수정한다. false로 설정하면 데이터베이스에 수정하지 않는다. false 옵션은 읽기 전용일 때 사용한다       | true                                    |
| table            | 하나의 엔티티를 두 개 이상의 테이블에 매핑할 때 사용한다.(@SecondaryTable 사용) 지정한 필드를 다른 테이블에 매핑할 수 있다.     | 현재 클래스가 매핑된 테이블                         |
| nullable (DDL)   | DDL 생성 시 null 값의 허용 여부를 설정한다. false로 설정하면 not null 제약조건이 붙는다.                       | true                                    |
| unique (DDL)     | @Table의 uniqueConstraints와 같으나 한 컬럼에 간단히 유니크 제약조건을 걸 때 사용한다.                        | false                                   |
| columnDefinition | 데이터베이스 컬럼 정보를 직접 줄 수 있다.                                                            | 자바 필드의 타입과, 데이터베이스 방언 설정 정보를 사용해 적절히 생성 |
| length (DDL)     | 문자 길이 제약조건, String 타입에만 사용한다.                                                       | 255                                     |
| precision, scale | BigDecimal 타입(혹은 BigInteger)에서 사용한다. precision은 소수점을 포함한 전체 자리수를, scale은 소수의 자리수다.  | precision = 0, <br/>scale = 0           |

#### insertable,updateable  
`@Column` 옵션 내에 `insertable=true`,`updateable=false`가 있는데,  
엔티티를 수정했을 때 데이터베이스에 `insert`,`update` 쿼리를 실행 여부를 결정할 수 있다.  
예를 들어 등록한 후에 수정은 절대하면 안된다고 한다면
```java
@Column(name = "name",updatable = false)
private String username;
```  
DB에서 강제로 수정을 하는건 막을수 없지만 JPA를 사용할 때에는 절대 반영되지 않습니다.  
  
#### nullable(중요)  
+ `default = true`;  
이 조건을 false로 할 경우에 not null 제약 조건이 됩니다.  
`nullable = false`라면 Hibernate 같은 경우에는 DDL과 null 체크도 합니다.  

#### unique(사용 안함)  
+ DDL은 만들어준다. 하지만 사용하지 않는다.  
1. 제약 객체 이름이 랜덤으로 지정이 된다.(식별이 어려움)  
`@Column`을 사용하면 제약 조건 이름을 명시하기 어렵다.  
2. `@Table(uniqueConstraints =)`을 사용하면 제약조건 이름을 명시할 수 있다.  

#### length  
+ 길이를 지정할 수 있다.  
  
#### columnDefinition  
+ 옵션에 명시한 DDL문이 그대로 들어가게 됩니다.  
+ 특정 DB의 종속적인 옵션도 다 넣을 수 있습니다.  
```java
@Column(name = "name",updatable = false,columnDefinition = "varchar(100) not null Default '둘리'")
private String username;
//SQL DDL  
name varchar(100) not null Default '둘리',
```  

#### enum 타입사용시 옵션 주의사항  
```java
@Enumerated(EnumType.STRING)
private RoleType roleType;

@Enumerated(EnumType.ORDINAL)
private RoleType roleTypeOrdinal;

public enum RoleType {
    USER,ADMIN
}
```  
```java
//테스트
public class EnumTest {

    public static void main(String[] args) {

        EntityManagerFactory emf = Persistence.createEntityManagerFactory("hello");
        EntityManager em = emf.createEntityManager();

        EntityTransaction tx = em.getTransaction();
        try {
            tx.begin();

            Member member = new Member(1L, "둘리");
            member.setRoleType(RoleType.ADMIN);
            member.setRoleTypeOrdinal(RoleType.ADMIN);

            em.persist(member);

            tx.commit();
        } catch (Exception e) {
            e.printStackTrace();
            tx.rollback();
        } finally {
            em.close();
        }
        emf.close();
    }
}
```
| ID | ROLETYPE | ROLETYPEORDINAL | NAME |
|----|----------|-----------------|------|
| 1  | ADMIN    | 1               | 둘리   |  

enum 타입은 저장된 순서로 ordinal 지정되기 때문에  
중간에 다른 타입이 들어오게 된다면 ordinal이 다른 값이 들어오게 됩니다.  
```java
public enum RoleType {
    USER,MASTER,ADMIN
}
```  
| ID | ROLETYPE | ROLETYPEORDINAL | NAME |
|----|----------|-----------------|------|
| 1  | ADMIN    | 2               | 둘리   |  

똑같은 정보이지만 enum 타입에 다른 중간 값이 들어왔을 때에는  
`@Enumerated(EnumType.ORDINAL)`은 값이 유동적으로 변경이 된다.  

필수로 `@Enumerated(EnumType.STRING)`을 사용해야합니다.  

#### Temporal  
날짜 타입(java.util.Date,java.util.Calendar)을 매핑할 때 사용
과거 버전을 써야하고, `Date` 타입을 사용해야한다면 사용해야합니다.  
참고: `java.time.LocalDate`,`java.time.LocalDateTime`을 사용할 때에는 생략가능  
 + (최신)하이버네이트 지원  

| 값                      | 설명                              | 예시                       |
|------------------------|---------------------------------|--------------------------|
| TemporalType.DATE      | 날짜, 데이터베이스 date 타입과 매핑          | (예: 2013–10–11)          |
| TemporalType.TIME      | 시간, 데이터베이스 time 타입과 매핑          | (예: 11:11:11)            |
| TemporalType.TIMESTAMP | 날짜와 시간, 데이터베이스 timestamp 타입과 매핑 | (예: 2013–10–11 11:11:11) |  

최신버전 하이버네이트를 사용한다면 아래와 같이 사용하면 됩니다.  
```java
private LocalDate localDate     => date
private LocalDate localDateTime => timestamp
```
  
#### Lob  
데이터베이스 BLOB, CLOB 타입과 매핑  
+ @Lob에는 지정할 수 있는 속성이 없다.  
+ 매핑하는 `필드 타입`이 문자면 CLOB 매핑, 나머지는 BLOB 매핑  
+ CLOB: String, char[], `java.sql.CLOB`
+ BLOB: byte[], `java.sql. BLOB`   
  
기본적인 테이블과 객체를 매핑하는건 어렵지 않습니다.  
이제 테이블과 테이블 관계를 객체로 표현해야하는 연관관계 매핑을 이해야합니다.  
  
