# 📋 WebSquare5 Grid CheckComboBox 세션 개발 및 Q&A 정리

본 문서는 이번 개발 세션에서 진행된 **요구사항 구현 내역, WebSquare5 특화 기술 질의응답(Q&A), 문제 해결 방식 및 설계 가이드**를 종합 정리한 결과물입니다.

---

## 📑 목차
1. [세션 개발 히스토리 요약](#1-세션-개발-히스토리-요약)
2. [세션 질의응답(Q&A) 및 구현 상세](#2-세션-질의응답qa-및-구현-상세)
   - [Q1. 3개 탭 구성 및 TabControl 제어](#q1-3개-탭-구성-및-tabcontrol-제어)
   - [Q2. 상단 공통 전달사항 & WebSquare API 엑셀 다운로드 및 프리뷰](#q2-상단-공통-전달사항--websquare-api-엑셀-다운로드-및-프리뷰)
   - [Q3. 엑셀 다운로드 및 프리뷰 시 플레이스홀더(Placeholder) 제외](#q3-엑셀-다운로드-및-프리뷰-시-플레이스홀더placeholder-제외)
   - [Q4. 동적 탭 생성 시 메인페이지 로딩 시점에 모든 탭 페이지 사전 로딩(Preload) 기법](#q4-동적-탭-생성-시-메인페이지-로딩-시점에-모든-탭-페이지-사전-로딩preload-기법)
   - [Q5. 탭 데이터 변경 시 WebSquare confirm($p.confirm) 이동 확인창](#q5-탭-데이터-변경-시-websquare-confirmpconfirm-이동-확인창)
3. [핵심 아키텍처 및 소스 코드 가이드](#3-핵심-아키텍처-및-소스-코드-가이드)
4. [산출물 및 검증 환경](#4-산출물-및-검증-환경)

---

## 1. 세션 개발 히스토리 요약

| 순번 | 사용자 요청 내용 | 구현 및 기술 솔루션 | 반영 파일 |
|:---:|:---|:---|:---|
| **1** | 탭을 3개로 만들어줘 | `w2:tabControl` 기반 3개 탭 구성(담당업무배정, 프로젝트팀현황, 코드설정/가이드) 및 탭 전환 시 플로팅 레이어 잔류 방지 핸들러 구현 | `grid_multicheck_combo.xml`<br>`demo_preview.html` |
| **2** | 전달사항 textarea 배치 및 엑셀 다운로드 / 프리뷰 버튼 추가 | 전체 탭 상단 `xf:textarea` 배치, WebSquare 공식 API(`WebSquare.util.multipleExcelDownload`) 연동 및 전체 탭 통합 프리뷰 모달 팝업 구현 | `grid_multicheck_combo.xml`<br>`demo_preview.html` |
| **3** | 다운로드에서 placeholder 안 나오게 가능? | 전달사항 미입력 시 상단 `infoArr` 생략, 그리드 미선택 시 화면엔 `(선택 없음)` 표시하되 엑셀 출력 시에는 빈 문자열(`""`)로 필터링하는 플래그(`scwin._isExcelDownloading`) 구현 | `grid_multicheck_combo.xml`<br>`demo_preview.html` |
| **4** | 탭이 동적 생성될 때 메인 로딩 시 모든 탭 페이지를 사전 로딩하는 방법은? | WebSquare5의 탭 지연 렌더링(Lazy Loading) 극복 가이드 제공 (`alwaysDraw="true"`, `addTab` 옵션, WFrame 순차 비동기 로딩 큐, 데이터 선조회 아키텍처) | 기술 자문 문서화 |
| **5** | 탭에 변경 내용 있을 때 탭 이동 확인창을 WebSquare confirm으로 묻기 | `ev:onbeforetabchange` 이벤트 가로채기, DataList `getModifiedIndex()` 수정 감지, 비동기 `$p.confirm` 연동 및 `_allowTabChange` 플래그를 통한 안전 탭 전환 구현 | `grid_multicheck_combo.xml`<br>`demo_preview.html` |
| **6** | 메인 그리드 행추가 버튼 추가, 담당자 text type placeholder “클릭하여 입력” 클릭후 편집모드 “입력후 엔터” 엔터하면 담당자 조회 | 메인 그리드 상단 `[➕ 행 추가]` 버튼(`dlt_sample.insertRow()`), `userName` 컬럼의 포커스 상태별 동적 placeholder(`"클릭하여 입력"` ↔ `"입력후 엔터"`), 엔터 키 이벤트(`oneditkeydown`) 시 WebSquare 스타일 [담당자 조회] 모달 팝업 연동 | `grid_multicheck_combo.xml`<br>`demo_preview.html` |
| **7** | 탭을 3개의 페이지에서 불러와 동적으로 생성하게 작성해줘 | 3개 탭을 독립 WFrame XML(`tab1_task_assign.xml`, `tab2_team_status.xml`, `tab3_guide.xml`)로 분리하고, 메인 화면 로딩 시(`scwin.onpageload`) `tac_main.addTab()` API로 사전 로딩(`alwaysDraw: true`) 및 동적 생성 구현 | `grid_multicheck_combo.xml`<br>`tab1_task_assign.xml`<br>`tab2_team_status.xml`<br>`tab3_guide.xml`<br>`demo_preview.html` |

---

## 2. 세션 질의응답(Q&A) 및 구현 상세

### Q1. 3개 탭 구성 및 TabControl 제어
* **요구사항**: 단일 화면에 머물던 그리드를 3개 탭 구조로 확장하고, 탭마다 독립된 그리드 및 가이드 영역 제공.
* **구현 내용**:
  - **탭 1 (담당 업무 배정)**: 기존 메인 그리드(`grd_main`) 및 다중 체크콤보 연동
  - **탭 2 (프로젝트 팀 현황)**: 보조 그리드(`grd_team`, `dlt_team`)를 배치하여 동일한 멀티체크 콤보 레이어를 컴포넌트 단위로 재사용
  - **탭 3 (코드 설정 및 가이드)**: 공통 코드 매핑 테이블 및 사용 지침 표시
  - **레이어 잔류 방지**: 탭 클릭 이벤트(`tac_main_ontabclick`) 발생 시 열려 있는 플로팅 체크콤보 드롭다운을 즉시 닫아 탭 간 UI 충돌 방지

---

### Q2. 상단 공통 전달사항 & WebSquare API 엑셀 다운로드 및 프리뷰
* **요구사항**: 탭 상단에 공통 전달사항 입력란을 두고, 엑셀 다운로드 및 팝업 프리뷰 시 전달사항과 모든 탭의 그리드 데이터를 화면 표시 그대로 출력.
* **WebSquare 엑셀 다운로드 API 적용**:
  - WebSquare5의 다중 그리드 내보내기 표준 함수인 `WebSquare.util.multipleExcelDownload` 활용
  - 옵션 구성:
    ```javascript
    var options = {
        common: {
            fileName: "전체업무현황_" + $p.getCurrentServerDate("yyyyMMdd") + ".xlsx",
            showProcess: true,
            infoArr: [
                { rowIndex: 0, colIndex: 0, rowSpan: 1, colSpan: 6, text: "■ 공통 전달사항: " + noticeText }
            ]
        },
        excelFiles: [
            { gridId: "grd_main", sheetName: "담당업무배정", useFormat: true },
            { gridId: "grd_team", sheetName: "프로젝트팀현황", useFormat: true }
        ]
    };
    WebSquare.util.multipleExcelDownload(options);
    ```
  - `useFormat: true` 속성을 지정하여 `<br/>` 줄바꿈 및 한글 라벨 치환 포맷터(`scwin.fn_formatMultiCheck`)가 엑셀 셀 내에 그대로 반영되도록 설정.
* **전체 탭 통합 프리뷰 팝업**:
  - [👁️ 데이터 전체 프리뷰] 클릭 시 상단 전달사항과 탭 1, 탭 2의 전체 그리드 테이블을 하나의 프리뷰 모달에 렌더링.
  - 모달 상단에 [🖨️ 인쇄] 버튼을 배치하여 즉시 A4/보고서 출력 지원.

---

### Q3. 엑셀 다운로드 및 프리뷰 시 플레이스홀더(Placeholder) 제외
* **요구사항**: 화면에는 안내용 플레이스홀더 텍스트가 표시되더라도, 엑셀 다운로드 및 프리뷰 시에는 플레이스홀더 텍스트가 출력되지 않도록 처리.
* **해결 방안**:
  1. **전달사항 Textarea**:
     - 사용자가 전달사항을 입력하지 않았거나 초기 placeholder 상태일 경우, 엑셀 상단 `infoArr` 생성을 완전히 제외하여 상단 dummy 행을 제거.
     - 프리뷰 팝업에서도 전달사항 영역을 자동 숨김(`display: none;`) 처리.
  2. **그리드 미선택 셀 `(선택 없음)` 처리**:
     - 화면에서는 사용자 편의를 위해 회색 이탤릭 `(선택 없음)` 라벨 유지.
     - 엑셀 다운로드 트리거 시 `scwin._isExcelDownloading = true` 플래그 설정 후 그리드 포맷터에서 미선택 데이터를 빈 문자열(`""`)로 반환하도록 분기 처리.

---

### Q4. 동적 탭 생성 시 메인페이지 로딩 시점에 모든 탭 페이지 사전 로딩(Preload) 기법
* **질문 배경**: WebSquare의 `w2:tabControl`은 기본적으로 **지연 로딩(Lazy Loading / Draw on Demand)** 방식으로 작동합니다. 즉, 사용자가 해당 탭을 처음 클릭하는 시점에 비로소 내부 DOM 또는 WFrame의 XML 파일을 서버에서 가져와 렌더링합니다. 따라서 메인 페이지 로딩 시 모든 탭 페이지를 미리 로딩해두려면 별도의 설정 및 스크립트 처리가 필요합니다.

#### 💡 해결 솔루션 (WebSquare5 권장 패턴 4가지)

#### 방법 1. XML 속성: `alwaysDraw="true"` 설정 (정적/초기 탭인 경우)
* `<w2:tabControl>` 컴포넌트에 `alwaysDraw="true"`를 지정하면, 활성화되지 않은 탭이라도 페이지 로드 시점에 모든 탭의 DOM을 백그라운드에서 즉시 렌더링합니다.
```xml
<w2:tabControl id="tac_main" alwaysDraw="true" style="width: 100%; height: 500px;">
    <w2:tabs id="tabs1">
        <w2:tab id="tab1" label="탭 1" />
        <w2:tab id="tab2" label="탭 2" />
        <w2:tab id="tab3" label="탭 3" />
    </w2:tabs>
    ...
</w2:tabControl>
```

#### 방법 2. 동적 탭 추가 API 호출 시 `alwaysDraw: true` 옵션 전달
* `tac_main.addTab()` API를 통해 동적으로 탭을 붙일 때 옵션 파라미터로 `alwaysDraw: true`를 지정합니다.
```javascript
// 동적으로 WFrame 기반 탭을 추가하면서 즉시 로딩하도록 설정
tac_main.addTab(
    "tab_dynamic_01",
    {
        label: "동적 탭 1",
        closable: true,
        openAction: "select",  // 추가 후 선택 여부
        alwaysDraw: true       // 핵심: 비활성 상태라도 즉시 백그라운드 렌더링
    },
    {
        src: "/ui/sample/tab_content_01.xml",
        wframe: true
    }
);
```

#### 방법 3. 스크립트를 통한 백그라운드 순차 사전 로딩 큐 (Preload Queue)
* 모든 탭을 한꺼번에 동시 로딩하면 초기 렌더링 지연(화면 멈춤) 현상이 발생할 수 있으므로, 메인 탭 1이 화면에 먼저 뜬 후 `setTimeout`을 통해 나머지 탭을 백그라운드에서 순차 로딩하는 기법입니다.
```javascript
scwin.onpageload = function() {
    // 1. 첫 번째 탭 동적 생성 및 즉시 활성화
    scwin.fn_createMainTab();

    // 2. 메인 렌더링 완료 후 나머지 탭들을 백그라운드에서 순차 프리로드
    setTimeout(function() {
        scwin.fn_preloadRemainingTabs();
    }, 100);
};

scwin.fn_preloadRemainingTabs = function() {
    var tabList = [
        { id: "tab2", label: "프로젝트 팀 현황", src: "/ui/sub/team_status.xml" },
        { id: "tab3", label: "공통 가이드", src: "/ui/sub/guide.xml" }
    ];

    tabList.forEach(function(item) {
        tac_main.addTab(
            item.id,
            { label: item.label, closable: false, alwaysDraw: true },
            { src: item.src, wframe: true }
        );
    });
};
```

#### 방법 4. DataCollection 사전 패치(Prefetch) 분리
* 탭 페이지의 렌더링 지연은 주로 각 탭 내부 WFrame에서 발생하는 `submission`(데이터 조회) 통신 때문입니다.
* 메인 화면의 부모 컨텍스트(`scwin`)에서 전역 `submission`을 통해 모든 탭에 필요한 데이터를 메인 로딩 시 일괄 조회한 뒤, 탭 생성 시 `setJSON()`으로 주입하면 네트워크 왕복 시간을 0으로 단축할 수 있습니다.

---

### Q5. 탭 데이터 변경 시 WebSquare confirm($p.confirm) 이동 확인창
* **요구사항**: 탭 내부 그리드 데이터에 수정 사항이 존재할 때 다른 탭으로 이동하려고 하면, 브라우저 기본 `confirm()`이 아닌 **WebSquare 공식 대화상자(`$p.confirm`)**를 띄워 변경 내용 유실을 경고하고 이동 여부를 확인.
* **핵심 기술 과제 및 해결**:
  1. **비동기 UI와 동기 이벤트 간의 충돌 해결**:
     - WebSquare5의 `onbeforetabchange(currentTabIndex, newTabIndex)` 이벤트는 함수 실행 즉시 `true`나 `false`를 반환(동기식)해야 합니다.
     - 하지만 `$p.confirm(message, callback)`은 콜백 방식으로 동작하는 비동기 레이어 팝업입니다.
     - **해결 패턴 (`_allowTabChange` 플래그)**:
       - 수정 내역 감지 시 일단 `return false;`를 반환하여 탭 전환을 일시 정지시킵니다.
       - confirm 팝업에서 사용자가 [확인]을 누르면, `scwin._allowTabChange = true` 플래그를 세우고 `tac_main.setSelectedTabIndex(newTabIndex)`를 호출합니다.
       - 다시 호출된 `onbeforetabchange`에서 플래그를 검사하여 `true`를 반환함으로써 무한 루프 없이 부드럽게 탭 전환을 완료합니다.
  2. **DataList 수정 감지 API (`getModifiedIndex`)**:
     - `dlt_sample.getModifiedIndex().length > 0` 또는 `dlt_sample.isModified()`를 호출하여 실제 데이터가 변경되었는지를 정밀하게 감별합니다.

---

### Q6. 메인 그리드 행 추가 및 담당자 동적 Placeholder & 엔터 시 담당자 조회 연동
* **요구사항**:
  - 메인 그리드 상단에 `[➕ 행 추가]` 버튼 추가
  - 담당자(`userName`) 컬럼을 `text` 타입으로 구성
  - 평상시 빈 값: `“클릭하여 입력”` placeholder 표시
  - 클릭 후 편집 모드(포커스): `“입력후 엔터”` placeholder로 동적 전환
  - 입력 후 엔터(키코드 13) 입력 시: WebSquare 스타일 **[담당자 조회 (직원 검색)]** 팝업 모달을 호출하고 선택 사원을 그리드에 자동 바인딩
* **구현 및 기술 솔루션**:
  1. **행 추가 (`dlt_sample.insertRow()`)**:
     - `seq`는 전체 행 수 기반 자동 채번, 기본값(`userName: ""`, `taskCodes: ""`, `remark: "신규 행"`) 설정
     - `grd_main.setFocusedCell(newRowIdx, "userName", true)`로 신규 행 담당자 셀에 즉시 포커스 유도
  2. **동적 Placeholder 전환 기법**:
     - `userName` 컬럼에 `editModeEvent="onclick"`, `customFormatter="scwin.fn_formatUserName"` 적용
     - 포커스 진입 시(`ev:oncellclick` / `handleUserFocus`): input 요소의 `placeholder`를 `"입력후 엔터"`로 동적 치환
     - 포커스 아웃 시(`ev:oneditblur` / `handleUserBlur`): 빈 값일 경우 다시 `"클릭하여 입력"`으로 복원
  3. **엔터 키 기반 담당자 조회 팝업 (`oneditkeydown`)**:
     - `keyCode === 13` 감지 시 사용자가 입력 중인 검색어를 추출하여 `scwin.fn_openManagerSearch(row, keyword)` 실행
     - WebSquare 블루 톤앤매너의 직원 검색 모달을 띄우고 사원명, 사번, 부서명 실시간 필터링
     - 사원 더블클릭 또는 `[선택]` 버튼 클릭 시 `dlt_sample.setCellData(targetRow, "userName", empName + " " + empTitle)` 반영 후 모달 닫기

---

### Q7. 3개 탭의 독립 WFrame 분리 및 메인 로딩 시 동적 `addTab()` 사전 로딩
* **요구사항**: 탭 3개의 화면을 개별 WFrame XML 파일로 분리하고, 메인 페이지 로딩 시 자바스크립트(`tac_main.addTab()`)를 통해 동적으로 로드 및 사전 렌더링.
* **구현 솔루션**:
  1. **개별 WFrame XML 모듈화**:
     - `tab1_task_assign.xml`: 담당 업무 배정, 행 추가, 담당자 검색, 메인 그리드 및 DataList(`dlt_sample`)
     - `tab2_team_status.xml`: 프로젝트 팀 현황 보조 그리드 및 DataList(`dlt_team`)
     - `tab3_guide.xml`: 공통 업무 코드 카드 및 WFrame 안내 가이드
  2. **`tac_main.addTab()` 동적 로딩 & `alwaysDraw: true` 사전 렌더링**:
     - `scwin.onpageload` ➔ `scwin.fn_initDynamicTabs()`에서 3개 탭을 순차적으로 `addTab()` 등록
     - `{ alwaysDraw: true }` 옵션 적용으로 탭을 클릭하기 전에 이미 백그라운드에서 모든 WFrame DOM과 DataList가 로드되어 첫 탭 클릭 시 딜레이가 전혀 발생하지 않음
  3. **스코프 연동 (`tac_main.getFrame`)**:
     - 부모 메인 화면에서 `tac_main.getFrame(idx).getWindow()`를 통해 각 WFrame 내부의 DataList 수정 여부 검사(`fn_isTabModified`) 및 엑셀 다운로드, 프리뷰 데이터 수집을 완벽히 연동

---

### Q8. 탭을 안 열고 다른 탭에 있는 데이터를 엑셀 다운로드나 일괄 저장(submission) 시 모두 포함되도록 보장하는 방법
* **핵심 질문 배경**:
  - WebSquare5의 `w2:tabControl`은 기본적으로 사용자가 탭을 클릭하는 시점에 비로소 화면을 로드하는 **지연 로딩(Lazy Loading / Draw on Demand)** 방식을 사용합니다.
  - 이로 인해 사용자가 탭 1에서만 작업하고 탭 2를 클릭하지 않은 상태에서 `[엑셀 다운로드]`나 `[전체 저장(submission)]`을 누르면, 탭 2의 WFrame XML이 아직 로드되지 않아 **데이터가 누락되거나 null 에러가 발생하는 치명적인 문제**가 발생합니다.
* **해결 및 보장 방법 (3가지 핵심 아키텍처)**:
  1. **[방법 1] `alwaysDraw: true` 사전 로딩 옵션 (현재 프로젝트 적용 방식)**:
     - `tac_main.addTab()` 동적 호출 시 `{ alwaysDraw: true }`를 전달하거나 `<w2:tabControl alwaysDraw="true">`를 선언합니다.
     - 사용자가 탭 2를 클릭하지 않았더라도 메인 화면 로딩 시점에 모든 탭의 WFrame XML과 DataList가 백그라운드에 즉시 로드됩니다.
     - 따라서 `tac_main.getFrame(1).getWindow().dlt_team.getAllJSON()`으로 미오픈 탭의 데이터도 100% 안전하게 추출할 수 있습니다.
  2. **[방법 2] 부모 화면의 전역 DataCollection 관리 (엔터프라이즈 권장 패턴)**:
     - 탭 내부 WFrame마다 개별 DataList를 두지 않고, 메인 부모 화면(`grid_multicheck_combo.xml`)의 `<w2:dataCollection>`에 `dlt_sample`, `dlt_team`을 전역 선언합니다.
     - WFrame 화면에서는 부모의 DataList를 바인딩(`dataList="data:parent.dlt_team"`)하거나 포인터로 참조합니다.
     - 이 방식은 탭이 열렸든 안 열렸든 상관없이 모든 데이터가 이미 부모 화면에 완벽하게 존재하므로, 일괄 submission이나 엑셀 다운로드 시 데이터 유실이 원천 차단됩니다.
  3. **[방법 3] 안전 데이터 추출 공통 헬퍼 (`scwin.fn_getAllTabData`) 활용**:
     - 엑셀 다운로드나 submission 직전에 `tac_main.getFrame(idx)`의 WFrame window 존재 여부를 안전하게 검사하여 데이터를 일괄 취합하는 헬퍼 함수를 구축합니다.

---

### Q9. WebSquare5에서 `alwaysDraw`가 동작하지 않는 대표적인 원인 및 해결책
* **1. `addTab()` API 파라미터 전달 위치 불일치 (`tabOptions` vs `contentOptions`)**:
  - WebSquare5의 `tac_main.addTab(tabId, tabOptions, contentOptions)`는 2번째 인자(`tabOptions`)와 3번째 인자(`contentOptions`)를 가집니다.
  - WebSquare SP 버전(SP3, SP4, SP5)에 따라 `alwaysDraw`를 탭 레벨 옵션으로 인식하는 엔진과 WFrame/컨텐츠 레벨 옵션으로 인식하는 엔진이 상이합니다.
  - **해결책**: `addTab` 호출 시 2번째와 3번째 인자 양쪽에 모두 `alwaysDraw: true`를 선언하여 엔진 버전 간 호환성을 100% 보장합니다.
* **2. WFrame 비동기(Asynchronous) 로딩 시점 차이 (Timing Issue)**:
  - `alwaysDraw: true`를 주더라도 WFrame XML 파일을 서버에서 HTTP로 다운로드하고 DOM을 파싱/생성하는 과정은 **비동기**로 처리됩니다.
  - 메인 화면의 `scwin.onpageload`에서 `tac_main.addTab()`을 호출한 직후, 같은 동기 실행 블록 내에서 바로 `tac_main.getFrame(1).getWindow().dlt_team`에 접근하면 아직 XML 파싱이 끝나지 않아 `getWindow()`가 `null`이거나 객체가 비어 있습니다.
  - **해결책**: 탭 WFrame 로딩 완료 이벤트(`onwframeload`)를 수신하거나, `setTimeout` 또는 비동기 큐를 통해 WFrame이 완전히 준비된 시점에 데이터에 접근해야 합니다.
* **3. 정적 XML 선언 시 `<w2:content>` 태그 속성 누락**:
  - `<w2:tabControl alwaysDraw="true">`만 선언하고 내부의 개별 탭 컨텐츠인 `<w2:content alwaysDraw="true">` 속성을 생략하면, 하위 엔진에서 컨텐츠 레벨 지연 로딩이 기본 적용되어 WFrame이 로드되지 않을 수 있습니다.
* **4. WFrame 내부의 `submission` 비동기 조회 미완료**:
  - `alwaysDraw: true`로 화면(DOM)과 컴포넌트는 그려졌더라도, WFrame의 `scwin.onpageload`에서 데이터를 가져오는 서버 `submission` 통신이 완료되지 않았을 수 있습니다. 화면 컴포넌트 렌더링 완료와 데이터 로딩 완료를 구분해야 합니다.
* **5. `websquare.xml` 엔진 전역 설정 및 프로젝트 공통 래퍼에 의한 오버라이드**:
  - 시스템 관리자의 `websquare.xml` 설정 파일에 `<tabControl><lazyDraw value="true"/></tabControl>` 또는 `<alwaysDraw value="false"/>`가 전역으로 강제되어 있으면 개별 화면 속성이 무시될 수 있습니다.
  - 또한 프로젝트 공통 유틸(예: `com.addTab(...)`)이 내부적으로 `alwaysDraw` 옵션을 전달하지 않고 필터링하는 경우도 빈번합니다.
* **6. WFrame 스코프 격리(`scope="true"`)로 인한 잘못된 접근**:
  - WFrame은 독립 스코프로 격리되므로 전역 `window.dlt_team`으로 접근하면 `undefined`가 발생합니다. 반드시 `tac_main.getFrame(tabIndex).getWindow()` 또는 `getObj("dlt_team")`으로 접근해야 합니다.

---

## 3. 핵심 아키텍처 및 소스 코드 가이드

### 미오픈 탭 데이터 안전 일괄 추출 헬퍼 (`grid_multicheck_combo.xml`)

```javascript
/**
 * [미오픈 탭 안전 데이터 수집 헬퍼]
 * 사용자가 특정 탭을 한 번도 클릭하지 않았더라도 alwaysDraw: true 덕분에 각 WFrame의 DataList에서
 * 전체 데이터(엑셀/프리뷰) 또는 수정된 데이터(일괄 저장 submission)를 안전하게 일괄 추출합니다.
 * @param {Boolean} onlyModified - true: getModifiedJSON() (저장용), false: getAllJSON() (조회/엑셀용)
 * @returns {Object} { notice: string, tab1Data: Array, tab2Data: Array }
 */
scwin.fn_getAllTabData = function(onlyModified) {
    var result = {
        notice: (txa_notice.getValue() || "").trim(),
        tab1Data: [],
        tab2Data: []
    };

    // 탭 1 (tab1_task_assign.xml) WFrame 데이터 추출
    var frame1 = tac_main.getFrame(0);
    var win1 = frame1 ? ((typeof frame1.getWindow === "function") ? frame1.getWindow() : frame1) : null;
    if (win1 && win1.dlt_sample) {
        result.tab1Data = onlyModified ? win1.dlt_sample.getModifiedJSON() : win1.dlt_sample.getAllJSON();
    }

    // 탭 2 (tab2_team_status.xml) WFrame 데이터 추출 (탭 미오픈 시에도 사전 로딩으로 즉시 추출 가능)
    var frame2 = tac_main.getFrame(1);
    var win2 = frame2 ? ((typeof frame2.getWindow === "function") ? frame2.getWindow() : frame2) : null;
    if (win2 && win2.dlt_team) {
        result.tab2Data = onlyModified ? win2.dlt_team.getModifiedJSON() : win2.dlt_team.getAllJSON();
    }

    return result;
};

// [일괄 저장 submission 예시]
scwin.btn_saveAll_onclick = function() {
    var payload = scwin.fn_getAllTabData(true); // 수정된 데이터만 추출
    console.log("전체 탭 일괄 저장 데이터:", payload);
    // $p.executeSubmission(...)
};
```

### WebSquare 메인 화면 동적 탭 생성 및 WFrame 연동 (`grid_multicheck_combo.xml`)

```javascript
scwin.onpageload = function() {
    // 1. 공통 코드 매핑
    scwin.itemCodeMap = { ... };

    // 2. 동적 탭 생성 실행
    scwin.fn_initDynamicTabs();
};

/**
 * [동적 탭 생성] 개별 WFrame 파일들을 addTab으로 동적 추가 및 백그라운드 사전 로딩
 */
scwin.fn_initDynamicTabs = function() {
    var tabConfigs = [
        { id: "tab1", label: "1. 담당 업무 배정 (메인 그리드)", src: "tab1_task_assign.xml", closable: false },
        { id: "tab2", label: "2. 프로젝트 팀 현황 (보조 그리드)", src: "tab2_team_status.xml", closable: false },
        { id: "tab3", label: "3. 공통 코드 설정 및 가이드", src: "tab3_guide.xml", closable: false }
    ];

    tabConfigs.forEach(function(tabInfo, idx) {
        tac_main.addTab(
            tabInfo.id,
            {
                label: tabInfo.label,
                closable: tabInfo.closable,
                openAction: (idx === 0) ? "select" : "exist",
                alwaysDraw: true // 메인 로딩 시 백그라운드 사전 렌더링!
            },
            {
                src: tabInfo.src,
                wframe: true
            }
        );
    });
};

// [탭 변경 확인] WFrame 스코프의 DataList 수정 여부 검사
scwin.fn_isTabModified = function(tabIndex) {
    var frame = tac_main.getFrame(tabIndex);
    if (!frame) return false;
    var win = (typeof frame.getWindow === "function") ? frame.getWindow() : frame;
    if (!win) return false;

    if (tabIndex === 0 && win.dlt_sample) {
        return win.dlt_sample.getModifiedIndex().length > 0;
    } else if (tabIndex === 1 && win.dlt_team) {
        return win.dlt_team.getModifiedIndex().length > 0;
    }
    return false;
};
```

```javascript
// [탭 이동 확인 플래그] $p.confirm 콜백 후 프로그래밍 탭 이동 허용
scwin._allowTabChange = false;

// 탭 데이터 수정 여부 검사 헬퍼 함수
scwin.fn_isTabModified = function(tabIndex) {
    if (tabIndex === 0) {
        return (typeof dlt_sample.getModifiedIndex === "function") 
               ? dlt_sample.getModifiedIndex().length > 0 
               : false;
    } else if (tabIndex === 1) {
        return (typeof dlt_team.getModifiedIndex === "function") 
               ? dlt_team.getModifiedIndex().length > 0 
               : false;
    }
    return false; // 탭 3은 읽기 전용 가이드
};

// WebSquare onbeforetabchange 이벤트 핸들러
scwin.tac_main_onbeforetabchange = function(currentTabIndex, newTabIndex) {
    // 사용자가 확인창에서 [확인]을 눌러 프로그래밍 탭 전환 중인 경우 통과
    if (scwin._allowTabChange) {
        scwin._allowTabChange = false;
        return true;
    }

    // 현재 탭에 수정 내역이 있는지 확인
    var isModified = scwin.fn_isTabModified(currentTabIndex);

    if (isModified) {
        var tabNames = ["담당 업무 배정", "프로젝트 팀 현황", "공통 가이드"];
        var curName = tabNames[currentTabIndex] || "현재 탭";
        var confirmMsg = "[" + curName + "] 탭에 저장되지 않은 변경사항이 있습니다.\n다른 탭으로 이동하시겠습니까?";

        // WebSquare 공식 confirm 호출
        if (window.$p && typeof $p.confirm === "function") {
            $p.confirm(confirmMsg, function(result) {
                if (result) {
                    scwin._allowTabChange = true;
                    tac_main.setSelectedTabIndex(newTabIndex);
                }
            });
        }
        // $p.confirm 콜백 응답 전까지 전환 일시 차단
        return false;
    }

    return true;
};
```

---

## 4. 산출물 및 검증 환경

| 산출물 | 설명 | 실행 / 확인 방법 |
|:---|:---|:---|
| [`grid_multicheck_combo.xml`](file:///c:/work/grid_multicheck_combo.xml) | 메인 화면 컨테이너 (동적 addTab WFrame 컨테이너, 상단 전달사항, 엑셀/프리뷰) | WebSquare Studio 화면 디렉토리에 복사 후 실행 |
| [`tab1_task_assign.xml`](file:///c:/work/tab1_task_assign.xml) | [동적 WFrame 1] 담당 업무 배정 그리드, 행 추가, 담당자 엔터 검색, 멀티체크 콤보 | 단독 또는 메인 화면을 통해 로딩 |
| [`tab2_team_status.xml`](file:///c:/work/tab2_team_status.xml) | [동적 WFrame 2] 프로젝트 팀 현황 보조 그리드, 팀 과업 멀티체크 콤보 | 단독 또는 메인 화면을 통해 로딩 |
| [`tab3_guide.xml`](file:///c:/work/tab3_guide.xml) | [동적 WFrame 3] 공통 업무 코드 카드 및 WFrame 가이드 안내 | 단독 또는 메인 화면을 통해 로딩 |
| [`demo_preview.html`](file:///c:/work/demo_preview.html) | WebSquare 무설치 독립형 브라우저 시뮬레이터 (동적 탭 아키텍처 반영) | Chrome / Edge에서 더블 클릭 실행 |
| [`README.md`](file:///c:/work/README.md) | 전체 프로젝트 기능 소개 및 가이드 문서 | 저장소 메인 README |
| [`SESSION_SUMMARY.md`](file:///c:/work/SESSION_SUMMARY.md) | 본 세션 질의응답 및 기술 솔루션 총정리 문서 | 본 문서 |

---
*작성일자: 2026-09-14*  
*저장소: [jhc1006/websquare-grid-checkcombobox](https://github.com/jhc1006/websquare-grid-checkcombobox)*
