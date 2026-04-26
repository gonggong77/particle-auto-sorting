# 2026-04-27 04:32 — 드래그앤드롭 → ▲/▼ 버튼 reorder 로 교체

## 배경

직전 패치(`20260427_0408_fix_reorder_hotcontrol.md`)에서 `GUIUtility.hotControl` 점유로 ScrollView 흡수를 차단했지만 사용자 환경에서 **여전히 드래그가 막힌 것처럼 동작**. 원인이 IMGUI 외 다른 요인(드라이버/입력 장치/Unity 버전 특수성 등)일 가능성이 있어, 입력 캡처에 의존하는 방식 자체를 폐기하고 클릭 기반 ▲/▼ 버튼으로 교체.

## 변경 사항

### UI

- 수정 모드 ON 행 좌측의 `≡` 핸들 1개 → ▲ ▼ 미니버튼 2개
- 첫 sibling 위치에서 ▲ disabled, 마지막 sibling 위치에서 ▼ disabled
- 클릭 1회당 같은 부모 안에서 한 칸씩 swap
- 같은 transform 의 Trail 행은 +0.5 frac 이 보존된 상태로 함께 이동
- `ColHandle` 폭 18px → 38px (버튼 2개 + 약간의 여유). 헤더 스페이서 / Trail 행 스페이서 모두 동일 폭으로 자동 정렬됨

### 코드 정리

제거됨 (드래그 인프라 일체):
- `DragState` 클래스 (Window.cs)
- `_drag` 필드 (Window.cs)
- `HandleDragCleanup` 메서드 + OnGUI 호출 (Window.cs)
- `DrawDragHandle` 메서드 (Table.cs)
- `HandleSiblingDrag` 메서드 (Table.cs)
- `ApplyReorder` 메서드 (Table.cs) — 인덱스 보정 로직은 단방향 swap에서 불필요
- `DragGuideColor` 상수 (Table.cs)
- `DrawSection` 의 `orderedTransforms` / `rowRectsByTransform` Dictionary 수집 코드

신규:
- `DrawReorderButtons(PrefabData, RendererInfo)` — ▲/▼ 미니버튼, 경계 비활성화 처리, 클릭 시 MoveSibling + Repaint
- `GetSiblingPosition(PrefabData, Transform, out int idx, out int total)` — 같은 부모 sibling 인덱스/총 개수 산출
- `MoveSibling(PrefabData, Transform, int direction)` — 같은 부모 안에서 ±1 swap, 정수부 slot 재배분 + frac 보존, `HasSiblingReorder` / `NeedsPreviewRefresh` 플래그 셋

## 변경 파일

- `D:\particleSortingTest\Assets\Editor\ParticleAutoSortingWindow.cs`
- `D:\particleSortingTest\Assets\Editor\ParticleAutoSortingWindow.Table.cs`
- `D:\idleDante_test\Assets\Editor\ParticleAutoSorting\ParticleAutoSortingWindow.cs`
- `D:\idleDante_test\Assets\Editor\ParticleAutoSorting\ParticleAutoSortingWindow.Table.cs`

## 보존된 동작

- Trail 행의 +0.5f 오프셋 — `MoveSibling` 의 frac 보존 로직(`r.HierarchyOrder - Mathf.Floor(...)`)으로 유지
- `HasSiblingReorder` / `NeedsPreviewRefresh` 플래그 → 미리보기 뱃지, [미리보기] 버튼, [적용] 다이얼로그 메시지 모두 기존과 동일
- cross-parent reorder 차단 — siblings 필터(`t.parent == target.parent`) 유지
- Renderer 없는 자식의 절대 hierarchy slot 위치 — Apply 단계의 `ApplySiblingOrder` 가 책임지므로 변경 없음

## 사용자 검증 포인트

1. 같은 부모 자식 2개 이상인 프리팹(Above/Below)에서 행 ▲/▼ 클릭 시 즉시 자리 바뀜
2. 첫/마지막 sibling 의 경계 버튼이 회색(disabled) 으로 보임
3. ParticleSystem + Trail 묶음이 함께 이동하고 Trail 의 +0.5 오프셋 유지
4. reorder 후 "↻ 미리보기 필요" 뱃지 표시 → [미리보기] 클릭 후 사라짐
5. [적용] 클릭 시 변경된 sibling 순서가 프리팹에 저장됨
6. Undo `EndLayoutGroup` 경고 회귀 없음 (`20260427_0348` 패치와 무회귀)

## 관련 문서

- 직전 패치: `history/20260427_0408_fix_reorder_hotcontrol.md` (이 패치로 obsolete)
- 직전 reorder 도입: `history/20260427_0316_hierarchy_reorder_preview.md`
- Tasks: `tasks/20260427_hierarchy_reorder_preview.md` H-07 검증 시 본 변경 함께 확인
- spec.json / CLAUDE.md 의 reorder 설명에 "드래그앤드롭" 표현이 있다면 추후 동기화 필요 (별건)
