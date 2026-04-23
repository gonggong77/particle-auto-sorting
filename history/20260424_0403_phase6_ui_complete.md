# Phase 6 UI Complete — #12, #13, #14, #15

**시각**: 2026-04-24 04:03
**브랜치**: main
**레포**: gonggong77/particle-auto-sorting

## 완료 항목

### #12 Renderer 테이블 UI (Above/Below 섹션)
- `ParticleAutoSortingWindow.Table.cs` 신규 파일로 분리
- 확장 시 Above/Below 섹션 분리 렌더링 (HierarchyOrder 오름차순)
- 컬럼: ⚠ 인터리브 아이콘 | Object | Material pill + ⓘ | OiL B/A | Fudge B/A
- 동일 Material = 동일 색상 pill (Material InstanceID 기반 HSV)
- ⓘ 툴팁 "GPU Instancing 비활성화"
- ⚠ 툴팁에 인터리브 상대 Material 이름 표시
- 수정 모드 ON → IntField/FloatField 편집 가능, 값이 `AfterAI`와 동일하면 override 해제(null)
- 수정 모드 OFF → 라벨만 표시
- override 값이 AI 예측값과 다르면 파란색(overrideLabelStyle) 강조
- 섹션 상단에 인터리브된 Material 쌍 목록 (중복 제거, 정렬된 키)

### #13 하단 버튼 영역
- `ParticleAutoSortingWindow.Bottom.cs` 신규 파일
- 전체 선택/해제 토글 (현재 상태가 전부 체크 → "전체 해제", 아니면 "전체 선택")
- `N / total개 선택` 표시
- 전체 배치 수 before → after (체크된 프리팹만 합산)
- 재계산 버튼 → `SortingOptimizer.Optimize` + `BatchCounter.CountPrefab` 재실행
- Undo 버튼 → `Undo.PerformUndo()`
- 적용 버튼 (bold) → `PrefabApplier.Apply(prefabs)`, 성공 시 BatchCounter 재계산 및 상태 메시지 갱신

### #14 CSV 내보내기 UI 패널
- 섹션 "보고서 내보내기 (CSV)" — helpBox로 감싼 vertical group
- 경로 TextField (편집 가능)
- 찾아보기 버튼 → `CsvReportExporter.ShowSavePanel()`
- 내보내기 버튼 → 경로 비어있으면 save panel 자동 호출 후 `CsvReportExporter.Export`
- 등록된 프리팹이 없으면 내보내기 disabled

### #15 상태 표시줄 최종화
- 메시지 상태: 대기 중 / 분석 완료: N개 등록 / 재계산 완료 / 적용 완료: N개 / 적용 취소됨 / CSV 내보내기 완료: <파일명> / Undo 실행 / 프리팹 제거됨
- 상태 전이 지점에서 `Repaint()` 호출
- 우측에 `등록된 프리팹: N` 표시

## 파일 구조

```
Assets/Editor/ParticleAutoSortingWindow.cs         (메인, 242줄)
Assets/Editor/ParticleAutoSortingWindow.List.cs    (엔트리/경고 pill)
Assets/Editor/ParticleAutoSortingWindow.Table.cs   (Renderer 테이블)
Assets/Editor/ParticleAutoSortingWindow.Bottom.cs  (하단 버튼 + CSV 패널)
```

partial class 4분할로 파일당 200줄 이하 유지.

## 부가 개선

- Char Sorting Order 변경 시 `RecomputeAll()` 자동 호출 → 즉시 after 값 반영
- `OilOverride`/`FudgeOverride`가 AI 예측값과 정확히 일치하면 자동으로 null 처리(=override 해제) → 수정 모드에서도 깔끔한 상태 유지

## 남은 작업

- [ ] #17 Unity 에디터 수동 검증 (사용자 진행)
