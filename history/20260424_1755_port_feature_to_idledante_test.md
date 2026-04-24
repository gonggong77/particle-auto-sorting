# 20260424_1755 — particle-auto-sorting 기능을 idleDante_test 프로젝트로 이식

## 목적
`D:\particleSortingTest`에서 개발한 Editor 전용 particle-auto-sorting 기능을 `D:\idleDante_test` 프로젝트에서도 사용할 수 있도록 이식.

## 사전 점검
- 원본 소스(`D:\particleSortingTest\Assets\Editor\*`)의 외부 의존성 확인 — UnityEditor/UnityEngine 기본 API와 내부 네임스페이스(`ParticleAutoSorting.Editor.*`)만 사용하는 것을 확인. 외부 에셋/패키지 종속성 없음.
- asmdef(`ParticleAutoSorting.Editor.asmdef`)는 references 비어 있음, Editor 플랫폼 한정 → 타 프로젝트에 그대로 이식 가능.
- `idleDante_test`에는 기존 `Assets/Editor/` 폴더가 있음 (SpineSettings.asset, iOSBuildPostProcessor.cs). 충돌 방지를 위해 하위에 `ParticleAutoSorting/` 서브폴더로 분리 배치.

## 수행
`D:\idleDante_test\Assets\Editor\ParticleAutoSorting\` 생성 후, 아래 항목을 `.meta` 파일과 함께 복사(GUID 보존):

```
Analysis/              (BatchCounter.cs, PrefabAnalyzer.cs)
Apply/                 (PrefabApplier.cs)
Data/                  (MaterialGroupInfo.cs, PrefabData.cs, RendererInfo.cs)
Optimization/          (SortingOptimizer.cs)
Report/                (CsvReportExporter.cs)
ParticleAutoSorting.Editor.asmdef
ParticleAutoSortingWindow.cs
ParticleAutoSortingWindow.Bottom.cs
ParticleAutoSortingWindow.List.cs
ParticleAutoSortingWindow.Table.cs
```

## 결과
- `idleDante_test` Unity 에디터에서 MenuItem으로 ParticleAutoSortingWindow 접근 가능.
- 기존 `Assets/Editor/` 내 스크립트와 네임스페이스/asmdef가 분리돼 있어 충돌 없음.

## 다음 확인 사항
- Unity 에디터로 프로젝트 열어 컴파일 정상 확인.
- 메뉴 진입 및 프리팹 분석 기능 동작 검증.
