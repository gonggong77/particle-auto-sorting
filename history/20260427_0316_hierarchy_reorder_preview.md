# 수정 모드에서 Hierarchy Sibling 순서 변경 + 미리보기

- 일시: 2026-04-27 03:16
- 별칭: `hierarchy-reorder` / `H-태스크`
- Plan: `plans/20260427_hierarchy_reorder_preview.md`
- Tasks: `tasks/20260427_hierarchy_reorder_preview.md` (H-01 ~ H-08)

## 증상 / 동기

`Particle Auto Sorting` 툴은 `RendererInfo.HierarchyOrder` (PrefabAnalyzer DFS 순회 결과, Trail = +0.5f) 로 행을 자동 정렬해 보여줄 뿐, 사용자가 직접 행 순서(= 같은 부모 GameObject 의 sibling 순서)를 바꿀 컨트롤이 없었다. 코드 어디에도 `Transform.SetSiblingIndex` 호출이 없었다. OiL/Fudge 만으로 해결되지 않는 인터리브/배치 분리 케이스를 툴 안에서 처리할 수 없었다.

## 변경 내용

### Unity 레포 (`D:\particleSortingTest\Assets\Editor\`)

| 파일 | 변경 |
|---|---|
| `Data/PrefabData.cs` | `bool HasSiblingReorder`, `bool NeedsPreviewRefresh` 필드 추가 (도메인 리로드 시 휘발) |
| `ParticleAutoSortingWindow.cs` | 중첩 `class DragState` (data, sourceParent, draggingTransform, hoverInsertIndex, hoverInsertRect) + `_drag` 필드. OnGUI 끝에 `HandleDragCleanup()` (MouseUp 시 잔여 _drag 클리어). 수정 모드 토글에 `≡ 핸들로 같은 부모 내 순서 변경` 툴팁. `RecomputeAll()` 끝에 `data.NeedsPreviewRefresh = false` |
| `ParticleAutoSortingWindow.Table.cs` | `ColHandle = 18f`, `DragGuideColor`. `DrawSection` — 첫 행 transform 추적 + 행 Rect 수집 + `HandleSiblingDrag` 호출. `DrawTableHeader` — editMode 시 핸들 cell 공간 예약. `DrawTableRow(data, r, isFirstRowOfTransform)` — 첫 행에만 `≡` 핸들 표시. `DrawDragHandle` — MouseDown 시 _drag 설정. `HandleSiblingDrag` — MouseDrag 위치로 insertIndex 계산 + 2px 가이드 라인 그리기 + MouseUp 시 `ApplyReorder` 호출. `ApplyReorder` — 같은 부모 transform 들의 정수부 슬롯을 보존한 채 새 순서대로 base 재할당, 같은 transform 의 모든 RendererInfo 에 동일 base + 기존 frac (Trail +0.5f) 보존. `HasSiblingReorder = true; NeedsPreviewRefresh = true` |
| `ParticleAutoSortingWindow.List.cs` | `ColorPreviewDirty` 색상. `DrawWarningPills` — `editMode && data.NeedsPreviewRefresh` 시 "↻ 미리보기 필요" 주황 pill 표시 |
| `ParticleAutoSortingWindow.Bottom.cs` | "재계산" 버튼을 editMode ON 일 때 "미리보기" 라벨 + 디스크 무변경 안내 툴팁으로 토글. 핸들러는 동일하게 `RecomputeAll()`. `RunApply()` 가 `PrefabApplier.Apply(prefabs, charSortingOrder)` 호출 |
| `Apply/PrefabApplier.cs` | `Apply(IEnumerable<PrefabData>, int charSortingOrder)` 시그니처 변경. 진입부에서 `targets` 각각에 `SortingOptimizer.Optimize` + `BatchCounter.CountPrefab` (미리보기 미클릭 케이스 보장) + `NeedsPreviewRefresh = false`. 다이얼로그 메시지에 `· 형제 순서 변경 프리팹: M개` + `· 전체 배치 수: a → b` 라인 추가. `ApplySingle` 시작부에 `ApplySiblingOrder(data)` 호출, 성공 후 `HasSiblingReorder = false; NeedsPreviewRefresh = false`. `ApplySiblingOrder` 신규 — 부모별로 자식 transform 그룹핑, 정수부 기준 새 순서, **Renderer 가 있는 자식이 차지하던 sibling slot 들에만** 새 순서대로 재배치 (Renderer 없는 자식의 절대 위치 보존), 같은 Undo 그룹 + 기존 `PrefabUtility.SavePrefabAsset` 한 번이 sibling 변경까지 함께 저장 |

### 계획 레포

| 파일 | 변경 |
|---|---|
| `particle_auto_sorting_spec.json` | `renderer_table_columns` 에 `hierarchy_reorder_handle`, `summary_and_actions.components` 에 `preview_button` 항목 추가 |
| `CLAUDE.md` | 도메인 핵심 규칙 끝에 "Hierarchy Sibling Reorder (수정 모드)" 한 줄 추가 |
| `tasks/20260427_hierarchy_reorder_preview.md` | H-01 [x] 표시 (이전 세션에서 처리) |
| `history/20260427_0316_hierarchy_reorder_preview.md` | 본 파일 |

## 핵심 설계 결정

- **이동 범위**: 같은 부모 GameObject 자식 사이 swap 만. 다른 부모 그룹 위 드롭은 no-op
- **이동 단위**: 같은 transform 의 모든 RendererInfo (PS+Trail) 동반 이동. 그룹 키 = `r.Renderer.transform`, base = `Mathf.Floor(HierarchyOrder)`, frac (`+0.5f` 등) 보존
- **반영 시점**: `HierarchyOrder` mutate 즉시 메모리 반영 → `SetSiblingIndex` + `SavePrefabAsset` 은 [적용] 시 일괄
- **자동 미리보기 안 함**: 매 swap 마다 `Optimize`/`CountPrefab` 비용 회피 + 사용자 명시 트리거 원칙. 대신 [미리보기] 클릭 = `RecomputeAll`
- **미리보기 미클릭 케이스**: Apply 진입부에서 한 번 `Optimize`+`CountPrefab` 강제 → 다이얼로그 배치수 합계가 항상 최신
- **Renderer 없는 자식 절대 위치 유지**: `parent.GetChild(i)` 순회로 Renderer 가 있는 자식이 차지하던 slot 인덱스만 수집 → 그 slot 들에 새 순서대로 채워 넣기

## 검증 (사용자 Unity 수동)

H-07 체크리스트는 `tasks/20260427_hierarchy_reorder_preview.md` 참고. 핵심 항목:

- [ ] 수정 모드 OFF → `≡` 핸들 비표시, "↻ 미리보기 필요" 뱃지 비표시
- [ ] 수정 모드 ON → 같은 부모 내 드래그 시 2px 가이드 라인, 드롭 시 행 swap + 뱃지 출현, OiL/Fudge/배치수 셀은 옛 값 유지
- [ ] [미리보기] 클릭 → 셀 갱신 + 뱃지 사라짐 + 프리팹 mtime 무변화
- [ ] 다른 부모 그룹 위 드롭 → no-op
- [ ] PS 행 이동 시 Trail 행 동반 이동, Trail 행 자체에는 핸들 없음
- [ ] [적용] 다이얼로그에 "형제 순서 변경 프리팹: M개" + "전체 배치 수: a → b"
- [ ] 적용 후 Unity Hierarchy 의 sibling 순서 = UI 순서
- [ ] Renderer 없는 자식 transform 절대 위치 유지
- [ ] Ctrl+Z 한 번에 sibling + OiL/Fudge + Material instancing 동시 롤백
- [ ] 미리보기 미클릭 후 [적용] → 다이얼로그 배치수가 최신
- [ ] (회귀) GPU Instancing 토글, 인터리브 경고, 오버플로 차단, CSV export 정상

## 비변경 영역 (침범 금지)

- 기존 `TASKS.md` (Phase 0-7, #1-#17)
- `tasks/20260424_csv_cumulative_batch_report.md` (R-NN)
- `tasks/20260426_gpu_instancing_toggle.md` (G-NN)
- `SortingOptimizer.cs`, `BatchCounter.cs`, `CsvReportExporter.cs` — 본 건은 sibling index 쓰기에 한정
- 기존 OiL/Fudge UI/적용/오버플로 차단/인터리브 경고 흐름
- `Trail_Module skip` (`PrefabApplier.cs` ApplySingle)
- `PrefabAnalyzer` 의 DFS 순회 / Trail +0.5f 오프셋 규칙
- short 범위 클램프 금지 (Spec)

## 사후 (사용자 작업)

- Unity 레포 (`D:\particleSortingTest`) 는 사용자가 직접 커밋
- 본 history + spec.json + CLAUDE.md + tasks 는 계획 레포에서 Claude 가 commit/push
