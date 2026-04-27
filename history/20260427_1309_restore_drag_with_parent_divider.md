# 2026-04-27 13:09 — 드래그앤드롭 reorder 복원 + 부모 경계 구분선/라벨 추가

## 배경

직전 04:32 패치 (`20260427_0432_replace_drag_with_arrow_buttons.md`) 에서 드래그 간헐 막힘 때문에 드래그를 ▲/▼ 버튼으로 교체했지만, 사용자 검증 결과 reorder 가 막힌 것처럼 느껴진 케이스 대다수는 **같은 부모 안에서만 sibling 순서 변경이 허용**된다는 설계 제약이 UI 상에 드러나지 않은 데서 비롯됨이 확인됨. 사용자는 화살표 클릭보다 드래그앤드롭 UX 를 선호하며, 다른 부모임을 시각적으로 인지할 수만 있다면 드래그가 막혀 보이는 체감 자체가 사라질 것이라고 판단.

## 변경 사항

### `Window.cs`

- `DragState` 중첩 클래스 복원 (`data`/`sourceParent`/`draggingTransform`/`hoverInsertIndex`/`hoverInsertRect`/`controlId`)
- `DragState _drag;` 필드 추가
- `OnGUI` 끝에 `HandleDragCleanup()` 호출 추가
- `HandleDragCleanup()` 신설 — `MouseUp` 또는 `MouseLeaveWindow` 시 `_drag.controlId == GUIUtility.hotControl` 일 때만 hotControl 해제 후 `_drag = null` + Repaint
- 수정 모드 토글 툴팁("≡ 핸들로 …") 그대로 유지 (이미 드래그용 문구)

### `Table.cs`

- `ColHandle` 38f → **18f** (≡ 핸들 1개 폭으로 복원)
- 신규 상수: `DragGuideColor` (드롭 가이드 라인용), `ParentDividerColor` (부모 경계선용)
- **제거**: `DrawReorderButtons`, `GetSiblingPosition`, `MoveSibling`
- **신규**:
  - `DrawDragHandle(PrefabData, Transform)` — `≡` 라벨 + cursor rect + MouseDown 시 `GUIUtility.GetControlID(FocusType.Passive)` → `_drag` 생성 → `GUIUtility.hotControl = id;` → `evt.Use()`. (04:08 패치의 hotControl 점유 적용)
  - `HandleSiblingDrag(PrefabData, List<Transform>, Dictionary<Transform, Rect>)` — `_drag.sourceParent` 와 동일 부모 sibling 만 후보로, mouseY 기준 insert 인덱스 산출, 가이드 라인 위치 계산, Repaint 시 2px 라인 그리기, MouseDrag 에서 evt.Use(), MouseUp 에서 `ApplyReorder` 호출 후 hotControl 해제
  - `ApplyReorder(PrefabData, Transform, int)` — 같은 부모 sibling 정수부 slot 추출 → target 을 `insertIndex` 위치로 reinsert (이동 후 oldIdx 보정 위해 `if (newIdx > oldIdx) newIdx--;`) → `r.HierarchyOrder = base + frac` 갱신 → `HasSiblingReorder` / `NeedsPreviewRefresh` 셋
  - `DrawParentDivider(Transform parent)` — 1px 가로선 + `▼ {parentName}` miniLabel 라벨, 툴팁 "부모 Transform 경계 — 위/아래 행은 부모가 달라 서로 드래그로 이동할 수 없습니다."
- **`DrawSection` 변경** — 직전 isFirst 행의 `transform.parent` 를 `prevParent` 로 추적하다 변경 시 `DrawParentDivider` 삽입. `orderedTransforms` 리스트 + `rowRectsByTransform` Dictionary 수집 (각 Transform 의 모든 행 rect 합집합). 끝에 `editMode && _drag != null && _drag.data == data` 일 때 `HandleSiblingDrag` 호출.
- **`DrawTableRow` 변경** — `using` 스코프를 `BeginHorizontal()`/`EndHorizontal()` 로 바꿔 행 Rect 반환. `DrawReorderButtons` 호출 자리를 `DrawDragHandle` 로 교체.

### Trail 행 처리

- 각 Transform 의 첫 행만 `≡` 핸들이 있고 (`isFirstRowOfTransform == true`), Trail_Module/Trail_Component 행은 기존대로 `Space(ColHandle)`
- `rowRectsByTransform[t]` 는 Particle 행 + Trail 행들 rect 의 합집합으로 저장 → 드롭 가이드라인이 블록 단위 경계에 정확히 표시됨
- `ApplyReorder` 의 `frac` 보존 로직은 04:32 의 `MoveSibling` 과 동일

### 부모 경계 시각화 동작

- 수정모드 ON/OFF 무관하게 항상 표시 (다른 부모임을 식별하는 용도)
- Above/Below 섹션 헤더는 그대로 유지, 섹션 *내부*의 부모 경계만 새 라벨로 표시
- 라벨 인덴트: `editMode` 시 `ColHandle + ColWarn`, 아니면 `ColWarn` 만큼 들여쓰기 → 행 내용 시작 위치에 정렬

## 변경 파일

- `D:\particleSortingTest\Assets\Editor\ParticleAutoSortingWindow.cs`
- `D:\particleSortingTest\Assets\Editor\ParticleAutoSortingWindow.Table.cs`
- `D:\idleDante_test\Assets\Editor\ParticleAutoSorting\ParticleAutoSortingWindow.cs`
- `D:\idleDante_test\Assets\Editor\ParticleAutoSorting\ParticleAutoSortingWindow.Table.cs`

두 작업 사본은 `cp` 로 동일 동기화. `diff -q` 로 무차이 확인 완료.

## 보존된 동작

- cross-parent reorder 차단 (`t.parent != target.parent` 필터) — 의도적으로 유지하되 부모 경계 라벨로 안내
- Trail 행 `+0.5f` frac 보존 — `ApplyReorder` 가 04:32 `MoveSibling` 과 동일 로직
- `HasSiblingReorder` / `NeedsPreviewRefresh` 플래그 → "↻ 미리보기 필요" 뱃지, [미리보기]/[적용] 흐름 그대로
- 04:08 hotControl 점유 패치 (`controlId` 필드, MouseDown 시 점유, MouseUp/Cleanup 에서 ID 일치 시만 해제)

## 사용자 검증 포인트

1. 같은 부모 sibling 2개 이상인 프리팹에서 `≡` 핸들 잡고 드래그 → 가이드 라인 따라 이동, 드롭 즉시 반영
2. 부모가 다른 행 사이 — 항상 구분선 + `▼ ParentName` miniLabel 표시, 수정모드 OFF 에서도
3. ScrollView 내부 빠른 드래그 반복 시 멈춤 없음
4. 드래그 도중 OilAfter/FudgeAfter `IntField`/`FloatField` 위로 마우스 통과해도 끊기지 않음
5. ScrollView 밖에서 MouseUp → 다른 행 클릭 정상
6. ParticleSystem + Trail_Module + Trail_Component 묶음이 함께 이동, Trail 의 +0.5 오프셋 유지
7. reorder 후 "↻ 미리보기 필요" 뱃지 → [미리보기] 클릭 → 사라짐
8. [적용] 클릭 시 sibling 순서가 프리팹에 저장됨
9. 다른 부모 영역으로 마우스를 가져가면 가이드라인이 사라지고 (siblings.Count == 0 분기), MouseUp 시 reorder 무효 — `HandleDragCleanup` 이 fallback 로 정리
10. Undo `EndLayoutGroup` 경고 회귀 없음

## 범위 외 (이번 변경 미포함)

- 드래그 중 ScrollView 가장자리 자동 스크롤
- 핸들 hitbox 폭 확장 / 행 전체 드래그 시작
- cross-parent reorder 허용 (의도상 금지 유지)
- spec.json / CLAUDE.md 의 reorder 설명 동기화

## 관련 문서

- 직전 ▲/▼ 교체: `history/20260427_0432_replace_drag_with_arrow_buttons.md`
- 직전 hotControl 패치: `history/20260427_0408_fix_reorder_hotcontrol.md`
- 최초 reorder 도입: `history/20260427_0316_hierarchy_reorder_preview.md`
- 계획: `C:\Users\lhg07\.claude\plans\particle-auto-sorting-sprightly-cloud.md`
