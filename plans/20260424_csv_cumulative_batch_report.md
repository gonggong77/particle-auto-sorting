# Plan: CSV 보고서 누적 배치 수 컬럼화 (Frame Debugger 스타일)

- 생성일: 2026-04-24
- 원본 플랜: `C:\Users\lhg07\.claude\plans\lovely-growing-treasure.md` (보관용 복제본)
- 관련 태스크: TASKS.md Phase 8 (#18–#21)

## Context

Particle Auto Sorting 툴의 CSV 보고서(`particle_auto_sorting_report.csv`)가 현재 "프리팹 배치 수 before/after" 컬럼에 프리팹 전체 총합만 모든 행에 동일하게 반복 출력 중. 사용자가 원하는 것은 **Unity Frame Debugger 처럼** 각 행에서 "여기까지 누적 배치 수가 몇 개인지" 보여주는 것. 즉, 행마다 단조 비감소하는 prefix counter.

`particle-report-ver1.csv`(현재) ↔ `particle-report-ver2.csv`(목표) 비교 결과:
- 헤더: `프리팹 배치 수 before/after` → `프리팹 내 누적 배치 수 before/after`
- 값: 프리팹 단위 고정 총합 → 행 단위 누적 값
- 컬럼 11-12 `전체 배치 수`는 그대로 (전 프리팹 합산 고정)

의도된 결과: Frame Debugger 처럼 보고서 첫 행부터 마지막 행으로 스캔하면서 "이 행 시점의 배치 개수"가 점진적으로 늘어가는 모습을 볼 수 있게 되어, 어떤 렌더러가 새 batch를 유발했는지 읽기 쉬워짐.

## 누적 배치 수 알고리즘 (두 CSV 대조로 역산 검증 완료)

주어진 `PrefabData.Renderers` (Hierarchy 순서) 에 대해:

1. 모든 렌더러를 SortKey `[SortingLayerID, OiL, Fudge, HierarchyOrder]` 로 정렬 (before / after 각각 별도 정렬).
2. 정렬 후 연속 동일 BatchKey `[SortingLayerID, SharedMaterial, RenderMode, Mesh]` 구간마다 전역 batch index (1, 2, 3, ...) 부여. 이 index 는 렌더러별 속성처럼 저장.
3. **Hierarchy 순서**로 다시 순회하면서, 지금까지 본 batch index 의 distinct 개수를 누적 카운터로 emit.

검증 (ver2 after 컬럼):
- sort 후 batch index: below01→1, below02→2, above01→3, above03→3, above02→4, above05→5, above04_first→6, above04_second→6
- hierarchy 순회: above01(seen={3},1), above03(seen={3},1), above02(seen={3,4},2), above05(seen={3,4,5},3), above04_first(seen={3,4,5,6},4), above04_second(seen={3,4,5,6},4), below01(seen={1,3,4,5,6},5), below02(seen={1,2,3,4,5,6},6) → `1,1,2,3,4,4,5,6` ✓

**프리팹마다 독립 리셋** (헤더 문구 "프리팹 내"가 명시). 여러 프리팹 export 시 각 프리팹의 첫 행부터 다시 1부터 시작.

## 변경 파일

### 1) `D:\particleSortingTest\Assets\Editor\Analysis\BatchCounter.cs`

기존 `CountPrefab(PrefabData)` 은 단일 int 를 `data.BatchBefore/BatchAfter` 에 쓰는 형태. 여기에 **per-row 누적 시퀀스**를 반환하는 메서드 추가:

```csharp
public static (int[] beforeCum, int[] afterCum) CumulativeBatchesPerRenderer(PrefabData data);
// 반환 배열 길이 = data.Renderers.Count, 인덱스는 data.Renderers 의 hierarchy 순서와 일치
```

내부 로직:
- 헬퍼 `AssignBatchIndices(List<RendererInfo>, bool useAfter) → int[]`:
  1. 렌더러를 SortKey 로 정렬 (기존 `CountPrefab` 이 이미 수행하는 것과 같은 키). 현재 구현을 추출해서 재사용.
  2. 연속 동일 BatchKey 마다 batchIndex 증가. 각 렌더러에 batchIndex 기록 (원래 `data.Renderers` 순서로 복원한 배열로 리턴).
- `CumulativeBatchesPerRenderer`:
  - beforeIdx = AssignBatchIndices(..., useAfter:false)
  - afterIdx  = AssignBatchIndices(..., useAfter:true)
  - 각각 prefix distinct-count 를 HashSet 으로 누적해 `beforeCum`, `afterCum` 배열 리턴.

기존 `CountPrefab`/`CountTotal` 시그니처는 그대로 유지 (전체 배치 수 컬럼 11-12 는 계속 필요). `CountPrefab` 내부의 정렬·BatchKey 비교 로직을 `AssignBatchIndices` 에 위임하고, 총합은 `beforeIdx.Max() / afterIdx.Max()` 로 도출하도록 리팩터링해도 되고, 중복 구현이어도 작은 규모라 둘 다 허용.

### 2) `D:\particleSortingTest\Assets\Editor\Report\CsvReportExporter.cs`

- 헤더 문구 2곳 교체:
  - `프리팹 배치 수 before` → `프리팹 내 누적 배치 수 before`
  - `프리팹 배치 수 after`  → `프리팹 내 누적 배치 수 after`
- 각 프리팹 block 시작 시 `var (beforeCum, afterCum) = BatchCounter.CumulativeBatchesPerRenderer(p);` 호출.
- 렌더러 row loop 에서 기존 `p.BatchBefore / p.BatchAfter` 자리에 `beforeCum[i] / afterCum[i]` 기록.
- 컬럼 11-12 `전체 배치 수 before/after` 는 기존 totalBefore/totalAfter 그대로 유지.
- UTF-8 BOM, 이스케이프, 줄바꿈 기존 규칙 준수.

### 3) `D:\particle-auto-sorting\particle-auto-sorting\CLAUDE.md`

"도메인 핵심 규칙" 또는 "CSV 보고서" 관련 기존 문구가 있으면 컬럼명 업데이트. 없으면 워크플로 섹션 하단에 한 줄:
- `CSV 보고서의 "프리팹 내 누적 배치 수" 컬럼은 Hierarchy 순서로 row 별 prefix 배치 수 (Frame Debugger 방식)`.

### (선택) `D:\particle-auto-sorting\particle-auto-sorting\particle_auto_sorting_spec.json`

Spec 에 CSV 컬럼 리스트가 있으면 2개 헤더 문구 정정. 없으면 skip.

## 재사용할 기존 코드

- `BatchCounter.CountPrefab` 내부의 SortKey 정렬 비교자 + BatchKey 동등성 판정 로직. 동일 기준으로 batch index 를 매겨야 하므로 반드시 재사용.
- `RendererInfo` 의 OilOverride/FudgeOverride 처리 (`OilOverride ?? OilAfterAI` 등). after 패스에서 동일하게 적용.
- `PrefabData.Renderers` 의 hierarchy 순서 가정. (PrefabAnalyzer DFS 로 이미 보장됨.)

## 비변경 범위

- 컬럼 1-8, 11-12 값/의미 불변.
- Unity 윈도우 UI (테이블/배치 수 표시) 불변. 누적 배치 수는 CSV 전용.
- `SortingOptimizer` 변경 없음 (직전 런 기반 할당 그대로).

## 엣지 케이스

- 렌더러 0개 프리팹: header 만 쓰이고 data 행 없음 → 기존 동작 유지.
- 렌더러 1개 프리팹: beforeCum=[1], afterCum=[1].
- 여러 프리팹 export: 프리팹마다 1부터 리셋.
- null material: `PrefabAnalyzer` 단계에서 silent skip 되어 `data.Renderers` 에 아예 없음 → 별도 처리 불필요.
- 인터리브 케이스: 알고리즘 특성상 자연스럽게 반영됨 (같은 material 이 여러 run 으로 쪼개지면 batch index 도 분리됨).

## 검증 (end-to-end)

1. Unity Editor 에서 샘플 프리팹(`D:\particleSortingTest\...`) 드래그 → 자동 정렬 실행.
2. "CSV 내보내기" 버튼으로 report 생성.
3. 생성 파일을 `particle-report-ver2.csv` 와 diff:
   - 헤더 9-10 문구 일치
   - 컬럼 9-10 값 시퀀스가 ver2 와 동일 (1,2,3,4,5,6,7,8 / 1,1,2,3,4,4,5,6)
   - 컬럼 11-12 는 여전히 모든 행에서 8 / 6
4. 여러 프리팹을 한꺼번에 넣고 export → 각 프리팹 첫 행이 1 로 리셋되는지 확인.
5. 오브젝트 하나만 있는 프리팹 → 1 이 한 번 나오고 다음 프리팹 행에서 리셋되는지.
6. (회귀) Unity 창의 배치 수 표시(before/after) 숫자가 이전과 동일한지 확인.
7. 작업 종료 시 `history/YYYYMMDD_HHMM_<슬러그>.md` 작성, 계획 레포 add/commit/push.

## 커밋 메시지 초안

```
feat: CSV 리포트에 Frame Debugger 스타일 누적 배치 수 컬럼 도입

- 컬럼 9-10 헤더: '프리팹 배치 수' → '프리팹 내 누적 배치 수'
- 값: 프리팹 단위 총합 반복 → 행 단위 누적 (hierarchy 순서 prefix count)
- BatchCounter.CumulativeBatchesPerRenderer 신설 (기존 SortKey/BatchKey 재사용)
- 전체 배치 수 (컬럼 11-12) 는 기존 동작 그대로 유지
```
