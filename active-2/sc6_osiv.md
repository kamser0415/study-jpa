# OSIV
<img src="C:\study-jpa\jpa-study\JPA-Active-1\active-2\image\osiv.png" width="600"/>

```xml
spring.jpa.open-in-view: true(default)
```
> OpenEntityManagerInViewInterceptor를 등록합니다.   
> 이 인터셉터는 요청 처리 전체 동안 JPA EntityManager를 스레드에 바인딩합니다.  
  
+ 공식문서 설명
> 이것은 "Open EntityManager in View" 패턴을 위해 의도된 스프링 웹 요청 인터셉터입니다. 이 인터셉터는 JPA EntityManager를 스레드에 바인딩하여 요청 전체 처리 동안 사용할 수 있게 합니다. 이는 웹 뷰에서 레이지 로딩을 가능하게 하며, 원래 트랜잭션이 이미 완료된 상태임에도 불구하고 가능하게 합니다.  
> 
  
정리하면 트랜잭션이 종료가 되어도 지연 로딩으로 데이터를 읽어올 수 있습니다.

```Java
// participateAttributeName을 가져옵니다.
String key = this.getParticipateAttributeName();

// 현재 요청에 대한 WebAsyncManager를 가져옵니다.
WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);

// 비동기 매니저에게 동시적인 결과가 없거나 EntityManager 바인딩 인터셉터를 적용할 수 없는 경우에만 실행됩니다.
if (!asyncManager.hasConcurrentResult() || !this.applyEntityManagerBindingInterceptor(asyncManager, key)) {

    // EntityManagerFactory를 가져와서 현재 요청에 이미 EntityManager가 바인딩되어 있는지 확인합니다.
    EntityManagerFactory emf = this.obtainEntityManagerFactory();

    // 이미 EntityManager가 존재하는 경우 요청의 속성에 계수 값을 증가시킵니다.
    if (TransactionSynchronizationManager.hasResource(emf)) {
        Integer count = (Integer)request.getAttribute(key, 0);
        int newCount = count != null ? count + 1 : 1;
        request.setAttribute(this.getParticipateAttributeName(), newCount, 0);
    } else {
        // EntityManager가 존재하지 않는 경우, JPA EntityManager를 생성하고 바인딩하여 영속성 컨텍스트를 유지합니다.
        this.logger.debug("Opening JPA EntityManager in OpenEntityManagerInViewInterceptor");

        try {
            // EntityManager를 생성합니다.
            EntityManager em = this.createEntityManager();

            // EntityManager를 EntityManagerHolder에 넣고, TransactionSynchronizationManager를 사용하여 리소스를 바인딩합니다.
            EntityManagerHolder emHolder = new EntityManagerHolder(em);
            TransactionSynchronizationManager.bindResource(emf, emHolder);

            // 비동기 요청 인터셉터를 등록합니다.
            AsyncRequestInterceptor interceptor = new AsyncRequestInterceptor(emf, emHolder);
            asyncManager.registerCallableInterceptor(key, interceptor);
            asyncManager.registerDeferredResultInterceptor(key, interceptor);
        } catch (PersistenceException var8) {
            // EntityManager 생성에 실패한 경우 예외 처리합니다.
            throw new DataAccessResourceFailureException("Could not create JPA EntityManager", var8);
        }
    }
}
```
#### 코드 간단 정리
> 이 코드는 요청을 처리하기 전에 영속성 컨텍스트를 유지하고 EntityManager를 스레드에 바인딩하여 Lazy Loading과 같은 기능을 활성화합니다. 이는 데이터베이스 연산을 처리하는 중에도 영속성 컨텍스트를 계속 유지하여 데이터베이스 연산과 뷰 렌더링 사이의 트랜잭션 경계를 확장하는 데 사용됩니다
  
### OSIV 진행 순서
1. 클라이언트의 요청이 들어오면 서블릿 필터나, 스프링 인터셉터에서 영속성 컨텍스트를 생성합니다. 
    단 이때 트랜잭션은 시작하지 않습니다.  
2. 서비스 계층에서 `@Transactional`로 트랜잭션을 시작할 때 1번에서 미리 생성해둔 영속성 컨텍스트를 찾아와서 
    트랜잭션을 시작합니다.
3. 서비스 계층이 끝나면 트랜잭션을 커밋하고 영속성 컨택스트를 플러시합니다. 
    이때 트랜잭션은 끝내지만 영속성 컨텍스트는 종료하지 않습니다. 
4. 컨트롤러와 뷰까지 영속성 컨텍스트가 유지되므로 조회한 엔티티는 영속 상태를 유지합니다.
5. 서블릿 필터나, 스프링 인터셉터로 요청이 돌아오면 영속성 컨텍스트를 종료합니다. 
    이때 플러시를 호출하지 않고 바로 종료합니다.  
  
### 트랜잭션 없이 읽기
스프링이 제공하는 OSIV를 사용하면 컨트롤러 계층에서 트랜잭션 없이 엔티티를 수정할 수 없습니다.  
그리고 영속성 컨텍스트는 남아 있기 때문에 지연로딩을 사용할 수 있는 상태입니다.  
  
**OSIV 특징**
1. 영속성 컨텍스트를 컨트롤러 계층까지 유지합니다.
2. 컨트롤러 계층에는 트랜잭션이 없으므로 엔티티를 수정 할 수 없습니다.
    단) 영속성 컨텍스트는 남아있기 때문에 변경감지는 가능합니다.  
3. 트랜잭션 없이 읽기가 가능하기 때문에 지연로딩이 가능합니다.
  
### 테스트로 알아본 특징
지연 로딩은 프록시 객체입니다. 프록시는 고유 값을 가지고 있는데 
데이터를 조회하는 시점에 고유 값을 가지고 데이터베이스에서 조회를 합니다.  
  
즉 모든 데이터를 조회하는 쿼리인데 지연로딩으로 초기화를 한다고 할 때 
id값이 1,2,3만 읽어왔다면 지연 로딩 사이에 데이터가 추가되어도 동일한 레코드를 읽어옵니다. 
하지만 해당 id 값을 가진 데이터가 수정되거나 삭제된건 반영이 됩니다.  
  
### 커맨드와 쿼리 분리
실무에서 OSIV를 끈 상태로 복잡성을 관리하는 좋은 방법이 있다.  
**바로 Command와 Query를 분리하는 것이다.**

보통 비즈니스 로직은 특정 엔티티 몇 개를 등록하거나 수정하는 것이므로 성능이 크게 문제가 되지 않는다. 그런데 복
잡한 화면을 출력하기 위한 쿼리는 화면에 맞추어 성능을 최적화 하는 것이 중요하다. 하지만 그 복잡성에 비해 핵심 비
즈니스에 큰 영향을 주는 것은 아니다.
그래서 크고 복잡한 애플리케이션을 개발한다면, 이 둘의 관심사를 명확하게 분리하는 선택은 유지보수 관점에서 충분
히 의미가 있습니다.  
  
```text
repository
│
├── order
│   ├── OrderRepository
│   ├── ItemRepository
│   │
│   └── query
│       ├── OrderQueryRepository
│       ├── OrderQueryDto
│       ├── OrderItemQueryDto
│       ├── OrderFlatDto
│       └── OrderQueryRepository
```  
+ 레포지토리 - 도메인 - 패지키 분리 방식


## 정리 
이렇게 별도의 패키지로 분리해서 관리하는 방법을 사용하는 것을 권장합니다. 
+ 트래픽이 많은 서비스 API는 OSIV를 `false`로 사용하고
+ 내부 API나 ADMIN API는 OSIV를 `true`로 사용한다.
  
