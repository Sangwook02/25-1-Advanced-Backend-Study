# Chapter 2. B-Tree Basics

## Binary Search Trees
BST는 정렬된 in-memory data structure이다.<br>
node를 대표하는 key가 있고 각 key는 value를 갖는다.<br>

하나의 root node로 부터 시작되고,
node의 key는 자신의 왼쪽에 있는 어떤 key보다 크다.<br>

### Tree Balancing
insertion을 하다보면 tree가 skewed해질 수 있다.<br>
이렇게 skewed해지면 tree의 height가 커지고, 탐색 속도가 느려진다.<br>

이를 방지하기 위해 balancing이 필요하다.<br>
balancing을 해주면 tree의 height는 log2 N으로 줄어든다.<br>

Balancing은 tree height를 줄이고 node들이 한 쪽으로 너무 치우치지 않도록 하는 방향으로 진행된다.<br>

### Trees for Disk-Based Storages
근데 BST는 fanout이 2로 고정이다보니 balancing이 자주 필요하다.<br>
만약, BST로 DB를 구현했다면 locality와 tree height의 측면에서 문제가 발생했을 것이다.<br>

=> 이런 문제를 해결하기 위해 B-Tree를 사용하는 것이다.

## Disk-Based Structures

### On-Disk Structures
디스크에 접근하는 시간 외에도 디스크에 한계가 있다면, 그것은 디스크 작동의 가장 작은 단위가 block이라는 점이다.<br>
특정 지점의 data를 읽기 위해서는 data가 있는 block을 통째로 읽어야한다.<br>

이 block을 가져올 때 pointer라는 것을 이용하는데, on-disk에서 pointer는 offset등을 계산하여 활용한다.<br>
찾고자 하는 block까지의 pointer chain이 길어질 수 있기 때문에 대체로 offset은 미리 계산해두거나 메모리에 캐싱하두는데,
이런 점을 고려했을 때 chain이 길어지지 않게 하기 위해서 fanout을 키우고 height를 줄이는 것이 좋다.<br>

#### 그렇다면 한 페이지에 여러 node를 저장한다면?
locality 측면에서 이점이 있다.<br>
다음 노드를 찾기 위해서는 이미 읽어온 page 내에서 pointer를 따라가면 되므로 disk I/O를 줄일 수 있다.<br>

하지만 tree balancing 과정에서 pointer 변경을 유발하는 page reorganization이 필요할 수 있다.<br>


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
B-Tree는 각 노드에 저장되는 key의 수를 의미하는 Fanout이라는 특징이 있다.<br>
높을수록 분할과 병합의 횟수를 줄일 수 있다. -> 구조 변경의 비용을 줄일 수 있다.<br>
분할과 병합같은 구조 변경을 Balancing Operation이라고 하는데 이런 작업들은 주로 node가 
가득 차거나 비어있을 때 발생한다.<br>

#### B+ Tree
지금까지 설명한 내용들은 B-Tree 계열의 모든 트리들이 가지는 공통적인 특징이다.<br>
B-Tree와 B+ Tree의 차이는 leaf node에서 발생한다.<br>
B+ Tree는 leaf node만이 value를 가진다.<br>
Internal node들은 탐색을 위한 separator key만을 가진다.<br>

value를 leaf node만 가지다보니 삽입, 수정, 삭제, 조회와 같은 모든 operation은 leaf node에서만 발생하고,<br>
병합이나 분할이 발생할 때만 상위 node에 영향을 미친다.<br>

### Separator Keys
B-Tree의 node에 정렬되어 있는 key들은 index entries, separator keys, divider cells라고 불린다.<br>
이 key들은 lower level의 subtree들을 분할하고 범위를 정의한다.<br>
이분 탐색을 하기 위해 항상 Key들은 정렬되어 있어야 한다.<br>

K1, K2, K3가 있다고 가정했을 때, 범위는 4개로 나뉘고 아래와 같다.<br>
1. x < K1
2. K1 <= x < K2
3. K2 <= x < K3
4. K3 <= x

원칙은 key의 왼쪽에는 key보다 작은 값이 있어야 한다는 것이다.<br>

#### Sibling Node Pointer
일부 B-Tree 변형에서는 sibling node pointer를 사용한다.<br>
주로 leaf node에서 범위 query를 효율적으로 수행하기 위해서 사용한다.<br>

예를 들어, K2-1 ~ K2+1을 찾는 범위 query를 해야하고 sibling node pointer가 없다면, 
k2-1을 찾은 후에 parent node를 통해 형제 노드로 가서 K2+1을 찾아야 한다.<br>
이를 leaf node간의 pointer를 통해 연결해주면,
K2-1을 찾은 후에 sibling node pointer를 통해 K2+1을 바로 찾을 수 있다.<br>

#### Bottome to Top
B-Tree는 Bottom to Top으로 확장된다.<br>
우선 leaf node에 데이터를 삽입하고,
leaf node가 가득 차 있을 때 분할 과정에서 separator key가 추가되면서 parent node에 영향을 미친다.<br>
이런 방식으로 B-Tree의 구조적 변화는 Bottom to Top으로 전파된다.<br>

### B-Tree Lookup Complexity
B-Tree에서 탐색의 복잡도는 아래 두 가지 관점에서 바라볼 수 있다.
1. block transfer의 수 (disk I/O 관점)
  - logK M개의 page만 읽으면 key를 찾을 수 있다.
  - 트리의 height만큼의 child pointer를 따라가면 검색을 할 수 있다.
2. comparison의 수 (비교 연산 관점)
  - 각 노드 내에서 key를 찾는 것은 binary search를 이용하므로 log2 M개의 비교 연산이 필요하다.

logK M * log2 M개의 비교 연산이 필요하고,
이는 log M으로 단순화할 수 있다.<br>

### B-Tree Lookup Algorithm
B-Tree에서 탐색을 하려면 root에서 시작해서 leaf까지 한 번의 traversal을 해야한다.<br>
탐색의 목적은 두 가지로 나뉘는데,
1. key를 찾는 것 
  - key를 찾는 경우는 point query나 update, deletion 같은 연산을 할 때
2. key의 predecessor를 찾는 것
  - key의 predecessor를 찾는 경우는 range query를 하거나 insertion을 할 때

root에서 시작해서 찾으려는 값보다 큰 첫번째 key를 찾고 그 subtree로 내려간다.<br>
이런 과정을 반복하다보면 leaf node에 도달하게 된다.<br>