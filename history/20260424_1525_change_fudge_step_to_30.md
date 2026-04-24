# change: Fudge 간격 30 고정으로 변경

- 날짜: 2026-04-24 15:25
- 구분: feat / spec 변경 (SortingOptimizer)
- 대상 파일:
  - `D:\particleSortingTest\Assets\Editor\Optimization\SortingOptimizer.cs`
  - `CLAUDE.md` (도메인 핵심 규칙 갱신)
  - `particle_auto_sorting_spec.json` (open_questions 결정 표기)

## 요청

사용자 지시: "fudge 설정 간격은 30 으로 해줘"

## 기존 정책

```csharp
private const float MaxStep = 0.1f;
private const float MaxSpan = 0.9f;
float step = size <= 1 ? 0f : Mathf.Min(MaxStep, MaxSpan / (size - 1));
```

- 동일 Material 그룹 내에서 step = min(0.1, 0.9 / (size - 1))
- 최대 span 0.9 내부로 제한, 스텝당 최대 0.1

## 변경 후

```csharp
private const float FudgeStep = 30f;
float step = size <= 1 ? 0f : FudgeStep;
```

- 그룹 크기와 무관하게 고정 간격 30
- 할당식은 동일: `fudge[rank] = (size - 1 - rank) * step`
- 예) size=4 → [90, 60, 30, 0]

`MaxStep`, `MaxSpan` 상수는 제거.

## 파급 영향

- 동일 Material 그룹이 여러 렌더러를 갖는 경우, Fudge 절댓값이 훨씬 커짐(이전 0~0.9 → 이후 0~(size-1)*30).
- OrderInLayer 간 경계(1.0)를 Fudge가 넘어갈 수 있음. Unity sortingFudge 는 float 이므로 수치적 제한은 없으나, 동일 OrderInLayer 안에서의 세부 정렬이라는 역할은 유지.
- 인터리브 감지(`DetectInterleave`)는 `OilAfterAI + FudgeAfterAI`로 정렬 후 인덱스 기반 판정이므로 스텝 값 변경에 영향 없음.
- CSV 보고서의 Fudge before/after 수치만 변경됨.

## 문서 갱신

- `CLAUDE.md`: "도메인 핵심 규칙 > Fudge 할당" 항목 갱신. 예시값 포함.
- `particle_auto_sorting_spec.json`: `open_questions`의 "Sorting Fudge 조정 값 단위 (0.1 간격 권장)" → "30 고정 간격, 2026-04-24 결정".

## 검증 체크리스트

- [ ] 테이블 Fudge AI 컬럼이 (size-1)*30, …, 60, 30, 0 으로 표기
- [ ] size=1 그룹은 Fudge=0
- [ ] Apply 후 Unity Inspector의 Sorting Fudge 값이 정확히 그대로 반영
- [ ] 배치 수 before/after 감소 유지 (간격 변경과 무관)
- [ ] Undo 한 번으로 롤백 가능
