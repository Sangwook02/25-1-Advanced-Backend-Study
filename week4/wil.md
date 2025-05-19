# Chapter 2. B-Tree Basics
## Ubiquitous B-Trees
### Counting Keys
key와 child offset의 개수를 설명하는 방법은 다양하다.<br>
우선, [BAYER72]에서는 페이지에 저장할 수 있는 키와 포인터의 최소/최대 개수를 k, l 등으로 설명하고,<br>
[GRAEFE11]에서는 N을 사용해 최대 N개의 키와 N+1개의 포인터로 설명합니다.

### B-Tree Node Splits
B-Tree에 값을 넣을 때는,
1. target leaf를 찾고
2. 노드에서 삽입 위치를 찾아야 한다.

만약 leaf가 가득 차 있다면, overflowed node이므로 split을 해야 한다.<br>
split은 새로운 노드를 할당하고 기존 노드의 key를 반으로 나누어 새로운 노드에 넣는 것이다.<br>
이때 옮기는 key중에 가장 작은 것을 부모 노드에 promote한다.<br>
promote 후에도 자식 노드에는 해당 key가 남아있다.<br>   

만약 이 과정에서 부모 노드도 가득 차 있다면, 부모 노드도 split을 해야 한다.<br>
이런 과정을 반복하다보면, root까지 올라가게 된다.<br>
leaf가 아닌 node의 split 상황에서 다른 점이 있다면, promote할 때 자식 노드에서는 사라진다는 것이다.<br>

분할의 과정을 정리하자면,
1. 새 노드를 할당한다.
2. 기존 노드의 절반을 새로운 노드로 옮긴다.
3. 새로운 노드를 부모 노드에 연결한다.
4. 부모 노드의 separator key를 추가한다.

### B-Tree Node Merges
key를 삭제하다보면 node가 비어버릴 수 있다.<br>
이런 경우에는 underflow이므로 merge를 해야 한다.<br>

만약 인접한 두 노드가 같은 부모를 가지고, 둘의 합이 하나의 node에 들어갈 수 있다면 merge를 한다.<br>
이런 상황을 concatenated라고 한다.<br>
만약 하나의 node에 들어갈 수 없다면 rebalancing을 한다.

하나의 node로 합치는 경우, child pointer가 하나 줄어들기 때문에 부모 노드의 key도 하나 줄어든다.<br>
이때 부모 노드의 key가 비어버리면 부모 노드에도 merge가 전파된다.<br>

## Summary
디스크에 저장하기 위한 적합한 데이터구조를 만들기 위해 B-Tree를 사용한다.<br>
fanout이 높고 height가 낮은 구조로 설계하여 balancing을 줄이고,<br>
삽입과 삭제 시에는 split과 merge를 통해 노드의 개수를 조절한다.<br>

# Chapter 3. File Formats
## Motivation
## Binary Encoding