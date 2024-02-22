---
title: ".editorconfig 를 통한 코틀린 코드 스타일 설정"
description: ""
date: 2024-02-22T20:46:04+09:00
url: "/editorconfig/"
tags: [TIL]
---

ktlint 1.0 버전부터 ktlint 의 기본 설정이 `ktlint_offcial` 로 변경되었다.
만약 다른 스타일을 사용하고 싶다면 `.editorconfig` 의 `ktlint_code_style` 을 통해 변경할 수 있다.

```
[*.{kt,kts}]
ktlint_code_style = intellij_idea # or android_studio or ktlint_official (default)
```

특정 규칙을 비활성화하고 싶다면 `ktlint_` prefix 와 rule set 의 id 를 사용하면 된다.
예를 들어 `ktlint_offcial` 코드 스타일을 사용하면서 standard rule set 의 `final-newline` 규칙을 비활성화 하고 싶다면 아래와 같이 설정할 수 있다.

```
[*.{kt,kts}]
ktlint_code_style = ktlint_official
ktlint_standard_final-newline = disabled
```

intellij 를 사용하는 경우 [ktlint 플러그인](https://plugins.jetbrains.com/plugin/15057-ktlint)을 설치하면 rule set 의 id 와 규칙 이름을 찾는데 도움이 된다.
플러그인을 설치 후 `Settings > Tools > Ktlint` 에서 mode 를 Manual 로 변경하면 코드 스타일이 맞지 않는 경우 rule set 의 id 와 규칙을 확인할 수 있다.

![.editorconfig](/images/ktlint-editorconfig.png)

만약 위 이미지에 표기되는 규칙을 disable 하고 싶다면 `.editorconfig` 에 아래와 같이 설정하면 된다.

```
[*.{kt,kts}]
ktlint_standard_no-multi-spaces = disabled
```

