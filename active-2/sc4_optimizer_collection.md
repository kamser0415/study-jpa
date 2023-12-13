# 일대다 컬렉션 최적화
<!-- TOC -->
* [일대다 컬렉션 최적화](#일대다-컬렉션-최적화)
  * [페이징과 한계 돌파](#페이징과-한계-돌파-)
      * [다대일의 N+1 방식](#다대일의-n1-방식)
      * [일대다의 N+1 방식](#일대다의-n1-방식)
      * [페이징 방법](#페이징-방법)
<!-- TOC -->
`@xxxToOne` 관계에서는 지연로딩으로 설정하고 필요한 연관관계 엔티티가 있다면 
`join fetch`를 통해서 여러 번 조회하는 `N+1`문제를 해결하고, 한 번에 데이터를 읽어오는 방법을 
통해 최적화를 했습니다.    


다대일과 다르게 일대다의 경우 조인 칼럼의 집합의 레벨은 `N`쪽으로 변경됩니다.  
집합 레벨이 `N`으로 변경된다는 이야기는 
조인 컬럼이 중복인지 아닌지를 확인하면 됩니다.   
예를들어 :  
Team.id와 Member.team_id를 조인 칼럼으로 관계를 맺는다면 
Team.id는 유니크로 집합의 레벨은 `1`이 됩니다. `Member.team_id`는 
중복을 허용하는 집합의 레벨 `N`을 갖습니다.  
따라서 조인 결과는 Member.team_id 집합의 레벨인 `N`으로 변경됩니다.  

```SQL
SELECT 
    *
FROM Team T
LEFT OUTER JOIN Members M
ON T.id(1) = M.team_id(N) 
```  
+ 팀 테이블  

    | 팀 ID | 팀명   | 설명         |
      |------|------|------------|
      | 1    | 개발팀  | 소프트웨어 개발   |
      | 2    | 마케팅팀 | 제품 홍보 및 광고 |
+ 회원 테이블  

  | 회원 ID | 이름      | 소속 팀 ID | 
  |-------|---------|---------|
  | 1     | John    | 1       |
  | 2     | Emily   | 2       |
  | 3     | Michael | 1       |
  | 4     | Sophia  | 2       |
+ 조인 결과  

  | 팀 ID | 팀명   | 설명         | 회원 ID | 이름      | 소속 팀 ID |
  |------|------|------------|-------|---------|---------|
  | 1    | 개발팀  | 소프트웨어 개발   | 1     | John    | 1       |
  | 1    | 개발팀  | 소프트웨어 개발   | 3     | Michael | 1       |
  | 2    | 마케팅팀 | 제품 홍보 및 광고 | 2     | Emily   | 2       |
  | 2    | 마케팅팀 | 제품 홍보 및 광고 | 4     | Sophia  | 2       |

JPA는 데이터베이스의 결과를 영속성 컨텍스트 로드를 합니다. 
그리고 ROW당 하나의 엔티티로 매핑해서 반환하기 때문에 중복된 엔티티가 발생하게 됩니다.  
  
중복을 제거하기 위해서 `distinct`를 사용하면 JPA는 실제 SQL에서도 
중복 제거를 처리하고 중복된 엔티티는 제거하고 반환합니다.

```SQL
select
    distinct 
    orders.*
    member.*,
    delivery.*,
    order_item.*,
    item.*
from
    orders orders 
inner join
    member member 
        on orders.member_id=member.member_id 
inner join
    delivery delivery 
        on orders.delivery_id=delivery.delivery_id 
inner join
    order_item order_item 
        on orders.order_id=order_item.order_id 
inner join
    item item 
        on order_item.item_id=item.item_id
```  
해당 데이터베이스 연관 관계는 다음과 같습니다.
+ Order : Member = `N: 1`
+ Order : Delivery = `1: 1` 
+ Order : OrderItem = `1: N`
+ OrderItem : Item = `N: 1`  
  
페치조인으로 컬렉션 엔티티를 읽어올 경우도 마찬가지로 최적화로 SQL 쿼리 하나로 연관된 엔티티의 
전부 읽어올 수 있습니다.  
  
페치 조인은 객체 그래프 탐색을 할 수 있어야하고, 초기화된 엔티티는 페치 조인으로 읽어온 
엔티티의 모든 정보가 매핑이 되어야합니다. 
 
따라서 일부 데이터만 읽어오는 페이징이나 `LIMIT`등은 사용할 수 없습니다. 
사용은 가능하지만 메모리에 저장해 놓고 필요할 경우에 해당 데이터 영역을 읽어오는 방식이라 
데이터가 커진다면 메모리 부족이 발생할 수 있습니다.  
  
## 페이징과 한계 돌파  
컬렉션을 페치조인하면 페이징이 불가능합니다. 
  
이럴 경우에는 페치 조인과 `@BatchSize`를 혼합하여 사용하는 방식을 사용합니다.  
단 기준이 되는 주 테이블을 기준으로 `@xxxToOne`의 관계는 객체 그래프를 타고 계속 읽을 수 있지만, 
`@xxxToMany`로 컬렉션을 조회하는 경우 컬렉션 조회는 한 번만 가능합니다.  

예를 들면:  
```text
Team : Member : Family = 1: N : M
```  
팀 엔티티에서 회원 엔티티를 조회하고 회원 엔티티에서 가족 엔티티를 조회하는 방식은 사용할 수 없습니다. 
  
1. `@xxxToOne` 관계는 모두 페치 조인을 합니다.  
    레코드의 수가 변하지 않기 때문에 페이징 쿼리에 영향을 주지 않습니다.  
2. 컬렉션은 지연 로딩으로 조회합니다.
3. 지연 로딩 최적화를 위해 `hibernate.default_batch_fetch_size`: 글로벌 환경 설정이나 
    특정 연관 관계에 대해 설정하는 `@BatchSize`를 적용합니다.
    해당 방식을 사용할 경우 적용한 숫자만큼 `WHERE IN (...)` 쿼리로 한번에 읽어옵니다.  
  
```Java
// N:1, 1:1 연관관계는 페치조인으로 읽어옵니다.
public List<Order> findAllWithMemberDelivery(int offset, int limit) {
    return em.createQuery(
            "select o from Order o" +
                    " join fetch o.member m" +
                    " join fetch o.delivery d", Order.class)
            .setFirstResult(offset)
            .setMaxResults(limit)
            .getResultList();
}
// 1:N 연관관계는 배치조인으로 추가 조회합니다.
@GetMapping("/api/v3.1/orders")
public List<OrderDto> ordersV3_page(
    @RequestParam(value = "offset", defaultValue = "0") int offset,
    @RequestParam(value = "limit", defaultValue = "100") int limit
) {
    List<Order> orders = orderRepository.findAllWithMemberDelivery(offset, limit);
    // 강제 조회로 초기화하기
    List<OrderDto> result = orders.stream()
            .map(o -> new OrderDto(o))
            .collect(toList());

    return result;
}
```  

#### 다대일의 N+1 방식
다대일의 경우 연관관계 주인은 `다`쪽이 가지고 있습니다. 테이블로 보면 참조 칼럼을 `다`쪽에서 관리하기 때문에 
`일`쪽의 PK값을 가지고 있기때문에 개별 조회가 가능합니다. 
`N`개의 엔티티가 있다면 최대 `M`번의 추가 조회 쿼리가 발생합니다.

#### 일대다의 N+1 방식
다대일과 다르게 일대다는 자신을 참조하는 엔티티를 개별로 조회할 수 없습니다. 
그래서 조회를 하면 자신을 참조하는 모든 엔티티를 한번에 가져옵니다.  

연관관계 구조를 다시 살펴보겠습니다. 
+ Order : Member = `N: 1`
+ Order : Delivery = `1: 1`
+ Order : OrderItem = `1: N`
+ OrderItem : Item = `N: 1`   
  
현재 데이터 베이스 엔티티 구조를 도식화했습니다.   
주문당 주문 상품 엔티티 2개가 참조하고 있고, 주문 상품당 하나의 엔티티를 가지고 있습니다.
```text
주문(Order)
    └── 주문-상품(Order-Product) 엔티티
         └── 아이템(Item) 엔티티
    └── 주문-상품(Order-Product) 엔티티
         └── 아이템(Item) 엔티티
주문(Order)
    └── 주문-상품(Order-Product) 엔티티
         └── 아이템(Item) 엔티티
    └── 주문-상품(Order-Product) 엔티티
         └── 아이템(Item) 엔티티
```  
여기서 배치사이즈를 사용하지 않고 조회를 한다면   
쿼리는 최대 7번이 실행됩니다.
1. 주문 엔티티 1번 조회
2. 주문을 참조하는 주문-상품 엔티티 * 2번 조회  
3. 주문-상품당 아이템 엔티티 조회 * 4번 조회  

```SQL
-- 주문 엔티티 조회
select order0_.* from orders order0_ inner join member member1_ on order0_.member_id=member1_.member_id inner join delivery delivery2_ on order0_.delivery_id=delivery2_.delivery_id limit ?
-- 첫번째 주문 엔티티를 참조하는 주문-상품 엔티티 전체 조회
select orderitems0_.* from order_item orderitems0_ where orderitems0_.order_id=?
-- 주문-상품이 참조하는 아이템 엔티티 조회
select item0_.* from item item0_ where item0_.item_id=?
-- 주문-상품이 참조하는 아이템 엔티티 조회
select item0_.* from item item0_ where item0_.item_id=?
-- 두번째 주문 엔티티를 참조하는 주문-상품 엔티티 전체 조회
select orderitems0_.* from order_item orderitems0_ where orderitems0_.order_id=?
-- 주문-상품이 참조하는 아이템 엔티티 조회
select item0_.* from item item0_ where item0_.item_id=?
-- 주문-상품이 참조하는 아이템 엔티티 조회
select item0_.* from item item0_ where item0_.item_id=?
```
  
이제 컬렉션 엔티티를 조회하는 경우 배치 사이즈를 설정하면 쿼리가 다음과 같이 변합니다.  
```yaml
default_batch_fetch_size: 1000 #최적화 옵션
```
```SQL
-- 주문-상품 조회
select
    orderitems0_.*
from
    order_item orderitems0_ 
where
    orderitems0_.order_id in ( ?, ? )
-- 아이템 조회
select
    item0_.* 
from
    item item0_ 
where
    item0_.item_id in ( ?, ?, ?, ? )
```   
조회를 할때마다 sql을 바로 실행하지 않고 배치사이즈만큼 모아서 `WHERE IN(...)`쿼리로 
최적화를 했습니다.  
+ 페치 조인 방식과 비교해서 쿼리 호출 수가 약간 증가하지만, DB 데이터 전송량이 감소합니다.
+ 컬렉션 페치 조인은 페이징이 불가능 하지만 이 방법은 페이징이 가능합니다.  

#### 페이징 방법
1. 다대일, 일대일 연관관계는 페치조인으로 페이징을 합니다.
2. 그 결과에서 일대다 연관관계 엔티티를 호출해서 초기화를 합니다.
```Java
// 다대일, 일대일 연관관계는 페치조인과 페이징 실행
public List<Order> findAllWithMemberDelivery(int offset, int limit) {
    return em.createQuery(
            "select o from Order o" +
                    " join fetch o.member m" +
                    " join fetch o.delivery d", Order.class)
            .setFirstResult(offset)
            .setMaxResults(limit)
            .getResultList();
} 
// 일대다는 외부에서 조회로 초기화를 하는데 배치사이즈로 최적화를 합니다.
List<OrderDto> result = orders.stream()
                            .map(o -> new OrderDto(o))
                            .collect(toList());
```
