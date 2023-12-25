# 스프링 데이터에서 제공하는 기능  
+ 제약이 커서 복잡한 실무 환경에서 사용하기에는 많이 부족하다.  
  
## QuerydslPredicateExecutor
+ [공식문서](https://docs.spring.io/spring-data/jpa/reference/repositories/core-extensions.html#core.extensions.querydsl)  
```Java
public interface QuerydslPredicateExecutor<T> {

  Optional<T> findById(Predicate predicate);

  Iterable<T> findAll(Predicate predicate);

  long count(Predicate predicate);

  boolean exists(Predicate predicate);

  // … more functionality omitted.
}
```  
#### 레포지토리에 적용
```Java
interface MemberRepository extends JpaRepository<Member, Long>, 
                                   QuerydslPredicateExecutor<Member> {
}
```  
+ Service 계층에서 사용할 경우
```Java
public MemberDto findAll(MemberSearchCond cond){
    return repository.findAll(QMember.member.age.eq(cond.getAge));
}
```  
#### 한계점  
1. `QuerydslPredicateExecutor`에 정의한 엔티티를 기준으로 묵시적 조인만 가능합니다.  
    **left join**이 불가능 합니다.
    + 테이블 하나로 할수 있는 기능이 거의 없습니다.
2. 클라이언트가 QueryDSL에 의존해야합니다.  
    단) 아래 코드와 같이 인터페이스로 분리할 경우 가능합니다.

#### 레파지토리 구현체 생성
```Java
public interface MemberServiceFacade {
    public List<Member> findAll(MemberSearchCondition cond);
}

@Repository
@RequiredArgsConstructor
public class MemberServiceFacadeImpl implements MemberServiceFacade {

    private final QueryMemberRepository queryMemberRepository;
    
    @Override
    public List<Member> findAll(MemberSearchCondition cond) {
        Iterator<Member> iterator = queryMemberRepository.findAll(
                teamNameEq(cond.getTeamName()).and(
                        ageGoe(cond.getAgeGoe())).and(
                        ageLoe(cond.getAgeLoe()))
        ).iterator();
        List<Member> newList = new ArrayList<>();
        while (iterator.hasNext()) {
            newList.add(iterator.next());
        }
        return newList;
    }
    private BooleanBuilder teamNameEq(String teamName) {
        return nullOrExpression(()->team.name.eq(teamName));
    }

    private BooleanBuilder ageGoe(Integer ageGoe) {
        return nullOrExpression(()->member.age.goe(ageGoe));
    }

    private BooleanBuilder ageLoe(Integer ageLoe) {
        return nullOrExpression(()->member.age.loe(ageLoe));
    }

    private BooleanBuilder nullOrExpression(Supplier<BooleanExpression> t) {
        try {
            return new BooleanBuilder(t.get());
        } catch (IllegalArgumentException e) {
            return new BooleanBuilder();
        }
    }
}
```  
#### 서비스 계층
```Java
@Component
@RequiredArgsConstructor
public class MemberServiceQuery {
     private final MemberServiceFacade memberServiceFacade;

    public List<Member> findAll(MemberSearchCondition condition) {
        return memberServiceFacade.findAll(condition);
    }
}
```  
+ `Service -> RepositoryInterface`  
+ `RepositoryImpl -> RepositoryInterface`  

#### 정리
구조로 만들면 서비스 계층에서는 QueryDSL을 사용하지 않아도 되지만, 
막상 코드로 작성해보니 묵시적 조인만 사용가능한 쿼리에 불필요한 작업이 많아져서 
실무에서는 사용을 잘 안할 거같습니다.  
  
서비스 계층에 QueryDSL이 들어와도 상관 없다면 
단일 테이블 조회는 편리하게 사용할 수 있습니다.
  
이유는 :  
+ 복잡한 조건들을 명시적으로 작성을 할 수 있기 때문입니다. 

> 참고:   
> QuerydslPredicateExecutor 는 Pagable, Sort를 모두 지원하고 정상 동작한다.  
  
