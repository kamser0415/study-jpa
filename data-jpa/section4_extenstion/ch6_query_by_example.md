# Query By Example  
+ [Query By Example 공식문서 설명](https://docs.spring.io/spring-data/jpa/reference/repositories/query-by-example.html)  

### 소개
Query by Example(QBE)는 사용자 친화적인 질의 기술로, 간단한 인터페이스를 가지고 있습니다. 
이는 동적 쿼리 생성을 허용하며 필드 이름을 포함한 **쿼리 작성을 요구하지 않습니다**. 
사실, Query by Example은 특정 저장소(store-specific) 쿼리 언어를 사용하여 
쿼리를 작성할 필요가 전혀 없습니다   

### 적용방법
```Java
public interface QueryByExampleExecutor<T> {} 를 상속합니다.
```

### 구성
+ `Probe`(탐사): 채워진 필드를 가진 도메인 객체의 실제 예시입니다.
+ `ExampleMatcher`(예시 매처): 특정 필드를 어떻게 일치시킬지에 대한 세부 정보를 가지고 있습니다. 이는 여러 Example에서 재사용될 수 있습니다.
+ `Example`(예시): Probe와 ExampleMatcher로 구성된 예시입니다. 이는 쿼리를 생성하는 데 사용됩니다.
+ `FetchableFluentQuery`(검색 가능한 플루언트 쿼리): Fluent API를 제공하여 Example에서 파생된 쿼리를 더 많이 사용자 정의할 수 있게 합니다. Fluent API를 사용하면 쿼리의 정렬, 프로젝션 및 결과 처리를 지정할 수 있습니다.
  
### 적합한 사례
+ 정적 또는 동적 제약 조건을 사용해야할 때
+ 도메인 객체를 자주 재구성해야 하는 경우에 기존 쿼리를 깨뜨리지 않고 사용할 수 있습니다.
+ 기본 데이터 저장소 API와 독립적으로 작업하는 경우  

### 장점
+ 동적 쿼리를 편리하게 처리
+ 도메인 객체를 그대로 사용
+ 데이터 저장소를 RDB에서 NOSQL로 변경해도 코드 변경이 없게 추상화 되어 있음
+ 스프링 데이터 `JPA JpaRepository` 인터페이스에 이미 포함
  
### 제약사항  
+ 조인은 가능하지만 내부 조인(INNER JOIN)만 가능함 외부 조인(LEFT JOIN) 안됨
+ `firstname = ?0 or (firstname = ?1 and lastname = ?2)`와   
    같은 중첩된 또는 그룹화된 속성 제약 조건을 지원하지 않습니다.
+ 문자열에 대해서는 `starts`/`contains`/`ends`/`regex` 일치를 지원하며,   
  다른 속성은 정확한 매칭( `=` )만 지원  
+ 기본적으로 null 값을 가진 필드는 무시되며, 문자열은 해당 저장소(store)의 기본값을 사용하여 일치시킵니다.  

> Query by Example의 기준에 속성을 포함하는 것은 nullability(널 여부)에 기반합니다. 
> 기본 타입(primitive types)을 사용하는 속성(int, double 등)은 
> ExampleMatcher가 속성 경로를 무시하지 않는 한 항상 포함됩니다.  
  
### 예제
+ `Example`만드는 예시:  
    쿼리를 생성하는데 사용합니다.
    ```Java
    // 도메인 객체의 새 인스턴스를 생성하세요.
    Person person = new Person();                         
    // 쿼리할 속성을 설정하세요.
    person.setFirstname("Dave");                          
    // 그리고 Example을 생성하세요.
    Example<Person> example = Example.of(person);  
    ```
+ `ExampleMatcher` 만드는 예시:
  `ExampleMatcher`을 사용하여 문자열 일치, Null 처리 및 속성별 설정에 대한 고유한 기본값을 지정할 수 있습니다.
    ```Java
    //모든 값이 일치할 것으로 예상하려면 ExampleMatcher을 만드세요. 추가 구성 없이도 이 단계에서 사용할 수 있습니다.
    ExampleMatcher matcher = ExampleMatcher.matching()     
      .withIgnorePaths("lastname") // lastname 속성 경로를 무시                         
      .withIncludeNullValues() // lastname는 무시하고 null 값을 포함하세요.                             
      .withStringMatcher(StringMatcher.ENDING); // 접미사 문자열 일치를 수행           
    
    Example<Person> example = Example.of(person, matcher);  
    ```  
  
### 테스트 코드  
+ Equals 조회
```Java
@DisplayName("이름이 둘리인 회원 조회하기")
@Test
void e1(){
    //given
    Member hoit = new Member("둘리", 15);
    Member gugu = new Member("비둘기", 3);
    memberRepository.saveAll(List.of(hoit, gugu));

    //when
    Member exam = new Member();
    exam.setUsername("둘리");

    ExampleMatcher matcher = ExampleMatcher.matching()
            .withIgnorePaths("age")
            .withIgnoreNullValues();

    Example<Member> memberExample = Example.of(exam, matcher);
    List<Member> findMember = memberRepository.findAll(memberExample);

    //then
    Assertions.assertThat(findMember).hasSize(1);
}
```
+ SQL
```SQL
select * from member where username='둘리';
```   
