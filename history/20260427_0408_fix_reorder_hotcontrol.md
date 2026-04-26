# 2026-04-27 04:08 — Reorder 드래그 간헐적 막힘 수정 (GUIUtility.hotControl 점유)

## 증상

수정 모드 ON 상태에서 행 좌측 `≡` 핸들로 같은 부모의 자식 순서를 바꾸는 드래그가 **간헐적으로 막힌 것처럼** 동작:

- 드래그가 슬쩍 시작되다 멈춤
- 가이드라인이 안 보임
- 드롭이 의도와 다른 위치에 떨어짐
- 마우스를 빠르게 움직이거나 핸들 영역(폭 18px)을 살짝 벗어나면 빈도 증가

## 원인

`Table.cs:DrawDragHandle` MouseDown 처리에 **`GUIUtility.hotControl` 점유가 빠져있었음**.

드래그 대상 행 전체가 `EditorGUILayout.ScrollViewScope` (`List.cs` L20) 내부에 있는데, hotControl을 점유하지 않으면 ScrollView 내부 핸들러가 MouseDrag 이벤트를 흡수하여 `HandleSiblingDrag`가 호출되지 않거나 늦게 호출됨. IMGUI 드래그 표준 패턴에서 누락된 단계.

## 수정 내용

### `DragState`에 `controlId` 필드 추가

```csharp
class DragState
{
    public PrefabData data;
    public Transform sourceParent;
    public Transform draggingTransform;
    public int hoverInsertIndex = -1;
    public Rect hoverInsertRect;
    public int controlId;  // 신규
}
```

### `DrawDragHandle` MouseDown — 발급 후 hotControl 점유

```csharp
int id = GUIUtility.GetControlID(FocusType.Passive);
_drag = new DragState
{
    data = data,
    sourceParent = t.parent,
    draggingTransform = t,
    hoverInsertIndex = -1,
    controlId = id,
};
GUIUtility.hotControl = id;
evt.Use();
```

### `HandleSiblingDrag` MouseUp — ID 일치 시에만 해제

```csharp
if (GUIUtility.hotControl == _drag.controlId) GUIUtility.hotControl = 0;
_drag = null;
Repaint();
evt.Use();
```

### `HandleDragCleanup` — fallback 경로에도 동일 해제 로직 추가

다른 컨트롤이 점유한 hotControl을 실수로 풀지 않기 위해 ID 일치 검사 후 0으로 설정.

## 변경 파일

- `D:\particleSortingTest\Assets\Editor\ParticleAutoSortingWindow.cs`
- `D:\particleSortingTest\Assets\Editor\ParticleAutoSortingWindow.Table.cs`
- `D:\idleDante_test\Assets\Editor\ParticleAutoSorting\ParticleAutoSortingWindow.cs`
- `D:\idleDante_test\Assets\Editor\ParticleAutoSorting\ParticleAutoSortingWindow.Table.cs`

두 배포 경로의 동일 파일을 함께 수정. ApplyReorder/소수부 보존/HierarchyOrder mutate 로직은 손대지 않음.

## 범위 외 (별건)

- 드래그 도중 ScrollView 가장자리 자동 스크롤
- 핸들 hitbox 폭 확장 / 행 전체 드래그 시작
- cross-parent reorder (의도상 금지)

## 사용자 검증 포인트

1. 같은 부모 자식 2개 이상인 프리팹의 Above/Below 섹션에서 핸들을 잡고 위/아래로 빠르게 반복 드래그 — 매번 가이드라인이 따라오고 드롭 즉시 반영되는지
2. 스크롤이 활성화된 긴 목록에서 드래그 유지되는지
3. 드래그 중 OilAfter/FudgeAfter IntField/FloatField 위로 마우스 지나가도 안 끊기는지
4. 드래그 중 ScrollView 밖에서 MouseUp → 정상 정리 후 다른 행 클릭 가능한지
5. Undo `EndLayoutGroup` 경고 회귀 없는지 (`20260427_0348_fix_undo_endlayoutgroup.md` 와의 회귀 회피)
6. Trail 행 reorder 후 [미리보기] 클릭 시 +0.5f 오프셋 유지되는지

## 관련 문서

- 계획: `C:\Users\lhg07\.claude\plans\particle-auto-sorting-on-proud-dewdrop.md`
- 직전 reorder 구현 history: `history/20260427_0316_hierarchy_reorder_preview.md`
- 직전 Undo 패치 history: `history/20260427_0348_fix_undo_endlayoutgroup.md`
- Tasks: `tasks/20260427_hierarchy_reorder_preview.md` (H-07 수동 검증과 함께 확인)
