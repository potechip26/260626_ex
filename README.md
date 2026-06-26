# 2026-06-26 학습 정리

오늘 학습한 내용은 **Java 예외 처리(Exception Handling)**, **Java 파일 I/O 및 NIO.2 API**, 그리고 **HttpClient를 이용한 외부 API(Naver 이미지 검색) 연동 및 이미지 다운로드 실습**입니다.

---

## 1. Java 예외 처리 (Exception Handling)

Java의 예외는 크게 **Checked Exception**과 **Unchecked Exception**으로 나뉩니다.

### Checked vs Unchecked Exception
*   **Checked Exception (`Exception` 상속)**
    *   컴파일 시점에 예외 처리를 강제합니다.
    *   `try-catch` 블록으로 직접 처리하거나, `throws` 키워드를 사용하여 호출부로 예외를 던져야(위임해야) 합니다.
    *   대표적인 예: `IOException`, `InterruptedException`, `SQLException` 등.
*   **Unchecked Exception (`RuntimeException` 상속)**
    *   컴파일 시점에 예외 처리를 강제하지 않습니다.
    *   개발자의 실수로 발생하는 예외가 많으며, 필요 시에 선택적으로 예외 처리를 합니다.
    *   대표적인 예: `NullPointerException`, `ArithmeticException`, `IllegalArgumentException` 등.

### 예외 다중 캐치 (Multi-Catch)
*   여러 예외를 하나의 catch 블록에서 동일하게 처리하고 싶을 때 `|` 연산자를 활용합니다.
    ```java
    try {
        // 예외 발생 가능 코드
    } catch (CustomException | UncheckedCustomException e) {
        System.out.println("예외 메시지: " + e.getMessage());
    }
    ```
*   **주의 사항**: 캐치 블록의 순서는 **좁은 범위(자식 클래스)에서 넓은 범위(부모 클래스)**의 순서로 작성해야 합니다. 부모 클래스가 위에 있으면 자식 클래스 캐치 블록에는 도달할 수 없어 컴파일 에러가 발생합니다.

### 예외 전환 패턴 (Exception Translation Pattern)
*   하위 레이어(`AppProvider`)의 구체적인 Checked Exception을 상위 레이어(`AppService`)에서 잡아서 `RuntimeException`으로 전환(Wrapping)하여 다시 던집니다.
*   이때 원인이 되는 예외(Cause)를 `new RuntimeException(e)` 형태로 함께 넘겨주어 디버깅 시 Call Stack 추적이 가능하도록 만듭니다.
*   이를 통해 프레젠테이션 레이어 또는 컨트롤러(Client 영역) 등의 최상위 단계에서 일괄적으로 예외 처리를 단순화할 수 있습니다.

---

## 2. Java 입출력 (I/O) & 리소스 관리

### 리소스 자동 반납 (`try-with-resources`)
*   기존의 `finally` 블록을 사용한 자원 반납(`close()`)은 변수의 스코프 제한 문제, 코드의 복잡성 및 `NullPointerException` 유발 등의 단점이 있었습니다.
*   **try-with-resources (Java 7+)** 구문을 사용하면 `AutoCloseable`을 구현한 리소스를 try 괄호 안에 선언하여, 블록을 벗어날 때 자동으로 `close()`가 호출되도록 안전하게 관리할 수 있습니다.

### 스트림의 분류
1.  **바이트 스트림 (Byte Stream)**
    *   `InputStream` / `OutputStream` (`FileInputStream`, `FileOutputStream`)
    *   이미지, 동영상 등 바이너리 데이터를 다룰 때 주로 사용됩니다.
    *   텍스트 데이터를 전송할 경우 인코딩 정보(예: `StandardCharsets.UTF_8`)를 바이트 배열(`getBytes()`)로 변환하여 처리해야 합니다.
2.  **문자 스트림 (Character Stream)**
    *   `Reader` / `Writer` (`FileReader`, `FileWriter`)
    *   문자/텍스트 데이터(특히 한글 등 다국어)를 처리하는 데 특화되어 있어 글자 깨짐 없이 다룰 수 있습니다.
3.  **버퍼 스트림 & NIO.2 API (Java 7+)**
    *   `BufferedReader` / `BufferedWriter`
    *   내부 버퍼를 사용하여 디스크 I/O 횟수를 획득함으로써 입출력 성능을 대폭 향상시킵니다.
    *   현대적인 `java.nio.file.Path`, `Paths.get()`, `Files.newBufferedWriter()` / `Files.newBufferedReader()` API를 함께 사용하여 파일을 보다 편리하고 직관적으로 관리할 수 있습니다.
    *   줄 바꿈 시 운영체제에 종속적이지 않은 `writer.newLine()`을 활용합니다.

---

## 3. Naver 이미지 검색 API 연동 및 이미지 다운로드 실습

`java.net.http.HttpClient`를 활용하여 네이버 이미지 검색 API에 요청을 보내고, 결과를 파싱하여 실제 이미지 파일을 로컬에 다운로드하는 실습을 진행했습니다.

### 주요 구현 포인트
1.  **HTTP API 요청 전송**:
    *   `HttpClient`, `HttpRequest`, `HttpResponse` API를 활용하여 HTTP GET 요청을 비동기/동기로 전송.
    *   `X-Naver-Client-Id`, `X-Naver-Client-Secret` 헤더를 통해 네이버 API 인증을 처리하고, 인증 정보는 소스코드 하드코딩 대신 환경변수(`System.getenv()`)에서 안전하게 로드.
2.  **URL 인코딩**:
    *   검색 키워드에 한글(예: `"창억떡"`)이 포함되는 경우 깨짐을 방지하기 위해 `URLEncoder.encode(keyword, StandardCharsets.UTF_8)`로 인코딩 처리.
3.  **정규표현식을 이용한 파싱**:
    *   JSON 응답 본문에서 `"link":"(.*?)"` 패턴을 추출하기 위해 Java의 `Pattern`과 `Matcher` 및 Java 9+의 `MatchResult.group(1)`을 활용.
    *   JSON 응답 문자열의 이스케이프 문자(`\/`)를 올바른 URL 형태로 가공(`replace("\\/", "/")`).
4.  **파일 다운로드**:
    *   `HttpResponse.BodyHandlers.ofFile(path)`를 사용하여 네트워크 입력 스트림에서 직접 지정한 로컬 파일 경로(`Path`)로 바이트 데이터를 바로 저장.
    *   중복 파일명을 피하기 위해 검색어 접두사, 현재 타임스탬프(`System.currentTimeMillis()`) 및 URL에서 추출한 이미지 확장자를 조합하여 파일명 생성.
    *   검색 URL 리스트는 `ZonedDateTime`과 `DateTimeFormatter`를 사용하여 한국 시간대 기준 `yyyy-MM-dd_HHmmss` 포맷으로 텍스트 파일에 별도 기록.
