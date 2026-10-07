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
(function runComplianceAudit() {
  // 1. 포맷 기반 탐지 패턴 (실제 값 미저장, 매칭 플래그 확인용)
  const REGEX_TARGETS = {
    "주민등록번호": {
      unmasked: /\b\d{6}-[1-4]\d{6}\b/,
      masked: /\b\d{6}-[1-4][*#●]{6}\b/
    },
    "휴대전화번호": {
      unmasked: /\b01[016789]-?\d{3,4}-?\d{4}\b/,
      masked: /\b01[016789]-?[*#●]{3,4}-?\d{4}\b/
    },
    "신용카드번호": {
      unmasked: /\b(?:\d{4}[-\s]?){3}\d{4}\b/,
      masked: /\b\d{4}[-\s]?[*#●]{4}[-\s]?[*#●]{4}[-\s]?\d{4}\b/
    },
    "계좌번호": {
      unmasked: /\b\d{3,6}[-\s]?\d{2,6}[-\s]?\d{3,6}\b/,
      masked: /\b\d{2,4}[-\s]?[*#●]{3,6}[-\s]?\d{3,5}\b/
    }
  };

  // 2. UI 라벨 점검 대상 항목
  const LABEL_TARGETS = [
    "이름", "성명", "계약자명", "주소", "청약번호", "계약번호", "증권번호",
    "상품명", "상품코드", "납입금액", "결제금액", "수신번호", "전화번호",
    "팩스번호", "주민번호", "계좌번호", "은행코드", "이메일"
  ];

  const DOWNLOAD_KEYWORDS = ['다운로드', '엑셀', 'excel', 'download', 'csv', 'export', '내려받기'];

  const findings = [];

  // 1. 메뉴 경로 확인
  const bc = document.querySelector('.breadcrumb, .location, .navi, .menu-path, nav[aria-label="breadcrumb"]');
  let menuPath = document.title || "현재 페이지";
  if (bc) {
    const items = Array.from(bc.querySelectorAll('li, span, a'))
      .map(el => el.innerText.trim())
      .filter(text => text.length > 0);
    if (items.length > 0) menuPath = items.join(' > ');
  }

  // 2. 조회 구분 (단건/개별 vs 대량/목록)
  const hasPaging = !!document.querySelector('.pagination, .paging, [class*="page"]');
  const rowCount = document.querySelectorAll('tr').length;
  const queryType = (hasPaging || rowCount >= 4) ? "대량(목록) 조회" : "개별 조회";

  // 3. 다운로드 버튼 탐지
  let canDownload = "X";
  const clickables = document.querySelectorAll('button, a, input');
  for (const el of clickables) {
    const text = `${el.innerText || ''} ${el.value || ''} ${el.title || ''}`.toLowerCase();
    if (DOWNLOAD_KEYWORDS.some(kw => text.includes(kw))) {
      canDownload = "O";
      break;
    }
  }

  // 4. 정규식 패턴 점검 (값 미저장)
  const bodyText = document.body.innerText || "";
  for (const [itemName, regexes] of Object.entries(REGEX_TARGETS)) {
    const hasUnmasked = regexes.unmasked.test(bodyText);
    const hasMasked = regexes.masked.test(bodyText);

    let isExposed = "X";
    let maskingStatus = "해당 없음";
    let verdict = "해당 없음";

    if (hasUnmasked || hasMasked) {
      isExposed = "O";
      if (hasUnmasked) {
        maskingStatus = "미적용 (노출 위험)";
        verdict = "취약";
      } else {
        maskingStatus = "적용 완료";
        verdict = "양호";
      }
    }

    findings.push({
      "메뉴 경로": menuPath,
      "조회 구분": queryType,
      "다운로드 기능": canDownload,
      "점검 항목": itemName,
      "노출 여부": isExposed,
      "마스킹 여부": maskingStatus,
      "최종 판정": verdict
    });
  }

  // 5. 라벨 기반 점검 (th-td, dt-dd, label-input)
  for (const label of LABEL_TARGETS) {
    let itemExposed = "X";
    let maskingStatus = "해당 없음";
    let verdict = "해당 없음";

    const labelElements = document.querySelectorAll('th, dt, label');
    for (const el of labelElements) {
      const text = el.innerText ? el.innerText.trim() : "";
      if (text.includes(label)) {
        let val = "";
        if (el.tagName === 'TH' || el.tagName === 'DT') {
          const sibling = el.nextElementSibling;
          if (sibling) val = sibling.innerText.trim();
        } else if (el.tagName === 'LABEL' && el.htmlFor) {
          const inputEl = document.getElementById(el.htmlFor);
          if (inputEl) val = inputEl.value || "";
        }

        if (val && val.length > 1 && !val.includes(label)) {
          itemExposed = "O";
          if (/[*#●]/.test(val)) {
            maskingStatus = "적용 완료";
            verdict = "양호";
          } else {
            maskingStatus = "미적용 (확인 필요)";
            verdict = "취약 의심";
          }
          break;
        }
      }
    }

    findings.push({
      "메뉴 경로": menuPath,
      "조회 구분": queryType,
      "다운로드 기능": canDownload,
      "점검 항목": label,
      "노출 여부": itemExposed,
      "마스킹 여부": maskingStatus,
      "최종 판정": verdict
    });
  }

  // 노출된 항목(O) 우선 정렬 후 콘솔 테이블 출력
  findings.sort((a, b) => (b["노출 여부"] === "O") - (a["노출 여부"] === "O"));

  console.clear();
  console.log("%c[+] 개인신용정보 점검 결과 요약", "color: #00ff00; font-size: 14px; font-weight: bold;");
  console.table(findings);

  return findings;
})();
```
