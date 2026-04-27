# Fix — Undo 버튼 클릭 시 `EndLayoutGroup: BeginLayoutGroup must be called first`

**시각**: 2026-04-27 03:48
**브랜치**: main
**레포**: gonggong77/particle-auto-sorting

## 증상

Particle Auto Sorting 윈도우 하단 **Undo** 버튼 클릭 시 콘솔에 다음 경고 출력:

```
EndLayoutGroup: BeginLayoutGroup must be called first.
UnityEngine.GUIUtility:ProcessEvent (int,intptr,bool&)
```

## 원인

`Assets/Editor/ParticleAutoSortingWindow.Bottom.cs:48-53`의 Undo 버튼이 `EditorGUILayout.HorizontalScope` 레이아웃 그룹 안에서 `Undo.PerformUndo()`를 **즉시 동기 호출**.

PerformUndo는 호출 즉시 프리팹 상태(이름, hierarchy, 머티리얼 등)를 되돌리며, 같은 OnGUI 프레임의 Layout 패스와 Repaint 패스 사이에 레이아웃 구조(예: `DrawPrefabList`가 그리는 entry 수, override label 표시 여부 등)가 달라질 수 있음.

IMGUI는 두 패스의 `Begin*/End*` 호출 수가 일치할 것을 가정하므로, 패스 사이에 트리 구조가 변하면 EndLayoutGroup 미스매치가 발생.

## 수정 내용

대상 파일 (두 프로젝트 동일 패치):

- `D:\particleSortingTest\Assets\Editor\ParticleAutoSortingWindow.Bottom.cs`
- `D:\idleDante_test\Assets\Editor\ParticleAutoSorting\ParticleAutoSortingWindow.Bottom.cs`

- `Undo.PerformUndo()` 호출을 `EditorApplication.delayCall`로 래핑하여 현재 GUI 프레임이 끝난 다음 에디터 업데이트에서 실행되도록 변경.
- `statusMessage` 갱신과 `Repaint()`도 같은 람다 안에서 실행 → 상태 변경과 그리기가 모두 일관된 다음 프레임에서 발생.

```csharp
if (GUILayout.Button("Undo", GUILayout.Width(70)))
{
    EditorApplication.delayCall += () =>
    {
        Undo.PerformUndo();
        statusMessage = "Undo 실행";
        Repaint();
    };
}
```

## 검증

- 사용자 측 Unity 에디터에서 창을 다시 열고 적용 → Undo 시 경고가 사라지는지 확인 필요.
- `RecomputeAll`, `RunApply`, `RunCsvExport` 등 다른 버튼 핸들러는 레이아웃 트리를 즉시 깨뜨리지 않는 자체 상태만 갱신하므로 동일 패턴 적용 불필요(현재는 문제 없음).
