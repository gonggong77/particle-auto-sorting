# Tasks: 수정 모드에서 Hierarchy Sibling 순서 변경 + 미리보기

- 생성일: 2026-04-27
- Plan: `plans/20260427_hierarchy_reorder_preview.md`
- 별칭: `hierarchy-reorder`, `H-태스크`
- 범위: 기존 `TASKS.md` (Phase 0-7, #1-#17), `tasks/20260424_csv_cumulative_batch_report.md` (R-NN), `tasks/20260426_gpu_instancing_toggle.md` (G-NN) 와 **독립**. 본 파일에서만 추적.
- Task ID 규칙: `H-NN` (Hierarchy reorder + Preview, 본 요청 한정)

## Dependency Graph

```
H-01 PrefabData 플래그 필드 추가 (HasSiblingReorder, NeedsPreviewRefresh)
  ├─ H-02 Table.cs 드래그앤드롭 reorder UI + HierarchyOrder mutate
  │    └─ H-03 List.cs "↻ 미리보기 필요" 뱃지 표시
  ├─ H-04 Bottom.cs 수정 모드일 때 "재계산"→"미리보기" 라벨/툴팁 토글
  └─ H-05 PrefabApplier ApplySiblingOrder + 다이얼로그 메시지 + 플래그 리셋
       └─ H-07 Unity 수동 검증
H-06 Spec/CLAUDE.md 동기화         ← H-02..H-05 와 병렬 가능
H-08 history 기록 + 계획 레포 commit/push  ← H-01..H-07 완료 후
```

---

## H-01 — `PrefabData` 에 reorder dirty 플래그 추가

- **Status**: [x]
- **File**: `D:\particleSortingTest\Assets\Editor\Data\PrefabData.cs`
- **BlockedBy**: 없음
- **변경**:
  ```csharp
  public bool HasSiblingReorder;       // Apply 다이얼로그 카운트, Apply 후 false 리셋
  public bool NeedsPreviewRefresh;     // 미리보기 뱃지, RecomputeAll 후 false 리셋
  ```
- **불변**: 기존 필드/직렬화/생성자 시그니처 그대로. 도메인 리로드 시 휘발 (기존 `OilOverride`/`InstancingOverrides` 와 동일 라이프사이클).

---

## H-02 — 드래그앤드롭 reorder UI + `HierarchyOrder` mutate

- **Status**: [x]
- **File**: `D:\particleSortingTest\Assets\Editor\ParticleAutoSortingWindow.Table.cs` (드래그 상태 필드는 `ParticleAutoSortingWindow.cs` 에 두어도 됨)
- **BlockedBy**: H-01
- **추가 상태 (window 인스턴스 필드)**:
  ```csharp
  struct DragState {
      PrefabData sourceData;
      Transform  sourceParent;       // 같은 부모 swap 가드용
      Transform  draggingTransform;  // 이동 단위
      int        startMouseY;
      Rect       hoverInsertRect;    // Repaint 시 가이드라인
  }
  DragState? _drag;
  ```
- **DrawTableRow 좌측 핸들 cell 추가**:
  - `editMode == true` 일 때만 `≡` (또는 `⋮⋮`) 핸들. 같은 transform 의 첫 행에만 표시 (Trail 등 종속 행은 비표시 — `HierarchyOrder` 소수부 0 인 행만)
  - `EditorGUIUtility.AddCursorRect(handleRect, MouseCursor.MoveArrow)`
- **이벤트 처리** (`Event.current` 기반 IMGUI):
  1. **MouseDown** on 핸들 → `_drag` 설정 (`sourceData`, `sourceParent = transform.parent`, `draggingTransform = transform`), `Event.current.Use()`
  2. **MouseDrag** while `_drag != null` → 같은 `sourceParent` 자식인 다른 행 Rect 와 마우스 Y 비교해 삽입 인덱스 산출, `hoverInsertRect` 갱신, `Repaint()`
  3. **Repaint** → `hoverInsertRect` 자리에 2px 강조 라인
  4. **MouseUp** → 동일 부모 그룹이면 swap 수행, 아니면 no-op. `_drag = null`
- **Swap 로직**:
  1. `data.Renderers` 를 `r.Renderer.transform.parent` 로 그룹핑
  2. `sourceParent` 그룹의 distinct transform 리스트를 현재 `Mathf.Floor(HierarchyOrder)` 기준 정렬
  3. `draggingTransform` 을 빼서 새 인덱스에 삽입
  4. 새 순서대로 정수 카운터 재할당 — 같은 transform 의 모든 RendererInfo 에 같은 정수부, 기존 소수부(Trail +0.5f 등) 보존:
     ```
     float frac = info.HierarchyOrder - Mathf.Floor(info.HierarchyOrder);
     info.HierarchyOrder = newBase + frac;
     ```
  5. `data.HasSiblingReorder = true; data.NeedsPreviewRefresh = true;`
  6. `Repaint()`
- **재사용**:
  - 행 정렬 — `FilterByGroup` (`Table.cs:141~151`) 의 `HierarchyOrder` 비교 정렬을 그대로 따름 (별도 정렬키 도입 X)
  - 섹션/행 그리기 — `DrawSection` / `DrawTableRow` 골격 유지, 좌측 cell 추가만

---

## H-03 — "↻ 미리보기 필요" 뱃지

- **Status**: [x]
- **File**: `D:\particleSortingTest\Assets\Editor\ParticleAutoSortingWindow.List.cs`
- **BlockedBy**: H-01
- **변경**: 프리팹 카드/행 그리기 시 `editMode && data.NeedsPreviewRefresh` 면 작은 뱃지(예: 주황색 pill 텍스트 "↻ 미리보기 필요") 추가. 위치는 기존 경고 pill (인터리브 등) 옆.
- **재사용**: `DrawPill` 패턴 (`List.cs` 기존 pill 그리기와 동일 스타일)
- **수정 모드 OFF 시**: 비표시.

---

## H-04 — 미리보기 버튼 라벨 토글

- **Status**: [x]
- **File**: `D:\particleSortingTest\Assets\Editor\ParticleAutoSortingWindow.Bottom.cs`
- **BlockedBy**: H-01 (플래그 의존 없음, 단독 가능)
- **변경**: 기존 "재계산" 버튼의 라벨/툴팁을 `editMode` 분기:
  - ON: 라벨 "미리보기", 툴팁 "현재 변경된 순서/오버라이드 기준으로 OiL · Fudge · 배치수를 다시 계산합니다. 프리팹은 수정되지 않습니다."
  - OFF: 라벨 "재계산" 그대로
- **핸들러**: 기존 `RecomputeAll()` 그대로 호출 (분기 없음)
- **`RecomputeAll()` 마지막에 추가**: `foreach (var d in prefabs) d.NeedsPreviewRefresh = false;`
  → `ParticleAutoSortingWindow.cs:251~259` 에 한 줄 추가 (이 변경만 `Window.cs` 에 영향)

---

## H-05 — Apply 파이프라인 확장

- **Status**: [x]
- **File**: `D:\particleSortingTest\Assets\Editor\Apply\PrefabApplier.cs`
- **BlockedBy**: H-01
- **삽입 지점 1 — `Apply` 진입부 (확인 다이얼로그 직전)**:
  - `RecomputeAll()` 한 번 호출(또는 동등 효과)로 미리보기 미클릭 케이스 보장.
  - 호출 경로: `PrefabApplier` 가 `ParticleAutoSortingWindow` 를 직접 참조하면 그 메서드 호출. 직접 참조 어려우면 `targets` 각각에 대해 `SortingOptimizer.Optimize` + `BatchCounter.CountPrefab` 직접 호출.
- **삽입 지점 2 — 다이얼로그 메시지 (PrefabApplier.cs:109~)**:
  - "· 형제 순서 변경 프리팹: M개"  (M = `targets.Count(t => t.HasSiblingReorder)`)
  - "· 전체 배치 수: {sumBefore} → {sumAfter}"  (`targets` 의 `BatchBefore`/`BatchAfter` 합)
- **삽입 지점 3 — `ApplySingle(data)` 시작부 (PrefabApplier.cs:155~)**:
  - `ApplySiblingOrder(data);` 호출
- **`ApplySiblingOrder(PrefabData data)` 신규**:
  1. `data.Renderers` 를 `r.Renderer.transform.parent` 로 그룹핑
  2. 각 그룹에서 `Mathf.Floor(HierarchyOrder)` 기준 distinct transform 리스트 (= 새 순서)
  3. 같은 부모의 자식 transform 전체를 `parent.childCount` 로 가져와 "Renderer 가 있는 자식의 현재 sibling slot" 위치만 수집 → 그 슬롯들에 새 순서대로 채워 넣음 (= **Renderer 없는 자식의 절대 위치 유지**)
  4. 각 transform 에 대해 현재 `GetSiblingIndex()` ≠ 목표 인덱스면 `Undo.RegisterCompleteObjectUndo(transform.parent, "Reorder Siblings")` 후 `transform.SetSiblingIndex(targetIdx)`
  5. (저장은 기존 `PrefabUtility.SavePrefabAsset(data.Prefab)` 가 함께 처리)
- **삽입 지점 4 — ApplySingle 성공 분기 끝**:
  - `data.HasSiblingReorder = false;`
  - `data.NeedsPreviewRefresh = false;`
- **불변**:
  - 기존 `Undo.RegisterCompleteObjectUndo` (PrefabApplier.cs:169) 와 같은 Undo 그룹에 sibling 변경도 포함되도록 같은 흐름에서 처리
  - `PrefabUtility.SavePrefabAsset(data.Prefab)` (PrefabApplier.cs:185) 한 번이 sibling 변경까지 함께 저장
  - GPU Instancing 토글 흐름 (G-03 결과) 그대로 보존

---

## H-06 — Spec / CLAUDE.md 동기화

- **Status**: [x]
- **Files**:
  - `D:\particle-auto-sorting\particle-auto-sorting\particle_auto_sorting_spec.json`
  - `D:\particle-auto-sorting\particle-auto-sorting\CLAUDE.md` (옵션)
- **BlockedBy**: H-02 / H-05 와 병렬 가능
- **spec.json 추가 항목** (Renderer 테이블 행 정의 옆):
  ```json
  {
    "id": "hierarchy_reorder_handle",
    "label": "행 순서 변경 핸들 (수정 모드)",
    "visible_when": "edit_mode_toggle == ON",
    "scope": "same-parent siblings only",
    "unit": "GameObject (transform) — same-transform RendererInfo move together",
    "control": "drag-and-drop reorder",
    "applied_at": "기존 [적용] 버튼과 같은 Undo 그룹"
  },
  {
    "id": "preview_button",
    "label": "미리보기 (수정 모드일 때 '재계산' 라벨 대체)",
    "action": "RecomputeAll — OiL/Fudge/BatchAfter 재계산 (디스크 무변경)",
    "dirty_indicator": "row 'NeedsPreviewRefresh' 뱃지"
  }
  ```
- **CLAUDE.md 추가 (옵션)**: 도메인 핵심 규칙 끝에
  `- **Hierarchy Sibling Reorder (수정 모드)**: 같은 부모 자식 사이만, 드래그앤드롭, [적용] 시 SetSiblingIndex 일괄, [미리보기] 는 RecomputeAll 만.`

---

## H-07 — Unity 수동 검증

- **Status**: [ ]
- **BlockedBy**: H-02 + H-03 + H-04 + H-05
- **체크리스트**:
  - [ ] 수정 모드 OFF → 행 좌측 `≡` 핸들 비표시, "↻ 미리보기 필요" 뱃지 비표시
  - [ ] 수정 모드 ON → 핸들 드래그 시 같은 부모 그룹 내에서만 2px 가이드라인 표시
  - [ ] 드롭 후 행 순서만 즉시 swap, OiL/Fudge/배치수 셀은 옛 값 유지, "↻ 미리보기 필요" 뱃지 출현
  - [ ] [미리보기] 클릭 → OiL/Fudge/배치수 재계산, 뱃지 사라짐, Project 창에서 프리팹 mtime 변화 없음
  - [ ] 다른 부모 그룹 위로 드롭 → no-op
  - [ ] ParticleSystem 행 이동 시 같은 GameObject 의 Trail 행 동반 이동, Trail 행 자체는 핸들 없음
  - [ ] [적용] 다이얼로그에 "형제 순서 변경 프리팹: M개" + "전체 배치 수: a → b" 라인 표시
  - [ ] 적용 후 Unity Hierarchy 의 sibling 순서가 UI 와 일치
  - [ ] Renderer 없는 자식 transform 의 절대 위치 유지
  - [ ] **Ctrl+Z** 한 번에 sibling 순서 + OiL/Fudge + (있다면) Material instancing 동시 롤백
  - [ ] 미리보기 미클릭 후 바로 [적용] → Apply 가 최신 값으로 처리됨 (다이얼로그 배치수가 최신)
  - [ ] 같은 부모 밑 RendererInfo transform 1개뿐 → 핸들 표시되지만 swap 대상 없으면 no-op (뱃지 미발생)
  - [ ] Above/Below 섹션 경계 cross-section drop 자동 차단
  - [ ] Unity Console 컴파일 에러 0개
  - [ ] (회귀) GPU Instancing 토글(G-NN), 인터리브 경고, 오버플로 차단, CSV export 정상

---

## H-08 — history 기록 + 계획 레포 commit/push

- **Status**: [ ]
- **BlockedBy**: H-01 ~ H-07
- **산출물**:
  - `history/YYYYMMDD_HHMM_hierarchy_reorder_preview.md` (증상/원인/수정/검증)
  - 계획 레포에서 `plans/20260427_hierarchy_reorder_preview.md`, `tasks/20260427_hierarchy_reorder_preview.md`, `particle_auto_sorting_spec.json`, `CLAUDE.md`(변경 시), `history/...md` 를 `git add` → commit → push
  - **Unity 레포(`D:\particleSortingTest`) 는 사용자가 직접 커밋** (CLAUDE.md 룰)
- **커밋 메시지 초안**:
  ```
  feat: 수정 모드에서 Hierarchy sibling reorder + 미리보기 추가

  - PrefabData 에 HasSiblingReorder / NeedsPreviewRefresh 플래그 신설
  - Table.cs 행 좌측 ≡ 핸들 드래그앤드롭, 같은 부모 자식 사이만 swap, 같은 transform RendererInfo 동반 이동, HierarchyOrder mutate
  - Bottom.cs 수정 모드 ON 일 때 "재계산"→"미리보기" 라벨 토글 (핸들러는 RecomputeAll 동일)
  - List.cs reorder dirty 행에 "↻ 미리보기 필요" 뱃지
  - PrefabApplier: ApplySiblingOrder 신규(같은 부모 자식 슬롯 보존 매핑), 다이얼로그에 "형제 순서 변경 프리팹: M개" + "전체 배치 수: a→b" 추가, 같은 Undo 그룹/SavePrefabAsset 재사용, 미리보기 미클릭 케이스 보장 위해 Apply 진입부 RecomputeAll
  - spec.json 에 hierarchy_reorder_handle / preview_button 추가
  ```

---

## 비변경 영역 (침범 금지)

- 기존 `TASKS.md` (Phase 0-7, #1-#17)
- `tasks/20260424_csv_cumulative_batch_report.md` (R-NN)
- `tasks/20260426_gpu_instancing_toggle.md` (G-NN)
- `SortingOptimizer.cs`, `BatchCounter.cs`, `CsvReportExporter.cs` — 본 건은 sibling index 쓰기에 한정
- 기존 OiL/Fudge UI/적용/오버플로 차단/인터리브 경고 흐름
- `Trail_Module skip` (`PrefabApplier.cs:111`)
- `PrefabAnalyzer` 의 DFS 순회 / Trail +0.5f 오프셋 규칙
- short 범위 클램프 금지 (Spec)
