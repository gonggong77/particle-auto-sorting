# Particle Auto Sorting — 프로젝트 컨텍스트

> 이 파일은 모든 Claude Code 세션에 자동 로드됩니다. 신규 세션이 즉시 작업할 수 있도록 핵심 컨텍스트만 담습니다.

## 프로젝트 정체

Unity Editor 확장 툴. 파티클 시스템의 **Order in Layer**와 **Sorting Fudge**를 자동 세팅하여 (1) 배치 수(Draw Call) 최소화, (2) Hierarchy 하단 우선 렌더링 순서 보장을 동시에 달성한다.

- 메뉴: `Tools > particle auto sorting`
- 진입 클래스: `ParticleAutoSorting.Editor.ParticleAutoSortingWindow` (EditorWindow, partial class)
- 언어/스택: C#, Unity IMGUI (EditorGUILayout/EditorGUI)
- 배포 형태: `.unitypackage` — `Assets/Editor/` 전체가 패키지 루트, 외부 의존 없음, asmdef 1개로 묶임
- Spec 단일 출처: `./particle_auto_sorting_spec.json` (Spec/Wireframe 단계 산출물, 변경 시 이 파일 우선 참고)
- Task 진행 상황: `./TASKS.md` (참고용, 1차 구현 완료 상태)

## 중요: 레포 두 개로 분리되어 있음

| 레포 | 경로 | git 관리 주체 | 내용 |
|---|---|---|---|
| **계획/이력 레포** (이 폴더) | `D:\particle-auto-sorting\particle-auto-sorting` | **Claude가 커밋·푸시** | spec, TASKS, history 로그 |
| **Unity 프로젝트** | `D:\particleSortingTest\` | **사용자가 직접 관리** | URP 템플릿, 실제 C# 소스 (`Assets/Editor/...`) |

**Claude는 Unity 프로젝트(`D:\particleSortingTest\`)에 git 명령을 실행하지 않는다.** 코드 수정만 하고, 사용자가 본인 워크플로로 커밋한다. 모든 Claude 커밋은 계획 레포에서만.

## 소스 코드 위치 (Unity 프로젝트)

```
D:\particleSortingTest\Assets\Editor\
├── ParticleAutoSorting.Editor.asmdef     (Editor 전용, autoReferenced)
├── ParticleAutoSortingWindow.cs          (메인: 진입점, 전역 컨트롤, 드래그앤드롭, 상태바)
├── ParticleAutoSortingWindow.List.cs     (프리팹 엔트리/경고 pill)
├── ParticleAutoSortingWindow.Table.cs    (Above/Below Renderer 테이블)
├── ParticleAutoSortingWindow.Bottom.cs   (하단 버튼 + CSV 패널)
├── Data/RendererInfo.cs                  (RendererKind enum 동일 파일)
├── Data/MaterialGroupInfo.cs
├── Data/PrefabData.cs
├── Analysis/PrefabAnalyzer.cs            (스캔: ParticleSystemRenderer + TrailRenderer + Trail Module)
├── Analysis/BatchCounter.cs              (BatchKey/SortKey 정렬 후 변경 횟수 카운트)
├── Optimization/SortingOptimizer.cs      (OiL/Fudge 할당 + 인터리브 감지 + 오버플로 플래그)
├── Apply/PrefabApplier.cs                (Undo 그룹 + PrefabUtility.SavePrefabAsset)
└── Report/CsvReportExporter.cs           (UTF-8 BOM)
```

partial class는 200~250줄 단위로 분할 유지.

## 도메인 핵심 규칙 (Spec 요약)

- **프리팹 구조 가정**: 루트 아래 `Above`/`Below` 자식이 있고, 각각의 자식들이 char(캐릭터) 위/아래 영역. 작업자가 의도적으로 만들어둔 구조.
- **렌더링 규칙**: Above/Below 각각의 자식들은 Hierarchy 하단일수록 앞에 그려져야 함.
- **Material 런(run)**: Hierarchy 순서(위→아래)로 정렬한 뒤, **연속된 동일 Material 구간**을 하나의 런으로 본다. 같은 Material 이라도 중간에 다른 Material 이 끼어들면 별도 런이다. OiL/Fudge 할당은 Material 전체가 아니라 이 **런 단위**로 이루어진다.
- **OiL 할당**: 각 런에 하나의 OiL. Above = `charSortingOrder + 1, +2, …, +N` / Below = `charSortingOrder - N, …, -1` (N = 해당 bucket 의 런 개수, Hierarchy 위→아래 순). **short 범위(-32768~32767) 클램프 금지**, 오버플로 시 `HasOverflow = true` + 적용 차단.
- **Fudge 할당**: 동일 런 내에서 `step = 30` 고정, `fudge[rank] = (size - 1 - rank) * step` (rank 0 = 런 최상단 = 뒤 = 최대 Fudge, 최하단 = 0). 예) size=4 → [90, 60, 30, 0].
- **BatchKey**: `[sortingLayerID, sharedMaterial, renderMode, mesh]`
- **SortKey**: `[sortingLayerID, orderInLayer, sortingFudge, hierarchyOrder]`
- **HierarchyOrder**: 같은 GameObject의 Particle/Trail 공존 시 Trail = Particle + 0.5 (Trail이 시각적으로 앞)
- **오브젝트 단위 불변식** (런 기반 할당으로 **항상 보장**):
  1. **OrderInLayer 단조성**: Hierarchy에서 인접한 두 오브젝트 A(위), B(아래)에 대해 `B.OrderInLayer >= A.OrderInLayer`. 같은 런이면 동일 OrderInLayer(등호), 런 경계면 엄격히 증가.
  2. **Fudge 단조성**: OrderInLayer가 같은(= 동일 런) 두 오브젝트 A(위), B(아래)에 대해 `B.sortingFudge < A.sortingFudge`. 런 최하단은 항상 0, 최상단은 `(size-1)*30`.
- **인터리브 감지**: 동일 Material 이 여러 런으로 쪼개지면(= Hierarchy 상 다른 Material 이 중간에 끼어든 경우) `HasInterleaveWarning` + 경고 pill. 불변식은 이미 보장되지만, **배치 수가 최적이 아님**을 사용자에게 알려 Hierarchy 재배치를 유도하는 목적.
- **null Material**: silent skip (알림 없음)
- **비활성 오브젝트**: 카운트 제외 + 경고 pill

## 워크플로 (현재 단계: 실사용 검증 + 버그 수정)

1. **사용자가 Unity 에디터에서 툴을 사용**, 문제/에러를 발견하면 메시지로 보고
2. Claude가 `D:\particleSortingTest\Assets\Editor\` 내 코드 수정 (직접 git 안 건드림)
3. **Task 단위 작업이 끝날 때마다** `history/YYYYMMDD_<슬러그>.md` 작성 (증상/원인/수정/검증 항목 명시)
4. 계획 레포에서 history 파일 add → commit → push (메시지 prefix: `fix:`, `feat:`, `docs:`, `refactor:` 등)
5. 사용자가 Unity에서 재현 확인

## 코드 수정 시 반드시 지킬 것

### IMGUI 함정 (이미 한 번 당했음)
- **`GUILayoutUtility.GetLastRect()`를 `BeginVertical`/`BeginHorizontal` 직후에 호출 금지** — `"You cannot call GetLast immediately after beginning a group"` 에러. 그룹 전체 rect가 필요하면 `var rect = EditorGUILayout.BeginVertical(...)` 반환값을 받아 두고 `EndVertical()` 후에 사용 + `Event.current.type == EventType.Repaint` 가드.
- `EditorGUI.DrawRect` 등 그리기 호출은 가능한 Repaint 이벤트로만 한정.

### 일반 원칙
- 외부 패키지 의존 금지 (asmdef 단일, `.unitypackage` 자립성).
- 한 파일 800줄 초과 금지 → partial class 분할.
- 새 데이터 필드는 `Data/` 하위에서만 추가, UI는 partial 파일 추가.
- `null` Material/Renderer는 silent skip (Spec 규칙 준수).
- `clamp` 같은 임의 보정 금지 — 오버플로/실패는 플래그 + 경고로 사용자에게 노출.
- 적용 동작은 항상 Undo 그룹으로 묶어 Ctrl+Z 한 번에 롤백 가능해야 함.

## 커밋/푸시 규칙

- 형식: `<type>: <description>` (`feat`, `fix`, `refactor`, `docs`, `test`, `chore`)
- 본문에 task 번호(`#7`, `#12` 등)가 의미 있을 때만 명시.
- Co-Authored-By 트레일러 비활성 (글로벌 settings에서 OFF).
- Claude는 사용자 명시 요청 없이 force push / 브랜치 삭제 / 히스토리 변경 금지.

## 응답 언어

항상 한국어로 응답한다 (글로벌 설정).

## 빠른 명령 모음

```bash
# 계획 레포 상태
cd /d/particle-auto-sorting/particle-auto-sorting
git status && git log --oneline -10

# Unity 코드 grep
grep -rn "패턴" /d/particleSortingTest/Assets/Editor/

# history 파일 작성 후 커밋
git add history/<file>.md && git commit -m "fix: ..." && git push
```
