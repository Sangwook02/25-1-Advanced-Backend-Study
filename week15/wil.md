# Chapter 7. Log-Structured Storage

## Concurrency in LSM Trees
플러시가 일어날 때 지켜져야 하는 것들이 있다.<br>
- 새로운 memtable이 쓰기와 읽기 가능하도록 해야 한다.
- 기존 memtable은 읽기 가능해야 한다.
- 플러시되는 memtable은 디스크에 쓰기 되어야 한다.
- 플러시된 테이블들을 discard 하는 작업은 원자적이어야 한다.
- WAL도 제거되어야 한다.

일반적으로는 Memtable switch, flush finalization, WAL truncation의 세가지 포인트가 있다.<br>
- Memtable switch: 새로운 memtable에 쓰기가 반영되도록 한다.
- Flush finalization: 테이블뷰에서 플러시된 memtable을 제거한다.
- WAL truncation: 플러시된 memtable에 대한 WAL을 제거한다.

## Log Stacking
LSM Tree와 SSD의 조합은 쓰기 증폭 감소, 성능 개선, 수명 연장의 효과가 있다.<br>
하지만, 여러 로그 구조 시스템을 조합하면 오히려 쓰기 증폭이나 단편화의 문제를 유발할 수도 있다.<br>

### Flash Translation Layer

SSD에서 로그 구조 매핑 계층을 쓰는 이유
- 작은 랜덤 쓰기를 모아서 한번에 쓸 수 있고
- 비어있는 곳에만 쓰기를 할 수 있음

하나의 페이지만 지울 수 있는 것이 아니고 블록 단위로 여러 페이지를 지워야 함.<br>
FTL은 논리적 페이지 주소를 물리적 페이지 주소로 매핑하는 계층이다.<br>
free 상태인 페이지가 부족해지면 가비지 컬렉션을 통해 비어있는 페이지를 확보한다.<br>

블록 내의 모든 페이지가 지워도 된다는 보장이 없으므로 FTL은 live인 페이지를 찾아서 다른 곳으로 재배치 해야한다.<br>
모든 페이지가 옮겨지면 블록이 안전하게 지워질 수 있고, 쓰기 가능한 상태가 된다.<br>

### Filesystem Logging
각 계층에서 book keeping을 해야 하는데 중복 로그가 발생하고 서로 다른 가비지 컬렉션이 발생한다.<br>
게다가 misaligned segment writes가 발생할 수 있는데 상위 계층의 로그를 버릴 때 단편화나 relocation을 유발할 수 있다.<br>


계층간에 소통이 없으니 하위 계층에서는 중복된 작업을 수행하게 된다.<br>
그리고 정해진 크기가 없으니 상위 레벨의 한 세그먼트가 하위 레벨의 여러 세그먼트로 나뉘어 저장될 수 있다.<br>

로그를 별도의 디바이스에 둬서 다른 작업과 섞이지 않게 하거나 쓰기 단위를 하드웨어 페이지 크기와 정렬하여 해결할 수 있다.<br>

## LLAMA and Mindful Stacking
Bw-Tree는 LLAMA 위에 구축된다.<br>
LLAMA는 latch-free, log-structured, access-method aware의 약자로,<br>
동적 확장이나 축소를 가능하게 하며, 가비지 컬렉션과 페이지 관리가 효율적이다.<br>

논리적 노드는 물리적 델타 노드의 연결 리스트로 구성되며,
update chain은 base node로 끝난다.<br>

저장은 delta node에 4MB씩 모아서 disk에 저장한다.
하지만, 서로 다른 노드의 델타가 삽입 순서대로 섞여서 기록되어 단편화가 발생할 수 있는데 이런 문제를 주기적으로 가비지 컬렉션으로 불필요한 node를 삭제하여 해소한다.<br>

### Open-Channel SSDs
여러 계층으로 쌓는 방식이 아니라 하드웨어를 직접 사용하는 방법이다.<br>
파일 시스템이나 FTL을 모두 사용하지 않고 직접 접근하는 것이다.<br>

내부 구조, 드라이브 관리, I/O 스케줄링 정보를 드러내어, FTL을 거치지 않으므로,<br>
개발자 입장에서 관리에 더 많은 노력이 필요하지만 성능 개선에서는 큰 효과를 볼 수 있다.<br>
간단한 API를 하나 제공하고 내부 복잡성을 숨기는 것이다.<br>
각 계층에 특별한 요구사항이 있으면 문제가 될 수도 있지만 내부 병렬 처리 성능으 높일 수도 있다.<br>