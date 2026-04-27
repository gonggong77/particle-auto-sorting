# Tasks: 수정 모드에서 GPU Instancing 토글

- 생성일: 2026-04-26
- Plan: `plans/20260426_gpu_instancing_toggle.md`
- 범위: 기존 `TASKS.md` (Phase 0-7, #1-#17) 및 `tasks/20260424_csv_cumulative_batch_report.md` (R-NN) 와 **독립**. 본 파일에서만 추적.
- Task ID 규칙: `G-NN` (GPU Instancing 관련, 본 요청 한정)

## Dependency Graph

```
G-01 PrefabData.InstancingOverrides 필드 추가
  ├─ G-02 Table.cs DrawInstancingSection 신규 + 통합
  └─ G-03 PrefabApplier 충돌 검증 + Material 쓰기 + post-apply 캐시 갱신
       └─ G-05 Unity 수동 검증
G-04 Spec/CLAUDE.md 동기화                ← G-02/G-03 와 병렬 가능
G-06 history 기록 + 계획 레포 commit/push  ← G-01..G-05 완료 후
```

---

## G-01 — `PrefabData` 에 instancing pending 저장소 추가

- **Status**: [x] (2026-04-26 완료)
- **File**: `D:\particleSortingTest\Assets\Editor\Data\PrefabData.cs`
- **BlockedBy**: 없음
- **변경**:
  ```csharp
  public Dictionary<Material, bool> InstancingOverrides = new Dictionary<Material, bool>();
  ```
  - 키 = 이 프리팹에서 사용 중인 Material 자산
  - 값 = 사용자가 원하는 `enableInstancing` 결과
  - 항목 없음 = 변경 없음 (현재 상태 유지)
- **불변**: 기존 필드/직렬화/생성자 시그니처 그대로. 도메인 리로드 시 휘발 (기존 `OilOverride`/`FudgeOverride` 와 동일 라이프사이클).
- **using**: `System.Collections.Generic`, `UnityEngine` 이 이미 있는지 확인 후 부족 시 추가.

---

## G-02 — Material 단위 토글 UI

- **Status**: [x] (2026-04-26 완료)
- **File**: `D:\particleSortingTest\Assets\Editor\ParticleAutoSortingWindow.Table.cs`
- **BlockedBy**: G-01
- **추가 메서드**:
  - `static List<Material> CollectUniqueMaterials(PrefabData data)` — null skip + 등장 순서 보존 (`HashSet<Material>` dedupe)
  - `void DrawInstancingSection(PrefabData data)` — 첫 줄 `if (!editMode) return;` 가드, `entryBoxStyle` + `sectionHeaderStyle("GPU Instancing (수정 모드)")`, Material 마다 한 행 [pill | name | "현재: ON/OFF" | toggle]
- **호출 삽입**: `DrawRendererTable` 첫 줄에 `DrawInstancingSection(data);` (`DrawInterleavePairList` 위)
- **상호작용 규칙**:
  - 토글값 == `mat.enableInstancing` (현재값) → `InstancingOverrides.Remove(mat)` (변경 없음으로 정리)
  - 다르면 → `InstancingOverrides[mat] = edited` (pending)
  - dirty 항목은 "현재: ON/OFF" 라벨을 `overrideLabelStyle` (파란 굵은체) 로 강조
- **재사용**:
  - `DrawPill` (`List.cs`)
  - `MaterialColor` (`Table.cs:211`)
  - `entryBoxStyle`, `sectionHeaderStyle`, `overrideLabelStyle` (`ParticleAutoSortingWindow.cs:82-115`)
- **수정 모드 OFF 시**: 섹션 자체 비노출. 기존 ⓘ 아이콘은 그대로.
- **참고 코드 스니펫**: `plans/20260426_gpu_instancing_toggle.md` 의 단축판이 아닌, 풀 plan (`C:\Users\lhg07\.claude\plans\particle-auto-sorting-melodic-biscuit.md`) "단계별 설계 § 3" 참고.

---

## G-03 — Apply 파이프라인 확장

- **Status**: [x] (2026-04-26 완료)
- **File**: `D:\particleSortingTest\Assets\Editor\Apply\PrefabApplier.cs`
- **BlockedBy**: G-01
- **삽입 지점 1 — 오버플로 검사 직후 (라인 52~54 사이)**:
  - `targets` 의 `InstancingOverrides` 를 `Dictionary<Material, bool> globalPending` 으로 머지
  - 동일 Material 에 ON/OFF 충돌 시 `conflictMats` 수집
  - `conflictMats.Count > 0` → `EditorUtility.DisplayDialog("GPU Instancing 충돌 — 적용 차단", ...)` 후 `return false`
- **삽입 지점 2 — 다이얼로그 메시지 (라인 70~74)**:
  - `globalPending` 중 `mat.enableInstancing != value` 인 개수 = `matChangeCount`
  - `matChangeCount > 0` 이면 메시지에 추가:
    `"· GPU Instancing 변경 Material: N개 (이 Material을 사용하는 다른 프리팹/씬에도 영향)\n"`
- **삽입 지점 3 — `ApplySingle` 의 renderer 루프 종료 후 / `SavePrefabAsset` 직전 (라인 121)**:
  - `data.InstancingOverrides` 순회
  - null skip / 같은 값 skip
  - `Undo.RegisterCompleteObjectUndo(mat, UndoGroupName)` → `mat.enableInstancing = value` → `EditorUtility.SetDirty(mat)`
- **삽입 지점 4 — 모든 `ApplySingle` 호출 후 / `AssetDatabase.SaveAssets()` 직전 (라인 99)**:
  - `targets` 의 모든 `RendererInfo` 에서 `IsInstancingDisabled = !SharedMaterial.enableInstancing` 갱신 (ⓘ 아이콘 stale 방지)
  - `p.InstancingOverrides.Clear()`
- **불변**:
  - 기존 `Undo.IncrementCurrentGroup` + `CollapseUndoOperations` (라인 83~96) → Material/Renderer 변경이 한 Undo 그룹
  - 기존 `AssetDatabase.SaveAssets()` (라인 99) 가 Material 자산 dirty flush
  - `Trail_Module skip` (라인 111) 그대로 — Trail Material 은 별도 RendererInfo 로 들어와 자체 처리

---

## G-04 — Spec / CLAUDE.md 동기화

- **Status**: [x] (2026-04-26 완료, spec.json만 갱신 / CLAUDE.md 옵션은 분량 부담으로 보류)
- **Files**:
  - `D:\particle-auto-sorting\particle-auto-sorting\particle_auto_sorting_spec.json`
  - `D:\particle-auto-sorting\particle-auto-sorting\CLAUDE.md` (옵션 → 미진행)
- **BlockedBy**: G-02 / G-03 와 병렬 가능 (문구만 확정되면 됨)
- **spec.json 추가 항목** (`wireframe.sections` 내 Renderer 테이블 직전):
  ```json
  {
    "id": "gpu_instancing_section",
    "label": "GPU Instancing (수정 모드)",
    "visible_when": "edit_mode_toggle == ON",
    "scope": "per Material asset (프리팹 내 unique Material)",
    "control": "toggle (per material)",
    "applied_at": "기존 [적용] 버튼과 같은 Undo 그룹",
    "warning_dialog": "변경되는 Material 자산 개수 + '다른 프리팹/씬에도 영향' 문구",
    "conflict_policy": "여러 프리팹 간 동일 Material ON/OFF 충돌 시 적용 차단"
  }
  ```
- **CLAUDE.md 추가 (옵션)**: "도메인 핵심 규칙" 끝에
  `- **GPU Instancing 토글 (수정 모드)**: Material 자산 단위로 ON/OFF, [적용] 버튼에 통합, 같은 Undo 그룹.`

---

## G-05 — Unity 수동 검증

- **Status**: [ ]
- **BlockedBy**: G-02 + G-03
- **체크리스트**:
  - [ ] 수정 모드 ON → "GPU Instancing" 섹션이 Renderer 테이블 위에 노출
  - [ ] 토글 변경 시 "현재: ON/OFF" 라벨이 파란 굵은체로 강조
  - [ ] [적용] 다이얼로그에 `GPU Instancing 변경 Material: N개` 라인 표시
  - [ ] 적용 후 Project 창의 Material Inspector → `Enable GPU Instancing` 실제 변경 확인
  - [ ] **Ctrl+Z** 한 번에 Material 변경 + OiL/Fudge 변경 동시 롤백
  - [ ] 수정 모드 OFF → 섹션 비노출, 기존 ⓘ 아이콘은 유지
  - [ ] 동일 Material 공유 프리팹 A/B 간 ON ↔ OFF → 충돌 다이얼로그로 [적용] 차단
  - [ ] 토글 미사용 → 기존 OiL/Fudge 만 적용 (Material asset mtime 변화 없음)
  - [ ] 적용 후 동일 행 Material 셀의 ⓘ 아이콘 stale 없이 갱신
  - [ ] Trails 모듈 사용 프리팹: trail Material 도 섹션에 항목으로 노출
  - [ ] Unity Console 컴파일 에러 0개
  - [ ] (회귀) 인터리브 경고 / 오버플로 차단 / CSV export 정상

---

## G-06 — history 기록 + 계획 레포 commit/push

- **Status**: [ ]
- **BlockedBy**: G-01 ~ G-05
- **산출물**:
  - `history/YYYYMMDD_HHMM_gpu_instancing_toggle.md` (증상/원인/수정/검증)
  - 계획 레포에서 `plans/20260426_gpu_instancing_toggle.md`, `tasks/20260426_gpu_instancing_toggle.md`, `particle_auto_sorting_spec.json`, `CLAUDE.md`(변경 시), `history/...md` 를 `git add` → commit → push
  - **Unity 레포(`D:\particleSortingTest`) 는 사용자가 직접 커밋** (CLAUDE.md 룰)
- **커밋 메시지 초안**:
  ```
  feat: 수정 모드에서 Material 단위 GPU Instancing 토글 추가

  - PrefabData.InstancingOverrides 신설 (pending Material 토글 저장소)
  - Table.cs DrawInstancingSection: 수정 모드 ON 일 때만 노출, Material 자산별 ON/OFF
  - PrefabApplier: ON/OFF 충돌 검증 → 적용 차단, 같은 Undo 그룹에서 Material.enableInstancing 쓰기, post-apply IsInstancingDisabled 캐시 갱신
  - 적용 다이얼로그에 변경 Material 개수 + 공유 영향 경고
  - spec.json 에 gpu_instancing_section 추가
  ```

---

## 비변경 영역 (침범 금지)

- 기존 `TASKS.md` (Phase 0-7, #1-#17)
- `tasks/20260424_csv_cumulative_batch_report.md` (R-NN)
- `SortingOptimizer.cs`, `BatchCounter.cs`, `CsvReportExporter.cs`
- 기존 OiL/Fudge UI/적용/오버플로 차단/인터리브 경고 흐름
- `Trail_Module skip` (`PrefabApplier.cs:111`)
- short 범위 클램프 금지 (Spec)
