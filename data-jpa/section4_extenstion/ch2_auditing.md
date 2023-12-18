# Auditing
[Auditing 공식문서](https://docs.spring.io/spring-data/jpa/reference/auditing.html)

> Spring Data는 엔티티를 생성하거나 변경한 사용자와 변경된 시간을 투명하게 추적하는 기능을 제공합니다.
>
+ 엔티티 클래스에 주석을 사용하거나 인터페이스를 구현하여 정의된 감사 메타데이터를 적용해야 합니다.
+ 감사 기능을 활성화하려면 Annotation 구성 또는 XML 구성을 통해 필요한 인프라 구성 요소를 등록해야 합니다.

**엔티티를 생성, 변경할 때 변경한 사람과 시간을 추적할 수 있습니다.**
+ 등록일
+ 수정일
+ 등록자
+ 수정자

### 순수 JPA 사용
예제 코드
```Java
@Getter
@MappedSuperclass // 엔티티 속성만 사용
public class JpaBaseEntity { 

    @Column(updatable = false) 
    private LocalDateTime createdDate; 
    private LocalDateTime updatedDate; 
    
    @PrePersist // persist 전에 실행됩니다. 
    public void prePersist() {
        LocalDateTime now = LocalDateTime.now();
        createdDate = now; 
        updatedDate = now; 
    }
    
    @PreUpdate // update 전에 실행됩니다.
    public void preUpdate() {
        updatedDate = LocalDateTime.now(); 
    }
}
```  
#### JPA 주요 이벤트 어노테이션
[공식문서](https://docs.jboss.org/hibernate/orm/6.4/userguide/html_single/Hibernate_User_Guide.html#events-jpa-callbacks)

| Type         | Description                                                                           |
|--------------|---------------------------------------------------------------------------------------|
| @PrePersist  | 실제로 엔티티 매니저의 persist 작업이 실행되거나 연쇄적으로 실행되기 전에 실행됩니다. 이 호출은 persist 작업과 동기적으로 수행됩니다.    |
| @PreRemove   | 실제로 엔티티 매니저의 remove 작업이 실행되거나 연쇄적으로 실행되기 전에 실행됩니다. 이 호출은 remove 작업과 동기적으로 수행됩니다.      |
| @PostPersist | 실제로 엔티티 매니저의 persist 작업이 실행되거나 연쇄적으로 실행된 후에 실행됩니다. 이 호출은 데이터베이스 INSERT가 실행된 후에 호출됩니다. |
| @PostRemove  | 실제로 엔티티 매니저의 remove 작업이 실행되거나 연쇄적으로 실행된 후에 실행됩니다. 이 호출은 remove 작업과 동기적으로 수행됩니다.       |
| @PreUpdate   | 데이터베이스의 UPDATE 작업 전에 실행됩니다.                                                           |
| @PostUpdate  | 데이터베이스의 UPDATE 작업 후에 실행됩니다.                                                           |
| @PostLoad    | 현재 영속성 컨텍스트로 엔티티가 로드되거나 엔티티가 리프레시된 후에 실행됩니다.                                          |

## 스프링 데이터 JPA 사용
+ `@EnableJpaAuditing`  스프링 부트 설정 클래스에 적용해야함
    ```Java
    @Configuration
    @EnableJpaAuditing
    class Config {
        @Bean
        public AuditorAware<AuditableUser> auditorProvider() {
        return new AuditorAwareImpl();
        }
    }
    ```
+ `@EntityListeners(AuditingEntityListener.class)`  엔티티에 적용
    ```Java
    @Entity
    @EntityListeners(AuditingEntityListener.class)
    public class MyEntity {...}
    ```  

[주석 기반 감사 메타데이터 공식 문서](https://docs.spring.io/spring-data/jpa/reference/auditing.html#auditing.annotations)

#### 사용 메타어노테이션
+ @CreatedDate
+ @LastModifiedDate
+ @CreatedBy
+ @LastModifiedBy

> 엔티티를 생성하거나 수정한 사용자를 캡처하거나 변경한 시점을 캡처합니다.
>
#### 변경 사항을 캡쳐하기 위한 주석 타입
+ JDK8 날짜 및 시간유형
+ long,Long
+ Date,Calendar

감시 메타데이터는 엔티티의 최상위 수준에 있을 필요가 없습니다.
사용중인 레포지토리에 따라 임베디드된 엔티티에 추가할 수 있습니다.
```Java
class Customer {

  private AuditMetadata auditingMetadata;
  // … further properties omitted
}

class AuditMetadata {

  @CreatedBy
  private User user;

  @CreatedDate
  private Instant createdDate;

}
```  

### AuditorAware
+ **@CreatedBy 또는 @LastModifiedBy 중 하나를 사용하는 경우**

> AuditorAware<T> SPI 인터페이스를 구현하여 인프라에
현재 응용 프로그램과 상호 작용하는 사용자 또는 시스템이
누구인지 알려야 합니다.
제네릭 타입 T는 @CreatedBy 또는 @LastModifiedBy로 주석
처리된 속성이어야 할 타입을 정의합니다.

+ 예시코드
```Java
class SpringSecurityAuditorAware implements AuditorAware<User> {

  @Override
  public Optional<User> getCurrentAuditor() {

    return Optional.ofNullable(SecurityContextHolder.getContext())
            .map(SecurityContext::getAuthentication)
            .filter(Authentication::isAuthenticated)
            .map(Authentication::getPrincipal)
            .map(User.class::cast);
  }
}
```  
간단하게 예제를 만들어볼 수 있습니다.
```Java
@Bean
public AuditorAware<String> auditorProvider() {
  return () -> Optional.of(UUID.randomUUID().toString());
}

@CreatedBy
@Column(updatable = false) 
private String createdBy;

@LastModifiedBy
private String lastModifiedBy;
```  
**제네릭 타입과 자료형은 일치해야합니다.**

#### BaseEntity을 분리해서 사용하기
모든 엔티티가 수정자,등록자,등록일,수정일이 필요없기 때문에
+ `BaseTimeEntity`: 수정일,등록일
+ `BaseAdminEntity`: 수정자,등록자

> 참고: 실무에서 대부분의 엔티티는 등록시간, 수정시간이 필요하지만, 등록자, 수정자는 없을 수도 있다. 그래서
다음과 같이 Base 타입을 분리하고, 원하는 타입을 선택해서 상속합니다.
>

```Java
public class BaseTimeEntity { 
  @CreatedDate
  @Column(updatable = false) 
  private LocalDateTime createdDate;
  @LastModifiedDate
  private LocalDateTime lastModifiedDate;
}
public class BaseEntity extends BaseTimeEntity { 
  @CreatedBy
  @Column(updatable = false) 
  private String createdBy;
  @LastModifiedBy
  private String lastModifiedBy; 
}
```  
> 참고: 저장시점에 등록일, 등록자는 물론이고, 수정일, 수정자도 같은 데이터가 저장된다. 데이터가 중복 저장되는
것 같지만, 이렇게 해두면 변경 컬럼만 확인해도 마지막에 업데이트한 유저를 확인 할 수 있으므로 유지보수 관점  
에서 편리하다. 이렇게 하지 않으면 변경 컬럼이 null 일때 등록 컬럼을 또 찾아야 한다.  
참고로 저장시점에 저장데이터만 입력하고 싶으면 @EnableJpaAuditing(modifyOnCreate = false)
옵션을 사용하면 된다.
