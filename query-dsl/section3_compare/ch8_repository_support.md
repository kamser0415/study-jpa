# 리포지토리 지원
## QuerydslRepositorySupport

### 장점 
+ getQuerydsl().applyPagination()  스프링 데이터가 제공하는 페이징을 Querydsl로 편하게 변환 가능합니다.
+ from()으로 시작가능
+ EntityManger 제공  
```Java
public Page<MemberTeamDto> searchPageCondition(MemberSearchCondition condition,
                                               Pageable pageable) {
    JPQLQuery<MemberTeamDto> jpaQuery = from(member)
            .leftJoin(member.team, team)
            .leftJoin(user).on(team.id.eq(user.id))
            .where(usernameEq(condition.getUsername()),
                    teamNameEq(condition.getTeamName()),
                    ageGoe(condition.getAgeGoe()),
                    ageLoe(condition.getAgeLoe()))
            .select(new QMemberTeamDto(
                    member.id,
                    member.username,
                    member.age,
                    team.id,
                    team.name,
                    user.name));

    JPQLQuery<MemberTeamDto> query = getQuerydsl().applyPagination(pageable, jpaQuery);
    List<MemberTeamDto> fetch = query.fetch();
    long total = from(member)
            .where(usernameEq(condition.getUsername()),
                    teamNameEq(condition.getTeamName()),
                    ageGoe(condition.getAgeGoe()),
                    ageLoe(condition.getAgeLoe()))
            .select(member.count())
            .fetchFirst();
    return new PageImpl<>(fetch, pageable, total);
}
```
+ 테스트 코드
```Java
@DisplayName("Support 조회 테스트")
@Test
void supportTest(){
    //given
    MemberSearchCondition condition = new MemberSearchCondition();
    condition.setAgeGoe(10);
    condition.setAgeLoe(40);
    condition.setTeamName("teamA");
    //when
    Page<MemberTeamDto> memberTeamDtos = memberRepository.searchPageCondition(condition, PageRequest.of(0, 3));
    //then
    Assertions.assertThat(memberTeamDtos.getContent()).extracting("username").containsExactly("member10","member12","member14");
}
```
SQL
```Java
select
    m1_0.member_id,
    m1_0.username,
    m1_0.age,
    m1_0.team_id,
    t1_0.name,
    u1_0.name 
from
    member m1_0 
left join
    team t1_0 
        on t1_0.team_id=m1_0.team_id 
left join
    users u1_0 
        on m1_0.team_id=u1_0.id 
where
    t1_0.name=? 
    and m1_0.age>=? 
    and m1_0.age<=? 
limit
    ?, ?
```  
+ 데이터가 많지 않을 경우에는 PageImpl을 사용해도 상관없다.
+ no offset를 구현하기 위해서는 페이징을 직접작성해야합니다.  
  
### Sort가 적용이 정말 안될까?
+ 정말 안된다.
  
테스트코드
```Java
@DisplayName("Support 조회 테스트")
@Test
void supportSortTest(){
    //given
    MemberSearchCondition condition = new MemberSearchCondition();
    condition.setAgeGoe(10);
    condition.setAgeLoe(40);
    condition.setTeamName("teamA");
    //when
    PageRequest pageRequest = PageRequest.of(0, 3, Sort.by(Sort.Direction.DESC, "username","age"));
    Page<MemberTeamDto> memberTeamDtos = memberRepository.searchPageCondition(condition, pageRequest);
    //then
    Assertions.assertThat(memberTeamDtos.getContent()).extracting("username").containsExactly("member10","member12","member14");
} 
```  

+ 예외메세지
```Java
InvalidDataAccessApiUsageException: org.hibernate.query.SemanticException: Could not interpret path expression 'member.username' 
```  

#### 발생원인
```Java
// Support 클래스를 초기화 할때
public CustomRepositoryImpl(EntityManager entityManager) {
    super(Member.class);
    this.factory = new JPAQueryFactory(entityManager);
}
// 상위 클래스에서 초기화 할때   
public QuerydslRepositorySupport(Class<?> domainClass) {
    Assert.notNull(domainClass, "Domain class must not be null");
    this.builder = new PathBuilderFactory().create(domainClass);
}
// new PathBuilderFactory()
public PathBuilderFactory() {
    this("");
}
// 이걸 사용해야하는데
public PathBuilderFactory(String suffix) {
    this.suffix = suffix;
}
기본 값이 "" 입니다.
private String variableName(Class<?> type) {
    return StringUtils.uncapitalize(type.getSimpleName()) + suffix;
}
변수명은 name+suffix; 결정
```  
그러다보니 Sort.by로 정렬 조건을 넘겨주면 
+ Querydsl이 만드는 식별자는 기본값은 `name+"1"`;
    ```Java
    QMember.member = new QMember("member1");
    ```
+ Data Jpa가 만드는 기본 식별자는 `name`;
    ```Java
    StringUtils.uncapitalize(type.getSimpleName()) + "";
    ```  
    
따라서 정렬을 하려고 하면 식별 변수값이 달라서 오류가 발생합니다.  
그래서 Sort.by에 식별 변수를 변경할 수 없고 
QMember.member도 고정 식별변수를 사용하기때문에 
정상 동작을 하기 위해서 new QMember("member");를 사용하면 됩니다.  
아래와 같이 코드를 수정하면 됩니다:  
물론 `static final`로 공통으로 사용해도 됩니다.   
`private static final QMember member = new QMember("member");`
```Java
private BooleanExpression usernameEq(String username) {
    QMember member = new QMember("member");
    return isEmpty(username) ? null : member.username.eq(username);
}
private BooleanExpression teamNameEq(String teamName) {
    QTeam team = new QTeam("team");
    return isEmpty(teamName) ? null : team.name.eq(teamName);
}
private BooleanExpression ageGoe(Integer ageGoe) {
    QMember member = new QMember("member");
    return ageGoe == null ? null : member.age.goe(ageGoe);
}
private BooleanExpression ageLoe(Integer ageLoe) {
    QMember member = new QMember("member");
    return ageLoe == null ? null : member.age.loe(ageLoe);
}
```  
`Sort.by(Sort.Direction.DESC, "username","age")`의 정렬을 사용할 경우 
식별 변수 값이 동일하게 맞춰주면 되지만 정렬 기준 식별 변수가 달라지면 오류가 발생하게 됩니다.  
  
개발자가 직접 `Sort -> SortSpecification`로 변환하는 것이 개발자가 유지보수 하기도 쉽고 
다른 사람들이 코드를 해석하기도 간편합니다.