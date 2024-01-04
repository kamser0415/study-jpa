# 검색 조건 쿼리

### 기본 검색 쿼리
#### 테스트코드
```Java
@DisplayName("기본 검색")
@Test
void q1(){
    //given
    Member findMember = queryFactory
            .select(member)
            .from(member)
            .where(member.age.eq(20),
                    member.username.startsWith("member"))
            .fetchOne();
    //when
    Assertions.assertThat(findMember).isNotNull();
    Assertions.assertThat(findMember.getUsername()).isEqualTo("member2");
}
```
+ AND 사용
  ```Java
  member.age.eq(20), member.username.startsWith("member")
  member.age.eq(20).and(member.username.startsWith("member"))
  ```
+ OR 사용
  ```Java
  member.username.startsWith("member")).or(member.id.eq(2L) 
  ```
+ A and ( B or C) 사용
  ```Java
    .where(member.age.eq(20),
      ((member.username.startsWith("member")).or(member.id.eq(2L)))) 
  ```
  ```SQL
  select
      m1_0.member_id,
      m1_0.age,
      m1_0.team_id,
      m1_0.username 
  from
      member m1_0 
  where
      m1_0.age=? 
      and (
          m1_0.username like ? escape '!' 
          or m1_0.member_id=?
      )
  ```  
+ selectFrom()은 the query source(소스)와 projection(프로젝션)을 같이 정의합니다.
  ```Java
  @DisplayName("selectFrom은 쿼리 소스와 프로젝션을 정의한다")
  @Test
  void sp1(){
      Member findMember = queryFactory.selectFrom(member)
              .where(member.id.eq(1L))
              .fetchOne();

      Assertions.assertThat(findMember).isNotNull();
      Assertions.assertThat(findMember.getId()).isEqualTo(1L);
      Assertions.assertThat(findMember).extracting("username","age")
              .containsExactly("member1",10);
  }
  ```  
### JPQL이 제공하는 문법을 제공합니다.
```Java
member.username.eq("member1") // username = 'member1'
member.username.ne("member1") //username != 'member1'
member.username.eq("member1").not() // username != 'member1'
member.username.isNotNull() //이름이 is not null
member.age.in(10, 20) // age in (10,20)
member.age.notIn(10, 20) // age not in (10, 20)
member.age.between(10,30) //between 10, 30
member.age.goe(30) // age >= 30
member.age.gt(30) // age > 30
member.age.loe(30) // age <= 30
member.age.lt(30) // age < 30
member.username.like("member%") //like 검색
member.username.contains("member") // like ‘%member%’ 검색
member.username.startsWith("member") //like ‘member%’ 검색
```

### 결과조회
#### fetchOne()
+ 단건조회
+ 결과가 없으면 : `null`
+ 결과가 둘 이상이면 : `com.querydsl.core.NonUniqueResultException`  
```Java
@DisplayName("단건 조회하지만 결과가 여러개라면")
@Test
void s1() {
    //given
    Assertions.assertThatThrownBy(() ->
            queryFactory
                    .selectFrom(member)
                    .where(member.id.in(1, 2, 3, 4))
                    .fetchOne())
            .isInstanceOf(NonUniqueResultException.class);
}
```
#### fetchFirst()
+ `limit(1).fetchOne()`
+ `SELECT EXISTS(SELECT 1 FROM tableA WHERE ...)`과 유사하게 사용가능합니다.
+ 데이터 유/무를 확인하기에 최적화된 문법입니다.
```Java
@DisplayName("가장 먼저 조회되는 엔티티 반환")
@Test
void s2() {
    //given
    Member findMember = queryFactory.selectFrom(member)
            .where(member.username.startsWith("member"))
            .fetchFirst();
//          .limit(1).fetchOne(); 동일
    Assertions.assertThat(findMember).isNotNull();
    Assertions.assertThat(findMember).extracting("username","age")
            .containsExactly("member1",10);
}
```
```SQL
select
    m1_0.member_id,
    m1_0.age,
    m1_0.team_id,
    m1_0.username 
from
    member m1_0 
where
    m1_0.username like ? escape '!' 
limit
    NULL
```

#### fetchResult,fetchCount는 사용금지
javaDoc에는 `fetchCount()` 대신에 `fetch().size()` 사용하라고 작성되어있지만 사용하면 안됩니다.  
> 이유는 전체 데이터를 불러오고 나서 size()로 구하는 방식은 영속성 컨텍스트에 데이터를 전부 받아온 뒤에 개수를 따로 세는 것이기 때문에 불필요하게 메모리를 잡아먹기 때문입니다.
> 만약 서버 메모리가 부족할 경우 서버가 죽을수 있습니다.  
> `다대일 페이징을 하듯이 메모리에 쿼리결과의 갯수를 계산합니다.`

```Java
@DisplayName("가장 먼저 조회되는 엔티티 반환")
@Test
void s3() {
    //given
    long count = queryFactory
            .selectFrom(member)
            .leftJoin(member.team)
            .where(member.username.startsWith("member"))
            .fetch().size();
    Assertions.assertThat(count).isEqualTo(4);
}
```
```SQL
select
    m1_0.member_id,
    m1_0.age,
    m1_0.team_id,
    m1_0.username 
from
    member m1_0 
where
    m1_0.username like ? escape '!'
```  
