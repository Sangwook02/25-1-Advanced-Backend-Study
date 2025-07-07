# Chapter 5. Transaction Processing and Recovery
## Recovery
데이터베이스는 여러 hw와 sw 위에 구성되는데, 이들은 모두 안정성과 신뢰성을 보장하지 않는다.<br>
따라서 데이터베이스는 fail에 대비해 recovery를 지원해야 한다.<br>
예를 들어 write 되기로 한 데이터는 꼭 디스크에 기록되어야 한다.<br>

그래서 WAL (Write-Ahead Logging)이라는 기술이 사용된다.<br>
WAL은 append-only auxiliary disk-resident structure인데,<br>
- 변경없이 항상 끝에만 추가되고(append-only),
- 실제 데이터와 따로 존재하여 복구를 지원하는 보조의 역할을 하며(auxiliary),
- 디스크에 저장되고(disk-resident),
- 특정한 형식과 규칙을 가지고 순서를 가지는 로그 레코드(structure)

지금까지 data flush전에는 이 내용들이 disk에 아예 존재하지 않는다고 알고 있었는데 사실은 WAL 형태로 존재한다는 것이다.<br>
WAL의 기능은 다음과 같다.
1. dirty page를 디스크에 flush하기 전에 메모리에서 기다리는 동안에 발생한 crash에 대한 durability를 보장
2. 캐싱된 내용이 disk에 flush되기 전에 WAL을 기록
3. 메모리에 있다가 유실된 데이터를 복구하기 위해 WAL을 사용

여기서 잠깐 WAL과 트랜잭션의 커밋 순서를 살펴보자.<br>
만약 update를 하려고 한다면 우선 대상 데이터를 메모리에 올릴 것이다.<br>

1. 메모리로 disk로부터 페이지를 로드
2. 메모리에 있는 페이지에 값을 수정
3. WAL를 작성해서 메모리에 두기
4. COMMIT 요청이 오면 WAL을 디스크에 저장 시도
5. WAL을 디스크에 flush
6. 5번이 성공한다면 commit 마무리
7. 나중에 백그라운드에서 메모리에 있는 dirty 페이지를 Disk로 flush

### POSTGRESQL VERSUS FSYNC()

우선, PostgreSQL의 메모리 계층을 좀 알아야 다음 내용을 이해할 수 있다.<br>
PostgreSQL은 2가지 메모리 캐시를 사용한다.<br>
1. **Shared Buffers**: PostgreSQL의 메모리 캐시로, 디스크에서 읽은 페이지를 저장.
2. **OS Cache**: 운영 체제의 파일 시스템 캐시로, 디스크에서 읽은 데이터를 저장. 우리 책에서는 kernel pages라고 부른다.<br>

update가 발생하면 Shared Buffers에 있는 페이지만 수정되고 dirty page로 표시된다.<br>
PostgreSQL이 write를 수행할 때, 먼저 Shared Buffers에 있는 dirty page를 OS Cache에 기록한다.<br>
이제 이 kernel page를 disk에 flush하기 위해 fsync()를 호출한다.<br>
fsync()는 kernel call로, kernel page를 disk에 flush 하는 역할을 한다.<br>

근데 이제 리눅스에서는 fsync가 disk에 쓰기를 성공하지 못해도 dirty flag를 꺼버린다.<br>
그래서 PostgreSQL은 fsync가 성공했다고 생각하고 넘어가지만 실제로는 disk에 쓰지 못한 상황이 발생할 수 있다.<br>

1. PostgreSQL가 write를 요청해서 OS가 fsync를 호출한다.
2. Linux에서는 fsync가 사실 쓰기를 실패한 상황에도 dirty flag는 꺼버리고 에러를 반환한다.<br>
   열려있지 않은 파일에 대해서는 에러를 반환하지 않는다.
3. PostgreSQL은 에러가 반환되지 않으면 fsync가 성공했다고 생각하고 dirty flag를 꺼버린다.

이렇게 되면 PostgreSQL은 데이터가 disk에 쓰이지 않았는데도 dirty flag를 꺼버리게 되므로 데이터 유실이 발생할 수 있다.<br>
checkpointer가 모든 파일을 열어두는게 아니기 때문에 발생 가능한 문제이다.<br>

### Log Semantics
앞서 봤듯이 WAL은 append-only다.<br>
이는 WAL가 불변하다는 의미다.<br>
가장 최근에 작성된 로그 뒤에 새로운 로그가 추가되므로 쓰기와 무관하게 읽기도 가능하다.<br>

WAL은 log record로 구성된다.<br>
모든 log record는 LSN(Log Sequence Number)을 가지고 있다.<br>
주로 LSN은 한 블록(4, 8KB)을 가득 채우지 못하기 때문에 log buffer에 쌓이다가,
가득 차면 force operation으로 디스크에 기록된다.<br>
이때 LSN에 의해 log record의 순서가 보장된다.<br>

#### Commit Record
Commit record는 트랜잭션이 commit되었음을 나타내는 log record이다.<br>
force operation으로 log record가 디스크에 기록될 때 commit record까지 기록되어야 커밋이 완료된 트랜잭션으로 간주한다.<br>

예를 들어 트랜잭션 T1에서 다음과 같은 작업을 한다고 하자.<br>
`INSERT INTO users VALUES ('Alice')`<br>

그럼 T1의 실행에 따라 다음과 같은 log record가 생성된다.<br>
1. [LSN=100] Alice를 users 테이블에 삽입하는 log record
2. [LSN=101] T1의 commit record<br>
이렇게 commit record인 [LSN=101]이 디스크에 기록되면 T1은 commit된 것으로 간주된다.<br>

만약 commit record가 디스크에 기록된 상태에서 crash가 발생한다면,<br>
WAL을 사용해서 T1의 작업을 Redo하여 복구할 수 있다.<br>

#### Compensation Log Record
충돌이 발생해도 시스템이 올바르게 작동할 수 있도록 하기 위해,<br>
롤백이나 복구 중에 CLR(Compensation Log Record)를 사용한다.<br>
정상적인 트랜잭션 처리중에는 생성되지 않고 롤백이나 복구 과정에서 생성된다.<br>

예를 들어 Undo 작업을 이미 실행했는데 이를 다시 Undo 하지 않도록 하기 위해,<br>
Undo를 실행했다는 기록을 CLR로 남긴다.<br>
한마디로 중복 Undo를 방지하기 위한 것이다.<br>

#### WAL Trimming
WAL은 checkpoint에 도달할 때마다 제거된다.<br>
데이터가 실제로 저장되는 것과 WAL이 제거되는 것 사이에 약간의 불일치만 생겨도 데이터 손실이 발생할 수 있으므로,<br>
most critical correctness aspects이다.<br>

Checkpoint는 로그가 특정 지점까지의 로그 레코드가 완전히 저장되었음을 아는 방법이다.<br>
데이터베이스가 시작할 때 어디서부터 복구를 시작해야 하는지 알 수 있으므로 시작 시의 작업량을 크게 줄여준다.<br>

모든 dirty page를 디스크에 강제로 플러시하는 과정을 sync checkpoint라고 하는데,<br>
전체 데이터를 한번에 flush하는 것은 성능에 좋지 않다.<br>
그래서 fuzzy checkpoint를 사용한다.<br>

#### Fuzzy Checkpoint
용어
- begin_checkpoint: 체크포인트 시작 지점을 기록하는 log record
- end_checkpoint: 체크포인트 종료 지점을 기록하는 log record. dirty page 등을 같이 기록함.
- last_checkpoint: 마지막 체크포인트의 begin_checkpoint를 기록하는 포인터. log header에 저장됨.

예시
1. [LSN=100] 트랜잭션 T1이 A 데이터 변경
2. [LSN=101] 트랜잭션 T1의 커밋 로그 / Commit Record
3. [LSN=102] 트랜잭션 T2의 B 데이터 변경
4. [LSN=103] 트랜잭션 T3의 C 데이터 변경
5. [LSN=110] 체크포인트 시작 로그 / begin_checkpoint
6. [LSN=111] 트랜잭션 T4의 D 데이터 변경
7. [LSN=115] 체크포인트 종료 로그 / end_checkpoint  << 여기에 더티 페이지 목록과 트랜잭션 테이블이 기록됨

7번에 기록된 더티 페이지 flush가 완료되면,<br>
이제 last_checkpoint는 5번의 begin_checkpoint를 가리키게 된다.<br>
7번이 아니라 5번을 가리키는 이유는 6번처럼 체크포인팅의 시작과 종료 사이에 발생한 트랜잭션은<br>
다음 체크포인팅을 할 때 체크포인팅을 해야 하기 때문이다.<br>

### Operation Log(논리적) Versus Data Log(물리적)
#### Shadow Paging
여러 데이터베이스에서 사용하는 기법으로, <br>
copy-on-write 방식으로 데이터의 durability를 보장하고 트랜잭션의 atomicity를 보장한다.<br>

구체적인 방법은 아래와 같다.<br>
1. 트랜잭션이 시작되면 해당 페이지의 복사본을 만든다.<br>
2. 트랜잭션이 페이지를 수정할 때, 원본 페이지가 아닌 복사본을 수정한다.<br>
3. 트랜잭션이 commit되면, 원본 페이지를 가리키던 포인터가 복사본을 가리키도록 변경한다.<br>

이렇게 하면 트랜잭션 실행 도중에 실패하면 복사본을 버리기만 하면 롤백이 끝나므로 durability와 atomicity 보장이 편해진다.<br>

#### Before-Image and After-Image
변경 전의 상태를 before-image, 변경 후의 상태를 after-image라고 한다.<br>
before-image에 redo 연산을 적용하면 after-image가 생성되고,<br>
after-image에 undo 연산을 적용하면 before-image가 생성된다.<br>

#### Physical Versus Logical Log
물리적 로그는 페이지 전체나 바이트 단위로 기록되는 반면,<br>
논리적 로그는 트랜잭션의 연산이나 명령어 단위로 기록된다.<br>

예를 들어 A의 잔고가 원래 1000원인데 600원으로 변경하는 경우,<br>
물리적 로그는 'A의 잔고가 1000에서 600으로 변경됨'을 통째로 기록한다.<br>
반면 논리적 로그는 'A의 잔고에서 400을 뻬기'와 같은 명령어 단위로 기록한다.<br>

즉, 물리적 로그는 before-image와 after-image를 모두 기록하고,
논리적 로그는 어떤 연산을 적용해야 하는지에 대한 정보만 기록한다.<br>
물리적 로그는 복구 시 단순하고 빠르지만 공간을 많이 차지하고,<br>
논리적 로그는 공간을 적게 차지하지만 복구 시 연산을 다시 실행해야 하므로 시간이 더 걸린다.<br>

둘다 장단점이 있기 때문에 많은 데이터베이스는 둘을 모두 사용한다.<br>
undo를 실행하기 위해서는 concurrency와 performance를 고려하여 논리적 로그를 쓰고,<br>
redo를 실행하기 위해서는 복구 시간을 위해 물리적 로그를 사용한다.<br>

### Steal and Force Policies
메모리에 변경된 내용을 디스크에 플러시하는 시점을 결정하기 위해,<br>
데이터베이스 관리 시스템은 steal/no-steal 및 force/no-force 정책을 정의한다.<br>
이 정책들은 페이지 캐시에 적용되지만 어떤 정책을 쓰느냐에 따라 복구 알고리즘의 설계가 달라지므로<br>
복구와 관련한 맥락에서도 중요하다.<br>

Steal policy: 아직 커밋되지 않은 트랜잭션이 수정한 dirty page를 disk에 flush 하는 것을 허용하는 것<br>
No-steal policy: 커밋되지 않은 트랜잭션이 수정한 dirty page를 disk에 flush 하지 않는 것<br>

Steal 정책을 사용하면 트랜잭션이 커밋되기 전에 dirty page를 disk에 flush할 수 있으므로,<br>
장애 발생 시 undo 해야 할 필요가 있지만 메모리에 있는 dirty page를 줄일 수 있으므로 장점이 있다.

No-steal 정책을 사용하면 트랜잭션이 커밋되기 전에는 dirty page를 disk에 flush 하지 않으므로,<br>
장애 발생 시 undo가 필요하지 않지만 메모리에 있는 dirty page가 많아질 수 있다.<br>
그리고 장애 발생 시 디스크에는 커밋 전 상태의 old copy가 남아있고 커밋되지 않은 내용은 그냥 날라간다.<br>
커밋되어 log로 남아있던 것들만 redo를 통해 복구할 수 있다.<br>

---
Force policy: 트랜잭션이 커밋되기 전에 dirty page를 disk에 flush 해야만 커밋이 완료되는 것<br>
No-force policy: 트랜잭션이 커밋되기 전에 dirty page를 disk에 flush 하지 않아도 커밋이 완료되는 것<br>

Force 정책을 사용하면 커밋된 트랜잭션의 변경 내용은 이미 disk에 기록되어 있으므로 장애가 발생해도 redo 작업이 필요하지 않다는 장점이 있지만,<br>
변경된 모든 페이지를 disk에 flush해야 하므로 불필요한 I/O가 발생해서 커밋 속도가 느려질 수 있다.<br>

No-force 정책을 사용하면 커밋된 트랜잭션의 변경 내용이 disk에 기록되지 않았을 수 있으므로 redo 로그를 사용해서 다시 적용해야 한다.<br>
또한, 메모리에 캐싱되어 있는 페이지의 수가 많아질 수 있으므로 더 큰 페이지 캐시가 필요하다.<br>

---
Undo: flush된 forced 페이지에 대해 커밋된 트랜잭션의 변경을 롤백하는 것.<br>
트랜잭션이 수정한 페이지가 커밋 전에 disk에 기록된다면 이를 롤백할 수 있도록 undo 로그를 기록해야 한다.<br> 

Redo: 커밋된 트랜잭션의 변경 내용이 disk에 반영되지 않았을 수 있으므로 로그를 보고 다시 적용하는 것.<br>
커밋 되었지만 disk에 기록되지 않은 상태에서 장애가 발생할 수 있으므로 redo 로그를 기록해야 한다.<br>

일반적으로는 Steal + No-force 정책을 사용하므로 undo와 redo 로그를 모두 기록해야 한다.<br>
커밋 전에 디스크에 기록된게 있을 수 있으므로 undo 로그를 남겨야 atomicity를,<br>
커밋되어도 디스크에 기록되지 않았을 수 있으므로 redo 로그를 남겨야 durability를 보장할 수 있다.<br>

### ARIES
Steal + No-force 복구 정책을 ARIES 라고 한다.<br>
Algorithms for Recovery and Isolation Exploiting Semantics의 약자이다.<br>

복구 performance(성능)을 높이기 위해 physical log를 사용하고,<br>
concurrency(동시성)를 높이기 위해 logical log를 사용한다.<br>

WAL를 사용하여 redo를 수행하고, <br>
아직 커밋되지 않은 트랜잭션만 골라서 undo 하여 장애 전의 상태로 복구한다.<br>

#### crash 발생 시 복구를 위한 3단계
1. 분석(Analysis) 단계<br>
분석 단계에서는 페이지 캐시에 있는 **dirty page**와 장애가 발생했을 당시 **진행 중이던 트랜잭션**을 식별한다.<br>
dirty page는 redo(재적용) 단계의 시작점을 결정하는 데 사용되고,<br>
진행 중이던 트랜잭션 목록은 undo(롤백) 단계에서 미완료 트랜잭션을 롤백하는데 사용된다.<br>
정리하자면 무엇을 redo하고 무엇을 undo할지 분석하는 단계이다.<br>

2. Redo(재적용) 단계<br>
redo 단계에서는 장애가 발생하기 전까지의 WAL 내역을 순서대로 다시 실행하여 적용한다.<br>
이 과정은 미완료 트랜잭션뿐만 아니라, 커밋되었지만 아직 디스크에 기록되지 않은 트랜잭션에도 적용된다.<br>

3. Undo(롤백) 단계<br>
undo 단계에서는 장애 시점에 완료되지 않은 모든 트랜잭션의 작업을 역순으로 되돌려서,<br>
데이터베이스가 가장 마지막으로 consistent 했던 상태로 복원한다.<br>
만약 복구 도중에 또 장애가 발생할 경우를 대비해, undo 작업 역시 로그에 남겨 동일한 작업이 반복되지 않도록 한다.<br>

---
ARIES는 로그 레코드를 식별하기 위해 LSN을 사용하고,<br>
실행 중인 트랜잭션이 수정한 페이지를 dirty page table로 추적하며,<br>
physical redo, logical undo, 그리고 fuzzy checkpointing을 활용한다.<br>

## Concurrency Control
### Serializability
### Transaction Isolation
### Read and Write Anomalies
### Isolation Levels