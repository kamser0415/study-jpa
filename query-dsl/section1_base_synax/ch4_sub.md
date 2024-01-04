# 서브쿼리
+ `com.querydsl.jpa.JPAExpressions `을 사용합니다.  
유틸성 클래스로 다양한 `select` 메서드 지원과 사용 범위가 서브쿼리에 맞춰져 있습니다.  
> 공식문서:  
서브쿼리를 생성하기 위해선 JPAExpressions의 정적 팩토리 메서드를 사용하고, from, where 등을 통해 쿼리 파라미터를 정의합니다.
### 서브쿼리의 eq 사용
#### 비상관 서브쿼리
```Java
@DisplayName("서브쿼리 eq 사용하기")
@Test
void sub1(){
    //given
    /*
     * 나이가 가장 많은 회원 조회
     */
    QMember sub = new QMember("sub");
    //when
    List<Member> findMember = queryFactory.selectFrom(member)
            .where(member.age.eq(
                    JPAExpressions
                            .select(sub.age.max())
                            .from(sub)))
            .fetch();

    //then
    Assertions.assertThat(findMember).hasSize(1);
    Assertions.assertThat(findMember).extracting("username","age")
            .contains(tuple("member4",40));
}
```  
`JPAExpressions`를 활용해서 서브쿼리를 만들수 있습니다.
```SQL
select
    m1_0.member_id,
    m1_0.age,
    m1_0.team_id,
    m1_0.username 
from
    member m1_0 
where
    m1_0.age=(
        select
            max(m2_0.age) 
        from
            member m2_0
    )
```  
### 상관서브쿼리
외부 테이블의 칼럼값을 서브쿼리에서 의존하는 방식  
`JPAExpressions`를 활용해서 서브쿼리를 만들수 있습니다  
+ 서브쿼리는 다양한 기능이 있는 `JPAExprssioins`를 활용합니다.
```
Java`JPAExpressions ㅜ
queryFactory.selectFrom(member)
                .where(member.age.eq(
                        JPAExpressions
                                .select(company.companyAge)
                                .from(company)
                                .where(member.age.eq(company.companyAge))
                )).fetch();
```
```SQL
select
    m1_0.member_id,
    m1_0.age,
    m1_0.team_id,
    m1_0.username 
from member m1_0 
where m1_0.age = (
                    select c1_0.company_age 
                    from company c1_0 
                    where m1_0.age=c1_0.company_age 
                )
```  
### 서브쿼리 여러 건 처리 IN 사용  
```Java
@DisplayName("팀 A에 소속된 회원 찾기")
@Test
void sub3(){
    //when
    List<Member> findMember = queryFactory.selectFrom(member)
            .where(member.team.id.in(
                    JPAExpressions.select(team.id)
                            .from(team)
                            .where(team.name.eq("teamA"))
            )).fetch();
    //then
    Assertions.assertThat(findMember).hasSize(2);
    Assertions.assertThat(findMember).extracting(
            "username","age"
    ).contains(
            tuple("member1",10),
            tuple("member2",20)
    );
}
```
+ SQL
```SQL
select
    m1_0.member_id,
    m1_0.age,
    m1_0.team_id,
    m1_0.username 
from
    member m1_0 
where
    m1_0.team_id in( select
                         t2_0.team_id 
                     from
                         team t2_0 
                     where
                         t2_0.name=?)
```  
### 스칼라 서브쿼리
```Java
@DisplayName("Member 조회시 스칼라값으로 TeamName가져오기")
@Test
void sub4() {
    //when
    List<Tuple> findMember = queryFactory.select(member,
            Expressions.as(select(team.name)
                            .from(team)
                            .where(team.id.eq(member.team.id)),"teamName")
            )
            .from(member)
            .fetch();

    Assertions.assertThat(findMember).hasSize(4);
    Assertions.assertThat(findMember).extracting(
            tuple -> tuple.get(member).getId(),
            tuple -> tuple.get(member).getUsername(),
            tuple -> tuple.get(member).getAge(),
            tuple -> tuple.get(Expressions.stringPath("teamName"))
    ).contains(
            tuple(1L, "member1", 10, "teamA"),
            tuple(2L, "member2", 20, "teamA"),
            tuple(3L, "member3", 30, "teamB"),
            tuple(4L, "member4", 40, "teamB")
    );
}
```
+ 스칼라 서브쿼리를 사용할 경우에는 `Expressions.as()`를 사용해서 별칭을 주면 됩니다.