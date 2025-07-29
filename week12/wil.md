# Chapter 6. B-Tree Variants

## Bw-Trees
문제점
1. 쓰기 증폭
2. 공간 증폭
3. 동시성의 복잡성

해결책 
1. Append-only storage
2. Link nodes into chains
3. Lock-free in-memory data structure
4. Single compare-and-swap operation

=> Bw-Tree

### Update Chain의 원리
- Base node: 최초의 데이터 상태를 저장하는 노드.
- Delta node: 각 삽입, 업데이트, 삭제 등 모든 수정 연산은 별도의 델타 노드로 생성되어, 최신에서 과거 순으로 연결 리스트(체인)를 이룸.
- 체인 구조: newest delta → … → old delta → base node
- 수정 방식: 기존 노드 재쓰기나 공간 예약 없이, 새 델타 노드를 기존 체인 앞에 붙임(Prepend). 디스크 I/O와 공간 증폭 해소.
- 저장 방식: base/delta 노드 크기가 다르므로 연속(Contiguous)하게 저장. 노드를 수정하지 않고 새로 붙이는 구조라서 별도 공간을 미리 마련할 필요 없음.
- 논리적 노드(Lazy update): 노드는 물리적인 고정 공간이 아닌 논리적 엔티티로 취급. 고정 크기, 연속 메모리 배치 등을 신경 쓰지 않아도 됨.
- 읽기 부하: 읽기 시 체인 전체의 델타를 모두 따라가 base node에 적용해 최신 상태를 복원해야 함(LA-Tree의 Lazy-Update와 유사).
- 장점
  - 공간 예약/재쓰기 없이 동적 구조, 모든 업데이트는 기존 노드 변경 없이 체인 선두에 새 노드 추가
  - 락프리(lock-free) 동시성 제어 가능.
- 단점
  - 체인이 길어질수록 읽기 성능 저하.

### Taming Concurrency with Compare-and-Swap in Bw-Tree
- 논리적 노드와 매핑 테이블
  - Bw-Tree의 노드는 실제 메모리/디스크 상의 위치가 아니라, 논리적 식별자(logical ID)로 관리됩니다.
  - 각 논리 노드는 in-memory 매핑 테이블을 통해 실제 위치(주소)를 찾게 됨. 이 테이블의 각 엔트리는 특정 논리 노드의 최신 상태(예: 최신 델타 체인 헤드)를 가리킵니다.
- CAS를 사용한 업데이트
  - 타겟 노드 탐색: 트리를 루트에서 리프 노드까지 탐색하며, 매핑 테이블을 통해 타겟 베이스 노드 또는 가장 최신 델타 노드에 접근.
  - 새로운 델타 노드 생성: 변경 연산(삽입, 수정, 삭제 등)을 나타내는 델타 노드를 생성하고, 기존 최신 노드를 가리키도록 포인터 설정.
  - 매핑 테이블 원자적 갱신: 이 델타 노드를 가리키도록 매핑 테이블 엔트리를 CAS로 원자적으로 교체.
     - 이 과정에서 여러 쓰레드가 동시에 엔트리를 갱신하려 할 수 있는데, CAS는 단 하나의 쓰레드만 성공하게 하며, 나머지는 재시도(retry)하게 됨.
     - 갱신 직전에 읽은 쓰레드(readers)는 이전 값을 보고, 그 이후의 쓰레드는 새로운 값을 참조함. 
     - 즉, 동시성 중에도 일관성(consistency)을 확보.
- Lock-free/Non-blocking
  - 어떠한 쓰레드도 락 대기 없이 항상 작업이 가능함.
  - CAS 연산 덕분에 다른 트랜잭션의 진행을 방해하지 않고도 데이터 상태를 안전하게 바꿀 수 있음.

### Structural Modification Operations

- **분할(Split)**
  - 분할할 노드의 논리적 내용을 정합(consolidation)하여 델타와 베이스 노드를 하나로 합침.
  - 분할 지점 오른쪽 데이터를 새 페이지에 저장.
  - **Split Delta Node** 추가: 분할 진행 중임을 알림. 분리 키(midpoint separator), 새 형제 노드의 링크 포함.
  - **Parent Update**: 부모 노드에 새 자식 노드 포인터 추가. 최적화 위헤서, 실제로는 부모가 갱신되지 않아도 모든 노드는 접근 가능.
  - Bw-Tree는 latch-free 구조로 어떤 스레드든 미완료된 SMO를 볼 수 있고, 협조적으로 SMO 처리를 완료 후 진행.

- **병합(Merge)**
  - **Remove Sibling**: 우측 형제 노드에 remove delta 추가, 해당 노드 삭제 대상임을 표시
  - **Merge**: 좌측 형제 노드에 merge delta를 추가하여 우측 노드의 내용을 연결
  - **Parent Update**: 부모 노드에서 우측 형제 링크 제거로 병합 완료
  - SMO 중복 방지: 동시 SMO에는 abort delta 설치로 병렬 시도 차단(사실상 write lock 역할)

- **루트 노드 분할**: 루트가 너무 커지면, 둘로 나누고 새로운 루트를 생성하여 두 노드를 자식으로 둠

### Consolidation & Garbage Collection

- **Delta Chain Consolidation**
  - 델타 체인이 길어지면, 베이스 노드와 델타 노드들을 합쳐 새 노드를 작성
  - 매핑 테이블의 포인터를 새로운 노드로 갱신

- **Garbage Collection**
  - Consolidation된 이전 노드는 바로 해제 불가(진행 중 연산이 접근할 수 있음)
  - Bw-Tree는 *epoch-based reclamation* 사용:
    - 각 epoch에서 해당 노드를 본 스레드가 전부 종료된 후에만 삭제 가능

## Cache-Oblivious B-Tree
- **동기화/튜닝 불필요**: 블록, 노드, 캐시라인 등 플랫폼에 따른 파라미터와 무관하게 최적 성능 보장.
- **설계 목적**: 어떤 두 계층(캐시/디스크 등) 사이에서도 최적화된 I/O 성능 제공.

### van Emde Boas 레이아웃
  - 트리를 절반으로 분할하면서 재귀적으로 서브트리를 메모리에 연속적으로 저장.
  - 트리의 논리적 구조와 메모리/디스크 상 실제 레이아웃이 다름.
- **동적 구조 지원**: Packed array 사용
  - 삽입/삭제 용으로 빈 공간(gap) 미리 생성, 필요시 재구성(rebuild).
  - Packed array 밀집도(density)가 임계치 초과/미만이면 전체 재구성.
  - 정적 트리는 packed array의 인덱스로 동작, 데이터 위치 이동시 포인터 갱신 필요.

- **실무적 한계**
  - 캐시-옵블리비어스 B-Tree의 실제 구현은 드묾. 페이지 교체, block transfer 등 실무상 제약.
  - 이론적 I/O 성능은 기존 B-Tree와 동일, 미래의 비휘발성 byte addressable storage에서 변화 가능성 있음.