# 정렬과 페이징

## 정렬
**기본 문법**
```Java
queryFactory.select(customer.firstName, customer.lastName)
    .from(customer)
    .orderBy(customer.lastName.asc(), customer.firstName.asc())
    .fetch();
```  
정렬 기준 칼럼이 인덱스가 없거나 드라이빙 테이블 칼럼이 아닐 경우 
메모리에서 정렬하지만, 정렬 대상이 많을 경우 디스크에 임시 저장하여 병합 정렬하기 때문에 
속도 저하가 발생할 수 있습니다. 
#### 테스트코드
  ```Java
  @DisplayName("정렬해보기")
  @Test
  void o1(){
      /**
       * 나이가 40살 이상
       * 나이 내림차순, 이름은 오름차순 NULL이 우선순위
       */
      //given
      em.persist(new Member(null,100));
      em.persist(new Member("member5",100));
      em.persist(new Member("member6",100));
  
      //when
      List<Member> findMember = queryFactory
              .select(member)
              .from(member)
              .where(member.age.gt(40))
              .orderBy(member.age.desc(), member.username.asc().nullsFirst())
              .fetch();
  
      //then
      Assertions.assertThat(findMember).hasSize(3);
      Assertions.assertThat(findMember).extracting("username","age")
              .containsExactly(
                      tuple(null,100),
                      tuple("member5",100),
                      tuple("member6",100)
              );
  }
  ```
+ `desc()` , `asc()` : 일반 정렬
+ `nullsLast()` , `nullsFirst()` : null 데이터 순서 부여

## 페이징
#### 조회 건수 제한 문법  
+ `.offset(startRecordIdx)`
+ `.limit(readRowAmount)`  
  
**테스트 코드**
```Java
@DisplayName("가장 나이가 많은 사람 2명을 찾으세요.")
@Test
void p1(){
    List<Member> findMember = queryFactory
            .select(member)
            .from(member)
            .orderBy(member.age.desc())
            .offset(0) // zero index
            .limit(2)
            .fetch();

    //then
    Assertions.assertThat(findMember).hasSize(2);
    Assertions.assertThat(findMember).extracting("username","age")
            .containsExactly(
                    tuple("member4",40),
                    tuple("member3",30)
            );
}
```