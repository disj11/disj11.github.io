---
title: "JetBrains Fleet에서 Kotest 테스트 실행하기: Run Configuration 설정 가이드"
description: "JetBrains의 차세대 IDE, Fleet에서 Kotest 테스트를 실행하는 방법을 알아봅니다. 아직 Kotest 플러그인이 지원되지 않는 환경에서, Gradle과 Fleet의 Run Configuration을 활용하여 특정 테스트 파일을 편리하게 실행하고 결과를 확인하는 방법을 단계별로 안내합니다."
date: 2025-01-18T11:01:48+09:00
tags: ["kotlin", "kotest", "fleet", "testing"]
---

## 들어가며: Fleet과 Kotest, 아직은 어색한 사이

JetBrains Fleet은 IntelliJ 기반의 차세대 경량 IDE로 많은 기대를 받고 있습니다. 하지만 아직 개발 초기 단계에 있어 기존 IntelliJ IDEA에서 당연하게 사용하던 기능 중 일부는 지원되지 않거나 별도의 설정이 필요합니다. 대표적인 예가 바로 **Kotest 플러그인의 부재**입니다.

IntelliJ에서는 Kotest 플러그인 덕분에 테스트 클래스나 메서드 옆의 '재생' 버튼을 눌러 손쉽게 테스트를 실행할 수 있었습니다. 하지만 Fleet에서는 이 기능이 기본적으로 작동하지 않아, Kotest 사용자들이 테스트 실행에 불편함을 겪고 있습니다. 

이 글에서는 Fleet의 강력한 **Run Configuration** 기능을 활용하여, 마치 플러그인이 있는 것처럼 특정 Kotest 테스트 파일을 편리하게 실행하는 방법을 소개합니다.

## 기본 방법: Gradle을 통한 테스트 실행

물론 Gradle을 사용하면 터미널에서 직접 테스트를 실행할 수 있습니다. 프로젝트의 모든 테스트를 실행하려면 다음 명령어를 사용합니다.

```shell
gradle test
```

만약 특정 파일의 테스트만 실행하고 싶다면, `--tests` 옵션을 사용하여 클래스의 전체 경로(fully qualified name)를 지정해야 합니다. 예를 들어, `com.demo.TestFile` 클래스의 테스트만 실행하려면 다음과 같이 입력합니다.

```shell
gradle test --tests "com.demo.TestFile"
```

하지만 이 방법은 다음과 같은 명백한 한계가 있습니다.

-   테스트 결과를 한눈에 파악하기 어렵습니다.
-   실행하려는 테스트 파일이 바뀔 때마다 긴 클래스 경로를 포함한 명령어를 매번 다시 입력해야 하는 번거로움이 있습니다.

## 해결책: Fleet의 사용자 정의 Run Configuration 활용하기

Fleet은 `.fleet/run.json` 파일을 통해 사용자 정의 실행 구성을 만들 수 있는 유연한 기능을 제공합니다. 이를 활용하여 현재 열려 있는 테스트 파일을 동적으로 실행하는 구성을 만들어 보겠습니다.

### 1. `run.json` 파일 생성

프로젝트의 루트 디렉터리에 `.fleet` 폴더를 생성하고, 그 안에 `run.json` 파일을 만듭니다. 그리고 아래 내용을 붙여넣습니다.

```json
{
    "configurations": [
      {
        "type": "gradle",
        "name": "Run Current Kotest File",
        "tasks": [ "test" ],
        "args": [ "--tests", "$FILE_NAME_NO_EXT$" ],
        "workingDir": "$PROJECT_DIR$"
      }
    ]
}
```

### 2. 설정 파일 분석

각 설정 항목이 어떤 의미를 가지는지 자세히 살펴보겠습니다.

-   `"type": "gradle"`
    이 실행 구성이 **Gradle**을 사용하여 실행됨을 Fleet에 알립니다.

-   `"name": "Run Current Kotest File"`
    Fleet의 실행 메뉴(Run & Debug 패널)에 표시될 이름입니다. 원하는 이름으로 자유롭게 변경할 수 있습니다.

-   `"tasks": [ "test" ]`
    실행할 Gradle 태스크를 지정합니다. 여기서는 표준 테스트 태스크인 `test`를 사용합니다.

-   `"args": [ "--tests", "$FILE_NAME_NO_EXT$" ]`
    **이 설정의 핵심 부분입니다.** `test` 태스크에 전달할 인자를 정의합니다.
    -   `--tests`: Gradle에게 특정 테스트만 실행하도록 지시하는 플래그입니다.
    -   `$FILE_NAME_NO_EXT$`: Fleet이 제공하는 **동적 변수**로, 현재 에디터에 열려 있는 파일의 전체 경로 이름(패키지 포함, 확장자 제외)으로 자동 치환됩니다. 예를 들어, `src/test/kotlin/com/demo/MyTest.kt` 파일을 열고 이 구성을 실행하면, 이 변수는 `com.demo.MyTest`라는 문자열로 바뀝니다.

-   `"workingDir": "$PROJECT_DIR$"`
    명령어가 실행될 작업 디렉터리를 지정합니다. `$PROJECT_DIR$` 변수는 프로젝트의 루트 폴더를 가리킵니다.

### 3. 사용 방법

설정이 완료되면 다음과 같이 사용할 수 있습니다.

1.  실행하고 싶은 Kotest 테스트 파일(예: `MyTest.kt`)을 에디터에서 엽니다.
2.  Fleet 우측 상단의 **Run & Debug** 패널을 엽니다.
3.  드롭다운 메뉴에서 방금 생성한 **"Run Current Kotest File"** 구성을 선택합니다.
4.  '재생(Run)' 버튼을 클릭합니다.

이제 Fleet은 Gradle을 통해 현재 열려 있는 파일의 테스트만을 실행하고, 그 결과를 IDE 내의 테스트 결과 창에 깔끔하게 보여줄 것입니다.

## 마무리

JetBrains Fleet은 아직 발전 중인 IDE이지만, 유연한 Run Configuration과 같은 강력한 기능을 통해 사용자가 직접 개발 환경을 최적화할 수 있는 가능성을 보여줍니다. 비록 공식적인 Kotest 플러그인은 아직 없지만([FL-19204 이슈](https://youtrack.jetbrains.com/issue/FL-19204/kotest-support)에서 관련 논의를 확인할 수 있습니다), 위와 같은 간단한 설정을 통해 IntelliJ와 유사한 개발 경험을 구성할 수 있습니다.

앞으로 Fleet 생태계가 더욱 성숙하여 다양한 플러그인이 지원되기를 기대하며, 그전까지는 이러한 사용자 정의 기능을 적극적으로 활용하여 생산성을 높여보시길 바랍니다.
