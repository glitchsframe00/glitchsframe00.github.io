```javascript
(function runComplianceAudit() {
  // 1. 포맷 기반 탐지 패턴 (* 마스킹 기준)
  const REGEX_TARGETS = {
    "주민등록번호": {
      unmasked: /\b\d{6}-[1-4]\d{6}\b/,
      masked: /\b\d{6}-[1-4*]\*{6}\b/
    },
    "휴대전화번호": {
      unmasked: /\b01[016789]-?\d{3,4}-?\d{4}\b/,
      masked: /\b01[016789]-?(?:\*{3,4}-?\d{4}|\d{3,4}-?\*{4})\b/
    },
    "신용카드번호": {
      unmasked: /\b(?:\d{4}[-\s]?){3}\d{4}\b/,
      masked: /\b\d{4}[-\s]?\*{4}[-\s]?\*{4}[-\s]?\d{4}\b/
    },
    "계좌번호": {
      unmasked: /\b\d{3,6}[-\s]?\d{2,6}[-\s]?\d{3,6}\b/,
      masked: /\b\d{2,4}[-\s]?\*{3,6}[-\s]?\d{3,5}\b/
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
  const clickables = document.querySelectorAll('button, a
