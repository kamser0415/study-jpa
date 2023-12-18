# 스프링 데이터 JPA 페이징과 정렬  
+ [페이징 및 정렬 공식문서](https://docs.spring.io/spring-data/jpa/reference/repositories/query-methods-details.html#repositories.special-parameters)  

페이징, 정렬 및 제한을 동적으로 적용하기 위해 Pageable, Sort 및 Limit과 같은 특정한 유형을 인식합니다.  
```Java
Page<User> findByLastname(String lastname, Pageable pageable);

Slice<User> findByLastname(String lastname, Pageable pageable);

List<User> findByLastname(String lastname, Sort sort);

List<User> findByLastname(String lastname, Sort sort, Limit limit);

List<User> findByLastname(String lastname, Pageable pageable);
```  
### 페이징과 정렬 파라미터  
+ `org.springframework.data.domain.Sort` : 정렬 기능
+ `org.springframework.data.domain.Pageable` : 페이징 기능 (내부에 Sort 포함)
### 특별한 반환 타입
+ `org.springframework.data.domain.Page` : 추가 count 쿼리 결과를 포함하는 페이징
+ `org.springframework.data.domain.Slice` : 추가 count 쿼리 없이 다음 페이지만 확인 가능(내부적으
로 limit + 1조회)
+ `List (자바 컬렉션)`: 추가 count 쿼리 없이 결과만 반환  

**공식문서 설명**
> Page 객체는 전체 요소 수와 사용 가능한 페이지 수에 대해 알고 있습니다.   
인프라는 전체 수를 계산하기 위해 count 쿼리를 트리거함으로써 이를 달성합니다.   
이는 비용이 많이 들 수 있으므로(사용된 스토어에 따라 다름), 
대신 Slice를 반환할 수 있습니다. 
Slice는 다음 Slice가 있는지 여부에 대해서만 알고 있으며, 
더 큰 결과 세트를 탐색할 때 충분할 수 있습니다.

> 정렬 옵션은 또한 Pageable 인스턴스를 통해 처리됩니다.   
정렬만 필요한 경우에는 메서드에 org.springframework.data.domain.Sort 매개변수를 
추가하면 됩니다. 또한 List를 반환하는 것도 가능합니다. 
이 경우 실제 Page 인스턴스를 구성하는 데 필요한 추가 메타데이터가 
생성되지 않습니다(따라서 필요한 추가 count 쿼리가 발행되지 않음).
대신, 쿼리를 엔티티의 주어진 범위만 조회하도록 제한합니다.

### 잘못된 파라미터 형식
다음의 잘못된 매개변수 조합 목록을 참고해주세요.  

| Parameters         | Example                             | Reason                         |
|--------------------|-------------------------------------|--------------------------------|
| Pageable and Sort  | findBy…(Pageable page, Sort sort)   | Pageable는 이미 Sort를 포함하고 있습니다.  |
| Pageable and Limit | findBy…(Pageable page, Limit limit) | Pageable은 이미 Limit를 포함하고 있습니다. |
  
### 사용 예제 및 테스트 코드 작성  
회원 이름으로 검색하고 페이징해서 가져오는 예제를 만들어보겠습니다.  

#### Page 테스트
```Java
@TestFactory
Collection<DynamicTest> pagingDynamicTest() {
    //given
    Member member1 = new Member("둘리아빠", 1);
    Member member2 = new Member("둘리엄마", 2);
    Member member3 = new Member("삼촌둘리", 3);
    Member member4 = new Member("로보트둘리", 4);
    Member member5 = new Member("앞둘리뒤", 5);
    Member member6 = new Member("뚫리", 6);

    memberRepository.saveAll(List.of(member1,member2,member3,member4,member5,member6));

    PageRequest page = PageRequest.of(0, 2, Sort.by(Sort.Direction.DESC, "age"));

    final String searchKeyword = "둘리";
    return Arrays.asList(
        dynamicTest("'둘리' 검색후 첫번째 페이지에서 이전 버튼을 누를 경우",()-> {
            PageRequest previous = page.previous();
            Page<Member> membersWithPaging = memberRepository.findMembersWithPaging(searchKeyword, previous);
            List<Member> content = membersWithPaging.getContent();
            Assertions.assertThat(content).extracting("username","age")
                    .containsExactly(
                            tuple("앞둘리뒤",5),
                            tuple("로보트둘리",4)
                    );
        }),
        dynamicTest("'둘리' 검색후 두번째 페이지로 이동할 때",()-> {
            PageRequest next = page.next();
            Page<Member> membersWithPaging = memberRepository.findMembersWithPaging(searchKeyword, next);
            List<Member> content = membersWithPaging.getContent();
            Assertions.assertThat(content).contains(member2,member3);
        }),
        dynamicTest("'둘리' 검색후 세번째 페이지로 이동할 때",()-> {
            PageRequest current = page.next();
            PageRequest next = current.next();

            Page<Member> membersWithPaging = memberRepository.findMembersWithPaging(searchKeyword, next);
            List<Member> content = membersWithPaging.getContent();
            Assertions.assertThat(content).contains(member1);
        }),
        dynamicTest("'둘리' 검색후 마지막 페이지에서 다음 버튼을 누를 때",()-> {
            PageRequest current = page.next();
            PageRequest next = current.next();
            PageRequest last = next.next();

            Page<Member> membersWithPaging = memberRepository.findMembersWithPaging(searchKeyword, last);
            List<Member> content = membersWithPaging.getContent();
            Assertions.assertThat(content).hasSize(0);
        })
    );
}
```  

테스트를 해보니 `PageRequest`의 단점이 있습니다.  
`previous()`는 방어 코드가 있지만 `next()`는 방어코드가 없습니다.
```Java
@Override
public PageRequest previous() {
    return getPageNumber() == 0 ? this : new PageRequest(getPageNumber() - 1, getPageSize(), getSort());
}
@Override
public PageRequest next() {
    return new PageRequest(getPageNumber() + 1, getPageSize(), getSort());
}
```
마지막 테스트 코드를 보면 `Assertions.assertThat(content).hasSize(0)`이 되는것을 확인할 수 있습니다.  

`Slice.hasNext()` 와 `Slice.isLast()` 를 사용하면 됩니다.
+ 테스트 코드
```Java
@DisplayName("마지막 페이지에서 hasNext를 호출하면 false가 반환된다.")
@Test
void p1(){
    //given
    Member member1 = new Member("둘리아빠", 1);

    memberRepository.saveAll(List.of(member1));
    //마지막 페이지
    PageRequest page = PageRequest.of(0, 2, Sort.by(Sort.Direction.DESC, "age"));

    //when
    Page<Member> pagingMember = memberRepository.findMembersWithPaging("둘리", page);
    boolean hasNext = pagingMember.hasNext();
    boolean isLast = pagingMember.isLast();

    //then
    Assertions.assertThat(hasNext).isFalse();
    Assertions.assertThat(isLast).isTrue();
}
```  

#### Slice 테스트
공식문서에 보면 `Page`와 다르게 `Slice`는 `Pageable.getPageSize()+1` 을 가져온다고 작성되어있습니다.  
**테스트 코드**
```Java
@DisplayName("Slice는 Page와 다르게 +1 개를 더 조회합니다.")
@Test
void p2(){
    //given
    Member member1 = new Member("둘리아빠", 1);
    Member member2 = new Member("둘리엄마", 2);

    memberRepository.saveAll(List.of(member1,member2));
    //마지막 페이지
    PageRequest page = PageRequest.of(0, 2, Sort.by(Sort.Direction.DESC, "age"));

    //when
    Slice<Member> sliceMember = memberRepository.findMembersWithPaging("둘리", page);
    Page<Member> pagingMember = memberRepository.findMembersWithPaging("둘리", page);
    int sliceSize = sliceMember.getSize();
    int pagingSize = pagingMember.getSize();

    //then
    Assertions.assertThat(sliceSize).isEqualTo(3); 
    Assertions.assertThat(pagingSize).isEqualTo(2);
}
```

#### Page의 count 쿼리 분리하기  
전체 페이지의 수를 확인하기 위해 `count` 쿼리를 실행할 때 불필요한 조인없이 최적화를 할 수 있습니다.
```Java
@Query(value = "select m from Member as m left join m.team as t where m.username like %:username% escape '\\'",
           countQuery = "select count(m) from Member m")
    Page<Member> findMembersWithCustomPaging(@Param("username") String name,Pageable pageable);
```
```SQL
select count(m1_0.member_id) 
from member m1_0
```  

#### 페이지 기능과 DTO 사용하기  
Page.map()을 통해서 반환된 객체를 DTO로 변환할 수 있습니다.
```Java  
Page<MemberDto> dtoPage = page.map(m -> new MemberDto());
```
  
### 정리  
| Method         | Amount of Data Fetched                             | Query Structure                                                      | Constraints                                                                           |
|----------------|----------------------------------------------------|----------------------------------------------------------------------|---------------------------------------------------------------------------------------|
| List\<T>       | 모든 결과                                              | 단일 쿼리                                                                | 쿼리 결과가 모든 메모리를 소진할 수 있으며, 모든 데이터를 가져오는 것이 시간이 소모적일 수 있습니다.                            |
| Streamable\<T> | 모든 결과                                              | 단일 쿼리                                                                | 쿼리 결과가 모든 메모리를 소진할 수 있으며, 모든 데이터를 가져오는 것이 시간이 소모적일 수 있습니다.                            |
| Stream\<T>     | 청크로 나뉘어짐 (하나씩 또는 일괄 처리)                            | 일반적으로 커서를 사용하는 단일 쿼리                                                 | 리소스 누수를 방지하기 위해 스트림을 닫아야 합니다.                                                         |
| Flux\<T>       | 청크로 나뉘어짐 (하나씩 또는 일괄 처리)                            | 일반적으로 커서를 사용하는 단일 쿼리                                                 | 저장 모듈로부터 반응형 인프라가 필요합니다.                                                              |
| Slice\<T>      | Pageable.getPageSize() + 1 at Pageable.getOffset() | Pageable.getOffset()에서 시작하는 여러 개의 쿼리, 제한이 적용됨                        | 더 많은 데이터를 가져올 수 있는지 여부에 대한 정보를 제공하며, 큰 오프셋에 대해 오프셋 기반 쿼리는 전체 결과를 생성하는 데 비효율적일 수 있습니다. |
| Page\<T>       | Pageable.getPageSize() at Pageable.getOffset()     | Pageable.getOffset()에서 시작하는 여러 개의 쿼리, 제한이 적용됨; COUNT(…) 쿼리가 필요할 수 있음 | 큰 오프셋에 대해 오프셋 기반 쿼리는 전체 결과를 생성하는 데 비효율적일 수 있으며, COUNT(…) 쿼리는 비용이 많이 들 수 있습니다.         |

### Hibernate 6 left join 최적화 설명
스프링 부트 3 이상을 사용하면 하이버네이트 6이 적용된다.
이 경우 하이버네이트 6에서 의미없는 left join을 최적화 해버린다. 따라서
다음을 실행하면 SQL이 LEFT JOIN을 하지 않는 것으로 보인다.
```java
@Query(value = "select m from Member m left join m.team t")
Page<Member> findByAge(int age, Pageable pageable); 
```
```SQL
select m.* from member m
실행 결과 - SQL
```
**하이버네이트 6은 이런 경우 왜 left join을 제거하는 최적화를 할까?**  
실행한 JPQL을 보면 left join을 사용하고 있다.  
`select m from Member m left join m.team t` JPQL을 살펴보면  
Member 와 Team 을 조인을 하지만 사실 이 쿼리를 Team 을 전혀 사용하지 않습니다.   
select 절이나, where 절에서 사용하지 않는 다는 뜻이다. 그렇다면 이 JPQL은 사실상 다음과 같다.  
`select m from Member m`는 left join 이기 때문에 왼쪽에 있는 member 자체를 다 조회한다는 뜻이 된다.
만약 select 나, where 에 team 의 조건이 들어간다면 정상적인 join 문이 보인다.
JPA는 이 경우 최적화를 해서 join 없이 해당 내용만으로 SQL을 만든다.
여기서 만약 Member 와 Team 을 하나의 SQL로 한번에 조회하고 싶으시다면 JPA가 제공하는 fetch join 을 사용
해야 합니다
`select m from Member m left join fetch m.team t`
이 경우에도 SQL에서 join문은 정상 수행됩니다.
  
