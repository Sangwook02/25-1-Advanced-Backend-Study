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
### ARIES

## Concurrency Control
### Serializability
### Transaction Isolation
### Read and Write Anomalies
### Isolation Levels