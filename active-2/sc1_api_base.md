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
```text
jpashop
├── controller
│   ├── (rendering controller 관련 클래스들)
│
└── api (new)
    ├── (api 관련 클래스들)
```

패키지나 구성 단위로 공통으로 예외를 처리를 할때 `API`와 `HTML`은 처리 방식이 다릅니다.

예를 들면:
+ API : 공통 에러용 JSON API 반환
+ HTML : 공통 에러 HTML 반환

현재 `controller`패키지는 클라이언트에 전달할 `html`을 랜더링해서 반환합니다. 
이제 클라이언트나 다른 서버쪽에 `json api`를 통해서 데이터를 주고 받아야 한다면 
별도의 패키지를 분리하는 것이 좋습니다.  

## 요청/응답 DTO를 별도로 만듭니다.
### 요청/응답을 Entity를 사용할 경우
클라이언트의 요청과 응답을 받을 때 `Entity`로 받고 전달할 수 있습니다.  

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
+ 엔티티에 프레젠테이션 계층을 위한 로직이 추가된다.(`@JsonIgnore`등)
+ 엔티티에 API 검증을 위한 로직이 들어간다. (`@NotEmpty` 등등)
+ 실무에서는 회원 엔티티를 위한 API가 다양하게 만들어지는데, 한 엔티티에 각각의 API를 위한 모든 요청 요구사항을 담기는 어렵다.
    > 저장,수정,삭제마다 필수와 옵션을 확인하려면 API문서를 확인하거나 해당 로직을 확인해야합니다. 
추가로 불필요한 데이터로 인해 사이드 이펙트가 발생 할 수 있습니다.
+ 엔티티가 변경되면 API 스펙이 변한다
  > 화면에서는 member_id,user_name을 사용하기로 약속을 했습니다. 
엔티티가 변경되면 화면에서 사용하기로한 필드명도 같이 변경해야하는 일이 발생합니다.
+ 테이블의 스키마 정보가 노출된다(칼럼명,테이블 구조)  
    > 회원 정보만 요청을 했지만, 연관관계 정보도 같이 반환이 됩니다.
+ 양방향 의존관계 발생  
    > 컨트롤러 -> 서비스 -> 레포지토리(엔티티) 사용 구조에서 `@JsonIgnore`나`@NotEmpty` 사용시 
엔티티 -> 컨트롤러(`Persentation`) 계층을 의존하게 됩니다.  
의존한다는 이야기는 해당 `@JsonIgnore`를 사용하려면 컨트롤러 계층에서 사용할 수 있습니다.


### 요청/응답을 DTO 사용하기
엔티티를 위한 API가 다양하게 만들어지는데, 
한 엔티티에 각각의 API를 위한 모든 요청 요구사항을 담기는 어렵다. 
- 엔티티가 변경되면 API 스펙이 변한다. 
- API 요청 스펙에 맞추어 별도의 DTO를 파라미터로 받는다.

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

이렇게 별도의 DTO를 만들어 반환할 경우 `API 문서`를 확인하지 않아도
해당 DTO를 봐도 어떤 값이 들어오고 필요한지 알 수 있습니다.
만약 `Member`엔티티를 그대로 사용했다면 어떤 값이 들어오는지
그리고 서비스 계층에서 데이터가 채워져서 로직이 실행되는지 알 수가 없습니다.
  
현재 반환은 요청에 대한 결과만 반환하고 있습니다.
```JSON
{
  "id": 1,
  "name": "둘리"
}
```  
### API 스펙이 1:1 매핑되는 DTO를 만든다.
만약 회원정보 리스트나, 수정된 결과가 몇개인지 등 추가 요구사항이 발생할 경우 
수정하기가 어렵기 때문에 API전용 매핑 클래스를 만듭니다.  
```Java
@Data
@AllArgsConstructor
static class Result<T> {
    private T data;
}
``` 
데이터 전송 클래스이기 때문에 엔티티와 다르게 setter/setter에 대해 자유롭습니다.  

```JSON
{
  "date": {
    //...
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
