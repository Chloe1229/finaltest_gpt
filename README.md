# Step 8 최종 작업지시서 (Codex용 완전 구현 지침서)

## 목적
의약품 제조방법 변경관리 신청서를 자동으로 구성하기 위한 Step 8 페이지를 Streamlit 기반으로 구현한다.
이 페이지는 step6, step7에서 저장된 데이터를 기반으로 title_key 단위로 화면에 신청서를 표 형태로 표시하고,
동일한 내용을 docx 양식으로 생성하여 다운로드 제공하며, 화면에서는 인쇄 가능한 출력용 화면을 구성한다.
각 표는 Word 문서와 화면 모두 동일한 구조, 내용, 줄간격, 폰트 크기로 구성되어야 하며, 
각 항목에 들어가는 텍스트는 이전 단계에서 저장된 정확한 키에 따라 자동으로 삽입된다.

## 환경 전제
- Streamlit 실행 환경에서 동작
- step6과 step7은 이미 완료된 상태이며,
  step6에서 조건에 대한 질문과 충족 여부를 저장했고,
  step7에서 신청 유형 및 필요서류 등 요약 정보를 저장했다.
- step8에서는 step7에서 도출된 title_key 기준으로 페이징하여 한 페이지씩 생성한다.
- step6_items, step6_selections, step7_results 는 모두 세션 상태에 저장되어 있으며, 그 구조는 아래와 같다:

## 세션 저장 구조
st.session_state["step6_items"] = {
  "s1_1": {
    "requirements": {
      "r1": "정의된 충족조건1",
      "r2": "정의된 충족조건2",
      ...
    },
    "title": "3.2.S.1 일반정보\n1. 원료의약품 명칭변경"
  },
  ...
}

st.session_state["step6_selections"] = {
  "s1_1_req_r1": "충족" or "미충족",
  ...
}

st.session_state["step7_results"] = {
  "s1_1": {
    "title_text": "3.2.S.1 일반정보\n1. 원료의약품 명칭변경",
    "output_1_tag": "분류명",
    "output_1_text": "신청유형 상세 설명",
    "output_2_text": "필요서류1\n필요서류2\n..."
  },
  ...
}

## 화면 구성 (Streamlit)

### 상단 영역
- 왼쪽: st.download_button("📄 파일 다운로드", ...)
- 가운데: <현재페이지 / 전체페이지> HTML 렌더링
- 오른쪽: st.button("🖨 인쇄하기") → window.print() 자바스크립트 실행

### 표 구성 (화면 + 워드 동일하게 구성)
1. 신청인
  | 항목 | 내용 |
  |------|------|
  | 성명 | 공란 |
  | 제조소(영업소) 명칭 | 공란 |
  | 변경신청 제품명 | 공란 |
  - 이 표는 사용자가 직접 입력하는 부분이므로 자동 입력 없음

2. 변경유형
  | 변경유형 |
  |------------|
  | step7_results[title_key]["title_text"] |

3. 신청유형
  | 분류 | step7_results[title_key]["output_1_tag"] |
  | 설명 | step7_results[title_key]["output_1_text"] (줄바꿈 적용) |

4. 충족조건
  | 충족조건 | 조건 충족 여부 |
  |------------|------------------|
  | step6_items[title_key]["requirements"][rk] | ○ or × |
  - ○: step6_selections[f"{title_key}_req_{rk}"] == "충족"
  - ×: step6_selections[f"{title_key}_req_{rk}"] == "미충족"
  - 최대 15행까지 생성됨 (조건 수만큼 자동으로)

5. 필요서류
  | 서류 |
  |------|
  | 각 줄마다 step7_results[title_key]["output_2_text"].split("\n") 의 결과 한 줄씩 |
  - 최대 15행까지 표시됨

### 워드 문서 서식
- 제목: Heading 0 (12pt, bold, 가운데 정렬)
- 표 제목: Heading 1 (11pt, bold, 왼쪽 정렬)
- 표 내용 텍스트: 모두 11pt
- 줄간격: 전체 문단 1.4 (140%)
- 표 스타일: Table Grid
- 워드는 `python-docx` 로 생성하고, `tempfile.NamedTemporaryFile(delete=False)`로 저장한다

### 워드 생성 함수 (필수)
def create_application_docx(current_key, result, requirements, selections, output2_text_list, file_path)

- 내부에서 Document 생성
- 표 1~5 순서대로 구성
- ○/×, output_2_text 분리, req 조건별 행 생성 등 포함

### 다운로드 버튼
- 파일 생성 후 open(file_path, "rb")로 읽어 st.download_button() 제공
- 파일명은 f"신청서_{current_key}.docx"

### 인쇄 버튼
- 버튼 클릭 시 window.print() 호출되는 HTML 삽입

### 페이지 이동
- step8_page = 현재 인덱스 (세션 저장)
- 리스트: title_keys = list(step7_results.keys())
- 현재 키: title_keys[step8_page]
- "⬅ 이전": 첫 페이지일 경우 step = 7 로 돌아가고 step8_page 삭제
- "다음 ➡": 마지막 페이지 전까지만 step8_page += 1 가능

### 기타
- 워드에서 폰트는 나눔고딕 등 지정하지 않음
- requirements.txt 수정 불필요 (python-docx만 있으면 됨)
- 생성된 .docx 파일은 화면에 표와 동일한 구조 및 내용
- 출력된 표와 다운로드된 문서 간 차이 없음

### 구현시 유의사항
- 줄바꿈 (`\n`) 은 화면 표에서도 <br>로 처리, 워드에서도 paragraph.splitlines() 형태로 줄 분리
- 버튼 명칭, 위치는 고정 (절대 변경 금지)
- 충족조건과 필요서류는 조건/서류 수만큼 동적으로 행이 생성되어야 함 (최대 15, 최소 5 기준)
- 인쇄 기능은 Streamlit에서 HTML로 구현하고, 실제 프린터 호출은 브라우저 기능 사용
- 각 페이지는 title_key 단위로 분리되며, 이전 페이지로 돌아가면 해당 페이지 상태는 초기화되며 누적 저장되지 않도록 해야 함

