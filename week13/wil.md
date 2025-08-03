# Chapter 7. Log-Structured Storage

Immutable storage structure는 수정을 허용하지 않음.<br>
테이블은 한 번 작성되면 절대 바뀌지 않고 대신 새로운 레코드가 파일의 끝에 추가된다.<br>
읽기 작업을 할 때는 여러 파일에서 레코드를 모아서 재구성한다.<br>

이와 달리 mutable storage structure는 수정을 허용한다.<br>

레코드의 수정을 허용하지 않기 때문에 동시성과 무결성을 보장함 -> 함수형 프로그래밍에서 Immutable data structure를 사용하는 이유<br>

지금까지 본 B-Tree가 mutable storage structure의 예시였다면, Log-Structured Merge Tree는 immutable storage structure의 예시.<br>

수정시 기존 위치를 찾아가서 수정하는 것(In-place update)은 읽기 성능에는 좋지만 쓰기 성능에는 좋지 않다.<br>
append-only로 구현하는 immutable storage structure는 쓰기 성능이 좋지만 읽기 성능이 좋지 않다.<br>

## LSM Trees
buffering과 append-only 방식으로 random이 아닌 sequential한 쓰기를 가능하게 한다.<br>
B-Tree와 유사하지만 파일을 가득 채우고, 순차적 디스크 접근에 최적화되어 있다.<br>

- 노드를 가득 채움 -> 내부 단편화가 줄고 쓰기 성능이 향상됨
- 순차적 디스크 접근에 최적화 -> 한 번에 많은 양의 데이터를 읽고 쓸 수 있음

LSM Tree에서의 merge는 merge sort와 유사한 방식으로 이루어진다.<br>
이 merge 작업은 불필요한 복제본들을 정리하거나 읽기 작업을 수행할 때 이루어진다.<br>

쓰기 작업과 버퍼의 변경을 메모리 위에 있는 테이블에 쌓아두고,<br>
immutable disk files에 옮긴다.<br>
파일이 완전히 저장될 때까지 모든 레코드들은 메모리에서 접근 가능한 상태로 존재한다.<br>

데이터들을 불변으로 유지하기 위해서는 sequential write가 유리하다.<br>
데이터들은 디스크에 한번에 기록되고 추가만 허용된다.<br>
변경 가능한 구조에서는 한번의 접근으로 미리 블록을 할당하지만,<br>
random read/write가 뒤따른다.<br>

불변 구조는 데이터 레코드를 순차적으로 배치할 수 있게 하여 단편화를 방지한다.<br>
따라서 불변 파일은 더 높은 밀도로 저장되고,<br>
이후에 기록될 데이터 레코드를 위해 여분의 공간을 예약하지 않아도 되며,<br>
업데이트된 레코드가 처음 기록된 것보다 더 많은 공간이 필요할 경우를 고려해 추가 공간을 남겨두지 않아도 된다.<br>

append-only이므로 삽입/수정/삭제 시 데이터 레코드를 찾을 필요가 없으므로 처리 속도가 빠르다.<br>
읽기 작업은 적고 쓰기 작업이 많은 경우에 LSM을 사용하는 것이 좋다.<br>

설계상 읽기와 쓰기가 겹치지 않기 때문에, 디스크의 데이터를 세그먼트 락 없이 읽고 쓸 수 있어 동시 접근이 단순하다.<br>  
반면, mutable 구조에서는 무결성을 보장하기 위해 hierarchical locks과 latches를 사용하며,<br>
쓰기 작업 시에는 해당 서브트리에 대한 배타적 소유권이 필요.<br>

LSM 기반 저장 엔진들은 linearizable in-memory views of data and index files를 사용하고,<br>
이런 구조들을 관리하는 부분만 동시 접근을 제어하면 된다.<br>

### LSM Tree Structure (ordered)
메모리에 상주하는 작은 컴포넌트와 디스크에 상주하는 큰 컴포넌트로 구성된다.<br>
불변 파일들을 쓰기 위해 메모리에서 버퍼링을 하고 이때 정렬이 이루어진다.<br>

메모리에 상주하는 그 작은 컴포넌트를 memtable이라고 부르는데 이건 변경이 가능하다.<br>
읽기나 쓰기 요청이 들어오면 가장 먼저 이 memtable을 참조하거나 수정한다.<br>
이 공간에 쌓인 data의 양이 임계치를 넘으면 디스크에 저장되고 memtable의 수정은 디스크 접근이 필요없다.<br>
durability를 위해 WAL를 사용한다.<br>

memtable은 정렬된 상태를 유지하여 동시 접근을 허용하고 내부 구조는 주로 정렬된 트리로 되어있다.<br>

디스크 상주 컴포넌트는 읽기만을 위해서 존재한다.<br>

### Two-component LSM Tree
Two-component LSM Tree는 디스크에 단 하나의 컴포넌트만 가지고 이는 불변 세그먼트로 구성된다.<br>
이 컴포넌트는 가득 찬 노드들로만 구성된 B-Tree이고 읽기 전용이다.<br>

메모리에 있는 데이터를 flush 할 때는 disk의 B-Tree에서 해당하는 서브트리를 찾고 적절한 위치에서 기존 노드와 병합하 새로운 segment를 작성한다.<br>
병합 후에는 병합에 사용한 메모리 출신 데이터와 디스크 출신 데이터를 discard하고 새로운 세그먼트로 대체한다.<br>
병합 과정에서 사용하는 두 source 모두 이미 정렬된 상태이므로 iterator로 훑기만 하면 금방 끝난다.<br>

#### subtree merge와 flush 주의사항
1. flush가 시작되면 모든 새로운 write 사항은 새로운 memtable에 저장된다.
2. flush중에도 두 테이블은 읽기를 위해 접근할 수 있어야 한다.
3. 병합된 내용을 저장하고 병합 대상들을 discard하는 작업들은 atomic하게 이뤄져야 한다.

### Multicomponent LSM Trees
여러 개의 디스크 테이블을 가지는 구조를 말한다.<br>
그렇기 때문에 데이터를 찾을 때는 여러 테이블을 훑어야 하는 문제가 생기는데,<br>
이를 개선하기 위해 compaction을 정기적으로 실행한다.<br>

### In-memory tables
memtable을 flush 하기 전에는 switch를 먼저 해야 한다.<br>
새로운 memtable을 먼저 할당하여 flush 하는 동안 들어오는 write 요청이 기록될 수 있도록 해야 한다.<br>
이 과정을 atomic하게 이뤄져야하고,<br>
flush 동안은 기존 memtable과 새로운 memtable이 모두 read 가능한 상태이다.<br>



