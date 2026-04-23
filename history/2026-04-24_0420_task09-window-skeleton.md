# Task #9 Complete — ParticleAutoSortingWindow 뼈대 — 2026-04-24 04:20

## 전체 진행률
- 전체 17개 중 **10개 완료** (#16, #1~#8, #9)
- Phase 6(UI) 진입, 다음 블록: #10 드래그앤드롭

## 이 세션에서 완료한 작업

### ✅ #9 ParticleAutoSortingWindow.cs
- 경로: `D:\particleSortingTest\Assets\Editor\ParticleAutoSortingWindow.cs`
- 네임스페이스: `ParticleAutoSorting.Editor`
- MenuItem: `Tools/particle auto sorting` (스펙 문구 그대로 소문자)
- `EditorWindow` 파생, `minSize = 640 x 480`

### 구현 요점
- **전역 상태 필드**: `[SerializeField] int charSortingOrder`, `[SerializeField] bool editMode` — 창 리로드 후에도 상태 보존.
- **Top bar (툴바 스타일)**:
  - "Char Sorting Order" IntField (기본 0, 범위 제한 없음 — 스펙 준수)
  - 수정 모드 토글 (기본 OFF, 라벨이 `수정 모드: ON/OFF`로 현재 상태 표시)
- **레이아웃 구역 분할** (스펙의 6단 레이아웃 뼈대로 #10~#15 채울 자리 확보):
  1. 드롭존 플레이스홀더 (#10)
  2. 프리팹 리스트 스크롤뷰 (#11 — 현재는 빈 상태 안내)
  3. 하단 버튼 영역 (#13)
  4. CSV 패널 (#14)
  5. 하단 상태 표시줄 (#15 — `statusMessage` + 프리팹 수)
- `List<PrefabData> prefabs` 필드 선언만 해두고 이후 태스크가 채움. 네임스페이스 `ParticleAutoSorting.Editor.Data` using 추가.
- 스크롤 뷰 `Vector2 scroll` 상태 유지.

### 스펙 해석 / 결정
- **플레이스홀더는 레이블 텍스트로 명시** ("#10에서 구현" 등)해 미완 상태가 드러나도록 함 — 수동 검증 시 진행 상태 파악 편의.
- `SerializeField` 사용: `EditorWindow`는 도메인 리로드 때 상태가 리셋되므로 직렬화 가능한 기본 전역 상태만 속성으로 노출. `prefabs`는 `PrefabData`가 `GameObject` 참조를 가진 런타임 편집 상태라 직렬화 대상에서 제외 (재등록 필요, 스펙상 세션 상태 보존 요구 없음).
- `Show()` + `GetWindow` 조합으로 도킹 가능.
- `toolbar` 스타일과 `helpBox` 상태바로 Unity 기본 에디터 톤에 맞춤 — 별도 커스텀 스타일 추가 없이 구현 최소화.

## 다음 작업 후보
- **#10 드래그앤드롭 + 프리팹 등록 UI** — BlockedBy: #9 (해제), #4 (완료)
- #10 완료 후 #11 → (#12, #13, #14) → #15 순 진행

## 수정한 파일
- `D:\particleSortingTest\Assets\Editor\ParticleAutoSortingWindow.cs` (신규)
- `D:\particle-auto-sorting\particle-auto-sorting\TASKS.md` (#9 체크)
