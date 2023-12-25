# QueryDSL 빈 등록 방법
### 생성자로 초기화
```Java
@Repository
public class MemberRepository {
    private final EntityManager em;
    private final JPAQueryFactory factory;
    public ConstInit(EntityManager em) {
        this.em = em;
        this.factory = new JPAQueryFactory(JPQLTemplates.DEFAULT, em);
    }
}
```
장점 : 
1. 테스트 코드를 작성할 때 엔티티 매니저만 주입하면 된다.    

단점 : 
1. QueryDSL의 의존관계가 숨겨져있다.
2. 레파지토리 생성시 마다 반복되는 코드가 발생한다.

### 빈 등록후 주입받기
```Java
@Configuration
class JpaQueryConfig {
    @Bean
    JPAQueryFactory jpaQueryFactory(EntityManager entityManager){
        return new JPAQueryFactory(JPQLTemplates.DEFAULT, entityManager);
    }
}
@Repository
@RequiredArgsConstructor
public class MemberRepository {
    private final JPAQueryFactory factory;
}
```
장점 :   
1. `@RequiredArgsConstructor` 사용하면 코드가 간결해진다.
2. 의존 관계를 생성자를 통해서 알 수 있다.
  
단점 : 
1. 테스트 코드 작성시 `JPAQueryFactory`를 주입해야하는 번거로움이 생긴다.  

### JpaQueryFactory를 주입받으면 테스트하기 불편한 이유
> 레포지토리 테스트를 할 경우 스프링 컨테이너는 실행이 되어야합니다.
  
`@SpringBootTest`에는 모든 빈 정보를 스프링 컨테이너 저장소에 초기화를 하지만, 
`@DataJpaTest`같은 경우 DataJpa에 관련된 클래스만 빈으로 등록합니다. 
아래와 같이 별도로 `JpaQueryFactory`를 등록해야합니다.  
> `JPAQueryFactory`는 `@DataJpaTest`에서 빈으로 등록하지 않기 때문에 
생성자 초기화를 하지못하여 에러가 발생하기 때문입니다.

```Java
public class QueryRepositoryImpl implements QueryRepository{

    private final JPAQueryFactory queryFactory;

    public QueryRepositoryImpl(JPAQueryFactory jpaQueryFactory) {
        this.queryFactory = jpaQueryFactory;
    }
    
}
```
```Java
@DataJpaTest
@Import(TestBean.TestConfig.class)
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
public class TestBean {

    @Autowired
    QueryRepositoryImpl queryRepository;
    
    @TestConfiguration
    static class TestConfig {
    
        @PersistenceContext
        EntityManager entityManager;
        
        @Bean
        public JPAQueryFactory jpaQueryFactory(EntityManager entityManager) {
            return new JPAQueryFactory(entityManager);
        }
        
    }
}
```
## JPAQueryFactory 동시성 문제
는 없습니다.  
QueryDSL의 `JPAQueryFactory`는 DB 커넥션을 `EntityManger`에게 위임하여 동작합니다. 
동시성 문제는 `EntityManger`가 제어하기 때문에 `JPAQueryFactory`도 동시성 문제가 없습니다.  

`EntityManger`는 트랜잭션 내에서 동작을 합니다.
1. 쓰기 지연
2. 변경 감지로 데이터 변경    
  
등 JPA가 제공하는 기능은 데이터베이스 트랜잭션이 동작해야 합니다. 
같은 트랜잭션 내에서 엔티티 매니저를 공유하여 여러 쓰레드가 접근할 경우에는 문제가 발생할 수 있지만, 
스프링 부트에서 주입받는 엔티티 매니저는 프록시 객체가 들어오게 됩니다.  
  
`@Transactional`과 같이 사용하게 되면 쓰레드 로컬에 트랜잭션 고유번호를 관리하기 때문에 
해당 고유번호로 엔티티 매니저가 커넥션을 연결하여 사용하기 때문에 동시성 문제는 발생하지 않습니다.  
  

