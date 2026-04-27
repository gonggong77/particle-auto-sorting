# [하이어라키만 재정렬] 정렬 방향 반전 (OiL DESC → ASC, Fudge ASC → DESC)

- 일시: 2026-04-27 19:15
- Plan: `C:\Users\lhg07\.claude\plans\snappy-bouncing-fog.md`
- 직전 관련 history: `20260427_1801_hierarchy_reorder_only.md` (해당 진입점 최초 추가)

## 증상 / 동기

`20260427_1801_hierarchy_reorder_only.md` 에서 추가된 [하이어라키만 재정렬] 진입점은 정렬 키를 `(sortingOrder DESC, sortingFudge ASC)` 로 잡아 **OiL 큰 자식이 트리 위쪽**, **Fudge 작은 자식이 트리 위쪽**에 배치되도록 동작했다. 당시 history 의 결정 이유 인용:

> "fudge 큰 자식을 아래로 보내면 '아래 = 뒤 = 먼저 그려짐' 사용자 요구를 만족"

이 이해 자체가 사용자 의도와 반대였다. 사용자가 실제로 원하는 일관 규칙은:

- **트리 하단 = 화면에서 앞에 그려짐**
- OiL 큰 쪽은 나중에 그려져 화면 앞 → 트리 하단
- Fudge 작은 쪽은 카메라에 가까워 화면 앞 → 트리 하단

따라서 `(OiL ASC, Fudge DESC)` 가 맞다.

## 변경 내용

### Unity 레포

| 파일 | 변경 |
|---|---|
| `D:\particleSortingTest\Assets\Editor\Apply\HierarchyReorderer.cs` | `ReorderChildrenAtParent` 의 `Sort` 비교자 두 줄 반전 + 주석 한 블록 갱신 (라인 139~149) |

> `D:\idleDante_test\Assets\Editor\ParticleAutoSorting\Apply\HierarchyReorderer.cs` 사본은 이번 변경에 포함되지 않음. 동일 동작 기대 시 별도 동기화 요청 필요.

### 계획 레포

| 파일 | 변경 |
|---|---|
| `history/20260427_1915_hierarchy_reorder_direction_flip.md` | 본 파일 |

## 정확한 코드 변경

Before:
```csharp
// 정렬 키: top → bottom 으로 (OiL DESC, Fudge ASC).
// OiL 큰 쪽 = 앞(나중에 그려짐), fudge 작은 쪽 = 앞.
psrChildren.Sort((a, b) =>
{
    int oilA = GetSortingOrder(a);
    int oilB = GetSortingOrder(b);
    if (oilA != oilB) return oilB.CompareTo(oilA); // DESC
    float fA = GetSortingFudge(a);
    float fB = GetSortingFudge(b);
    return fA.CompareTo(fB); // ASC
});
```

After:
```csharp
// 정렬 키: top → bottom 으로 (OiL ASC, Fudge DESC).
// 트리 하단 = 화면 앞. OiL 큰 쪽이 하단(나중에 그려짐), Fudge 작은 쪽이 하단(앞에 그려짐).
psrChildren.Sort((a, b) =>
{
    int oilA = GetSortingOrder(a);
    int oilB = GetSortingOrder(b);
    if (oilA != oilB) return oilA.CompareTo(oilB); // ASC
    float fA = GetSortingFudge(a);
    float fB = GetSortingFudge(b);
    return fB.CompareTo(fA); // DESC
});
```

알고리즘·슬롯 보존·OiL/Fudge 값 read-only 정책은 모두 그대로. `psrChildren[0]` 이 새 규칙에서 "트리 위쪽 후보" 의미만 바뀜.

## 검증 (사용자 작업)

1. Unity Editor 자동 컴파일. 콘솔 에러/워닝 없어야 함.
2. 같은 부모 밑에 OiL=10/20/30 자식 + PSR 없는 자식 섞인 프리팹에서 [하이어라키만 재정렬] → Hierarchy 의 PSR 자식이 **OiL 10 → 20 → 30 (위 → 아래)** 인지 확인.
3. PSR 없는 자식의 절대 sibling index 가 변경 전과 동일한지 확인 (슬롯 보존).
4. OiL 동률 + Fudge 0/5/10 케이스 → **Fudge 10 → 5 → 0 (위 → 아래)** 인지 확인.
5. `Above`/`Below` 컨테이너 있는/없는 양쪽 프리팹에서 동작 확인.
6. 회귀: [적용], 드래그 reorder, [재계산]/[미리보기], CSV export 가 영향받지 않음을 한 번씩 클릭해 확인.

## 비변경 영역 (침범 금지)

- `ApplyOne` / `ReorderSubtreeByExistingPSR` 의 DFS 순회·격리 로딩 구조
- `Above`/`Below` 자손 식별 (`FindDescendant`)
- `HasSortingRenderer` / `GetSortingOrder` / `GetSortingFudge` 헬퍼
- 비-PSR 자식의 절대 sibling index 보존
- `LoadPrefabContents` + `SaveAsPrefabAsset` + `UnloadPrefabContents` 격리 저장 패턴
- 확인 다이얼로그 텍스트 — 정렬 방향 자체를 사용자 문구로 노출하지 않으므로 불변
- 프리팹의 `sortingOrder` / `sortingFudge` 값 자체 (이 기능의 본질: read-only)
- 다른 기능: 드래그 reorder, [적용], GPU Instancing, CSV, 인터리브/오버플로 검출, [재계산]/[미리보기]

## 사후 (사용자 작업)

- Unity 레포 `D:\particleSortingTest\Assets\Editor\` 는 사용자가 직접 커밋.
- `D:\idleDante_test\Assets\Editor\ParticleAutoSorting\Apply\HierarchyReorderer.cs` 동기화 여부는 사용자 확인 후 별도 요청.
- 본 history 는 계획 레포에서 Claude 가 commit/push.
