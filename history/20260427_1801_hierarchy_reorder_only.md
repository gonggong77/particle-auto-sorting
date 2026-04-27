# 기존 PSR 값 유지 채로 하이어라키만 재정렬하는 진입점 추가

- 일시: 2026-04-27 18:01
- Plan: `C:\Users\lhg07\.claude\plans\idledante-test-test3-smooth-stearns.md`
- 직전 관련 history: `20260427_0316_hierarchy_reorder_preview.md` (수정 모드 + 드래그/Apply 흐름)

## 증상 / 동기

`Particle Auto Sorting` 의 [적용] 흐름은 항상 `SortingOptimizer.Optimize` 로 OiL/Fudge 값 자체를 새로 계산한 뒤 sibling 순서까지 재배치한다. 하지만 다음과 같은 운영 케이스가 있다:

- 이전 세션의 `randomize_sorting.py` 로 의도적으로 어지럽힌 `test 2` 폴더 8개 prefab(skill101~107 + skill105_2). 사용자는 **랜덤화된 OiL/Fudge 값을 그대로 유지**한 채로, 그 값에 일치하도록 **하이어라키 sibling 순서만** 정리해 자동정렬 도구의 *기대 출력(reference output)* 을 사람 손으로 빠르게 확보하고자 함.
- "test3 폴더" 라는 표현이 있었으나 사실은 "test 2" 의 오타 — 사용자가 정정함.

기존 진입점은 OiL/Fudge 재계산을 항상 동반하므로 위 케이스를 직접 만족시키지 못했다.

## 변경 내용

### Unity 레포 (사본 2곳 동시 적용)

`D:\particleSortingTest\Assets\Editor\` 와 `D:\idleDante_test\Assets\Editor\ParticleAutoSorting\` 두 사본 모두에 동일 적용.

| 파일 | 변경 |
|---|---|
| `Apply/HierarchyReorderer.cs` | **신규 파일** — `HierarchyReorderer.Apply(IEnumerable<PrefabData>)` 정적 메서드. 선택된 prefab 들을 `PrefabUtility.LoadPrefabContents` 로 열어 `Above`/`Below` 서브트리(없으면 root) 안의 모든 부모 노드를 깊이 우선 방문, 각 부모의 PSR/TrailRenderer 자식만 골라 `(sortingOrder DESC, sortingFudge ASC)` 키로 재정렬한 뒤 PSR-자식이 차지하던 슬롯 인덱스에 새 순서대로 `SetSiblingIndex` 적용. `SaveAsPrefabAsset` 후 `UnloadPrefabContents`. 비-PSR 자식의 절대 위치 보존은 `PrefabApplier.ApplySiblingOrder` 의 슬롯 보존 정책과 동일. `AssetDatabase.StartAssetEditing` 묶음 + 단일 다이얼로그 확인 |
| `ParticleAutoSortingWindow.Bottom.cs` | `DrawBottomButtons` 의 [Undo] 와 [적용] 사이에 **"하이어라키만 재정렬"** 버튼 추가 (`width: 140`). `RunHierarchyReorder()` 메서드 신규 — `HierarchyReorderer.Apply(prefabs)` 호출 후 상태바에 `하이어라키 재정렬 완료: N개` 또는 `하이어라키 재정렬 취소됨` 표기 |

### 계획 레포 (`D:\particle-auto-sorting\particle-auto-sorting`)

| 파일 | 변경 |
|---|---|
| `history/20260427_1801_hierarchy_reorder_only.md` | 본 파일 |

## 핵심 설계 결정

- **OiL/Fudge 값은 절대 건드리지 않음**: `SortingOptimizer.Optimize` 호출 경로를 완전히 우회한다. PSR/TrailRenderer 의 `sortingOrder`/`sortingFudge` 는 read-only 로만 사용.
- **정렬 키 (top → bottom)**: `sortingOrder DESC` (큰 OiL 위, 작은 OiL 아래) → `sortingFudge ASC` (작은 fudge 위, 큰 fudge 아래). Unity 매뉴얼 기준 positive `sortingFudge` 가 카메라에서 더 멀게 처리되므로, 같은 OiL 안에서 fudge 큰 자식을 아래로 보내면 "아래 = 뒤 = 먼저 그려짐" 사용자 요구를 만족.
- **Above/Below 격리**: `Above` 서브트리와 `Below` 서브트리를 따로 호출하여 두 그룹 사이로 자식이 절대 옮겨가지 않음. `PrefabAnalyzer` 와 동일한 `FindDescendant` (이름 정확 매칭, 깊이 우선) 사용.
- **재귀 깊이**: 각 부모 노드 단위로 자식만 정렬 후 모든 자식에 대해 재귀. PSR 이 깊게 중첩된 케이스(예: skill101 의 `Slash` → 9개 손자) 도 모든 레벨에서 정렬됨.
- **슬롯 보존 정책**: PSR 이 없는 형제(예: 빈 컨테이너 GameObject) 의 sibling index 는 건드리지 않는다. PSR 자식의 원래 인덱스 리스트(`[1, 3, 4, …]`) 에 새 순서대로 `SetSiblingIndex` 호출 — `PrefabApplier.ApplySiblingOrder` (lines 211-262) 의 검증된 패턴.
- **Undo 흐름**: `Apply` 흐름과 분리 — 본 진입점은 `PrefabUtility.LoadPrefabContents` 기반이라 자체 Undo stack 을 사용하지 않고, 디스크 저장 후 변경 사항은 git diff/Unity asset history 로 추적.

## 검증 (사용자 작업)

1. Unity Editor 에서 자동 컴파일. 콘솔에 에러/워닝 없어야 함.
2. `Tools > particle auto sorting` 윈도우 → `D:/idleDante_test/Assets/02_Prefabs/Skills/test 2/` 의 8개 prefab 드래그 등록 → "전체 선택" → **"하이어라키만 재정렬"** 클릭. 상태바에 `하이어라키 재정렬 완료: 8개` 표시.
3. `skill101.prefab` 등을 Prefab Mode 로 진입해 `Above`/`Below` 자식 트리가 위→아래로 `(OiL DESC, Fudge ASC)` 단조 감소함을 1~2개 sample 에서 시각 확인. Above ↔ Below 사이 자식 이동 없음 확인.
4. (idleDante_test 가 git repo 가 아니므로 git diff 검증은 mtime 기반 또는 Unity Asset Snapshot 으로 대체) prefab 의 `m_SortingOrder` / `m_SortingFudge` 값 자체는 변경되지 않았음을 임의 prefab 1~2개 열어 비교.

## 비변경 영역 (침범 금지)

- 기존 [적용] 버튼 흐름 (`PrefabApplier.Apply` → `SortingOptimizer.Optimize` → `ApplySingle` → `WriteRenderer`) 일체.
- `SortingOptimizer`, `BatchCounter`, `CsvReportExporter`, `PrefabAnalyzer` — 본 건은 **읽기 전용** 으로만 PSR 값을 사용.
- `PrefabApplier.ApplySiblingOrder` — 호출하지 않음. 신규 `HierarchyReorderer` 가 동일 정책을 자체 구현.
- 수정 모드 / 드래그 핸들 / "↻ 미리보기 필요" 뱃지 / Char Sorting Order 입력 — 모두 영향 없음.
- `randomize_sorting.py` (이전 세션의 fixture 생성기) — 변경 없음.

## 사후 (사용자 작업)

- Unity 레포 (`D:\particleSortingTest\Assets\Editor\` 와 `D:\idleDante_test\Assets\Editor\ParticleAutoSorting\`) 는 사용자가 직접 커밋 (idleDante_test 사본은 git 추적 외부).
- 본 history 는 계획 레포에서 Claude 가 commit/push.
