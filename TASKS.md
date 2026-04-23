# Particle Auto Sorting — Implementation Tasks

> Plan 파일: `C:\Users\lhg07\.claude\plans\partitioned-questing-porcupine.md`
> 생성일: 2026-04-23

## Task Dependency Graph

```
#16 폴더 초기화 (시작점)
 ├─ #1 RendererInfo ┐
 ├─ #2 MaterialGroupInfo ├─ #4 PrefabAnalyzer ─ #5 BatchCounter ─ #6 SortingOptimizer ┬─ #7 PrefabApplier
 ├─ #3 PrefabData  ┘                                                                    └─ #8 CsvReportExporter
 └─ #9 Window 뼈대
    ├─ #10 드래그앤드롭 (+ #4)
    │   └─ #11 프리팹 리스트 (+ #6)
    │       ├─ #12 Renderer 테이블 (+ #5)
    │       ├─ #13 하단 버튼 (+ #5, #7)
    │       └─ #14 CSV UI (+ #8)
    └─ #15 상태 표시줄

#17 Unity 수동 검증 ← #12, #13, #14, #15 완료 후
```

---

## Phase 0 — 초기화

### [x] #16 Unity 프로젝트 Editor 툴 폴더 구조 준비
- **Blocks**: #1, #2, #3, #9
- **Unity 프로젝트 경로**: `D:\particleSortingTest\` (URP 템플릿)
- **git 관리**: 사용자가 GitHub에서 직접 관리 (Claude는 Unity 레포에 git 명령 실행하지 않음)
- **계획 문서 경로**: `D:\particle-auto-sorting\particle-auto-sorting\` (GitHub: `gonggong77/particle-auto-sorting`) — Claude 커밋 대상
- [x] `Assets/Editor/` 하위 폴더 생성 완료: `Data/`, `Analysis/`, `Optimization/`, `Apply/`, `Report/`
- [x] `.gitignore` 생성 완료 (사용자가 필요시 활용)
- [x] `.unitypackage` 배포 전제 — `Assets/Editor/` 전체가 패키지 루트가 되도록 외부 의존 없이 독립적으로 구성 (다른 폴더 참조 금지, asmdef 하나로 묶기 권장) → `Assets/Editor/ParticleAutoSorting.Editor.asmdef` 생성 (Editor 전용, autoReferenced: true)

---

## Phase 1 — 데이터 모델

### [x] #1 RendererInfo.cs 데이터 모델 + RendererKind enum 작성
- **File**: `Assets/Editor/Data/RendererInfo.cs`
- **BlockedBy**: #16
- **Enum (동일 파일 내 선언)**: `RendererKind { Particle, Trail_Component, Trail_Module }`
- **Fields**: ObjectName, Renderer, SharedMaterial, RenderMode, Mesh, SortingLayerID, HierarchyOrder(Trail offset 포함), GroupTag, Kind(RendererKind), OilBefore, FudgeBefore, OilAfterAI, FudgeAfterAI, OilOverride(nullable), FudgeOverride(nullable), HasInterleaveWarning, InterleavedWith(List&lt;string&gt;), IsInstancingDisabled

### [x] #2 MaterialGroupInfo.cs 데이터 모델 작성
- **File**: `Assets/Editor/Data/MaterialGroupInfo.cs`
- **BlockedBy**: #16
- **Fields**: SharedMaterial, Members(List&lt;RendererInfo&gt;), RepresentativeOrder(min hierarchyOrder), AssignedOrderInLayer, GroupColor(UI pill용)

### [x] #3 PrefabData.cs 데이터 모델 작성
- **File**: `Assets/Editor/Data/PrefabData.cs`
- **BlockedBy**: #16
- **Fields**: Prefab(GameObject), Renderers(List&lt;RendererInfo&gt;), Warnings(List&lt;string&gt;), BatchBefore, BatchAfter, IsExpanded, HasAboveBelow, HasOverflow(적용 차단용), IsSelectedForApply(체크박스 상태, 기본 true)

---

## Phase 2 — 분석 레이어

### [x] #4 PrefabAnalyzer.cs 프리팹 스캔 구현
- **File**: `Assets/Editor/Analysis/PrefabAnalyzer.cs`
- **BlockedBy**: #1, #2, #3
- Above/Below 루트 탐색 (없으면 경고)
- 활성 ParticleSystemRenderer + TrailRenderer 수집
- `ParticleSystem.trails.enabled` 감지 → Trail_Module 항목 추가
- HierarchyOrder 계산 (Trail = Particle + 0.5)
- 비활성 오브젝트 제외 + 경고
- `Material.enableInstancing` 체크 → `IsInstancingDisabled` 세팅
- null Material silent skip
- Sub Emitter 해석 실패 시 경고

### [x] #5 BatchCounter.cs 배치 수 계산 구현
- **File**: `Assets/Editor/Analysis/BatchCounter.cs`
- **BlockedBy**: #1, #4
- BatchKey = `[sortingLayerID, sharedMaterial, renderMode, mesh]`
- SortKey = `[sortingLayerID, orderInLayer, sortingFudge, hierarchyOrder]`
- SortKey 오름차순 정렬 → 인접 BatchKey 비교 → 변경 시마다 +1
- Above/Below 그룹별 계산 후 합산
- 프리팹별 / 전체 before-after 산출

---

## Phase 3 — 최적화 레이어

### [x] #6 SortingOptimizer.cs OiL/Fudge 최적화 구현
- **File**: `Assets/Editor/Optimization/SortingOptimizer.cs`
- **BlockedBy**: #1, #2, #5
- Material 그룹핑 후 대표 순서 = min(hierarchyOrder)
- OiL 할당: Above는 charSortingOrder+1,+2... / Below는 charSortingOrder-N,...,-1
- OiL short 범위(-32768~32767) 오버플로 체크 → **clamp 금지, 오버플로 발생 시 `HasOverflow` 플래그 세팅 + 경고 메시지 수집** (적용 차단은 #7에서 처리)
- Fudge 자동 스케일링: `step = min(0.1, 0.9 / (group_size - 1))`
- Fudge 공식: `fudge[rank] = (group_size - 1 - rank) * step` (rank 0 = 가장 위 = 뒤 = 최대 Fudge)
- 인터리브 감지: SortKey 기준 정렬된 전체 리스트에서의 위치 인덱스를 Material 그룹별로 수집 → `max(A.positions) > min(B.positions) AND max(B.positions) > min(A.positions)` → HasInterleaveWarning + InterleavedWith 세팅
  - `positions`: 정렬된 리스트에서 해당 Renderer의 0-based 인덱스

---

## Phase 4 — 적용 레이어

### [ ] #7 PrefabApplier.cs 적용 + Undo 구현
- **File**: `Assets/Editor/Apply/PrefabApplier.cs`
- **BlockedBy**: #1, #6
- **적용 차단 조건 (최우선 체크)**: 체크된 프리팹 중 하나라도 OiL 오버플로(`HasOverflow`) 있으면 에러 대화상자 띄우고 적용 전체 중단 — 사용자가 Char Sorting Order 값을 수정하거나 해당 프리팹을 해제해야 진행 가능
- `Undo.IncrementCurrentGroup` + `Undo.SetCurrentGroupName("Particle Auto Sorting Apply")`로 Undo 그룹 생성
- `Undo.RegisterCompleteObjectUndo`로 각 Renderer 상태 기록
- 각 Renderer에 OiL/Fudge 쓰기
- `PrefabUtility.SavePrefabAsset` 호출
- 적용 전 `EditorUtility.DisplayDialog` 확인 대화상자 (오버플로 없을 때만 표시):
  - 인터리브 경고 프리팹 수 문구
  - GPU Instancing 꺼진 Material 수 문구
- **체크박스로 선택된 프리팹만 일괄 적용** (#11에서 엔트리별 체크박스 추가)

---

## Phase 5 — 보고서

### [ ] #8 CsvReportExporter.cs CSV 내보내기 구현
- **File**: `Assets/Editor/Report/CsvReportExporter.cs`
- **BlockedBy**: #1, #3, #6
- UTF-8 BOM 인코딩 (`new UTF8Encoding(true)`)
- `EditorUtility.SaveFilePanel`로 경로 선택
- 컬럼: 프리팹명, 오브젝트명, Above/Below, Material, OiL before/after, Fudge before/after, 프리팹별 배치 수 before/after, 전체 배치 수 before/after
- 쉼표/줄바꿈/쌍따옴표 포함 시 따옴표 이스케이프

---

## Phase 6 — UI

### [ ] #9 ParticleAutoSortingWindow 뼈대 + 전역 컨트롤
- **File**: `Assets/Editor/ParticleAutoSortingWindow.cs`
- **BlockedBy**: #16
- MenuItem `"Tools/particle auto sorting"`로 창 오픈
- 전역 컨트롤:
  - Char Sorting Order (IntField, 기본 0, 범위 제한 없음)
  - 수정 모드 toggle (기본 OFF = 읽기 전용 / ON = OiL/Fudge 편집 가능)
- 창 상단 레이아웃

### [ ] #10 드래그앤드롭 + 프리팹 등록 UI
- **BlockedBy**: #9, #4
- `DragAndDrop` API로 등록 존 구현
- 복수 등록 지원
- 중복 등록 시 경고 + 거부
- 등록 시 `PrefabAnalyzer.Analyze` 자동 호출
- 각 엔트리에 × 제거 버튼

### [ ] #11 프리팹 리스트 UI + 경고 pill + 적용 체크박스
- **BlockedBy**: #10, #6
- 헤더: **적용 체크박스(기본 체크됨)**, 확장 toggle(▼/▶), 파일명, 배치 수 요약(before→after), × 제거 버튼
- 체크 해제된 엔트리는 적용 대상에서 제외 (#7이 이 상태를 읽음), UI상 흐리게(dim) 표시
- OiL 오버플로 있는 엔트리는 빨간 경고 pill `"OiL 범위 초과"` 추가 표시 (체크는 유지하되 적용 시 차단됨을 사용자가 인지)
- 경고 pill:
  - 인터리브: 주황색
  - Instancing 비활성: 빨간색
  - Above/Below 구조 누락: 회색
  - 비활성 오브젝트 포함: 노란색
  - OiL 오버플로: 진한 빨강 (적용 차단)
- 경고 있는 엔트리 테두리에 색상 적용

### [ ] #12 Renderer 테이블 UI (Above/Below 섹션)
- **BlockedBy**: #11, #5
- 확장 시 Above/Below 섹션별 테이블 렌더링
- 컬럼: 오브젝트명, Material(같은 Material = 같은 색 pill + ⓘ 아이콘 if Instancing off), OiL before/after, Fudge before/after
- 수정 모드일 때 OiL/Fudge 편집 가능
- AI 예측값과 다르면 파란색 강조
- 인터리브 연루 행에 ⚠ 아이콘 + 호버 툴팁
- 확장 섹션 상단에 인터리브된 Material 쌍 목록

### [ ] #13 하단 버튼 영역 (재계산/Undo/적용) + 전체 배치 수 + 전체 선택/해제
- **BlockedBy**: #11, #5, #7
- 수평 레이아웃:
  - **전체 선택/해제 토글** (체크박스 또는 버튼, 현재 상태가 전부 체크면 "전체 해제", 아니면 "전체 선택")
  - **선택된 프리팹 수 표시** (`3 / 5개 선택`)
  - 전체 배치 수 (before→after, 체크된 프리팹만 합산, 숫자만 강조 없이)
  - 재계산 버튼 (수정 모드 값 반영 후 BatchCounter 재실행)
  - Undo 버튼 (Unity Undo 시스템 호출, 직전 적용분 일괄 되돌림)
  - 적용 버튼 (primary 스타일, PrefabApplier.Apply 호출 — 체크된 것만 전달)

### [ ] #14 CSV 내보내기 UI 패널
- **BlockedBy**: #11, #8
- 섹션 "보고서 내보내기 (CSV)"
- 경로 입력 필드 (편집 가능)
- 찾아보기 버튼 (`EditorUtility.SaveFilePanel`)
- 내보내기 버튼 (`CsvReportExporter` 호출)

### [ ] #15 하단 상태 표시줄
- **BlockedBy**: #9
- 표시 상태: "분석 중", "적용 중", "완료", 등록된 프리팹 수
- 타임스탬프 없음
- 상태 전이는 분석/적용 작업 시작·완료 시점에 갱신

---

## Phase 7 — 검증

### [ ] #17 Unity 에디터에서 수동 검증 (before/after 배치 수 비교)
- **BlockedBy**: #12, #13, #14, #15
- **통과 기준**: 고정 % 기준 없음. 반복 최적화 전제이므로 각 테스트 프리팹에서 Frame Debugger의 after 배치 수 ≤ before 배치 수만 확인 (회귀 없음)
- (1) 동일 Material 3개 이상 포함 프리팹에서 Frame Debugger의 before/after 배치 수를 기록, after가 before보다 크지 않은지 확인
- (2) CSV Excel에서 한글 정상 표시
- (3) Undo 복원 확인 (OiL/Fudge 값이 적용 전 상태로 돌아감)
- (4) 비활성 오브젝트 / Above-Below 구조 누락 프리팹에서 경고 pill 표시
- (5) 인터리브 케이스에서 ⚠ 아이콘 및 툴팁 동작
- (6) GPU Instancing OFF Material에서 ⓘ 아이콘 및 적용 대화상자 문구 확인
- (7) OiL 오버플로 유발 프리팹(Char Sorting Order = 32000, Material 그룹 다수)에서 적용 버튼이 차단되는지 확인
- (8) 프리팹 체크박스 해제 시 해당 프리팹이 적용 대상에서 제외되는지, 전체 배치 수 합산에서 빠지는지 확인
