# @QueryProjection
생성자 + `@QueryProjection`  
**@QueryProjection**는 QueryDsl이 Q파일을 생성합니다.  
예제:
```Java
public class AnnotationConst {

    private final String name;
    
    @QueryProjection
    public AnnotationConst(String name) {
        this.name = name;
    }
}
```  
+ 테스트 코드
```Java
//when
List<AnnotationConst> findMember = queryFactory.select(new QAnnotationConst(member.username))
        .from(member)
        .fetch();
```
> 참고 :  
> 매개변수 명과 엔티티 필드명이 달라도 파입이 일치하면 초기화가 됩니다.  
> DTO를 Service에서도 사용한다면 Service에 QueryDSL 의존이 생깁니다.

**엔티티 생성자 위에 붙어있는 경우**
```Java
@Entity
public class Member {
  @QueryProjection
  public Member(String userName){
    this.name = userName
  }
}
```  
다음과 같이 호출해서 사용할 수 있습니다. 그리고 반환된 값은 엔티티가 됩니다.
```Java
@DisplayName("엔티티 생성자 위에 붙을 경우")
@Test
void entity2(){
    //given
    em.persist(new Member("둘리",15));
    em.persist(new Member("또치",20));
    em.flush();
    em.clear();
    QMember member = QMember.member;
    //when
    List<Member> findMember = queryFactory.select(QMember.create(member.username))
            .from(member)
            .fetch();
    //then
    Assertions.assertThat(findMember)
            .allMatch( entity-> Member.class.isAssignableFrom(entity.getClass()));
}
```
