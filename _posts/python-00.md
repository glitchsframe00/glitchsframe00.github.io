import asyncio
import re
import pandas as pd
from playwright.async_api import async_playwright
from bs4 import BeautifulSoup

# 1. 포맷 기반 탐지 패턴 (값 추출이 아닌 매칭 여부 판별용)
REGEX_TARGETS = {
    "주민등록번호": {
        "unmasked": re.compile(r"\b\d{6}-[1-4]\d{6}\b"),
        "masked": re.compile(r"\b\d{6}-[1-4][*#●]{6}\b")
    },
    "휴대전화번호": {
        "unmasked": re.compile(r"\b01[016789]-?\d{3,4}-?\d{4}\b"),
        "masked": re.compile(r"\b01[016789]-?[*#●]{3,4}-?\d{4}\b")
    },
    "신용카드번호": {
        "unmasked": re.compile(r"\b(?:\d{4}[-\s]?){3}\d{4}\b"),
        "masked": re.compile(r"\b\d{4}[-\s]?[*#●]{4}[-\s]?[*#●]{4}[-\s]?\d{4}\b")
    },
    "계좌번호": {
        "unmasked": re.compile(r"\b\d{3,6}[-\s]?\d{2,6}[-\s]?\d{3,6}\b"),
        "masked": re.compile(r"\b\d{2,4}[-\s]?[*#●]{3,6}[-\s]?\d{3,5}\b")
    }
}

# 2. 라벨 매핑 점검 항목
LABEL_TARGETS = ["이름", "성명", "주소", "청약번호", "계약번호", "상품명", "납입금액", "결제금액"]


def analyze_page_safely(soup: BeautifulSoup, page_name: str) -> list:
    """실제 값은 저장하지 않고 O/X 플래그와 상태만 추출"""
    findings = []

    # 1. 메뉴 경로 추출
    bc = soup.select_one('.breadcrumb, .location, .navi, .menu-path, nav[aria-label="breadcrumb"]')
    if bc:
        menu_path = " > ".join([el.get_text(strip=True) for el in bc.find_all(['li', 'span', 'a']) if el.get_text(strip=True)])
    else:
        menu_path = soup.title.string.strip() if soup.title and soup.title.string else page_name

    # 2. 조회 구분 (단건/개별 vs 대량/목록)
    has_paging = bool(soup.select('.pagination, .paging, [class*="page"]'))
    row_count = len(soup.find_all('tr'))
    query_type = "대량(목록) 조회" if (has_paging or row_count >= 4) else "개별 조회"

    # 3. 다운로드 버튼 탐지
    download_kws = ['다운로드', '엑셀', 'excel', 'download', 'csv', 'export', '내려받기']
    can_download = "X"
    for btn in soup.find_all(['button', 'a', 'input']):
        btn_text = f"{btn.get_text(strip=True)} {btn.get('value', '')} {btn.get('title', '')}".lower()
        if any(kw in btn_text for kw in download_kws):
            can_download = "O"
            break

    # 4. 정규식 항목 판별 (실제 텍스트 값은 변수에 남기지 않고 매칭 여부만 체크)
    body_text = soup.get_text()

    for item_name, regex_dict in REGEX_TARGETS.items():
        has_unmasked = bool(regex_dict["unmasked"].search(body_text))
        has_masked = bool(regex_dict["masked"].search(body_text))

        if has_unmasked or has_masked:
            is_exposed = "O"
            if has_unmasked:
                masking_status = "미적용 (노출 위험)"
                verdict = "취약"
            else:
                masking_status = "적용 완료"
                verdict = "양호"
        else:
            is_exposed = "X"
            masking_status = "해당 없음"
            verdict = "해당 없음"

        findings.append({
            "메뉴 경로": menu_path,
            "조회 구분": query_type,
            "다운로드 기능": can_download,
            "점검 항목": item_name,
            "노출 여부": is_exposed,
            "마스킹 여부": masking_status,
            "최종 판정": verdict
        })

    # 5. 라벨 기반 항목 판별 (이름, 주소, 납입금액 등)
    # th-td, dt-dd, label-input 구조 탐색
    for target_label in LABEL_TARGETS:
        item_exposed = "X"
        masking_status = "해당 없음"
        verdict = "해당 없음"

        for cell in soup.find_all(['th', 'dt', 'label']):
            cell_text = cell.get_text(strip=True)
            if target_label in cell_text:
                val = ""
                if cell.name == 'th':
                    sibling = cell.find_next_sibling('td')
                    if sibling:
                        val = sibling.get_text(strip=True)
                elif cell.name == 'dt':
                    sibling = cell.find_next_sibling('dd')
                    if sibling:
                        val = sibling.get_text(strip=True)
                elif cell.name == 'label' and cell.get('for'):
                    input_el = soup.find(id=cell.get('for'))
                    if input_el:
                        val = input_el.get('value', '')

                if val and len(val) > 1 and target_label not in val:
                    item_exposed = "O"
                    # 마스킹 기호 포함 여부만 판정
                    if any(c in val for c in ['*', '#', '●']):
                        masking_status = "적용 완료"
                        verdict = "양호"
                    else:
                        masking_status = "미적용 (확인 필요)"
                        verdict = "취약 의심"
                    break

        findings.append({
            "메뉴 경로": menu_path,
            "조회 구분": query_type,
            "다운로드 기능": can_download,
            "점검 항목": target_label,
            "노출 여부": item_exposed,
            "마스킹 여부": masking_status,
            "최종 판정": verdict
        })

    return findings


async def run_compliance_audit():
    async with async_playwright() as p:
        # 로그인된 디버깅 브라우저에 연결 (CDP)
        try:
            browser = await p.chromium.connect_over_cdp("http://localhost:9222")
        except Exception:
            print("❌ 디버깅 모드로 실행된 브라우저(포트 9222)를 찾을 수 없습니다.")
            print("실행 명령: chrome.exe --remote-debugging-port=9222 --user-data-dir=\"C:\\chrometemp\"")
            return

        context = browser.contexts[0]
        page = context.pages[0]
        print(f"[+] 브라우저 연결 성공: {await page.title()}")

        # 좌측 LNB/사이드바 메뉴 링크 요소 탐색 (시스템 UI에 맞게 클래스명 조절 가능)
        menu_elements = await page.query_selector_all(".lnb a, .menu a, .sidebar a, nav a")
        
        menu_list = []
        for el in menu_elements:
            text = (await el.inner_text()).strip()
            href = await el.get_attribute("href")
            if text and href and not href.startswith("javascript:void(0)"):
                menu_list.append(text)

        # 중복 제거
        menu_list = list(dict.fromkeys(menu_list))
        total_results = []

        print(f"[+] 총 {len(menu_list)}개 메뉴 자동 점검을 시작합니다.\n")

        # 메뉴가 따로 없으면 현재 단일 페이지만 점검
        if not menu_list:
            soup = BeautifulSoup(await page.content(), "html.parser")
            total_results.extend(analyze_page_safely(soup, await page.title()))
        else:
            for name in menu_list:
                print(f"[*] 점검 중: [{name}]")
                try:
                    await page.click(f"text={name}", timeout=5000)
                    # WAF/FDS 탐지 방지 및 렌더링을 위한 2초 대기
                    await page.wait_for_load_state("networkidle")
                    await asyncio.sleep(2)

                    soup = BeautifulSoup(await page.content(), "html.parser")
                    results = analyze_page_safely(soup, name)
                    total_results.extend(results)

                except Exception as e:
                    print(f"    └ 접근 건너뜀 ({name}): {e}")
                    continue

        # 엑셀 파일 저장
        if total_results:
            df = pd.DataFrame(total_results)
            output_filename = "개인신용정보처리시스템_현황_점검결과.xlsx"
            
            # 노출 여부가 'O'인 항목을 우선적으로 볼 수 있도록 정렬
            df.sort_values(by=["메뉴 경로", "노출 여부"], ascending=[True, False], inplace=True)
            df.to_excel(output_filename, index=False)
            
            print("\n" + "="*60)
            print("          ✅ 점검 완료 (개인정보 원본 미저장 안전 모드)")
            print("="*60)
            print(f"■ 보고서 저장 완료: {output_filename}")
            print("■ 기록 항목      : 메뉴 경로, 조회구분, 다운로드기능, 항목명, 노출여부, 마스킹여부, 최종판정")
            print("■ 원본 데이터    : 완전 배제 (0건 기록)")
            print("="*60)


if __name__ == "__main__":
    asyncio.run(run_compliance_audit())
