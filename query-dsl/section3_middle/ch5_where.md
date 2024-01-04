
## 동적 쿼리 - Where 다중 파라미터 사용
`BooleanExpression`와 같이 `Expression`을 활용해서 사용하는 방법입니다.  
`Expression`은 `SQL`에서 사용하는 조건을 코드로 표현한 객체입니다.  
`BooleanExpression`은 조건을 나태내는 부분에 들어가는 참/거짓 표현식 입니다.  

```Java
@DisplayName("where 다중 파라미터")
@Test
void expresstion(){
    QMember member = QMember.member;
    queryFactory.select(member)
            .from(member)
            .where(usernameEq("둘리"),ageEq(15))
            .fetch();
}
private BooleanExpression usernameEq(String usernameCond) {
    return usernameCond != null ? QMember.member.username.eq(usernameCond) : null;
}
private BooleanExpression ageEq(Integer ageCond) {
    return ageCond != null ? QMember.member.age.eq(ageCond) : null;
}
```
**SQL**
```SQL
select
    m1_0.member_id,
    m1_0.city,
    m1_0.zipcode,
    m1_0.age,
    m1_0.team_id,
    m1_0.username 
from
    member m1_0 
where
    m1_0.username=? 
    and m1_0.age=?
```  
> BooleanBilder를 사용할 때보다 더 명확하고, 체이닝도 불필요한 코드 없이 사용이 가능합니다.  
> 만약 여러 조건을 한번에 사용한다면 다음과 같이 작성할 수 있습니다.
> 1. where 조건에서 ( A -> null  , B -> null , C ) 일 경우   
   결과가 NULL인 표현식은 무시가 됩니다.( C )만 남게 됩니다.  
   다만 모든 결과가 NULL 일경우에는 필터링 없이 데이터가 반환되므로 주의해야합니다.
> 2. 메서드를 다른 쿼리에서도 사용 가능합니다.
> 3. 쿼리 자체의 가독성이 높아집니다.
#### 주의사항
메소드 체이닝으로 `A.or(B)`형식으로 사용할 수 있지만 A에서 `NULL`이 반환된다면 예외가 발생합니다.
```Java
private BooleanExpression allCondEx(String username,Integer age) {
    //if(username == null) throw Exception
    return usernameEq(username).or(ageEq(age));
}
```  
#### 응용하기
1. BooleanBuilder를 그대로 사용하기
    ```Java
    rivate BooleanBuilder searchAge(Integer age) {
        if (age == null) {
            return new BooleanBuilder();
        }
        return new BooleanBuilder(QMember.member.age.eq(age));
    }
    ```
    + 장점: 검증에 대한 코드를 명시적으로 작성할 수 있다.
    + 단점: 모든 null체크에 대한 if문이 계속 반복된다.

2. 람다를 이용하여 공통화를 시킨다.
    ```Java
      private BooleanBuilder searchNameWithLambda(String name) {
          return nullOrExpression(()->QMember.member.username.eq(name));
      }
      private  BooleanBuilder nullOrExpression(Supplier<BooleanExpression> exp) {
          BooleanExpression booleanExpression = exp.get();
          if (booleanExpression == null) {
              return new BooleanBuilder();
          }
          return new BooleanBuilder(booleanExpression);
      } 
    ```  
   이 방식으로 하면 반복되는 null체크를 공통 처리할 수있지만 예외가 발생한다.
   `eq(null)` 자체가 허용되지 않기 때문에 `try-catch`로 감싸야합니다.
3. 예외 처리 로직을 추가한 람다 방식
    ```Java
    private BooleanBuilder nullOrExpression(Supplier<BooleanExpression> exp) {
        try {
            return new BooleanBuilder(exp.get());
        } catch (IllegalArgumentException e) {
            return new BooleanBuilder();
        }
    }
    ```  
   장점: `null` 체크에 대한 로직이 사라졌습니다.  
   단점: `eq`처럼 동등 비교는 `null` 체크를 하지만 `IN`은 처리를 하지 못합니다.  
   `NOT IN`처럼 `NULL`으로 발생하는 문제나 `""`같이 공백 처리는 별로로 해야합니다.
