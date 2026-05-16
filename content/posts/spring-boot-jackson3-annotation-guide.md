---
title: "Spring Boot 4에서 Jackson 어노테이션이 런타임에 무시되는 이유"
description: "Spring Boot 4의 Jackson 3와 의존성 라이브러리의 Jackson 2.x가 클래스패스에 함께 있을 때, @JsonNaming 같은 일부 어노테이션이 컴파일은 되지만 런타임에 무시되는 이유를 정리합니다."
date: 2026-01-23T19:26:00+09:00
url: "/spring-boot-jackson3-annotation-guide/"
tags: ["spring-boot", "jackson", "troubleshooting", "annotation"]
---

## 목차

1. [문제 상황](#문제-상황)
2. [원인 분석](#원인-분석)
   - [Jackson 3의 패키지 변경 전략](#jackson-3의-패키지-변경-전략)
   - [jackson-annotations의 특별한 처리](#jackson-annotations의-특별한-처리)
   - [클래스패스에 jackson-databind 2.x가 포함된 경우](#클래스패스에-jacksondatabind-2x가-포함된-경우)
3. [해결 방법](#해결-방법)
   - [올바른 import 사용법](#올바른-import-사용법)
   - [실수 방지 방법](#실수-방지-방법)
4. [패키지별 정리](#패키지별-정리)
5. [요약 및 빠른 참조](#요약-및-빠른-참조)

---

## 문제 상황

Spring Boot 4 프로젝트에서 Jackson 어노테이션을 쓰다 보면 이런 상황을 만날 수 있습니다.

- ✅ `@JsonFormat`, `@JsonProperty`, `@JsonTypeInfo`, `@JsonManagedReference` 등은 정상 동작
- ❌ `@JsonNaming`은 동작하지 않음

겉으로는 같은 Jackson 어노테이션처럼 보이는데, 왜 어떤 것은 동작하고 어떤 것은 무시될까요?

---

## 원인 분석

### Jackson 3의 패키지 변경 전략

Spring Boot 3까지는 Jackson 2.x를 사용했지만, **Spring Boot 4부터는 Jackson 3.x를 지원**합니다. Jackson 3로 올라오면서 대부분의 패키지가 `com.fasterxml.jackson`에서 `tools.jackson`으로 바뀌었습니다. 다만 예외가 있습니다.

> **버전 정보**
> - Spring Boot 3.x → Jackson 2.x 사용
> - Spring Boot 4.x → Jackson 3.x 사용

#### 변경된 패키지

Jackson 3에서 다음 패키지들은 `tools.jackson`으로 변경되었습니다:

```kotlin
// ❌ Jackson 2.x (Spring Boot 4에서 동작하지 않음)
import com.fasterxml.jackson.databind.ObjectMapper
import com.fasterxml.jackson.databind.annotation.JsonNaming
import com.fasterxml.jackson.databind.PropertyNamingStrategies

// ✅ Jackson 3.x (올바른 사용)
import tools.jackson.databind.ObjectMapper
import tools.jackson.databind.annotation.JsonNaming
import tools.jackson.databind.PropertyNamingStrategies
```

#### 변경되지 않은 패키지 (중요!)

**`com.fasterxml.jackson.annotation` 패키지는 Jackson 2.x와 3.x가 함께 쓰기 때문에 그대로 유지됩니다.**

```kotlin
// ✅ Jackson 2.x와 3.x 모두에서 동작
import com.fasterxml.jackson.annotation.JsonFormat
import com.fasterxml.jackson.annotation.JsonProperty
import com.fasterxml.jackson.annotation.JsonTypeInfo
import com.fasterxml.jackson.annotation.JsonManagedReference
import com.fasterxml.jackson.annotation.JsonBackReference
import com.fasterxml.jackson.annotation.JsonInclude
import com.fasterxml.jackson.annotation.JsonIgnore
```

### jackson-annotations의 특별한 처리

Jackson 3에서 `jackson-annotations` 모듈은 다른 모듈과 조금 다르게 처리됩니다.

1. **버전 관리**: Jackson 3.x 버전을 따로 발행하지 않고 Jackson 2.x 버전을 계속 사용합니다
2. **패키지 유지**: `com.fasterxml.jackson.annotation` 패키지명을 그대로 둡니다
3. **목적**: Jackson 2.x와 3.x가 같은 어노테이션 세트를 함께 사용할 수 있게 합니다

이는 [JSTEP-1 문서](https://github.com/FasterXML/jackson-future-ideas/wiki/JSTEP-1#handling-of-jackson-annotations)와 [Discussion #90](https://github.com/FasterXML/jackson-future-ideas/discussions/90)에서 결정된 설계입니다.

### 왜 `@JsonNaming`만 동작하지 않았나?

`@JsonNaming`은 `com.fasterxml.jackson.databind.annotation` 패키지에 있었습니다. 이 패키지는 Jackson 3에서 **제거**되고 `tools.jackson.databind.annotation`으로 이동했습니다.

반면 `@JsonFormat`, `@JsonProperty`, `@JsonTypeInfo`, `@JsonManagedReference` 등은 `com.fasterxml.jackson.annotation` 패키지에 있으므로 그대로 사용할 수 있습니다.

**핵심 차이점:**
- `com.fasterxml.jackson.annotation` → 변경 없음 (Jackson 2.x와 3.x 공유)
- `com.fasterxml.jackson.databind.annotation` → `tools.jackson.databind.annotation`으로 변경 (Jackson 3.x 전용)

### 왜 컴파일은 되는데 런타임에 동작하지 않을까?

Spring Boot 4는 Jackson 3.x를 사용하지만, 프로젝트의 클래스패스에는 **다른 라이브러리의 의존성으로 인해 Jackson 2.x가 함께 포함**될 수 있습니다:

```gradle
// 예시: 의존성 트리 확인 결과
+--- org.springframework.boot:spring-boot-starter-web:4.0.0
|    \--- tools.jackson.databind:jackson-databind:3.0.0  // ✅ Jackson 3.x
+--- org.springframework.cloud:spring-cloud-starter-circuitbreaker-resilience4j
     \--- com.fasterxml.jackson.databind:jackson-databind:2.20.1  // ⚠️ Jackson 2.x
```

여기서 문제가 시작됩니다.

**1. IDE가 잘못된 import를 제안**
- 클래스패스에 Jackson 2.x가 있으므로, IDE는 `com.fasterxml.jackson.databind.annotation.JsonNaming`을 자동완성 후보로 보여줌
- 개발자는 자연스럽게 그 import를 선택함

**2. 컴파일은 성공**
- Jackson 2.x 클래스가 클래스패스에 있으므로 컴파일 에러는 나지 않음

**3. 런타임에 동작하지 않음**
- Spring Boot 4는 Jackson 3의 어노테이션만 인식
- Jackson 2.x 패키지의 `@JsonNaming`은 무시됨
- **조용한 실패(Silent Failure)** 로 이어져 원인을 찾기 어려움

---

## 해결 방법

### 올바른 import 사용법

#### `@JsonNaming` 사용 시

```kotlin
// ❌ 동작하지 않음 (컴파일은 되지만 런타임에 동작하지 않음)
import com.fasterxml.jackson.databind.annotation.JsonNaming  // Jackson 2.x 패키지
import com.fasterxml.jackson.databind.PropertyNamingStrategies

@JsonNaming(PropertyNamingStrategies.UpperCamelCaseStrategy::class)
data class CommonTrackingParameter(
    val version: Int,          // JSON: version (변환 안됨)
    val inventoryKey: String,  // JSON: inventoryKey (변환 안됨)
)

// ✅ 올바른 사용
import tools.jackson.databind.annotation.JsonNaming  // Jackson 3.x 패키지
import tools.jackson.databind.PropertyNamingStrategies

@JsonNaming(PropertyNamingStrategies.UpperCamelCaseStrategy::class)
data class CommonTrackingParameter(
    val version: Int,          // JSON: Version (정상 변환)
    val inventoryKey: String,  // JSON: InventoryKey (정상 변환)
)
```

#### 다른 어노테이션들은 그대로 사용

```kotlin
// ✅ com.fasterxml.jackson.annotation 패키지는 변경하지 않음
import com.fasterxml.jackson.annotation.JsonFormat
import com.fasterxml.jackson.annotation.JsonProperty
import com.fasterxml.jackson.annotation.JsonTypeInfo
import com.fasterxml.jackson.annotation.JsonManagedReference

@JsonFormat(shape = JsonFormat.Shape.STRING, pattern = "yyyy-MM-dd HH:mm:ss")
val serverDatetime: LocalDateTime

@JsonTypeInfo(use = JsonTypeInfo.Id.NAME, include = JsonTypeInfo.As.PROPERTY, property = "sectionType")
interface SectionContent

@JsonManagedReference
val sections: MutableList<Section>
```

### 실수 방지 방법

#### 1. Detekt를 사용한 정적 분석

Detekt는 Kotlin 코드의 정적 분석 도구로, 커스텀 규칙을 추가하여 잘못된 import를 감지할 수 있습니다.

**설정 방법:**

1. **Detekt 플러그인 추가** (`build.gradle.kts`):

```kotlin
plugins {
    id("io.gitlab.arturbosch.detekt") version "1.23.0"
}

detekt {
    buildUponDefaultConfig = true
    allRules = false
    config.setFrom("$projectDir/config/detekt/detekt.yml")
}
```

2. **커스텀 규칙 작성** (`config/detekt/rules/JacksonImportRule.kt`):

```kotlin
import io.gitlab.arturbosch.detekt.api.*
import org.jetbrains.kotlin.psi.KtImportDirective

class JacksonImportRule(config: Config) : Rule(config) {
    override val issue = Issue(
        id = "JacksonImportRule",
        severity = Severity.Maintainability,
        description = "Jackson 3에서는 com.fasterxml.jackson.databind.annotation 패키지를 사용하면 안 됩니다",
        debt = Debt.TWENTY_MINUTES
    )

    override fun visitImportDirective(importDirective: KtImportDirective) {
        val importPath = importDirective.importPath?.pathStr
        if (importPath?.startsWith("com.fasterxml.jackson.databind.annotation") == true) {
            report(
                CodeSmell(
                    issue = issue,
                    entity = Entity.from(importDirective),
                    message = "Jackson 3에서는 'com.fasterxml.jackson.databind.annotation' 대신 " +
                            "'tools.jackson.databind.annotation'을 사용해야 합니다. " +
                            "잘못된 import: $importPath"
                )
            )
        }
    }
}
```

3. **Detekt 설정 파일** (`config/detekt/detekt.yml`):

```yaml
processors:
  active: true
  exclude:
    - 'FunctionCountProcessor'
    - 'PropertyCountProcessor'

custom:
  JacksonImportRule:
    active: true
    severity: error
```

4. **빌드 시 실행**:

```bash
./gradlew detekt
```

#### 2. Gradle 빌드 스크립트를 통한 검증

빌드 시점에 잘못된 import를 검사하는 Gradle 태스크를 추가할 수 있습니다:

```kotlin
// build.gradle.kts
tasks.register("checkJacksonImports") {
    doLast {
        val kotlinFiles = fileTree("src") {
            include("**/*.kt")
        }
        
        var hasError = false
        kotlinFiles.forEach { file ->
            val content = file.readText()
            if (content.contains("import com.fasterxml.jackson.databind.annotation")) {
                println("ERROR: ${file.path} contains incorrect Jackson import")
                println("  Use 'tools.jackson.databind.annotation' instead of 'com.fasterxml.jackson.databind.annotation'")
                hasError = true
            }
        }
        
        if (hasError) {
            throw GradleException("Found incorrect Jackson imports. Please use 'tools.jackson.databind.annotation' for Jackson 3.")
        }
    }
}

tasks.named("check") {
    dependsOn("checkJacksonImports")
}
```

#### 3. 코드 리뷰 체크리스트

코드 리뷰 시 다음 패턴을 확인하세요:

```kotlin
// ❌ 잘못된 import (컴파일은 되지만 런타임에 동작하지 않음)
import com.fasterxml.jackson.databind.annotation.JsonNaming
import com.fasterxml.jackson.databind.PropertyNamingStrategies

// ✅ 올바른 import
import tools.jackson.databind.annotation.JsonNaming
import tools.jackson.databind.PropertyNamingStrategies
```

#### 4. 의존성 확인

클래스패스에 `jackson-databind 2.x`가 포함되어 있는지 확인:

```bash
./gradlew :your-module:dependencies --configuration runtimeClasspath | grep "jackson-databind"
```

**참고**: `jackson-databind 2.x`는 외부 라이브러리 의존성으로 인해 제거할 수 없을 수 있습니다. 하지만 실제로는 Jackson 3.x가 사용되므로, 올바른 import만 사용하면 문제없습니다.

---

## 패키지별 정리

### `com.fasterxml.jackson.annotation` (변경 없음)

다음 어노테이션들은 Jackson 2.x와 3.x 모두에서 동일하게 사용합니다:

- `@JsonFormat` - 날짜/시간 형식 지정
- `@JsonProperty` - 필드 이름 매핑
- `@JsonTypeInfo` - 다형성 타입 정보
- `@JsonSubTypes` - 서브타입 지정
- `@JsonInclude` - 직렬화 포함 조건
- `@JsonManagedReference` / `@JsonBackReference` - 순환 참조 처리
- `@JsonIgnore` - 직렬화/역직렬화 제외
- `@JsonView` - 뷰 기반 직렬화
- 기타 대부분의 어노테이션

**사용 예시:**
```kotlin
import com.fasterxml.jackson.annotation.JsonFormat
import com.fasterxml.jackson.annotation.JsonTypeInfo

@JsonFormat(shape = JsonFormat.Shape.STRING, pattern = "yyyy-MM-dd HH:mm:ss")
val serverDatetime: LocalDateTime

@JsonTypeInfo(use = JsonTypeInfo.Id.NAME, include = JsonTypeInfo.As.PROPERTY, property = "sectionType")
interface SectionContent
```

### `tools.jackson.databind.annotation` (Jackson 3.x 전용)

다음 어노테이션들은 Jackson 3에서 패키지가 변경되었으므로, 반드시 `tools.jackson.databind.annotation`을 사용해야 합니다:

- `@JsonNaming` - 필드 명명 전략 지정 (예: camelCase → UpperCamelCase)
- `@JsonDeserialize` - 커스텀 역직렬화 처리
- `@JsonSerialize` - 커스텀 직렬화 처리

**사용 예시:**
```kotlin
import tools.jackson.databind.annotation.JsonNaming
import tools.jackson.databind.PropertyNamingStrategies

@JsonNaming(PropertyNamingStrategies.UpperCamelCaseStrategy::class)
data class CommonTrackingParameter(
    val version: Int,          // JSON으로 직렬화 시 "Version"으로 변환
    val inventoryKey: String,  // JSON으로 직렬화 시 "InventoryKey"로 변환
)
```

> **왜 패키지가 분리되었나?**
>
> `jackson-annotations` 패키지는 Jackson 2와 3 간 호환성을 위해 `com.fasterxml.jackson.annotation`을 유지합니다. 반면 `jackson-databind` 모듈의 어노테이션들(`@JsonSerialize`, `@JsonDeserialize`, `@JsonNaming` 등)은 databind 라이브러리의 구현에 더 밀접하게 연결되어 있어, Jackson 3에서 `tools.jackson.databind.annotation`으로 완전히 이동했습니다.

---

## 요약 및 빠른 참조

### 핵심 규칙

Spring Boot 4 (Jackson 3) 사용 시 기억해야 할 세 가지 규칙입니다:

1. **`com.fasterxml.jackson.annotation`**: 변경하지 않음 ✅
   - Jackson 2.x와 3.x 간 공유
   - 대부분의 어노테이션이 여기에 있음
   - 예: `@JsonFormat`, `@JsonProperty`, `@JsonTypeInfo`

2. **`com.fasterxml.jackson.databind.annotation`**: `tools.jackson.databind.annotation`으로 변경 ✅
   - `jackson-databind` 모듈의 일부
   - `@JsonNaming`, `@JsonSerialize`, `@JsonDeserialize` 등이 여기에 있음
   - **반드시 패키지 변경 필요**

3. **`com.fasterxml.jackson.*` (기타)**: `tools.jackson.*`으로 변경 ✅
   - `ObjectMapper` → `tools.jackson.databind.ObjectMapper`
   - 기타 핵심 클래스들

### 빠른 참조: 어노테이션별 패키지 정리

#### ✅ 변경 불필요 (com.fasterxml.jackson.annotation)

다음 어노테이션들은 Jackson 2.x와 3.x 모두 **동일한 패키지**를 사용합니다:

```kotlin
// Jackson 2.x와 3.x 모두 동일
import com.fasterxml.jackson.annotation.JsonFormat
import com.fasterxml.jackson.annotation.JsonProperty
import com.fasterxml.jackson.annotation.JsonTypeInfo
import com.fasterxml.jackson.annotation.JsonManagedReference
```

#### ⚠️ 변경 필요 (databind.annotation)

다음 어노테이션들은 **패키지 변경이 필수**입니다:

**`@JsonNaming`**
```kotlin
// ❌ Jackson 2.x (Spring Boot 4에서 동작 안 함)
import com.fasterxml.jackson.databind.annotation.JsonNaming

// ✅ Jackson 3.x (올바른 import)
import tools.jackson.databind.annotation.JsonNaming
```

**`@JsonSerialize`**
```kotlin
// ❌ Jackson 2.x
import com.fasterxml.jackson.databind.annotation.JsonSerialize

// ✅ Jackson 3.x
import tools.jackson.databind.annotation.JsonSerialize
```

**`@JsonDeserialize`**
```kotlin
// ❌ Jackson 2.x
import com.fasterxml.jackson.databind.annotation.JsonDeserialize

// ✅ Jackson 3.x
import tools.jackson.databind.annotation.JsonDeserialize
```

### 왜 혼란스러웠나?

이 문제가 헷갈리는 이유는 대략 이렇습니다.

**1. 같은 Jackson인데 어노테이션마다 동작이 다름**
- `@JsonFormat`, `@JsonProperty` → `com.fasterxml.jackson.annotation` 패키지 → ✅ 정상 동작
- `@JsonNaming` → `com.fasterxml.jackson.databind.annotation` 패키지 → ❌ 동작 안 함
- 겉으로는 모두 Jackson 어노테이션처럼 보이므로 패키지 차이를 놓치기 쉬움

**2. IDE가 잘못된 import를 자동완성으로 제안**
- 클래스패스에 Jackson 2.x 라이브러리가 남아 있음
- IDE는 이를 보고 `com.fasterxml.jackson.databind.annotation.JsonNaming`을 제안함
- 개발자는 IDE를 믿고 선택했지만 런타임에서는 동작하지 않음

**3. 컴파일은 성공하지만 런타임에 실패**
- 잘못된 Jackson 2.x import를 사용해도 컴파일 에러가 나지 않음
- 런타임에서만 어노테이션이 무시되어 문제 발견이 늦어짐
- **조용한 실패(Silent Failure)** 라서 디버깅이 까다로움

**4. Jackson 공식 문서에도 명확히 나와있지 않음**
- 마이그레이션 가이드에는 패키지 변경이 나오지만, `jackson-annotations`가 왜 예외인지는 JSTEP-1 문서까지 봐야 이해하기 쉬움
- 대부분의 개발자는 이런 설계 배경을 모른 상태에서 문제를 만남

### 공식 문서 근거

관련 설계 배경은 아래 문서에서 확인할 수 있습니다.

- **[JSTEP-1: Handling of jackson-annotations](https://github.com/FasterXML/jackson-future-ideas/wiki/JSTEP-1#handling-of-jackson-annotations)**: `jackson-annotations`의 특별한 처리 방식 설명
- **[Jackson 3 Migration Guide: Package Changes](https://github.com/FasterXML/jackson/blob/main/jackson3/MIGRATING_TO_JACKSON_3.md#2-new-maven-group-id-and-java-package)**: 패키지 변경 규칙 및 예외 사항
- **[Discussion #90: Fix upcoming confusing versioning of jackson-annotations](https://github.com/FasterXML/jackson-future-ideas/discussions/90)**: `jackson-annotations` 버전 관리 결정 과정
- **[Spring Blog: Introducing Jackson 3 support in Spring](https://spring.io/blog/2025/10/07/introducing-jackson-3-support-in-spring)**: Spring Boot 4의 Jackson 3 지원 소개

---

*이 글은 AI의 도움을 받아 교정 및 정리되었습니다.*
