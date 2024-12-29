---
title: "Ktlint: 기본 설정 변경 및 커스터마이징 가이드"
description: ".editorconfig를 활용한 스타일 커스터마이징 방법을 알아봅니다. 특정 규칙 비활성화, IntelliJ 플러그인 사용법, 그리고 효율적인 코드 스타일 관리 팁을 제공합니다."
date: 2024-02-22T20:46:04+09:00
lastmod: 2024-12-29T09:36:43+09:00
url: "/editorconfig/"
tags: [kotlin]
---

Ktlint는 Kotlin 코드 스타일을 검사하고 포맷팅하는 도구로, 협업 시 코드의 일관성을 유지하는 데 유용합니다. Ktlint **1.0 버전**부터 기본 설정이 `ktlint_official` 코드 스타일로 변경되었습니다. 만약 다른 스타일을 사용하고 싶다면, `.editorconfig` 파일의 `ktlint_code_style` 속성을 통해 이를 변경할 수 있습니다.

### **코드 스타일 변경하기**
`.editorconfig` 파일에서 아래와 같이 설정하여 원하는 코드 스타일을 지정할 수 있습니다:

```
[*.{kt,kts}]
ktlint_code_style = intellij_idea # 또는 android_studio, ktlint_official (기본값)
```

### **특정 규칙 비활성화하기**
특정 규칙을 비활성화하려면 `ktlint_` 접두사와 규칙 세트의 ID를 조합하여 설정하면 됩니다. 예를 들어, `ktlint_official` 코드 스타일을 사용하면서 `standard` 규칙 세트의 `final-newline` 규칙을 비활성화하려면 다음과 같이 설정합니다:

```
[*.{kt,kts}]
ktlint_code_style = ktlint_official
ktlint_standard_final-newline = disabled
```

### **IntelliJ에서 Ktlint 플러그인 활용하기**
IntelliJ IDEA를 사용하는 경우, [Ktlint 플러그인](https://plugins.jetbrains.com/plugin/15057-ktlint)을 설치하면 규칙 세트 ID와 규칙 이름을 쉽게 확인할 수 있습니다. 플러그인을 설치한 후, **Settings > Tools > Ktlint**에서 모드를 *Manual*로 변경하면 코드 스타일이 맞지 않는 경우 관련 정보를 확인할 수 있습니다:

![](/images/ktlint-editorconfig.png)

### **규칙 비활성화 예시**
위의 이미지는 IntelliJ에서 표시되는 특정 규칙의 예시입니다. 해당 규칙을 비활성화하려면 `.editorconfig` 파일에 다음과 같이 추가합니다:

```
[*.{kt,kts}]
ktlint_standard_no-multi-spaces = disabled
```

### **추가 참고 사항**
- **전체 규칙 세트 비활성화**: 특정 규칙 세트 전체를 비활성화하려면 아래와 같이 설정할 수 있습니다:
  ```
  [*.{kt,kts}]
  ktlint_standard = disabled # 'standard' 규칙 세트 전체 비활성화
  ```
- **실험적 규칙 활성화**: 실험적 규칙은 기본적으로 비활성화되어 있으며, 명시적으로 활성화해야 사용할 수 있습니다:
  ```
  [*.{kt,kts}]
  ktlint_experimental = enabled
  ```

### **Pre-commit Hook 및 자동화**
Ktlint는 Git pre-commit hook과 통합하여 커밋 전에 자동으로 코드 스타일 검사를 실행할 수 있습니다. Gradle을 사용하는 경우 아래 명령어로 Git Hook을 추가할 수 있습니다:
```
./gradlew addKtlintCheckGitPreCommitHook
```

### **참고 자료**
더 자세한 설정 옵션은 [KtLint 공식 문서](https://pinterest.github.io/ktlint/latest/rules/configuration-ktlint/)를 참조하세요.

