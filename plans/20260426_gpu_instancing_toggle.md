# Plan: 수정 모드에서 GPU Instancing 토글

- 생성일: 2026-04-26
- Tasks: `tasks/20260426_gpu_instancing_toggle.md` (G-NN)
- 범위: 기존 `TASKS.md` (Phase 0-7, #1-#17) 및 `tasks/20260424_csv_cumulative_batch_report.md` (R-NN) 와 **독립**.

## 배경

`Particle Auto Sorting` 툴은 Material 의 `enableInstancing` 상태를 **읽기 전용**으로만 노출한다 — Renderer 행의 ⓘ 아이콘(`Table.cs:157-162`), 프리팹 카드의 빨간 pill(`List.cs`), [적용] 다이얼로그의 카운트(`PrefabApplier.cs:63-67`). 데이터 필드 `RendererInfo.IsInstancingDisabled` 도 스캔 시점에 채워진다(`PrefabAnalyzer.cs:99,128,168`). 그러나 **쓰기 경로가 없음** — 사용자가 토글하려면 Material 자산을 직접 열어야 한다.

본 변경은 **수정 모드 ON 일 때** Material 단위로 GPU Instancing 을 토글할 수 있는 UI 와 [적용] 파이프라인을 추가한다.

## 설계 결정 (사용자 확정)

| 항목 | 결정 |
|---|---|
| 토글 단위 | **Material 자산 단위** (Renderer 행 단위 아님). 같은 Material 을 공유하는 여러 행은 단일 토글에 묶임. |
| UI 위치 | 수정 모드 ON 일 때만 Renderer 테이블 **위쪽**에 "GPU Instancing" 섹션 노출 |
| 적용 시점 | pending 상태 보관 → **기존 [적용] 버튼**에서 OiL/Fudge 와 같은 **Undo 그룹**으로 함께 처리 |
| 공유 경고 | 평상시 UI 깔끔, [적용] 확인 다이얼로그에서만 "변경 Material N개 — 다른 사용처 영향" 표시 |
| 충돌 정책 | 여러 프리팹에서 동일 Material 에 ON/OFF 가 충돌하면 [적용] 차단 + 다이얼로그 |

## 변경 파일 (Unity 레포)

| 파일 | 변경 |
|---|---|
| `D:\particleSortingTest\Assets\Editor\Data\PrefabData.cs` | `Dictionary<Material, bool> InstancingOverrides` 필드 추가 |
| `D:\particleSortingTest\Assets\Editor\ParticleAutoSortingWindow.Table.cs` | `DrawInstancingSection` + `CollectUniqueMaterials` 신규, `DrawRendererTable` 첫 줄에서 호출 (editMode 가드) |
| `D:\particleSortingTest\Assets\Editor\Apply\PrefabApplier.cs` | 충돌 검증 → 다이얼로그 메시지 갱신 → `ApplySingle` Material 쓰기 + Undo + post-apply `IsInstancingDisabled` 갱신 |

## 변경 파일 (계획 레포)

| 파일 | 변경 |
|---|---|
| `particle_auto_sorting_spec.json` | `gpu_instancing_section` 항목 추가 |
| `CLAUDE.md` (옵션) | 도메인 규칙 끝에 한 줄 추가 |
| `history/YYYYMMDD_HHMM_gpu_instancing_toggle.md` | 작업 완료 시점 기록 |

## 핵심 재사용 코드

| 항목 | 위치 |
|---|---|
| `editMode` 가드 패턴 | `Table.cs:173-186, 194-208` |
| `Undo.RegisterCompleteObjectUndo` + `CollapseUndoOperations` | `PrefabApplier.cs:83-96, 117` |
| `HashSet<Material>` dedupe | `PrefabApplier.cs:59-65` |
| `DrawPill` / `MaterialColor` | `List.cs` / `Table.cs:211` |
| `overrideLabelStyle` (변경 강조) | `ParticleAutoSortingWindow.cs:108-115` |
| `entryBoxStyle` / `sectionHeaderStyle` | `ParticleAutoSortingWindow.cs:82-98` |
| 적용 차단 dialog 패턴 | `PrefabApplier.cs:39-52` (오버플로 케이스) |

## 비변경 영역 (침범 금지)

- 기존 `TASKS.md` (Phase 0-7, #1-#17) 및 `tasks/20260424_csv_cumulative_batch_report.md` (R-NN)
- `SortingOptimizer.cs`, `BatchCounter.cs`, `CsvReportExporter.cs` — 본 건은 Material 쓰기에 한정
- 기존 OiL/Fudge UI/적용/오버플로 차단/인터리브 경고 흐름
- `Trail_Module skip` 룰 (`PrefabApplier.cs:111`)
- short 범위 클램프 금지 (Spec)

## 검증 (요약)

1. 수정 모드 ON → Renderer 테이블 위에 "GPU Instancing" 섹션 노출
2. 토글 변경 → "현재: ON/OFF" 라벨 파란 굵은체
3. [적용] 다이얼로그에 "GPU Instancing 변경 Material: N개" 라인
4. 적용 후 Material Inspector 의 `Enable GPU Instancing` 실제 변경, Ctrl+Z 한 번에 복원
5. 수정 모드 OFF → 섹션 비노출 / 기존 ⓘ 아이콘 유지
6. 동일 Material 공유 프리팹 간 ON↔OFF → 충돌 다이얼로그 차단
7. 토글 미사용 → 기존 OiL/Fudge 만 적용, Material asset mtime 변화 없음
8. (회귀) 인터리브/오버플로/CSV export 모두 정상

상세 검증 항목은 `tasks/20260426_gpu_instancing_toggle.md` G-05 참고.
