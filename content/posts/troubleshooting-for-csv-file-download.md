---
title: "CSV 다운로드 기능 구현 시 발생하는 주요 문제와 해결 방안"
description: "CSV 파일 다운로드 기능 구현 시 흔히 겪는 3가지 문제(엑셀 한글 깨짐, 프론트엔드 API 호출 시 파일 손상, 특수 문자로 인한 형식 오류)의 원인을 분석하고, BOM 추가, Blob 타입 응답, RFC 4180 표준을 준수한 특수 문자 처리 등 명확한 해결 방안을 제시합니다."
date: 2025-01-18T13:47:47+09:00
tags: ["troubleshooting", "csv", "backend", "frontend"]
---

## 들어가며: 간단해 보이지만 까다로운 CSV 다운로드

'CSV 파일 다운로드' 기능은 많은 웹 애플리케이션에서 흔히 볼 수 있는 기능입니다. 간단해 보이지만, 막상 구현하다 보면 문자 인코딩, 프론트엔드와의 연동, 데이터 형식 문제 등 예상치 못한 여러 복병을 만나게 됩니다.

이 글에서는 Spring Boot와 Kotlin 환경에서 CSV 다운로드 기능을 구현하며 겪었던 대표적인 세 가지 문제와 그 해결 과정을 공유하여, 다른 개발자분들이 비슷한 문제에 부딪혔을 때 빠르고 효과적으로 해결하는 데 도움을 드리고자 합니다.

## 문제 1: 엑셀에서 한글이 깨지는 인코딩 문제

-   **현상**: 서버에서 UTF-8로 인코딩된 CSV 파일을 생성하여 다운로드했는데, 유독 Microsoft Excel에서 파일을 열면 한글이 `ê°€ë‚˜ë‹¤`와 같이 깨져 보입니다. (메모장이나 다른 텍스트 에디터에서는 정상적으로 보입니다.)
-   **원인**: 많은 버전의 Excel(특히 Windows용)은 파일의 인코딩을 자동으로 감지하는 능력이 부족합니다. 파일 맨 앞에 어떤 인코딩을 사용했는지 알려주는 특별한 '힌트'가 없으면, Excel은 파일을 시스템의 기본 인코딩(예: ANSI)으로 해석하려고 시도하여 UTF-8로 작성된 한글이 깨지게 됩니다.
-   **해결책**: 파일의 가장 앞부분에 **BOM(Byte Order Mark)** 을 추가합니다. BOM은 눈에 보이지 않는 특수한 바이트 시퀀스(`EF BB BF`)로, Excel을 비롯한 여러 프로그램에 "이 파일은 UTF-8로 인코딩되었습니다"라고 알려주는 역할을 합니다.

### 백엔드 해결 코드 (Kotlin/Spring)

CSV 데이터를 `OutputStream`에 쓰기 전에, BOM 바이트 배열을 먼저 써주면 됩니다.

```kotlin
// Controller 또는 Service
fun downloadCsv(response: HttpServletResponse) {
    response.contentType = "text/csv; charset=UTF-8"
    response.setHeader("Content-Disposition", "attachment; filename=\"data.csv\"")

    // 1. UTF-8 BOM을 정의합니다.
    val bom = byteArrayOf(0xEF.toByte(), 0xBB.toByte(), 0xBF.toByte())

    response.outputStream.use { outputStream ->
        // 2. CSV 데이터를 쓰기 전에 BOM을 먼저 씁니다.
        outputStream.write(bom)

        // 3. 실제 CSV 데이터를 씁니다.
        val writer = outputStream.bufferedWriter(Charsets.UTF_8)
        writer.write("이름,나이\n")
        writer.write("홍길동,30\n")
        writer.flush()
    }
}
```

## 문제 2: 프론트엔드 API 호출 시 파일이 깨지는 문제

-   **현상**: 백엔드 API 엔드포인트 URL을 브라우저 주소창에 직접 입력하여 다운로드하면 파일이 정상이지만, 프론트엔드(React, Vue 등)에서 JavaScript의 `axios`나 `fetch`로 API를 호출하여 다운로드하면 파일이 깨지거나 열리지 않습니다.
-   **원인**: 기본적으로 JavaScript HTTP 클라이언트는 서버의 응답을 텍스트(문자열)로 해석하려고 시도합니다. 이 과정에서 BOM을 포함한 순수한 바이트 스트림이 텍스트로 변환되면서 데이터가 손상될 수 있습니다.
-   **해결책**: API를 호출할 때, 응답 타입을 텍스트가 아닌 **`blob` (Binary Large Object)** 으로 명시해야 합니다. `blob` 타입은 응답을 바이너리 데이터 그대로 받아오므로, 데이터 손상 없이 파일을 처리할 수 있습니다.

### 프론트엔드 해결 코드 (JavaScript/axios)

```javascript
import axios from 'axios';

async function downloadCsvFile() {
    try {
        const response = await axios.get('/api/download-csv', {
            // 1. 응답 타입을 'blob'으로 지정합니다.
            responseType: 'blob',
        });

        // 2. 응답받은 blob 데이터로 다운로드 링크를 생성합니다.
        const url = window.URL.createObjectURL(new Blob([response.data]));
        const link = document.createElement('a');
        link.href = url;
        link.setAttribute('download', 'data.csv'); // 다운로드될 파일 이름 지정
        document.body.appendChild(link);
        
        // 3. 링크를 클릭하여 파일 다운로드를 실행합니다.
        link.click();

        // 다운로드 후 생성된 링크와 URL을 정리합니다.
        link.parentNode.removeChild(link);
        window.URL.revokeObjectURL(url);

    } catch (error) {
        console.error('CSV 다운로드 중 오류 발생:', error);
    }
}
```

## 문제 3: 데이터에 포함된 특수 문자로 형식이 깨지는 문제

-   **현상**: CSV 데이터 필드 값에 쉼표(`,`), 큰따옴표(`"`), 또는 줄바꿈 문자가 포함되자, Excel에서 열었을 때 열이 밀리거나 줄이 합쳐지는 등 전체적인 형식이 깨집니다.
-   **원인**: CSV(Comma-Separated Values) 형식은 쉼표를 열 구분자로, 줄바꿈을 행 구분자로 사용합니다. 따라서 데이터 내부에 이러한 구분자가 포함되면, 이를 데이터의 일부가 아닌 형식의 일부로 오해하게 됩니다.
-   **해결책**: CSV 표준 명세인 **RFC 4180** 의 규칙에 따라 특수 문자를 이스케이프(escape) 처리해야 합니다.
    1.  필드에 **쉼표(`,`), 큰따옴표(`"`), 또는 줄바꿈 문자** 가 포함된 경우, 해당 필드 전체를 **큰따옴표(`"`)로 감싸야 합니다.**
    2.  필드 내부에 원래 큰따옴표(`"`)가 있었다면, 이를 **두 개의 연속된 큰따옴표(`""`)** 로 바꿔주어야 합니다.

### 백엔드 해결 코드 (Kotlin)

```kotlin
fun escapeCsvField(field: String?): String {
    if (field == null) return ""
    // 필드에 특수문자가 포함되어 있는지 확인
    val needsQuotes = field.contains(",") || field.contains("\"") || field.contains("\n")

    if (needsQuotes) {
        // 큰따옴표를 두 개로 치환하고, 전체를 큰따옴표로 감싼다.
        return "\"${field.replace("\"", "\"\"")}\"";
    }
    return field
}

// CSV 행 생성 시 예시
val fields = listOf("홍길동", "메모: \"중요\", 특이사항 없음")
val csvLine = fields.joinToString(separator = ",") { escapeCsvField(it) }
// 결과: "홍길동","메모: ""중요"", 특이사항 없음"
```

### 더 나은 해결책: 검증된 CSV 라이브러리 사용

위와 같이 직접 이스케이프 로직을 구현하는 것도 가능하지만, 모든 엣지 케이스를 완벽하게 처리하기는 까다롭습니다. 따라서 **Apache Commons CSV** 나 **OpenCSV** 와 같은 검증된 CSV 라이브러리를 사용하는 것이 훨씬 안전하고 효율적입니다. 이 라이브러리들은 RFC 4180 표준을 완벽하게 지원하므로, 개발자는 특수 문자 처리에 대한 걱정 없이 비즈니스 로직에만 집중할 수 있습니다.

## 결론: 안정적인 CSV 다운로드 기능을 위한 체크리스트

성공적인 CSV 다운로드 기능을 구현하기 위해 다음 세 가지를 반드시 확인해야 합니다.

1.  **백엔드**: Excel과의 호환성을 위해 파일 시작 부분에 **UTF-8 BOM** 을 추가했는가?
2.  **백엔드**: 데이터에 포함될 수 있는 쉼표, 큰따옴표, 줄바꿈 문자를 **RFC 4180 표준에 맞게 이스케이프 처리** 했는가? (라이브러리 사용을 적극 권장)
3.  **프론트엔드**: JavaScript로 파일을 다운로드할 때, API 응답 타입을 **`blob`** 으로 지정했는가?

위의 체크리스트를 따르면, 사용자가 어떤 환경에서든 데이터를 안정적으로 받아볼 수 있는 견고한 CSV 다운로드 기능을 구현할 수 있을 것입니다.