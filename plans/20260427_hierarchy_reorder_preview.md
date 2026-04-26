# Plan: 수정 모드에서 Hierarchy Sibling 순서 변경 + 미리보기

- 생성일: 2026-04-27
- Tasks: `tasks/20260427_hierarchy_reorder_preview.md` (H-NN)
- 별칭: `hierarchy-reorder`, `H-태스크`
- 범위: 기존 `TASKS.md` (Phase 0-7, #1-#17), `tasks/20260424_csv_cumulative_batch_report.md` (R-NN), `tasks/20260426_gpu_instancing_toggle.md` (G-NN) 와 **독립**.

## 배경

`Particle Auto Sorting` 툴은 행을 `RendererInfo.HierarchyOrder`(DFS 순회 순으로 부여된 float, `PrefabAnalyzer.cs:113` Trail 만 +0.5f) 기준으로 자동 정렬해 표시할 뿐, 사용자가 직접 행 순서(=프리팹 내 GameObject sibling 순서)를 바꿀 컨트롤이 없다. `Transform.SetSiblingIndex` 는 코드 어디에도 사용되지 않는다.

OiL/Fudge 만으로 풀리지 않는 인터리브/배치 분리 케이스를 이 툴 안에서 해결할 수 있도록, **수정 모드 ON 일 때 같은 부모 GameObject 의 자식들 사이에서 드래그앤드롭 reorder**를 추가한다. 변경은 메모리에만 반영되고, 실제 `SetSiblingIndex` + `SavePrefabAsset` 은 기존 [적용] 버튼에서 일괄 처리한다.

추가로, 순서를 바꿔본 결과의 **OiL · Fudge · 예상 배치수**를 프리팹 변경 없이 미리 보기 위해 [미리보기] 버튼을 둔다 — 핵심 발견: `RecomputeAll()` (`Window.cs:251~259`) 은 `PrefabAnalyzer.Analyze` 를 다시 호출하지 않고 캐시된 `data.Renderers` 의 현재 `HierarchyOrder` 만으로 `SortingOptimizer.Optimize` + `BatchCounter.CountPrefab` 를 다시 돌려 OiL/Fudge/배치수를 갱신한다. 즉 미리보기는 `RecomputeAll()` 호출 한 줄로 충분하며, 신규 계산기 코드는 0줄.

## 설계 결정 (사용자 확정)

| 항목 | 결정 |
|---|---|
| 이동 범위 | **같은 부모 GameObject** 자식 사이 swap 만. 부모 변경/트리 이동 금지 |
| 이동 단위 | 같은 transform 의 모든 RendererInfo (PS+Trail 동반 이동). 그룹 키 = `r.Renderer.transform` |
| 반영 시점 | 메모리(`HierarchyOrder` mutate) 즉시 → 실제 `SetSiblingIndex` + `SavePrefabAsset` 은 기존 [적용] 버튼에서만 |
| UI | 행 좌측 `≡` 핸들 드래그앤드롭 (수정 모드 OFF 시 비표시) |
| 미리보기 | 별도 [미리보기] 버튼. 클릭 시 `RecomputeAll()` 호출 → OiL/Fudge/배치수 재산출 (디스크 무변경) |
| 자동 미리보기 | **하지 않음**. 매 swap 마다 전체 재계산 비용 회피 + 사용자 명시 트리거 원칙 |
| 변경 표식 | reorder 발생한 프리팹은 행에 "↻ 미리보기 필요" 뱃지. RecomputeAll 후 클리어 |

## 변경 파일 (Unity 레포)

| 파일 | 변경 |
|---|---|
| `D:\particleSortingTest\Assets\Editor\Data\PrefabData.cs` | `bool HasSiblingReorder`, `bool NeedsPreviewRefresh` 필드 추가 |
| `D:\particleSortingTest\Assets\Editor\ParticleAutoSortingWindow.Table.cs` | 행 좌측 드래그 핸들 cell + 마우스 이벤트(MouseDown/Drag/Up + Repaint 가이드라인) + reorder swap (`HierarchyOrder` mutate, dirty 플래그 set) |
| `D:\particleSortingTest\Assets\Editor\ParticleAutoSortingWindow.cs` | 수정 모드 토글 옆 툴팁 ("≡ 핸들로 같은 부모 내 순서 변경"), 드래그 상태(`_drag`) 필드 |
| `D:\particleSortingTest\Assets\Editor\ParticleAutoSortingWindow.Bottom.cs` | 수정 모드 ON 일 때 "재계산" 버튼 라벨/툴팁을 "미리보기" 로 토글 (핸들러는 `RecomputeAll` 동일) |
| `D:\particleSortingTest\Assets\Editor\ParticleAutoSortingWindow.List.cs` | reorder dirty(`NeedsPreviewRefresh==true`) 행에 "↻ 미리보기 필요" 뱃지 |
| `D:\particleSortingTest\Assets\Editor\Apply\PrefabApplier.cs` | `ApplySiblingOrder(PrefabData)` 신규 + `ApplySingle` 시작부 호출, 다이얼로그 메시지(형제 순서 변경 프리팹 수 + 전체 배치수 합계) 추가, 성공 후 `HasSiblingReorder = false` 리셋. Apply 시작 시점에 한 번 `RecomputeAll()` 호출(미리보기 미클릭 케이스 보장) |

## 변경 파일 (계획 레포)

| 파일 | 변경 |
|---|---|
| `particle_auto_sorting_spec.json` | `hierarchy_reorder_section`, `preview_button` 항목 추가 |
| `CLAUDE.md` (옵션) | 도메인 규칙 끝에 한 줄 추가 |
| `history/YYYYMMDD_HHMM_hierarchy_reorder_preview.md` | 작업 완료 시점 기록 |

## 핵심 재사용 코드

| 항목 | 위치 |
|---|---|
| 행 정렬 (`HierarchyOrder` 비교) | `Table.cs:141~151` `FilterByGroup` |
| 섹션/행 그리기 골격 | `Table.cs:129~` `DrawSection`, `:168~` `DrawTableRow` |
| **OiL/Fudge/배치수 미리보기 = `RecomputeAll()`** | `Window.cs:251~259` (내부 `SortingOptimizer.Optimize` + `BatchCounter.CountPrefab`) |
| 배치수 표시 | `Window.List.cs:68` 행 우측, `Window.Bottom.cs:33` 하단 합계 |
| Apply Undo 그룹 | `PrefabApplier.cs:169` `Undo.RegisterCompleteObjectUndo` |
| 프리팹 자산 저장 | `PrefabApplier.cs:185` `PrefabUtility.SavePrefabAsset` |
| `editMode` 가드 패턴 | `Table.cs` 기존 OiL/Fudge IntField/FloatField 분기 |

## 비변경 영역 (침범 금지)

- 기존 `TASKS.md` (Phase 0-7, #1-#17)
- `tasks/20260424_csv_cumulative_batch_report.md` (R-NN)
- `tasks/20260426_gpu_instancing_toggle.md` (G-NN)
- `SortingOptimizer.cs`, `BatchCounter.cs`, `CsvReportExporter.cs` — 본 건은 sibling index 쓰기에 한정
- 기존 OiL/Fudge UI/적용/오버플로 차단/인터리브 경고 흐름
- `Trail_Module skip` (`PrefabApplier.cs:111`)
- short 범위 클램프 금지 (Spec)
- `PrefabAnalyzer` 의 DFS 순회 / Trail +0.5f 오프셋 규칙

## 검증 (요약)

1. 수정 모드 OFF → 행 좌측 `≡` 핸들 비표시
2. 수정 모드 ON → 핸들 드래그 시 같은 부모 그룹 내에서만 가이드라인 표시, 드롭하면 행 순서만 즉시 swap. OiL/Fudge/배치수 셀은 옛 값 유지 + "↻ 미리보기 필요" 뱃지
3. [미리보기] 클릭 → OiL/Fudge/배치수 재계산, 뱃지 사라짐, 디스크 무변경
4. 다른 부모 그룹 위로 드롭 → no-op
5. ParticleSystem 행 이동 시 같은 GameObject 의 Trail 행 동반 이동
6. [적용] 다이얼로그에 "형제 순서 변경 프리팹: M개" + "전체 배치 수: a → b" 표시
7. 적용 후 Unity Hierarchy 의 sibling 순서가 UI 와 일치, Renderer 없는 자식 절대 위치 유지
8. **Ctrl+Z** 한 번에 sibling 순서 + OiL/Fudge 동시 롤백
9. 미리보기 미클릭 후 바로 [적용] → Apply 시작부의 `RecomputeAll()` 이 최신 값 보장
10. (회귀) GPU Instancing 토글, 인터리브 경고, 오버플로 차단, CSV export 정상

상세 검증 항목은 `tasks/20260427_hierarchy_reorder_preview.md` H-06 참고.
