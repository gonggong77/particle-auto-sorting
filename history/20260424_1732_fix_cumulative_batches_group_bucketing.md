# 누적 배치 수 전역 정렬 버그 수정 — GroupTag 버킷팅 복원

- 날짜: 2026-04-24 17:32
- Plan: `plans/20260424_csv_cumulative_batch_report.md`
- 선행 작업: `history/20260424_1720_csv_cumulative_batches.md` (초기 구현)

## 증상

샘플 프리팹으로 CSV export 후 ver2.csv 와 diff:

- `I8` (before 컬럼, below01 행) = 잘못된 값 (7 이어야 맞음)
- `I9` (before 컬럼, below02 행) = 잘못된 값 (8 이어야 맞음)
- BEFORE 컬럼 전체가 `1,2,3,4,5,6,7,8` 이 아니라 중간에 중복되며 8 에 못 도달.

AFTER 컬럼 (`1,1,2,3,4,4,5,6`) 은 정상.

## 원인

플랜의 "SortKey 전역 정렬 → 연속 동일 BatchKey 런마다 배치 인덱스 부여" 알고리즘이
잘못이었다. 플랜의 worked example 은 AFTER 케이스였고, AFTER 에선 Below 의
OiL 가 음수(-2, -1)라 Above 와 OiL 구간이 겹치지 않아 우연히 맞았다.

하지만 BEFORE 에선:

- `above01` : Layer=Default, OiL=0, Fudge=0, Material=FXM_Dust03
- `below01` : Layer=Default, OiL=0, Fudge=0, Material=FXM_Dust03

전역 정렬 시 두 렌더러가 인접(또는 동일 SortKey 로 tie)하고 BatchKey 가 완전
동일하므로 **같은 batch 로 merge** 되어 버린다. Above 그룹 6 배치 + Below 그룹 2
배치 = 8 이어야 할 BEFORE 총 배치 수가 전역 정렬로는 6 으로 줄어든다.

이는 도메인 규칙과 충돌한다. CLAUDE.md 도메인 섹션:

> Above 배치 수 + Below 배치 수 = 전체 배치 수 (Above와 Below 사이는 char가 끼므로
> 자동으로 끊김)

즉 Above/Below 는 개념적으로 **별도 렌더 패스** 이므로 BatchKey 가 같더라도
절대 같은 배치로 묶여선 안 된다. 기존 `CountAcrossGroups` 가 GroupTag 로 버킷팅
후 per-bucket 카운트를 합산하는 이유도 동일. 내가 전역 정렬로 바꾸면서 이 전제를
깨뜨렸다.

AFTER 가 정상이었던 것도 단순히 OiL 범위 분리 덕분이지 알고리즘이 맞은 게 아니다.

## 수정

`BatchCounter.AssignBatchIndices` 를 GroupTag 버킷팅 방식으로 재작성.

```csharp
// 1) renderers 를 GroupTag 로 버킷 분할 (예: "Above", "Below", "Root")
// 2) 각 버킷 내에서만 SortKey 정렬 + 연속 동일 BatchKey 런 카운트 → local batchIndex
// 3) 버킷마다 offset 을 더해 전역 유니크 ID 부여 (Above 가 K 개면 Below 는 K+1, K+2...)
// 4) result[origIdx] = offset + localBatchIndex
```

이로써:

- 같은 batch ID 공간에 Above/Below 가 섞이지 않음 → PrefixDistinctCount 의 HashSet
  기반 누적이 그대로 정확.
- 그룹별 배치 수 합계가 `CountAcrossGroups` 와 완전히 일치하여 총합 회귀 0.
- 한 그룹만 있는 경우 (`HasAboveBelow == false`, GroupTag="Root")도 단일 버킷으로
  동작.

## 재검증 (정적)

샘플 프리팹 (Above 6개, Below 2개):

### BEFORE

- Above 버킷 정렬 후 런: Dust03(OiL=0) → 1, hit02(OiL=0) → 2, Dust04(OiL=0) → 3,
  Dust03(OiL=1) → 4, Dust04(OiL=2) → 5, Dust03(OiL=4) → 6
- Below 버킷 정렬 후 런 (offset=6): Dust03(OiL=0) → 7, hit02(OiL=0) → 8
- Hierarchy 순회 prefix distinct-count:
  - above01(1)→1, above03(4)→2, above02(2)→3, above05(6)→4,
    above04_first(3)→5, above04_second(5)→6, below01(7)→**7**, below02(8)→**8** ✓

### AFTER

- Above 버킷 (offset=0): above03(1), above01(1, same mat), above02(2), above05(3),
  above04_first(4), above04_second(4)
- Below 버킷 (offset=4): below01(5), below02(6)
- Hierarchy 순회:
  - above01(1)→1, above03(1)→1, above02(2)→2, above05(3)→3,
    above04_first(4)→4, above04_second(4)→4, below01(5)→5, below02(6)→6 ✓

ver2.csv BEFORE `1,2,3,4,5,6,7,8` / AFTER `1,1,2,3,4,4,5,6` 완전 재현.

## 문서 업데이트

`CLAUDE.md` CSV 누적 컬럼 규칙을 정정: "SortKey 전역 정렬" → "Above/Below 그룹
단위로 정렬 후 offset 으로 전역 유니크 ID 화". Above/Below 가 char 로 인해 별도
렌더 패스라는 도메인 규칙과 정합.

## 회귀 영향

- `CountPrefab` / `CountTotal` 미수정 → Unity 창 배치 수 표시 회귀 없음.
- CSV 컬럼 1-8, 11-12 미변경.
- AFTER 컬럼 (이미 우연히 맞았던 것)도 같은 값으로 유지.
