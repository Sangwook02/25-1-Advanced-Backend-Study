# Part I. Storage Engines

DBMS(Database Management System)의 근본적인 목적은 data를 reliable하게 저장하고,<br>사용자가 그 data를 이용할 수 있게 하는 것이다.<br>
DBMS가 있기 때문에 개발자는 data를 어떻게 저장할지에 대한 고민을 덜어내고 비즈니스 로직에 집중할 수 있다.

DB는 transport layer, query processor, storage engine 등으로 구성된 모듈형 시스템이다.
이 중 storage engine은 저장, 조회, 관리의 책임이 있다.
DB는 복잡한 쿼리들을 다루지만 Storage Engine은 간단한 조작 API를 제공하여 사용자가 생성, 수정, 삭제, 조회의 기능을 간단히 사용할 수 있도록 한다.
이런 유연성을 위해 Storage Engine은 모두 직렬화하여 'sequences of bytes'의 형태로 저장한다.

# Chapter 1. Introduction and Overview

## Introduction and Overview

DBMS는 여러 목적으로 data를 제공할 수 있다.

- 일시적 hot data
- 장기 보관할 cold storage
- 복잡한 분석 쿼리

이외에도 다양한 목적으로 활용 가능하다.<br>
이번 챕터는 이 책을 읽으면서 마주할 여러 단어들의 의미를 정리하는 것이 목적이다.

우선, DBMS의 아키텍쳐를 살펴보고 시스템 컴포넌트들과 각각이 어떤 책임을 가지는지 알아본다.
이후, 저장 매체의 측면에서 Memory-based와 Disk-based DBMS의 차이를 살펴본다.
레이아웃 측면에서 Column-oriented와 Row-oriented DBMS를 비교하여 공부한다.
물론, 저장 매체와 레이아웃이 모든 DBMS에 대한 분류는 아니다. 이외에도 OLTP, OLAP, HTAP와 같은 분류도 있다.
high-level data organization 접근을 위해 data와 index file에 대해 공부한다.
buffering, immutability, ordering에서는 효율적인 storage structure를 구성하고 이 세 가지 개념이 어떻게 설계와 구현에 영향을 주는지를 살펴본다.

## DBMS Architecture

모든 DBMS는 조금씩 다른 구조를 가지고 있고 각 컴포넌트들의 경계는 명확하지 않다.<br>
docs만 봤을 때는 각각의 독립적으로 존재하는 것처럼 보일 수 있지만, 실제로는 꽤나 coupling(결합)되어 동작한다.

DBMS는 client/server 모델로 작동한다.<br>
아마 컴퓨터 네트워크 수업에서 다루었던 것 같은데, client/server 모델이란 client가 server에 요청을 보내고 server는 이 요청을 처리하여 응답을 주는 모델이다.<br>
DBMS의 활용에서는 DBMS가 주로 server가 되고 DBMS에 쿼리를 보내는 등의 요청을 하는 application이 client가 된다.<br>

application이 보낸 request는 query의 형태로로 DBMS 아키텍쳐의 최상단에 있는 Transport Subsystem을 통해 도착한다.<br>
이뿐만 아니라, Transport Subsystem은 node간의 통신도 담당한다.<br>
여기에서 node란, 분산된 환경에서 DB를 사용하는 경우 여러 db 인스턴스가 있을텐데 각각의 인스턴스를 node라고 한다.

Transport Subsystem을 통해 들어온 query는 Query Processor로 넘어간다.<br>
Query Processor는 query를 파싱, 해석, 검증한다.<br>
query의 해석이 끝나면 access control check를 한다.<br>
access control check란 사용자가 해당 데이터에 접근할 권한이 있는지 확인하는 작업을 말한다.<br>

이렇게 파싱된 query를 Query Optimizer가 받아서 불가능하거나 중복되는 부분을 제거하여 optimizing 작업을 준비한다.<br>
이후 가장 효율적으로 실행할 수 있는 방식을 internal statistics와 data placement(data가 어떤 node에 있고, 옮기는 비용 등)를 활용해 찾는다.
여기에서 internal statistics란 index cardinality나 approximate intersection size를 말한다.
index cardinality라고만 하면 감이 잘 안 올 수 있는데 인덱스 컬럼에 존재하는 값들의 다양한 정도라고 생각하면 좋다.
이게 높을수록 인덱스로 걸러낼 수 있는 것들이 많기 때문에 호율이 좋아진다.
approximate intersection size도 간단히 설명하자면 where에 담긴 조건들을 동시에 만족시키는 결과의 크기를 말한다.

Query Optimizer는 쿼리를 실행하기 위해 다음과 같은 작업들을 실행한다.
쿼리에 있는 조인, 필터링, 그룹화 연산의 처리 순서를 결정하고 쿼리를 해결하기 위해 dependency tree를 그린다.
dependency tree는 쿼리에서 각 연산들의 순서를 결정하는데 사용된다.

Query는 가장 효율적인 실행 순서로 결정되어 Execution plan(혹은 Query plan)의 형태로 Execution Engine에 전달된다.
Execution Engine은 remote와 local execution으로 나뉜다.
Remote Execution은 분산 환경에서 클러스터의 다른 노드에 data를 read/write 하는 것이다.
쉽게 말해 작업을 여러 노드에 분산하여 read/write 하는 것이다.

Local query들은 Storage Engine에 의해 실행된다.
Local query란, client로부터 직접 받거나 다른 노드들로부터 받은 쿼리를 모두 포함한다.

이제 마지막으로 Storage Engine이 일할 차례다.
Storage Engine은 아래의 요소들로 구성되어 있다.

- Transaction Manager<br>
  트랜잭션을 스케줄링하고 일관성 유지에 기여한다.
- Lock Manager<br>
  물리적 data integrity를 위반하지 않도록 하고 트랜잭션의 고립성을 지켜준다.
- Access Methods<br>
  B-tree나 Log structured merge tree와 같이 db에서 data를 찾는 방법을 말한다.
- Buffer Manager<br>
  disk에서 읽어온 data page들을 메모리에 캐싱하는 역할을 한다.
- Recovery Manager<br>
  로그를 관리하고 장애가 발생한다면 이 로그를 이용해서 복구하는 역할을 한다.

## Memory- Versus Disk-Based DBMS

Memory 기반과 Disk 기반 DBMS 모두 memory와 disk를 둘 다 사용한다.<br>
당연히 data를 주로 저장하는 곳은 각각 memory와 disk로 다르지만,<br>
Memory 기반은 disk를 복구와 로깅 목적으로 사용하고 Disk 기반은 memory를 disk에서 읽어온 data page들을 캐싱하거나 정렬이나 조인 시에 활용한다.

### Memory-based가 가지는 이점

우선, memory 접근 시간은 disk에 접근하는 시간보다 훨씬 빠르다.<br>
그리고 data를 워드 단위로 읽기 때문에 page 단위로 읽는 disk 기반에 비해 access granularity가 크다.<br>
access granularity라고 하는 것은 더 섬세하게 필요한 data만 읽어올 수 있다 정도로 이해하면 될 것 같다.<br>

또한, OS 덕에 메모리 관리는 굉장히 간단해진다.<br>
주소로 접근하여 필요한 크기만큼만 할당하고 free할 수 있는 반면, disk는 접근 시에 disk의 어디에 있는지 찾아가는 과정이 메모리에 비해 훨씬 복잡하고,
disk의 fragmentation 등 더 고려해야 할 부분이 많다.

### Memory가 이렇게 좋은데 Disk-based도 쓰는 이유?

Memory는 비싸다.
저장 공간의 크기에 비해 disk보다 훨씬 비싸다.

그리고 memory는 휘발성이 있다.
전원 공급이 안 되면 data가 휘발된다.

이런 점을 보면disk가 훨씬 관리하기 쉽고 비용 측면에서도 유리하다.

### Memory 기반의 Durability를 위한 것들

Disk 백업을 지원하여 data loss를 방지한다.

이를 위해 WAL과 Backup Copy를 활용한다.
WAL, 풀어서 Write Ahead Logging이라는 것이 있는데 Operation이 완료되려면 memory에 기록하고 로그 파일에도 기록되어야 한다는 것이다.
Backup Copy는 충돌 시 이미 완료된 operation을 반복하지 않기 위해 스냅샷을 sorted disk-based structure로 disk에 저장해둔 것이다.

이제 이 두가지를 이용해 checkpointing을 한다.
disk에서 backup copy를 가져와서 앞서 log file에 기록해둔 것들 중에서 아직 완료되지 않은 것들을 반영한다.
이렇게 새로운 backup copy를 만들어서 복구한다.
