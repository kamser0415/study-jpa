# 기본키 매핑  
엔티티 코드
```java
@Entity
public class Member {

    @Id
    private Long id;

}
```  

### 기본 키 매핑 어노테이션  
+ `@Id`  
+ `@GeneratedValue`  
```java
@Id @GeneratedValue(strategy = GenerationType.AUTO)
private Long id;
```  

### 기본 키 매핑 방법  
+ 직접 할당: `@Id`만 사용  
  여러가지 코드성 데이터를 조합을 해서 직접 `PK`를 생성후 할당할 때 사용합니다.    
+ 자동 생성(`@GeneratedValue`) 데이터 베이스가 자동으로 할당  
  
#### @GeneratedValue(strategy = GenerationType.AUTO)  
+ DB 방언에 맞춰서 생성됩니다.  
+ 예를 들면 오라클이면 시퀀스, MySQL이면 auto_increment  
+ AUTO는 `SEQUENCE`,`IDENTITY`,`TABLE` 셋 중에서 선택이 됩니다.  

### IDENTITY 전략 - 특징  
+ 기본 키 생성을 데이터베이스에 위임
+ 주로 MySQL, PostgreSQL, SQL Server, DB2에서 사용 (예: MySQL의 auto_increment)
+ JPA는 보통 트랜잭션 커밋 시점에 INSERT SQL 실행
+ AUTO_ INCREMENT는 데이터베이스에 INSERT SQL을 실행한 이후에 ID 값을 알 수 있음
+ IDENTITY 전략은 em.persist() 시점에 **즉시 INSERT SQL 실행**하고 DB에서 식별자를 조회  
```SQL
-- ORACLE
id number(19,0) generated as identity,
-- MYSQL
id bigint not null auto_increment,
```  
  
### SEQUENCE - 전략 - 특징  
+ 데이터베이스 시퀀스는 유일한 값을 순서대로 생성하는 특별한 데이터베이스 오브젝트  
  + (예: 오라클 시퀀스)
+ 오라클, PostgreSQL, DB2, H2 데이터베이스에서 사용  
```sql
create sequence hibernate_sequence start with 1 increment by 1
call next value for hibernate_sequence
```   
sequence Object를 통해서 next value를 가져온다음 데이터를 삽입할 때 사용한다.  

**그러면 엔티티 `@Id` 필드의 타입은 어떤걸 사용해야할까?**  
> [JPA 공식문서](https://docs.oracle.com/javaee/7/tutorial/persistence-intro001.htm#BNBQA)
```java
Java primitive types
java.lang.String
Other serializable types, including:
Wrappers of Java primitive types
java.math.BigInteger
java.math.BigDecimal
java.util.Date
java.util.Calendar
java.sql.Date
Enumerated types
Other entities and/or collections of entities
Embeddable classes
```
JPA에서는 기본 형 타입도 허용을 합니다.  

> [Hibernate 6.3.1 final](https://docs.jboss.org/hibernate/orm/current/userguide/html_single/Hibernate_User_Guide.html#entity-pojo-identifier)
```text
We recommend that you declare consistently-named identifier attributes on persistent classes 
and that you use a wrapper (i.e., non-primitive) type (e.g. Long or Integer).
우리는 영속 클래스에서 일관된 이름의 식별자 속성을 선언하고, 
래퍼(즉, 원시 타입이 아닌) 타입(예: Long 또는 Integer)을 사용하는 것을 권장합니다
```  
하이버네이트에서는 래퍼 타입을 사용하는 것을 권장합니다.  
+ id를 primitive type으로 하면 0과 null을 구분할 수 없습니다.  
    primitive type의 기본 값이 0이기 때문입니다.  
+ null이 허용되는 wrapper type을 사용합니다.

그러면 식별자를 매핑하는 `@Id`를 제외한 나머지부분은 어떻게 사용하는게 좋을까?  
  
> [스택오버플로우](https://stackoverflow.com/questions/2331630/entity-members-should-they-be-primitive-data-types-or-java-data-types)  
  
+ 필수 값일 경우 `기본형 타입`을 사용한다. 
  + 기본형 타입을 사용할 경우 DDL 자동 생성 옵션 사용할 경우 `NOT NULL` 제약 조건이 추가된다. 
+ 필수 값이 아닐 경우 `래퍼형 타입`을 사용한다.

다시 돌아와서 `@Id`의 타입이 래퍼형을 권장한다는데  
`Integer`와 `Long`중에서 어떤 것을 사용하는게 좋을까?  
  
```text
MySQL에서는 Primary Key의 크기가 커지면 
InnoDB에서는 보조 인덱스 마다 Primary Key를 가지고 있기 때문에 곱연산이 된다.  
한번에 읽어올 수 있는 인덱스 페이지의 크기가 정해져 있기 때문인데  
단건으로 조회하는건 의미가 없을 수 있지만 ,  
range로 읽어오는 환경이라면 최소한의 크기로 정하는게 좋다고 생각은 든다.  
```
만약 21억 이상의 데이터가 들어가는 경우라면 미리 큰 타입으로 설정하는 것이 좋다.  
  
#### 사용자 정의 시퀀스  
```java
@Entity
@SequenceGenerator(
    name = "MEMBER_SEQ_GENERATED",
    sequenceName = "MEMBER_SEQ", // 매핑할 데이터베이스 시퀀스 이름
    initialValue = 1,allocationSize = 1
)
public class Member {

    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE,
            generator = "MEMBER_SEQ_GENERATED"
    )
    private Long id;
    
}

Hibernate: create sequence MEMBER_SEQ start with 1 increment by 1
```  
+ 직접 생성한 시퀀스를 사용할 수도 있습니다. 지금은 AutoDDL에서 자동으로 만들었습니다.  
만약 `ddl=none`이고, 시퀀스가 존재하지 않는다면 오류가 발생합니다.  
```java
Hibernate: 
    call next value for MEMBER_SEQ
11월 13, 2023 5:58:20 오후 org.hibernate.engine.jdbc.spi.SqlExceptionHelper logExceptions
WARN: SQL Error: 90036, SQLState: 90036
```

#### Sequence 전략으로 최적화하기  
> JPA가 시퀀스로 버퍼링 하는 방법  
  
```java
tx.begin();

MemberTable member = new MemberTable();
member.setName("둘리");
// 1) member.getId = null 
em.persist(member);
// 1) 해당 엔티티의 @GeneratedValue 전략을 확인
// 2) sequence 전략일 경우 DB에서 시퀀스의 nextVal를 읽어옴
// 3) member.setId(sequence.nextVal);
// 3) 영속성 컨택스트 1차 캐시에 저장
// fin) member.getId = 1;
tx.commit();
// 1) insert SQL excute;
```  
정리하면,  
엔티티의 `@GeneratedValue` 전략이 `sequence`일 경우 DB에서 시퀀스의 nextVal를 엔티티의 id에 넣어줌  
그리고 id값으로 영속성 컨택스트 1차 캐시에 저장을 합니다.  

`em.persist()`가 호출될 때마다 DB내 시퀀스 객체에 I/O가 발생합니다.  
시퀀스를 만들때 설정 값으로 시작할 숫자, 호출마다 건너뛸 숫자크기를 지정합니다.  
`initialValue = 1,allocationSize = 50`라고 설정하고   
현재 MEMBER_SEQ_A 값은 1입니다.  

#### call next value for Sequence 2번 호출하는 이유  
조건
1. call next value for MEMBER_SEQ 가 initial value인 1이 되어야한다.
2. em.persist()가 두 번이상 호출이 되어야한다.  
  
내부 로직을 정리하면 다음과 같습니다.  
hiValue,value 두 개의 값을 가지고 있는데 
+ hiValue는 가지고 있는 시퀀시의 값입니다.  
+ value는 현재 em.persist()를 호출할때마다 증가하는 값을 저장합니다.
1. getNextValue값을 hiValue에 저장합니다.
2. hiValue와 value를 비교합니다.
   1. value가 hiValue보다 클 경우 한번 더 getNextValue를 실행합니다.  
     
이런 로직이다 보니
처음 시퀀스를 호출했는데 1이 나오면 hiValue는 1이 됩니다.  
em.persist()가 두번 호출되면 value는 2가 될테고 hiValue를 비교해보니  
value(2) > hiValue(1) 이다보니 시퀀스를 한번 더 호출해서 캐시할 시퀀스를 저장합니다.  

결국 JPA도 DB에 위임하고 돌아온 값으로 캐싱을 해야하기때문이죠.

이렇게 allocation 필요한 만큼 설정을 하면  
여러 웹서버가 있어도 동시성 이슈없이 다양한 문제가 해결됩니다.  
```sql
create sequence MEMBER_SEQ_AP start with 1 increment by 50
```    


### Table 전략
+ 키 생성 전용 테이블을 하나 만들어서 데이터베이스 시퀀스를 흉내내는 전략  
+ 장점: 모든 데이터베이스에 적용 가능  
+ 단점: 성능 (테이블 사용으로 인한 `Lock` 발생도 생김)  

테이블을 생성하고 거기서 Generated 값을 꺼내오는 방식  
```java
@Entity
@TableGenerator(
        name = "MEMBER_SEQ_GENERATOR",
        table = "MY_SEQUENCES",
        pkColumnValue = "MEMBER_SEQ",allocationSize = 1)
public class MemberTable {
    @Id
    @GeneratedValue(strategy = GenerationType.TABLE,
            generator = "MEMBER_SEQ_GENERATOR")
    private Long id;
}
```
```sql
create table MY_SEQUENCES (
    sequence_name varchar(255) not null,
    next_val bigint,
    primary key (sequence_name)
)
insert into MY_SEQUENCES(sequence_name, next_val) values ('MEMBER_SEQ',0)
select
    tbl.next_val
from
    MY_SEQUENCES tbl
where
    tbl.sequence_name=? for update
```
+ MY_SEQUENCES  
   
    | SEQUENCE_NAME | NEXT_VAL |
    |---------------|----------|
    | MEMBER_SEQ    | 1        |

> DB에서 관례로 사용하는 것들이 있고, 최적화가 된걸 사용하는게 낫다.  

| 속성                     | 설명                               | 기본값                 |
|------------------------|----------------------------------|---------------------|
| name                   | 식별자 생성기 이름                       | 필수                  |
| table                  | 키생성 테이블명                         | hibernate_sequences |
| pkColumnName           | 시퀀스 컬럼명                          | sequence_name       |
| valueColumnName        | 시퀀스 값 컬럼명                        | next_val            |
| pkColumnValue          | 키로 사용할 값 이름                      | 엔티티 이름              |
| **initialValue**       | 초기 값, 마지막으로 생성된 값이 기준이다.         | 0                   |
| **allocationSize**     | 시퀀스 한 번 호출에 증가하는 수 (성능 최적화에 사용됨) | 50                  |
| catalog, schema        | 데이터베이스 catalog, schema 이름        |                     |
| uniqueConstraints(DDL) | 유니크 제약 조건을 지정할 수 있다.             |                     |
  
### 권장하는 식별자 전략  
> DB의 PK 제약조건에 대해서 먼저 생각해야합니다.  
> 1. NOT NULL
> 2. UNIQUE  
> 
> 추가로 변하지 않는 값이여야합니다. 
  
애플리케이션 서비스가 5년,10년 길어져서 20년넘게 살아남을 수 있는데  
그때까지 식별자의 값은 변하면 안된다는 의미입니다.  
이 조건을 만족하는 자연키(`natural key`)라고 하는데 찾기가 어렵습니다.  
  
대신에, 대리키, 대체키가 `Generate Value` 처럼  
랜덤값이나 비즈니스와 상관없는 키를 사용하는 걸 추천합니다.  
  
실제 객체의 값을 사용하다보면, 어떻게 변하거나, 정책이 변할수 있기 때문이다.  
1. DB마다 최적화된 방식의 Generator를 사용
2. UUID,RANDOM 값을 조합한 회사의 룰을 사용  




