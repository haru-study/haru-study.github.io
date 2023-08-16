---
title: "하루스터디의 Jenkins를 이용한 CD 구축기"
excerpt: "하루스터디 CD 구축 배경과 Jenkins 선택 과정, 그리고 어떻게 CD를 구축했는지에 대해서 알아보겠습니다."

categories:
  - DevOps
tags:
  - [CI/CD, Jenkins]

toc: true
toc_sticky: true

date: 2023-07-25
last_modified_at: 2023-07-25
---

> 이 글은 백엔드 크루 마코와 프론트엔드 크루 엽토가 작성했습니다.

## 들어가며

[지난 글](https://haru-study.github.io/devops/%ED%95%98%EB%A3%A8%EC%8A%A4%ED%84%B0%EB%94%94%EC%9D%98-Github-Action%EC%9D%84-%EC%9D%B4%EC%9A%A9%ED%95%9C-CI-%EA%B5%AC%EC%B6%95%EA%B8%B0/)에서 하루스터디의 CI에 대해서 알아보았습니다.
이번글에는 하루스터디 CD 구축 배경과 Jenkins 선택 과정, 그리고 어떻게 CD를 구축했는지에 대해서 알아보겠습니다.

## CD 구축 배경

하루스터디는 현재 서비스 개발 단계에 있습니다.
기능이 완성될 때 마다 코드를 통합하고, 배포 서버에 배포해서 기능이 잘 동작하는지, 프론트엔드와 백엔드의 통신이 잘 이뤄지고 있는지 직접 확인합니다.

### 배포를 하기 위해 백엔드에서 벌어지는 일

1. 최신 커밋이 반영된 github repository의 소스 코드를 클론합니다.
2. ./gradlew bootJar로 jar파일을 빌드합니다.
3. 8080포트에 실행중인 프로세스가 있으면 프로세스를 종료합니다.
4. java -jar 빌드된_파일명.jar 로 서버를 실행합니다.

### 배포를 하기 위해 프론트엔드에서 벌어지는 일

1. 최신 커밋이 반영된 github repository의 소스 코드를 클론합니다.
2. yarn build로 빌드합니다. (이 단계에서 package.json에 yarn build 스크립트를 yarn test까지 추가해주어 test가 통과하지 않을 시 빌드파일를 생성하지 않도록 했습니다.)
3. 빌드한 파일을 nginx의 html 경로로 이동시킵니다.

새로운 기능이 개발될 때 마다 배포 서버에 직접 ssh 접속을 해서 위 과정을 모두 수행하는 것은 매우 귀찮은 일이었습니다. 또한, 배포 서버의 ssh 포트인 22번은 우테코 캠퍼스 내의 ip에서만 열려있기 때문에 집이나 외부에서 급하게 배포를 해야하는 경우가 생기면 캠퍼스에 나와서 직접 배포를 해야합니다.

빌드 스크립트를 작성해서 클론하고, 빌드하고, 서버를 실행하는 과정은 간소화할 수 있었으나 여전히 직접 ssh 접속을 해야하는 불편함은 남아있었습니다.

이러한 불편함을 해결하기 위해 우리 팀은 CD(Continues Deploy)를 구축하기로 결정했습니다.

## Jenkins

> Jenkins is a self-contained, open source automation server which can be used to automate all sorts of tasks related to building, testing, and delivering or deploying software.

jenkins의 공식 문서에 따르면 jenkins는 빌드, 테스트, 배포와 관련된 모든 작업을 자동화할 수 있는 오픈 소스 자동화 서버라고 이야기하고 있습니다.

하루스터디는 CI는 github action을, CD는 jenkins를 선택했습니다.

하루스터디가 jenkins를 선택한 이유는 다음과 같습니다.

- github action에 비해 비교적 오래된 기술이기 때문에 많은 문서가 존재하여 기술 습득 및 트러블 슈팅에 용이합니다.
- jenkins의 단점은 러닝 커브가 높다는 단점인데 이 러닝 커브는 초기 설치 및 설정에서의 어려움이 큽니다. 초기 설치와 설정을 해본 팀원이있어 어렵지 않다고 판단했습니다.

CI에서 사용했던 github action을 self-hosted-runner로 CD를 구축하는 데 사용해도 되지만 저희 팀은 첫 번째 이유에 크게 이끌려 jenkins를 선택했습니다.

## CD 목표

먼저, 목표로 하는 CD 파이프라인을 보겠습니다.

<img src = "https://github.com/haru-study/haru-study.github.io/assets/35948985/ac9b148c-b722-434a-896d-ab250e819c72">

jenkins 서버에서는 CD(continues delivery)만 수행하고 CD(continues deploy)는 배포 서버에서 수행합니다.
continues delivery와 continues deploy는 조금 다릅니다.

<img src ="https://www.redhat.com/rhdc/managed-files/ci-cd-flow-desktop.png?cicd=32h281b">

jenkins 서버에서는 빌드만 수행해서 배포 서버로 전달(delivery)하고, 배포(deploy)는 배포 서버에서 수행합니다.

배포 서버에서 빌드까지 진행하면 메모리가 부족해지는 등 서버에 영향을 줄 수 있기 때문에 jenkins서버에서 빌드 하고, 배포 서버로 전달하도록 설계했습니다.

### 파이프라인 스크립트

jenkins의 파이프라인 스크립트를 먼저 보고, jenkins 파이프라인을 생성해보겠습니다.

백엔드 파이프라인은 다음과 같습니다.

```groovy
pipeline {
    agent any
    tools {
        jdk "amazon-corretto-17"
    }
    environment {
        JAVA_HOME = "tool amazon-corretto-17"
    }
    stages {
        stage('git clone') {
            steps {
                git branch: 'develop', url: 'https://github.com/woowacourse-teams/2023-haru-study/'
            }
        }

        stage('build') {
            steps {
                dir('backend') {
                    sh '''
                        echo 'start bootJar'
                        ./gradlew clean bootJar
                    '''
                }
            }
        }

        stage('publish over ssh') {
            steps {
                dir('backend') {
                    sshPublisher(publishers: [sshPublisherDesc(configName: 'haru-study-prod', transfers: [sshTransfer(cleanRemote: false, excludes: '', execCommand: 'sh /home/ubuntu/2023-haru-study/script/backend_deploy.sh > backend_deploy.out 2>&1', execTimeout: 120000, flatten: false, makeEmptyDirs: false, noDefaultExcludes: false, patternSeparator: '[, ]+', remoteDirectory: '/2023-haru-study/deploy', remoteDirectorySDF: false, removePrefix: 'build/libs', sourceFiles: 'build/libs/*.jar')], usePromotionTimestamp: false, useWorkspaceInPromotion: false, verbose: true)])
                }
            }
        }
    }
}

```

프론트엔드 파이프라인은 다음과 같습니다.

```groovy
pipeline {
    agent any

    stages {
        stage('github') {
            steps {
                git branch: 'develop', credentialsId: 'repo-and-hook-access-token-username-and-password', url: 'https://github.com/woowacourse-teams/2023-haru-study/'
            }
        }
        stage('test & build') {
            steps {
                dir('frontend') {
                    nodejs('NodeJS 18.16.0') {
                        sh 'yarn && yarn build'
                        sh 'yarn build-storybook'
                    }
                }
            }
        }
        stage('deploy') {
            steps {
                dir('frontend') {
                    sshPublisher(publishers: [sshPublisherDesc(configName: 'haru-study-prod', transfers: [sshTransfer(cleanRemote: false, excludes: '', execCommand: '', execTimeout: 120000, flatten: false, makeEmptyDirs: false, noDefaultExcludes: false, patternSeparator: '[, ]+', remoteDirectory: '/2023-haru-study/html', remoteDirectorySDF: false, removePrefix: 'dist', sourceFiles: 'dist/*')], usePromotionTimestamp: false, useWorkspaceInPromotion: false, verbose: false)])
                }
            }
        }
        stage('storybook-deploy') {
            steps {
                dir('frontend') {
                    sshPublisher(publishers: [sshPublisherDesc(configName: 'haru-study-prod', transfers: [sshTransfer(cleanRemote: false, excludes: '', execCommand: '', execTimeout: 120000, flatten: false, makeEmptyDirs: false, noDefaultExcludes: false, patternSeparator: '[, ]+', remoteDirectory: '/2023-haru-study/storybook', remoteDirectorySDF: false, removePrefix: 'storybook-static', sourceFiles: 'storybook-static/**/*')], usePromotionTimestamp: false, useWorkspaceInPromotion: false, verbose: false)])
                }
            }
        }
    }
}

```

git clone stage와 build stage는 스크립트만 보고 이해하실수 있을거라고 생각합니다.

각 파이프라인에서 java version 설정과 node.js version 설정을 알아보고, publish over ssh stage에서 빌드된 파일을 배포 서버로 전송하고, 배포 스크립트를 실행시키는 과정을 알아보겠습니다.

publish over ssh stage의 스크립트는 뒤에서 jenkins를 통해 생성할 것이니 일단 넘어가시면 됩니다.

### 파이프라인 생성하기

<img src="https://github.com/haru-study/haru-study.github.io/assets/35948985/68f1293e-ff27-4f07-96c2-c5b8fe1a51f0">

좌측 탭의 새로운 item을 선택합니다.

<img src="https://github.com/haru-study/haru-study.github.io/assets/35948985/a2d8ca37-df80-4d5c-8cdc-eb05f23fb091">

pipeline을 선택하고 파이프라인의 이름을 입력합니다.

<img src="https://github.com/haru-study/haru-study.github.io/assets/35948985/f82751d9-a166-4ce9-8df8-d8db5aff72f7">

clone할 github repository의 url을 입력합니다.

<img src="https://github.com/haru-study/haru-study.github.io/assets/35948985/ef42f5b2-26f5-43e7-9b3d-d1029a14a64f">

아래로 쭉 내려와서 위에서 작성했던 파이프라인 스크립트를 작성하면 파이프라인은 완성입니다.

### tools을 이용한 java version 설정

백엔드는 java 17 버전을 사용하기로 했습니다. 이에 맞게 빌드 환경을 java 17로 구성해보겠습니다.

<img src="https://github.com/haru-study/haru-study.github.io/assets/35948985/9f2daba5-37ba-4f96-95df-c4d47c13270d">

Jenkins관리 > Tools를 선택합니다.

<img src="https://github.com/haru-study/haru-study.github.io/assets/35948985/f8910b2c-db3b-4695-abcd-cb392b854c74">

JDK 부분에서 ADD JDK를 클릭합니다.

<img src="https://github.com/haru-study/haru-study.github.io/assets/35948985/655bfcb4-a7c0-4916-8f80-71d22da26bcb">

JDK Name을 입력하고 Install automatically를 선택하면 binary archive의 경로와 서브 디렉토리 경로를 입력하는 창이 나옵니다. 아래의 과정을 통해 입력을 마치고 Save를 누르면 설정이 완료됩니다. 여기서 입력한 JDK Name을 파이프라인에서 가져다 쓸 것입니다.

https://github.com/corretto/corretto-17/releases/tag/17.0.7.7.1 서버의 아키텍처에 맞는 tar.gz의 주소를 복사하여 붙여넣습니다. 서브디렉토리는 tar.gz파일을 다운 받고 압축을 풀었을 때의 디렉토리의 이름을 적어주시면 됩니다.

<img src="https://github.com/haru-study/haru-study.github.io/assets/35948985/4553bd8a-c23e-45a0-8eb6-bd34c216f16c">

https://docs.aws.amazon.com/corretto/latest/corretto-17-ug/downloads-list.html 아마존 공식 홈페이지의 다운로드 링크에서는 latest이기 때문에 버전이 업그레이드되면 문제가 발생할 수 있어 직접 버전을 명시했습니다.

### Node.js Plugin & tools를 이용한 Node.js version 설정

프론트엔드는 node 18.16.0 버전을 사용하기로 했습니다. 이에 맞게 버전 설정을 해보겠습니다.

먼저 node.js plugin을 설치해야 젠킨스 서버에 node.js를 설치하고 사용할 수 있습니다. node.js plugin 설치부터 알아보겠습니다.

<img src="https://user-images.githubusercontent.com/78894403/259620597-94526d7f-d644-4b1f-af43-aad5b674c615.png">

Jenkins 관리 > plugins로 이동합니다.

<img src="https://user-images.githubusercontent.com/78894403/259620480-b84805c6-5921-404b-a8c4-dae7555773a0.png">

좌측 Available plugins 탭을 클릭하여 검색 창에 `Node`를 입력하고 NodeJS를 체크하여 `Download now and install after restart` 버튼을 클릭하여 설치해줍니다. 이제 버전 설정을 알아보겠습니다.

<img src="https://github.com/haru-study/haru-study.github.io/assets/35948985/9f2daba5-37ba-4f96-95df-c4d47c13270d">

다시 Jenkins 관리로 돌아와 Tools를 선택합니다.

<img src="https://user-images.githubusercontent.com/78894403/260882005-558be314-04e8-453b-94d3-4630ad460763.png">

스크롤을 내려 NodeJS 설정에서 설치할 버전 및 패키지 메니저 설정을 해주고 save 버튼을 클릭합니다.

```groovy
stage('test & build') {
            steps {
                dir('frontend') {
                    nodejs('NodeJS 18.16.0') {
                        sh 'yarn && yarn build'
                        sh 'yarn build-storybook'
                    }
                }
            }
        }
```

젠킨스 스크립트에 `nodejs('NodeJS 18.16.0)`를 명시해주면 젠킨스 서버에 우리가 설정한 버전과 패키지 매니저가 설치되므로 이제 Node 환경에서 빌드 및 테스트를 진행할 수 있습니다.

### publish over ssh로 빌드한 파일 배포 서버에 전송 및 배포 스크립트 실행하기

위에서 plugin을 이용해 node.js 버전을 설정할 수 있었던 것처럼, jenkins는 다양한 기능을 플러그인을 통해 제공합니다.

<img src="https://github.com/haru-study/haru-study.github.io/assets/35948985/d074ab33-62cb-47be-b6d4-273d4f20d804">

Jenkins 관리 > Plugins로 이동합니다.

<img src="https://github.com/haru-study/haru-study.github.io/assets/35948985/7d58d7df-69ce-4018-8e7f-23ed30b79d28">

좌측 탭에서 Avaliable plugins를 선택하고 publish over ssh를 검색하고 체크합니다.
Download now and install after restart를 눌러 설치하고, 완료되면 재시작하면 됩니다.

이제 어느 서버에 ssh로 연결할지 설정하겠습니다. Jenkins 관리 > System 에서 Publish over SSH까지 내려갑니다.

<img src="https://github.com/haru-study/haru-study.github.io/assets/35948985/4e7c0cee-22bf-42dd-8fe1-8ad21d2bb9c2">

Key에 ssh접속에 필요한 key 파일을 텍스트 형태로 넣습니다.

<img src="https://github.com/haru-study/haru-study.github.io/assets/35948985/3ff47554-5b1a-4646-a691-e9f9872e954a">

서버 이름, ip, username, remote direcory를 입력하고 우측에 있는 Test Configuration을 눌러서 접속이 잘 되는지 테스트하고 잘 된다면 저장합니다.
현재 jenkins 서버에서 ssh 접속할 배포 서버는 같은 VPC내에 있기 때문에 프라이빗 ip인 192.168.x.x로 설정했습니다.

이제 이 플러그인을 이용해서 파이프라인 스크립트를 생성해보겠습니다. 다시 파이프라인 스크립트를 작성하는 페이지로 돌아갑니다.

<img src="https://github.com/haru-study/haru-study.github.io/assets/35948985/bd969eb1-658f-494e-a9e8-1df43b2dec2b">

맨 아래의 Pipeline Syntax로 이동합니다.

<img src="https://github.com/haru-study/haru-study.github.io/assets/35948985/f8bb8788-1682-44ec-8c63-572860871986">

드롭박스의 Step중 sshPublisher를 선택합니다.

<img src="https://github.com/haru-study/haru-study.github.io/assets/35948985/ad5de5d7-26b2-44d7-9f9c-bbbda7a177ef">

아까 등록했던 SSH Server를 선택합니다.

- Source files : 배포 서버로 전송할 파일입니다.
- Remove prefix : Source files에 설정한 경로에서 build/libs가 prefix로 붙기 때문에 remove 해줍니다.
- Remote directory : 배포 서버에서 어느 경로에 빌드된 파일을 전송할지 정합니다.
- Exec command : 파일을 전송하고 어떤 명령어를 실행시킬지 정합니다. 저는 미리 작성해둔 backend_deploy.sh를 실행시킬 것입니다.

<img src="https://github.com/haru-study/haru-study.github.io/assets/35948985/4dd2d682-eff2-48c2-b1df-5bbbfb5ba085">

Generate Pipeline Script를 누르면 스크립트가 완성됩니다. 이 스크립트를 복사하여 파이프라인 스크립트에 붙여넣으면 됩니다.

<img src="https://github.com/haru-study/haru-study.github.io/assets/35948985/e1629db2-d9ea-44a0-a90d-0b484398f963">

github repository에서 특정 브랜치에 merge가 될 때 webhook을 트리거로 받아 자동으로 스크립트를 실행하도록 설정할수도 있습니다.
하지만 저희는 그 과정까지는 자동화할 필요성을 느끼지 못하고 배포를 하면 각 stage가 잘 동작했는지 jenkins에서 바로 확인할 것이기 때문에 직접 손으로 스크립트를 실행하도록 했습니다.
위 사진에서 좌측의 `지금 빌드`를 클릭하면 스크립트가 실행됩니다.

저희는 맨 앞에 Declearative Checkout SCM stage가 추가로 있습니다.

### SCM에서 pipeline script 가져오기

파이프라인 스크립트를 github repository에서 관리하고 여기서 가져와서 실행하도록 설정한 것입니다. 이렇게 파이프라인 스크립트를 SCM에서 관리하면 파이프라인 스크립트도 코드 리뷰를 받을 수 있고 형상관리도 할 수 있습니다.

<img src="https://github.com/haru-study/haru-study.github.io/assets/35948985/1228c735-6bfa-4df5-acf5-4594ed5a149a">

파이프라인 구성에서 Definition > Pipeline script from SCM을 선택합니다. 만약 서브 모듈이나 private repository와 같이 Credential이 필요한 경우라면 추가로 설정해주어야 합니다. 하루스터디 repository는 public이기 때문에 Credential은 설정하지 않았습니다.

<img src="https://github.com/haru-study/haru-study.github.io/assets/35948985/4e2103a8-38a7-47be-ace7-81beb401f546">

어느 브랜치의 파이프라인 스크립트를 가져올지 설정하고, github repository에서 스크립트가 존재하는 경로를 Script Path에 입력하고 저장하면 끝입니다.

## 결론

하루스터디 CD 구축 배경과 Jenkins 선택 과정, 그리고 어떻게 CD를 구축했는지에 대해서 알아보았습니다.
저희 팀은 자동화된 CD를 구축함으로써 배포를 하는데 필요한 과정을 버튼 한번으로 간편하게 배포하고 다시 개발에 집중할 수 있었습니다.

참고 문서

- https://www.redhat.com/ko/topics/devops/what-is-ci-cd
- https://velog.io/@sihyung92/%EC%9A%B0%EC%A0%A0%EA%B5%AC2%ED%8E%B8-%EC%A0%A0%ED%82%A8%EC%8A%A4-%ED%8C%8C%EC%9D%B4%ED%94%84%EB%9D%BC%EC%9D%B8%EC%9D%84-%ED%99%9C%EC%9A%A9%ED%95%9C-%EB%B0%B0%ED%8F%AC-%EC%9E%90%EB%8F%99%ED%99%94#ssh%EB%A5%BC-%EC%9D%B4%EC%9A%A9%ED%95%98%EC%97%AC-%EC%84%9C%EB%B2%84%EB%A1%9C-jar%ED%8C%8C%EC%9D%BC-%EC%A0%84%EB%8B%AC%ED%95%98%EA%B8%B0
