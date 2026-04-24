# fix: 프리팹 애셋에서 activeInHierarchy 오탐 수정

- 날짜: 2026-04-24 15:17
- 구분: fix (PrefabAnalyzer)
- 대상 파일: `D:\particleSortingTest\Assets\Editor\Analysis\PrefabAnalyzer.cs`

## 증상

프리팹 내부 모든 GameObject가 켜져 있음(Inspector 체크박스 ON)에도 불구하고 상태바/경고 pill에 **"비활성 오브젝트 포함"** 경고가 뜸. 렌더러도 inactive 처리되어 테이블에서 제외됨.

## 원인

`GameObject.activeInHierarchy`는 **씬에 배치된 GameObject**를 전제로 동작. 드래그앤드롭으로 들어온 프리팹 애셋(Asset, not scene instance)은 씬 컨텍스트가 없어 `activeInHierarchy`가 `activeSelf`와 무관하게 `false`를 반환함. Unity의 알려진 동작.

기존 코드:

```csharp
if (!go.activeInHierarchy || !psr.enabled) { inactiveHere = true; }
```

→ 프리팹 애셋 대상이면 `go.activeInHierarchy == false`여서 항상 비활성 판정.

## 수정

부모 체인을 따라 `activeSelf`를 직접 누적 판정하는 헬퍼 추가.

```csharp
private static bool IsActiveInPrefab(GameObject go)
{
    var t = go.transform;
    while (t != null)
    {
        if (!t.gameObject.activeSelf) return false;
        t = t.parent;
    }
    return true;
}
```

두 렌더러 분기(`ParticleSystemRenderer`, `TrailRenderer`)에서 `go.activeInHierarchy` → `IsActiveInPrefab(go)` 교체.

## 영향 범위

- 프리팹 애셋을 드래그로 넣은 경우: 체크박스 상태가 그대로 반영됨 (정상 동작).
- 씬 인스턴스/애셋 양쪽 모두 커버: `activeSelf` 체인은 씬에서도 `activeInHierarchy`와 동일한 결과를 냄.
- 렌더러 컴포넌트의 `enabled == false`는 기존처럼 비활성 판정 유지.
- null Material silent skip 규칙은 그대로.

## 검증 체크리스트

- [ ] 샘플 프리팹(모든 GO 켜짐) 드래그 → "비활성 오브젝트 포함" 경고 사라지고 전체 렌더러 수집
- [ ] 특정 자식 GO 체크박스 OFF → 해당 항목만 비활성 처리 + 경고 pill 유지
- [ ] 부모 GO만 OFF → 그 하위 전체 비활성 처리
- [ ] 컴포넌트 enabled=false (GO는 ON) → 해당 항목만 비활성 처리
