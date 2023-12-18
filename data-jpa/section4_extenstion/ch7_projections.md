# Projections
+ [Projections 공식문서](https://docs.spring.io/spring-data/jpa/reference/repositories/projections.html)  

`Spring Data JPA`의 쿼리 메소드는 엔티티 집합이나 집계 결과를 반환홥니다.
`Spring Data`를 사용하면 전용 반환 유형을 모델링하여 관리되는 집계의 부분 보기를 보다 선택적으로 검색할 수 있습니다.  
  
```Java
class Person {

  @Id UUID id;
  String firstname, lastname;
  Address address;

  static class Address {
    String zipCode, city, street;
  }
}

interface PersonRepository extends Repository<Person, UUID> {
  Collection<Person> findByLastname(String lastname);
}
```  
회원의 이름만 검색하고 싶을때 사용할 수 있는 기능입니다.  

## 인터페이스 기반 프로젝션 
가장 간단한 방법으로 읽을 속성에 대한 접근자 메서드를 가진 인터페이스를 생성합니다.  

```Java
interface NamesOnly {
  String getFirstname();
  String getLastname();
}
```
#### 인터페이스 기반 프로젝션을 사용하는 레포지토리
```Java
interface PersonRepository extends Repository<Person, UUID> {
  Collection<NamesOnly> findByLastname(String lastname);
}
```  
쿼리 실행 엔진은 각 요소에 대해 런타임에서 해당 인터페이스의 프록시 인스턴스를 생성하고, 
노출된 메서드로의 호출을 대상 객체로 전달합니다.  

#### 테스트 코드
+ 인터페이스 프로젝션
```Java
public interface MemberSimpleInfo {
    String getUsername();
    Integer getAge();
}
```
+ 레포지토리 메서드 생성
```Java
List<MemberSimpleInfo> findMembersByUsernameStartingWith(String name);
```  
#### 테스트 코드
```Java
@DisplayName("둘리로 시작하는 사람들의 이름과 나이만 간단하게 조회하기")
@Test
void pro1(){
    //given
    Member member1 = new Member("둘리동생", 20);
    Member member2 = new Member("희동이", 5);
    Member member3 = new Member("둘리아빠", 40);
    memberRepository.saveAllAndFlush(List.of(member1, member2, member3));
    em.clear();
    //when
    List<MemberSimpleInfo> findMember = memberRepository.findMembersByUsernameStartingWith("둘리");

    //then
    Assertions.assertThat(findMember).hasSize(2);
    Assertions.assertThat(findMember).extracting("username","age")
            .containsExactlyInAnyOrder(
                    tuple("둘리동생",20),
                    tuple("둘리아빠",40));
}
```
```SQL
select 
    m1_0.username,
    m1_0.age 
from member m1_0 
where m1_0.username like '둘리%' escape '\';
```  
SQL에서도 select절에서 username과 age만 조회(Projection)하는 것을 확인할 수 있습니다.  
  
## Open Projections  
@Value를 사용한 프로젝션 인터페이스는 오픈 프로젝션이라고 합니다.   

`@Value`어노테이션은 애그리게이트 루트(주로 엔티티)를 나타내는 target 변수를 사용하여 새로운 값을 계산합니다.
```Java
interface NamesOnly {
  @Value("#{target.firstname + ' ' + target.lastname}")
  String getFullName();
}
```
#### 주의사항
+ `@Value` 어노테이션에서 사용되는 표현식은 너무 복잡하지 않아야 합니다.   
+ 이 경우 Spring Data는 쿼리 실행 최적화를 적용할 수 없습니다.     
  왜냐하면 SpEL(스프링 표현 언어) 표현식에서 에그리게이트 루트(주로 엔티티)의 모든 속성을
  사용할 수 있기 때문입니다.   
+ 가능하다면 문자열 변수로 프로그래밍하는 것을 피해야합니다.  

### 디폴트 메서드 사용
매우 간단한 경우 사용
```Java
public interface MemberSimpleInfo {
    String getUsername();
    Integer getAge();

    default String getSimpleInfo(){
        return String.format("이름: %s 나이: %d", getUsername(), getAge());
    }
}
```  
#### 테스트 코드
```Java
@DisplayName("디폴트 메서드로 테스트해보기")
@Test
void pro2(){
    //given
    Member member1 = new Member("둘리동생", 20);
    Member member2 = new Member("희동이", 5);
    Member member3 = new Member("둘리아빠", 40);
    memberRepository.saveAllAndFlush(List.of(member1, member2, member3));
    em.clear();
    
    //when
    List<MemberSimpleInfo> findMember = memberRepository.findMembersByUsernameStartingWith("둘리");

    //then
    Assertions.assertThat(findMember).extracting("simpleInfo")
            .containsExactlyInAnyOrder(
                    String.format("이름: %s 나이: %d", member1.getUsername(), member1.getAge()),
                    String.format("이름: %s 나이: %d", member3.getUsername(), member3.getAge()));
}
```  
### 빈 오브젝트 사용하기
더 복잡한 표현식의 경우 Spring Bean을 사용하고 표현식에서 메서드를 호출하는 것이 좋습니다
```Java
@Component
public class NameComponent {
    public String getUsername(Member member) {
        return String.format("이름: %s 나이: %d", member.getUsername(), member.getAge());
    }
}
public interface MemberSimpleInfo {
    @Value("#{@nameComponent.getComponent(target)}")
    String getComponent();
}
```
#### 테스트코드
```Java
@DisplayName("컴포넌트로 테스트해보기")
@Test
void pro3(){
    //given
    Member member1 = new Member("둘리동생", 20);
    Member member2 = new Member("희동이", 5);
    Member member3 = new Member("둘리아빠", 40);
    memberRepository.saveAllAndFlush(List.of(member1, member2, member3));
    em.clear();
    //when
    List<MemberSimpleInfo> findMember = memberRepository.findMembersByUsernameStartingWith("둘리");

    //then
    Assertions.assertThat(findMember).extracting("component")
            .containsExactlyInAnyOrder(
                    String.format("이름: %s 나이: %d", member1.getUsername(), member1.getAge()),
                    String.format("이름: %s 나이: %d", member3.getUsername(), member3.getAge()));
}
```
#### SQL
```SQL
select
    m1_0.member_id,
    m1_0.address_id,
    m1_0.age,
    m1_0.created_date,
    m1_0.team_id,
    m1_0.updated_date,
    m1_0.username 
from
    member m1_0 
where
    m1_0.username like ? escape '\'
```
**쿼리를 보면 알 수 있듯이 최적화가 되지 않고 모든 필드를 불러옵니다.**
  
## 값 타입 DTO(Data Transfer Objects)
프로젝션 인터페이스를 사용하는 것과 동일한 방식으로 이 DTO 타입을 사용할 수 있지만, 
프록시화(proxying)되지 않으며 **중첩 프로젝션을 적용할 수 없습니다.**

만약 저장소(store)가 쿼리 실행을 최적화하여 필드 로딩을 제한하는 경우, 
로딩할 필드는 **노출된 생성자의 매개변수 이름으로 결정**됩니다.  
  
+ DTO
```Java
public class MemberSimpleInfoDto {
    private final String username;
    private final Integer age;

    public MemberSimpleInfoDto(String username, Integer age) {
        this.username = username;
        this.age = age;
    }

    public String getUsername() {
        return username;
    }
    public Integer getAge() {
        return age;
    }
}
```  
+ 레포지토리
```Java
List<MemberSimpleInfoDto> findMembersDtoByUsername(String name);
```  
#### 테스트코드
```Java
오류난다..
class가 아니라 record면 동작한다....
스프링부트 버전 3.2 ..
```

### 동적 Projection
조회 타입을 invocation(호출) 시간에 선택하는 방법이 포함됩니다.   
동적 프로젝션을 적용하기 위해 다음 예시와 같이 쿼리 메서드를 사용할 수 있습니다:  
```Java
<T> List<T> findProjectionsByUsername(String username, Class<T> type);
```
#### 테스트코드
```Java
@DisplayName("DTO로도 가능합니다.")
@Test
void dto2(){
    //given
    Member member1 = new Member("둘리동생", 20);
    Member member2 = new Member("희동이", 5);
    Member member3 = new Member("둘리아빠", 40);
    memberRepository.saveAllAndFlush(List.of(member1, member2, member3));
    em.clear();

    //when
    List<MemberInfoDto> findMembers = memberRepository.findProjectionsByUsernameStartingWith("둘리", MemberInfoDto.class);

    //then
    Assertions.assertThat(findMembers).hasSize(2);
    Assertions.assertThat(findMembers).extracting("username","age")
            .containsExactlyInAnyOrder(
                    tuple("둘리동생",20),
                    tuple("둘리아빠",40)
            );
}
```  
이건 또 동작 잘합니다..

### 중첩 구조 처리
중첩 구조를 가진 인터페이스 생성
> 참고:  
> getXXXX()는 엔티티의 필드명이 와야합니다.  
> 예) Team team; 이라면 getTeam이 되어야 합니다.
```Java
@Entity
@Getter @Setter
@NoArgsConstructor
public class Company {
    @Id
    @GeneratedValue
    private Long id;

    private String companyName;
}
```
```Java
@DisplayName("중첩 처리")
@Test
void nested(){
    //given
    Company company = new Company();
    company.setCompanyName("이마트");
    em.persist(company);

    Member member1 = new Member("둘리동생", 20,company);
    Member member2 = new Member("희동이", 5,company);
    memberRepository.saveAllAndFlush(List.of(member1, member2));
    em.clear();

    //when
    List<NestedClosedProjections> findMember = memberRepository.findMembersByUsername("희동이");

    //then
    Assertions.assertThat(findMember).hasSize(1);
    Assertions.assertThat(findMember).extracting("username","company.companyName")
            .contains(tuple("희동이","이마트"));
}
```
```SQL
select
    m1_0.username,
    c1_0.id,
    c1_0.company_name,
    c1_0.nothing1,
    c1_0.nothing2,
    c1_0.nothing3,
    c1_0.nothing4 
from
    member m1_0 
left join
    company c1_0 
        on c1_0.id=m1_0.company_id 
where
    m1_0.username=?
```  
#### 중첩 인터페이스 주의사항 
+ 프로젝션 대상이 root 엔티티면, JPQL SELECT 절 최적화 가능 
+ 프로젝션 대상이 ROOT가 아니면
 + LEFT OUTER JOIN 처리
 + 모든 필드를 SELECT해서 엔티티로 조회한 다음에 계산

> 가장 상위 엔티티는 최적화를 통해서 필요한 칼럼만 읽어오지만 
> 나머지는 LEFT JOIN으로 모든 필드를 조회합니다.  
  
### 정리
+ 프로젝션 대상이 root 엔티티면 유용합니다.
+ 프로젝션 대상이 root 엔티티를 넘어가면 JPQL SELECT 최적화가 안된다!
+ 실무의 복잡한 쿼리를 해결하기에는 한계가 있다.

