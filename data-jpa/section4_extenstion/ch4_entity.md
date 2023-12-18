# 스프링 데이터 JPA 구현체 분석 
+ 스프링 데이터 JPA가 제공하는 공통 인터페이스의 구현체
+ `org.springframework.data.jpa.repository.support.SimpleJpaRepository`  

```Java
@Repository
@Transactional(readOnly=true)
public class SimpleJpaRepository<T,ID> ... {

    @Transasctional
    public <S extend T> S save(S entity) {
        
        if(entityInfomation.isNew(entity) {
            em.persist(entity);
            return entity;
        } else {
            return em.merge(entity);
        }                         
    }
}

```
+ `@Repository 적용`: JPA 예외를 스프링이 추상화한 예외로 변환합니다.
+ `@Transactional` 트랜잭션 적용
  + JPA의 모든 변경은 트랜잭션 안에서 동작
  + 스프링 데이터 JPA는 변경(등록,수정,삭제)메소드를 트랜잭션 시작
  + 서비스 계층에 트랜잭션이 없을 경오 레포지토리에 적용하여 시작
  + 서비스 계층에 있다면 트랜잭션 전파로 같은 EntityManager 사용
  + 스프링 데이터 JPA를 사용할 때 트랜잭션이 없어도 데이터 등록,변경 가능
    이유는 JPA 구현체내에 트랜잭션을 가지고 있기 때문입니다.   

### `@Transactional(readOnly = true)` 최적화  
하이버네이트 전용 힌트인 `org.hibernate.readOnly`를 사용하면 엔티티를 읽기 전용으로 조회가 가능합니다. 
읽기 전용이므로 영속성 컨택스트는 스냅샷을 따로 보관하지 않습니다. 
+ 스냅샷을 저장하는 메모리 사용량을 최소화 할 수 있습니다.
+ 스냅샷이 없으므로 변경감지로 인한 업데이트가 데이터베이스에 반영되지 않습니다.  
  
스프링에서는 트랜잭션 읽기 전용 모드를 설정할 수 있습니다.  
트랜잭션에 `readOnly=true` 옵션을 주면 스프링 프레임워크가 하이버네이트 세션 플레시 모드를 
`NANUAL`로 설정합니다. 강제로 호출하지 않는 이상 `flush()`가 커밋전에 실행되지 않습니다.  
  
엔티티 매니저의 플러시 설정에는 `AUTO`,`COMMIT` 모드만 있고, `MANUAL` 모드는 없습니다.  
해당 모드는 강제로 플러시를 호출하지 않습니다. 
엔티티매니저의 `.unwarp`로 하이버네이트 세션이 반환됩니다.  
```Java
Sesstion session = entityManager.unwrap(Session.class);
```  
```Java  
@Transactional(readOnly=true) // 읽기전용 트랜잭션
public List<DataEntity> findDatas(){
    return em.createQuery("select ...",DataEntity.class)
             .setHint("org.hibernate.readOnly",true)
             .getResultList();

}
```
1. 읽기전용 트랜잭션 사용 : 플러시를 작동하지 않도록해서 성능향상가능
2. 읽기전용 엔티티 사용 : 엔티티를 읽기전용으로 읽어서 메모리 탈락

### merge와 persist 차이
+ `save() 메소드`
  + 새로운 엔티티면 저장(`persist`)
  + 새로운 엔티티가 아니면 병합(`merge`)  
  
저장하는 엔티티의 정보중 `ID`가 드어있으면 false,`true면` 즉시 로딩을 합니다.  

> 주의사항  
> `save()` 매서드를 실행하기전에 id값이 있으면 merge를 합니다.  


## 새로운 엔티티 구별하는 방법(중요)
**save() 매서드**  
+ save() 메서드 
    + 새로운 엔티티면 저장(persist)
    + 새로운 엔티티가 아니면 병합(merge) 

**새로운 엔티티를 구분하는 기준**  
+ 식별가의 객체일 때 `null`로 판단
  
`Persistable`을 구현합니다.  
```Java
package org.springframework.data.domain; 
public interface Persistable<ID> {
    ID getId(); 
    boolean isNew();
}
```
> 참고:  
    JPA는 식별자 생성 전략이 아니라 개발자가 래퍼형 클래스를 반환한다면, 
>   Persistable.isNew()를 구현해서 사용합니다.
  
**새로운 엔티티 확인 여부를 구현하는 Entity**입니다  
```Java
class EntityDIkey extends Persistable {
    @CreatedDate
    private LocalDateTime createdDate

    @Id
    private String id;
    @Overried
    public boolean isNew(){
        return createdDate == null;
    }
}
```
기초 레포지토리 구현 클래스 내부를 다시 살펴보면 
`save()`의 내부에 isNew로 새로운 엔티티 구분한 기준을 오버라이드를 통해 변경했습니다.  
  
`@GeneratedValue`를 제외하고 수동으로 값을 넣어준다면 위와 같이 코드를 작성해야합니다. 
수동으로 넣을경우 id 필드에 값이 들어가기 때문에 persist가 아니라 merge가 동작하기 때문입니다.  
  
> 참고 :  
> merge는 persist와 다르게 준영속에서 영속상태로 변경할 때 사용합니다.
> merge는 들어온 Entity의 id값이 있으면 영속성 상태로 판단합니다.
> 수동으로 Id값을 넣었기 때문에 persist가 아니라 merge가 동작합니다.  
> 따라서 수동으로 id값을 넣어야하는 상황이라면 위 인터페이스를 사용합니다.  
>  
