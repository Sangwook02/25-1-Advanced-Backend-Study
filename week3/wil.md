# Chapter 2. B-Tree Basics
## B-Tree Basics
## Binary Search Trees
## Disk-Based Structures

### On-Disk Structures
디스크에 접근하는 시간 외에도 디스크에 한계가 있다면, 그것은 디스크 작동의 가장 작은 단위가 block이라는 점이다.<br>
특정 지점의 data를 읽기 위해서는 data가 있는 block을 통째로 읽어야한다.<br>
..
## Ubiquitous B-Trees
도서관에서 책을 찾아가는 과정처럼 B-Tree도 계층적이다.<br>
B-Tree는 balanced search tree와 유사하지만,
더 많은 자식 노드를 가질 수 있다는 점(higher fanout)과 높이가 더 작다는 점(smaller height)에서 다르다.<br>

일반적으로 binary tree의 노드는 원으로 그린다.<br>
각 노드가 하나의 key만 가지고, child가 두 개의 범위로만 쪼개지기 때문이다.<br>
이와 달리 B-Tree는 노드가 여러 개의 key를 가지고, 여러 개의 범위로 나눠진다.<br>
이런 차이를 반영하기 위해 B-Tree의 노드는 사각형으로 그린다.<br>

다음은 B-Tree 노드의 내부에서 일어나는 일을 살펴보자.<br>
B-Tree의 노드는 여러 개의 key를 가진다고 했는데, 이 key들은 정렬되어 있다.<br>
덕분에 bianry search 같은 알고리즘을 B-Tree에서도 활용할 수 있다.<br>
B-Tree에서는 point query와 range query 모두 효율적으로 실행할 수 있다.<br>

### B-Tree Hierarchy
B-Tree는 여러 노드들로 구성된다.<br>
각 노드는 N개의 key를 가지고, N+1개의 pointer를 가진다.<br>
이 pointer들은 child node를 가리킨다.<br>

Node들은 3가지로 나뉘는데,
1. Root Node
    - parent가 없는 노드
    - 가장 위에 있는 노드
2. Leaf Node
    - 가장 아래에 있는 노드
    - child가 없는 노드
3. Internal Node
   - 이외 모든 노드
   - root와 leaf를 연결

Occupancy란 node의 capacity와 실제로 들고 있는 key의 수 간의 관계를 말한다.<br>

### Separator Keys
### B-Tree Lookup Complexity
### B-Tree Lookup Algorithm