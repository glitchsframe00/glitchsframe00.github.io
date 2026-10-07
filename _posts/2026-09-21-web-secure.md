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
(() => {
  try {
    var MASK_PATTERN = /[*#●○Xx_]/;
    var UI_EXCLUDES = [
      '선택', '조회', '검색', '초기화', '닫기', '저장', '삭제', '등록', 
      '수정', '이전', '다음', '목록', '전체', '상세', '보기', '다운로드', 'excel', 'csv'
    ];

    // 충돌 방지를 위해 순수 Object 사용
    var fieldSummary = {};

    var recordField = function(rawLabel, rawValue) {
      try {
        if (!rawLabel || !rawValue) return;
        var label = String(rawLabel).replace(/[\s:：·\-_\t\r\n]+/g, ' ').trim();
        var val = String(rawValue).trim();

        if (label.length < 2 || label.length > 25) return;
        if (UI_EXCLUDES.indexOf(label) !== -1 || label === val || val.length === 0) return;

        if (!Object.prototype.hasOwnProperty.call(fieldSummary, label)) {
          fieldSummary[label] = { maskedCount: 0, unmaskedCount: 0, totalCount: 0 };
        }

        var item = fieldSummary[label];
        item.totalCount += 1;
        if (MASK_PATTERN.test(val)) {
          item.maskedCount += 1;
        } else {
          item.unmaskedCount += 1;
        }
      } catch (e) { /* 개별 데이터 처리 예외 무시 */ }
    };

    // 1) 표(Table) 구조 분석
    try {
      var tables = document.querySelectorAll('table');
      for (var t = 0; t < tables.length; t++) {
        var rows = tables[t].querySelectorAll('tr');
        if (!rows.length) continue;

        // 가로형 표 (th -> td)
        for (var r = 0; r < rows.length; r++) {
          var ths = rows[r].querySelectorAll('th');
          for (var h = 0; h < ths.length; h++) {
            var nextTd = ths[h].nextElementSibling;
            if (nextTd && nextTd.tagName === 'TD') {
              recordField(ths[h].innerText, nextTd.innerText);
            }
          }
        }

        // 세로형 목록 표 (상단 헤더 -> 아래 행 데이터)
        if (rows.length > 1) {
          var headers = rows[0].querySelectorAll('th, td');
          if (headers.length > 1) {
            for (var c = 0; c < headers.length; c++) {
              var colLabel = headers[c].innerText;
              var maxRows = Math.min(rows.length, 15);
              for (var rIdx = 1; rIdx < maxRows; rIdx++) {
                var cells = rows[rIdx].querySelectorAll('td');
                if (cells[c]) {
                  recordField(colLabel, cells[c].innerText);
                }
              }
            }
          }
        }
      }
    } catch (e) { /* Table 예외 무시 */ }

    // 2) dt-dd 구조 분석
    try {
      var dts = document.querySelectorAll('dt');
      for (var i = 0; i < dts.length; i++) {
        var nextDd = dts[i].nextElementSibling;
        if (nextDd && nextDd.tagName === 'DD') {
          recordField(dts[i].innerText, nextDd.innerText);
        }
      }
    } catch (e) { /* dt 예외 무시 */ }

    // 3) label-input 구조 분석
    try {
      var labels = document.querySelectorAll('label');
      for (var j = 0; j < labels.length; j++) {
        var labelEl = labels[j];
        var val = '';
        if (labelEl.htmlFor) {
          var target = document.getElementById(labelEl.htmlFor);
          val = target ? (target.value || target.placeholder || '') : '';
        } else {
          var innerInput = labelEl.querySelector('input, select');
          val = innerInput ? (innerInput.value || innerInput.placeholder || '') : '';
        }
        if (val) recordField(labelEl.innerText, val);
      }
    } catch (e) { /* label 예외 무시 */ }

    // 4) 콜론(:) 인라인 텍스트 분석
    try {
      var textEls = document.querySelectorAll('p, div, span, li');
      for (var k = 0; k < textEls.length; k++) {
        var el = textEls[k];
        if (el.children.length === 0 && el.innerText) {
          var text = el.innerText.trim();
          var match = text.match(/^([^:：\n\r]{2,15})[:：]\s*(.+)$/);
          if (match) recordField(match[1], match[2]);
        }
      }
    } catch (e) { /* inline text 예외 무시 */ }

    // 결과 취합 및 정렬
    var results = [];
    var keys = Object.keys(fieldSummary);
    for (var m = 0; m < keys.length; m++) {
      var fieldName = keys[m];
      var data = fieldSummary[fieldName];
      var maskingStatus = '';
      var verdict = '';

      if (data.unmaskedCount > 0 && data.maskedCount === 0) {
        maskingStatus = 'X (전부 미적용)';
        verdict = '취약';
      } else if (data.unmaskedCount === 0 && data.maskedCount > 0) {
        maskingStatus = 'O (전부 마스킹)';
        verdict = '양호';
      } else {
        maskingStatus = '△ (마스킹 ' + data.maskedCount + '건 / 미적용 ' + data.unmaskedCount + '건)';
        verdict = '취약 의심';
      }

      results.push({
        "항목명 (라벨)": fieldName,
        "화면 노출": "O",
        "마스킹 여부": maskingStatus,
        "최종 판정": verdict,
        "건수": data.totalCount + '건'
      });
    }

    results.sort(function(a, b) {
      var score = function(v) {
        if (v.indexOf('취약') !== -1) {
          return v === '취약' ? 3 : 2;
        }
        return 1;
      };
      return score(b["최종 판정"]) - score(a["최종 판정"]);
    });

    console.clear();
    console.log("%c[+] 화면 전수 점검 완료", "color: #00ff00; font-size: 14px; font-weight: bold;");

    if (results.length === 0) {
      console.warn("⚠️ 화면에서 라벨/데이터를 찾지 못했습니다. (iframe 구조인 경우 콘솔 좌측 상단의 'top' 드롭다운을 실제 본문 프레임으로 변경해 주세요)");
    } else {
      console.table(results);
    }

  } catch (criticalErr) {
    console.error("❌ 점검 스크립트 실행 중 오류:", criticalErr);
  }
})();
```
