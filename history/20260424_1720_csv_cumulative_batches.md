# CSV 리포트 "프리팹 내 누적 배치 수" 컬럼 도입

- 날짜: 2026-04-24 17:20
- Plan: `plans/20260424_csv_cumulative_batch_report.md`
- Tasks: `tasks/20260424_csv_cumulative_batch_report.md` (R-01 ~ R-05)

## 증상 (Before)

`particle_auto_sorting_report.csv` 의 컬럼 9-10 "프리팹 배치 수 before/after" 가
프리팹 전체 총합을 모든 행에 동일하게 반복 출력. 사용자 요구는 Unity Frame
Debugger 처럼 "각 행까지의 누적 배치 수" 를 보여주는 단조 비감소 prefix 카운터.

`particle-report-ver1.csv` (현재) 대 `particle-report-ver2.csv` (목표) 비교로
알고리즘을 역산 검증했고, ver2 의 after 컬럼 시퀀스 `1,1,2,3,4,4,5,6` 재현이 목표.

## 원인

`CsvReportExporter` 가 `p.BatchBefore / p.BatchAfter` (프리팹 총합) 을 각 row
마다 그대로 쓰고 있어 "누적" 이 아닌 "고정 총합" 이 출력됨.

## 수정

### 1) `BatchCounter.cs` — 누적 배치 수 메서드 추가

```csharp
public static (int[] beforeCum, int[] afterCum) CumulativeBatchesPerRenderer(PrefabData data);
```

- 헬퍼 `AssignBatchIndices(List<RendererInfo>, bool useAfter) → int[]`
  1. 모든 렌더러를 SortKey `[SortingLayerID, OiL, Fudge, HierarchyOrder]` 로
     전역 정렬 (useAfter 이면 `OilOverride ?? OilAfterAI`, `FudgeOverride ?? FudgeAfterAI`).
  2. 연속 동일 BatchKey `[SortingLayerID, SharedMaterial, RenderMode, Mesh]`
     구간마다 batchIndex 증가 (1 부터).
  3. `data.Renderers` 원래 인덱스 순서로 복원한 배열 리턴.
- `CumulativeBatchesPerRenderer` 는 before/after 각각에 대해 AssignBatchIndices
  호출 후 `PrefixDistinctCount` 로 HashSet 기반 prefix distinct-count 누적.
- 기존 `CountPrefab`, `CountTotal`, `CountAcrossGroups`, `CountSingleGroup`,
  `CompareSortKey`, `ResolveOil`, `ResolveFudge`, `BuildBatchKey`, `BatchKey`
  는 **전부 그대로 유지**. 새 로직이 기존 정렬/키 헬퍼를 재사용하도록 private
  메서드는 수정하지 않음 → UI 배치 수 회귀 0.

### 2) `CsvReportExporter.cs` — 헤더 + 행별 값 교체

- 헤더 2곳:
  - `프리팹 배치 수 before` → `프리팹 내 누적 배치 수 before`
  - `프리팹 배치 수 after`  → `프리팹 내 누적 배치 수 after`
- 각 프리팹 블록 시작 시 `BatchCounter.CumulativeBatchesPerRenderer(p)` 호출.
- 렌더러 루프를 `for (int i ...)` 로 바꿔 인덱스 `i` 에서 `beforeCum[i]` /
  `afterCum[i]` 를 기록.
- `ParticleAutoSorting.Editor.Analysis` using 추가.
- 컬럼 1-8, 11-12 (전체 배치 수 before/after) 는 기존 동작 그대로.
- 렌더러 0개 프리팹의 컬럼 9-10 은 빈 문자열 (누적의 의미가 없음). 기존엔
  프리팹 총합을 썼으나, 컬럼 의미가 "내부 누적" 으로 바뀌었으므로 empty 가
  더 자연스러움.

### 3) 문서

- `CLAUDE.md` "도메인 핵심 규칙" 에 CSV 누적 컬럼 설명 한 줄 추가.
- `particle_auto_sorting_spec.json` `csv_contents` 문구 정정 (`프리팹별 배치 수`
  → `프리팹 내 누적 배치 수`).

## 검증

- [x] (정적) 계획서 알고리즘 재검토: sort 후 batch index 와 prefix distinct-count
      계산이 ver2 after 컬럼 `1,1,2,3,4,4,5,6` 와 수식적으로 일치.
- [x] (정적) `CountPrefab`/`CountTotal` 경로 비변경 → Unity 창의 배치 수 표시
      (before/after) 회귀 없음.
- [ ] (수동) Unity Editor 에서 샘플 프리팹 드래그 → CSV export → ver2.csv 와 diff
      - 헤더 9-10 문구 일치
      - 컬럼 9-10 값 before `1..8`, after `1,1,2,3,4,4,5,6`
      - 컬럼 11-12 모든 행에서 `8` / `6`
- [ ] (수동) 여러 프리팹 동시 export 시 프리팹마다 첫 행 `1` 리셋.
- [ ] (수동) 렌더러 1개 프리팹 → `1` 한 번, 다음 프리팹에서 리셋.
- [ ] (수동) Undo 1회로 복원.

## 비변경 영역

- `TASKS.md` Phase 0-7.
- `SortingOptimizer` (직전 런 기반 할당 그대로).
- Unity 윈도우 UI 테이블/배치 수 표시.
- CSV 컬럼 1-8, 11-12 의미/값.
