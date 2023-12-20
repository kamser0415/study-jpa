# 집계
### 집합함수
```Java
@DisplayName("집계함수")
@Test
void ag1(){
    List<com.querydsl.core.Tuple> resultTuple = queryFactory
            .select(
                    member.age.avg(),
                    member.age.countDistinct(),
                    member.age.sum()
            ).from(member)
            .fetch();
            
    Assertions.assertThat(resultTuple.size()).isOne();
    com.querydsl.core.Tuple actual = resultTuple.get(0);
    Assertions.assertThat(actual).
            extracting(
                    tuple -> tuple.get(member.age.avg()),
                    tuple -> tuple.get(member.age.countDistinct()),
                    tuple -> tuple.get(member.age.sum())
            )
            .contains(25.0,4L,100);
}
```  
+ tuple은 Assertj의 Tuple이 아니라  
  `com.querydsl.core.Tuple`입니다.

### Group By 사용
> 팀의 이름과 각 팀의 평균 연령을 구해라.
  
```Java
@DisplayName("나이별로 몇명이 있는지 확인하기")
@Test
void g1(){
    //given
    em.persist(new Member("sample1",10));
    em.persist(new Member("sample2",20));
    em.persist(new Member("sample3",20));
    em.persist(new Member("sample4",30));
    em.flush();
    em.clear();

    //when
    List<com.querydsl.core.Tuple> result = queryFactory.select(
                    member.age, member.count())
            .from(member)
            .groupBy(member.age)
            .fetch();

    //then
    Assertions.assertThat(result).hasSize(4);
    Assertions.assertThat(result).extracting(
             tuple -> tuple.get(member.age),tuple -> tuple.get(member.count())
            )
            .contains(
                    tuple(10,2L),
                    tuple(20,3L),
                    tuple(30,2L),
                    tuple(40,1L)
            );
}
```
```SQL
select
    m1_0.age,
    count(m1_0.member_id) 
from
    member m1_0 
group by
    m1_0.age
```
+ 테스트코드로 작성시 Tuple은 Map이라고 생각해야합니다.
  일반적인 사용자 정의 클래스는 `extracting("age","count")`로 추출할 수 있지만,  
  Map일경우 별도의 값을 꺼내오는 방법을 설정해야합니다.
  
#### Having 도 추가로 테스트해보기
**테스트코드**
```Java
@DisplayName("having도 같이 테스트 해봅니다.")
@Test
void t2(){
    //given
    em.persist(new Member("sample1",10));
    em.persist(new Member("sample2",20));
    em.persist(new Member("sample3",20));
    em.persist(new Member("sample4",30));
    em.flush();
    em.clear();

    //when
    List<com.querydsl.core.Tuple> result = queryFactory.select(
                    member.age, member.count())
            .from(member)
            .groupBy(member.age)
            .having(member.age.goe(20))
            .fetch();
    //then
    Assertions.assertThat(result).hasSize(3);
    Assertions.assertThat(result).extracting(
                    tuple -> tuple.get(member.age),tuple -> tuple.get(member.count())
            )
            .contains(
                    tuple(20,3L),
                    tuple(30,2L),
                    tuple(40,1L)
            );
}
```
```SQL
select
    m1_0.age,
    count(m1_0.member_id) 
from
    member m1_0 
group by
    m1_0.age 
having
    m1_0.age>=?
```
