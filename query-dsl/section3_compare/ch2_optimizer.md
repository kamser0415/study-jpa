# 동적 쿼리와 성능 최적화 조회
동적 쿼리와 성능 최적화를 하기 위해서 필요한 데이터만 프로젝션하여 
DTO로 반환하는 방법을 사용합니다.    
  
그리고 DTO로 반환히 `N+1`문제를 해결 할 수 있습니다.
#### DTO
```Java
@Data
public class CommentListDto {

    private String username;
    private String text;
    private String title;

    @QueryProjection
    public CommentListDto(String username, String text, String title) {
        this.username = username;
        this.text = text;
        this.title = title;
    }
}
```  
해당 DTO는 글쓴이 이름과,댓글 내용, 댓글단 게시글만 저장합니다.  
> 복습 :   
> QueryDSL에서 DTO로 매핑하여 반환하는 방법
> 1. Projection.bean - 프로퍼티 빈 초기화 방식
> 2. Projection.fields - 필드명 초기화 방식
> 3. Projection.constructor - 생성자 초기화 방식
> 4. @QueryProjection - queryDSL 전용 방식  
  
## 동적 쿼리용 Condition - DTO 생성  
```Java
@Data
public class SearchCommentCondition {

    private String username;
    private String title;
    private String comment;
    private String searchType;

    public SearchCommentCondition(String username, String keyword, String searchType) {
        this.username = username;
        this.keyword = keyword;
        this.searchType = searchType;
    }
}
```  
동적 쿼리 검색 조건으로 글쓴이 이름, 제목, 댓글, 제목+댓글, 검색타입을 받을 수 있는
DTO 입니다.   

### 동적쿼리 - Builder 사용
```Java
@Override
public List<CommentListDto> searchByBuilder(SearchCommentCondition condition) {
    BooleanBuilder builder = new BooleanBuilder();

    String keyword = condition.getKeyword();
    if (StringUtils.hasText(keyword)) {
        String searchType = condition.getSearchType();
        if ("name".equals(searchType)) {
            builder.and(user.name.contains(keyword));
        }
        else if ("text".equals(searchType)) {
            builder.and(comment.text.contains(keyword));
        }
        else if ("title".equals(searchType)) {
            builder.and(post.name.contains(keyword));
        } else {
            builder.and((user.name.contains(keyword))
                    .or(comment.text.contains(keyword))
                    .or(post.name.contains(keyword)));
        }
    }
    return queryFactory
            .select(new QCommentListDto(
                    user.name.as("username"),
                    comment.text.as("text"),
                    post.name.as("title")))
            .from(comment)
            .leftJoin(comment.post, post)
            .leftJoin(comment.user, user)
            .where(builder)
            .fetch();
}
```  
> QCommentListDto는 생성자를 이용해 초기화 하기 때문에 필드이름을 맞추지 않아도 됩니다.
**동적 테스트 코드**
```Java
@DisplayName("동적 쿼리 테스트")
@Test
void dynamic_query_test(){
    //given
    User admin = new User(1L, "상담사");
    User developer = new User(2L, "개발자");
    Post request1 = new Post(1L, "전화번호 6060님이 ~가 안된데요", admin);
    Post request2 = new Post(2L, "전화번호 7070님이 ~가 안된데요", admin);
    Post updatePost = new Post(3L, "내부 프로그램 업데이트 했습니다.", developer);
    Comment comment1 = new Comment(1L, "수정했습니다.", developer, request1);
    Comment comment2 = new Comment(2L, "6060님에게 전달했습니다.", admin, request1);
    Comment comment3 = new Comment(3L, "수정했습니다.", admin, request2);
    Comment comment4 = new Comment(4L, "금칙어 기능이 추가되었습니다.", developer, updatePost);
    // 데이터를 저장하고 em.clear()를 했습니다.

    //when
    SearchCommentCondition cond = new SearchCommentCondition( "발", "name");
    List<CommentListDto> result = repository.searchByBuilder(cond);

    //then
    Assertions.assertThat(result).hasSize(2);
}
```  
```SQL
select
    u1_0.name,
    c1_0.text,
    p1_0.name 
from
    comment c1_0 
left join
    post p1_0 
        on p1_0.id=c1_0.post_id 
left join
    users u1_0 
        on u1_0.id=c1_0.user_id 
where
    u1_0.name like ? escape '!'
```  
정상적으로 동적 조회가 가능합니다.  
  
### Where 절 파라미터 사용
```Java
@Override
public List<CommentListDto> searchByWhere(SearchCommentCondition condition) {
    return queryFactory
            .select(new QCommentListDto(
                    user.name.as("username"),
                    comment.text.as("text"),
                    post.name.as("title")))
            .from(comment)
            .leftJoin(comment.post, post)
            .leftJoin(comment.user, user)
            .where(
                    searchCond(condition)
            )
            .fetch();
}
private BooleanExpression searchCond(SearchCommentCondition cond) {
    if (!StringUtils.hasText(cond.getKeyword())) {
        return null;
    }
    if (cond.getSearchType().equals("name")) {
        return searchName(cond);
    }
    else if (cond.getSearchType().equals("text")) {
        return searchText(cond);
    }
    else if (cond.getSearchType().equals("title")) {
        return searchTitle(cond);
    }
    else {
        return searchName(cond).or(searchText(cond)).or(searchTitle(cond));
    }
}

private BooleanExpression searchTitle(SearchCommentCondition cond) {
    return post.name.contains(cond.getKeyword());
}

private BooleanExpression searchText(SearchCommentCondition cond) {
    return comment.text.contains(cond.getKeyword());
}

private BooleanExpression searchName(SearchCommentCondition cond) {
    return user.name.contains(cond.getKeyword());
}
```  
> where절에 파라미터 방식을 사용하면 재사용이 가능하다.  
> 테스트 코드를 작성할 때 Builder와 where절의 장단점이 있습니다.
> 1. 단순 null이나 공백만 체크하면 결과는 항상 동일한 조건은 Expression이 유리
> 2. 들어오는 값에 따라 다 다른 Expression을 반환하는 경우는 Builder가 유리

추가로 함수를 만들어서 하나로 합치면 가독성도 좋아집니다.  
예시 코드 :
```Java
public BooleanExpression dateBetween(cond){
    return dateGoe(cond.getDateGoe())and(dateLoe(cond.getDateLoe()));
} 
```  
> 반환 타입을 BooleanExpression으로 변경하는게 좋습니다.  
이유는 컴포지션이 가능하기 때문에 메소드 체이닝이 가능합니다.