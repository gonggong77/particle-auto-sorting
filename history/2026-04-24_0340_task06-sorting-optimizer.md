# Task #6 Complete — SortingOptimizer — 2026-04-24 03:40

## 전체 진행률
- 전체 17개 중 **7개 완료** (#16, #1, #2, #3, #4, #5, #6)
- Phase 3(최적화 레이어) 100% 완료 → Phase 4(적용 레이어) 진입 가능

## 이 세션에서 완료한 작업

### ✅ #6 SortingOptimizer.cs
- 경로: `D:\particleSortingTest\Assets\Editor\Optimization\SortingOptimizer.cs`
- 네임스페이스: `ParticleAutoSorting.Editor.Optimization`
- 공개 API:
  - `Optimize(PrefabData, int charSortingOrder) → List<MaterialGroupInfo>`
  - `OptimizeAll(IEnumerable<PrefabData>, int charSortingOrder)`

### 구현 요점
- **그룹 버킷**: GroupTag(`Above`/`Below`/`Root`)별로 렌더러를 분리 후 각 버킷에서 Material로 묶어 `MaterialGroupInfo` 생성. 대표 순서 = `min(HierarchyOrder)`.
- **OiL 할당** (Material 그룹 단위):
  - Above/Root: 대표 순서 오름차순 rank `i` → `char + (i + 1)` (위쪽 그룹이 낮은 OiL = 뒤에 그려짐)
  - Below: 대표 순서 오름차순 rank `i` → `char - (n - i)` (위쪽 그룹이 가장 낮은 OiL = 맨 뒤)
  - Root는 Above와 동일한 정책을 적용 (Above/Below 구조 누락 경고는 #4가 이미 세팅)
- **오버플로**: 할당 OiL이 `short` 범위(-32768~32767) 벗어나면 `PrefabData.HasOverflow = true` (clamp 금지, 적용 차단은 #7에서 처리)
- **Fudge 공식**: `step = min(0.1, 0.9 / (size - 1))`, `fudge[rank] = (size - 1 - rank) * step`
  - 그룹 내 렌더러를 `HierarchyOrder` 오름차순으로 정렬 → rank 0(가장 위) = 최대 Fudge = 뒤
  - `size == 1`이면 step=0, fudge=0
- **OilAfterAI / FudgeAfterAI**에 결과 기록 (OilOverride/FudgeOverride는 사용자 수정 영역이라 건드리지 않음)
- **GroupColor**: `Material.GetInstanceID()` 해시 기반 hue로 HSV→RGB 변환해 UI pill용 색상 선할당
- **인터리브 감지**:
  - after 값 기준 SortKey(`SortingLayerID, OiL(override || AI), Fudge(override || AI), HierarchyOrder`)로 전체 렌더러 정렬
  - Material별 position(0-based index) 목록 수집
  - 두 Material A, B 비교: `max(A) > min(B) && max(B) > min(A)` → 서로의 이름을 `InterleavedWith`에 추가, `HasInterleaveWarning = true`
  - 매 호출 시점에 리셋 후 재계산 (원래 값 덮어쓰기)

### 스펙 해석 / 결정
- "group_size"는 매티리얼 그룹 내 렌더러 수로 해석 (같은 OiL이 부여되므로 그룹 내 정렬은 Fudge만으로 결정)
- Root 버킷은 Above/Below 어디에도 속하지 않으므로 Above와 같은 양의 방향으로 부여하되, #4의 "Above/Below 구조 누락" 경고로 사용자가 인지하게 유지
- OiL override로 오버플로가 발생하는 케이스는 이번 단계에서 체크하지 않음 (AI 할당 결과만 검증)

## 다음 작업 — Phase 4
- **#7 PrefabApplier.cs** — BlockedBy: #1, #6 (해제됨)
  - `HasOverflow` 체크박스 프리팹이 하나라도 있으면 에러 다이얼로그 후 전체 적용 중단
  - `Undo.IncrementCurrentGroup` + `SetCurrentGroupName("Particle Auto Sorting Apply")`
  - Renderer마다 `Undo.RegisterCompleteObjectUndo` 후 OiL/Fudge 쓰기
  - `PrefabUtility.SavePrefabAsset` 호출
  - 인터리브/Instancing 경고 카운트를 확인 다이얼로그에 표시
