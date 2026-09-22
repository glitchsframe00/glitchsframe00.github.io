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


