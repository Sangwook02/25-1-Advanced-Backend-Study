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

## LLAMA and Mindful Stacking

### Open-Channel SSDs
