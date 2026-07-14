# Changelog — 소화전 점검 지도 (119hyd-Map)

버전 규칙: `v주.부.수`
- **주(major)**: DB 마이그레이션(SQL 실행) 필요 또는 기존 방식과 호환이 깨질 때
- **부(minor)**: 기능 추가
- **수(patch)**: 버그 수정, 문구/스타일 변경

---

## v1.2.1 (2026-07-14)

### 개선
- **펼침 버튼 시인성 개선**: 이름 옆 작은 화살표 대신 카드 하단의 명확한 버튼으로 변경.
  주 계획: "▼ 세분계획 16개 펼치기" (파랑), 팀: "▼ 5개 조 펼치기" (초록).
  세분계획 존재 여부가 한눈에 인식됨. 생성 버튼 문구도 상태에 따라 "생성"/"갱신"으로 구분.

---

## v1.2.0 (2026-07-14)

### 기능 추가
- **계획 목록 트리 표시**: 홈 화면을 그리드에서 트리 구조로 변경.
  주 계획(📋) → 팀(🏢) → 조(👥) 순으로 들여쓰기 + 가이드선 표시.
  주 계획·팀 노드에 접기/펼치기(▸/▾) 토글, 세분계획 개수 뱃지 표시.
  펼침 상태는 localStorage에 기억 (기본: 주 계획 접힘).

---

## v1.1.2 (2026-07-14)

### 버그 수정
- **팀/조 세분계획 진입 시 처음에 전체가 보이던 문제**: 데이터 로드 직후 applyFilter()를
  호출하지 않아 초기 화면에 필터 잠금이 반영되지 않던 버그. 이제 진입 즉시 해당 팀/조만 표시.
- **조가 7개 이상일 때 마커 색상이 전부 빨강으로 나오던 문제**: CSS 클래스가 g1~g6까지만
  존재하는데 7번째 그룹부터 존재하지 않는 g7, g8...을 부여하던 버그 → 6색 순환으로 수정.
  색상표를 정렬된 그룹 순서로 배정해 같은 조는 항상 같은 색, 팀 내 조끼리는 색이 겹치지 않음.
  프로젝트 전환 시 색상표가 리셋되지 않던 문제도 함께 수정.

---

## v1.1.1 (2026-07-14)

### 버그 수정
- **엑셀 업로드 "가져올 행이 없습니다" 수정**: 새 excel.js(동적 컬럼 버전)가 매핑 결과를
  `mapping.name`이 아닌 내부 키(`mapping[col._id]`, 예: `col_0_name`)로 전달하도록 바뀐 것에 대응.
  구/신 excel.js 모두 호환되는 매핑 해석기(`_mi`) 추가. 이름 컬럼 미매핑 시 명확한 안내 표시.

---

## v1.1.0 (2026-07-13)

### ⚠ 이 버전부터 필요한 DB 변경
```sql
-- hydmap_projects: 세분계획용 컬럼 2개 (id 타입에 따라 bigint/uuid 자동 선택)
DO $$
DECLARE idtype text;
BEGIN
  SELECT data_type INTO idtype FROM information_schema.columns
  WHERE table_name='hydmap_projects' AND column_name='id';
  IF idtype='uuid' THEN
    EXECUTE 'ALTER TABLE hydmap_projects ADD COLUMN IF NOT EXISTS parent_id uuid REFERENCES hydmap_projects(id) ON DELETE CASCADE';
  ELSE
    EXECUTE 'ALTER TABLE hydmap_projects ADD COLUMN IF NOT EXISTS parent_id bigint REFERENCES hydmap_projects(id) ON DELETE CASCADE';
  END IF;
END $$;
ALTER TABLE hydmap_projects ADD COLUMN IF NOT EXISTS filter_group text;
NOTIFY pgrst, 'reload schema';
```
(참고: `hydmap_done/memo/photos/records.item_id → hydmap_items.id` FK는 기존에 이미 적용되어 있음을 확인함)

### 기능 추가
- **조별 세분계획**: 부모 계획의 팀/조 값 기준으로 세분계획 카드 자동 생성/갱신.
  세분계획은 자체 데이터 없는 필터 뷰 — 열면 부모 데이터가 해당 조 필터로 잠긴 상태로 표시.
  조별 진행률 카드 표시. 모든 조의 기록은 부모 계획 한 곳에 저장.
- **팀 단위 뷰(팀장용)**: 세분계획 생성 시 조 카드와 함께 팀 카드(🏢 "1팀 전체")도 자동 생성.
  팀 카드로 들어가면 팀 소속 소화전만 조별 색상으로 표시되고, 필터는 팀 내 조들로만 전환 가능.
  팀 카드 진행률은 소속 조 합산. → 서 전체(부모)/팀장(팀)/조원(조) 3계층 뷰
- **엑셀 팀/조 2컬럼 지원**: 팀·조 컬럼을 각각 매핑하면 `"1팀 2조"` 형태로 자동 결합
  (숫자만 있으면 팀/조 접미어 자동 부여)
- **동적 컬럼 → 정보카드 자동 표시**: 새 excel.js의 컬럼 추가 기능으로 만든 커스텀 컬럼이
  `extra`(jsonb)에 수집되어 정보카드에 자동 표시 (`storageKey:'hydmap'`으로 앱별 설정 분리)
- **버전 표시**: 홈 화면 하단 + 콘솔에 앱 버전 표시

### 성능 개선
- 데이터 로드 쿼리 5회 → 1회 (PostgREST FK 임베딩, 실패 시 자동 폴백)
- 정보카드 팝업 지연 렌더링 (열 때 한 번만 생성 — 초기 로딩 부담 대폭 감소)
- 상태 저장소 통합: doneMap/memoMap/photoMap/hydrantMap → `hyState` 단일 객체 + Proxy 뷰

### 안정성 개선
- localStorage 안전 래핑 (`safeStorage` — 사파리 프라이빗 모드/용량 초과 대응)
- Cloudinary 사진 업로드 지수 백오프 재시도 (최대 2회, 4xx는 즉시 실패, 재시도 토스트 표시)

### 버그 수정
- 정의되지 않은 `loadSheetJS()` 잔재 호출 제거 (매 로드마다 콘솔 에러 발생하던 문제)

---

## v1.0.0 (2026-07-05)

- 최초 안정 버전 (현장 실사용 시작)
- 계획 관리, 엑셀 업로드/컬럼 매핑(공통 ExcelUtil.createReader), 지오코딩,
  카카오맵 마커/클러스터/조별 색상, 정보카드(점검·사진·메모·로드뷰·위치수정),
  그룹 필터, 검색, 거리재기, 결과 엑셀 출력, 컬럼 재설정, 모바일 대응
- Kakao REST 키 제거 → 지도 SDK services 라이브러리(도메인 제한되는 JS 키)로
  역지오코딩·주소/장소 검색 이전
