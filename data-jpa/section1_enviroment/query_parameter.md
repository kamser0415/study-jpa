### 쿼리 파리미터 로그 남기기
1. `org.hibernate.type = trace`  
   [공식문서 Docs](https://docs.jboss.org/hibernate/orm/6.4/userguide/html_single/Hibernate_User_Guide.html#best-practices-logging)
2. 외부라이브러리 : `p6spy`
    ```Gradle
    //gradle
    dependencies {
        //스프링부트 3.0 이상
        implementation 'com.github.gavlyukovskiy:p6spy-spring-boot-starter:1.9.0'
        //스프링부트 3.0 이하
        implementation 'com.github.gavlyukovskiy:p6spy-spring-boot-starter:1.5.7'
    }
    ```  
> 참고 :  
> 쿼리 파라미터를 로그로 남기는 외부 라이브러리는 시스템 자원을 사용합니다.  
> 개발 단계에서 사용하고 운영 시스템에서는 성능 테스트를 필수로 확인하고 사용합니다.

**외부 라이브러리 커스텀 방법 [문서](https://github.com/gavlyukovskiy/spring-boot-data-source-decorator)**