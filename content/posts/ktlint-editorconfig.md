---
title: "Ktlint와 .editorconfig로 Kotlin 코드 스타일 커스터마이징하기"
description: "Kotlin의 대표적인 린터(Linter)인 Ktlint의 규칙을 프로젝트에 맞게 커스터마이징하는 방법을 알아봅니다. .editorconfig 파일을 사용하여 코드 스타일을 변경하고, 특정 규칙을 비활성화하며, IntelliJ 플러그인을 활용해 규칙 ID를 쉽게 찾는 실용적인 팁을 제공합니다."
date: 2024-02-22T20:46:04+09:00
lastmod: 2024-12-29T09:36:43+09:00
tags: ["kotlin", "ktlint", "editorconfig", "code-style"]
---

## 들어가며: Ktlint와 .editorconfig

**Ktlint**는 Kotlin 코드의 스타일을 검사하고 자동으로 포맷팅해주는 매우 인기 있는 도구(Linter)입니다. Ktlint의 핵심 철학은 **"설정 없는(no configuration)"** 방식으로, 팀이나 프로젝트 간의 코드 스타일 논쟁을 최소화하고 일관성을 유지하는 데 큰 도움을 줍니다.

하지만 때로는 프로젝트의 기존 규칙이나 팀의 합의에 따라 특정 스타일 가이드를 따르거나 일부 규칙을 변경해야 할 필요가 있습니다. Ktlint는 이러한 요구사항을 위해 **`.editorconfig`** 파일을 통한 커스터마이징을 지원합니다. `.editorconfig`는 다양한 에디터와 IDE에서 코드 스타일을 일관되게 유지하기 위한 표준 파일 형식으로, Ktlint는 이 파일을 읽어 설정을 적용합니다.

이 글에서는 `.editorconfig` 파일을 사용하여 Ktlint의 동작을 프로젝트에 맞게 설정하는 방법을 자세히 알아보겠습니다.

## 1. 기본 코드 스타일 변경하기

Ktlint 1.0 버전부터 기본 코드 스타일은 `ktlint_official`로 지정되어 있습니다. 만약 IntelliJ IDEA의 기본 포맷터나 Android 개발 스타일 가이드를 따르고 싶다면, `.editorconfig` 파일에 다음과 같이 `ktlint_code_style` 속성을 추가하여 변경할 수 있습니다.

```editorconfig
# 프로젝트 루트 디렉터리에 .editorconfig 파일을 생성하거나 기존 파일에 추가합니다.

# Kotlin 파일에 적용될 규칙임을 명시합니다.
[*.{kt,kts}]

# 원하는 코드 스타일을 지정합니다.
# 선택 가능: intellij_idea, android_studio, ktlint_official (기본값)
ktlint_code_style = intellij_idea
```

## 2. 특정 규칙 비활성화하기

가장 흔한 커스터마이징 요구사항은 특정 규칙을 비활성화하는 것입니다. 예를 들어, 팀에서 와일드카드 임포트(`import com.example.*`)를 허용하기로 합의했다면, `no-wildcard-imports` 규칙을 비활성화할 수 있습니다.

규칙을 비활성화하는 형식은 `ktlint_<규칙세트ID>_<규칙ID> = disabled` 입니다.

```editorconfig
[*.{kt,kts}]

# standard 규칙 세트의 no-wildcard-imports 규칙을 비활성화합니다.
ktlint_standard_no-wildcard-imports = disabled

# experimental 규칙 세트의 some-rule 규칙을 비활성화합니다.
ktlint_experimental_some-rule = disabled
```

### 규칙 ID는 어떻게 찾을 수 있을까요?

비활성화하고 싶은 규칙의 정확한 ID를 알아내는 것이 중요합니다. 규칙 ID를 찾는 방법은 다음과 같습니다.

#### 방법 1: IntelliJ Ktlint 플러그인 활용하기 (가장 쉬운 방법)

IntelliJ IDEA를 사용한다면 [Ktlint 플러그인](https://plugins.jetbrains.com/plugin/15057-ktlint)을 설치하는 것이 가장 편리합니다. 플러그인을 설치하면, 코드 스타일 위반 시 해당 부분에 경고가 표시되며, 경고 메시지에서 규칙 ID를 쉽게 확인할 수 있습니다.

![](/images/ktlint-editorconfig.png)

위 이미지처럼, 여러 개의 공백 사용을 금지하는 규칙에 대한 경고가 표시되면, 메시지 끝에 있는 괄호 안의 `(standard:no-multi-spaces)` 부분을 통해 규칙 세트 ID가 `standard`이고 규칙 ID가 `no-multi-spaces`임을 알 수 있습니다. 따라서 이 규칙을 비활성화하려면 `.editorconfig`에 `ktlint_standard_no-multi-spaces = disabled`를 추가하면 됩니다.

#### 방법 2: Ktlint CLI 출력 확인하기

커맨드 라인에서 `ktlint` 명령을 실행하여 코드 스타일을 검사하면, 각 위반 사항과 함께 규칙 ID가 출력됩니다. 이를 통해서도 정확한 ID를 확인할 수 있습니다.

#### 방법 3: 공식 문서 참고하기

[Ktlint 공식 규칙 문서](https://pinterest.github.io/ktlint/latest/rules/standard/)에서 현재 사용 가능한 모든 표준 규칙 목록과 각 규칙의 ID를 확인할 수 있습니다.

## 3. 추가 설정 팁

### 실험적 규칙(Experimental Rules) 활성화

Ktlint는 아직 표준으로 채택되지 않았지만 유용할 수 있는 실험적 규칙들을 제공합니다. 이 규칙들은 기본적으로 비활성화되어 있으며, 사용하려면 명시적으로 활성화해야 합니다.

```editorconfig
[*.{kt,kts}]
ktlint_experimental = enabled
```

### 특정 규칙 세트 전체 비활성화

만약 `standard` 규칙 세트 전체를 사용하고 싶지 않다면, 다음과 같이 설정할 수 있습니다.

```editorconfig
[*.{kt,kts}]
ktlint_standard = disabled
```

## 4. Git Hook을 이용한 자동화

팀원들이 커밋하기 전에 항상 Ktlint 검사를 실행하도록 강제하면 코드 스타일의 일관성을 더욱 효과적으로 유지할 수 있습니다. `ktlint-gradle` 플러그인을 사용하는 경우, 다음 명령어를 통해 Git pre-commit hook을 자동으로 설정할 수 있습니다.

```shell
./gradlew addKtlintCheckGitPreCommitHook
```

이 명령을 실행하면, `git commit` 시 자동으로 `ktlintCheck` 태스크가 실행되어 코드 스타일에 맞지 않는 코드가 커밋되는 것을 방지합니다.

## 결론

Ktlint는 강력한 코드 스타일 관리 도구이며, `.editorconfig`를 통해 프로젝트의 특성에 맞게 유연하게 설정을 변경할 수 있습니다. 기본 설정을 최대한 따르는 것이 좋지만, 불가피하게 규칙을 변경해야 할 경우 오늘 소개한 방법을 활용하여 팀의 코드 스타일 가이드를 효과적으로 적용해 보시길 바랍니다. 잘 구성된 `.editorconfig` 파일은 그 자체로 훌륭한 코드 스타일 문서가 될 수 있습니다.

---

**참고 자료:**
- [Ktlint 공식 문서 - Configuration](https://pinterest.github.io/ktlint/latest/rules/configuration-ktlint/)
