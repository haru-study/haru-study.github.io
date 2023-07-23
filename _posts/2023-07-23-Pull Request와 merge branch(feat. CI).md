---
title:  "Pull Request와 merge branch(feat. CI)"
excerpt: "Pull Request의 숨겨진 특성과 하루스터디의 CI 환경에서는 이를 어떻게 활용하는지를 알아보았습니다."

categories:
  - Github, Backend
tags:
  - [Pull Request, merge branch]

toc: true
toc_sticky: true
 
date: 2023-07-23
last_modified_at: 2023-07-23
---

> 이 글은 백엔드 크루 테오가 작성했습니다.
>

# 들어가면서
하루스터디 팀은 Github Actions를 사용해 CI 환경을 구축하고 있습니다. 즉, Pull Request가 발생할 때마다 자동으로 빌드와 테스트를 진행해 코드가 병합되어도 되는지를 판단해줍니다. 이를 통해 코드가 병합되었을 때의 파급효과를 최소화할 수 있습니다.

그런데 이런 기능은 어떻게 동작하는 걸까요? 아직 merge도 하지 않았고, Pull Request만 보냈을 뿐인데 어떻게 통합된 코드를 빌드하고 테스트할 수 있는 것일까요?

그 해답은 Pull Request에 있습니다. Pull Request는 단순한 `merge 요청` 이라고 생각하기 쉽상이지만, 사실 숨겨진 원리가 존재하기 때문입니다.

그리고 이 숨겨진 원리를 안다면 Pull Request를 활용한 CI 파이프라인을 보다 쉽게 이해할 수 있을 것입니다.
## Pull Request가 생성되면 발생하는 일

Pull Ruquest가 생성되면 단순히 Github에서 merge 요청만 발생하는 것이 아닙니다. 사실 Pull Request가 생성되는 순간 **총 두 개의 브랜치**가 생깁니다.

1. **refs/pull/{PR번호}/head**
2. **refs/pull/{PR번호}/merge**

이 두 개의 브랜치는 Pull Request가 닫히거나 병합될때까지 관리됩니다. 어떤 역할을 하는 브랜치들일까요? 네이밍에서 유추할 수 있듯이, `refs/pull/{PR번호}/head`는 병합되길 원하는 브랜치의 HEAD를 가리키고 `refs/pull/{PR번호}/merge` 는 병합이 된 시점을 가정해 존재하는 브랜치입니다. 즉, 쉽게 말해 merge가 된 시점을 `미리보기`할 수 있는 브랜치라는 것이죠. 이는 `merge branch` 라고 불리기도 합니다.

merge branch는 언제 사용될 수 있을까요? 예상하셨다시피 CI 환경에서 사용될 수 있습니다. Pull Request만 생성되더라도 미리 병합된 코드를 `미리보기`해서 빌드와 테스트까지 수행할 수 있는 것이죠.

따라서 Pull Request에서 충돌이 발생하는 경우에는 CI가 작동하지 않습니다(병합된 코드를 `미리보기` 할 수 없으니까요). 충돌을 해결하고서야 비로소 CI 파이프라인이 동작하게 됩니다.

## 도식화한다면

<div style="text-align: center"> <img src="https://github.com/haru-study/haru-study.github.io/assets/78679830/70e61317-f14d-4709-9c02-938558566911" style="width: 80%; height: 80%"> </div>

그림으로 도식화하면 위와 같습니다. develop 브랜치를 main 브랜치에 병합하기 위해 Pull Request를 보냈다고 가정해봅시다. 이 상황에서 앞서 설명드렸다시피 Github는 자동으로 `refs/pull/1/merge` 와 `refs/pull/1/head` 브랜치를 생성합니다.

그리고 merge branch(`refs/pull/1/merge)` 의 경우에는 보시다시피 가상으로 병합된 커밋(`merge preview`)을 포함하고 있습니다.

## CI 환경에서 실제로 확인해보기

위에서 이론적으로만 이야기했던 내용들이 실제로 맞는지 확인해보겠습니다. Github Actions를 기준으로 설명하는 내용이니 참고해 주세요.

<div style="text-align: center"> <img src="https://github.com/haru-study/haru-study.github.io/assets/78679830/310671a9-a8a1-402b-8e00-0958e4c05740">

<img src="https://github.com/haru-study/haru-study.github.io/assets/78679830/e3c22163-2edf-4140-a793-a135cb3cad2e" style="align-content: center"> </div>

하루스터디팀의 workflow는 다음과 같습니다. develop 브랜치로의 pull_request가 발생했을 때 빌드가 테스트가 수행되도록 설정해두었습니다.

이제 actions가 동작한 로그를 확인하러 가봅시다.

<div style="text-align: center"> <img src="https://github.com/haru-study/haru-study.github.io/assets/78679830/3f4dec37-2367-456f-a193-65576f8ab2f8"> </div>
Actions 탭에 들어가 성공적으로 빌드와 테스트가 완료된 workflow 중 아무 것이나 선택하고, 로그를 살펴보면

<div style="text-align: center"> <img src="https://github.com/haru-study/haru-study.github.io/assets/78679830/c69559a5-3a71-4a11-b35c-257ae7ae3268"> </div>
`fetching the repository` 부분에서 `9e607...` 해시를 가진 커밋을 `refs/remotes/pull/116/merge` 라는 이름으로 fetch하는 것을 알 수 있습니다. 

이 `9e607...` 해시를 가진 커밋이 바로 앞서 설명했던 merge branch 의 최신 커밋입니다. 앞서 설명했듯 가상으로 병합되어 생겨난 커밋이기 때문에, 원격 Repository 어디에서도 이 해시 값을 가진 커밋을 찾을 수는 없습니다. 

Github Actions의 경우, Pull Request 이벤트 트리거가 작동하면 내부적으로 `merge branch의 최신 커밋`을 환경 변수로 저장합니다(`GITHUB_SHA` 라는 값에 저장됩니다). 그리고 이 환경 변수 값을 이용해 위처럼 fetch를 수행하는 것이죠. 
<div style="text-align:center"> <img src="https://github.com/haru-study/haru-study.github.io/assets/78679830/0cfb51bb-0031-45b0-9e78-3469b7f9262b"> </div>

## 마치며
이번 아티클을 통해 CI 환경에서 병합된 상태를 가정하여 빌드와 테스트를 돌리는 것은 마법같은 일이 아니라, Pull Request의 특성을 이용한 결과물임을 확인할 수 있었습니다.



감사합니다.

## 참고 자료
<a href = "https://docs.github.com/en/actions/using-workflows/events-that-trigger-workflows#pull_request">Github Actions events that trigger workflows</a> <br>
<a href = "https://fluffyandflakey.blog/2022/12/21/what-is-a-github-pull-request-merge-branch/#conclusions">What is a GitHub Pull Request merge branch?</a>
