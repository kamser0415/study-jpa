# MySQL 외래 키 사용시  
외래 키란 ?(`MySQL8.0 기준`)  
외래키 제약이 설저오디면 연관되는 테이블의 칼럼에 인덱스가 생성됩니다.  
외래키가 제거되지 않은 상태에서는 자동으로 생성된 인덱스는 삭제할 수 없습니다.  
InnoDB의 외래키 관리에는 중요한 두 가지 특징이 있습니다.
1. 테이블 변경(쓰기 잠금)이 발생하는 경우에만 잠금 경합(잠금 대기)이 발생한다.
2. 외래키와 연관되지 않은 칼럼의 변경은 최대한 창금 경함(참금 대기)을 발생시키지 않는다.  
  
```sql
CREATE TABLE tb_team (
    id varchar(10) primary key,
    director varchar(255) not null
) ENGINE = InnoDB;
CREATE TABLE tb_member (
    id bigint primary key,
    username varchar(255) not null,
    name varchar(10),
    key ix_teamid(team_id),
    CONSTRAINT member_ibfk_1 
        FOREIGN KEY (team_id) 
            REFERENCES tb_team (id) 
            ON DELETE CASCADE
) ENGINE = InnoDB;
INSERT INTO tb_team VALUE ('T1','김정수'),('DRX','씨맥');
INSERT INTO tb_member value (1,'케리아','DRX')
```  
## 자식 테이블의 변경이 대기하는 경우  

| 작업번호 | 커넥션-1                                            | 커넥션-2                                         |
|------|--------------------------------------------------|-----------------------------------------------|
| 1    | BEGIN;                                           |                                               |
| 2    | UPDATE tb_team set director='양대인' WHERE id='T1'; |                                               |
| 3    |                                                  | BEGIN;                                        |
| 4    |                                                  | UPDATE tb_member SET team_id='T1' WHERE id=1; |
| 5    | ROLLBACK;                                        |                                               |
| 6    |                                                  | Query OK, 1 row affercted                     |

현재 상황은 팀 테이블에는 T1과 DRX가 있고, 팀 DRX 소속으로 케리아 선수가 뛰고 있습니다.
선수 테이블은 팀 테이블을 부모 테이블로 참조하고 있습니다.  
커넥션-1 : T1 팀은 감독을 김정수 -> 양대인으로 변경하려고 합니다.  
그 사이 커넥션-2 : 케리아 선수는 DRX -> T1으로 팀을 옴기려고 합니다.   
이때 테이블은 수정시 쓰기잠금이 발생합니다. id='T1'인 레코드가 쓰기 잠금 상태가 되었습니다.  
케리아 선수가 팀을 `T1`으로 옴기려고할 때 `잠금 대기`가 발생합니다.  
커넥션-1에서 id=`T1`인 레코드의 수정을 마무리하고 트랜잭션을 종료해야 커넥션-2가 동작합니다.  
**자식 테이블의 외래 키 칼럼의 변경(INSERT,UPDATE)은 부모테이블의 확인이 필요합니다.**  
이 상태에서 부모 테이블의 해당 레코드가 쓰기 잠금이 걸려 있으면 해당 쓰기 잠금이 해제 될 때까지 대기합니다.  
이것이 위에서 설명드린 InnoDB의 외래키 관리의 첫 번째 특징에 해당합니다.  
  
자식 테이블의 외래키(team_id)가 아닌 칼럼 (tb_member 테이블의 username)의 변경은  
외래키로 인한 잠금 확장(위 예제와 같은 상황)이 발생하지 않습니다.  
이게 외래키 관리의 두 번째 특징에 해당합니다.  
  
## 부모 테이블의 변경 작업이 대기하는 경우  

| 작업번호 | 커넥션-1                                           | 커넥션-2                              |
|------|-------------------------------------------------|------------------------------------|
| 1    | BEGIN;                                          |                                    |
| 2    | UPDATE tb_member SET username="류민석" WHERE id=1; |                                    |
| 3    |                                                 | BEGIN;                             |
| 4    |                                                 | DELETE FROM tb_team where id='T1'; |
| 5    | ROLLBACK;                                       |                                    |
| 6    |                                                 | Query OK, 1 row affercted;         |

 이제는 반대 상황입니다. 자식 테이블에서 쓰기 잠금이 걸려 있습니다.  
 ON DELETE CASCADE 옵션은    
 부모 테이블의 레코드가 삭제될 경우 레코드를 참조하는 자식 테이블의 레코드도 삭제하는 옵션입니다.  
 부모 테이블을 삭제하려고 할때, 추가 발생하는 자식 테이블의 레코드 수정을 하기위해 잠금 대기가 발생했습니다.  
   
데이터베이스에서 외래 키를 물리적으로 생성하려면 이러한 현상으로 인한 잠금 경합까지 고려해 모델링을 진행해야합니다.  
물리적으로 외래키를 생성하면 자식테이블에 레코드가 추가되는 경우 해당 참조키가 부모 테이블에 있는지 확인합니다.  
  
물리적인 외래키의 고려사항은 이러한 체크 작업이 아니라,  
**_체크를 위해 연관 테이블에 읽기 잠금을 걸어야 한다는 것입니다._**  
이렇게 잠금이 다른 테이블로 확장되면 그만큼 전체적으로 쿼리의 동시성 처리에 영향을 끼칩니다.  
  
[쓰기잠금-MySQL](https://dev.mysql.com/doc/refman/8.0/en/lock-tables.html)