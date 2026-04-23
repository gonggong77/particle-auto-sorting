# Task #8 Complete — CsvReportExporter — 2026-04-24 04:00

## 전체 진행률
- 전체 17개 중 **9개 완료** (#16, #1, #2, #3, #4, #5, #6, #7, #8)
- Phase 5(보고서) 100% 완료 → Phase 6(UI) 본격 진입

## 이 세션에서 완료한 작업

### ✅ #8 CsvReportExporter.cs
- 경로: `D:\particleSortingTest\Assets\Editor\Report\CsvReportExporter.cs`
- 네임스페이스: `ParticleAutoSorting.Editor.Report`
- 공개 API:
  - `Export(IEnumerable<PrefabData>, string filePath) → bool`
  - `ShowSavePanel(string defaultName = "particle_auto_sorting_report.csv") → string`

### 구현 요점
- **UTF-8 BOM 인코딩**: `new UTF8Encoding(true)`로 Excel 한글 정상 표시 (Phase 7 검증 항목 2번 대응).
- **경로 선택 분리**: UI가 `ShowSavePanel`로 경로만 먼저 받을 수 있도록 Export와 별도 API 제공 (#14 UI가 "찾아보기" / "내보내기" 버튼을 분리해 사용).
- **12개 컬럼 헤더**: 프리팹명 / 오브젝트명 / Above/Below / Material / OiL before / OiL after / Fudge before / Fudge after / 프리팹 배치 수 before / 프리팹 배치 수 after / 전체 배치 수 before / 전체 배치 수 after.
- **after 값 우선순위**: `OilOverride ?? OilAfterAI`, `FudgeOverride ?? FudgeAfterAI` — 사용자 수동 편집값 우선 반영 (PrefabApplier와 동일 정책).
- **전체 배치 수**: 전달된 프리팹 리스트 전체 합산 (`BatchBefore`/`BatchAfter` 누적). UI가 "체크된 프리팹만 전달"하면 자연히 체크 기반 합산이 됨. CSV 호출 측(#14)의 책임 분리.
- **CSV 이스케이프**: `,`, `"`, `\n`, `\r` 중 하나라도 포함 시 전체 필드를 `"`로 감싸고 내부 `"`를 `""`로 치환.
- **Culture 고정**: 숫자는 `CultureInfo.InvariantCulture`로 포맷 — 한국 로케일에서 콤마 소수점 이슈 방지. Fudge는 `"0.###"` 포맷.
- **안전 장치**:
  - Renderer가 없는 프리팹도 한 줄 기록 (프리팹 배치 수 추적 가능).
  - 디렉토리 없으면 `Directory.CreateDirectory`로 생성.
  - `IOException` 캐치 → `EditorUtility.DisplayDialog`로 실패 안내 + `false` 반환.
  - 빈 리스트/null 입력 시 안내 다이얼로그 + `false` 반환.
- **줄바꿈**: `\r\n` (Windows/Excel 호환).

### 스펙 해석 / 결정
- Trail_Module 엔트리는 **CSV에 포함**. PrefabApplier에서는 물리적 쓰기를 스킵하지만, CSV는 "논리적 렌더러 엔트리별 리포트"이므로 AI가 산출한 값을 그대로 노출.
- "프리팹별 배치 수"와 "전체 배치 수"를 모든 Renderer 행에 반복 출력. Excel에서 필터링/피벗 시 편의성 우선. 플랜 컬럼 스펙과 일치.
- `ShowSavePanel` 래퍼는 UI 코드 중복 방지용. UI는 단순히 이 메서드를 호출하고 빈 문자열 체크만 하면 됨.

## 다음 작업 후보
- **#9 ParticleAutoSortingWindow 뼈대** — BlockedBy: #16 (해제) — UI 체인 시작점
- #9 → #10 → #11 → (#12, #13, #14) → #15 순으로 UI 전면 구현 가능

## 수정한 파일
- `D:\particleSortingTest\Assets\Editor\Report\CsvReportExporter.cs` (신규)
- `D:\particle-auto-sorting\particle-auto-sorting\TASKS.md` (#8 체크)
