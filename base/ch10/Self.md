# 엔티티 직접 사용    
<!-- TOC -->
* [엔티티 직접 사용](#엔티티-직접-사용-)
    * [외래키도 동일하다](#외래키도-동일하다)
  * [정리](#정리-)
<!-- TOC -->
+ JPQL에서 엔티티를 직접 사용하면 SQL에서 해당 엔티티의 기
본 키 값을 사용
    ```jpaql
    select count(m.id) from Member m -- 엔티티의 아이디를 사용
    select count(m) from Member m    -- 엔티티를 직접 사용
    ```
+ JPQL 둘다 같은 다음 SQL 실행
    ```jpaql
    select count(m.id) as cnt from Member m
    ```

+ 테스트 코드
    ```java
    @DisplayName("엔티티 직접 사용")
    @Test
    void t3(){
        Member member = new Member();
        member.setUsername("둘리");
        em.persist(member);
    
        em.flush();
        em.clear();
    
        Member findByEntity = em.createQuery("select m from Member as m where m = :member", Member.class)
                .setParameter("member",member)
                .getSingleResult();
        Member findById = em.createQuery("select m from Member as m where m.id = :memberId", Member.class)
                .setParameter("memberId",member.getId())
                .getSingleResult();
    
        Assertions.assertEquals(findByEntity,findById);
    }
    ```
    ```sql
    select
        member0_.*
    from 
        Member member0_ 
    where 
        member0_.id=?
    ```  

### 외래키도 동일하다
+ 테스트 코드
    ```java
    @DisplayName("엔티티 직접 사용-외래 키")
    @Test
    void t4(){
        Team team = new Team();
        team.setName("아기 공룡");
        em.persist(team);
    
        Member member = new Member();
        member.setUsername("둘리");
        member.setTeam(team);
        em.persist(member);
    
        em.flush();
        em.clear();
    
        Member findByEntity = em.createQuery("select m from Member as m where m.team = :team", Member.class)
                .setParameter("team",team)
                .getSingleResult();
    }
    ```
+ 외래 키도 동일하게 엔티티를 사용할 수 있습니다.
    ```jpaql
    select m from Member as m where m.team = :team
    ```
    ```sql
    select
        member0_.id as id1_2_,
        member0_.addressAlias_id as addressa7_2_,
        member0_.city as city2_2_,
        member0_.street as street3_2_,
        member0_.zipcode as zipcode4_2_,
        member0_.age as age5_2_,
        member0_.team_id as team_id8_2_,
        member0_.username as username6_2_ 
    from
        Member member0_ 
    where
        member0_.team_id=?
    ```
  
## 정리  
root 테이블 (`Member`)가 연관관계인 엔티티는 `id`와 `Entity`로 조회할 수 있습니다.  
