# 컬렉션 조회 DTO로 최적화하기  

### eq_ref 조회  
장점 :  
1. 코드가 단순하다.
2. 특정 엔티티 한 건만 조회하는 방식일 경우 성능이 잘 나온다.  

단점 :
1. 범위 조회일 경우나 페이징시 성능이 떨어진다.
2. `N+1`조회가 발생한다. 

#### JPA에서 DTO 직접 조회하기
```Java
//DTO class
@EqualsAndHashCode(of = "orderId")
public class OrderQueryDto {

    private Long orderId;
    private String name;
    private LocalDateTime orderDate; //주문시간
    private OrderStatus orderStatus;
    private Address address;
    private List<OrderItemQueryDto> orderItems;

    public OrderQueryDto(Long orderId, String name, LocalDateTime orderDate, OrderStatus orderStatus, Address address) {
        this.orderId = orderId;
        this.name = name;
        this.orderDate = orderDate;
        this.orderStatus = orderStatus;
        this.address = address;
    }

    public OrderQueryDto(Long orderId, String name, LocalDateTime orderDate, OrderStatus orderStatus, Address address, List<OrderItemQueryDto> orderItems) {
        this.orderId = orderId;
        this.name = name;
        this.orderDate = orderDate;
        this.orderStatus = orderStatus;
        this.address = address;
        this.orderItems = orderItems;
    }
}
```
```Java
/* 
 * 1:N 관계(컬렉션)을 제외한 나머지를 한번에 조회
 */
private List<OrderQueryDto> findOrders() {
    String sql =
            "select new jpabook.jpashop.repository.order.query.OrderQueryDto(o.id, m.name, o.orderDate, o.status, d.address)" +
            " from Order o" +
            " join o.member m" +
            " join o.delivery d";
    return em.createQuery(sql, OrderQueryDto.class)
            .getResultList();
}
/**
 * 1:N 관계인 orderItems 조회
 */
private List<OrderItemQueryDto> findOrderItems(Long orderId) {
    return em.createQuery(
            "select new jpabook.jpashop.repository.order.query.OrderItemQueryDto(oi.order.id, i.name, oi.orderPrice, oi.count)" +
                    " from OrderItem oi" +
                    " join oi.item i" +
                    " where oi.order.id = : orderId", OrderItemQueryDto.class)
            .setParameter("orderId", orderId)
            .getResultList();
}
public List<OrderQueryDto> findOrderQueryDtos() {
    //루트 조회(toOne 코드를 모두 한번에 조회)
    List<OrderQueryDto> result = findOrders();

    //루프를 돌면서 컬렉션 추가(추가 쿼리 실행)
    result.forEach(o -> {
        List<OrderItemQueryDto> orderItems = findOrderItems(o.getOrderId());
        o.setOrderItems(orderItems);
    });
    return result;
}
```
해당 방식으로 실행할 경우
```SQL
-- findOrders();
select
    order0_.order_id as col_0_0_,
    member1_.name as col_1_0_,
    order0_.order_date as col_2_0_,
    order0_.status as col_3_0_,
    delivery2_.city as col_4_0_,
    delivery2_.street as col_4_1_,
    delivery2_.zipcode as col_4_2_ 
from
    orders order0_ 
inner join
    member member1_ 
        on order0_.member_id=member1_.member_id 
inner join
    delivery delivery2_ 
        on order0_.delivery_id=delivery2_.delivery_id
-- findOrderItems(o.getOrderId());
select
    orderitem0_.order_id as col_0_0_,
    item1_.name as col_1_0_,
    orderitem0_.order_price as col_2_0_,
    orderitem0_.count as col_3_0_ 
from
    order_item orderitem0_ 
inner join
    item item1_ 
        on orderitem0_.item_id=item1_.item_id 
where
    orderitem0_.order_id=?
-- findOrderItems(o.getOrderId());
select
    orderitem0_.order_id as col_0_0_,
    item1_.name as col_1_0_,
    orderitem0_.order_price as col_2_0_,
    orderitem0_.count as col_3_0_ 
from
    order_item orderitem0_ 
inner join
    item item1_ 
        on orderitem0_.item_id=item1_.item_id 
where
    orderitem0_.order_id=?
```
엔티티로 최적화 할 때와 다르게 BatchSize가 적용이 되지 않습니다. 


### range 조회  
장점 : 
1. 범위 조회거나 페이징시 성능이 제일 우수하다.  

단점 :  
1. eq_ref 방식보다 코드가 복잡하다.
2. `1+1`조회가 발생한다.

```Java
public List<OrderQueryDto> findAllByDto_optimization() {

    //루트 조회(toOne 코드를 모두 한번에 조회)
    List<OrderQueryDto> result = findOrders();

    //orderItem 컬렉션을 MAP 한방에 조회
    Map<Long, List<OrderItemQueryDto>> orderItemMap = findOrderItemMap(toOrderIds(result));

    //루프를 돌면서 컬렉션 추가(추가 쿼리 실행X)
    result.forEach(o -> o.setOrderItems(orderItemMap.get(o.getOrderId())));

    return result;
}

private Map<Long, List<OrderItemQueryDto>> findOrderItemMap(List<Long> orderIds) {
    List<OrderItemQueryDto> orderItems = em.createQuery(
            "select new jpabook.jpashop.repository.order.query.OrderItemQueryDto(oi.order.id, i.name, oi.orderPrice, oi.count)" +
                    " from OrderItem oi" +
                    " join oi.item i" +
                    " where oi.order.id in :orderIds", OrderItemQueryDto.class)
            .setParameter("orderIds", orderIds)
            .getResultList();

    return orderItems.stream()
            .collect(Collectors.groupingBy(OrderItemQueryDto::getOrderId));
}
```
Map으로 반환하는 이유는 `Map<Order_Id,List<OrderIteam>>`으로 만들어서   
`result.forEach(o -> o.setOrderItems(orderItemMap.get(o.getOrderId())));`  
OrderDto에 반복문을 돌아서 꺼내서 사용할 수 있게 `stream.map()`을 사용합니다.   
+ equals 방식과 다르게 `where in(...)`으로 조회를 하기 때문에 범위나 페이징이 가능합니다.    

#### 스프링 부트 3.1 이상 하이버네이트 6.2 버전 이상
스프링 부트 3.1 부터는 하이버네이트 6.2를 사용한다.  
하이버네이트 6.2 부터는 where in 대신에 array_contains 를 사용합니다.  

where in 사용 문법
```
array_contains 사용 문법
where item.item_id in(?,?,?,?)
where array_contains(?,item.item_id)
```
참고로 where in 에서 array_contains 를 사용하도록 변경해도 결과는 완전히 동일하다.  
그런데 이렇게 변경하는 이유는 성능 최적화 때문입니다.  
  
SQL 구문을 JPA는 캐싱해서 사용합니다.  
+ WHERE IN(...)
    ```Java
    where item.item_id in(?)
    where item.item_id in(?,?)
    where item.item_id in(?,?,?,?)
    ```  
  SQL 입장에서는 `?` 로 바인딩 되는 숫자 자체가 다르기 때문에 완전히 다른 SQL이다. 따라서 총 3개의 SQL 구문이 만
  들어지고, 캐싱도 3개를 따로 해야한다. 이렇게 되면 성능 관점에서 좋지않습니다.
+ array_contains
    ```
    select ... where array_contains([1,2,3],item.item_id)
    select ... where array_contains(?,item.item_id)
    ```
  배열에 들어가는 데이터가 늘어도 SQL 구문 자체가 변하지 않는다. ? 에는 배열 하나만 들어가면 된다.
  이런 방법을 사용하면 앞서 이야기한 동적으로 늘어나는 SQL 구문을 걱정하지 않아도 된다.
  결과적으로 데이터가 동적으로 늘어나도 같은 SQL 구문을 그대로 사용해서 성능을 최적화 할 수 있습니다.  


### SQL 1번  
장점 :  
1. `SQL` 한 번으로 필요한 데이터를 가져올 수 있다.  
  
단점 : 
1. 조인 결과에 대해 중복 제거 로직이 필요하다.
2. 데이터가 많은 경우 페이징 처리가 필요한데 페이징 처리가 
    `1`이 아니라 `N`이 기준이 된다.
3. 생각보다 range 조회보다 성능이 빠르지 않다.  
  
```Java
/**
 * SQL 한번으로 모든 데이터 가져오기
 */
public List<OrderFlatDto> findAllByDto_flat() {
    return em.createQuery(
            "select new jpabook.jpashop.repository.order.query.OrderFlatDto(o.id, m.name, o.orderDate, o.status, d.address, i.name, oi.orderPrice, oi.count)" +
                    " from Order o" +
                    " join o.member m" +
                    " join o.delivery d" +
                    " join o.orderItems oi" +
                    " join oi.item i", OrderFlatDto.class)
            .getResultList();
}
@GetMapping("/api/v6/orders")
public List<OrderQueryDto> ordersV6() {
    List<OrderFlatDto> flats = orderQueryRepository.findAllByDto_flat();
    return flats.stream()
            .collect(groupingBy(o -> new OrderQueryDto(o.getOrderId(), o.getName(), o.getOrderDate(), o.getOrderStatus(), o.getAddress()),
                    mapping(o -> new OrderItemQueryDto(o.getOrderId(), o.getItemName(), o.getOrderPrice(), o.getCount()), toList())
            )).entrySet().stream()
            .map(e -> new OrderQueryDto(e.getKey().getOrderId(), e.getKey().getName(), e.getKey().getOrderDate(), e.getKey().getOrderStatus(), e.getKey().getAddress(), e.getValue()))
            .collect(toList());
}
```  
이제 반환된 값을 스트림을 통해서 매핑을 해야합니다. 
코드 가독성도 떨어지고 유지보수가 어렵습니다. 

## 권장 순서

1. 엔티티 조회 방식으로 우선 접근
   1.   페치조인으로 쿼리 수를 최적화
   2.   컬렉션 최적화
        1. 페이징 필요 hibernate.default_batch_fetch_size , @BatchSize 로 최적화
         2. 페이징 필요X    페치 조인 사용
2.   엔티티 조회 방식으로 해결이 안되면 DTO 조회 방식 사용
3.   DTO 조회 방식으로 해결이 안되면 NativeSQL or 스프링 JdbcTemplate  
  

> 참고: 엔티티 조회 방식은 페치 조인이나, hibernate.default_batch_fetch_size , @BatchSize 같
 이 코드를 거의 수정하지 않고, 옵션만 약간 변경해서, 다양한 성능 최적화를 시도할 수 있다. 반면에 DTO를 직접
 조회하는 방식은 성능을 최적화 하거나 성능 최적화 방식을 변경할 때 많은 코드를 변경해야 한다.   

>참고: 개발자는 성능 최적화와 코드 복잡도 사이에서 줄타기를 해야 한다. 항상 그런 것은 아니지만, 보통 성능 최
 적화는 단순한 코드를 복잡한 코드로 몰고간다.
 엔티티 조회 방식은 JPA가 많은 부분을 최적화 해주기 때문에, 단순한 코드를 유지하면서, 성능을 최적화 할 수
 있다.
 반면에 DTO 조회 방식은 SQL을 직접 다루는 것과 유사하기 때문에, 트레이드오프가 발생합니다.