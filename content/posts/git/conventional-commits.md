---
title: Conventional Commits
date: 2021-06-05T09:18:00+09:00
menu:
  sidebar:
    name: Conventional Commits
    identifier: git-conventional-commits
    parent: git
    weight: 10
---

## 개요

`Conventional Commits` 스펙은 명확한 커밋 히스토리를 생성하기 위한 규칙을 제공하며, 커밋 메시지는 다음과 같아야 한다.

```
<type>[optional scope]: <description>

[optional body]

[optional footer(s)]
```

커밋에는 다음과 같은 구조적 요소가 포함된다.

1. **fix:** 버그 픽스를 위한 커밋 (PATCH)
2. **feat:** 새 기능 추가를 위한 커밋 (MINOR)
3. **BREAKING CHANGE:** footer 영역의 시작이 `BREAKING CHANGE` 이거나, type/scope 영역 뒤에 `!` 가 붙어있다면 주요 API의 변경이 있다는 것을 의미 (MAJOR)
4. `fix:` 와 `feat:` 이외의 type도 허용되며 `build:`, `chore:`, `ci:` `docs:`, `style:`, `refactor:`, `pref:`, `test:` 등이 있다.
5. `BREAKING CHANGE:<description>` 이외의 footer를 작성할 수 있으며 [git trailer format](https://git-scm.com/docs/git-interpret-trailers)과 비슷한 규칙을 따른다.

type은 커밋 규격에 의해 의무화 되지 않는다.

## 예제

### description과 breaking change footer를 사용한 커밋 메시지

```
feat: allow provided config object to extend other configs

BREAKING CHANGE: `extends` key in config file is now used for extending other config files
```

### breaking change에 주의를 주기 위해 `!` 를 사용한 커밋 메시지

```
refactor!: drop support for Node 6
```

### `!` 와 BREAKING CHANGE footer를 모두 사용한 커밋 메시지

```
refactor!: drop support for Node 6

BREAKING CHANGE: refactor to use JavaScript features not available in Node 6.
```

### body가 없는 커밋 메시지

```
docs: correct spelling of CHANGELOG
```

### scope가 있는 커밋 메시지

```
feat(lang): add polish language
```

### 다중 단락의 body와 다수의 footer를 사용한 커밋 메시지

```
fix: correct minor typos in code

see the issue for details

on typos fixed.

Reviewed-by: Z
Refs #133
```

## Conventional Commits를 위한 도구

[Git Commit Template](https://plugins.jetbrains.com/plugin/9861-git-commit-template): [JetBrains](https://plugins.jetbrains.com/plugin/9861-git-commit-template)

## 참고

[https://www.conventionalcommits.org/ko/v1.0.0/](https://www.conventionalcommits.org/ko/v1.0.0/)