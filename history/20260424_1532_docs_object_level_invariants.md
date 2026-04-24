# docs: 오브젝트 단위 정렬 불변식 명문화

- 날짜: 2026-04-24 15:32
- 구분: docs (CLAUDE.md)
- 대상 파일: `CLAUDE.md`

## 요청

사용자 정의:
1. Hierarchy 상에서 인접한 두 오브젝트의 경우, 하단 쪽에 더 가까운 오브젝트가 다른 오브젝트와 비교해 **OrderInLayer 가 같거나 높다**.
2. **OrderInLayer 가 같은** 두 오브젝트는 Hierarchy 상에서 더 아래쪽이 **sortingFudge 가 더 낮다**.

"충돌할만한 내용이 있으면 알려줘"

## 충돌 검토 결과

**결론: 직접 충돌 없음.** 현 구현(`SortingOptimizer.Optimize`, `AssignFudgeAndWriteAI`) 및 기존 CLAUDE.md "도메인 핵심 규칙" 과 모두 일관.

두 가지 해석 여지는 명시화가 필요:

### 1) OiL은 그룹 단위 할당

CLAUDE.md의 `Above = charSortingOrder + 1, +2, …` 표기만 읽으면 오브젝트마다 +1씩 증가하는 것처럼 보이나, 실제 코드는 **Material 그룹 단위**로 할당한다. 같은 Material을 공유하는 인접 오브젝트 2개는 동일 OrderInLayer. 사용자의 "같거나 높아"가 이 등호 케이스를 포함하므로 불일치 없음. 다만 CLAUDE.md에 명시 추가.

### 2) 인터리브 케이스에서 불변식 깨짐

Hierarchy 상에서 서로 다른 Material 그룹이 교차 배치되면(예: matA → matB → matA) 단조성이 국소적으로 깨진다. 현재 `SortingOptimizer.DetectInterleave`가 정확히 이 상황을 감지해 `HasInterleaveWarning` + 경고 pill을 켠다. 따라서 불변식은 **"인터리브가 없을 때"**라는 전제 아래 보장됨을 CLAUDE.md에 표기.

## 코드 검증

- `PrefabAnalyzer`: DFS 순회로 `HierarchyOrder` 0, 1, 2, … 부여. 위→아래 단조 증가. ✓
- `SortingOptimizer.Optimize`: 그룹을 `RepresentativeOrder`(그룹 내 최상단 멤버의 HierarchyOrder) 오름차순 정렬 후, Above는 `+i+1`, Below는 `-(n-i)` 할당. 하단에 가까운 그룹일수록 큰 OrderInLayer. ✓
- `AssignFudgeAndWriteAI`: 멤버를 HierarchyOrder 오름차순 정렬, `fudge[rank] = (size-1-rank)*step`. rank 0(최상단) → 최대, rank size-1(최하단) → 0. ✓

## CLAUDE.md 변경

"도메인 핵심 규칙" 섹션 인터리브 감지 bullet 뒤에 아래를 추가:

```md
- **오브젝트 단위 불변식** (인터리브 없을 때 보장):
  1. OrderInLayer 단조성: Hierarchy 상 인접한 A(위), B(아래)에 대해 B.OrderInLayer >= A.OrderInLayer. 같은 Material 그룹이면 등호, 그 외엔 엄격 증가. (OiL 은 그룹 단위 할당, +1/+2 증분은 그룹 수 기준)
  2. Fudge 단조성: OrderInLayer 가 같은(= 동일 Material 그룹) A(위), B(아래)에 대해 B.sortingFudge < A.sortingFudge. 최하단 = 0, 최상단 = (size-1)*30.
- 위 불변식은 인터리브 시 깨짐 → DetectInterleave 가 HasInterleaveWarning + 경고 pill
```

## 검증 체크리스트

- [ ] 샘플 프리팹 적용 후 Inspector 에서 Above 그룹 하단이 상단보다 OrderInLayer 가 같거나 큼
- [ ] 동일 Material 을 공유하는 인접 렌더러 2개의 OrderInLayer 가 동일함
- [ ] 같은 OrderInLayer 인 렌더러들 중 Hierarchy 아래쪽이 sortingFudge 가 더 작음 (최하단 = 0)
- [ ] 인터리브가 있는 프리팹에서 경고 pill 점등 + 위 불변식 위반 가능
