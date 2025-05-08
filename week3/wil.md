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
B-Tree는 계층적 구조를 가지고 있다.<br>
구성하는 노드들을 3가지로 구분 가능한데
1. Root Node
  - 부모 노드가 없는 노드
  - 트리의 최상단에 위치
2. Leaf Nodes
    - 자식 노드가 없는 노드
    - 트리의 가장 하단에 위치
3. Internal Nodes
  - root node와 leaf node를 연결하는 노드

