# feat: Material 런 기반 OIL 할당으로 오브젝트 단위 불변식 강제

- 날짜: 2026-04-24 15:48
- 구분: feat (SortingOptimizer) + docs (CLAUDE.md)
- 대상 파일:
  - `D:\particleSortingTest\Assets\Editor\Optimization\SortingOptimizer.cs`
  - `CLAUDE.md`

## 요청

직전 커밋(`5de69e2`)에서 명문화한 두 불변식을 **프로그램이 실제로 강제**하도록 수정.

1. OrderInLayer 단조성: 인접 오브젝트 A(위), B(아래) → `B.OrderInLayer >= A.OrderInLayer`
2. Fudge 단조성: 같은 OrderInLayer 인 A(위), B(아래) → `B.sortingFudge < A.sortingFudge`

## 기존 알고리즘의 한계

기존 `BuildMaterialGroups`는 **동일 Material 을 모두 하나의 그룹**으로 묶어 OIL 하나를 부여 → 배치 수 최소화에는 유리하나, Hierarchy 상에서 같은 Material 이 다른 Material 에 의해 끊긴 경우(인터리브) 불변식 1 이 국소적으로 깨짐.

예) Hierarchy: matA, matB, matA → matA 그룹 OIL=+1, matB OIL=+2
- pos 0 (matA, +1) → pos 1 (matB, +2): 증가 ✓
- pos 1 (matB, +2) → pos 2 (matA, +1): **감소 ✗**

`DetectInterleave` 가 경고를 띄우지만 값 자체는 그대로 두어 시각적 오류 잠재.

## 새 알고리즘: Material 런 기반 할당

Hierarchy 순서(위→아래)로 정렬 후, **연속된 동일 Material 구간**(run)을 하나의 그룹으로 묶는다. 같은 Material 이라도 끊기면 별도 런 = 별도 OIL.

```csharp
private static List<MaterialGroupInfo> BuildMaterialRuns(List<RendererInfo> list)
{
    var sorted = new List<RendererInfo>(list);
    sorted.Sort((a, b) => a.HierarchyOrder.CompareTo(b.HierarchyOrder));

    var runs = new List<MaterialGroupInfo>();
    MaterialGroupInfo current = null;
    foreach (var r in sorted)
    {
        if (r.SharedMaterial == null) continue;
        if (current == null || !ReferenceEquals(current.SharedMaterial, r.SharedMaterial))
        {
            current = new MaterialGroupInfo { SharedMaterial = r.SharedMaterial, RepresentativeOrder = r.HierarchyOrder };
            runs.Add(current);
        }
        current.Members.Add(r);
    }
    return runs;
}
```

`Optimize`는 `BuildMaterialGroups` → `BuildMaterialRuns` 로 교체. 런 개수 N 을 bucket 별로 계산, Above = `+1…+N`, Below = `-N…-1`.

### 불변식 강제 증명

**Rule 1** (OIL 단조성):
- 한 bucket(Above/Below) 내 런들은 Hierarchy 순서로 정렬되어 있음
- 위쪽 런 = 작은 i = 작은 OIL (Above의 경우 `i+1`), 아래쪽 런 = 큰 i = 큰 OIL
- 같은 런 내 인접 오브젝트 → 동일 OIL (등호)
- 런 경계를 건너는 인접 오브젝트 → 아래쪽 런이 엄격히 큰 OIL
- 따라서 인접 A(위), B(아래) → `B.OIL >= A.OIL` 항상 ✓

**Rule 2** (Fudge 단조성):
- 같은 OIL = 같은 런 (보장)
- 런 내 멤버는 HierarchyOrder 오름차순 정렬 후 `fudge[rank] = (size - 1 - rank) * 30`
- rank 0 (최상단) → `(size-1)*30` 최대, rank size-1 (최하단) → 0 최소
- 인접 A(위 rank k), B(아래 rank k+1) → `A.fudge - B.fudge = 30 > 0` ✓

## 트레이드오프

- **배치 수**: 기존 대비 **증가 가능**. 같은 Material 이 Hierarchy 상 인접하지 않으면 각 런마다 OIL 이 달라져 별도 배치가 됨.
- **시각적 정합성**: 항상 보장.
- **권장 Hierarchy 구성**: 같은 Material 을 쓰는 렌더러는 연속 배치. 그렇게 하면 기존과 동일한 최소 배치 수 유지.

`DetectInterleave` 의 의미 변화:
- 이전: "시각 순서 꼬임" 경고
- 이후: "배치 수 최적 아님 — Hierarchy 재배치 권장" 신호 (불변식 자체는 보장됨)

## 삭제된 함수

`BuildMaterialGroups` 제거. `BuildMaterialRuns` 로 대체.

## CLAUDE.md 변경

"도메인 핵심 규칙" 섹션 재구성:
- "Material 런(run)" 개념 bullet 신설
- "OiL 할당" 문구를 런 기반으로 재작성
- "Fudge 할당" 문구를 "동일 런 내" 로 수정
- "오브젝트 단위 불변식" 의 전제 조건 "(인터리브 없을 때)" → "(런 기반 할당으로 항상 보장)"
- "인터리브 감지" bullet 을 맨 아래로 옮기고 의미 재정의 (배치 효율 신호)

## 검증 체크리스트

- [ ] 연속 Material 케이스 (matA, matA, matB, matB): 기존과 동일한 OIL 할당, 배치 수 동일
- [ ] 인터리브 케이스 (matA, matB, matA): 3개 런 → OIL 3개 → pos 2 (matA) 가 이전 +1 에서 +3 으로 바뀜
- [ ] 경고 pill: 인터리브 케이스에서 여전히 표시 (의미는 바뀌었지만 검출은 정상)
- [ ] Fudge: 런 size=4 일 때 [90, 60, 30, 0], size=1 일 때 0
- [ ] Apply 후 Unity Inspector OIL/Fudge 정확히 반영
- [ ] Undo 한 번으로 롤백
- [ ] 오버플로 (short 범위 초과) 시 HasOverflow + 적용 차단 유지
