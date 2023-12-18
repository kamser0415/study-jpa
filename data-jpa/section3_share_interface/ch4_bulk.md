# 벌크성 수정 쿼리  
#### JPA를 사용한 벌크성 수정 쿼리  
```Java
public int bulkAgePlus(int age) { 
    int resultCount = em.createQuery("update Member m set m.age = m.age + where m.age >= :age")
                        .setParameter("age", age) 
                        .executeUpdate();
    return resultCount; 
}
```  
#### Data JPA를 사용한 벌크성 수정 쿼리
`@Modifying` 어노테이션을 사용하여 수정할 수 있습니다.
```Java
@Modifying
@Query("update Member m set m.age = m.age + 1 where m.age >= :age") 
int bulkAgePlus(@Param("age") int age)
```  
위와 같이 하면 해당 메서드에 어노테이트된 쿼리가 **선택 쿼리**가 아닌 **벌크성 업데이트 쿼리**로 실행됩니다. 
수정 쿼리 실행 후에 EntityManager에는 동기화가 안된 엔티티가 포함될 수 있으므로, 
Data JPA는 자동으로 영속성 컨텍스르를 비우지 않습니다.  

왜냐하면 이렇게 하면 **_EntityManager에 아직 플러시되지 않은 변경 사항이 모두 삭제되기 때문_** 입니다.  

EntityManager를 자동으로 비우기를 원한다면, 
`@Modifying` 어노테이션의 `clearAutomatically` 속성을 `true`로 설정할 수 있습니다.

> 참고:  
> `@Modifying` 어노테이션은 `@Query` 어노테이션과 함께 사용할 때에만 관련이 있습니다. 
> 쿼리 이름 메서드 또는 사용자 지정 메서드는 이 어노테이션을 요구하지 않습니다  
 
#### 쿼리 이름 메소드와 @Query 쿼리의 차이점  
실무에서 레코드에서 데이터를 삭제하는 일이 극히 드물거라 생각합니다. 
그래도 같은 쿼리를 실행하지만 동작 방식이 차이가 있습니다.  
[Derived Delete Queries 공식문서](https://docs.spring.io/spring-data/jpa/reference/jpa/query-methods.html#jpa.modifying-queries.derived-delete)
```Java
@DisplayName("쿼리 이름 메소드로 데이터를 삭제할 경우")
@Test
void d1(){
    //given
    Member member1 = new Member("둘리아빠", 1);
    Member member2 = new Member("둘리", 2);
    memberRepository.saveAll(List.of(member1, member2));

    //when
    memberRepository.deleteMemberByUsername("둘리");
    Optional<Member> deleteMember = memberRepository.findById(member2.getId());

    //then
    Assertions.assertThat(deleteMember.isEmpty()).isTrue();

}
@DisplayName("@Query로 데이터를 삭제할 경우")
@Test
void d2(){
    //given
    Member member1 = new Member("둘리아빠", 1);
    Member member2 = new Member("둘리", 2);
    memberRepository.saveAll(List.of(member1, member2));

    //when
    memberRepository.deleteMemberInBulksByUsername("둘리");
    Optional<Member> deleteMember = memberRepository.findById(member2.getId());

    //then
    Assertions.assertThat(deleteMember.isEmpty()).isFalse();
    Assertions.assertThat(deleteMember.orElseGet(Member::new)).isSameAs(member2);

}
```
이름 메소드로 할 경우에는 메모리에 있는 해당 데이터도 같이 삭제가 됩니다. 
하지만 `@Query`로 삭제한 경우에는 동기화를 하지 않기 때문에 메모리에 데이터가 남아있습니다.  
  
> 참고:   
> 벌크 연산은 영속성 컨텍스트를 무시하고 실행하기 때문에, 영속성 컨텍스트에 있는 엔티티의 상태와 DB에
> 엔티티 상태가 달라질 수 있습니다.  
> 
> **권장하는 방안**
> 1. 영속성 컨텍스트에 엔티티가 없는 상태에서 벌크 연산을 먼저 실행한다.
> 2. 부득이하게 영속성 컨텍스트에 엔티티가 있으면 벌크 연산 직후 영속성 컨텍스트를 초기화 한다