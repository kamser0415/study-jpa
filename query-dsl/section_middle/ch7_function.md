# SQL function 호출하기  
SQL function은 JPA와 같이 Dialect에 등록된 내용만 호출할 수 있습니다.
### ANSI 표준 함수
`com.querydsl.core.types.dsl.StringExpression`을 참조하면 됩니다.  
기본적인 표준 함수는 queryDsl이 내장하고 있습니다.
+ **lower()**
    ```Java
    List<String> names = queryFactory.select(member.username)
                    .from(member)
                    .where(member.username.lower().eq("john"))
                    .fetch();
    ```
    ```SQL
    select
        m1_0.username 
    from
        member m1_0 
    where
        lower(m1_0.username)=?
    ```
### MySQL Dialect전용 함수 사용 방법
```Yyaml
spring:
  datasource:
    url: jdbc:h2:tcp:XXXXXXX;MODE=MySQL
            
  jpa:
    hibernate:
      ddl-auto: create
    properties:
      hibernate:
        dialect: org.hibernate.dialect.MySQLDialect
```  
방언을 설정하고 해당 방언 클래스에 내장함수를 확인할 수 있습니다.
+ **group_concat()**
  ```Java
  QMember member = QMember.member;
  List<String> fetch = queryFactory
          .select(Expressions.stringTemplate("function('listagg',{0},',')", member.username))
          .from(member)
          .fetch();
  ```  
