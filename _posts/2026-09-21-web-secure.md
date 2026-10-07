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
(async function runHumanLikeAudit() {
  const MASK_CHARS = "[*#●○Xx_]";
  const maskRegex = new RegExp(MASK_CHARS);

  // 1. 포맷 기반 정규식 (주민번호, 전화번호, 카드, 계좌)
  const REGEX_TARGETS = {
    "주민등록번호(포맷)": {
      unmasked: /\b\d{6}\s*[-]?\s*[1-4]\d{6}\b/,
      masked: new RegExp(`\\b(?:\\d{6}\\s*[-]?\\s*${MASK_CHARS}{6,7}|${MASK_CHARS}{6}\\s*[-]?\\s*[1-4\\d]${MASK_CHARS}{5,6}|\\d{6}\\s*[-]?\\s*[1-4]${MASK_CHARS}{6})\\b`)
    },
    "전화번호(포맷)": {
      unmasked: /\b01[016789]\s*[-.]?\s*\d{3,4}\s*[-.]?\s*\d{4}\b/,
      masked: new RegExp(`\\b01[016789]\\s*[-.]?\\s*(?:${MASK_CHARS}{3,4}\\s*[-.]?\\s*\\d{4}|\\d{3,4}\\s*[-.]?\\s*${MASK_CHARS}{4}|${MASK_CHARS}{3,4}\\s*[-.]?\\s*${MASK_CHARS}{4})\\b`)
    },
    "신용카드번호(포맷)": {
      unmasked: /\b(?:\d{4}[-\s]?){3}\d{4}\b/,
      masked: new RegExp(`\\b(?:\\d{4}[-\\s]?${MASK_CHARS}{4}[-\\s]?${MASK_CHARS}{4}[-\\s]?\\d{4}|\\d{4}[-\\s]?\\d{4}[-\\s]?${MASK_CHARS}{4}[-\\s]?${MASK_CHARS}{4})\\b`)
    },
    "계좌번호(포맷)": {
      unmasked: /\b\d{3,6}[-\s]?\d{2,6}[-\s]?\d{3,6}\b/,
      masked: new RegExp(`\\b\\d{2,6}[-\\s]?[${MASK_CHARS}\\d]{2,6}[-\\s]?${MASK_CHARS}{3,6}\\b`)
    }
  };

  // 2. UI 라벨 키워드 그룹
  const LABEL_GROUPS = [
    { label: "주민등록번호", keywords: ["주민등록번호", "주민번호", "실명번호", "주민등록"] },
    { label: "전화번호", keywords: ["전화번호", "휴대전화", "휴대폰", "핸드폰", "연락처", "수신번호", "팩스번호", "팩스"] },
    { label: "성명/고객명", keywords: ["이름", "성명", "계약자명", "고객명", "피보험자명"] },
    { label: "계좌번호", keywords: ["계좌번호", "계좌", "환불계좌", "입금계좌"] },
    { label: "카드번호", keywords: ["카드번호", "신용카드번호", "체크카드번호"] },
    { label: "증권/계약번호", keywords: ["증권번호", "계약번호", "청약번호"] },
    { label: "주소", keywords: ["주소", "자택주소", "사업장주소", "배송지"] },
    { label: "이메일", keywords: ["이메일", "전자우편", "e-mail", "email"] },
    { label: "금액정보", keywords: ["납입금액", "결제금액", "보험료", "환급금"] },
    { label: "기타코드", keywords: ["상품코드", "상품명", "은행코드"] }
  ];

  const DOWNLOAD_KEYWORDS = ['다운로드', '엑셀', 'excel', 'download', 'csv', 'export', '내려받기'];

  // 접근 가능한 모든 프레임(iframe 포함) 수집
  function getAllDocuments() {
    const docs = [document];
    document.querySelectorAll('iframe').forEach(iframe => {
      try {
        if (iframe.contentDocument) docs.push(iframe.contentDocument);
      } catch (e) { /* Cross-Origin 프레임 접근 제한 무시 */ }
    });
    return docs;
  }

  const allDocs = getAllDocuments();
  let totalFindings = [];

  for (const doc of allDocs) {
    // 1. 메뉴 경로 추출
    const bc = doc.querySelector('.breadcrumb, .location, .navi, .menu-path, nav[aria-label="breadcrumb"]');
    let menuPath = doc.title || document.title || "현재 페이지";
    if (bc) {
      const items = Array.from(bc.querySelectorAll('li, span, a'))
        .map(el => el.innerText.trim())
        .filter(t => t.length > 0);
      if (items.length > 0) menuPath = items.join(' > ');
    }

    // 2. 조회 구분
    const hasPaging = !!doc.querySelector('.pagination, .paging, [class*="page"]');
    const rowCount = doc.querySelectorAll('tr').length;
    const queryType = (hasPaging || rowCount >= 4) ? "대량(목록) 조회" : "개별 조회";

    // 3. 다운로드 버튼 탐지
    let canDownload = "X";
    doc.querySelectorAll('button, a, input').forEach(el => {
      const text = `${el.innerText || ''} ${el.value || ''} ${el.title || ''}`.toLowerCase();
      if (DOWNLOAD_KEYWORDS.some(kw => text.includes(kw))) canDownload = "O";
    });

    const bodyText = doc.body ? doc.body.innerText || "" : "";

    // 4. 정규식 포맷 검사
    for (const [itemName, regexes] of Object.entries(REGEX_TARGETS)) {
      const hasUnmasked = regexes.unmasked.test(bodyText);
      const hasMasked = regexes.masked.test(bodyText);

      let isExposed = "X";
      let isMasked = "-";
      let verdict = "해당 없음";

      if (hasUnmasked || hasMasked) {
        isExposed = "O";
        if (hasUnmasked) {
          isMasked = "X";
          verdict = "취약";
        } else {
          isMasked = "O";
          verdict = "양호";
        }
      }

      totalFindings.push({
        "메뉴 경로": menuPath,
        "조회 구분": queryType,
        "다운로드 기능": canDownload,
        "점검 항목": itemName,
        "노출 여부": isExposed,
        "마스킹 여부": isMasked,
        "최종 판정": verdict
      });
    }

    // 5. 사람 시각 모사 라벨 탐색
    for (const group of LABEL_GROUPS) {
      let itemExposed = "X";
      let isMasked = "-";
      let verdict = "해당 없음";

      let matchedValues = [];

      // 방식 A: 인라인 텍스트 탐색 (예: "주민번호 : 123456-1******" 형태)
      for (const kw of group.keywords) {
        const inlineRegex = new RegExp(`${kw}\\s*[:：\\-]\\s*([^\\n\\r\\t,;<>]{2,30})`, 'g');
        let match;
        while ((match = inlineRegex.exec(bodyText)) !== null) {
          const val = match[1].trim();
          if (val && !group.keywords.some(k => val.includes(k))) {
            matchedValues.push(val);
          }
        }
      }

      // 방식 B: 표(Table) 세로 열(Column) 추적 (헤더 아래 위치한 데이터 행들 탐색)
      const tables = doc.querySelectorAll('table');
      tables.forEach(table => {
        const rows = Array.from(table.querySelectorAll('tr'));
        if (rows.length < 2) return;

        const headerRow = rows[0];
        const headers = Array.from(headerRow.querySelectorAll('th, td'));
        
        headers.forEach((th, colIdx) => {
          const thText = (th.innerText || "").replace(/[\s:：]/g, "");
          if (group.keywords.some(kw => thText.includes(kw))) {
            // 해당 열의 아래 데이터 행(최대 5개 행 표본) 값 확인
            for (let r = 1; r < Math.min(rows.length, 6); r++) {
              const cells = rows[r].querySelectorAll('td');
              if (cells[colIdx]) {
                const val = cells[colIdx].innerText.trim();
                if (val && val.length > 1) matchedValues.push(val);
              }
            }
          }
        });
      });

      // 방식 C: 시각적 인접 요소 (Div/Span/Label 옆 또는 입력창)
      const candidateElements = doc.querySelectorAll('th, dt, label, div, span, p, b, strong');
      for (const el of candidateElements) {
        // 긴 문장이 아닌 라벨 성격의 짧은 텍스트 요소만 필터
        const rawText = el.innerText ? el.innerText.trim() : "";
        if (rawText.length > 25 || rawText.length === 0) continue;

        const cleanText = rawText.replace(/[\s:：]/g, "");
        if (group.keywords.some(kw => cleanText === kw || cleanText.startsWith(kw))) {
          let val = "";

          // 1) 형제 요소 확인
          const sibling = el.nextElementSibling;
          if (sibling) {
            val = sibling.innerText ? sibling.innerText.trim() : (sibling.value || "");
          }
          // 2) 부모의 다음 형제 요소 확인 (Flex/Grid 컬럼 구조 대응)
          if (!val && el.parentElement && el.parentElement.nextElementSibling) {
            const parentSibling = el.parentElement.nextElementSibling;
            val = parentSibling.innerText ? parentSibling.innerText.trim() : (parentSibling.value || "");
          }
          // 3) label의 for 속성 타겟 input 확인
          if (!val && el.tagName === 'LABEL' && el.htmlFor) {
            const inputEl = doc.getElementById(el.htmlFor);
            if (inputEl) val = inputEl.value || inputEl.placeholder || "";
          }

          if (val && val.length > 1 && !cleanText.includes(val.replace(/\s/g, ""))) {
            matchedValues.push(val);
          }
        }
      }

      // 판정 로직: 수집된 값들 중 하나라도 노출이 있으면 점검
      if (matchedValues.length > 0) {
        itemExposed = "O";
        // 수집된 값들 중 마스킹 기호가 없는 원문이 하나라도 노출되어 있으면 취약
        const hasUnmaskedValue = matchedValues.some(v => !maskRegex.test(v));
        if (hasUnmaskedValue) {
          isMasked = "X";
          verdict = "취약";
        } else {
          isMasked = "O";
          verdict = "양호";
        }
      }

      totalFindings.push({
        "메뉴 경로": menuPath,
        "조회 구분": queryType,
        "다운로드 기능": canDownload,
        "점검 항목": group.label,
        "노출 여부": itemExposed,
        "마스킹 여부": isMasked,
        "최종 판정": verdict
      });
    }
  }

  // 중복 항목 정리 및 취약 항목 우선 정렬
  const uniqueFindings = [];
  const seenKeys = new Set();
  for (const item of totalFindings) {
    const key = `${item["점검 항목"]}_${item["노출 여부"]}_${item["마스킹 여부"]}`;
    if (!seenKeys.has(key)) {
      seenKeys.add(key);
      uniqueFindings.push(item);
    }
  }

  uniqueFindings.sort((a, b) => {
    if (a["노출 여부"] === b["노출 여부"]) {
      return (a["마스킹 여부"] === "X" ? -1 : 1);
    }
    return a["노출 여부"] === "O" ? -1 : 1;
  });

  console.clear();
  console.log("%c[+] 인간 시각 모사 점검 완료 (iframe/그리드/인라인 다각도 분석)", "color: #00e676; font-size: 14px; font-weight: bold;");
  console.table(uniqueFindings);

  return uniqueFindings;
})();
```
