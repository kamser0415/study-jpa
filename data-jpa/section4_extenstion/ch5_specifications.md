# specifications
+ [specifications 공식문서 설명](https://docs.spring.io/spring-data/jpa/reference/jpa/specifications.html)

JPA는 **프로그래밍 방식**으로 쿼리를 작성할 수 있는 `Criteria API`를 했습니다.

+ Criteria는 도메인 클래스에 대한 **쿼리의 where 절을 정의**합니다.

Criteria는 JPA Criteria API 제약 조건을 따르는 엔티티에 대한 술어(predicate)로 간주할 수 있습니다.
+ 참 또는 거짓으로 평가
+ `AND , OR`같은 연산자로 조합해서 다양한 검색 조건을 쉽게 생성합니다.  
  (컴포넌트 패턴: 연결해서 사용할 수 있습니다.)  
+ `org.springframework.data.jpa.domain.Specification`를 클래스로 정의합니다.

> 참고:  
Spring Data JPA는 에릭 에반스(Eric Evans)의 책 "도메인 주도 설계(Domain Driven Design)"
에서 명세(specification)의 개념을 채택하며, JPA Criteria API로 이러한 명세를 정의할 수 
있는 API를 제공합니다. 명세(specification)를 지원하기 위해 **JpaSpecificationExecutor**
인터페이스를 사용하여 리포지토리 인터페이스를 확장할 수 있습니다.  
>  
  
```Java
public interface CustomerRepository extends CrudRepository<Customer, Long>, 
                                        JpaSpecificationExecutor<EntityClass> {
    //.. 생략
}
```  
+ **JpaSpecificationExecutor** 인터페이스  
    추가된 인터페이스에는 명세(specification)를 다양한 방식으로 실행할 수 있는 메서드들이 포함되어 있습니다. 
    예를 들어, findAll 메서드는 명세(specification)과 일치하는 모든 엔티티를 반환합니다.  
    아래는 해당 예시입니다:  
    ```Java
    public interface JpaSpecificationExecutor<T> {
        Optional<T> findOne(@Nullable Specification<T> spec); 
        List<T> findAll(Specification<T> spec);
        Page<T> findAll(Specification<T> spec, Pageable pageable); 
        List<T> findAll(Specification<T> spec, Sort sort);
        long count(Specification<T> spec); 
        boolean exists(Specification<T> spec);
        <S extends T, R> R findBy(Specification<T> spec, Function<FluentQuery.FetchableFluentQuery<S>, R> queryFunction);
    }
    ```  
### 명세 사용 코드
실제 구현해야하는 매서드는 `toPredicate()` 입니다. 
람다식으로 정의를 해서 간단하게 사용할 수 있습니다.
```Java
return new Specification<Member>(){
    @Override
    public Predicate toPredicate(Root<Member> root, CriteriaQuery<?> query, CriteriaBuilder criteriaBuilder) {
        /*
        root는 주 테이블이라 생각하면 됩니다.
        where 절의 문법은 CriteriaBuilder를 통해서 템플릿을 만들고
        CriteriaBuilder의 메서드를 통해서 사용할 명령어를 선택하면 됩니다.
        */
    }
};
```
+ MemberSpec(JPA criteria)
```Java
public static class MemberSpec {
    // 조인하고 팀 이름이 T로 시작하는
    public static Specification<Member> likeTeamName(final String teamName) {
        return (root, query, builder) -> {
            if (!StringUtils.hasText(teamName)) {
                return null;
            }
            Join<Member, Team> t = root.join("team", JoinType.INNER);// 이너조인
            return builder.like(t.get("username"),teamName+"%",'\\');
        };
    }
    // 성인 찾기
    public static Specification<Member> isAdult() {
        return (root, query, criteriaBuilder) ->
                criteriaBuilder.greaterThanOrEqualTo(root.get("age"),20);
    }
    // 정렬
    public static Specification<Member> orderByDesc(String columnName) {
        return (root, query, criteriaBuilder) -> {
            Path<Object> columnPath = root.get(columnName);
            Order desc = criteriaBuilder.desc(columnPath);
            query.orderBy(desc);
            return null;
        };
    }
}
```  
+ 테스트 코드
```Java
@DisplayName("팀 이름이 T로 시작하고 회원나이가 성인인 사람을 찾습니다.")
@Test
void c1(){
    //given
    Team team = new Team("T1");
    entityManager.persist(team);
    entityManager.flush();
    entityManager.clear();

    Member member1 = new Member("고딩둘리", 19,team);
    Member member2 = new Member("대딩둘리", 20,team);
    Member member3 = new Member("직딩둘리", 30,team);
    memberRepository.saveAll(List.of(member1,member2,member3));
    
    //when
    List<Member> findMember = memberRepository.findAll(MemberSpec.likeTeamName("T")
            .and(MemberSpec.isAdult())
            .and(MemberSpec.orderByDesc("age"))
    );

    //then
    Assertions.assertThat(findMember).hasSize(2);
    //순서 일치
    Assertions.assertThat(findMember).containsExactly(member3, member2);
//        Assertions.assertThat(findMember).containsExactly(member2, member3);
}
```  
```SQL
select
    m1_0.member_id,
    m1_0.address_id,
    m1_0.age,
    m1_0.created_date,
    m1_0.team_id,
    m1_0.updated_date,
    m1_0.username 
from
    member m1_0 
join
    team t1_0 
        on t1_0.team_id=m1_0.team_id 
where
    t1_0.username like ? escape '\' 
    and m1_0.age>=? 
order by
    m1_0.age desc
```