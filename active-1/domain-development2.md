# JPA 활용 - 1 하편

_**목차**_
1. Entity 내부에 비즈니스 로직을 추가하기
2. 도메인 모델 패턴과 트랜잭션 스크립트 패턴 구분하기
3. 두 패턴의 테스트 코드 비교하기
4. JPA Cascade 옵션
   1. SQL Mapper로 구현할 경우
   2. JPA로 구현할 경우
5. 테스트
6. Entity 메서드와 Controller에 대한 강사님 생각

## 1. Entity 내부에 비즈니스 로직 추가
+ 기본 칼럼
```java
public class Order {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "order_id")
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "member_id") // FK 칼럼이기때문에
    private Member member;

    @OneToMany(mappedBy = "order",cascade = CascadeType.ALL)
    private List<OrderItem> orderItems = new ArrayList<>();

    @OneToOne(fetch = FetchType.LAZY,cascade = CascadeType.ALL)
    @JoinColumn(name = "delivery_id")
    private Delivery delivery;

    private LocalDateTime orderDate;

    @Enumerated(EnumType.STRING)
    private OrderStatus status;
    
}
```  
+ 전체 주문 가격 조회 로직
```java
    //== 조회 로직 ==/
public int getTotalPrice() {
//        int totalPrice = 0;
//        for (OrderItem orderItem : orderItems) {
//            totalPrice += orderItem.getOrderPrice() * orderItem.getCount();
//        }
     int sum = orderItems.stream()
     .mapToInt(OrderItem::getTotalPrice)
     .sum();
     return sum;
}
```
+ 강사님 생성자 메서드 코드
```java
//=== 생성자 메서드 ===//
public static Order createOrder(Member member,Delivery delvery,OrderItem... orderItem){
    Order order = new Order();
    order.setMember(member);
    order.setDelivery(delivery);
    for(OrderItem orderItem: OrderItems){
        order.addOrderItem(orderItem);    
    }
    order.setStatus(OrderStatus.ORDER);
    order.setOrderDate(LocalDateTime.now());
    return order;
}
```
강사님께서는 주문을 생성할 때 정적 팩토리 매서드를 사용했습니다.  
단점은 기본 생성자 메서드에서 발생할 수 있는 문제는 동일합니다.
```java
new Member(String firstName,String lastName){}
public Member Member.create(String firstName,String lastName){}
String firstName = "존";
String lastName = "박";
new Member(lastName,firstName);
Member.create(lastName,firstName);
```
하지만, 정적 팩토리 매서드는  
프로덕션 코드에서는 생성자나 빌더 보다는 행간의 의미와 생성 의도를 드러낼 수 있기에  
생성자 메서드보다 정적 팩토리 매서드를 사용하는게 좋습니다.

```java
// Order.class
private int totalPrice;

// 주문 Entity 생성자 함수
@Builder
private Order(Member member, Delivery delivery, LocalDateTime localDateTime, List<OrderItem> orderItems, OrderStatus orderStatus) {
   this.member = member;
   this.delivery = delivery;
   this.orderDate = localDateTime;
   this.orderItems = orderItems;
   this.status = orderStatus;
   this.totalPrice = getTotalPrice();
}

//== 생성자 메서드 ==//
public static Order createOrder(Member member, Delivery delivery, ClockHolder clockHolder, OrderItem... orderItems) {
   return Order.builder().member(member)
   .delivery(delivery)
   .localDateTime(LocalDateTime.now(clockHolder.getClock()))
   .orderItems(List.of(orderItems))
   .orderStatus(OrderStatus.ORDER)
   .build();
}
```
Order 테이블은 **전체 주문 금액을 저장하는 칼럼**이 없습니다.  
주문 정보를 조회할 때마다 전체 주문 금액을 얻기 위해 해당 비즈니스 로직을 수행해야 합니다.  
차라리 전체 주문 금액을 저장하는 칼럼을 만들고,주문이 생성되면 처음 초기화 할 때  
저장한다면 비즈니스 로직을 수행할 필요가 없다고 생각해서 **추가했습니다.**  
> totalPrice과 같은 파생 칼럼 사용시 주의점  
> 주문상품 최종 금액의 내용이 변경이 많다면 주문상품이 바뀔때  
> totalPrice를 새로 계산해서 입력하는 과정이 오히려 리소스를 더 차지할 수도 있습니다.   
> 또, 파생속성이 데이터 무결성을 위반할 여지를 남겨두는 일이라며 사용하지 말아야 한다고  
> 주장하시는 개발자분도 있습니다.  
 
상황에 따라 알맞는 방법을 선택을 하면 됩니다.  

### 주문 상품 엔티티 설계  

```java
package jpabook.jpashop.domain;

import jpabook.jpashop.domain.item.Item;
import lombok.*;

import javax.persistence.*;

@Entity
@Getter
@Setter
@Table(name = "order_item")
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class OrderItem {

    @Id
    @GeneratedValue
    @Column(name = "order_item_id")
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "item_id")
    private Item item;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "order_id")
    private Order order;

    private int orderPrice;
    private int count;


    @Builder
    private OrderItem(Item item, int orderPrice, int count) {
        this.item = item;
        this.orderPrice = orderPrice;
        this.count = count;
        item.removeStock(count);
    }
        /** 생성 메서드 */
    public static OrderItem createOrderItem(Item item, int orderPrice, int count) {
        return OrderItem.builder()
                .item(item)
                .orderPrice(orderPrice)
                .count(count)
                .build();
    }

    public void cancel() {
        getItem().addQuantity(count);
    }

    //== 조회 로직 ==//
    public int getTotalPrice() {
        return getOrderPrice()*getCount();
    }
    
}
```
+ 기능 설명  
1. 생성 매서드 : 상품,가격, 수량 정보를 사용해서 주문 상품 엔티티를 생성한다.  
2. 주문 취소 : 비스니스 로직이 서비스 계층에서 처리하는게 아니라  
      엔티티 내부에서 처리할 수 있도록 설계를 했다.
   ```java
   item.removeStock(count);
   ```
3. 주문 가격 조회
   ```java
   //== 조회 로직 ==//
    public int getTotalPrice() {
        return getOrderPrice()*getCount();
    }
   //== 조회 로직 ==//
    public int getTotalPrice() {
        return this.orderPrice * this.count;
    }
   ``` 
### this.orderPrice * this.count 하지 않은 이유가 뭘까?   
+ 객체 외부에서는 당연히 필드에 직접 접근하면 안된다.
+ **객체 내부**에서는 필드에 직접 접근해도 아무런 **문제는 없다.**  
번거롭게 getXxx를 호출하는 것 보다는 필드를 직접 호출하는 것이 코드도 깔끔하다  
그런데 객체 내부에서 필드에 직접 접근하는 가,아니면 getter를 통해서 접근하는가가  
JPA 프록시를 많이 다루게 되면 중요해진다. 일반적으로 이런 상황을 겪을 일이 거의 없지만,  
조회한 프록시가 프록시 객체인 경우 필드에 직접 접근하면  
원본 객체를 가져오지 못하고 프록시 객체의 필드에 직접 접근하게 된다.  
equals,hashcode를 JPA 프록시 객체로 구현할 때 문제가 발생할 수 있다.  

<div style="text-align: center;">프록시 객체의 equals를 호출했는데 필드에 직접 접근하면,</br>
프록시 객체는 필드에 값이 없으므로 항상 null이 된디.</div>

### 주문 리포지토리 코드
```java
@Repository
@RequiredArgsConstructor
public class OrderRepository {

    private final EntityManager entityManager;

    public Long save(Order order) {
        entityManager.persist(order);
        return order.getId();
    }

    public Order findOne(Long id) {
        return entityManager.find(Order.class, id);
    }

//    public List<Order> findAll(OrderSearch orderSearch)
//
}
```
편하 테스트를 위해 save반환 값으로 Long 타입으로 지정해서 처리 했다.  

### 주문 서비스 계발
#### 서비스 코드
```java
package jpabook.jpashop.service;

import com.sun.xml.bind.v2.model.nav.Navigator;
import jpabook.jpashop.domain.*;
import jpabook.jpashop.domain.item.Item;
import jpabook.jpashop.repository.ItemRepository;
import jpabook.jpashop.repository.MemberRepository;
import jpabook.jpashop.repository.OrderRepository;
import jpabook.util.ClockHolder;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.List;

@Service
@Transactional(readOnly = true)
@RequiredArgsConstructor
public class OrderService {

    private final MemberRepository memberRepository;
    private final OrderRepository orderRepository;
    private final ItemRepository itemRepository;
    private final ClockHolder clockHolder;

    /**
     * 주문
     */
    @Transactional(readOnly = false)
    public Long order(Long memberId, Long itemId, int count) {
        //주문시간 홀더
        /**
         * 서비스의 역할은 필요한 리소스를 가져와서
         * 비즈니스 로직을 수행하는 엔티티 모델 객체에게 전달만 한다.
         * 만약 엔티티 모델 객체에게 전달하기 애마한 경우 서비스 계층에서 처리하지만
         * 만약 책임을 부여할 수 있다면 별도의 엔티티 클래스를 만들어서 처리를 하도록한다.
         */
        //엔티티 조회
        Member member = memberRepository.findOne(memberId);
        Item item = itemRepository.findOne(itemId);

        //배송정보 생성
        Delivery delivery = new Delivery();
        delivery.setAddress(member.getAddress());
        delivery.setStatus(DeliveryStatus.READY);

        //주문상품 생성
        OrderItem orderItem = OrderItem.createOrderItem(item, item.getPrice(), count);

        //주문생성
        Order order = Order.createOrder(member, delivery, clockHolder, orderItem);

        orderRepository.save(order);
        return order.getId();
    }

    /**
     * 주문취소
     */
    @Transactional(readOnly = false)
    public void cancelOrder(Long orderId) {
        //주문 엔티티 조회
        Order order = orderRepository.findOne(orderId);
        //주문취소
        order.cancel();
    }
    
}
```
+ 주문 (order()):  
    Cascade.All 을 활용하여 엔티티를 저장하지 않고, 최종 Order 엔티티만 저장하면  
    나머지 엔티티도 자동으로 DB에 반영이 된다.
+ 주문 취소(cancel()):  
    주문 식별자와 주문 엔티티를 조회하고 주문 엔티티에 취소요청을 보낸다.

## 2. 트랜잭션 스크립트 패턴 vs 도메인 모델 패턴 차이
+ 트랜잭션 스크립트 패턴
    ```java
    @Service
    @RequiredArgsConstructor
    @Transactional(readonly=true)
    class TransactionService {
        
        private final RepositoryA repositoryA;
        private final RepositoryB repositoryB
        private final RepositoryC repositoryC;
        
        @Transactional(readonly=false)
        public functionA(){
            A a = repositoryA.find();
            B b = repositoryB.find();
            C c = repositoryC.find();
  
            //START
            //비즈니스
            //END
            repositoryA.save(a);
        } 
    }
    ```
+ 도메인 모델 패턴
  ```java
    @Service
    @RequiredArgsConstructor
    @Transactional(readonly=true)
    class DomainService {
    
        private final RepositoryA repositoryA;
        private final RepositoryB repositoryB;
        private final RepositoryC repositoryC;
        
        public functionA(){
            A a = repositoryA.find();
            B b = repositoryB.find();
            C c = repositoryC.find();
            //도메인 객체 모델
            a.funcationA(b,c);
            repositoryA.save(a);
        } 
    }
  ```
### 테스트 코드에 의한 차이
+ 트랜잭션 스크립트 패턴
    ```java
    
    @SpringBootTest
    @Transaction
    public class ServiceTest {
        @Autowired
        private ServiceA repositoryA;
        @Autowired
        private RepositoryA repositoryA;
        @Autowired
        private RepositoryB repositoryB;
        @Autowired
        private RepositoryC repositoryC;
    
        @Test
        public void functionTest() {
            //given
            A a = new A();
            B a = new B();
            C a = new C();
            repositoryA.save(a);    
            repositoryB.save(b);    
            repositoryC.save(c);    
            
            //when
            serviceA.funtionA();
    
            //then
            assertThat(...)
        }
    }
    ```
+ 도메인 모델 객체 테스트시

    ```java
    public class testFinal {
        @Test
        public void functionAtest() {
            //given
            A a = new A();
            B a = new B();
            C a = new C();
            //when
            a.funtionnalA(b,e);
            
            //then
            assert(...);
        }
    }
    ```
### 트랜잭션 스크립트 패턴과 도메인 패턴 정리  
트랜잭션 스크립트 패턴의 장점은 빠른 개발이 가능하다.  
하나의 계층에 해당 비즈니스에 대한 로직을 처리하기 때문에 비즈니스가 복잡해질 경우  
비즈니스 로직도 같이 복잡해지고, 해당 계층이 두꺼워지면 유지보수하기도 어렵다.

도메인 패턴의 장점은 객체 지향을 기반하기 때문에  
해당 도메인을 재사용하고, 유지보수하기가 편하기 때문에 확장성에 좋다.  
다만, 모메인 모델을 구축하는데 많은 노력이 필요하다.  

테스트 코드에서도 차이점이 나타나는데  
도메인 패턴일 경우에는 불필요한 의존성이나 외부 네트워크를 사용하지 않기 때문에    
속도도 빠릅니다.  

### 트랜잭션 스크립트에서 도메인 패턴으로  
처음부터 도메인 패턴으로 작성하는 것보다 트랜잭션 스크립트 유형으로 코드를 작성하고  
전체 구조가 어느정도 윤곽이 잡힐 경우에 도메인에게 비즈니스 로직을 넣을 때  
필요한 의존성도 파악하기 쉽다고 생각이 듭니다.

### JPA Cascade 주의사항
```java
@OneToMany(mappedBy = "order",cascade = CascadeType.ALL)
private List<OrderItem> orderItems = new ArrayList<>();

@OneToOne(fetch = FetchType.LAZY,cascade = CascadeType.ALL)
@JoinColumn(name = "delivery_id")
private Delivery delivery;
```
1. 동일한 라이플 사이클을 가진 엔티티 관계(Order, OrderItem)여야 합니다.
2. 엔티티가 하나만 연관관계를 맺을때 Order,Delivery 처럼  
   Order는 Delivery만 사용하고 관리하는게 편하다.
3. 다른 것이 참조할수 없는 `private` Onwer인 경우에 사용을 한다.

-- 처음에는 사용할 `Cascade범위`를 생각하기 어렵다면 사용하지 않고 코드를 작성한다.  
-- 이후에 어느정도 엔티티의 라이프사이클이 생긴다면 그때 cascadeType을 지정해도 된다.

#### 1.JPA cascade기능을 SqlMapper로 구현한다면  
```java
Parent parent = new Parent("배달음식","희동");
Child hanam = new Child("삼겹살","하남돼지");
Child domino = new Child("피자","도미노");
Child cj = new Child("돈까스","백설");
parent.addMenu(listOf(hanam,domino,cj));
em.persist(parent);
```  
JPA로 작성한 코드를 간단하게 SqlMapper로 작성한다면 아래와 유사할겁니다.
```xml
-- MyBatis xml
<insert id="addParent" parameterType="Parent" useGeneratedKeys="true" keyProperty="parentKey">
    INSERT INTO parent_table(name, nikname) VALUES (#{parent.food}, #{parent.username})
</insert>
<insert id="addChild" parameterType="hashmap">
    INSERT INTO child_table (name, nikname, parent_id) VALUES (#{child.category},#{parentSeq});
</insert>
``` 
```java
private final ParentMapper parentMapper;
private final ChildMapper childMapper;
@Transactional
public void addMenu(Parent parent,List<Child> childList) {
    Long parentId= parentMapper.addParent(parent);
    for(int i = 0 ; i<childList.length; i++){
        childMapper.addChild(childList[i],parentId);
    }
}

```
여기에 `<foreach>`기능을 사용한다면 더 간단하게 작성할 수 있습니다,
핵심은 JPA는 상황에 따라 옵션에 Cascade옵션을 추가한다면  
불필요한 SQL를 작성하지 않고, 그리고 참조하는 순서도 JPA가 알아서 DB에 넣어준다.

## 테스트  

```java
@SpringBootTest
@Transactional
@Slf4j
class OrderServiceTest {

    @Autowired
    OrderService orderService;
    @Autowired
    OrderRepository orderRepository;
    @PersistenceContext
    EntityManager em;

    @Test
    @DisplayName("상품 주문이 되면 주문 수량만큼 재고가 줄어야한다.")
    void order() {
        //given
        Member member = createMember(new Address("서울시", "마포구", "노고산동"), "둘리");
        int orderItemCount = 3;
        Album album = createAlbum("또치야", 1000, orderItemCount);

        //when
        Long orderId = orderService.order(member.getId(), album.getId(), orderItemCount);

        //then
        Order findOrder = orderRepository.findOne(orderId);

        assertThat(album.getStockQuantity()).isZero();
        assertThat(findOrder.getTotalPrice()).isEqualTo(3000);
        assertThat(findOrder.getStatus()).isEqualByComparingTo(OrderStatus.ORDER);
    }

    @DisplayName("상품의 재고 수량보다 많은 수량을 주문할 경우 예외가 발생한다.")
    @Test
    void OrderWithEx(){
        //given
        Album album = createAlbum("탕후루", 1000, 3);
        Member member = createMember(new Address("신촌", "영덕", "대게"), "둘리");

        //when // then
        Assertions.assertThatCode(() -> orderService.order(member.getId(), album.getId(), 4))
                .isInstanceOf(NotEnoughStockException.class)
                .hasMessage("need more stock");
    }

    @DisplayName("주문을 취소할 경우 주문 수량만큼 상품의 재고 수량이 증가한다.")
    @Test
    void cancelOrder(){
        //given
        Album album = createAlbum("강아지풀", 1000, 3);
        Member member = createMember(new Address("서울시", "관악구", "논현동"), "또치");
        Long order = orderService.order(member.getId(), album.getId(), 3);

        //when
        orderService.cancelOrder(order);
        Order findOrder = orderRepository.findOne(order);

        //then
        Assertions.assertThat(album.getStockQuantity()).isNotZero();
        Assertions.assertThat(album.getStockQuantity()).isEqualByComparingTo(3);
        Assertions.assertThat(findOrder.getStatus()).isEqualByComparingTo(OrderStatus.CANCEL);
    }

    private Delivery createDelivery(Address address) {
        Delivery delivery = new Delivery();
        delivery.setAddress(address);
        return delivery;
    }

    private Member createMember(Address address, String name) {
        Member member = new Member();
        member.setName(name);
        member.setAddress(address);
        em.persist(member);
        return member;
    }

    private Album createAlbum(String name, int price, int quantity) {
        Album album = new Album();
        album.setStockQuantity(quantity);
        album.setName(name);
        album.setPrice(price);
        em.persist(album);
        return album;
    }

}
```
### Controller 계층에서 Entity의 사용은?
1. 화면을 그리는데 필요한 템플릿용 DTO(혹은 Model)과 데이터를 저장하는 Entity는 구분해서 사용한다.  
    이유는 DTO 역할을 하기위한 `@Valid` 어노테이션과 Entity 역할을 하기위한 `@Entity` 애노테이션이 하나의 클래스에 있다면   
   엔티티 역할을 고치기 위해서 수정했다가 화면에 올바르지 않는 데이터가 나온다거나    
   반대로 화면을 그리기 위해서 DTO 역할을 수정하면 데이터를 저장하는 정보가 원지않게 변경될 수 있다  
2. 준영속 상태에서 파라미터로 넘긴 엔티티는 데이터를 전달하는 VO(`불변 객체`)의 역할이고    
   데이터를 저장한다면  내부 로직은 em.merge(Member memberFake)가 실행이 된다.  
   변경 감지 기능을 사용하면 원하는 속성만 선택해서 변경할수 있지만, 병합을 사용하면 모든속성이 변경된다.  
   병합시 값이 없으면 `null`로 업데이트 할위험도 있다. (병합은모든필드를 교체한다.)  
3. 변경 감지이용시 set은 안된다.  
   해당 변경이 무슨 이유 때문에 수정되는지 알 수있는 메서드 명으로 변경해야한다.  
   예) member.setName(),set.age() -> member.updateInfomation() 이런 느낌  
    + 역 추적이 가능하다.  

4. 어설프게 DTO를 바로 객체에 전달하지 않고,  
   필요한 데이터만 가지고 서비스 계층에 넘길수 있도록해야한다.유지보수도 간단해지고, 명확해진다.
    ```java
    //controller
    public void addMember(@RequestBody CreateRequestMember createMember){
        memberService.saveMember(createMember.toServiceRequest())
    }
    //CreateRequestMember
    public class CreateRequestMember {
        //..생략
        MemberCreateServiceRequest toServiceRequest(){
            // 필요한 정보만 가지고 서비스용 DTO를 만든다.
        }        
    }
    ```


