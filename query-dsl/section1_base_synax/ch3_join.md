# 조인-기본 조인과 페치조인
## 기본 조인  
- **연관관계 조인일 경우**  
첫 번째 파라미터에 조인 대상을 지정,  
두 번째 파라미터에 별칭(`alias`)으로 사용할 Q타입을 지정하면 됩니다.  
  > 주의할점:
묵시적 조인이 발생하면 `CROSS JOIN`이 발생합니다.
- **연관관계가 아닌 조인일 경우**  
  내부,외부 조인할 `QClass`만 인수로 전달하면 됩니다.

+ 내부조인
    ```Java
    QCustomer customer = QCustomer.customer;
    QCompany company = QCompany.company;
    queryFactory.select(customer.firstName, customer.lastName, company.name)
        .from(customer)
        .innerJoin(customer.company, company)
        .fetch();
    ```
+ 외부조인
    ```Java
    queryFactory.select(customer.firstName, customer.lastName, company.name)
        .from(customer)
        .leftJoin(customer.company, company)
        .fetch();
    ```  
+ 조인 조건 추가
    ```Java
    queryFactory.select(customer.firstName, customer.lastName, company.name)
        .from(customer)
        .leftJoin(company).on(customer.company.eq(company))
        .fetch();
    ```  
  
#### 테스트코드 작성
+ 이너조인 테스트
  ```Java
  @DisplayName("연관관계 inner join")
  @Test
  void test(){
      /*
      회원과 N:1 인 팀을 불러옵니다.
       */
      em.flush();
      em.clear();
  
      QMember member = QMember.member;
      QTeam team = QTeam.team;
      //when
      List<Member> teamA = queryFactory.select(member)
              .from(member)
              .join(member.team, team)
              .where(team.name.eq("teamA"))
              .fetch();
      // then
      Assertions.assertThat(teamA).hasSize(2);
      Assertions.assertThat(Hibernate.isInitialized(teamA.get(0).getTeam())).isTrue();
      Assertions.assertThat(Hibernate.isInitialized(teamA.get(1).getTeam())).isTrue();
      // 1번
      Assertions.assertThat(teamA.get(1).getTeam().getName()).isEqualTo("teamA");
  }
  ```
+ SQL
  ```SQL
  select
      m.member_id,
      m.age,
      m.team_id,
      m.username 
  from
      member m 
  join
      team t on t.team_id=m.team_id 
  where
      t.name=?
      
  -- 이너 조인을 했지만 LAZY 관계라서 1번 코드때문에 추가 SQL 실행
  select
      t1_0.team_id,
      t1_0.name 
  from
      team t1_0 
  where
      t1_0.team_id=?
  ```  
+ 아우터 조인
  ```Java
  List<Member> teamA = queryFactory.select(member)
                  .from(member)
                  .leftJoin(member.team, team)
                  .where(team.name.eq("teamA"))
                  .fetch();
  ```
  ```SQL
  select
      m1_0.member_id,
      m1_0.age,
      m1_0.team_id,
      m1_0.username 
  from
      member m1_0 
  left join
      team t1_0 
          on t1_0.team_id=m1_0.team_id 
  where
      t1_0.name=?
  ```
## 세타조인
+ 연관관계가 없는 필드로 조인
+ 아무런 상관없는 테이블과 조인
##### 테스트코드
```Java
@DisplayName("연관관계아닌 세타조인")
@Test
void join_without_r(){
    /*
    아무런 상관없는 회사 엔티티의 회사 나이와 조인
     */
    em.persist(new Company(10));
    em.persist(new Company(20));
    em.flush();
    em.clear();

    QMember member = QMember.member;
    //when
    QCompany company = QCompany.company;
    List<Member> teamA = queryFactory.select(member)
            .from(member, company)
            .where(member.age.eq(company.companyAge))
            .fetch();
}
```
+ SQL
  ```SQL
  select
      m1_0.member_id,
      m1_0.age,
      m1_0.team_id,
      m1_0.username 
  from
      member m1_0,
      company c1_0 
  where
      m1_0.age=c1_0.company_age
  ```
> Member와 Team은 team.id로 연관관계가 맺어져 있고,
> Member와 Company는 아예 서로 관련이 없는 테이블입니다. 
> 두 테이블을 조인을 해야할때 사용할 수 있습니다.
  
+ leftjoin()후 where로 필터를 사용할 경우 예외가 발생합니다.  
  `LEFTJOIN()`과 `WHERE`의 조건이 있다면 `INNER JOIN`과 동일한 코드가 실행됩니다.
  ```Java
  @DisplayName("on절 없이 세타조인는 예외가 발생합니다.")
  @Test
  void join_without_on() {
      /*
      아무런 상관없는 회사 엔티티의 회사 나이와 조인
       */
      em.persist(new Company(10));
      em.persist(new Company(20));
      em.flush();
      em.clear();
  
      QMember member = QMember.member;
      QCompany company = QCompany.company;
      //when
      Assertions.assertThatThrownBy(() ->
              queryFactory.select(member)
                      .from(member)
                      .leftJoin(company)
                      .where(member.age.eq(company.companyAge))
                      .fetch()
      ).isInstanceOf(IllegalArgumentException.class);
  }
  ```  
> 이때 Hibernate 내부에서 예외 컨버터가 동작해서 queryDSL 예외를 
> Java RuntimeException으로 변경합니다.  
  
## 조인 - on 절  
드라이빙 테이블의 데이터는 유지하고 드리븐 테이블에서 필요한 데이터만 가져올 때 사용합니다.  
  
ㅓ
```Java
@DisplayName("on절을 추가해서 세타조인 사용하기")
@Test
void join_with_on() {
    /*
    팀원의 정보는 모두 가져오고 팀원의 나이와 회사의 나이가 같을 경우
    회사의 정보도 가져오기.
     */
    em.persist(new Company(10));
    em.persist(new Company(20));
    em.flush();
    em.clear();

    QMember member = QMember.member;
    QCompany company = QCompany.company;
    //when
    List<Member> findMember = queryFactory.select(member)
            .from(member)
            .leftJoin(company)
            .on(member.age.eq(company.companyAge))
            .fetch();

    Assertions.assertThat(findMember).hasSize(4);
}
```
+ **SQL**
  ```SQL
  select
      m1_0.member_id,
      m1_0.age,
      m1_0.team_id,
      m1_0.username 
  from
      member m1_0 
  left join
      company c1_0 
          on m1_0.age=c1_0.company_age
  ```  
> 참고 :  
> JOIN ~ ON 과 LEFT JOIN ~ WHERE 는 동일한 코드입니다. 
> JOIN은 공통적인 부분만 가져오기 때문에 ON의 결과도 전체 필터링 조건에 포함되지만,
> WHERE는 시작전이 이미 읽을때 필터링하기 때문에 두 콭드는 모두 동일합니다.  
  
## 조인 - 페치조인
페치조인은 SQL에서 제공하는 기능은 아닙니다.  
SQL 조인을 활용해서 연관된 엔티티를 SQL 한번에 조회하는 기능입니다. 
주로 성능 최적화에 사용하는 방법입니다.  

#### 페치조인
```Java
@DisplayName("fetch 조인으로 N+1을 해결하기")
@Test
void join_with_fetch() {
    /*
    Member를 가져오고 내부의 Team도 즉시 초기화하기
    */
    QMember member = QMember.member;
    QTeam team = QTeam.team;
    //when
    List<Member> findMember = queryFactory
            .select(member)
            .from(member)
            .leftJoin(member.team,team).fetchJoin()
            .fetch();

    Assertions.assertThat(findMember).hasSize(4);
    Assertions.assertThat(Hibernate.isInitialized(findMember.get(0).getTeam())).isTrue();
    Assertions.assertThat(Hibernate.isInitialized(findMember.get(1).getTeam())).isTrue();
    Assertions.assertThat(Hibernate.isInitialized(findMember.get(2).getTeam())).isTrue();
    Assertions.assertThat(Hibernate.isInitialized(findMember.get(3).getTeam())).isTrue();

    //혹은 아래와 같이 검증할 수 있습니다.
    Assertions.assertThat(findMember).extracting(Member::getTeam)
            .allSatisfy(Hibernate::isInitialized);
}
```  
Member > Team 이 `FETCH`지연 설정되어있을때 `.fetchJoin()`으로 즉시로딩 할 수 있습니다.  
  
#### 사용방법
+ `join()`,`leftJoin()`등 조인 기능 뒤에 `fetchJoin()`을 추가합니다.  
  
