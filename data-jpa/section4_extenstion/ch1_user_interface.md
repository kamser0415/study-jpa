# 사용자 정의 인터페이스  
[Custom Repository Implementations 공식문서](https://docs.spring.io/spring-data/jpa/reference/repositories/custom-implementations.html)  
## 사용 목적
스프링 데이터는 적은 양의 코딩으로 쿼리 메서드를 생성할 수 있는 다양한 옵션을 제공합니다.   
그러나 이러한 옵션이 필요에 맞지 않는 경우 리포지토리 메서드에 자체적인 커스텀 구현을 
제공할 수도 있습니다.    
+ JPA 직접 사용( EntityManager )
+ 스프링 JDBC Template 사용
+ MyBatis 사용
+ 데이터베이스 커넥션 직접 사용 등등...
+ Querydsl 사용

> 참고:  
> 무조건 사용하는 기능은 아닙니다. 애플리케이션 규모에 필요할 경우 사용하면 됩니다.  

개별 리포지토리에 기능을 추가하려면 먼저 사용자 정의 기능을 위한 프래그먼트 인터페이스와 구현체를 다음과 같이 정의해야 합니다:

**사용자 정의 리포지토리 기능을 위한 인터페이스**
```Java
interface CustomizedUserRepository {
void someCustomMethod(User user);
}
```
**사용자 정의 리포지토리 기능의 구현**
```Java
class CustomizedUserRepositoryImpl implements CustomizedUserRepository {
    public void someCustomMethod(User user) {
    // 여러분의 커스텀 구현 내용
    }
}

// EntityRepository에 파편화된 인터페이스를 상속해서 사용합니다.
interface UserRepository extends CrudRepository<User, Long>, 
                                    CustomizedUserRepository {
  // Declare query methods here
}
```
프래그먼트 인터페이스에 해당하는 클래스의 이름에서 가장 중요한 부분은 **Impl 접미사**입니다.   
Impl 접미사를 사용함으로써 **스프링은 해당 구현체를 프래그먼트로 인식하게 됩니다.**  
  
**사용자 정의 구현 클래스**  
+ 규칙: 리포지토리 인터페이스 이름 + Impl
+ 스프링 데이터 JPA가 인식해서 스프링 빈으로 등록

**사용자 정의 구현은 기본 구현 및 저장소 측면보다 우선순위가 높습니다.**   
이 순서를 사용하면 기본 저장소 및 측면 메서드를 재정의할 수 있으며 두 조각이 
동일한 메서드 시그니처을 제공하는 경우 모호성을 해결할 수 있습니다.
`Repository fragments`은 단일 리포지토리 인터페이스에서만 사용하도록 제한되지 않습니다. 
여러 저장소에서 조각 인터페이스를 사용할 수 있으므로 여러 저장소에서 
사용자 정의를 재사용할 수 있습니다.  

기본 레포지토리의 save를 오버라이딩 할 수 있습니다.
```Java
interface CustomizedSave<T> {
  <S extends T> S save(S entity);
}
class CustomizedSaveImpl<T> implements CustomizedSave<T> {
    public <S extends T> S save(S entity) {
    // Your custom implementation
    }
}
```  
해당 `Repository fragments`를 재사용할 수 있습니다.
```Java
interface UserRepository extends CrudRepository<User, Long>, CustomizedSave<User> {}
interface PersonRepository extends CrudRepository<Person, Long>, CustomizedSave<Person> {}
```

### Impl 접미사 설정방법
+ 자바 설정
    ```Java
    @EnableJpaRepositories(repositoryImplementationPostfix = "MyPostfix")
    class Configuration { … }
    ```  
    설정시 `CustomizedSaveImpl`을 찾지 않고 `CustomizedSaveMyPostfix`를 찾습니다.  
  

+ XML 설정
    ```XML
    <repositories base-package="study.datajpa.repository" repository-impl-postfix="Impl" />
    ```  

## 기본 저장소 사용자 정의로 사용하기  
+ [Customize the Base Repository](https://docs.spring.io/spring-data/jpa/reference/repositories/custom-implementations.html#repositories.customize-base-repository)    

이전 방식은 모든 리포지토리에 영향을 주려면 각각의 리포지토리 인터페이스를 커스터마이징해야 합니다. 
모든 리포지토리의 동작을 변경하려면, 특정 지속성 기술에 특화된 리포지토리 기본 클래스를 확장하는 구현체를 생성할 수 있습니다. 
이 클래스는 리포지토리 프록시의 사용자 지정된 기본 클래스로 작동합니다.  

```Java
class MyRepositoryImpl<T, ID>
  extends SimpleJpaRepository<T, ID> {

  private final EntityManager entityManager;

  MyRepositoryImpl(JpaEntityInformation entityInformation,
                          EntityManager entityManager) {
    super(entityInformation, entityManager);

    // 새롭게 추가된 메서드에서 사용하기 위해 EntityManager를 유지합니다.
    this.entityManager = entityManager;
  }

  @Transactional
  public <S extends T> S save(S entity) {
    // 구현 내용을 여기에 작성합니다.
  }
}
```  
+ 이 클래스는 슈퍼 클래스의 생성자를 가져야 합니다.   
이 생성자는 스토어별 리포지토리 팩토리 구현에서 사용됩니다. 
리포지토리 기본 클래스에 여러 생성자가 있는 경우, 
`EntityInformation`과 스토어별 인프라스트럭처 객체(`EntityManager` 또는 템플릿 클래스와 같은)를 
인자로 받는 생성자를 오버라이드해야 합니다.

마지막 단계는 Spring Data 인프라스트럭처가 커스터마이징된 리포지토리 기본 클래스를 인식하도록 하는 것입니다. 
설정에서 repositoryBaseClass를 사용하여 이를 수행할 수 있습니다.  

```Java
@Configuration
@EnableJpaRepositories(repositoryBaseClass = MyRepositoryImpl.class)
class ApplicationConfiguration { … }
```  
