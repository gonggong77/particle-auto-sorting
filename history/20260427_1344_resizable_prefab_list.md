# 20260427_1344 — 등록된 프리팹 영역 상하 리사이즈 가능

## 요청
사용자: "particle auto sorting UI 에서 '등록된 프리팹' 칸 사이즈를 상하로 조절 가능하면 좋겠어 지금은 좁아서 불편해"

## 증상 / 배경
- `ParticleAutoSortingWindow.List.cs` 의 `DrawPrefabList()` 가 ScrollView 를 `GUILayout.MinHeight(220)` 으로 잡고 있어 사실상 220px 근처에 고정.
- 등록된 프리팹이 많아지면 좁은 영역에서 스크롤만 길어져 한눈에 비교/수정이 어려움.
- 윈도우 크기를 늘려도 프리팹 리스트는 늘어나지 않고 하단 빈공간이 남는 구조.

## 원인
`DrawPrefabList()` 의 ScrollView 높이가 상수(`MinHeight(220)`) 이고, 사용자가 직접 조절할 수 있는 UI 가 없음. `OnGUI` 흐름이 위→아래 단방향 stack 이라 단순 `FlexibleSpace` 만으로는 영역 비율 제어가 안 됨.

## 수정
1. **`ParticleAutoSortingWindow.cs`**
   - `[SerializeField] float prefabListHeight = 220f;` 추가 (윈도우 인스턴스 단위로 직렬화 → 도메인 리로드/재오픈 시 유지).
   - 리사이즈 관련 상수: `PrefabListMinHeight = 80f`, `PrefabListBottomReserve = 200f` (하단 버튼/CSV/스테이터스 영역 보전), `PrefabListSplitterHeight = 6f`, hover/idle 색상.
   - 드래그 상태 필드: `_resizingPrefabList`, `_resizeStartMouseY`, `_resizeStartHeight`.
   - `OnGUI` 에 `DrawPrefabList()` 직후 `DrawPrefabListSplitter()` 호출 삽입.
   - `DrawPrefabListSplitter()` 신규 메서드:
     - `GUILayoutUtility.GetRect` 로 6px 가로 띠 확보.
     - `EditorGUIUtility.AddCursorRect` 로 `MouseCursor.ResizeVertical` 적용.
     - `EventType.Repaint` 에서만 라인 그리기 (hover/drag 시 강조색).
     - `MouseDown(0)` 으로 드래그 시작 — 시작 시점의 mouseY/높이 캡처.
     - `MouseDrag` 에서 `Mathf.Clamp(시작높이 + delta, MinHeight, position.height - BottomReserve)` 로 갱신 + `Repaint()`.
     - `MouseUp` 에서 드래그 해제.

2. **`ParticleAutoSortingWindow.List.cs`**
   - `DrawPrefabList()` 의 `GUILayout.MinHeight(220)` → `GUILayout.Height(prefabListHeight)` 로 교체. `Height` 를 쓰는 이유: 사용자가 명시적으로 정한 높이를 그대로 따라가야 함 (자동 확장 X).

## 미러 동기화
- `D:/idleDante_test/Assets/Editor/ParticleAutoSorting/ParticleAutoSortingWindow.cs`, `...List.cs` 동일 패치 적용.
- `diff` 로 두 사본이 일치함 확인.

## 검증 항목 (사용자 확인 필요)
- [ ] Unity 에디터 재컴파일 후 `Tools > particle auto sorting` 창 열기.
- [ ] '등록된 프리팹' 영역 바로 아래의 얇은 띠에 마우스 올렸을 때 vertical resize 커서 표시.
- [ ] 위/아래로 드래그 시 리스트 영역이 부드럽게 늘어나고 줄어듦.
- [ ] 최소 80px 이하로 줄어들지 않고, 윈도우 높이 - 200px 이상으로 커지지 않음.
- [ ] 윈도우 닫고 다시 열어도 마지막 높이 유지 (SerializeField 효과).
- [ ] 수정 모드 ON 상태에서 드래그 핸들(≡) 동작에 영향 없음 — 스플리터는 리스트 외부에 위치.

## IMGUI 함정 체크
- 본 구현은 `BeginVertical/Horizontal` 직후 `GetLastRect` 사용 X.
- `EditorGUI.DrawRect` 호출은 `EventType.Repaint` 가드 안에서만 실행.
- `GUIUtility.hotControl` 은 사용하지 않고 자체 boolean (`_resizingPrefabList`) 로 드래그 상태 관리 — 기존 `HandleDragCleanup` (HierarchyReorder 용) 과 충돌 없음.
