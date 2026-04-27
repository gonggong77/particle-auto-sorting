# 사용자 지정 가능한 Fudge 간격 (수정 모드 한정)

- **일자**: 2026-04-27
- **유형**: feat
- **요청**: "particle auto sorting 시스템에 sorting fudge 간격을 default 는 30으로 하고 사용자가 직접 이 값을 지정할 수 있으면 좋겠어 (수정모드가 켜졌을 때만)"

## 변경 요지

지금까지 `SortingOptimizer` 내부의 `private const float FudgeStep = 30f;` 로 박혀 있던 fudge 간격을 사용자 조정 가능한 값으로 승격시킨다. 단, 의도치 않은 변경을 막기 위해 입력란은 **수정 모드 ON 일 때만** TopBar 에 노출한다.

- Default: 30 (기존 spec 동일)
- 범위: 1~10000 (정수, IntField + Mathf.Clamp)
- 변경 시 자동 `RecomputeAll` (즉시 미리보기 갱신)

## 코드 변경

### `Optimization/SortingOptimizer.cs`
- `private const float FudgeStep = 30f;` → `public const float DefaultFudgeStep = 30f;`
- `OptimizeAll(prefabs, charSortingOrder)` → `OptimizeAll(prefabs, charSortingOrder, fudgeStep = DefaultFudgeStep)`
- `Optimize(data, charSortingOrder)` → `Optimize(data, charSortingOrder, fudgeStep = DefaultFudgeStep)`
- 내부 `AssignFudgeAndWriteAI(g, oil)` → `AssignFudgeAndWriteAI(g, oil, fudgeStep)`
- `AssignFudgeAndWriteAI` 의 `float step = size <= 1 ? 0f : FudgeStep` → 파라미터 `fudgeStep` 사용

### `Apply/PrefabApplier.cs`
- `Apply(prefabs, charSortingOrder)` → `Apply(prefabs, charSortingOrder, fudgeStep = SortingOptimizer.DefaultFudgeStep)`
- 내부 `Optimize(p, charSortingOrder)` → `Optimize(p, charSortingOrder, fudgeStep)` (적용 직전 재계산 시에도 동일 step 보장)

### `ParticleAutoSortingWindow.cs`
- `[SerializeField] int fudgeStep = (int)SortingOptimizer.DefaultFudgeStep;` 추가
- `const int FudgeStepMin = 1, FudgeStepMax = 10000;` 추가
- `OnEnable()` 신설: 도메인 리로드 후 직렬화된 값이 0/음수일 때 default 로 보정
- `DrawTopBar()` 에 `editMode` 토글 우측 영역에서 ON 일 때만 "Fudge 간격" 라벨 + IntField 노출. 변경 시 `RecomputeAll()` 자동 호출
- `RegisterPrefabs`/`RecomputeAll` 의 `Optimize` 호출에 `fudgeStep` 인자 전달

### `ParticleAutoSortingWindow.Bottom.cs`
- `RunApply()` 에서 `PrefabApplier.Apply(prefabs, charSortingOrder, fudgeStep)` 로 전달

### `idleDante_test` 사본 동기화
`D:\idleDante_test\Assets\Editor\ParticleAutoSorting\` 의 동일 4개 파일에 동일 패치를 복사 (`diff -q` 통과 확인).

## 문서/Spec 갱신

- `particle_auto_sorting_spec.json`
  - `wireframe.sections[global_controls].fields` 에 `fudge_step` 필드 추가 (`visible_when: edit_mode_toggle == ON`, default 30, min 1, max 10000)
  - `pending_decisions_for_implementation_stage` 에서 "Sorting Fudge 조정 값 단위" 항목을 "default 30, 수정 모드에서 사용자 지정 가능 (2026-04-27 변경)" 로 갱신
- `CLAUDE.md` 의 "Fudge 할당" 규칙 줄을 step 가변 + 수정 모드 한정 노출로 다시 작성

## 불변식 영향

- 사용자 입력 step ≥ 1 이 강제되므로 "동일 런 내 Fudge 단조성"(최상단 `(size-1)*step`, 최하단 0, 인접 차 `step`)은 그대로 보장된다.
- step = 0 은 입력 차단되어 단조성 위반(전부 0) 발생하지 않음.
- step 값과 무관하게 OrderInLayer 단조성, OiL 오버플로 검사, 인터리브 감지 로직은 그대로.

## 검증 항목 (사용자 측 Unity 에디터)

- [ ] 수정 모드 OFF 상태에서 TopBar 에 "Fudge 간격" 입력란이 보이지 않는다.
- [ ] 수정 모드 ON 시 "Fudge 간격" IntField 가 등장하고 default 값 30 이 표시된다.
- [ ] 값을 50 으로 변경 → 모든 등록 프리팹의 Above/Below 미리보기 fudge 가 (size-1)*50 부터 0 까지 50 간격으로 갱신된다.
- [ ] 값을 1 미만 (0, -5 등) 입력 시 1 로 클램프된다.
- [ ] [적용] 클릭 시 디스크에도 사용자가 지정한 step 기준으로 fudge 가 기록되고 Undo 한 번에 롤백된다.
- [ ] 수정 모드 OFF 로 다시 토글해도 마지막에 지정한 fudgeStep 값이 내부적으로 유지된다 ([재계산] 버튼이 동일 step 으로 동작).
- [ ] 도메인 리로드 (스크립트 변경, 컴파일) 후에도 fudgeStep 값이 보존되거나 0 일 경우 30 으로 자동 복원된다.
