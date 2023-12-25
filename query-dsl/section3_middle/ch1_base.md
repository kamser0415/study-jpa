# 프로젝션과 결과 반환 - 기본
프로젝션: select 할 대상을 지정합니다.  
  

### 프로젝션 대상이 하나 일 경우  
```Java
@DisplayName("프로젝션 대상이 값 타입이나 기본 타입일 경우")
@Test
void p1(){
    //given
    String username = "둘리";
    int userAge = 10;
    String city = "서울시";
    String zipcode = "15135";

    Member memberEntity = new Member(username, userAge);
    memberEntity.setAddress(new Address(zipcode, city));
    em.persist(memberEntity);

    em.flush();
    em.clear();
    //when
    List<Address> findMember = qFactory.select(member.address)
            .from(member)
            .where(member.age.eq(userAge))
            .fetch();
    //then
    Assertions.assertThat(findMember).hasSize(1);
    Assertions.assertThat(findMember).extracting("city","zipcode")
            .contains(Tuple.tuple(city, zipcode));
}
```  
+ 기본 타입이나 값타입일 경우에는 타입을 지정할 수 있습니다.  
  
