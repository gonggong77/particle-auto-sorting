# [하이어라키만 재정렬] 정렬 방향 반전 — idleDante_test 사본 동기화

- 일시: 2026-04-27 19:20
- 직전 history: `20260427_1915_hierarchy_reorder_direction_flip.md`
- Plan: `C:\Users\lhg07\.claude\plans\snappy-bouncing-fog.md`

## 동기

직전 1915 작업은 `D:\particleSortingTest\Assets\Editor\Apply\HierarchyReorderer.cs` 한 사본만 변경했다. 사용자 확인을 거쳐 idleDante_test 사본도 동일 동작을 갖도록 동기화한다.

## 변경 내용

### Unity 레포 (사본 1곳)

| 파일 | 변경 |
|---|---|
| `D:\idleDante_test\Assets\Editor\ParticleAutoSorting\Apply\HierarchyReorderer.cs` | `ReorderChildrenAtParent` 의 `Sort` 비교자 두 줄 반전 + 주석 한 블록 갱신 (라인 139~149) — particleSortingTest 사본과 동일 패치 |

### 계획 레포

| 파일 | 변경 |
|---|---|
| `history/20260427_1920_port_direction_flip_to_idledante_test.md` | 본 파일 |

## 정확한 패치

`(OiL DESC, Fudge ASC)` → `(OiL ASC, Fudge DESC)`. 코드 디프는 1915 history 의 Before/After 블록과 동일하므로 중복 게재 생략.

## 사후

- 두 사본의 정렬 동작이 다시 일치. 이후 1915 패치를 추가로 손볼 일이 생기면 두 사본 동시 적용 패턴(`20260424_1755_port_feature_to_idledante_test.md` 흐름)을 유지.
- Unity 레포는 사용자가 직접 커밋/푸시. 본 history 만 계획 레포 commit/push.
