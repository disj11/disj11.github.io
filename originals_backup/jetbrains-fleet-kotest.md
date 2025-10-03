---
title: "JetBrains Fleet에서 Kotest 사용하기"
description: "JetBrains의 새로운 IDE인 Fleet은 IntelliJ와 달리 Kotest 플러그인을 기본적으로 지원하지 않습니다. 따라서 Kotest 테스트를 실행하려면 약간의 설정이 필요합니다. 이 글에서는 Gradle을 사용하여 Kotest 테스트를 실행하는 방법과 Fleet에서 테스트 실행을 편리하게 설정하는 방법을 소개합니다"
date: 2025-01-18T11:01:48+09:00
url: "/jetbrains-fleet-kotest/"
tags: [TIL]
---

## JetBrains Fleet에서 Kotest 사용하기

JetBrains의 Fleet은 IntelliJ와 달리 Kotest 플러그인을 기본적으로 지원하지 않습니다. 따라서 Kotest 테스트를 실행하려면 약간의 설정이 필요합니다. 이 글에서는 Gradle을 사용하여 Kotest 테스트를 실행하는 방법과 Fleet에서 테스트 실행을 편리하게 설정하는 방법을 소개합니다.

## Gradle을 사용한 테스트 실행

Gradle을 사용하면 아래 명령어를 통해 프로젝트의 모든 테스트를 실행할 수 있습니다:
```shell
gradle test
```

특정 파일의 테스트만 실행하고 싶다면 `--tests` 옵션을 추가로 사용해야 합니다. 예를 들어, `com.demo.TestFile` 클래스의 테스트만 실행하려면 다음과 같이 입력합니다:

```shell
gradle test --tests "com.demo.TestFile"
```

## CLI의 한계와 대안

CLI를 사용하여 테스트를 실행하면 다음과 같은 불편함이 있을 수 있습니다:
- 어떤 테스트가 성공했는지 결과를 확인하기 어렵습니다.
- 특정 테스트만 실행하려면 매번 명령어를 입력해야 합니다.

Fleet에서는 Run Configuration을 직접 생성하여 이러한 문제를 해결할 수 있습니다.

## Run Configuration 설정하기

아래와 같은 내용을 포함한 .fleet/run.conf 파일을 프로젝트에 추가합니다:

```json
{
    "configurations": [
      {
        "type": "gradle",
        "name": "Run Gradle Test",
        "tasks": [ "test" ],
        "args": [ "--tests", "$FILE_NAME_NO_EXT$" ],
        "workingDir": "$PROJECT_DIR$"
      }
    ]
}
```
[FL-19204](https://youtrack.jetbrains.com/issue/FL-19204/kotest-support) 참고

이 설정 파일은 Fleet의 Run 메뉴에 Run Gradle Test 항목을 추가합니다. 이를 통해 특정 테스트 파일의 테스트를 실행할 수 있으며, 결과를 IDE 내에서 쉽게 확인할 수 있습니다.

## 마무리

JetBrains Fleet은 Kotest 플러그인을 기본 제공하지 않지만, 위와 같은 방법으로 Gradle과 Run Configuration을 활용하면 효율적으로 테스트를 실행하고 관리할 수 있습니다. 앞으로 Fleet에서 Kotest 플러그인을 지원하게 된다면 더욱 편리한 환경이 제공될 것으로 기대됩니다.
