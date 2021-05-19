---
title: Git branching model (깃 브랜치 전략)
date: 2020-08-15T15:30:00+09:00
menu:
  sidebar:
    name: Git branching model (깃 브랜치 전략)
    identifier: git-branching-model
    parent: git
    weight: 10
---

## 개요

가장 대중적으로 알려진 깃 브랜치 모델에 대하여 설명합니다. 빠른 개발과 잦은 배포를 원한다면, [GitHub flow](https://guides.github.com/introduction/flow/)와 같은 간단한 방법도 존재합니다. 완벽한 깃 플로우는 없으며, 여러가지 사항을 고려하여 스스로 선택하여야 합니다.

## flow

![Git branching model](https://nvie.com/img/git-model@2x.png)

## The main branches

* master
* develop

## Supporting branches

* Feature branches
* Release branches
* Hotfix branches

## Feature branches

* `develop` 브랜치에서 생성
* `develop` 브랜치로 병합
* 브랜치 이름은 master, develop, release-*, hotfix-* 를 제외하고 모두 가능

기능 브랜치는 새로운 기능을 개발하는데 사용됩니다. 기능 개발이 완료되면 `develop` 브랜치로 병합되거나, 필요 없는 기능이라면 폐기됩니다.

## Release branches

* `develop` 브랜치에서 생성
* `develop` 브랜치와 `master` 브랜치로 병합
* 브랜치 이름은 `release-*`

릴리즈 브랜치는 새 프로덕션을 준비하기 위해 사용됩니다. 프로덕션에 배포될 기능을 테스트하고, 사소한 버그를 수정합니다. 버그 수정이 완료되면 master 브랜치와 develop 브랜치로 각각 병합합니다.
develop 브랜치로 병합하는 과정에서 충돌이 일어날 수 있습니다. 만약 충돌이 발생했다면 적절히 수정후 커밋하세요.
(릴리즈 브랜치가 생성되었다면 develop 브랜치 프리징 권장합니다.)
여기까지 완료되었다면 버전 관리를 위해 tag 를 생성합니다.

## hotfix branches

* `master` 브랜치에서 생성
* `develop` 브랜치와 `master` 브랜치로 병함
* 브랜치 이름은 `hotfix-*`

프로덕션에서 중요한 버그를 즉시 해결해야하는 경우 사용됩니다. 버그 수정이 완료되면 `master` 브랜치와 `develop` 브랜치에 각각 병합합니다.
(만약, 이미 `release` 브랜치가 존재한다면, `develop` 브랜치가 아닌 `release` 브랜치로 병합합니다.)
병합이 완료되면 tag 를 생성합니다.

## 참고

[참고 사이트](https://nvie.com/posts/a-successful-git-branching-model/)