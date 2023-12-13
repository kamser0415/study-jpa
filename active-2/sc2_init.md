# `@PostConstruct`  
`@PostContructor`를 사용해서 데이터를 초기화 할때 `@Transactional`이 적용되지 않는 이유  
```Java
@PostConstructor
@Transactional
public void init() {
    //..
}
```  
해당 메소드는 트랜잭션이 적용되지 않습니다.  

Spring Framework에서 `@PostConstruct` 어노테이션이 붙은 메소드는 
빈이 생성되고 의존성 주입이 완료된 후에 실행됩니다. 
그러나 `@Transactional` 어노테이션이 적용된 `init()` 메소드는 
프록시 기반의 AOP(Aspect-Oriented Programming)를 사용하여 
트랜잭션 관리를 합니다.

`@PostConstruct`로 초기화된 메소드는 빈이 생성되고 
의존성 주입이 완료된 후 실행되므로, 해당 메소드 내에서 
`@Transactional`이 적용되어 있더라도 트랜잭션 관리가 되지 않을 수 있습니다.

일반적으로 스프링에서 `@Transactional`이 적용된 메소드는 프록시를 통해 실제 동작되며, 
이는 빈의 초기화(`@PostConstruct`)가 끝난 후에 적용됩니다. 
그래서 `@PostConstruct` 메소드 내부에서 `@Transactional`이 적용된 메소드를 
호출하더라도 트랜잭션 관리가 적용되지 않을 수 있습니다.  
  
### 최종 빈 등록 순서
1. 스프링 빈 생성
2. 생성된 빈 `@PostConstruct`로 초기화 실행
3. 초기화 이후 빈 후처리기가 `@Transactional`이 붙은 클래스를 프록시 객체로 만듭니다.

2번에서 3번을 같이 처리하려고 하다보니 문제가 발생합니다.  
트랜잭션이 적용된 빈을 주입 받아서 사용하면 해결됩니다.  
만약 new 연산자로 인스턴스를 생성하면 프록시가 적용되지 않기 때문에 주의해야합니다.

해결 방법은 `@Transactional`이 적용된 클래스의 메소드를 호출하면 됩니다. 
```Java
@Component
@Transactional
static class InitService {

    private final EntityManager em;
    public void dbInit() {
        //.. 초기화 로직
    }
}
```  
트랜잭션을 적용한 클래스를 호출해서 사용합니다.  
```Java
@Component
@RequiredArgsConstructor
public class InitDb {

    private final InitService initService;

    @PostConstruct
    public void init() {
        initService.dbInit();
    }
}
```  

이때 주의할점은 스프링 컨테이너에서 초기화된 빈을 호출해야합니다. 
만약 직접 `new`를 통해서 만든 인스턴스는 트랜잭션이 적용되지 않습니다.  
