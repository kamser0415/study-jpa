# API용 Controller 기본
<!-- TOC -->
* [API용 Controller 기본](#api용-controller-기본)
      * [현재 패키지 구성 방식](#현재-패키지-구성-방식-)
  * [요청/응답 DTO를 별도로 만듭니다.](#요청응답-dto를-별도로-만듭니다)
    * [요청/응답을 Entity를 사용할 경우](#요청응답을-entity를-사용할-경우)
      * [Entity를 사용시 발생할 수 있는 문제](#entity를-사용시-발생할-수-있는-문제)
    * [요청/응답을 DTO 사용하기](#요청응답을-dto-사용하기)
      * [회원 추가 DTO 생성 로직](#회원-추가-dto-생성-로직)
      * [회원 수정 DTO 생성 로직](#회원-수정-dto-생성-로직)
    * [API 스펙이 1:1 매핑되는 DTO를 만든다.](#api-스펙이-11-매핑되는-dto를-만든다)
    * [CQRS](#cqrs)
<!-- TOC -->

#### 현재 패키지 구성 방식  
```text
jpashop
│
├── domain
│   ├── (domain 관련 클래스들)
│
├── exception
│   ├── (exception 관련 클래스들)
│
├── service
│   ├── (service 관련 클래스들)
│
├── repository
│   ├── (repository 관련 클래스들)
│
├── controller
│   ├── (rendering controller 관련 클래스들)
```  
패키지나 구성 단위로 공통으로 예외를 처리를 할때 `API`와 `HTML`은 처리 방식이 다릅니다.

예를 들면:
+ API : 공통 에러용 JSON API 반환  
  API에 맞는 응답 DTO와 같은 패키지로 관리하는 방식이 유지보수에 유리합니다.  
+ HTML : 공통 에러 HTML 반환  
  에러 화면 HTML,응답 HTML 반환 및 뒤에 사용할 OSIV등 활용으로 API와 분리

현재 `controller`패키지는 클라이언트에 전달할 `html`을 랜더링해서 반환합니다. 
이제 클라이언트나 다른 서버쪽에 `API HTTP Json`을 통해서 데이터를 주고 받아야 한다면 
별도의 패키지를 아래와 같이 분리하는 것이 좋습니다.
```text
jpashop
├── controller
│   ├── (rendering controller 관련 클래스들)
│   
└── api (new)
    ├── (api controller)
    ├── (api package
```   
데이터 전송 오브젝트 DTO는 계층별로 나누어 사용합니다.  

**예를들어:**  
>컨트롤러 > 서비스 > 레포지토리 구조 사이에 데이터 전송 객체가 들어갑니다.
>1. 클라언트의 요청을 컨트롤러에게 전달할 때 사용하는 데이터를 전송하는 DTO
>2. 컨트롤러와 서비스 사이에서 전달되는 데이터 전송하는 DTO
>3. 서비스와 레포지토리 사이에 전달되는 데이터 Entity
API를 사용하는 프레젠테이션 계층과 서비스 계층을 별도로 분리하는 방식으로 구조를 만들 수 있습니다.  
  
현재 패키지는 1번까지 적용했습니다. 
컨트롤러와 서비스 계층에서 데이터 전송을 하는데 DTO가 별도로 필요한 이유는  
1. 서비스 계층에 전달할 매개변수를 한꺼번에 전달할 수 있는 객체가 필요합니다.  
2. 서비스 계층은 POJO로 순수 자바로 이루어지는게 좋습니다.  
  현재 컨트롤러가 서비스 계층에서 사용하지 않은 불필요한 애노테이션과 의존성이 있습니다.  
3. 컨트롤러가 사용하는 DTO인데 서비스 계층이 의존하게 됩니다.  
  회원 가입 서비스를 이용하려면 A 컨트롤러에서 사용하는 DTO를 다른 B 컨트롤러에서도 맞춰서 사용해야합니다.  
  
아래 구조로 계층별로 사용할 DTO를 분리하는 것이 애플리케이션이 커진다면 유지보수하기 쉽습니다.
```Java
sample
│
└── hello
    │
    └── spring
        │
        └── api
            │
            ├── controller
            │   │
            │   ├── order
            │   │   │
            │   │   ├── request
            │   │   │   └── OrderCreateRequest.java
            │   │   │
            │   │   └── OrderController.java
            │   │
            │   ├── product
            │   │   │
            │   │   ├── dto
            │   │   │   │
            │   │   │   └── request
            │   │   │       └── ProductCreateRequest.java
            │   │   │
            │   │   └── ProductController.java
            │   │
            │   └── ApiControllerAdvice.java
            │
            └── service
                │
                ├── order
                │   │
                │   ├── request
                │   │   └── OrderCreateServiceRequest.java
                │   │
                │   ├── response
                │   │   └── OrderResponse.java
                │   │
                │   ├── OrderService.java
                │   │
                │   └── OrderStatisticsService.java
                │
                └── product
                    │   //product 유틸리티
                    ├── ProductNumberFactory.java 
                    ├── ProductService.java
                    │   //서비스용 DTO
                    ├── request
                    │   └── ProductCreateServiceRequest.java
                    │   //서비스용 DTO
                    └── response
                        └── ProductResponse.java
```

## 요청/응답 DTO를 별도로 만듭니다.
### 요청/응답을 Entity를 사용할 경우
클라이언트의 요청과 응답을 받을 때 `Entity`클래스를 사용할 수 있습니다  

```Java
// api/memberController
@PostMapping("/api/v1/members")
public Member saveMemberV1(@RequestBody @Valid Member member) {
    Long id = memberService.join(member);
    return memberService.find(id);
}
// Member.class
@Entity
public class Member {
    //.. 
    @NotEmpty(message="lalala")
    private String name;
    //...
}
```

`Entity`를 통해서 API json을 매핑하고, 응답으로 반환할 경우 다음과 같은 문제가 발생할 수 있습니다.  
  
#### Entity를 사용시 발생할 수 있는 문제
+ 엔티티에 컨트롤러 계층을 위한 로직이 추가된다.(`@JsonIgnore`등)
+ 엔티티에 API 검증을 위한 로직이 들어간다. (`@NotEmpty` 등등)
+ 엔티티 필드만 보고 비즈니스 로직에 필요한 값이나 로직 내에 추가되는 값을 확인할 수 없습니다. 
    > 저장,수정,삭제마다 필수와 옵션을 확인하려면 API문서를 확인하거나 해당 로직을 확인해야합니다. 
추가로 불필요한 데이터로 인해 사이드 이펙트가 발생 할 수 있습니다.
+ 엔티티가 변경되면 API 스펙이 변합니다.
  > 화면에서는 member_id,user_name을 사용하기로 약속을 했습니다. 
엔티티가 변경되면 화면에서 사용하기로한 필드명도 같이 변경해야하는 일이 발생합니다.
+ 테이블의 스키마 정보가 노출된다(칼럼명,테이블 구조)  
    > 회원 정보만 요청을 했지만, 연관관계 정보도 같이 반환이 됩니다. 
  만약 프론트에서 그 정보를 사용한다면 수정하기도 어렵습니다.
+ 양방향 의존관계 발생  
    > 컨트롤러 -> 서비스 -> 레포지토리(엔티티) 사용 구조에서 `@JsonIgnore`나`@NotEmpty` 사용시 
엔티티 -> 컨트롤러(`Persentation`) 계층을 의존하게 됩니다.  
의존한다는 이야기는 해당 `@JsonIgnore`를 사용하려면 컨트롤러 계층에서 사용할 수 있습니다.


### 요청/응답을 DTO 사용하기
엔티티로 요청/응답 객체로 사용하기에는 유지보수가 어렵습니다. 
- 엔티티가 변경되면 API 스펙이 변한다. 
- API 요청 스펙에 맞추어 별도의 DTO를 파라미터로 받는다.  

`API`와 1:1매핑하는 별도의 데이터 전송 객체(DTO)를 만들어서 관리합니다. 
#### 회원 추가 DTO 생성 로직
+ Controller
    ```Java
    @PostMapping("/api/v2/members")
    public CreateMemberResponse saveMemberV2(@RequestBody @Valid CreateMemberRequest request) {
    
        Member member = new Member();
        member.setName(request.getName());
    
        Long id = memberService.join(member);
        return new CreateMemberResponse(id);
    }
    ```
+ DTO  
  API 요청과 응답에 필요한 정보만 전송하는 클래스를 생성합니다. 
    ```Java
    @Data
    static class CreateMemberResponse {
        private Long id;
        public CreateMemberResponse(Long id) {
            this.id = id;
        }
    }
    @Data
    static class CreateMemberRequest {
        private String name;
    }
    ```  

#### 회원 수정 DTO 생성 로직
+ Controller
    ```Java
    @PutMapping("/api/v2/members/{id}")
    public UpdateMemberResponse updateMemberV2(@PathVariable("id") Long id, @RequestBody @Valid UpdateMemberRequest request) {
        memberService.update(id, request.getName());
        Member findMember = memberService.findOne(id);
        return new UpdateMemberResponse(findMember.getId(), findMember.getName());
    }
    ```
+ DTO
    ```Java
    @Data
    static class UpdateMemberRequest {
        private String name;
    }
    
    @Data
    @AllArgsConstructor
    static class UpdateMemberResponse {
        private Long id;
        private String name;
    }
    ```  

`API 문서`를 확인하지 않아도 Controller DTO의 필드를 보면 API 요청/응답에 필요한 필드를 알 수 있습니다.

```JSON
{
  "id": 1,
  "name": "둘리"
}
```  
### API 스펙이 1:1 매핑되는 DTO를 만든다.
API 응답 객체로 DTO를 그대로 반환할 경우 추가 정보를 넣거나 가공하려면 
DTO 클래스도 불필요한 필드가 필요하게 됩니다. 
DTO wrapper class를 만들어 요청/응답에 변경과 `API`를 위한 별도의 필드도 추가하여 전달합니다.
```Java
@Data
@AllArgsConstructor
static class Result<T> {
    private T data;
}
``` 
데이터 전송 클래스이기 때문에 엔티티와 다르게 setter/setter에 대해 자유롭습니다. 
제어자를 private을 사용해서 필드 값을 얻거나 수정하려면  getter/setter 추가 생성해서 만들어야합니다. 
DTO 클래스는 데이터 변경이나 수정에 자유롭기 때문에 public으로 선언도 가능합니다.

```JSON
{
  "date": {
    "name": "둘리"
  }
}
```  

### CQRS

```Java
@PutMapping("/api/v2/members/{id}")
public UpdateMemberResponse updateMemberV2(@PathVariable("id") Long id, @RequestBody @Valid UpdateMemberRequest request) {
    Meber findMember = memberService.update(id, request.getName());
    return new UpdateMemberResponse(findMember.getId(), findMember.getName());
}
```  
위 코드처럼 수정한 엔티티를 그대로 반환하는 서비스 로직은 수정,조회 쿼리가 같이 있는 방식입니다.  
하지만 수정과 조회 쿼리를 분리하는 방식이 유지보수가 편합니다.

```Java
@PutMapping("/api/v2/members/{id}")
public UpdateMemberResponse updateMemberV2(@PathVariable("id") Long id, @RequestBody @Valid UpdateMemberRequest request) {
    memberService.update(id, request.getName());   //수정 
    Member findMember = memberService.findOne(id); //조회 (Query)
    return new UpdateMemberResponse(findMember.getId(), findMember.getName());
}
```  