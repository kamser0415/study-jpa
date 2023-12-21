# 스프링 부트 JPA 스터디
## 실전! 스프링 부트와 JPA 활용 1
+ [회원 도메인까지 - 상편](https://github.com/kamser0415/study-jpa/blob/main/active-1/domain-development.md)
+ [상품부터 마지막까지 - 하편](https://github.com/kamser0415/study-jpa/blob/main/active-1/domain-development2.md)

## 실전! 스프링 부트와 JPA 활용 2
+ [API 패지키가 추가되는 이유](https://github.com/kamser0415/study-jpa/blob/main/active-2/sc1_api_base.md#%EC%9A%94%EC%B2%AD%EC%9D%91%EB%8B%B5%EC%9D%84-entity%EB%A5%BC-%EC%82%AC%EC%9A%A9%ED%95%A0-%EA%B2%BD%EC%9A%B0)
+ [PostContruct가 Transaction이 적용 안되는 이유](https://github.com/kamser0415/study-jpa/blob/main/active-2/sc2_init.md)
+ [쿼리 성능 최적화-1:1,N:1 관계](https://github.com/kamser0415/study-jpa/blob/main/active-2/sc3_optimizer.md)
+ [쿼리 성능 최적화-N:1 관계](https://github.com/kamser0415/study-jpa/blob/main/active-2/sc4_optimizer_collection.md)
+ [쿼리 성능 최적화-인스턴스(DTO)활용](https://github.com/kamser0415/study-jpa/blob/main/active-2/sc5_optimizer_collection_dto.md)
+ [OSIV - 엔티티매니저 인터셉터 활용](https://github.com/kamser0415/study-jpa/blob/main/active-2/sc6_osiv.md)

## JPA 기본편

### section 1~5
+ [JPA 시작하기](https://github.com/kamser0415/study-jpa/blob/main/base/jpa-base-ch2.md)
+ [영속성 관리-내부동작방식](https://github.com/kamser0415/study-jpa/blob/main/base/jpa-base-ch3.md)  
+ [엔티티 매핑](https://github.com/kamser0415/study-jpa/blob/main/base/jpa-base-ch4.md)
+ [기본키 매핑](https://github.com/kamser0415/study-jpa/blob/main/base/jpa-base-ch4-id-mapping.md)
+ [실전 예제-1](https://github.com/kamser0415/study-jpa/blob/main/base/jpa-base-ch-active.md)
+ [연관관계 기초](https://github.com/kamser0415/study-jpa/blob/main/base/jpa-base-ch5-multi.md)
+ [연관관계 기초-양방향](https://github.com/kamser0415/study-jpa/blob/main/base/jpa-base-ch5-bidirectional.md)

### section 6~8  
#### 다양한 연관관계
+ [일대일](https://github.com/kamser0415/study-jpa/blob/main/base/ch6/oneToOne.md)
+ [일대다](https://github.com/kamser0415/study-jpa/blob/main/base/ch6/oneToMany.md)
+ [다대다](https://github.com/kamser0415/study-jpa/blob/main/base/ch6/manyToMany.md)
  
#### 고급매핑
+ [상속 매핑](https://github.com/kamser0415/study-jpa/blob/main/base/ch7/inheritance.md)
+ [MappedSupperClass](https://github.com/kamser0415/study-jpa/blob/main/base/ch7/MappedSuperClass.md)
+ [복합키+식별/비식별 관계 매핑](https://github.com/kamser0415/study-jpa/blob/main/base/ch7/CompositeKey.md)
#### 프록시와 연관관계 정리
+ [프록시와 연관관계](https://github.com/kamser0415/study-jpa/blob/main/base/ch8/proxy.md)
+ [즉시로딩과 지연로딩](https://github.com/kamser0415/study-jpa/blob/main/base/ch8/lazy.md)
+ [영속성 전이와 고아객체](https://github.com/kamser0415/study-jpa/blob/main/base/ch8/cascade.md)
### section 9 ~ 11
#### 값 타입
+ [값 타입](https://github.com/kamser0415/study-jpa/blob/main/base/ch9/embedded.md)
#### 객체지향 쿼리 언어 - 기본 문법 
+ [JPA 기초-프로젝선,페이징](https://github.com/kamser0415/study-jpa/blob/main/base/ch10/Jpql.md)
+ [조인](https://github.com/kamser0415/study-jpa/blob/main/base/ch10/jpql-join.md)
+ [서브쿼리,타입표현,조건식,함수,경로 표현식 등](https://github.com/kamser0415/study-jpa/blob/main/base/ch10/jpql-sub.md)
#### 객체지향 쿼리 언어 - 중급 문법
+ [페치조인](https://github.com/kamser0415/study-jpa/blob/main/base/ch10/fetch.md)
+ [다형성 쿼리](https://github.com/kamser0415/study-jpa/blob/main/base/ch10/Polymorphism.md)
+ [엔티티 직접사용](https://github.com/kamser0415/study-jpa/blob/main/base/ch10/Self.md)
+ [Named 쿼리](https://github.com/kamser0415/study-jpa/blob/main/base/ch10/Named.md)
+ [벌크 연산](https://github.com/kamser0415/study-jpa/blob/main/base/ch10/bulk.md)
#### 외래키 주의사항
+ [외래키 MySQL](https://github.com/kamser0415/study-jpa/blob/main/base/referencekey.md)

## Spring Data JPA
#### 환경설정
+ [라이브러리 설정 및 의존관계](https://github.com/kamser0415/study-jpa/blob/main/data-jpa/section1_enviroment/dependencies.md)
+ [파라미터 로그 남기기](https://github.com/https://github.com/kamser0415/study-jpa/blob/main/data-jpa/section1_enviroment/query_parameter.md/study-jpa/blob/main/data-jpa/section1_enviroment/dependencies.md)  

#### 스프링 DataJPA 기본
+ [공용 인터페이스 설정](https://github.com/kamser0415/study-jpa/blob/main/data-jpa/section3_share_interface/ch1_share.md)
+ [공통 인터페이스 사용 방법](https://github.com/kamser0415/study-jpa/blob/main/data-jpa/section3_share_interface/ch2_use_data_jpa.md)
+ [페이징과 정렬](https://github.com/kamser0415/study-jpa/blob/main/data-jpa/section3_share_interface/ch3_paging.md)
+ [벌크 수정 및 삭제](https://github.com/kamser0415/study-jpa/blob/main/data-jpa/section3_share_interface/ch4_bulk.md)
+ [엔티티 그래프 활용](https://github.com/kamser0415/study-jpa/blob/main/data-jpa/section3_share_interface/ch5_entitygraph.md)
+ [JPA 힌트와 락](https://github.com/kamser0415/study-jpa/blob/main/data-jpa/section3_share_interface/ch6_hintAndLock.md)

#### 스프링 Data JPA 확장
+ [사용자 정의 인터페이스](https://github.com/kamser0415/study-jpa/blob/main/data-jpa/section4_extenstion/ch1_user_interface.md)
+ [Auditing](https://github.com/kamser0415/study-jpa/blob/main/data-jpa/section4_extenstion/ch2_auditing.md)
+ [web 확장](https://github.com/kamser0415/study-jpa/blob/main/data-jpa/section4_extenstion/ch3_web.md)
+ [Data JPA 구현체 분석](https://github.com/kamser0415/study-jpa/blob/main/data-jpa/section4_extenstion/ch4_entity.md)
+ [specifications](https://github.com/kamser0415/study-jpa/blob/main/data-jpa/section4_extenstion/ch5_specifications.md)
+ [QueryByExample](https://github.com/kamser0415/study-jpa/blob/main/data-jpa/section4_extenstion/ch6_query_by_example.md)
+ [Projection](https://github.com/kamser0415/study-jpa/blob/main/data-jpa/section4_extenstion/ch7_projections.md)
+ [Native](https://github.com/kamser0415/study-jpa/blob/main/data-jpa/section4_extenstion/ch8_native.md)
