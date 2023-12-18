# 라이브러리 살펴보기    
### gradle 설정
```Gradle
dependencies {
    // jpa orm
    implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
    // web
    implementation 'org.springframework.boot:spring-boot-starter-web'
    // database
    runtimeOnly 'com.h2database:h2'
    runtimeOnly 'com.mysql:mysql-connector-j'
    // lombok
    compileOnly 'org.projectlombok:lombok'
    annotationProcessor 'org.projectlombok:lombok'
    // test
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
}
```

### gradle 의존관계 살펴보기   
해당 프로젝트 디렉토리에서 아래 명령어를 실행합니다. 
```Bash
pwd # 현재 위치 
/c/kamser/inflearn/jpa/datajpa # 현재 프로젝트 위치입니다.
```
```Bash
./gradlew dependencies --configuration compileClasspath
```   
```Bash
+--- org.springframework.boot:spring-boot-starter-data-jpa -> 3.2.0
|    +--- org.springframework.boot:spring-boot-starter-aop:3.2.0
|    +--- org.springframework.boot:spring-boot-starter-jdbc:3.2.0
|    |    +--- com.zaxxer:HikariCP:5.0.1
|    |    \--- org.springframework:spring-jdbc:6.1.1
|    +--- org.hibernate.orm:hibernate-core:6.3.1.Final
|    +--- org.springframework.data:spring-data-jpa:3.2.0
// 이하 생략
```  
간단 하게 의존성 관계를 정리해보면 다음과 같습니다.   
```Bash
+--- spring-boot-starter-web
|   +--- spring-boot-starter-tomcat: 톰캣 (웹서버) 
|   +--- spring-webmvc: 스프링 웹 MVC
+--- spring-boot-starter-data-jpa 
|   +--- spring-boot-starter-aop 
|   +--- spring-boot-starter-jdbc
|   \--- HikariCP 커넥션 풀 (부트 2.0 기본) 
|   +--- hibernate + JPA: 하이버네이트 + JPA 
|   +--- spring-data-jpa: 스프링 데이터 JPA
+--- spring-boot-starter(공통): 스프링 부트 + 스프링 코어 + 로깅 
|   +--- spring-boot
|   \--- spring-core
|   +--- spring-boot-starter-logging 
|   \--- logback, slf4j
```

간단하게 의존관계를 살펴보면 스프링 부트의 기본 설정으로 어떤 것을 가져오는지 확인할 수 있습니다. 
  
스프링 부트는 web기술을 사용할 때 톰캣을 사용하고, 톰캣 라이브러리를 jar로 들고오는게 아니라 
스프링 부트에서 기본 값을 설정한 톰캣을 가져오고 스프링 web Mvc를 가져오는 것을 확인할 수 있습니다.  
  
스프링 Data JPA를 사용하면 스프링 부트의 AOP 기술, 그리고 JDBC와 커넥션 풀은 히카리CP를 기본으로 설정하는 것을 확인할 수 있습니다. 
하이버네이트를 기본으로 가져오고, 스프링 data jpa라는 스프링에서 만든 기술도 가져옵니다.
