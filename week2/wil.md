# Chapter 1. Introduction and Overview

## Column-Versus Row-Oriented DBMS

대부분의 DBMS는 record의 형태로 값을 저장한다.<br>
테이블에는 컬럼과 로우가 있고, 이 둘이 교차하는 어느 한 지점을 필드라고 한다.<br>
같은 컬럼의 필드들은 모두 같은 data type을 가진다.

데이터베이스를 layout으로 분류하면 컬럼 지향과 로우 지향이 있다.<br>
MySQL이나 PostgreSQL 같은 것들이 대표적인 로우 지향 DBMS이다.<br>
당연히 로우 지향 데이터베이스는 로우 단위로 접근하는 일이 많거나 모든 값을 비슷한 위치에 저장하는 것(spatial locality)이 유리할 때 사용한다.<br>
특히 하나의 블록에 같이 저장된다는 점이 조회나 저장 등의 작업 시 유리하다.

그렇다면 컬럼 지향이라는 것은 뭘까?<br>
컬럼 지향은 같은 컬럼에 있는 값들을 모아서 비슷한 위치에 저장한다는 것이다.<br>
이는 평균 내기와 같이 aggregation 작업을 할 때 유리하다.<br>

다만, 로우 단위로 재조합할 때 생기는 문제가 있다.<br>
컬럼의 어떤 값이 어떤 pk와 연결되는지를 파악하기 위해 각 필드에 explicit하게 혹은 implicit 하게 식별자를 기록해야 한다.<br>
explicit 하게 기록한다는 것은 말그래도 매번 id를 같이 기록하는 것이고,<br>
implicit 하게 기록한다는 것은 offset, 위치에 근거하여 id와 연결한다는 뜻이다.

### Distinctions and Optimizations

layout에 따른 DBMS의 분류는 단순히 data의 저장방식에 의한 차이가 아니다.<br>
이는 최적화에 대한 이야기이기도 하다.

컬럼 지향은 캐시 활용도와 계산 효율성을 향상시키고,<br>
컬럼 지향의 경우 같은 데이터 타입을 모아서 저장하므로 압축의 측면에서도 이점이 있다.

이 둘 중 어떤 것을 쓸지를 고민할 때는 access pattern을 이해해야 한다.<br>
레코드 단위로 접근하느냐, 특정 열의 부분 집합에 대한 집계를 하느냐에 따라 선택하는 것이 좋다.

## Data Files and Index Files

DBMS의 최우선 목표는 저장과 접근을 빠르게 할 수 있도록 제공하는 것이다.<br>
그렇다면 data는 어떻게 조직되어야 DBMS가 효율성을 높일 수 있을까?<br>

DBMS는 데이터를 저장하기 위해 file을 사용하지만,<br>
file system의 구조를 사용하지 않고 specific formats를 사용한다.<br>
그 이유는 아래와 같다.

- Storage Efficiency  
  storage overhead가 최소화되도록
- Access efficiency  
  최소한의 단계로 접근 가능하도록
- Update efficiency  
  disk의 변화를 최소화하면서 값을 수정할 수 있도록

각 테이블은 보통 하나의 file로 관리되고 그 파일 내에서 각 레코드는 search key를 이용해 찾을 수 있다.<br>
이때 index라는 것을 이용하는데, index는 필드의 부분 집합이고 레코드를 식별할 수 있다.

DBMS는 보통 data files와 index files를 분리하여 저장한다.<br>
data file에는 data 레코드가 저장되고,<br>
index file에는 레코드의 메타데이터가 저장되어 있다.<br>
그렇기 때문에 대체로 index file이 data file에 비해 작은 편이다.<br>

각 file은 page들로 구성되는데, page란 하나 혹은 여럿의 disk block의 사이즈다.<br>
각 페이지는 레코드의 순차적인 배열로 구성될 수도 있고 slotted page로 구성될 수도 있다.<br>
Slotted Page란 offset과 length를 이용해서 기록하는 것을 의미한다고 한다.

다음은 삭제에 대한 이야기인데, delete를 한다고 해서 진짜 지워지는게 아니라고 한다.<br>
최근에 다른 아티클에서도 봤는데 deletion marker라는 것을 이용하여, 지워진 데이터라고 마킹을 해둔다고 한다.

### Data Files

Date file은 index-organized, heap-organized, hash-organized table로 구현될 수 있다.<br>

힙 파일에서는 특정 순서를 따르지 않고, write 되는 순서대로 저장된다.<br>
새로운 page 추가 시 추가 작업 없이 그냥 레코드만 추가하면 된다는 장점이 있다.<br>
힙 파일은 index structure가 있어야 레코드를 빠르게 찾을 수 있다.<br>

해시 파일에서 레코드는 버켓에 저장된다.<br>
키의 해시 값이 어느 버켓에 저장할지를 결정한다.<br>
키로 정렬하여 조회 속도를 개선할 수 있다.<br>

index-organized table에서는 key로 정렬되어 저장되므로, 범위 스캔시 연속된 블록들을 순차적으로 읽기만 하면 된다.<br>
인덱스를 찾는 것이 곧 레코드를 찾는 것이므로 한번의 I/O로 레코드를 찾을 수 있다.<br>

### Index Files
인덱스란 레코드를 효율적으로 찾기 위해서 사용하는 구조이다.<br>
인덱스 파일은 인덱스나 키로 식별되는 레코드가 매핑된 구조이다.<br>



## Buffering, Immutability, and Ordering

## Summary
