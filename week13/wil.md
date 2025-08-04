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


### Updates and Deletes
같은 key를 가지는 data record가 여러 개 존재할 수 있다.<br>
그렇기 때문에 삭제를 할 때는 memtable에서 지우는 것만으로는 부족하다.<br>

예를 들어, key가 k1인 레코드가 disk table에 있고 값으로는 v1을 가진다고 하자.<br>
그리고 memtable에 k1의 레코드가 v2로 업데이트되어 있다고 하자.<br>
memtable에서 k1의 레코드를 발견했다고 이것만 지우면 disk table에 있는 v1이 남아있게 된다.<br>

이런 문제를 해결하기 위해서 명시적으로 삭제되었음을 표시할 수 있는 방법이 필요하고,<br>
LSM Tree는 tombstone이라는 개념을 사용한다.<br>
앞서 FD-Tree에서 삭제를 구현할 때 언급했던 내용이다.<br>

    11주차에 안 오셨던 분들을 위해 tombstone에 대해 다시 설명하자면,
    삭제된 데이터임을 표시하는 데이터의 묘비인 셈이다.
    이 tombstone은 실제로 데이터를 삭제하는 것이 아니라,
    다른 data들처럼 insert되어 읽기 작업 중에 발견되면 해당 key는 삭제된 것으로 간주한다.


#### 연속된 키 삭제하기
연속된 키 n개를 모두 삭제한다고 하자.<br>
위에서 소개한 방법대로라면 n개의 tombstone을 추가해야 한다.<br>
이렇게 했을 때 발생하는 문제점들이 여럿 있다.<br>

- 삽입해야 하는 tombstone이 많아져서 쓰기 성능이 떨어진다.
- tombstone이 차지하는 공간이 많아져서 디스크 공간을 낭비한다.
- 읽기 작업 시 tombstone을 모두 훑어야 하므로 읽기 성능이 떨어진다.
- compaction 시 tombstone을 모두 훑어야 하므로 compaction 성능도 떨어진다.

##### 대책: Predicate Deletes
연속된 키 여러 개를 두 개의 tombstone으로 처리하는 방법이다.<br>
예를 들어, Disk Table 1에 key가 1, 3, 5, 7, 9, 11, 13인 레코드가 있다고 하자.<br>

이때 4보다 크거나 같고, 9보다 작은 레코드를 삭제한다고 하자.<br>
DELETE FROM table WHERE key ≥ 3 AND key < 9

그렇다면 아래와 같이 추가된다.<br>
```
[Disk Table 1]
1, 3, 5, 7, 9, 11, 13

[Disk Table 2]
3(tombstone_start), 9(tombstome_end)
```

이렇게 하면 tombstone이 차지하는 공간이 줄어들고,<br>
읽기 작업을 할 때 3, 5, 7은 조회되지 않는다.<br>

