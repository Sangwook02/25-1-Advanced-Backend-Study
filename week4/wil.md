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
메모리 접근 시에는 가상 메모리를 사용하므로 offset을 계산할 필요가 없지만,<br>
디스크에 접근할 때는 system call을 사용하므로 target file의 어디에 있는지 offset을 계산해야 한다.<br>

따라서 구축하고 수정하고 해석하기 쉬운 디스크 파일 포맷을 설계해야 한다.<br>
이 챕터에서는 B-Tree 뿐만 아니라 온디스크 데이터 구조를 설계하는 방법을 본다.

## Motivation
file format을 만드는 것은 C와 같은 프로그래밍 언어에서 malloc과 free를 사용하는 것과 비슷하다.<br>
이때 코드를 짜는 사람은 이 공간이 연속적인지 단편화 되어 있는지를 몰라도 된다.<br>
하지만, disk를 관리할 때는 이를 직접 관리해야 한다.<br>

디스크에 비해 메모리에서 데이터 레이아웃은 덜 중요하다.<br>
디스크에 상주하는 데이터 구조가 효율적이려면,<br>
1. 디스크에 데이터를 배치하는 방법을 고려해야 하며
2. 지속적인 저장 매체의 세부 사항을 고려하고
3. binary 데이터 형식을 생각하고
4. 데이터를 효율적으로 직렬화하고 역직렬화하는 방법을 찾아야 한다

## Binary Encoding 
디스크에 data를 효율적으로 저장하기 위해서는 compact하고 직/역직렬화하기 쉬운 형식으로 인코딩해야 한다.<br>
binary format에 대해 이야기할 때 layout이라는 단어를 자주 듣는다.<br>
malloc과 free가 없고 read와 write만 있기 때문에 접근 방식을 다르게 생각하고 데이터를 준비해야 한다.<br>

효율적인 page layout을 만들기 위한 원칙은 모든 binary format에 적용된다.<br>
레코드들을 페이지에 정리하기 전에 아래의 내용을 고려해야 한다.<br>
- binary 형식으로 키와 데이터 레코드를 표현하는 방법
- 여러 값을 더 복잡한 구조로 결합하는 방법
- 가변 크기 유형과 배열을 구현하는 방법

### Primitive Types
B-Tree의 key와 value는 integer, date, string과 같은 type이고,<br>
binary 형태로 직/역직렬화 되어 활용된다.<br>

대부분의 숫자 type은 고정 길이로 저장되는데,<br>
직/역직렬화 할 때는 같은 byte-order를 사용하는 것이 중요하다.
byte-order는 big-endian과 little-endian이 있다.<br>
- big-endian: MSB부터 가장 낮은 주소에 저장된다.<br>
- little-endian: LSB가 가장 낮은 주소에 저장된다.

primitives로 구성된 레코드를 저장할 때는 byte sequence를 사용한다.<br>
즉, write 할 때는 직렬화하고 read 할 때는 역직렬화 해야 한다는 것이다.

이진 데이터 형식에서는 더 복잡한 구조를 만들기 위해 원시 데이터 형식을 활용하는데,<br>
예를 들어 숫자들은 크기에 따라 1, 2, 4, 8 byte로 저장된다.<br>

소수는 float와 double로 저장되는데,<br>
이는 fraction을 이용하므로 정확한 표현이 불가능하다.<br>

### Strings and Variable-Size Data

All primitive numeric types have a fixed size. Composing more complex
values together is much like struct in C. You can combine primitive
values into structures and use fixed-size arrays or pointers to other
memory regions.
Strings and other variable-size data types (such as arrays of fixed-size
data) can be serialized as a number, representing the length of the array or
string, followed by size bytes: the actual data. For strings, this
representation is often called UCSD String or Pascal String, named after
the popular implementation of the Pascal programming language. 


모든 primitive numeric type은 고정된 크기를 가지는데,<br>
primitive value를 구조체로 조합하거나, 고정 크기의 배열이나 다른 메모리 영역에 대한 포인터를 사용할 수 있다.<br>

문자열과 다른 가변 크기 데이터 타입은 숫자로 직렬화할 수 있는데,<br>
수도 코드는 다음과 같다.

```c
String {
    size uint16
    data byte[size]
}
```

### Bit-Packed Data: Booleans, Enums, and Flags
boolean, enum, flag와 같은 데이터는 bit 단위로도 저장할 수 있다.<br>

boolean은 값이 true 아니면 false일테니 1bit로 저장할 수 있다.<br>

enum은 enum의 개수에 따라 bit를 사용하면 된다.<br>
예를 들어 4개의 enum이 있다면 2bit로 저장할 수 있다.<br>

