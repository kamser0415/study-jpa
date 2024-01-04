# 결과 핸들링 하는 방법
+ 공식문서 : [Result handling](http://querydsl.com/static/querydsl/5.0.0/reference/html_single/#result_handling)
  결과를 DTO로 반환할 때 사용하는 방법은 공식문서에서 5가지를 제공합니다.

**사용할 DTO 생성**
```Java
@Data
public class MemberInfo {
    private String username;
    private String age;
    private String teamName;
}
```
## Tuple
+ 이전 예제에서 계속 사용했기 때문에 생략합니다.
## 프로퍼티 접근 (setter)
> 해당 방법은 디폴트 생성자와 프로퍼티 접근 방식으로 setter으로 바인딩합니다.
```Java
@DisplayName("프로퍼티 접근 방식")
@Test
void p2(){
    //given
    String username = "둘리";
    Integer userAge = 10;
    String teamName = "T1";

    Team teamEntity = new Team(teamName);
    Member memberEntity = new Member(username, userAge);
    memberEntity.setTeam(teamEntity);
    em.persist(teamEntity);
    em.persist(memberEntity);

    em.flush();
    em.clear();

    //when
    List<MemberInfo> findMember = qFactory.select(Projections.bean(MemberInfo.class, member.username, member.age, team.name))
            .from(member)
            .join(member.team, team)
            .fetch();

    //then
    Assertions.assertThat(findMember).hasSize(1);
    Assertions.assertThat(findMember).extracting("username", "age", "teamName")
            .contains(Tuple.tuple(username, userAge, teamName));
}
```  
**해당 코드는 실패합니다.**
```Java
Expecting ArrayList: [("둘리", 10, null)]
```  
```Java
//public void setTeamName(String teamName) -- 실패
public void setName(String teamName) {
    this.teamName = teamName;
}
```  
## Expresstions.as(), ExpressionUtils.as QEntity.as()
를 활용해서 해당 DTO setter 메서드의 프로퍼티 명과 일치하는 별칭으로 변경합니다.
```Java
Projections.bean(
        MemberInfo.class, member.username, member.age,
        Expressions.as(team.name,"teamName")
))
```
`Projection.bean`은 setter 메서드의 이름을 보고 바인딩을 합니다.

## 필드 직접 접근 (fields)
필드명을 보고 직접 접근하여 바인딩합니다  
예제:
```Java
List<UserDTO> dtos = query.select(Projections.fields(UserDTO.class, 
                                  user.firstName, 
                                  user.lastName))
                          .fetch();
```  
필드로 초기화 해볼 테스트 DTO입니다.
+ **기본 생성자 (필수)**
```Java
@NoArgsConstructor
@ToString
public class FieldsProjection {
    private String fieldsName;
}
```
+ 테스트 코드
```Java
@DisplayName("fields 이름이 같으면 초기화가 됩니다.")
@Test
void fieldsInit(){
    //given
    String memberName1 = "둘리";
    String memberName2 = "또치";

    em.persist(new Member(memberName1,15));
    em.persist(new Member(memberName2,20));
    em.flush();
    em.clear();
    QMember member = QMember.member;

    //when
    List<FieldsProjection> fieldsNames = queryFactory.select(
            Projections.fields(FieldsProjection.class,
                    member.username.as("fieldsName")))
            .from(member)
            .fetch();
    //then
    Assertions.assertThat(fieldsNames).hasSize(2);
    Assertions.assertThat(fieldsNames).extracting("fieldsName")
            .contains(memberName2, memberName1);
}
```
> 참고 :  
> 필드 주입에 실패를 해도 쿼리의 row 수와 List<DTO>의 size()는 동일합니다.  
**필드명이 다를 경우**
> 별칭을 설정하면 됩니다.

## 생성자 접근
생성자 매개변수 자료형과 일치할 경우 초기화가 됩니다.
+ 매개변수명은 전혀 상관 없습니다.
```Java
public NewConstructor(String constName) {
    this.constName = constName;
}
```
+ 테스트 코드
```Java
@DisplayName("생성자 초기화 방법")
@Test
void newC1(){
    //given
    em.persist(new Member("둘리",15));
    em.persist(new Member("또치",20));
    em.flush();
    em.clear();
    //when
    QMember member = QMember.member;
    List<NewConstructor> alal = queryFactory
            .select(Projections.constructor(NewConstructor.class, member.username.as("alalalal")))
            .from(member)
            .fetch();
    //then
    Assertions.assertThat(alal).hasSize(2);
    Assertions.assertThat(alal).extracting("constName")
            .contains("둘리", "또치");
}
```  
> 참고:  
> 생성자 초기화시 타입만 일치하면 초기화가 됩니다.
> 순서만 일치한다면 별칭을 주지 않아도 된다는 간편한 방법이지만
> 만약 생성자가 수정될 경우 문제가 발생합니다.  
> **타입과 매개변수 갯수가 완벽하게 일치하는 생성자가 필요합니다.**

## Result aggregation
`.transform()`가 스프링 3.x부터 동작하지 않습니다.  
JPAQueryFactory를 수정해야합니다.

> java.lang.NoSuchMethodError: 'java.lang.Object org.hibernate.ScrollableResults.get(int)' with Hibernate 6.1.5.Final

+ [해결 코드 링크](https://github.com/querydsl/querydsl/issues/3428)
+ 하이버네이트 6.x 일때 수정해야합니다.
    ```Java
    @Configuration
    public class QueryDslConfig {
        @PersistenceContext
        private EntityManager entityManager;
        @Bean
        JPAQueryFactory jpaQueryFactory() {
            return new JPAQueryFactory(JPQLTemplates.DEFAULT, entityManager);
        }
    }
    ```