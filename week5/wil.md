# Chapter 3. File Formats
## General Principles
file format을 결정할 때는 addressing을 어떻게 할 것인지에 대한 고려로 시작된다.<br>

대체로 storage structure들은 같은 크기의 페이지를 사용하므로 읽기와 쓰기 접근이 간단하다.<br>
페이지가 메모리에서 가득 차면, 디스크에 쓰기 위해 페이지를 flush한다.<br>

파일은 주로 header와 page, 그리고 trailer로 구성된다.<br>
이런 구조를 가지기 때문에 disk에 저장할 data의 양을 줄일 수 있다.<br>
고정된 크기의 값들은 header에 저장하고, 가변 크기를 갖는 값의 offset이나 length도 header에 저장한다.<br>
이런 구조를 취하기 때문에 가변 크기의 값들을 분리해서 저장할 수 있다.<br>

## Page Structure
데이터베이스 시스템은 데이터 레코드를 데이터 파일과 인덱스 파일에 저장하는데 이 파일들을 페이지라고 한다.<br>

키와 데이터 레코드 쌍을 저장하는 리프 노드와 다른 노드를 가리키는 키와 포인터를 저장하는 비리프 노드를 구분한다.<br>
페이지는 고정 크기의 데이터 레코드를 단순히 연결한 구조로 구성된다.<br>

## Slotted Pages
가변 크기의 레코드를 저장할 때 가장 큰 문제는 free space를 회수하는 것이다.<br>
레코드가 차지하고 있던 공간을 회수하는 것이다.<br>
만약, 크기가 n인 레코드를 크기가 m인 레코드가 차지했던 공간에 넣으려고 한다면,<br>
m == n이거나 (m - n) 크기의 레코드를 찾을 수 없는 한, 이 공간은 낭비된다.<br>
마찬가지로, 크기가 m인 segment에 크기가 k(k > m)인 레코드를 넣으려고 하면, 이 공간은 사용되지 않은 채로 남게 된다.<br>

가변 크기의 레코드를 저장하기 위해 페이지를 고정 크기의 segment로 나누는 방법이 있다.<br>
하지만, 이 경우에도 공간 낭비가 발생할 수 있다.<br>
각 segment의 안에서 단편화가 발생할 수 있기 때문이다.<br>

다시 쓰거나 record를 이동하는 방식으로 공간 회수를 할 수 있지만 레코드 오프셋을 보존해야 한다.<br>
해당 페이지 밖의 포인터가 이 오프셋을 사용할 수 있기 때문이다.<br>

정리하자면,
- 최소한의 낭비로 가변 크기의 레코드를 저장할 수 있어야
- 삭제 레코드의 공간을 회수할 수 있어야
- 정확한 위치를 없이 페이지 내에서 레코드를 참조할 수 있어야 한다.

PostgreSQL에서도 사용하는 slotted page 구조를 사용하면 이러한 문제를 해결할 수 있다.<br>

슬롯이나 셀의 모음으로 페이지를 구성하고, 포인터와 셀을 페이지의 서로 다른 쪽에 있는 두 개의 독립적인 메모리 영역으로 분리한다.<br>
셀을 참조하는 포인터만 재구성하면 순서를 유지할 수 있고, 레코드를 삭제하는 것은 포인터를 null로 설정하거나 제거하는 것으로 할 수 있다.<br>

slotted page는 페이지와 셀에 대한 중요한 정보를 담은 고정된 크기의 header를 가진다.<br>
셀은 크기가 다를 수 있고, 키, 포인터, 데이터 레코드 등 임의의 데이터를 담을 수 있다.<br>
유지와 관리를 위한 영역인 header, cell, 그리고 각 셀을 가리키는 포인터로 구성된다.<br>

## Cell Layout
cell은 key와 key-value cell로 나뉜다.<br>
key cell은 separator key와 하위 페이지 포인터로 구성되고, key-value cell은 key와 data record로 구성된다.<br>

같은 페이지에 속한 cell들은 통일성이 있어야 한다.<br>
key cell만 있거나 key-value cell만 있어야 하고,<br>
고정된 크기만 있거나 가변 크기만 있어야 한다.<br>
이렇게 함으로써, 페이지 안에 있는 cell들이 각각 metadata를 가지지 않아도 되며,<br>
page 내에서 metadata가 중복되어 존재하지 않는다.<br>

### key cell 구성
아래의 것들을 알 수 있어야 한다.<br>
cell type은 metadata에 있으므로 cell에서는 관리할 필요가 없다.

- cell type
- key size
- 가리키는 child page의 id
- key bytes

cell에서는 key bytes가 가변 크기이므로 cell의 뒷부분에 둔다.<br>
이렇게 함으로써
- 고정 크기 필드는 미리 정해진 위치에서 바로 읽을 수 있고
- 가변 크기 필드는 offset 계산을 간단히 할 수 있다는 
장점이 있다.<br>

### key-value cell 구성
key-value cell은 child page의 id 대신 data record를 가진다.

- cell type
- key size
- value size
- key bytes
- data record bytes

### Page Id, Offset
page의 크기는 고정되어 있으므로, page id만 알면된다.<br>
만약 page의 크기가 4KB이고 page id가 3이라면, page의 시작 위치는 3 * 4KB = 12KB가 된다.<br>
이렇게 file에서 page의 위치를 찾아가고 그 안에서 cell offset을 이용하여 cell의 시작 위치를 찾을 수 있다.<br>
cell offset이 page의 안에서만 사용되기 때문에 page-local이라고 부르기도 한다.<br>

## Combining Cells into Slotted Pages
cell들을 page에 넣기 위해서 cell들은 page의 오른쪽에서부터 왼쪽으로 채워지고,<br>
cell의 offset과 pointer들은 page의 왼쪽에서부터 오른쪽으로 채워진다.<br>

따라서 cell들은 순서대로 배치되지 않고 논리적 정렬 순서는 cell offset 포인터를 정렬하여 유지한다.<br>
이렇게 설계했기 때문에 cell들은 삽입, 수정, 삭제 작업 중에 재배치할 필요 없이 페이지에 추가될 수 있다.<br>

cell들이 정렬되어야 할 때는 cell 자체가 정렬된 순서로 저장되는 것이 아니라,<br>
cell offset 포인터가 정렬된 순서로 저장된다.<br>

## Managing Variable-Size Data
Removing an item from the page does not have to remove the actual cell and shift other cells to reoccupy the freed space.

페이지에서 삭제하는 것은 셀을 실제로 제거하고 다른 셀들을 이동시켜서 비워진 공간을 다시 채우는 것이 아니라,<br>
'삭제됨'으로 표시하고 **availability list**에 해제된 공간의 크기와 pointer를 저장하는 것이다.

**availability list**에는 해제된 segment의 offset과 크기가 저장된다.<br>

새로운 cell을 저장할 때는 availability list를 확인하여 쓸 수 있는 공간을 찾는데 이때 사용하는 전략으로 First Fit과 Best Fit이 있다.<br>
- First Fit
  - availability list를 처음부터 끝까지 순회하면서 첫 번째로 적합한 공간을 찾는다.
  - overhead가 커질 수 있다.
- Best Fit
  - 필요한 공간의 크기보다 큰 공간 중에 가장 작은 공간을 찾는다.

연속된 공간 중에는 필요한 공간만한 크기가 없지만 전체 사용 가능한 공간은 충분하다면 live cell들을 옮겨서 공간을 만든다.<br>
단편화 제거 작업을 해도 공간이 부족하다면 overflow page를 만들어서 사용한다.

## Versioning
데이터베이스 시스템은 기능 추가, 버그 수정, 성능 개선 등으로 계속 발전합니다.
이 과정에서 바이너리 파일 포맷(저장 방식)이 바뀔 수 있습니다.
새로운 버전의 소프트웨어는 예전 버전의 파일도 읽을 수 있어야 하므로, 여러 버전의 포맷을 동시에 지원해야 합니다.

- 파일명에 버전 정보 포함
  - 파일 이름에 버전 접두어(prefix) 붙이기 -> 파일을 열지 않고도 버전을 알 수 있음
- 별도의 파일에 저장
- 인덱스 파일 헤더에 직접 저장
  - 헤더는 버전에 상관없이 같은 형식으로 인코딩 되어야 함
  - 파일의 헤더에 버전 정보를 기록 -> 버전별로 파일을 해석하는 방법을 다르게 적용할 수 있게 해줌

## Checksumming
파일은 sw나 hw의 오류로 인해 손상될 수 있다.<br>
이런 손상을 파악하기 위해서 checksums나 CRCs를 사용할 수 있다.<br>

이들은 목적이나 보장 수준에서 차이가 있지만, 공통점은 큰 data의 덩어리를 작은 숫자로 변환한다는 것이다.<br>

checksums는 가장 약한 수준의 보장을 제공하고 여러 bit의 오류는 감지할 수 없다.<br>
XOR 연산을 parity 검사나 덧셈과 함께 사용한다.<br>

CRCs는 연속적인 bit가 손상된 경우를 감지하는 데 유용하며,<br>
그 구현은 일반적으로 lookup table과 polynomial division을 사용한다.<br>
상당 수의 에러는 multibit 에러로 나타나기 때문에 multibit 에러를 감지하는 것이 중요하다.<br>

data를 disk에 작성하기 전에 checksum을 계산하고 data와 함께 저장한다.<br>
읽어올 때는 checksum을 다시 계산하여 저장된 checksum과 비교한다.<br>
불일치 시, corruption 발생한 것을 파악할 수 있고 그 읽어온 data를 사용하지 않아야 한다.<br>

전체 파일에 대해서 checksum을 계산하는 것은 실용적이지 않기 때문에 주로 page checksum을 사용한다.<br>
이렇게 하면 파일에서 corruption이 발생한 부분만 discard 하면 되기 때문에 효율적이다.<br>


## Summary
바이너리 데이터 조직
- 원시 데이터 유형을 직렬화하는 방법
- 셀로 결합하는 방법
- 셀로부터 슬롯 페이지를 구축하는 방법
- 이러한 구조를 탐색하는 방법

가변 크기 데이터 유형을 처리하는 방법, 가지고 있는 값의 크기를 보유하는 특수 셀을 구성하는 방법

페이지 밖에서 셀 ID로 개별 셀을 참조하고, 
레코드를 삽입 순서대로 저장하며,
key 순서를 cell offset으로 정렬하여 유지하는 slotted page 형식

이러한 원칙은 온디스크 구조와 네트워크 프로토콜을 위한 바이너리 형식을 구성하는 데 사용할 수 있다.