---
title: "CSV 파일을 맥용 엑셀에서 볼 때 한글이 깨지는 문제"
description: "CSV 파일 다운로드 기능을 구현할 때 발생하는 주요 문제들과 그 해결 방법을 다룹니다. 주요 내용으로는 한글 깨짐 문제 해결을 위한 BOM 문자 추가, 프론트엔드에서의 다운로드 문제 해결을 위한 'blob' 타입 지정, CSV 형식 오류 해결을 위한 특수 문자 처리 방법 등이 포함됩니다."
date: 2025-01-18T13:47:47+09:00
url: "/troubleshooting-for-csv-file-download/"
tags: [TIL]
---

## 스프링 부트와 코틀린을 사용한 CSV 파일 다운로드 트러블슈팅

스프링 부트와 코틀린을 사용하여 CSV 파일 다운로드 기능을 구현하는 과정에서 발생한 여러 문제와 그 해결 방법에 대해 설명하겠습니다.

### 1. 한글 깨짐 문제

**문제**: CSV 파일을 다운로드하여 맥용 엑셀에서 열었을 때 한글이 깨지는 현상이 발생했습니다.

**해결**:
- API 서버에서 CSV 파일을 생성할 때 BOM(Byte Order Mark) 문자를 추가했습니다.
- BOM은 UTF-8로 인코딩된 파일의 시작 부분에 추가되는 특수한 문자 시퀀스로, 일부 애플리케이션(특히 Microsoft Excel)이 파일의 인코딩을 올바르게 인식하도록 돕습니다.

**코드 예시**:
```kotlin
val bom = byteArrayOf(0xEF.toByte(), 0xBB.toByte(), 0xBF.toByte())
outputStream.write(bom)
// CSV 내용 작성 코드
```

### 2. 프론트엔드에서의 다운로드 문제

**문제**: API 엔드포인트를 브라우저에 직접 입력하여 다운로드할 때는 문제가 없었지만, 프론트엔드에서 API를 호출하여 다운로드할 때 여전히 파일이 깨지는 현상이 발생했습니다.

**해결**:
- 프론트엔드에서 API를 호출할 때, 응답 타입을 'blob'으로 지정했습니다.
- 'blob' 타입은 이진 데이터를 그대로 받아올 수 있게 해주어, 텍스트 인코딩 문제를 방지합니다.

**코드 예시 (JavaScript)**:
```javascript
axios.get('/api/download-csv', {
  responseType: 'blob'
})
.then(response => {
  // 파일 다운로드 로직
})
```

### 3. CSV 형식 오류

**문제**: CSV 데이터에 큰따옴표(`"`)나 쉼표(`,`)가 포함된 경우 필드가 밀리는 문제가 발생했습니다.

**해결**:
- 필드에 쉼표, 큰따옴표, 개행 문자가 포함된 경우, 해당 필드 전체를 큰따옴표로 감쌌습니다.
- 필드 내의 큰따옴표는 두 번 연속 사용하여 이스케이프 처리했습니다.

이 해결 방법은 RFC 4180 표준을 참고하여 구현되었습니다. RFC 4180은 CSV 파일 형식에 대한 공식적인 명세를 제공하며, 특수 문자 처리에 대한 가이드라인을 포함하고 있습니다.

RFC 4180에 따르면:
- 필드는 쉼표로 구분됩니다.
- 필드에 쉼표, 큰따옴표, 또는 줄바꿈이 포함된 경우 해당 필드를 큰따옴표로 묶어야 합니다.
- 필드 내에서 큰따옴표를 사용할 경우, 두 개의 연속된 큰따옴표로 이스케이프 처리해야 합니다.

**코드 예시**:
```kotlin
fun escapeSpecialCharacters(field: String): String {
    return if (field.contains("\n") || field.contains("\"") || field.contains(",")) {
        "\"${field.replace("\"", "\"\"")}\""
    } else {
        field
    }
}

// CSV 행 생성 시
val csvLine = listOf(field1, field2, field3)
    .map { escapeSpecialCharacters(it) }
    .joinToString(",")
```

이 방식을 적용함으로써 RFC 4180 표준을 준수하고, CSV 파일의 구조적 무결성을 유지할 수 있습니다.

**Apache Commons CSV 사용**:

또 다른 해결 방법으로 Apache Commons CSV 라이브러리를 사용할 수 있습니다. 이 라이브러리는 CSV 파일 처리를 위한 강력하고 유연한 도구를 제공합니다.

Apache Commons CSV를 사용하면 특수 문자 처리와 같은 복잡한 로직을 직접 구현할 필요 없이 CSV 파일을 쉽게 읽고 쓸 수 있습니다. 예를 들어:

```java
CSVPrinter csvPrinter = new CSVPrinter(writer, CSVFormat.DEFAULT);
csvPrinter.printRecord("John Doe", "30", "New York, NY");
```

이 코드는 자동으로 필요한 이스케이프 처리를 수행하여 "New York, NY"와 같은 쉼표를 포함한 필드를 올바르게 처리합니다.

Apache Commons CSV를 사용하면 CSV 형식 오류를 방지하고 표준을 준수하는 CSV 파일을 쉽게 생성할 수 있으며, 특수 문자 처리에 대한 걱정 없이 데이터를 안전하게 다룰 수 있습니다

### 결론

CSV 파일 다운로드 기능 구현 시 다음 사항들을 고려해야 합니다:

1. 인코딩 문제 해결을 위한 BOM 추가
2. 프론트엔드에서의 올바른 응답 타입 설정
3. CSV 형식에 맞는 특수 문자 처리

이러한 방법들을 적용함으로써 안정적이고 정확한 CSV 파일 다운로드 기능을 구현할 수 있습니다.