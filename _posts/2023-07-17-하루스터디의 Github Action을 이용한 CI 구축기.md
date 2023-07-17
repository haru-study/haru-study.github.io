---
title:  "하루스터디의 Github Action을 이용한 CI 구축기"
excerpt: "하루스터디 백엔드의 CI 도입 배경과 Github Action 선택 과정, 그리고 어떻게 CI를 구축했는지에 대해서 알아보겠습니다."

categories:
  - BackEnd
tags:
  - [CI/CD, Github Action]

toc: true
toc_sticky: true

date: 2023-07-17
last_modified_at: 2023-07-17
---

> 이 글은 백엔드 크루 마코가 작성했습니다.
>

## 들어가며

하루스터디 백엔드의 CI 도입 배경과 Github Action 선택 과정, 그리고 어떻게 CI를 구축했는지에 대해서 알아보겠습니다.

## CI 도입 배경

하루스터디는 다음과 같은 팀 규칙을 만들었습니다.

1. 모든 코드는 Pull Request를 제출하여 모든 팀원에게 코드 리뷰를 받고, approve를 받은 후에 머지한다.
2. 모든 Pull Request는 빌드에 성공하고 모든 테스트가 통과해야 한다.

위와 같은 규칙을 지키기 위해서 매번 Pull Request를 요청하기 전에 빌드를 직접 해보고, 테스트가 통과하는지 확인해야 했습니다.
본인이 제출한 코드는 빌드와 테스트가 통과하는지 확인하는 과정은 필요하지만 다른 팀원들의 코드를 모두 확인하기는 피곤하고 힘들었습니다.
이러한 문제점을 해결하기 위해 하루스터디는 이 과정을 자동화한 CI를 구축하였습니다.

## Github Action

CI를 구축하기 위한 도구 중에서 하루스터디는 Github Action을 선택했는데요, 그 이유는 다음과 같습니다.

### 컴퓨팅 파워

Pull Request를 제출할 때 마다 빌드와 테스트를 돌리는 것은 꽤 많은 컴퓨팅 파워를 소모하고 이는 빌드와 테스트만을 위한 서버를 필요로 하게 됩니다.
하루스터디는 RAM 2GB 사양의 EC2 인스턴스를 보유하고 있습니다. RAM 2GB로 빌드와 테스트를 모두 수행하기에는 조금 불안정한 사양입니다.
Github-Action은 github에서 제공하는 클라우드 환경에서 실행되는 github-hosted-runner 를 사용하여 CI를 수행할 수 있습니다.
github-hosted-runner를 사용하기 위해 제공되는 클라우드의 성능은 최소 RAM 7GB 이상이기 때문에 훨씬 안정적인 환경에서 빌드와 테스트를 수행함은 물론 서버 자원도 아낄 수 있습니다.

### Github와 통합

Github Action은 github와 잘 통합되어있기 때문에 github와 연동해서 사용하기에 좋습니다.
따로 구현하지 않아도 CI결과를 Pull Request화면에서 쉽게 확인할 수 있습니다.

추후에 CI과정에 정적 코드 분석을 해주는 소나큐브도 도입할 예정인데요, 이 결과를 Pull Request에 댓글로 달아준다면 팀원들이 확인하기 좋겠죠?
이렇게 댓글을 달아주는 기능도 마켓플레이스에서 구현된 action을 가져다 쓰기만 하면 쉽게 사용할 수 있습니다.

이렇게 컴퓨팅 파워와 Github와의 통합이라는 장점에 의해 하루스터디는 CI를 구축하기 위한 도구로 Github Action을 선택했습니다.

## CI workflow

하루스터디의 초기 CI workflow는 간단합니다.

### 트리거

트리거는 develop 브랜치에 Pull Request가 발생했을 때, 변경됐을 때, reopened 됐을 때 수행됩니다.
그 중에서 백엔드 코드의 기능 추가나 리팩토링에 대해서만 빌드와 테스트를 진행하면 되기 때문에 Pull Request의 라벨 중 BE와 feautre 또는 BE와 refactor가 붙어있는 경우에만 실행하도록 설계했습니다.

### job

job은 빌드와 테스트를 수행하는 하나만 존재하며 순서는 다음과 같습니다.

![job.png](img%2Fjob.png)

build-test라는 하나의 job에서 빨간 색으로 네모 친 부분이 각 step입니다.

1. 소스 코드를 클론합니다.
2. 빌드 환경을 JDK17로 설정합니다.
3. 빌드와 테스트를 진행합니다.
4. 결과를 하루스터디의 슬랙으로 알림을 보내줍니다.

이렇게 결과를 팀원 모두가 슬랙으로 확인할 수 있도록 했습니다.

![action_slack.png](img%2Faction_slack.png)

작성한 workflow는 다음과 같습니다.

```yml
name: backend-build-test

on:
	pull_request:
		branches:
			- develop
		types: [ opened, synchronize, reopened ]
	workflow_dispatch:

defaults:
	run:
		working-directory: ./backend

jobs:
	build-test:
		# label이 (BE && feature) || (BE && refactor)일 경우 실행
		if: |
			(contains(github.event.pull_request.labels.*.id, 5681136383) &&
			contains(github.event.pull_request.labels.*.id, 5681142648)) ||
			(contains(github.event.pull_request.labels.*.id, 5681136383) &&
			contains(github.event.pull_request.labels.*.id, 5681143873))

		runs-on: ubuntu-latest
		steps:
			- name: Checkout source code
			uses: actions/checkout@v3

			- name: Set up JDK 17
			uses: actions/setup-java@v3
			with:
				java-version: '17'
				distribution: 'corretto'

			- name: Build Test
			run: ./gradlew build

			- name: action-slack
			uses: 8398a7/action-slack@v3
			with:
				status: ${{ job.status }}
				author_name: Github Action # default: 8398a7@action-slack
				fields: repo,message,commit,author,action,eventName,ref,workflow,job,took
			env:
				SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }} # required
			if: always()
```

이렇게 작성한 workflow파일을 repository내의 .github/workflows 경로에 추가하면 github action이 작동합니다.

다음은 Pull Request창에서 확인할 수 있는 결과입니다.

![pull_request.png](img%2Fpull_request.png)

현재 backend-build-test와 front-end-build-test 두 가지 workflow가 작성되어있습니다.
해당 PR은 라벨 중 BE,feature 또는 BE,refactor 가 붙어있지 않기 때문에 workflow가 Skipped되어 체크가 패스된 것을 확인할 수 있습니다.
만약 해당 라벨이 붙어있다면 빌드와 테스트가 성공해야만 체크가 패스하게 됩니다.

## 결론

하루스터디 백엔드의 CI 도입 배경과 Github Action 선택 과정, 그리고 어떻게 CI를 구축했는지 알아봤습니다.
이렇게 CI를 구축함으로써 저희 팀원들은 코드를 통합하는 과정을 자동화하여 코드를 작성하는 작업에 더 집중할 수 있었는데요~🎵
다음으로는 정적 코드 분석을 위한 소나큐브를 CI과정에 도입하고 배포까지 자동화하는 CD도 구축할 예정이니 다음 글을 기다려주세요. 감사합니다.