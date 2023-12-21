# 프로젝션과 결과 반환 - 기본
프로젝션: select 할 대상을 지정합니다.  
  

### 프로젝션 대상이 하나 일 경우  
```Java
@DisplayName("프로젝션 대상이 값 타입이나 기본 타입일 경우")
@Test
void p1(){
    //given
    String username = "둘리";
    int userAge = 10;
    String city = "서울시";
    String zipcode = "15135";

    Member memberEntity = new Member(username, userAge);
    memberEntity.setAddress(new Address(zipcode, city));
    em.persist(memberEntity);

    em.flush();
    em.clear();
    //when
    List<Address> findMember = qFactory.select(member.address)
            .from(member)
            .where(member.age.eq(userAge))
            .fetch();
    //then
    Assertions.assertThat(findMember).hasSize(1);
    Assertions.assertThat(findMember).extracting("city","zipcode")
            .contains(Tuple.tuple(city, zipcode));
}
```  
+ 기본 타입이나 값타입일 경우에는 타입을 지정할 수 있습니다.  
  
### 결과 핸들링 하는 방법
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
#### Tuple
+ 이전 예제에서 계속 사용했기 때문에 생략합니다.
#### 프로퍼티 접근 (setter)
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
#### Expresstions.as(), ExpressionUtils.as  
를 활용해서 해당 DTO setter 메서드의 프로퍼티 명과 일치하는 별칭으로 변경합니다.
```Java
Projections.bean(
        MemberInfo.class, member.username, member.age,
        Expressions.as(team.name,"teamName")
))
```
`Projection.bean`은 setter 메서드의 이름을 보고 바인딩을 합니다.  

#### 필드 직접 접근 (fields)  
필드명을 보고 직접 접근하여 바인딩합니다
+ 

#### 생성자 
#### Mab<T,Group> 