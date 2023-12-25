# QueryDSL 환경 설정 방법
### 스프링 부트 3.0 이상
```Gradle
plugins {
   id 'java'
   id 'org.springframework.boot' version '3.2.0'
   id 'io.spring.dependency-management' version '1.1.4'
}


group = 'study'
version = '0.0.1-SNAPSHOT'
sourceCompatibility = '17'


configurations {
   compileOnly {
       extendsFrom annotationProcessor
   }
}


repositories {
   mavenCentral()
}


dependencies {
   implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
   implementation 'org.springframework.boot:spring-boot-starter-web'
   implementation 'com.github.gavlyukovskiy:p6spy-spring-boot-starter:1.9.0'
   compileOnly 'org.projectlombok:lombok'
   runtimeOnly 'com.h2database:h2'
   annotationProcessor 'org.projectlombok:lombok'
   testImplementation 'org.springframework.boot:spring-boot-starter-test'


   //test 롬복 사용
   testCompileOnly 'org.projectlombok:lombok'
   testAnnotationProcessor 'org.projectlombok:lombok'


   //Querydsl 추가
   implementation 'com.querydsl:querydsl-jpa:5.0.0:jakarta'
   annotationProcessor "com.querydsl:querydsl-apt:${dependencyManagement.importedProperties['querydsl.version']}:jakarta"
   annotationProcessor "jakarta.annotation:jakarta.annotation-api"
   annotationProcessor "jakarta.persistence:jakarta.persistence-api"
}


tasks.named('test') {
   useJUnitPlatform()
}


clean {
   delete file('src/main/generated')
}
```

### 스프링부트 3.0 미만
```Gradle
plugins {
	id 'org.springframework.boot' version '2.6.5'
	id 'io.spring.dependency-management' version '1.0.11.RELEASE'
	id 'java'
}

group = 'com.example'
version = '0.0.1-SNAPSHOT'
sourceCompatibility = '11'

ext["hibernate.version"] = "5.6.5.Final"

configurations {
	compileOnly {
		extendsFrom annotationProcessor
	}
}

repositories {
	mavenCentral()
}

dependencies {
	implementation 'org.springframework.boot:spring-boot-starter-thymeleaf'
	implementation 'org.springframework.boot:spring-boot-starter-web'

	//JdbcTemplate 추가
	//implementation 'org.springframework.boot:spring-boot-starter-jdbc'
	//MyBatis 추가
	implementation 'org.mybatis.spring.boot:mybatis-spring-boot-starter:2.2.0'
	//JPA, 스프링 데이터 JPA 추가
	implementation 'org.springframework.boot:spring-boot-starter-data-jpa'

	//Querydsl 추가
	implementation 'com.querydsl:querydsl-jpa'
	annotationProcessor "com.querydsl:querydsl-apt:${dependencyManagement.importedProperties['querydsl.version']}:jpa"
	annotationProcessor "jakarta.annotation:jakarta.annotation-api"
	annotationProcessor "jakarta.persistence:jakarta.persistence-api"

	//H2 데이터베이스 추가
	runtimeOnly 'com.h2database:h2'
	compileOnly 'org.projectlombok:lombok'
	annotationProcessor 'org.projectlombok:lombok'
	testImplementation 'org.springframework.boot:spring-boot-starter-test'

	//테스트에서 lombok 사용
	testCompileOnly 'org.projectlombok:lombok'
	testAnnotationProcessor 'org.projectlombok:lombok'
}

tasks.named('test') {
	useJUnitPlatform()
}

//Querydsl 추가, 자동 생성된 Q클래스 gradle clean으로 제거
clean {
	delete file('src/main/generated')
}
```  

### 기본 문법
```Java
@Entity
public class Customer {
    private String firstName;
    private String lastName;

    public String getFirstName() {
        return firstName;
    }

    public String getLastName() {
        return lastName;
    }

    public void setFirstName(String fn) {
        firstName = fn;
    }

    public void setLastName(String ln) {
        lastName = ln;
    }
}
```
QueryDSL는 정적으로 사용할 수 있는 엔티티와 동일한 클래스를 만듭니다.  
```Java
// 기본 인스턴스 사용
QCustomer custmer = QCustomer.customer;
// 해당 엔티티에 대한 고유 유의어를 지정할 수 있습니다.
QCustomer custmer = new QCustomer("myCustomer");
```
템플릿을 만드는 JPAQueryFactory에 EntityManager를 넣습니다.
```Java
JPAQueryFactory queryFactory = new JPAQueryFactory(em);
```  
+ synax
```Java
QCustomer custmer = new QCustomer("myCustomer");
 Member findMember = queryFactory 
            .select(custmer)
            .from(custmer)
            .where(custmer.username.eq("custmer1"))//파라미터 바인딩 처리 
            .fetchOne();
```