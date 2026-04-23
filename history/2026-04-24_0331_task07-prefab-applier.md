# Task #7 Complete — PrefabApplier — 2026-04-24 03:31

## 전체 진행률
- 전체 17개 중 **8개 완료** (#16, #1, #2, #3, #4, #5, #6, #7)
- Phase 4(적용 레이어) 100% 완료 → Phase 5(보고서) / Phase 6(UI) 진입 가능

## 이 세션에서 완료한 작업

### ✅ #7 PrefabApplier.cs
- 경로: `D:\particleSortingTest\Assets\Editor\Apply\PrefabApplier.cs`
- 네임스페이스: `ParticleAutoSorting.Editor.Apply`
- 공개 API:
  - `Apply(IEnumerable<PrefabData>) → bool` — 성공적으로 적용되면 true, 취소/차단/대상 없음은 false

### 구현 요점
- **선택 필터링**: `IsSelectedForApply == true`인 프리팹만 대상으로 취합. 타겟 0개면 안내 다이얼로그 후 false.
- **오버플로 차단 (최우선)**: 선택된 프리팹 중 하나라도 `HasOverflow`면 문제 프리팹 이름 목록과 함께 에러 다이얼로그 띄우고 전체 적용 중단 (플랜의 "최우선 체크" 준수).
- **확인 다이얼로그**: 오버플로 없을 때만 노출. 인터리브 경고 프리팹 수 + GPU Instancing 꺼진 Material 수(프리팹별 set 중복 제거 후 합산) 표시.
  - "취소" 선택 시 어떤 변경도 없이 false 반환.
- **Undo 그룹**: `Undo.IncrementCurrentGroup` → `SetCurrentGroupName("Particle Auto Sorting Apply")` → `GetCurrentGroup()` 저장 → 모든 쓰기 끝난 뒤 `CollapseUndoOperations(group)`로 한 묶음으로 병합. 예외 발생 시에도 `try/finally`로 collapse 보장.
- **렌더러별 쓰기**:
  - `Kind == Trail_Module`은 스킵 (동일 `ParticleSystemRenderer`의 물리적 `sortingOrder/sortingFudge`는 Particle 엔트리에서 이미 기록됨 — 동일 Renderer 중복 쓰기 방지용 `HashSet<Renderer>` 도입).
  - 사용자 오버라이드 우선: `OilOverride ?? OilAfterAI`, `FudgeOverride ?? FudgeAfterAI`.
  - `Undo.RegisterCompleteObjectUndo` 호출 후 `WriteRenderer(...)` 수행 → `EditorUtility.SetDirty`.
- **WriteRenderer 분기**:
  - `ParticleSystemRenderer`: `sortingOrder`, `sortingFudge` 직접 대입.
  - `TrailRenderer`: `sortingOrder` 대입 + `SerializedObject`로 `m_SortingFudge` 대입 (Unity API에 public setter 없음).
  - 기타 Renderer(미래 확장 대비): `sortingOrder`만 대입.
- **프리팹 저장**: 각 프리팹의 Renderer 쓰기 후 `PrefabUtility.SavePrefabAsset(prefab)`. 전체 완료 후 `AssetDatabase.SaveAssets()`로 디스크 플러시.

### 스펙 해석 / 결정
- Trail_Module 엔트리는 물리적 렌더러가 부모 Particle과 공유되므로 별도 쓰기 대상에서 제외 (같은 Renderer에 두 번 쓰면 마지막 값만 남아 의미가 없음). 예기치 않은 overwrite를 막기 위한 안전 장치.
- "체크박스 해제된 엔트리는 제외" 규칙은 `IsSelectedForApply` 필드로만 판단 (플랜과 일치). UI 구현(#11, #13)이 이 필드를 토글함.
- Undo 기록은 Renderer 컴포넌트에만 수행. 프리팹 자산 저장 자체는 Undo 대상이 아님 — 재적용으로 되돌려야 함 (Unity에서는 `PrefabUtility.SavePrefabAsset` 결과가 Undo 스택에 포함되지 않음). 실제 Undo 버튼 동작은 `Undo.PerformUndo()`로 Renderer 상태 복원 후 `SavePrefabAsset`을 다시 호출하는 책임이 UI(#13)에 있음.
- 적용 확인 다이얼로그의 인터리브/Instancing 카운트는 현재 분석 결과 기반. #13의 "재계산"이 먼저 실행되어야 최신 수치. 이 조정은 UI 책임.

## 다음 작업 후보
- **#8 CsvReportExporter.cs** — BlockedBy: #1, #3, #6 (모두 해제)
- **#9 ParticleAutoSortingWindow 뼈대** — BlockedBy: #16 (해제) — UI 체인 시작점

## 수정한 파일
- `D:\particleSortingTest\Assets\Editor\Apply\PrefabApplier.cs` (신규)
- `D:\particle-auto-sorting\particle-auto-sorting\TASKS.md` (#7 체크)
