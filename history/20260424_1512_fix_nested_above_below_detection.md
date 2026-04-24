# fix: Above/Below 재귀 탐색으로 변경 (중첩 구조 지원)

- 날짜: 2026-04-24 15:12
- 구분: fix (PrefabAnalyzer)
- 대상 파일: `D:\particleSortingTest\Assets\Editor\Analysis\PrefabAnalyzer.cs`

## 증상

프리팹 루트 직계 자식에만 `Above`/`Below`가 있는 경우는 인식되지만, 아래처럼 중간 계층(`effect`) 하위에 위치한 경우 "Above/Below 구조 누락" 경고가 뜨고 Root에서 전체를 수집하게 됨.

```json
{
  "name": "samplePrefab",
  "children": [
    {
      "name": "effect",
      "children": [
        { "name": "Above", "children": [ ... ] },
        { "name": "Below", "children": [ ... ] }
      ]
    }
  ]
}
```

## 원인

`PrefabAnalyzer.Analyze`가 `FindDirectChild(prefab.transform, "Above")` / `"Below"`로 루트의 **직계 자식만** 탐색. 중간 계층 하위에 있는 구조를 감지하지 못함.

## 수정

`FindDirectChild` → `FindDescendant`로 교체. DFS 재귀 탐색, 첫 매치 반환. 매치된 노드 내부는 더 이상 탐색하지 않음(중첩 Above 안의 또 다른 Above 같은 엣지 케이스 방지).

```csharp
private static Transform FindDescendant(Transform parent, string name)
{
    for (int i = 0; i < parent.childCount; i++)
    {
        var c = parent.GetChild(i);
        if (c.name == name) return c;
        var nested = FindDescendant(c, name);
        if (nested != null) return nested;
    }
    return null;
}
```

호출부도 `FindDescendant`로 변경.

## 영향 범위

- 기존(직계 자식에 Above/Below가 있는 프리팹): 동작 동일 (첫 매치가 직계 자식이므로).
- 새로 지원되는 케이스: `root/any/.../Above`, `root/any/.../Below` 임의 깊이.
- `HasAboveBelow`, 경고 pill("Above 루트 누락" / "Below 루트 누락"), `CollectFromRoot` 호출 로직은 그대로.

## 검증 체크리스트

- [ ] 샘플 프리팹(`samplePrefab/effect/Above`, `samplePrefab/effect/Below`) 드래그 → 테이블에 Above/Below 그룹이 각각 채워지는지
- [ ] Above/Below 중 한쪽만 중첩된 경우에도 해당 쪽만 정상 수집, 반대쪽은 누락 경고
- [ ] 기존 "직계 자식" 형태 프리팹 회귀 없음 (기존 동작 유지)
- [ ] 둘 다 없는 경우 기존처럼 "Above/Below 구조 누락" 경고 + Root 수집

## 비고

- 같은 이름(Above/Below)이 서로 다른 서브트리에 중복되어 존재할 경우 DFS 첫 매치만 사용. 현업 구조상 하나의 프리팹에 Above/Below가 각각 하나씩만 있다는 가정을 유지.
- Spec 규칙(null silent skip / clamp 금지 / Undo 그룹)에는 영향 없음.
