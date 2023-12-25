# 동적쿼리 - BooleanBuilder
동적 쿼리를 해결하는 두가지 방법이 있습니다.
1. BooleanBuilder
2. Where 다중 파라미터 사용


### BooleanBuilder - 계단식 predicate 사용 방법
> 복잡한 부울 표현식을 구성하려면 com.querydsl.core.BooleanBuilder 클래스를 사용합니다.
> 이 클래스는 Predicate를 구현하며, 연쇄적으로 사용할 수 있습니다  
BooleanBuilder는 가변적이며, 처음에는 null을 나타내고 각 and 또는 or 호출 후에는 연산 결과를 나타냅니다.
+ BooleanBuilder 코드
    ```Java
    private BooleanBuilder searchPredicate(String name,Integer age) {
        BooleanBuilder booleanBuilder = new BooleanBuilder();
        if (name != null && !name.isBlank()) {
            booleanBuilder.and(QMember.member.username.eq(name));
        }
        if (age != null) {
            booleanBuilder.and(QMember.member.age.eq(age));
        }
        return booleanBuilder;
    }
    ```  
+ 테스트 코드
    ```Java
    @DisplayName("predicate operation")
    @Test
    void b1(){
        //given
        em.persist(new Member("둘리",15));
        em.persist(new Member("또치",20));
        em.flush();
        em.clear();
    
    
        //when
        QMember member = QMember.member;
        List<Member> findMember = queryFactory
                .selectFrom(member)
                .where(searchPredicate("둘리",null))
                .fetch();
        //then
        for (Member findMember1 : findMember) {
            System.out.println("findMember1 = " + findMember1);
        }
    }
    ```  
+ SQL
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
    ```  
```Java
private Predicate searchName(String name) {
    if (name == null || name.isBlank()) {
        return null;
    }
    return new BooleanBuilder()
            .and(QMember.member.username.eq(name));
}
private Predicate searchAge(Integer age) {
    if (age == null) {
        return null;
    }
    return new BooleanBuilder()
            .and(QMember.member.age.eq(age));
}
@DisplayName("BooleanBuilder 연속 사용")
@Test
void pr2(){
    QMember member = QMember.member;
    queryFactory
            .select(member)
            .from(member)
            .where(searchName(null),searchAge(55))
            .fetch();
}
```
> 조건마다 개별로 매서드를 만들어서 사용할 수 있으며
> Predicate 값이 null일경우 해당 조건은 무시됩니다.
> where(A,B) 로 AND 로 엮어서 사용할 수 있습니다.
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
    m1_0.age=?
``` 
#### 반환타입 BooleanBuilder
Predicate 반환 타입으로 할경우 체인을 사용하지 못하지만,
BooleanBuilder를 사용할 경우에는 체이닝이 가능합니다.
```Java
private BooleanBuilder searchAge(Integer age) {
    if (age == null) {
        return new BooleanBuilder();
    }
    return new BooleanBuilder(QMember.member.age.eq(age));
}
@DisplayName("BooleanBuilder 연속 사용")
@Test
void pr2(){
    QMember member = QMember.member;
    queryFactory
            .select(member)
            .from(member)
            .where(searchName(null).or(searchAge(15)).or(searchAge(15)))
            .fetch();
}
```
이렇게 메소드 체이닝이 가능한 이유는 `BooleanBuilder` 내부 로직을 보면 알 수 있습니다.
```Java
public final class BooleanBuilder implements Predicate, Cloneable  {
    @Nullable
    private Predicate predicate;
}
public BooleanBuilder and(@Nullable Predicate right) {
    if (right != null) {
        if (predicate == null) {
            predicate = right;
        } else {
            predicate = ExpressionUtils.and(predicate, right);
        }
    }
    return this;
}
```
BooleanBuilder는 내부에 predicate를 가지고 있습니다.
+ 기본 생성자는 predicate가 null 입니다
+ 매개변수로 predicate를 받으면 내부 필드로 저장합니다.
> AND 메소드를 보면
> 1. 매개변수로 넘어온 predicate null일 경우 기존 Builder 반환
> 2. 기본 Builder의 Predicate가 null일 경우 교체
> 3. Builder의 predicate와 매개변수 모두 null이 아닐경우 새 predicate 생성

**따라서 return BooleanBuilder를 반환하면 체이닝이 가능합니다.**

+ SQL로 확인하기
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
    m1_0.age=? 
    or m1_0.age=?
```
