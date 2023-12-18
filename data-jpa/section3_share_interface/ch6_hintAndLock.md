# JPA Hint & Lock  

## JPA Hint
JPA 쿼리 힌트(SQL 힌트가 아니라 JPA 구현체에게 제공하는 힌트)  
쿼리 힌트 사용
```java
@QueryHints(value = @QueryHint(name = "org.hibernate.readOnly", value = "true")) 
Member findReadOnlyByUsername(String username);
```
쿼리 힌트 사용 확인
```java
@Test
public void queryHint() throws Exception { 
//given
    memberRepository.save(new Member("member1", 10)); 
    em.flush();
    em.clear();
    //when
    Member member = memberRepository.findReadOnlyByUsername("member1"); 
    member.setUsername("member2");
    em.flush(); //Update Query 실행X 
}
```
쿼리 힌트 Page 추가 예제
```java
@QueryHints(value = { @QueryHint(name = "org.hibernate.readOnly", 
                                 value = "true")},
            forCounting = true)
Page<Member> findByUsername(String name, Pageable pageable);
```
`org.springframework.data.jpa.repository.QueryHints` 어노테이션을 사용   
+ `forCounting` : 반환 타입으로 Page 인터페이스를 적용하면 추가로 호출하는 페이징을 위한count 쿼리도 쿼리 힌트 적용(기본값 true )  
  
## LOCK
JPA도 락을 걸수 있는데 JPA가 제공하는 기능입니다.  
어노테이션으로 간편하게 사용할 수 있도록 제공합니다.  

실시간 트레픽이 많은 서비스에는 가급적이면 Lock을 걸면 안된다.
perssimistic을 걸면 해당 관련된 Lock을 전부 처리된다.

걸려면 실제 db락을 거는게 아니라 버전링이라는 매커니즘으로 해결하는 방법이 있다.  
그 방법으로 풀거나 아니면 락을 안걸고 좀 다른 방법으로 해결하거나 이런 것을들 찾아보는걸 권장합니다.

실시간 트래픽이 많지 않고 실시간 트래픽보다 돈을 맞추거나 중요한 경우
쓰기 잠금을 사용하는 방법도 좋다고 합니다.
```Java
@Lock(LockModeType.PESSIMISTIC_WRITE)
List<Member> findByUsername(String name);

// Redeclaration of a CRUD method
@Lock(LockModeType.READ)
List<User> findAll();
```
+ org.springframework.data.jpa.repository.Lock 어노테이션을 사용
