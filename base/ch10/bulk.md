# 벌크 연산   
<!-- TOC -->
* [벌크 연산](#벌크-연산-)
    * [벌크 연산의 주의점](#벌크-연산의-주의점)
    * [해결 방법](#해결-방법-)
      * [em.refresh() 사용](#emrefresh-사용-)
      * [벌크 연산 먼저 실행](#벌크-연산-먼저-실행)
      * [벌크 연산 수행후 영속성 컨텍스트 초기화](#벌크-연산-수행후-영속성-컨텍스트-초기화-)
<!-- TOC -->
엔티티를 수정하려면 영속성 컨텍스트의 **변경 감지** 기능이나 **병합**을 사용하고, 
삭제하려면 `EntityManager.remove()`메소드를 사용합니다.  
  
이 방법으로 수백개 이상의 엔티티를 하나씩 처리하기에는 시간이 너무 오래 걸립니다. 
이런 경우 한 번에 수정하거나 삭제하는 벌크 연산을 사용하면 됩니다.  
  
```java
@DisplayName("벌크 연산으로 한번에 처리하기")
@Test
void t7(){
    for (int i = 1; i <= 5; i++) {
        Member member = new Member();
        member.setAge(i);
        member.setUsername("손님 "+i);
        em.persist(member);
    }

    String sql = "update Member as m set m.age = m.age + 1";
    int updateCount = em.createQuery(sql).executeUpdate();
    System.out.println("updateCount = 5 ? " + (updateCount == 5));
    //true
}
```
```sql
update
    Member 
set
    age=age+1
```  

결과값으로 수정된 엔티티의 갯수가 반환 됩니다.  

### 벌크 연산의 주의점
업데이트 또는 삭제 구문의 효과는 실행될 때의 영속성 컨텍스트나 메모리에 보유된 **엔티티 객체의 상태에 반영되지 않습니다**.  
업데이트 또는 삭제 구문 실행 이후에 메모리에 보유된 상태를 데이터베이스와 **동기화하는 것은 클라이언트 프로그램의 책임**입니다.  

+ 테스트 코드
```sql
@DisplayName("벌크 연산으로 한번에 처리하기")
@Test
void t7(){
    Member member = new Member();
    member.setAge(1);
    member.setUsername("손님");
    em.persist(member);

    String sql = "update Member as m set m.age = m.age + 1 where m = :member";
    em.createQuery(sql)
            .setParameter("member",member)
            .executeUpdate();

    Member findMember = em.find(Member.class, member.getId());

    Assertions.assertEquals(2,findMember.getAge());
}

org.opentest4j.AssertionFailedError: 
Expected :2
Actual   :1
// fail!
```  
**벌크 연산도 JPQL의 종류중 하나입니다.**  
즉, 영속성 컨텍스트를 통하지 않고 데이터베이스에 직접 쿼리합니다. 
따라서 영속성 컨텍스트에 있는 1살 손님은 메모리에 저장된 상태로 남아 있고 
데이터베이스에는 2살 손님으로 변경되었습니다.  
  
### 해결 방법  
#### em.refresh() 사용  
벌크 연산을 수행한 직후에 정확한 엔티티를 사용해야한다면 `em.refresh()`를 사용해서 
데이터베이스에서 해당 엔티티를 다시 조회하면 됩니다.  
  
#### 벌크 연산 먼저 실행
가장 실용적인 방법은 벌크 연산을 가장 먼저 실행합니다. 
먼저 벌크 연산을 실행하고 나서 엔티티를 조회하면 변경된 엔티티를 조회할 수 있습니다. 
이 방법은 JPA와 JDBC를 사용할 때도 마찬가지 입니다.  
영속성 컨텍스트를 먼저 확인하는 프로세스와 데이터베이스에 직접 접근하는 프로세스를 같이 사용할 경우 
이 방법이 유용합니다.  
  
#### 벌크 연산 수행후 영속성 컨텍스트 초기화  
벌크 연산을 수행하고 바로 영속성 컨텍스트를 초기화해서 남아 있는 엔티티를 제거하는 방법입니다. 
초기화 이후 엔티티를 조회할 때 벌크 연산이 적용된 엔티티를 조회합니다.  
  
**_벌크 연산이나 JDBC , MyBatis 쿼리를 먼저 실행하는 추천합니다._**
