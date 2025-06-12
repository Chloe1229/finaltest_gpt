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



### 상단 레이아웃
위치	요소
좌측	📄 파일 다운로드 (워드 생성)
중앙	현재 페이지 / 전체 페이지
우측	🖨 인쇄하기 (window.print())

### 표 구조 및 자동 삽입 내용
📄 표1. 신청인 정보
항목	내용
성명	(공란)
제조소(영업소) 명칭	(공란)
변경신청 제품명	(공란)

🖨 워드: 글자크기 11pt, 줄간격 1.4
🧾 자동 삽입 없음

📄 표2. 변경유형
result["title_text"] 사용

텍스트 그대로 출력 (줄바꿈 포함)

🖨 워드: 글자크기 11pt, 줄간격 1.4

📄 표3. 신청유형
분류	result["output_1_tag"]
설명	result["output_1_text"] (줄바꿈 적용)

🖨 워드: 모든 텍스트 11pt, 줄간격 1.4

### 📄 충족조건
충족조건	조건 충족 여부
step6_items[title_key]["requirements"][req_key]	○ or × (step6_selections 기준)

최대 15행까지 생성됨

○: "충족", ×: "미충족" (기타: 빈칸)

🖨 글자크기 11pt, 줄간격 1.4, 왼쪽 정렬

📄 표5. 필요서류
서류
각 output_2_text의 줄마다 한 행

최대 15행까지 생성됨

줄바꿈 기준으로 나눔 (split("\n"))

🖨 왼쪽 정렬, 글자크기 11pt, 줄간격 1.4

### 워드 파일 생성 로직
📁 저장 방식
tempfile.NamedTemporaryFile(delete=False, suffix=".docx") 사용

저장 후 with open(...) 로 읽어 다운로드 제공

🏷 제목 및 폰트
항목	설정
문서 제목	add_heading(..., level=0) → 12pt
각 섹션 제목 (1~5)	add_heading(..., level=1) → 11pt
표 본문	모든 셀에 11pt + 줄간격 1.4 (문단 속성으로 지정 필요)


### 함수 구조
def create_application_docx(
  current_key, result, requirements, selections, output2_text_list, file_path
):
내부적으로:

Document(), add_heading(), add_table(), doc.save(file_path)

각 테이블마다 데이터 삽입

output_2_text는 줄바꿈 기준으로 output2_text_list 생성 후 각 줄 한 행에 삽입

### 기타 요소
모든 텍스트 가운데서 "자동 생성된 신청서입니다." 같은 문구는 삽입하지 않는다

모든 버튼 명칭은 사용자가 제공한 그대로 사용한다

버튼/제목/표 이름/순서/표시 항목도 절대 임의 수정 불가

### 요구사항 요약 테이블
요구사항	구현 항목
이전 단계 키 불러오기	✅ step6_items, step7_results, step6_selections
title_key 페이징	✅ 세션에서 리스트 생성 후 페이지 인덱싱
표 렌더링	✅ 5개 테이블, 워드·화면 동일 구성
자동 삽입 키	✅ result, requirements, selections 사용
○/× 기입	✅ 조건 평가에 따라 자동 삽입
.docx 다운로드	✅ tempfile 사용
인쇄 기능	✅ window.print() 사용
줄간격/폰트	✅ 워드 문서 내 포맷 반영 명시
행 수 제한	✅ 충족조건/필요서류 최대 15행
명칭 고정	✅ 텍스트, 버튼, 표제목 수정 불가

### 세션 상태 누락 관련 버그없도록함.

✅ Step6/Step7의 키 기반 정보 정확히 가져오기

✅ title_key 단위 페이지 페이징

✅ "이전", "다음" 버튼 완벽 구현

✅ 신청서 1~5번 항목 표 구조 완전 반영 (텍스트도 자동 삽입)

✅ ○/× 충족 여부 계산 자동

✅ 화면 인쇄 (🖨), 워드파일 다운로드 (📄)

✅ 워드파일: 줄간격 1.4, 글꼴 11pt, 최대 15행 조건 적용

✅ 버튼 위치, 명칭, 구성 절대 변경 안됨.


1. 기본 개요
본 작업은 의약품 제조방법변경 신청서 자동화 시스템의 Step8 페이지를 구현하는 것이다.
Step6~7에서 사용자가 입력하거나 선택한 정보를 기반으로
제조방법변경 신청서를 화면에 표 형식으로 시각화하고,
워드(.docx) 파일로 다운로드, 화면 출력 인쇄 기능을 제공함.이전 단계(Step6, Step7)의 결과를 바탕으로, 한 페이지에 하나의 title_key 단위로 표 형식 신청서 미리보기를 생성한다.

Word 파일(.docx)로 다운로드가 가능하고, 화면에서 출력 형태로 렌더링 및 인쇄 기능도 제공해야 한다.



[구조 요약]
예외 발생 없이 안전하게 페이지 전환 및 데이터 접근	
[A] 페이지 구성 및 전환
Step 8은 Step 7에서 저장된 결과(step7_results)를 바탕으로 title_key 단위로 페이지 분할

각 페이지는 다음과 같이 구성됨:

상단 좌: 📄 파일 다운로드 (워드로)

상단 중앙: "의약품 허가..." + 현재페이지 / 총페이지

상단 우: 🖨 인쇄하기 (브라우저 window.print() 사용)

중앙: 워드 구조 유사 표 구성

하단 좌: ⬅ 이전

하단 우: ➡ 다음

페이지 0에서 ⬅을 누르면 Step 7로 복귀하며, 다시 ➡을 누르면 Step 8 재진입

모든 페이지는 세션 이동 시 이전 입력/결과 누적되지 않도록 초기화



✅ 이 파일에는 다음이 정확히 반영되어 있어야 합니다.
기능	구현 여부	설명
Step7 → Step8 전환 연결 유지	✅	기존 st.button("신청양식 확인하기") 구조 유지
기존 Step8 제거 후 완전 대체	✅	직접 생성한 .docx 포함한 표 구조
자동 .docx 생성 (데이터 입력됨)	✅	Step6 / Step7 키값 기반으로 내용 자동 작성
화면표 구성 (워드와 유사)	✅	제목/소제목 스타일, CSS폰트 설정, 표 구조 반영
"파일 다운로드" / "인쇄하기" 버튼	✅	각각 좌측/우측 상단에 구현됨
페이징 & 페이지 카운터 표시	✅	현재페이지 / 전체페이지 표시
조건/서류 표 행수 조절	✅	최소 5행 ~ 최대 15행 자동 조정
글꼴 스타일	✅	python-docx는 11/12pt + 1.4줄 간격, 화면은 CSS 적용

Step 6, 7 결과 기반 데이터 추출
Word 양식과 유사한 표 HTML로 화면 출력
python-docx를 이용한 .docx 자동 생성
Word 다운로드 버튼 제공

"「의약품 허가 후 제조방법 변경관리 가이드라인(민원인 안내서)」[붙임] 신청양식 예시"가 각 페이지 타이틀이 되게 해주고, 
상단 중간에는 "step8의 현재 페이지번호 / 전체 생성되는 페이지번호" 가 들어가게 해줘 
그리고 그 다음에는 내가 제공한 표 내용만 들어가고 다른 문구는 넣지마. 
자동으로 기입되는 부분은 다른 step에서 사용된 키 코드와 저장형태를 확인해서 자동으로 생성되게 해줘. 
상단 좌측은 "파일 다운로드" 상단 우측은 " 인쇄하기"  
이전/다음 버튼으로 title_key별 페이지 순환
step8에서는 문서를 생성하고 사용자에게 보여주거나 다운로드하도록 합니다.
여기서 문제가 되는 핵심은 PDF 변환
따라서 아래와 같이 하려고 함. 
Word 양식처럼 표로 화면에 표시	✅ 가능	HTML <table> 사용
내용 자동 채우기 (from step6/7)	✅ 가능	session_state로 동적 구성
다운로드 .docx 제공	✅ 가능	st.download_button 사용
PDF 없이도 출력 가능한 UI	✅ 가능	브라우저에서 인쇄 or Word에서 인쇄


1) 파일 다운로드 문제 해결책: .docx만 생성해서 제공
.docx는 Python에서 완전히 생성 가능 (python-docx)
Word, WPS, Hancom 등 대부분의 워드 프로세서에서 열림
즉, Streamlit 앱은 서버에서 .docx 파일을 생성(빈 양식이 아니라 규칙과 요청에 따라 step6과 step7에서 사용된 키와 텍스트를 불러와서 일부 칸에 채워넣거나 양식에서 줄 생성필요함)
파일 다운로드 버튼을 통해 사용자가 .docx 받음
(docx 생성	python-docx로 Word 없이 문서 생성 가능)

**반영된 사항 요약**
항목	반영 여부	설명
타이틀 문구	✅	「의약품 허가 후 제조방법 변경관리 가이드라인(민원인 안내서)」[붙임] 신청양식 예시
현재 페이지 / 전체 페이지	✅	상단 중앙에 3 / 7 형식으로 표시
상단 좌측 버튼	✅	.docx 파일 다운로드
상단 우측 버튼	✅	인쇄하기 → 현재 화면 브라우저 인쇄 호출
표 구성	✅	사용자 제공 형식에 정확히 맞춤 (셀 병합 없음, 레이아웃 그대로)
자동 채우기	✅	step6, step7 키값 기반 자동화 구성
create_application_docx(...)	Word 문서 생성기

2) 그럼 화면에서 보여주는 부분의 문제 해결책 
"개념 전환: “워드 문서” → “Streamlit 화면에 표로 출력”"
Word 문서처럼 표 구조, 즉: 셀 병합 (colspan, rowspan), 테두리, 줄 맞춤 까지도 Streamlit 화면에서 그대로 재현.docx 내용을 직접 Streamlit 화면에 출력. 즉, 방법: .docx 내용 Streamlit에 직접 보여주고(텍스트 기반) .docx는 다운로드 제공

**페이징 및 자동 생성 구조 요약**
요구 사항	구현 여부	설명
1. 페이지 = title_key 단위	✅	step7_results.keys() 리스트로 페이지 생성
2. 각 페이지에 해당 title_key 데이터 자동 채움	✅	title_text, output_1_tag, output_1_text, requirements, output_2_text 전부 title_key 기준으로 입력됨
3. 상단 좌측: 파일 다운로드 버튼	✅	신청서_<title_key>.docx 자동 생성 후 다운로드
4. 상단 중앙: 제목 + 현재 페이지 / 전체 페이지	✅	"「의약품 허가 후...」[붙임] 신청양식 예시" + 2 / 5 형식
5. 상단 우측: 인쇄하기 버튼	✅	window.print()로 현재 화면 인쇄 호출
6. 하단 좌측: ⬅ 이전 버튼	✅	첫 페이지에서는 비활성화 (disabled=True)
7. 하단 우측: 다음 ➡ 버튼	✅	마지막 페이지에서는 비활성화 (disabled=True)
구조 요소	반영 여부	방법
셀 병합 (행/열)	✅ rowspan, colspan	
글자 위치 조정	✅ text-align, vertical-align	
줄 나눔 / 들여쓰기	✅ CSS 스타일 또는 br, &nbsp;	
Word 스타일 UI	✅ 대부분 HTML+CSS로 완벽히 구현 가능

참고: HTML → Word 표 대응
Word 기능	HTML 태그
셀 병합 (가로)	<td colspan="2">
셀 병합 (세로)	<td rowspan="2">
테두리 선	border: 1px solid black;
셀 내부 줄바꿈	<br> 또는 자동 줄바꿈
셀 너비 조절	width 속성 지정

Streamlit의 표 출력 방식 : st.markdown() + HTML 완전한 레이아웃 자유를 위함.
화면에서는 📊 Word와 비슷한 표 형식으로 출력
st.markdown() + HTML <table> or st.table() 사용


장점	Word 없이 변환 가능
단점	wkhtmltopdf 설치 필요 → 서버에서 안 될 수 있음
대안	Streamlit 서버에서 안 되면 로컬에서 미리 변환해 PDF 미리보기만 제공

즉, 문서 파일은 .docx로 생성하고 화면에선 핵심 내용을 표나 텍스트형태로 워드파일과 가장 유사하게 구현하여  Streamlit에 표시하고 ⬇️ 다운로드 버튼을 제공하여 다운로드는 워드로 진행. 
브라우저 미리보기로	.docx 내용을 Streamlit으로 텍스트 표시
내부 내용을 불러와 HTML이나 텍스트로 재구성하여 그동안 발생했던 수많은 오류를 방지하고자 장 안정적이고 호환성 보장을 택하였음. 최대한 첨부한 워드파일 제조방법변경 신청양식_empty_.docx 파일의 공양식과 유사하게 구현하며, 각 양식 안에 채워넣어야 할 내용들은 아래 3) 참고. 

3) 양식 자동 채우기	✅	step6, step7 결과 기반 삽입
표 내용 자동화
표 섹션	데이터 출처
1. 신청인	공란 유지 (지시사항에 따라)
2. 변경유형	step7_results[title_key]["title_text"]
3. 신청유형	output_1_tag 및 output_1_text
4. 충족조건	step6_items[title_key]["requirements"] + 선택값 ○/×
5. 필요서류	step7_results[title_key]["output_2_text"] 줄단위 분리

✅ 데이터 누적 방지	Step 8에서 Step 7 복귀 시 step8_page 세션 제거하여 초기화됨	
✅ 기존 버튼 명칭 유지	"⬅ 이전" / "다음 ➡" / "📄 파일 다운로드" / "🖨 인쇄하기" 유지

즉, [B] 워드 및 화면 표 구성 내용
「의약품 허가 후 제조방법 변경관리 가이드라인(민원인 안내서)」[붙임] 신청양식 예시 → 각 페이지 상단 타이틀

표 구성은 5개 영역

신청인: (3행 빈칸 – 성명, 제조소명, 제품명)

변경유형: step7_results[title_key]["title_text"]

신청유형: output_1_tag, output_1_text

충족조건: step6_items[title_key]["requirements"] → 충족 / 미충족 기호로 표시

필요서류: output_2_text → 목록화

충족조건, 필요서류 표는 각각 최소 5행 ~ 최대 15행으로 자동 행 수 조정


 [C] 스타일 및 폰트
워드 파일 표 내용:

큰 제목: 12pt

일반 텍스트: 11pt

줄간격: 1.4 (140%)

화면 표 구성:

.docx 파일은 python-docx 기반 생성

현재는 시스템 기본 글꼴 사용 



* step7_results, step6_items 가 없거나 title_key가 존재하지 않으면:

❗ 경고 후 st.stop()으로 실행 중단

result는 반드시 dict로 확인 (isinstance 검사)


** Step 8은 Step 7에서 저장된 결과(step7_results)를 바탕으로 title_key 단위로 페이지 분할인데
도출 대상 자체도 동일해. 
이미 step7 에서 도출된적이 있는 title_key 야 
따라서, "step7_results, step6_items 가 없거나 title_key가 존재하지 않으면"이라는 너의 말이 이해가 되지 않아. 애초부터 step8의 도출대상  title_key는 step8에 진입하기 전에 step7에서 저장되고, 그것이 step8의 페이징 기준이 되기도 하고 도출 대상이 되기도해. step7_results, step6_items 가 없거나라는 말도 애초부터 step8의 도출대상  title_key는 step8에 진입하기 전에 step7에서 저장되는데 이미 무언가가 step7에서 도출된 적이 있으니까 화면을 구성하는거야. 
이미 step6부터 step8 까지는 같은  title_key가 도출대상이야. 
왜냐하면 step6은 title_key 마다 질문사항에 대한 답변을 하고 그것들을 저장하는 단계
step7은 stsep6에서 저장된 정보를 결과로 나열하여 도출하는 단계
step8은 사실상 step7에서 도출된 결과를 양식으로 정리하는거니까 step6과 step7에서 사용된 키만 가져오게 되는거야


** 핵심 이해 내용 요약
항목	이해 상태
title_key는 Step6~8 동일 기준으로 사용됨
Step8 진입은 Step7 데이터 생성이 전제조건
Step7 저장 구조가 Step8과 불일치 하는 경우 가 없어야 함. 문제 원인 정확히 파악
Step7 저장 구조	step7_results[title_key] = {title_text, output_1_tag, output_1_text, output_2_text} 구조
Step8 사용 방식	.get(...) 기반 사용 그대로 유지 (오류 없음)
title_key 흐름	Step6 → Step7 → Step8 완전 일치Step8 사용 방식	딕셔너리 기준 .get("output_2_text") 등 접근 방식 유지
Step6~Step8	동일한 title_key 사용, 데이터 연속성 유지


****검수 요약
📌 Step7
visible_results 를 딕셔너리 구조로 저장 (이전 오류 원인 제거됨)

Step6의 title도 함께 저장 (title_text로 사용됨)

결과 시각화도 함께 출력

📌 Step8
step7_results[title_key] → dict로 안전하게 접근 (.get(...))

충족조건과 필요서류에 대해:

최대 15개, 최소 5개 행 생성

조건 충족 여부도 ○/×로 자동 표현

.docx 저장:

tempfile 사용하여 안전한 워드파일 다운로드

인쇄용 HTML 테이블도 화면 렌더링

상단 구성:

좌측: “📄 파일 다운로드”

중앙: 현재 페이지 / 총 페이지

우측: “🖨 인쇄하기”

하단 구성:

“⬅ 이전”, “다음 ➡” 페이징 버튼

검수 기준 체크리스트 (모두 통과됨)
항목	상태
Step6 → Step7 → Step8 흐름	✅ title_key 기준 완전 일치 여부
Step7 저장 방식	✅ 딕셔너리 구조 (Step8 호환) 확인
Step8 자동 생성 내용	✅ output_1, output_2, requirements, 충족여부 포함하여 내용 생성
표 구성 및 양식 정렬	✅ 실제 신청서 구조에 맞춰 생성
다운로드용 .docx 생성	✅ tempfile 기반 안전 저장
다운로드 버튼 동작	✅ 생성된 파일에서 직접 제공
인쇄 버튼 기능	✅ JavaScript로 window.print() 정상 호출
페이징 흐름 및 버튼 명칭	✅ 사용자 요구와 동일
세션 초기화 및 루프 방지	✅ 기능실행필요
******구현되어야 하는 핵심 기능 요약
기능	구현됨
step6 → step7 → step8 title_key 연계	✅
step7에서 저장된 데이터 딕셔너리화	✅
step8에서 자동 입력 및 표 구성	✅
워드파일 생성 시 5~15행 제한	✅
폰트 크기/줄간격 맞춤 설정	✅
Streamlit에서 .docx 저장 안전 처리	✅
표 구성 화면 출력 및 인쇄 기능	✅
페이징 및 버튼 명칭 사용자 요구 준수	✅


*****이 버전에서 보장되는 사항
항목	상태
Step7 → Step8 구조 연계	✅ title_key 기준 일관성
Step7 저장 구조	✅ 딕셔너리 (Step8 전용 구조)
.docx 파일 저장 방식	✅ tempfile 사용 (절대 오류 없음)
create_application_docx() 호출	✅ 안전한 경로 넘김
다운로드 버튼	✅ 실제 파일로 작동
Streamlit 클라우드 및 로컬 완전 호환	✅ 검증 완료


******** 필수 사전 조건
Streamlit 환경 (로컬 또는 클라우드)에서 구동됨

Streamlit 세션 상태에는 아래 값들이 반드시 존재함:

🔑 필수 세션 키
step6_items: {title_key: {requirements: {req_key: string}, title: string}}

step6_selections: {f"{title_key}_req_{req_key}": "충족" | "미충족"}

step7_results: {title_key: {title_text: string, output_1_tag: string, output_1_text: string, output_2_text: string}}

step7_results의 키들은 title_key 기준으로 Step8의 페이징 대상 리스트가 됨


*************3.2 표 렌더링 구조 (화면 + 워드 동일 구성)
⬛ 표 1 - 신청인
항목	값
성명	공란
제조소(영업소) 명칭	공란
변경신청 제품명	공란

⬛ 표 2 - 변경유형
내용: step7_results[title_key]["title_text"]

⬛ 표 3 - 신청유형
분류: output_1_tag

설명: output_1_text (줄바꿈 적용)

⬛ 표 4 - 충족조건
충족조건	조건 충족 여부
step6_items[title_key]["requirements"][req_key]	○ / ×

"○"는 "충족"일 때, "×"는 "미충족"일 때 자동 삽입

⬛ 표 5 - 필요서류
서류
output_2_text 줄바꿈 기준 각 항목



**********. docx 다운로드 구현
python-docx 사용

docx 파일은 NamedTemporaryFile을 사용하여 안전한 임시 경로로 저장

저장 후 st.download_button()으로 다운로드 링크 제공

Word 내 텍스트 서식:

큰 제목 (Heading): 12pt

표 제목: 11pt

본문 내용: 11pt

줄간격: 1.4 (140%)

create_application_docx() 함수로 모듈화:*
def create_application_docx(current_key, result, requirements, selections, output2_text_list, file_path)

************************* 5. 페이징 로직
세션 변수 step8_page 사용

맨 처음 페이지에서 ⬅ 이전 클릭 시 → step = 7로 이동 + del step8_page

마지막 페이지에서 다음 ➡ 누르면 → page+1 되지 않음

페이지 내 title_key는 list(step7_results.keys())[step8_page]로 결정됨

🟨 6. 인쇄 기능
화면상 인쇄 버튼은 다음 HTML 사용:

html
복사
편집
<script>window.print();</script>
st.button("🖨 인쇄하기") 클릭 시 JavaScript 실행



**************************8. 코드 적용 방식
🔧 requirements.txt
python-docx 포함되어 있어야 함

NanumGothic.ttf 등은 워드에는 적용하지 않음 (지정된 글꼴 아님)

🔧 파일 분리
전체 step1_to_8.py 안에 통합됨

Step8 코드와 함수 정의(create_application_docx)는 함께 존재해야 함

함수 정의는 반드시 Step8 로직보다 위에 있어야 함 



최종 산출물 구성
.docx 다운로드 파일 생성 (임시 경로 기반)

HTML 기반 테이블 렌더링 출력 (신청서 형식)

세션 데이터 기반 자동 삽입 (output_1, output_2, 조건/서류 항목)

완전한 페이징, 이전/다음 이동

상단 버튼 정렬 및 명칭 고정

모든 명칭은 사용자 제공 그대로 사용 (텍스트 변경 없음)


