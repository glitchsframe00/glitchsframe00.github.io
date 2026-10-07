# Web Personal Data Masking Auditor (Console Ver.)

브라우저 개발자 도구(Console)에서 웹 화면 내 개인(신용)정보 노출 및 마스킹 현황을 즉시 진단하는 스크립트입니다. 별도의 프로그램 설치나 외부 통신 없이 브라우저 자체 DOM을 분석하므로 사내망 및 폐쇄망 환경에서도 제약 없이 실행할 수 있습니다.

---

## 📌 주요 특징
- **무설치 / 보안 솔루션 우회**: Python 환경 구축이나 CDP 디버깅 옵션 없이 브라우저 콘솔에서 즉시 실행
- **안전 모드 (개인정보 원본 미저장)**: 실제 데이터 값은 메모리나 로컬에 저장하지 않고, 노출 및 마스킹 적용 여부(O/X 플래그)만 진단
- **직관적인 결과 출력**: `console.table()`을 통해 취약/의심 항목을 우선 정렬한 표 형태로 즉각 확인 가능

---

## 🔍 점검 대상 항목
- **포맷 기반 (정규식)**: 주민등록번호, 휴대전화번호, 신용카드번호, 계좌번호
- **라벨 매핑 기반**: 이름, 성명, 계약자명, 주소, 청약번호, 계약번호, 증권번호, 상품명, 상품코드, 납입금액, 결제금액, 수신번호, 전화번호, 팩스번호, 주민번호, 계좌번호, 은행코드, 이메일

---

## 🚀 사용 방법
1. 점검 대상 웹 페이지에 접속한 뒤 **F12**를 눌러 브라우저 개발자 도구를 엽니다.
2. **Console(콘솔)** 탭으로 이동합니다.
   > **참고**: 최초 붙여넣기 차단 경고가 뜨는 경우, 콘솔에 `allow pasting` (또는 `붙여넣기 허용`)을 입력하고 Enter를 친 후 다시 붙여넣으세요.
3. 아래의 스크립트 전체를 복사하여 콘솔 창에 붙여넣고 **Enter**를 누릅니다.
4. 콘솔 창에 출력된 점검 결과 표를 확인합니다.

---

## 💻 스크립트 코드
```javascript
(function runDynamicFieldAudit() {
  // 1. 마스킹 기호 식별 패턴 (*, #, ●, ○ 등)
  const MASK_PATTERN = /[*#●○]/;

  // 2. 화면 제어용 버튼/UI 공통 키워드 (라벨 추출 시 제외할 노이즈)
  const UI_EXCLUDES = new Set([
    '선택', '조회', '검색', '초기화', '닫기', '저장', '삭제', '등록', 
    '수정', '이전', '다음', '목록', '전체', '상세', '보기', '다운로드', 'excel'
  ]);

  // 라벨별 진단 결과 집계 맵 (중복 제거용)
  const fieldSummary = new Map();

  function recordField(rawLabel, rawValue) {
    if (!rawLabel || !rawValue) return;

    // 라벨 정제 (공백, 콜론, 특수기호 제거)
    const label = rawLabel.replace(/[\s:：·\-_\t\r\n]+/g, ' ').trim();
    const val = rawValue.trim();

    // 유효성 검증: 라벨 길이 및 UI 노이즈 필터링
    if (label.length < 2 || label.length > 20) return;
    if (UI_EXCLUDES.has(label) || label === val) return;
    if (val.length === 0) return;

    if (!fieldSummary.has(label)) {
      fieldSummary.set(label, {
        maskedCount: 0,
        unmaskedCount: 0,
        totalCount: 0
      });
    }

    const item = fieldSummary.get(label);
    item.totalCount += 1;

    // 마스킹 적용 여부 판정
    if (MASK_PATTERN.test(val)) {
      item.maskedCount += 1;
    } else {
      item.unmaskedCount += 1;
    }
  }

  // --- [DOM 자동 탐색 엔진] ---

  // 1) 表(Table) 구조 자동 분석
  document.querySelectorAll('table').forEach(table => {
    const rows = Array.from(table.querySelectorAll('tr'));
    if (rows.length === 0) return;

    // A. 가로형 표 (th 바로 옆의 td)
    rows.forEach(tr => {
      const ths = tr.querySelectorAll('th');
      ths.forEach(th => {
        const nextTd = th.nextElementSibling;
        if (nextTd && nextTd.tagName === 'TD') {
          recordField(th.innerText, nextTd.innerText);
        }
      });
    });

    // B. 세로형 목록 표 (상단 th 헤더 열 -> 하단 td 데이터 열 추적)
    const headerRow = rows[0];
    const headers = Array.from(headerRow.querySelectorAll('th, td'));
    if (rows.length > 1 && headers.length > 1) {
      headers.forEach((header, colIdx) => {
        const colLabel = header.innerText;
        // 최대 10개 행 데이터 샘플링 검사
        for (let r = 1; r < Math.min(rows.length, 11); r++) {
          const cells = rows[r].querySelectorAll('td');
          if (cells[colIdx]) {
            recordField(colLabel, cells[colIdx].innerText);
          }
        }
      });
    }
  });

  // 2) dt - dd 정의 목록 구조
  document.querySelectorAll('dt').forEach(dt => {
    const nextDd = dt.nextElementSibling;
    if (nextDd && nextDd.tagName === 'DD') {
      recordField(dt.innerText, nextDd.innerText);
    }
  });

  // 3) label - input/select 구조
  document.querySelectorAll('label').forEach(labelEl => {
    let val = '';
    if (labelEl.htmlFor) {
      const target = document.getElementById(labelEl.htmlFor);
      if (target) val = target.value || target.placeholder || '';
    } else {
      const innerInput = labelEl.querySelector('input, select');
      if (innerInput) val = innerInput.value || innerInput.placeholder || '';
    }
    if (val) recordField(labelEl.innerText, val);
  });

  // 4) 콜론(:) 인라인 텍스트 및 Div/Span 그리드 구조
  const textContainers = document.querySelectorAll('p, div, span, li');
  textContainers.forEach(el => {
    // 직계 자식이 없고 순수 텍스트만 있는 요소 대상
    if (el.children.length === 0 && el.innerText) {
      const text = el.innerText.trim();
      const colonMatch = text.match(/^([^:：\n\r]{2,15})[:：]\s*(.+)$/);
      if (colonMatch) {
        recordField(colonMatch[1], colonMatch[2]);
      }
    }
  });

  // --- [결과 데이터 가공 및 중복 정리] ---
  const results = [];
  for (const [label, data] of fieldSummary.entries()) {
    let maskingStatus = "";
    let verdict = "";

    if (data.unmaskedCount > 0 && data.maskedCount === 0) {
      maskingStatus = "X (전부 미적용)";
      verdict = "취약";
    } else if (data.unmaskedCount === 0 && data.maskedCount > 0) {
      maskingStatus = "O (전부 마스킹)";
      verdict = "양호";
    } else {
      maskingStatus = `△ (혼재: 마스킹 ${data.maskedCount}건 / 미적용 ${data.unmaskedCount}건)`;
      verdict = "취약 의심 (확인 필요)";
    }

    results.push({
      "항목명 (라벨)": label,
      "화면 노출 여부": "O",
      "마스킹 여부": maskingStatus,
      "최종 판정": verdict,
      "발견 건수": `${data.totalCount}건`
    });
  }

  // 정렬: 취약(마스킹 X) 항목을 맨 위에 배치
  results.sort((a, b) => {
    const score = v => (v.includes("취약") ? (v === "취약" ? 3 : 2) : 1);
    return score(b["최종 판정"]) - score(a["최종 판정"]);
  });

  // --- [콘솔 표 출력] ---
  console.clear();
  console.log("%c[+] 화면 내 전수 항목 동적 탐색 진단 결과", "color: #00e676; font-size: 15px; font-weight: bold;");
  
  if (results.length === 0) {
    console.warn("⚠️ 화면에서 분석 가능한 데이터 라벨을 찾지 못했습니다. (프레임 확인 필요)");
  } else {
    console.table(results);
  }

  return results;
})(); 
```
