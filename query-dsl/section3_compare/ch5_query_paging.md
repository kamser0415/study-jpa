# 스프링 데이터 페이징 활용 1 - Querydsl 페이징 연동
+ 스프링 데이터의 Page 객체, Pageable 객체를 활용합니다.
+ 전체 카운트를 한번에 조회하는 단순 방법
+ 데이터 내용과 전체 카운트를 별도로 조회하는 방법

### 사용자 정의 인터페이스에 페이징 메소드 추가  
```Java
public interface CustomRepository {
    Page<MemberTeamDto> searchPageSimple(MemberSearchCondition condition, Pageable pageable);
    Page<MemberTeamDto> searchPageComplex(MemberSearchCondition condition,Pageable pageable);
}
```
#### 전체 카운트를 한번에 조회하는 단순한 방법
> + searchPageSimple(직접 구현)
> + 지금은 사용 중지된 fetchResults() 사용
```Java
@Override
public Page<MemberTeamDto> searchPageSimple(MemberSearchCondition condition, Pageable pageable) {
    QueryResults<MemberTeamDto> result = factory
            .select(new QMemberTeamDto(
                    member.id,
                    member.username,
                    member.age,
                    team.id,
                    team.name))
            .from(member)
            .leftJoin(member.team, team)
            .where(usernameEq(condition.getUsername()),
                    teamNameEq(condition.getTeamName()),
                    ageGoe(condition.getAgeGoe()),
                    ageLoe(condition.getAgeLoe()))
            .offset(pageable.getOffset())
            .limit(pageable.getPageSize())
            .orderBy(member.age.desc())
            .fetchResults();

    List<MemberTeamDto> content = result.getResults();
    long total = result.getTotal();
    return new PageImpl<>(content, pageable, total);
}
```  
+ 간단한 코드는 상관없지만, 연관관계가 아닌 엔티티와 조인할 경우 
    조회쿼리가 불필요하게 복잡해집니다.
  ```Java
   leftJoin(post).on(member.id.eq(post.id))
  ```
  ```SQL
  select
      count(m1_0.member_id) 
  from
      member m1_0 
  left join
      post p1_0 
          on m1_0.member_id=p1_0.id
  ```  
+ Querydsl이 제공하는 `fetchResults()` 를 사용하면 내용과 전체 카운트를 한번에 조회할 수 있다.(실제 쿼리
  는 2번 호출)
+ `fetchResults()`는 카운스 쿼리 실행시 필요없는 `ORDER BY`는 제거합니다.  
  
#### 데이터 내용과 전체 카운트를 별도로 조회하는 방법  
+ `fetchCount()` 대신에 `fetchOne()`을 사용합니다.
  ```Java
  @Override
  public Page<MemberTeamDto> searchPageComplex(MemberSearchCondition condition,
                                               Pageable pageable) {
      List<MemberTeamDto> content = factory
              .select(new QMemberTeamDto(
                      member.id,
                      member.username,
                      member.age,
                      team.id,
                      team.name))
              .from(member)
              .leftJoin(member.team, team)
              .where(usernameEq(condition.getUsername()),
                      teamNameEq(condition.getTeamName()),
                      ageGoe(condition.getAgeGoe()),
                      ageLoe(condition.getAgeLoe()))
              .offset(pageable.getOffset())
              .limit(pageable.getPageSize())
              .fetch();
   //== 별도의 전체 Count 구하기
      Long total = factory
              .select(member.count())
              .from(member)
              .leftJoin(member.team, team)
              .where(usernameEq(condition.getUsername()),
                      teamNameEq(condition.getTeamName()),
                      ageGoe(condition.getAgeGoe()),
                      ageLoe(condition.getAgeLoe()))
              .fetchOne();
   //== fetchOne() 결과를 하나 가져옵니다.
      return new PageImpl<>(content, pageable, total);
  }
  ```  
+ 전체 카운트를 조회하는 방법을 최적화 할 수 있으면 분리하는 것이 좋습니다.  
  (조인 테이블을 줄이기)
  
### CountQuery 최적화  
> org.springframework.data.support.PageableExecutionUtils.getPage()  
> 주어진 contant, Pageable 및 LongSupplier에 따라 최적화를 적용하여 페이지를 구성합니다. 페이지의 구성은 결과 크기와 Pageable을 기반으로 총 수를 결정할 수 있는 경우 카운트 쿼리를 생략합니다.  
> 즉, 전체 페이지 크기가 페이징 객체의 페이지 사이즈보다 같거나 작을 경우등을 말합니다.
```Java
@Override
public Page<MemberTeamDto> searchPageComplex(MemberSearchCondition condition,
                                             Pageable pageable) {
    List<MemberTeamDto> content = factory
            .select(new QMemberTeamDto(
                    member.id,
                    member.username,
                    member.age,
                    team.id,
                    team.name))
            .from(member)
            .leftJoin(member.team, team)
            .where(usernameEq(condition.getUsername()),
                    teamNameEq(condition.getTeamName()),
                    ageGoe(condition.getAgeGoe()),
                    ageLoe(condition.getAgeLoe()))
            .offset(pageable.getOffset())
            .limit(pageable.getPageSize())
            .fetch();
    JPAQuery<Long> countQuery = factory
            .select(member.count())
            .from(member)
            .leftJoin(member.team, team)
            .where(usernameEq(condition.getUsername()),
                    teamNameEq(condition.getTeamName()),
                    ageGoe(condition.getAgeGoe()),
                    ageLoe(condition.getAgeLoe()));

    return PageableExecutionUtils.getPage(content,pageable,countQuery::fetchOne);
}
```  
+ 스프링 데이터 라이브러리가 제공
+ count 쿼리가 생략 가능한 경우 생략해서 처리  
  + 페이지 시작이면서 컨텐츠 사이즈가 페이지 사이즈보다 작을 때
  + 마지막 페이지 일 때 (offset + 컨텐츠 사이즈를 더해서 전체 사이즈 구함, 더 정확히는 마지막 페이지이면
  서 컨텐츠 사이즈가 페이지 사이즈보다 작을 때)
  
### QueryDSL API 예제코드
```Java
@RestController
@RequiredArgsConstructor
public class MemberController {

    private final MemberJpaRepository memberJpaRepository;
    private final MemberRepository memberRepository;
    
    @GetMapping("/v1/members")
    public List<MemberTeamDto> searchMemberV1(MemberSearchCondition condition) {
        return memberJpaRepository.search(condition);
    }
    @GetMapping("/v2/members")
    public Page<MemberTeamDto> searchMemberV2(MemberSearchCondition condition, Pageable pageable) {
        return memberRepository.searchPageSimple(condition, pageable);
    }
    @GetMapping("/v3/members")
    public Page<MemberTeamDto> searchMemberV3(MemberSearchCondition condition, Pageable pageable) {
        return memberRepository.searchPageComplex(condition, pageable);
    }
}
```  
> GET /v2/members?size=5&page=2  
>
### 스프링 데이터 정렬(Sort)  
스프링 데이터 JPA는 자신의 정렬(Sort)을 Querydsl의 정렬(OrderSpecifier)로 편리하게 변경하는 기능을 제공합니다.  

스프링 데이터의 정렬을 Querydsl의 정렬로 직접 전환하는 방법은 다음 코드를 참고하면 됩니다.
#### 스프링 데이터 Sort를 Querydsl의 OrderSpecifier로 변환  
```Java
@Override
public Page<MemberTeamDto> searchPageComplex(MemberSearchCondition condition,
                                             Pageable pageable) {
    JPAQuery<MemberTeamDto> query = factory
            .select(new QMemberTeamDto(
                    member.id,
                    member.username,
                    member.age,
                    team.id,
                    team.name))
            .from(member)
            .leftJoin(member.team, team)
            .where(usernameEq(condition.getUsername()),
                    teamNameEq(condition.getTeamName()),
                    ageGoe(condition.getAgeGoe()),
                    ageLoe(condition.getAgeLoe()))
            .offset(pageable.getOffset())
            .limit(pageable.getPageSize());

    JPAQuery<Long> countQuery = factory
            .select(member.count())
            .from(member)
            .leftJoin(member.team, team)
            .where(usernameEq(condition.getUsername()),
                    teamNameEq(condition.getTeamName()),
                    ageGoe(condition.getAgeGoe()),
                    ageLoe(condition.getAgeLoe()));
    for (Sort.Order o : pageable.getSort()) {
        // pageable.getSort()는 Pageable 객체에서 정렬 조건을 가져옵니다.
        // 이는 데이터를 정렬할 때 사용되는 필드와 방향(오름차순/내림차순)을 포함합니다.
        PathBuilder pathBuilder = new PathBuilder(member.getType(), member.getMetadata());
        
        // PathBuilder는 엔터티의 속성에 접근하기 위한 도구입니다. 
        // 여기서는 member 엔터티의 타입과 메타데이터를 사용하여 경로를 구축합니다.
        query.orderBy(new OrderSpecifier(
                o.isAscending() ? Order.ASC : Order.DESC,  // 오름차순 또는 내림차순 결정
                pathBuilder.get(o.getProperty())           // 정렬할 속성의 경로를 가져옵니다.
        ));
         List<MemberTeamDto> content = query.fetch();
    }
    
    return PageableExecutionUtils.getPage(content, pageable, countQuery::fetchOne);
}
```  
```Java
new PathBuilder(member.getType(), member.getMetadata())
```
> 참고:  
> 위 방식으로 동작하는건 시작하는 엔티티 PathBuilder를 통해서 동적으로 Entity의 프로퍼티를 가져올 수 있습니다. 
> 들어오는 값에 따라 분기를 태워서 정렬할 엔티티 기준을 세워도 동작을 잘 합니다.
> 루트 엔티티 범위를 넘어서는 동적 정렬이 필요하다면 데이터 페이징 객체가 제공하는 Sort를 사용하기 보다는 파라미터를 직접 처리하는 방식을 권장합니다.  
 [10분 우테코 정리](https://github.com/kamser0415/study-jpa/blob/main/query-dsl/section4_beamin/ch1_techtolk.md)
  
```Java
for (Sort.Order order : pageable.getSort()) {
    PathBuilder pathBuilder;
    if (order.getProperty().equals("name")) {
        pathBuilder = new PathBuilder<>(user.getType(), user.getMetadata());
    } else {
        pathBuilder = new PathBuilder<>(member.getType(), member.getMetadata());
    }
    query.orderBy(new OrderSpecifier(order.isAscending()? Order.ASC:Order.DESC,pathBuilder.get(order.getProperty())));
}
List<MemberTeamDto> content = query.fetch();
```  
+ `/v2/members?page=0&size=3&sort=name,asc&sort=username,asc`  
  
이렇게 작성을 하면 쿼리 스트링이 매핑이되서 Sort로 사용할 수 있습니다.  
```SQL
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
        on m1_0.member_id=u1_0.id 
order by
    u1_0.name,
    m1_0.username 
limit
    ?, ?
```

정렬도 동작을 잘 합니다.