# 수정,삭제 벌크 연산
## 쿼리 한번으로 대량 데이터 수정  
### JPA와 QueryDSL 비교
+ **JPA**
    ```Java
    int updatedEntities = entityManager.createQuery(
        "update Person p set p.name = :newName where p.name = :oldName")
        .setParameter("oldName", oldName)
        .setParameter("newName", newName)
        .executeUpdate();
    ```  
+ **QueryDSL**
    #### With where
    ```Java
    QSurvey survey = QSurvey.survey;
    queryFactory.update(survey)
                .where(survey.name.eq("XXX"))
                .set(survey.name, "S")
                .execute();
    ```
    #### Without where
    ```Java
    queryFactory.update(survey)
                .set(survey.name, "S")
                .execute();
    ```
### 연산
```Java
queryFactory.update(member)
              .set(member.age,member.age.add(15))
              .execute();
// 결과
update member set age=(age+?)
```
### 동적 업데이트
```Java
JPAUpdateClause update = queryFactory.update(member);
if(condition < 16){
    update.set(member.age,member.age.add(15))
}
update.execute();
```

## 삭제  
```Java
QCustomer customer = QCustomer.customer;
// delete all customers
queryFactory.delete(customer).execute();
// delete all customers with a level less than 3
queryFactory.delete(customer).where(customer.level.lt(3)).execute();
```
+ **where 호출은 선택입니다.**  

execute 호출은 삭제를 수행하고 삭제된 엔티티의 수를 반환합니다.  
JPA에서의 DML 절은 **JPA 레벨의 cascade 규칙을 고려하지않습니다.**  

```Java
@Entity
public class Post {
    @OneToMany(mappedBy = "post",cascade = CascadeType.ALL,orphanRemoval = true)
    private Set<Comment> comments = new HashSet<>();
}
```
#### 테스트 코드
+ QueryDsl
  ```Java
  @DisplayName("고아 객체 테스트")
  @Test
  void t3(){
      //given
      Post post = new Post(1L,"제목 1",null);
      Comment comment = new Comment(1L, "댓글 1", null, post);
      post.getComments().add(comment);
      em.persist(post);
  
      em.flush();
      em.clear();
      //when
      queryFactory.delete(QPost.post)
              .execute();
      em.flush();
      em.clear();
  
      List<Comment> fetch = queryFactory.selectFrom(QComment.comment).fetch();
      //then
      Assertions.assertThat(fetch).hasSize(1);
  }
  ```
+ 영속성 컨텍스트
  ```Java
  @DisplayName("영속성 컨텍스트를 이용할때")
  @Test
  void t1(){
      //given
      Post post = new Post(1L,"제목 1",null);
      Comment comment = new Comment(1L, "댓글 1", null, post);
      post.getComments().add(comment);
      em.persist(post);
  
      em.flush();
      em.clear();
      //when
      Post findPost = em.find(Post.class, 1L);
      em.remove(findPost);
      em.flush();
      em.clear();
      //then
      Comment comment1 = em.find(Comment.class, comment.getId());
      Assertions.assertThat(comment1).isNull();
  }
  ```  
#### 정리  
JPQL 배치와 마찬가지로 영속성 컨텍스트에 먼저 접근하는게 아니라, 데이터베이스에 
실행을 하고 결과를 가져오기 때문에 JPA 기술인 Cascade나 고아객체는 사용되지 않습니다.  
  

