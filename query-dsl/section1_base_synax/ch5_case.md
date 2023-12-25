# Case문
select,where,order by에서 사용가능합니다.  

### 단순한 조건
```Java
@DisplayName("case로 나이대별로 나누기")
@Test
void sub5() {
    //when
    List<String> findMember = queryFactory
            .select(Expressions.as(member.age
                    .when(10).then("열살")
                    .when(20).then("스무살")
                    .otherwise("기타"), "kor"))
            .from(member)
            .fetch();
    Assertions.assertThat(findMember).extracting(
        tuple -> tuple(Expressions.stringPath("kor"))
    ).contains(
            tuple("열살"),
            tuple("스무살"),
            tuple("기타"),
            tuple("기타")
    );
}
```
### 복잡한 조건
+ 공식문서 예제
```Java
QCustomer customer = QCustomer.customer;
Expression<String> cases = new CaseBuilder()
    .when(customer.annualSpending.gt(10000)).then("Premier")
    .when(customer.annualSpending.gt(5000)).then("Gold")
    .when(customer.annualSpending.gt(2000)).then("Silver")
    .otherwise("Bronze");
// The cases expression can now be used in a projection or condition
```
+ BETWEEN
```Java
List<String> result = queryFactory 
        .select(new CaseBuilder()
                .when(member.age.between(0, 20)).then("0~20살") 
                .when(member.age.between(21, 30)).then("21~30살") 
                .otherwise("기타"))
        .from(member) 
        .fetch();
```
+ 윈도우 함수처럼 사용해보기
> 윈도우 함수처럼 특정 범위를 지정해서 RANK를 임의로 만들수 있습니다.
```Java
@DisplayName("case로 윈도우 함수처럼 사용하기")
@Test
void sub6() {
    //when
    NumberExpression<Integer> rankPath = new CaseBuilder()
            .when(member.age.between(0,20)).then(1)
            .when(member.age.between(21,30)).then(2)
            .otherwise(3);
    List<Tuple> result = queryFactory
            .select(member.username, member.age, Expressions.as(rankPath,"grade"))
            .from(member)
            .orderBy(rankPath.desc())
            .fetch();
    Assertions.assertThat(result).extracting(
            tuple -> tuple.get(member.username),
            tuple -> tuple.get(member.age),
            tuple -> tuple.get(Expressions.stringPath("grade"))
    ).contains(
            tuple("member4",40,3),
            tuple("member3",30,2),
            tuple("member2",20,1),
            tuple("member1",10,1)
    );
}
```
```Java
select
    m1_0.username,
    m1_0.age,
    case 
        when (m1_0.age between ? and ?) 
            then cast(? as signed) 
        when (m1_0.age between ? and ?) 
            then cast(? as signed) 
        else 3 
    end 
from
    member m1_0 
order by
    case 
        when (m1_0.age between ? and ?) 
            then ? 
        when (m1_0.age between ? and ?) 
            then ? 
        else 3 
    end desc
```  
+ Expression는 인터페이스로 다양한 구현체로 활용할 수 있습니다.