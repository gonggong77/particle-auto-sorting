# Fix — `GetLast immediately after beginning a group`

**시각**: 2026-04-24
**브랜치**: main
**레포**: gonggong77/particle-auto-sorting

## 증상

Unity 에디터에서 Particle Auto Sorting 창을 열고 프리팹을 등록할 때 다음 에러 발생:

```
You cannot call GetLast immediately after beginning a group.
UnityEngine.GUILayoutUtility:GetLastRect ()
ParticleAutoSorting.Editor.ParticleAutoSortingWindow:DrawEntry (...)
  at Assets/Editor/ParticleAutoSortingWindow.List.cs:55
ParticleAutoSorting.Editor.ParticleAutoSortingWindow:DrawPrefabList ()
  at Assets/Editor/ParticleAutoSortingWindow.List.cs:31
```

## 원인

`ParticleAutoSortingWindow.List.cs:55`에서 `EditorGUILayout.VerticalScope`(=`BeginVertical`)를 시작한 직후 `GUILayoutUtility.GetLastRect()`를 호출.

IMGUI는 그룹을 막 시작한 시점에는 "직전 rect"가 존재하지 않으므로 호출 자체가 잘못된 순서. 또한 entry 박스 전체를 감싸는 테두리는 그룹의 **내용물 레이아웃이 끝난 뒤에** 그려야 정확한 rect를 알 수 있음.

## 수정 내용

`Assets/Editor/ParticleAutoSortingWindow.List.cs` `DrawEntry`:

- `using (new VerticalScope(...))` → `var boxRect = EditorGUILayout.BeginVertical(...) ... EndVertical()` 패턴으로 변경
- 테두리 그리기를 `EndVertical` **이후로 이동**
- `Event.current.type == EventType.Repaint` 가드 추가 (Layout 이벤트에서는 rect가 무효이므로 그리지 않음)

## 검증

- 동일 파일에서 다른 `GetLastRect` 호출 없음 (전체 grep 결과 0건)
- 다른 partial 파일(`Bottom.cs`, `Table.cs`, 메인)에도 같은 패턴 없음

사용자 측 Unity 에디터에서 창을 다시 열어 프리팹 드래그 시 에러가 사라지는지 확인 필요.
