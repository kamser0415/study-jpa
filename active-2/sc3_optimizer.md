# 쿼리 성능 최적화
이번 장에서는 지연로딩 및 즉시로딩으로 발생할 수 있는 문제를 
단계별로 해결하는 방법을 이해하는 것입니다.    

<!-- TOC -->
* [쿼리 성능 최적화](#쿼리-성능-최적화)
  * [Ver.1 Entity 그대로 사용하기](#ver1-entity-그대로-사용하기-)
    * [발생하는 문제](#발생하는-문제)
      * [java.lang.StackOverflowError: null](#javalangstackoverflowerror-null-)
      * [ByteBuddyInterceptor](#bytebuddyinterceptor)
  * [지연로딩 및 즉시로딩 N+1 발생문제](#지연로딩-및-즉시로딩-n1-발생문제-)
  * [fetch join 사용하기](#fetch-join-사용하기-)
    * [fetch 와 DTO](#fetch-와-dto-)
      * [트레이드오프](#트레이드오프-)
      * [DTO 주의사항](#dto-주의사항-)
  * [정리](#정리)
    * [쿼리 방식 선택 권장 순서](#쿼리-방식-선택-권장-순서)
<!-- TOC -->

## Ver.1 Entity 그대로 사용하기  
+ **Entity**
    ```Java
    public class Order {
        @Id @GeneratedValue
        private Long id;
    
        @ManyToOne(fetch = LAZY)
        private Member member;
    
        @OneToOne(fetch = LAZY, cascade = CascadeType.ALL)
        private Delivery delivery;
   }
   public class Member {
        //..생략
        @OneToMany(mappedBy = "member")
        private List<Order> orders = new ArrayList<>();
    }
    ```  
+ **코드**  
    ```Java
    @GetMapping("/api/v1/orders")
    public List<Order> ordersV1() {
        List<Order> all = orderRepository.findAll();
        return all;
    }
    ```  
    ```Java
    public List<Order> findAll() {
        return em.createQuery("select o from Order o", Order.class)
                .getResultList();
    }
    ```
  
### 발생하는 문제
#### java.lang.StackOverflowError: null  
양방향 연관관계 매핑으로 인해 순환참조로 오류가 발생합니다. 
직렬화를 할때 order.member -> member.orders > order.member ... 
로 순환 참조로 무한 루프에 빠지게 됩니다.  

**해결방법**  
1. 관련된 양방향 연관관계 엔티티 한쪽에 `@JsonIgnore`사용

#### ByteBuddyInterceptor
지연 로딩일 경우 Order 엔티티만 먼저 읽어오고 추가 조회가 없으면 
필드는 프록시 객체 참조를 가지고 있습니다. 
직렬화를 할 경우 프록시 객체는 직렬화를 하기 위한 필요한 메서드나 애노테이션이 구현되어있지 않습니다. 

**해결방법**
1. Hibernate5Module 등록
    ```Gradle
    //build.gradle
    implementation 'com.fasterxml.jackson.datatype:jackson-datatype-hibernate5'
    // 스피링부트 3.0 이상
    implementation 'com.fasterxml.jackson.datatype:jackson-datatype-hibernate5- 
    jakarta'
    ```
    ```Java  
    @Bean
    Hibernate5Module hibernate5Module() {
        Hibernate5Module hibernate5Module = new Hibernate5Module();
        //강제 지연 로딩 설정
        hibernate5Module.configure(Hibernate5Module.Feature.FORCE_LAZY_LOADING, true);
        return hibernate5Module;
    }
    ```  
    주의: 스프링 부트 3.0 이상이면 Hibernate5Module 대신에 Hibernate5JakartaModule 을 사용해야
    한다.  
 
    프록시 객체를 직렬화 할 때, 지연로딩된 경우 강제 로딩으로 데이터를 조회합니다. 
    강제로딩이 아니라 null로 매핑하고 싶은 경우 `hibernate5Module.configure`구문을 주석처리합니다.  
    그리고 로딩이 필요한 연관관계는 조회 메소드를 통해서 초기화를 진행합니다.
    ```Java
    for (Order order : all) {
        order.getMember().getName(); //Lazy 강제 초기화
        order.getDelivery().getAddress(); //Lazy 강제 초기환
        List<OrderItem> orderItems = order.getOrderItems();
        orderItems.stream().forEach(o -> o.getItem().getName()); //Lazy 강제 초기화
    }
    ```  
**정리**  
- Hibernate5Module 모듈 등록, LAZY=null 처리  
- 양방향 관계 문제 발생 -> @JsonIgnore  

**_절대 엔티티를 그대로 반환 및 매핑해서 사용하면 안됩니다_**  

## 지연로딩 및 즉시로딩 N+1 발생문제  
JPA API가 제공하는 `find()` 메소드는 최적화가 되어있습니다. 
그래서 필요한 엔티티를 즉시로딩으로 설정하면 `JOIN`을 통해 한꺼번에 읽어옵니다.  
  
하지만 `JPQL`은 SQL로 파싱되어 DB에서 실행됩니다. 
그 후에 Entity에 추가된 로딩타입을 확인하고 다시 SQL을 조회합니다.  

```Java
String sql = "select o from Order as o join o.member as m join o.delivery as d";
em.createQuery(sql,Order.class)
  .getResultList();
```
+ **SQL(member,delivery 즉시 로딩일 경우)**
```SQL
// 1
select
    order0_.*
from
    orders order0_ 
inner join
    member member1_ 
        on order0_.member_id=member1_.member_id 
inner join
    delivery delivery2_ 
        on order0_.delivery_id=delivery2_.delivery_id
// 2        
select
    order0_.*,
    delivery1_.*,
    member2_.*
from
    orders order0_ 
left outer join
    delivery delivery1_ 
        on order0_.delivery_id=delivery1_.delivery_id 
left outer join
    member member2_ 
        on order0_.member_id=member2_.member_id 
where
    order0_.delivery_id=?
```  
즉시 로딩, 지연 로딩의 공통점은 먼저 주 테이블 한번 읽어옵니다. 
`1`번 쿼리를 실행하고 엔티티에 매핑할 때 지연로딩은 프록시 객체를 넣어주고 
즉시로딩은 다시 SQL을 실행해서 추가 쿼리가 발생합니다.  
  
## fetch join 사용하기  
`N+1`문제는 fetch join을 사용해서 해결합니다. 
회원 엔티티와 관련된 연관관계 엔티티를 같이 조회해야할 경우 사용합니다.
```Java
public List<Order> findAllWithMemberDelivery() {
    return em.createQuery(
            "select o from Order o" +
                    " join fetch o.member m" +
                    " join fetch o.delivery d", Order.class)
            .getResultList();
}
```
해당 방법으로 조회할 경우 SQL은 아래와 같이 실행됩니다.
```SQL
select
    order0_.*,
    delivery1_.*,
    member2_.*
from
    orders order0_ 
left outer join
    delivery delivery1_ 
        on order0_.delivery_id=delivery1_.delivery_id 
left outer join
    member member2_ 
        on order0_.member_id=member2_.member_id 
where
    order0_.delivery_id=?
```
페치 조인은 객체 그래프를 활용할 수 있게 데이터의 일부를 필터링하여 가져오지 않습니다. 
연관관계 엔티티와 가지고 있는 필드를 전부 읽어서 매핑을 해줍니다.  
  
### fetch 와 DTO  
조인을 통해서 읽어오는 엔티티의 필드가 많지 않거나 복잡하지 않다면 페치 조인은 충분한 최적화가 됩니다. 
다만 조인 테이블이 많거나 필드가 20여개가 넘어가서 불필요한 데이터도 읽어야하는 경우가 발생합니다.  
이럴때 사용할 수 있는 방법이 `DTO class`를 통해 매핑할 수 있습니다  
  
+ DTO class
    ```Java
    @Data
    public class OrderSimpleQueryDto {
    
        private Long orderId;
        private String name;
        private LocalDateTime orderDate;
        private OrderStatus orderStatus;
        private Address address;
    
        public OrderSimpleQueryDto(Long orderId, String name, LocalDateTime orderDate, OrderStatus orderStatus, Address address) {
            this.orderId = orderId;
            this.name = name;
            this.orderDate = orderDate;
            this.orderStatus = orderStatus;
            this.address = address;
        }
    }
    ```  
+ 레포지토리 코드  
    ```Java
    public List<OrderSimpleQueryDto> findOrderDtos() {
        return em.createQuery(
                "select new jpabook.jpashop.repository.order.simplequery.OrderSimpleQueryDto(o.id, m.name, o.orderDate, o.status, d.address)" +
                        " from Order o" +
                        " join o.member m" +
                        " join o.delivery d", OrderSimpleQueryDto.class)
                .getResultList();
    }
    ```  
+ **SQL**
    ```SQL
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
    ```  
    
필요한 필드 데이터만 조회하고 DTO 클래스에 매핑해서 반환하는 방법입니다. 
  
#### 트레이드오프  
1. **재사용성**  
    엔티티를 반환하는 경우 가공되지 않았기 때문에 재사용할 수 있지만, 
    DTO로 매핑할 경우 가공이 되어 재사용하기 어렵습니다.  
  
2. **성능**  
    조회 칼럼이 많이 없을 경우에는 차이가 미비합니다. 다만, 조회 칼럼이 20개가 넘어가거나 
    트레픽이 많은 환경, 사용자가 반복 클릭을 하는 경우등 많은 트레픽을 유발하는 경우 
    유의미한 차이가 발생합니다.  

3. **영속성 컨텍스트 관리여부**  
    엔티티를 반환하는 경우 `변경 감지 기능`이나 `cascade`등 영속성 컨텍스트 기능을 활용할 수 있지만, 
    DTO는 영속성 컨텍스트에서 관리대상이 아닙니다. 

4. **코드 가독성**  
    엔티티를 반환하는 경우 `JPQL`이 단순하지만, DTO를 반환하는 경우 
    코드 가독성이 떨어질 수 있습니다.

#### DTO 주의사항  
+ **API에 매칭된 DTO입니다.**  
    엔티티의 경우 API가 변경되어도 수정하는 DTO는 API 관련 클래스를 수정하지만, 
    DTO로 매핑하는 경우 API 변경시 같이 수정이 됩니다. 의존관계는 없지만 암묵적인 의존이 발생합니다.  

DTO로 매핑해서 반환하는 메소드와 엔티티를 반환하는 메소드를 같은 레포지토리에서 관리하는 것보다 
별도의 패키지로 분리해서 관리하는 방식이 유지보수할 때 편합니다.  
```text
- jpashop
    - repository
        - order
            - simplequery
                - OrderSimpleQueryDto.java
                - OrderSimpleQueryRepository.java
        - OrderRepository.java
```  
위와 같은 패키지구조로 순수 엔티티를 사용하는 레포지토리와 
DTO로 종속된 SQL을 사용하는 통계 쿼리나 API에 최적화된 쿼리를 사용하는 레포지토리를 
별도의 패키지로 분류하여 관리합니다.  

## 정리
엔티티를 DTO로 변환하거나, DTO로 바로 조회하는 두가지 방법은 각각 장단점이 있다. 둘중 상황에 따라서 더 나은
방법을 선택하면 된다. 엔티티로 조회하면 리포지토리 재사용성도 좋고, 개발도 단순해진다. 따라서 권장하는 방법은 다
음과 같다.  

### 쿼리 방식 선택 권장 순서
1. 우선 엔티티를 DTO로 변환하는 방법을 선택한다.
2. 필요하면 페치 조인으로 성능을 최적화 한다.    
   대부분의 성능 이슈가 해결된다.
3. 그래도 안되면 DTO로 직접 조회하는 방법을 사용한다.
4. 최후의 방법은 JPA가 제공하는 네이티브 SQL이나 스프링 JDBC Template을 사용해서 SQL을 직접 사용한다.
