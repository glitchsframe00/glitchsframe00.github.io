[수정 후 (Secure Code)]
> 오픈소스 라이브러리(Apache PDFBox)를 활용하여 업로드 시 PDF 내부의 자바스크립트 실행 트리거를 검증 및 무력화하는 시큐어 코딩 예시입니다.
> 
import org.apache.pdfbox.pdmodel.PDDocument;
import org.apache.pdfbox.pdmodel.interactive.action.PDAction;
import org.apache.pdfbox.pdmodel.interactive.action.PDActionJavaScript;

public void uploadPdfFile(MultipartFile file) throws Exception {
    String fileName = file.getOriginalFilename();
    
    // 1. 확장자 화이트리스트 검증
    if (fileName == null || !fileName.toLowerCase().endsWith(".pdf")) {
        throw new IllegalArgumentException("허용되지 않은 파일 형식입니다.");
    }
    
    // 2. PDF 내부 자바스크립트 및 자동 실행 구문 검증
    try (PDDocument document = PDDocument.load(file.getInputStream())) {
        // PDF 실행 트리거(OpenAction) 검사
        PDAction openAction = document.getDocumentCatalog().getOpenAction();
        if (openAction instanceof PDActionJavaScript) {
            throw new SecurityException("자바스크립트가 포함된 PDF 문서는 업로드할 수 없습니다.");
        }
        
        // 문서 내 자바스크립트 명명 항목 검사
        if (document.getDocumentCatalog().getNames() != null && 
            document.getDocumentCatalog().getNames().getJavaScript() != null) {
            throw new SecurityException("문서 내부 스크립트 실행 요소가 감지되었습니다.");
        }
        
        // 3. 안전한 임의 파일명(UUID)으로 저장
        File targetFile = new File(UPLOAD_DIR, UUID.randomUUID().toString() + ".pdf");
        file.transferTo(targetFile);
    }
}

"""
file_name = "PDF_XSS_Vulnerability_Guide.md"
with open(file_name, "w", encoding="utf-8") as f:
f.write(markdown_content.strip() + "\n")
print(f"File created successfully: {file_name}")

```text?code_stdout&code_event_index=1
File created successfully: PDF_XSS_Vulnerability_Guide.md


GitHub(깃허브) 리포지토리나 위키에 바로 업로드하여 렌더링할 수 있도록 텍스트 누락 없이 테이블과 코드 블록의 가독성을 살려 작성했습니다. 아래 본문 내용을 복사해서 쓰실 수도 있습니다.
# PDF 내부 자바스크립트 삽입(PDF XSS) 취약점 조치 가이드

---

### 5.2.1 취약점 설명

| 구분 | 통제 구분 | 평가 항목 ID | 위험도 |
| :--- | :--- | :--- | :--- |
| **기술적 보안(일반공통)** | 서비스 보호 | WEB-SER-002 (또는 인라인 XSS/파일 업로드 ID) | **중 (3~4)** |

#### 취약점 설명
첨부파일 업로드 기능을 제공하는 웹 애플리케이션에서 파일 확장자 및 MIME 타입 검증 외에 파일 내부 콘텐츠 검증이 미흡할 경우, 공격자가 자바스크립트 자동 실행 구문(`/OpenAction`, `/JavaScript` 등)이 포함된 악성 PDF 문서를 업로드하여 열람자의 브라우저 권한 내에서 악성 스크립트를 실행(XSS)시킬 수 있는 취약점

---

### 5.2.2 조치 방안

| 구분 | 내용 |
| :--- | :--- |
| **조치 방안** | <ul><li>**내부 스트림 정제(Sanitization)**: PDF 파싱 라이브러리(Apache PDFBox, iText 등)를 사용하여 파일 업로드 시 문서 내 악성 액션(`/OpenAction`, `/AA`) 및 자바스크립트(`/JavaScript`, `/JS`) 객체를 탐지하여 업로드를 차단하거나 해당 객체를 제거 후 저장</li><li>**강제 다운로드 처리**: 첨부파일 다운로드 응답 시 `Content-Disposition: attachment; filename="..."` 헤더를 설정하여 브라우저 내장 뷰어에서 문서가 직접 렌더링/실행되지 않도록 강제</li><li>**다운로드 도메인 격리**: 세션 및 인증 쿠키를 공유하지 않는 별도의 정적 콘텐츠 전용 도메인(Storage/CDN)을 통해 파일 다운로드를 제공하여 메인 도메인의 세션 탈취 위험 최소화</li></ul> |

---

### 소스코드 수정 가이드 (Java 서블릿 / 스프링 예시)

보고서 양식의 소스코드 블록에는 개발자가 직관적으로 이해할 수 있도록 **[수정 전]**과 **[수정 후]**를 나누어 기재합니다.

#### [수정 전 (Vulnerable Code)]
> 단순히 파일 확장자나 크기만 검사하고 파일 내부의 스크립트 객체를 검증하지 않은 채 그대로 저장하는 취약한 코드입니다.

```java
// 파일 업로드 처리 메서드
public void uploadPdfFile(MultipartFile file) throws Exception {
    String fileName = file.getOriginalFilename();
    
    // 단순 확장자만 검증 후 저장 (PDF 내부의 악성 JS 미검증)
    if (fileName != null && fileName.toLowerCase().endsWith(".pdf")) {
        File targetFile = new File(UPLOAD_DIR, UUID.randomUUID().toString() + ".pdf");
        file.transferTo(targetFile);
    } else {
        throw new IllegalArgumentException("PDF 파일만 업로드 가능합니다.");
    }
}

[수정 후 (Secure Code)]
> 오픈소스 라이브러리(Apache PDFBox)를 활용하여 업로드 시 PDF 내부의 자바스크립트 실행 트리거를 검증 및 무력화하는 시큐어 코딩 예시입니다.
> 
import org.apache.pdfbox.pdmodel.PDDocument;
import org.apache.pdfbox.pdmodel.interactive.action.PDAction;
import org.apache.pdfbox.pdmodel.interactive.action.PDActionJavaScript;

public void uploadPdfFile(MultipartFile file) throws Exception {
    String fileName = file.getOriginalFilename();
    
    // 1. 확장자 화이트리스트 검증
    if (fileName == null || !fileName.toLowerCase().endsWith(".pdf")) {
        throw new IllegalArgumentException("허용되지 않은 파일 형식입니다.");
    }
    
    // 2. PDF 내부 자바스크립트 및 자동 실행 구문 검증
    try (PDDocument document = PDDocument.load(file.getInputStream())) {
        // PDF 실행 트리거(OpenAction) 검사
        PDAction openAction = document.getDocumentCatalog().getOpenAction();
        if (openAction instanceof PDActionJavaScript) {
            throw new SecurityException("자바스크립트가 포함된 PDF 문서는 업로드할 수 없습니다.");
        }
        
        // 문서 내 자바스크립트 명명 항목 검사
        if (document.getDocumentCatalog().getNames() != null && 
            document.getDocumentCatalog().getNames().getJavaScript() != null) {
            throw new SecurityException("문서 내부 스크립트 실행 요소가 감지되었습니다.");
        }
        
        // 3. 안전한 임의 파일명(UUID)으로 저장
        File targetFile = new File(UPLOAD_DIR, UUID.randomUUID().toString() + ".pdf");
        file.transferTo(targetFile);
    }
}




파일 내용 검증 (File Content Validation)
​파일 내용에는 악성, 부적절 또는 불법 데이터가 포함될 수 있습니다. 예상되는 파일 형식에 따라 특화된 내용 검증을 적용할 수 있습니다.
​이미지: 이미지 재작성(Image rewriting) 기술을 적용하여 이미지 내에 삽입된 모든 악성 콘텐츠를 파괴할 수 있으며, 이는 무작위화(Randomization) 등을 통해 수행될 수 있습니다.
​Microsoft 문서: Apache POI 등을 활용하여 업로드된 문서를 검증합니다.
​ZIP 파일: 온갖 유형의 파일이 포함될 수 있고 관련 공격 벡터가 매우 다양하므로 업로드를 권장하지 않습니다.
​파일 업로드 서비스는 사용자가 불법 콘텐츠를 신고하고 저작권자가 침해를 신고할 수 있는 창구를 제공해야 합니다.
​자원이 충분하다면 파일을 외부에 공개하기 전에 샌드박스 환경에서 수동 파일 검토를 진행해야 합니다.
​검토 과정에 자동화를 추가하는 것도 도움이 될 수 있습니다(VirusTotal API를 통한 악성 해시 검사, ASP.NET Drawing Library 등 프레임워크를 통한 원시 콘텐츠 유형 검증 등). 단, 공개 서비스를 이용할 경우 데이터 유출 위협 및 정보 수집 가능성에 주의해야 합니다.



※ 참고 규정
- OWASP (Cheat Sheet Series: File Upload Cheat Sheet - File Content Validation & Serving Files Securely)
- 행정안전부·한국인터넷진흥원 (소프트웨어 개발보안 가이드 - 크로스사이트 스크립트, 위험한 형식 파일 업로드)

* 파일 내용 검증 (File Content Validation)
  - PDF 문서 내부 파싱 및 악성 스크립트 검증:
    문서 파일(PDF) 업로드 시 파일 헤더(Magic Bytes)뿐만 아니라 내부 구조를 검증하여, 문서 열람 시 자동 실행되는 트리거(/OpenAction, /AA) 및 악성 자바스크립트(/JavaScript, /JS) 객체 포함 여부를 서버 측에서 확인하고 차단하거나 해당 요소를 제거(Sanitize) 후 저장

* 안전한 파일 서빙 (Serving Files Securely)
  - 브라우저 인라인 렌더링 방지 (강제 다운로드 처리):
    사용자가 파일을 열람하거나 다운로드할 때 웹 브라우저 내장 뷰어에서 자바스크립트가 직접 실행(XSS)되지 않도록, HTTP 응답 헤더에 "Content-Disposition: attachment; filename=파일명" 설정을 적용하여 강제 다운로드 유도
  - 콘텐츠 타입 스니핑 방지:
    브라우저가 파일 형식을 임의로 해석하여 실행하는 것을 차단하기 위해 응답 헤더에 "X-Content-Type-Options: nosniff" 설정 필수 적용
  - 서빙 도메인(스토리지) 격리:
    업로드된 파일은 메인 서비스 도메인과 분리된 별도의 정적 전용 도메인(별도 CDN 또는 스토리지 도메인)을 통해 제공하여 악성 스크립트가 실행되더라도 메인 애플리케이션의 인증 쿠키/세션 탈취 위험을 원천 격리





현재 작성하신 "실행 권한 제거(Apache HTTP Server, Nginx 예시)" 설정만으로는 이번 취약점(PDF XSS)이 전혀 조치되지 않습니다.
그 이유와 추가해야 할 부분을 단계별로 짚어드리겠습니다.
1. 왜 조치가 불충분할까?
 * 서버사이드 vs 클라이언트사이드 스크립트 차이:
   * 작성하신 설정은 jsp, php, pl, cgi, asp, js, sh 같은 웹 서버가 해석하여 실행하는 서버사이드 스크립트(웹셸)의 실행을 막는 룰입니다.
   * 하지만 이번 공격 파일은 확장자가 .pdf이고, 스크립트가 실행되는 위치는 서버가 아니라 문서를 열람하는 피해자의 웹 브라우저(클라이언트)입니다.
 * 허용 확장자 매칭:
   * 설정 내 <FilesMatch "\.(png|jpg|jpeg|gif|txt)$"> 또는 PDF 허용 규칙을 두더라도, 정상 확장자 파일로 인정되어 파일 다운로드/접근 경로가 그대로 열립니다.
   * 사용자가 해당 PDF 링크를 브라우저로 클릭하면 웹 서버는 파일을 그대로 전송하고, 브라우저 내장 뷰어에서 /OpenAction과 자바스크립트가 문제없이 실행됩니다.
2. 웹 서버 설정 칸에 진짜로 들어가야 하는 내용
해당 "Apache HTTP Server 예시 / Nginx 예시" 칸에는 실행 권한 차단 문법 대신, 브라우저 인라인 실행을 차단하는 응답 헤더 설정이 들어가야 취약점이 완벽히 방어됩니다.
Apache HTTP Server 예시 대체안
# PDF 파일 다운로드 시 브라우저 내 직접 실행 방지 (강제 다운로드)
<FilesMatch "\.(pdf|PDF)$">
    Header set Content-Disposition "attachment"
    Header set X-Content-Type-Options "nosniff"
</FilesMatch>

Nginx 예시 대체안
# PDF 파일 다운로드 시 브라우저 내 직접 실행 방지 (강제 다운로드)
location ~* \.pdf$ {
    add_header Content-Disposition "attachment";
    add_header X-Content-Type-Options "nosniff";
}

기존에 적어두신 '실행 권한 제거'는 일반 웹셸 공격을 방어하는 보조 대책으로는 맞지만, 이번 보고서의 본질인 PDF 스크립트 실행(XSS) 취약점을 직접 조치하는 방안은 위와 같은 "Content-Disposition 헤더 설정"과 "애플리케이션(Java/PDFBox) 내부 객체 검증"입니다.
