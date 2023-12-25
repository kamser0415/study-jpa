# QueryDSL Web 지원
+ [공식문서](https://docs.spring.io/spring-data/jpa/reference/repositories/core-extensions.html#core.web.type-safe)  
  
```Java
@GetMapping("/web/members")
public Page<MemberTeamDto> findByCondition(@QuerydslPredicate(root = Member.class) Predicate predicate,
                                    Pageable pageable) {
    Page<Member> findMembers = memberServiceFacade.findAll(predicate, pageable);
    return findMembers.map(MemberTeamDto::new); 
}
```  

웹 계층에 `@QuerydslPredicate(root = Member.class)`을 추가해야 합니다.  
만약 실행되는 쿼리를 커스터마이징 하고 싶다면 아래와 같이 인터페이스를 상속합니다.  


```Java
public interface QueryMemberRepository extends Repository<Member,Long>, CustomRepository, QuerydslPredicateExecutor<Member>, QuerydslBinderCustomizer<QMember> {
    @Override
    default void customize(QuerydslBindings bindings, QMember user) {
        bindings.bind(user.username).first((path, value) -> path.startsWith(value));
        bindings.bind(String.class)
                .first((StringPath path, String value) -> path.contains(value));
    }
}
```

#### URL
```HTTP
GET /web/members?username=mem&page=0&size=20&sort=id HTTP/1.1
Host: localhost:8080
```

#### 커스텀 
+ **startWith**
    ```Java
    bindings.bind(user.username).first((path, value) -> path.startsWith(value));
    ```
    SQL
    ```SQL
    select count(member1)
    from Member member1
    where member1.username like 'mem%'1 escape '!' 
    ```
+ **endWith**
    ```Java
    bindings.bind(user.username).first((path, value) -> path.endsWith(value));
    ```
     SQL
    ```SQL
    select member1
    from Member member1
    where member1.username like '%mem'1 escape '!'
    order by member1.id asc
    ```

### 한계점  
+ 단순한 조건만 가능
+ 컨트롤러 계층에서 간단하게 사용하는 목적이지만 조건을 커스텀하는 기능을 확인하려면 결국 레포지토리 계층까지 가야합니다.
+ 컨트롤러가 `@QuerydslPredicate`을 사용하여, QueryDSL 기술을 의존합니다.

