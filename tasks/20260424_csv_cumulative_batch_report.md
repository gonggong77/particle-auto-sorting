# Tasks: CSV 보고서 누적 배치 수 컬럼화

- 생성일: 2026-04-24
- Plan: `plans/20260424_csv_cumulative_batch_report.md`
- 범위: 기존 `TASKS.md` Phase 0-7 와 독립. 이 파일에서만 추적.
- Task ID 규칙: `R-NN` (Report 관련, 본 요청 한정)

## Dependency Graph

```
R-01 BatchCounter 확장 (CumulativeBatchesPerRenderer)
  └─ R-02 CsvReportExporter 헤더·값 업데이트
       └─ R-04 Unity 수동 검증
R-03 문서 업데이트 (CLAUDE.md, spec.json)        ← 병렬 가능
R-05 history 기록 + 계획 레포 commit/push        ← R-01..R-04 완료 후
```

---

## R-01 — BatchCounter 에 누적 배치 수 반환 메서드 추가

- **Status**: [ ]
- **File**: `D:\particleSortingTest\Assets\Editor\Analysis\BatchCounter.cs`
- **BlockedBy**: (없음, 기존 `#5 BatchCounter` 위에 확장)
- **시그니처**:
  ```csharp
  public static (int[] beforeCum, int[] afterCum) CumulativeBatchesPerRenderer(PrefabData data);
  ```
  - 반환 배열 길이 = `data.Renderers.Count`
  - 인덱스는 `data.Renderers` 의 hierarchy 순서와 일치
- **내부 로직**:
  - 헬퍼 `AssignBatchIndices(List<RendererInfo>, bool useAfter) → int[]`
    1. SortKey `[SortingLayerID, OiL, Fudge, HierarchyOrder]` 정렬 (useAfter 이면 `OilOverride ?? OilAfterAI`, `FudgeOverride ?? FudgeAfterAI`)
    2. 연속 동일 BatchKey `[SortingLayerID, SharedMaterial, RenderMode, Mesh]` 구간마다 batchIndex 증가
    3. 원래 `data.Renderers` 인덱스 순서로 복원한 배열 리턴
  - `CumulativeBatchesPerRenderer`
    - `beforeIdx = AssignBatchIndices(list, useAfter:false)`
    - `afterIdx  = AssignBatchIndices(list, useAfter:true)`
    - 각각 HashSet 으로 prefix distinct-count 누적 → `beforeCum`, `afterCum`
- **기존 API 유지**: `CountPrefab`, `CountTotal` 시그니처 불변. 정렬/BatchKey 비교 로직은 가능하면 `AssignBatchIndices` 에 일원화.
- **검증 샘플**: ver2 after 컬럼 결과 `1,1,2,3,4,4,5,6` 재현.

---

## R-02 — CsvReportExporter 헤더 문구 및 행별 값 교체

- **Status**: [ ]
- **File**: `D:\particleSortingTest\Assets\Editor\Report\CsvReportExporter.cs`
- **BlockedBy**: R-01
- **헤더 변경** (2곳):
  - `프리팹 배치 수 before` → `프리팹 내 누적 배치 수 before`
  - `프리팹 배치 수 after`  → `프리팹 내 누적 배치 수 after`
- **행별 값**:
  - 각 프리팹 block 시작 시 `var (beforeCum, afterCum) = BatchCounter.CumulativeBatchesPerRenderer(p);`
  - 렌더러 루프 인덱스 `i` 에서 기존 `p.BatchBefore / p.BatchAfter` 자리를 `beforeCum[i] / afterCum[i]` 로 교체
- **불변**:
  - 컬럼 1-8: 그대로
  - 컬럼 11-12 (전체 배치 수 before/after): `totalBefore` / `totalAfter` 그대로
  - UTF-8 BOM, 쉼표/줄바꿈/따옴표 이스케이프 규칙 유지

---

## R-03 — 문서 업데이트

- **Status**: [ ]
- **Files**:
  - `D:\particle-auto-sorting\particle-auto-sorting\CLAUDE.md`
  - `D:\particle-auto-sorting\particle-auto-sorting\particle_auto_sorting_spec.json` (해당 필드 존재 시만)
- **BlockedBy**: R-02 와 병행 가능 (문구 확정만 되어 있으면 됨)
- **변경**:
  - CLAUDE.md: CSV 컬럼 언급부가 있으면 헤더명 정정. 없으면 간단한 한 줄 (`CSV 보고서 "프리팹 내 누적 배치 수" 컬럼은 Hierarchy 순서 prefix 배치 수 (Frame Debugger 방식)`) 추가.
  - spec.json: CSV 컬럼 리스트가 있으면 2개 문구 정정. 없으면 skip.

---

## R-04 — Unity 수동 검증

- **Status**: [ ]
- **BlockedBy**: R-02
- **체크**:
  - [ ] 샘플 프리팹 드래그 → 자동 정렬 → CSV export → `particle-report-ver2.csv` 와 diff 시 헤더 9-10 문구 일치
  - [ ] 컬럼 9-10 값 시퀀스가 ver2 와 동일 (before: `1,2,3,4,5,6,7,8` / after: `1,1,2,3,4,4,5,6`)
  - [ ] 컬럼 11-12 는 모든 행에서 동일 (예제 기준 `8` / `6`)
  - [ ] 여러 프리팹 export 시 프리팹마다 첫 행이 `1` 로 리셋
  - [ ] 렌더러 1개 프리팹 → `1` 한 번, 다음 프리팹에서 리셋
  - [ ] (회귀) Unity 창의 배치 수 표시(before/after) 값이 이전과 동일
  - [ ] (회귀) Undo 한 번에 복원

---

## R-05 — history 기록 + 계획 레포 commit/push

- **Status**: [ ]
- **BlockedBy**: R-01 ~ R-04
- **산출물**:
  - `history/YYYYMMDD_HHMM_csv_cumulative_batches.md` (증상/원인/수정/검증)
  - CLAUDE.md / spec.json / history 파일을 `git add` → commit → push
- **커밋 메시지 초안**:
  ```
  feat: CSV 리포트에 Frame Debugger 스타일 누적 배치 수 컬럼 도입

  - 컬럼 9-10 헤더: '프리팹 배치 수' → '프리팹 내 누적 배치 수'
  - 값: 프리팹 단위 총합 반복 → 행 단위 누적 (hierarchy 순서 prefix count)
  - BatchCounter.CumulativeBatchesPerRenderer 신설 (기존 SortKey/BatchKey 재사용)
  - 전체 배치 수 (컬럼 11-12) 는 기존 동작 그대로 유지
  ```

---

## 비변경 영역 (침범 금지)

- `TASKS.md` (기존 Phase 0-7, #1-#17)
- `SortingOptimizer.cs` (직전 런 기반 할당 그대로)
- Unity 윈도우 UI 테이블/배치 수 표시 (CSV 전용 변경)
- 컬럼 1-8, 11-12 의미/값
