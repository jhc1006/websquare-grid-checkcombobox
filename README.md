# WebSquare5 GridView 검색형 CheckComboBox & 줄바꿈 렌더링

WebSquare5의 GridView 내 특정 컬럼에서 **검색 기능이 있는 다중 체크 콤보(CheckComboBox)**를 호출하고, 선택된 항목들을 셀 내부에서 `<br/>` 태그를 통해 개행(줄바꿈)하여 표시하는 컴포넌트 구현체입니다.

---

## 📌 주요 특징 및 기능

1. **셀 내부 HTML 개행 렌더링 (`escape="false"`)**:
   - `escape="false"` 속성과 커스텀 포맷터(`scwin.fn_formatMultiCheck`)를 적용하여 콤마(`,`) 구분 코드값을 한글 라벨로 치환하고 `<br/>`로 개행 렌더링합니다.
   - `.cell-multiline` 클래스 및 `autoRowHeight="all"`을 통해 셀 높이가 내용에 맞춰 유연하게 자동 확장됩니다.

2. **반응형 플로팅 드롭다운 레이어**:
   - `grd_main.getCellDom(row, col)`과 `getBoundingClientRect()`를 활용하여 클릭한 그리드 셀 바로 아래에 정확히 도킹됩니다.
   - 화면 하단 넘침 방지(Auto-flip) 알고리즘이 적용되어 있습니다.

3. **실시간 검색 및 체크 상태 동기화**:
   - 드롭다운 상단의 인풋 박스에서 실시간으로 키워드를 입력해 원하는 항목을 필터링할 수 있습니다.
   - 검색 중에도 기존에 체크한 항목 상태가 유실되지 않고 안전하게 보존됩니다.

4. **단방향 데이터 동기화**:
   - [적용] 클릭 시 `dlt_sample.setCellData()`를 호출하여 WebSquare 데이터 바인딩 엔진을 통해 셀 포맷터를 자동 재실행합니다.

5. **다중 탭(TabControl) 환경 지원 및 레이어 자동 제어**:
   - `w2:tabControl` 기반 3개 탭(1. 담당 업무 배정, 2. 프로젝트 팀 현황, 3. 공통 코드 설정 및 가이드) 구성
   - 탭 전환 시(`tac_main_ontabclick`) 열려 있는 플로팅 체크콤보 레이어를 자동 닫음 처리하여 잔류 현상 방지
   - 복수의 GridView 및 DataList(`dlt_sample`, `dlt_team`)에서 동일한 CheckComboBox 컴포넌트를 유연하게 재사용 가능

6. **상단 공통 전달사항 & WebSquare API 기반 엑셀 다운로드 / 프리뷰**:
   - 전체 탭 상단에 공통 지침 및 공지사항을 입력할 수 있는 `xf:textarea` (`txa_notice`) 영역 배치
   - **WebSquare 공식 API (`WebSquare.util.multipleExcelDownload`) 연동**: 전달사항 내용(`infoArr`)과 모든 탭의 그리드 데이터를 `useFormat: true`(커스텀 포맷터 줄바꿈 및 한글 라벨 보존) 옵션으로 엑셀 내보내기
   - **플레이스홀더 필터링**: 전달사항 미입력 시 엑셀 및 프리뷰에서 전달사항 블록과 placeholder 텍스트가 자동 생략되며, 그리드 미선택 셀의 `(선택 없음)` placeholder도 엑셀 다운로드 시 깔끔한 빈 셀(`""`)로 출력
   - **통합 프리뷰 모달**: [프리뷰] 버튼 클릭 시 전달사항과 탭 1, 탭 2의 전체 데이터를 화면에 보이는 상태 그대로 팝업에 렌더링하며 즉시 인쇄(Print) 지원

7. **탭 데이터 변경 감지 및 WebSquare confirm 탭 이동 확인**:
   - `w2:tabControl`의 `ev:onbeforetabchange` 이벤트를 활용하여 사용자가 다른 탭으로 전환하기 전 현재 탭의 DataList(`dlt_sample`, `dlt_team`) 수정 여부를 자동 검사
   - 미저장 수정사항이 존재할 경우 **WebSquare 공식 대화상자(`$p.confirm`)**를 호출하여 이동 여부를 재확인하고, 취소 시 현재 탭에 안전하게 잔류
   - 브라우저 데모에서도 데이터 변경 시 탭에 수정 뱃지(`●`) 표시 및 WebSquare 스타일의 Confirm 모달 팝업 연동

8. **메인 그리드 행 추가 & 담당자 동적 Placeholder 및 엔터 시 담당자 조회**:
   - 탭 1 상단에 `[➕ 행 추가]` 버튼을 제공하여 클릭 시 신규 행(`seq` 자동 채번) 삽입 및 담당자 입력란 자동 포커스
   - **담당자 동적 Placeholder**: 평상시 빈 값일 때 `“클릭하여 입력”` ➔ 클릭/편집 포커스 진입 시 `“입력후 엔터”`로 실시간 전환
   - **엔터 시 담당자 조회 연동**: 담당자 입력 후 엔터(키코드 13) 입력 시 WebSquare 스타일의 **[담당자 조회 (직원 검색)]** 모달 팝업 오픈, 사원 목록 실시간 필터링 및 더블클릭/선택 버튼으로 그리드 셀 자동 반영

---

## 📂 파일 구조

* `grid_multicheck_combo.xml`: WebSquare5 표준 XML 컴포넌트 소스 (DataList, CSS, scwin 스크립트, GridView 마크업 포함)
* `demo_preview.html`: 브라우저에서 더블 클릭하여 즉시 테스트해볼 수 있는 인터랙티브 시뮬레이터 데모
* `SESSION_SUMMARY.md`: 개발 세션 질의응답(Q&A), 동적 탭 사전 로딩(Preload) 가이드, WebSquare confirm 연동 분석 등 총정리 문서

---

## 🚀 빠른 시작

### 1. 웹 브라우저에서 바로 확인하기
`demo_preview.html` 파일을 크롬(Chrome)이나 엣지(Edge) 브라우저로 열면 WebSquare 환경 없이도 동작을 직접 체험할 수 있습니다.

### 2. WebSquare Studio에 적용하기
`grid_multicheck_combo.xml` 파일을 WebSquare 프로젝트의 화면 디렉토리(예: `src/main/webapp/ui/...`)에 복사하여 스튜디오에서 열거나 WAS에 배포하여 확인합니다.
