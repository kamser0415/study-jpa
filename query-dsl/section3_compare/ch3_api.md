# 조회 API 컨트롤러 개발
API 테스트를 위해 샘플 데이터를 추가하되, 테스트 코드에 영향이 없도록 
프로파일 설정을 분리해서 초기화합니다.  
  
#### main 프로파일 설정
```xml
src/main/resource/application.yml
spring:
  profiles:
    activive: local
```

#### test 프로파일 설정
```xml
src/test/resource/application.yml
spring:
  profiles:
    activive: test
```  
  
main 소스 코드와 테스트 소스 코드 실행시 사용할 프로파일을 분리할 수 있습니다.  
  
### 샘플 데이터 추가
```Java
@Profile("local")
@Component
@RequiredArgsConstructor
public class InitMember {
    private final InitMemberService service;

    @PostConstruct
    public void init() {
        service.init();
    }

    @Component
    static class InitMemberService {

        @PersistenceContext
        EntityManager em;

        @Transactional
        public void init() {
            Team teamA = new Team("teamA");
            Team teamB = new Team("teamB");
            em.persist(teamA);
            em.persist(teamB);
            for (int i = 0; i < 100; i++) {
                Team selectedTeam = i % 2 == 0 ? teamA : teamB;
                em.persist(new Member("member" + i, i, selectedTeam));
            }
        }
    }
}
```  
`@Profile("xxxx")`를 설정하면 동작하는 프로파일 명과 일치 할 경우 실행됩니다.  
  
### API 생성
```Java
@RestController
@RequiredArgsConstructor
public class MemberConstructor {
    private final MemberJpaRepository memberJpaRepository;
    @GetMapping("/v1/members")
    public List<MemberTeamDto> searchMemberV1(MemberSearchCondition condition) {
        return memberJpaRepository.search(condition);
    }
}
```  
`@ModelAttribue`가 생략된 객체로 조건 검색에 필요한 엔티티가 담아져서
DTO로 반환이 된다.  
  
