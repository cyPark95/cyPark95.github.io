---
title: GitHub Actions Self-Hosted Runner란? 개념부터 설정까지
description: GitHub Actions Self-Hosted Runner의 개념과 동작 원리, GitHub-Hosted Runner와의 차이점, 온프레미스 환경에서의 설정 방법을 알아본다.
date: 2025-07-27 00:00:00+0000
math: true
categories:
  - DevOps
tags:
  - CI/CD
  - GitHub-Actions
  - Self-Hosted-Runner
---

GitHub Actions 기반의 CI/CD를 구현할 때, 가상 머신의 IP는 매번 변경된다.<br/>
이로 인해 내부 인프라에 서버를 두고 있는 환경에서는 특정 IP에 대한 인바운드 규칙을 설정할 수 없고, 모든 IP에 대한 인바운드를 허용해야 하는 문제가 발생한다.

이 글에서는 이러한 제약을 해결할 수 있는 방법으로 **Self-Hosted Runner의 개념과 동작 원리**, 그리고 **실제 설정 방법**을 알아본다.

## GitHub Actions란?

**GitHub Actions**는 GitHub에서 제공하는 CI/CD 플랫폼으로, Repository에 이벤트가 발생하면 미리 작성된 Workflow를 통해 자동으로 빌드·테스트·배포 등의 작업을 실행할 수 있다.

- 다양한 이벤트 기반 실행 (push, PR, cron 등)
- Ubuntu, Windows, macOS 등 여러 가상 머신 제공
- [GitHub Marketplace](https://github.com/marketplace?type=actions)에서 제공하는 액션 재사용 가능
- GitHub의 Issues, PR 등과 연동

즉, GitHub Actions는 **개발과 배포 과정 전체를 자동화**하는 플랫폼이다.

### 구성 요소

- **Workflow**: 하나 이상의 작업을 실행하는 자동화 프로세스
- **Event**: Workflow를 실행시키는 트리거
- **Job**: Workflow 안에서 실행되는 작업 단위 (병렬/순차 실행 가능)
- **Action**: 미리 정의하여 재사용 가능한 작업
- **Runner**: Workflow가 실제로 실행되는 환경
    - GitHub이 제공하는 **GitHub-Hosted Runner**
    - 사용자가 직접 운영하는 **Self-Hosted Runner**

### GitHub-Hosted Runner란?

GitHub Actions에서 Workflow를 실행하려면 **Runner**라는 실행 환경이 필요하다.<br/>
**GitHub-Hosted Runner**는 GitHub이 제공하는 클라우드 VM에서 Workflow를 실행하는 Runner다.

- 매번 새로운 VM에서 실행되므로 서버 관리가 필요 없고, 독립적인 환경이 보장된다.
- 환경 커스터마이징에는 제한이 있고, 캐시 활용에 한계가 있으며, 매번 다른 VM에서 실행되므로 고정된 IP를 보장하지 않는다.

### Self-Hosted Runner란?

**Self-Hosted Runner**는 GitHub이 아닌 사용자가 직접 준비한 서버(온프레미스, 클라우드 VM, 컨테이너 등)에 Runner 프로그램을 설치하여 Workflow를 실행하는 방식이다.<br/>
GitHub은 이 Runner에게 실행할 Job을 전달하기만 하고, 실제 실행은 사용자가 관리하는 머신에서 이루어진다.<br/>
즉, 실행 주체는 GitHub가 아니라 사용자가 관리하는 머신이다.

- 서버 사양, 설치된 소프트웨어, 네트워크 환경 등을 자유롭게 커스터마이징할 수 있다.
- 내부 인프라(사내망, VPN 등)에 있는 리소스에 직접 접근할 수 있다.
- 고정된 IP를 가지므로, 인바운드 규칙을 특정 IP로 제한할 수 있다.
- Runner 자체의 보안·업데이트·장애 대응을 사용자가 직접 책임져야 한다.

### 동작 원리

Self-Hosted Runner의 핵심은 **Runner가 GitHub에 접속하는 방향**에 있다.<br/>
Runner는 GitHub 서버에 인바운드로 열려있는 포트를 통해 연결을 받는 것이 아니라, Runner 프로그램이 주기적으로 GitHub Actions 서버에 **아웃바운드로 접속(polling)**하여 실행할 Job이 있는지 확인한다.

1. Runner를 Repository에 등록하면, GitHub은 해당 Runner의 정보를 저장한다.
2. Runner는 자신이 설치된 머신에서 GitHub Actions 서버로 지속적으로 연결을 유지하며 새로운 Job을 대기한다.
3. Workflow가 트리거되어 실행 대상 Runner로 이 Self-Hosted Runner가 지정되면, GitHub은 대기 중이던 연결을 통해 Job을 전달한다.
4. Runner는 전달받은 Job을 로컬 환경에서 실행하고, 결과를 다시 GitHub에 보고한다.

이 방식 덕분에 내부 인프라에 있는 서버는 **외부에서 접근 가능한 포트를 별도로 열어둘 필요가 없다.**<br/>
Runner가 항상 먼저 GitHub 쪽으로 연결을 시작하기 때문에, 앞서 언급한 "GitHub-Hosted Runner의 IP가 매번 바뀌어 인바운드 규칙을 특정할 수 없는 문제"를 근본적으로 우회할 수 있다.

## Self-Hosted Runner 적용하기

Self-Hosted Runner를 실제로 등록하고 워크플로우에서 사용하는 과정은 다음과 같다.

### 1. Runner 다운로드 및 등록

Repository 설정의 `Settings > Actions > Runners > New self-hosted runner`로 이동하면, OS에 맞는 설치 스크립트를 안내해준다.

```bash
$ mkdir actions-runner && cd actions-runner

$ curl -o actions-runner-osx-x64-2.336.0.tar.gz -L https://github.com/actions/runner/releases/download/v2.336.0/actions-runner-osx-x64-2.336.0.tar.gz

$ tar xzf ./actions-runner-osx-x64-2.336.0.tar.gz
```

압축을 풀고 나면, `config.sh`로 이 Runner를 Repository에 등록하고 실행한다.

```bash
# Runner 등록 및 설정 마법사 실행
$ ./config.sh --url https://github.com/{owner}/{repo} --token {TOKEN}

$ ./run.sh
```

`--token`은 GitHub에서 발급하는 일회성 등록 토큰으로, 해당 Runner를 어떤 Repository에 연결할지를 결정한다.<br/>
`config.sh` 과정에서 Runner의 이름과 커스텀 라벨(예: `deploy`)을 지정할 수 있다.

### 2. 상태 확인

Runner가 정상적으로 등록되어 GitHub과 연결되어 있는지는 Repository 설정의 `Settings > Actions > Runners`에서 확인할 수 있다.<br/>
Job을 받을 준비가 된 상태면 **Idle**, Job을 실행 중이면 **Active**, 연결이 끊겼다면 **Offline**으로 표시된다.

### 3. Workflow에서 지정

등록이 끝나면 GitHub은 Workflow에 다음 한 줄만 추가하면 된다고 안내한다.

```yaml
# Job을 Self-Hosted Runner에서 실행하도록 지정
runs-on: self-hosted
```

`self-hosted`는 등록된 모든 Self-Hosted Runner를 의미하고, 라벨을 추가하면 특정 Runner만 선택할 수 있다.<br/>
`runs-on`에 `self-hosted`를 지정하면 해당 Job은 GitHub-Hosted Runner 대신 등록해둔 Runner에서 실행되며, 등록 시 지정한 커스텀 라벨(예: `deploy`)을 함께 명시하면 그 라벨을 가진 Runner로 Job을 한정할 수 있다.

```yaml
jobs:
  deploy:
    runs-on: [self-hosted, deploy]
    steps:
      - uses: actions/checkout@v4
      - name: Deploy
        run: ./deploy.sh
```

## 한계와 주의할 점

Self-Hosted Runner가 인바운드 문제는 해결해주지만, 그 대신 떠안아야 하는 부담도 있다.

- **보안 위험**: Public 저장소에 Self-Hosted Runner를 연결하면, 외부에서 올린 PR의 코드가 이 Runner에서 그대로 실행될 수 있다. GitHub도 Runner 등록 화면에서 이를 명시적으로 경고하며, Private 저장소나 승인된 워크플로우에서만 사용하는 것을 권장한다.
- **운영 부담**: GitHub-Hosted Runner와 달리 OS 패치, Runner 버전 업데이트, 디스크·리소스 관리를 직접 해야 한다.
- **격리 부족**: GitHub-Hosted Runner는 Job마다 새로운 VM을 할당하지만, Self-Hosted Runner는 기본적으로 같은 머신을 재사용하므로 이전 Job의 파일이나 프로세스가 남아 다음 Job에 영향을 줄 수 있다. Job마다 격리된 실행 환경(Docker 컨테이너 등)을 별도로 구성해야 한다.
- **확장 한계**: Job이 몰려도 GitHub-Hosted Runner처럼 자동으로 VM이 늘어나지 않는다. 동시에 처리할 수 있는 Job 수는 등록해둔 Runner 대수에 그대로 묶인다.
- **장애 위험**: Runner가 설치된 서버가 다운되거나 네트워크가 끊기면, 그 Runner에 할당된 Job은 대기하거나 실패한다.

## 정리

- GitHub-Hosted Runner는 매번 다른 VM에서 실행되어 고정된 IP를 보장하지 않으므로, 온프레미스 방화벽에서 특정 IP로 인바운드 규칙을 제한할 수 없다.
- Self-Hosted Runner는 Runner가 GitHub에 아웃바운드로 접속하는 구조이기 때문에, 별도의 인바운드 규칙 없이도 내부 인프라를 대상으로 한 CI/CD를 구성할 수 있다.
- Runner 등록과 Workflow에서의 라벨 지정만으로 온프레미스 환경을 대상으로 하는 CI/CD 파이프라인을 구축할 수 있다.
- 다만 이는 방화벽 문제를 해결하는 대신 보안·운영 부담을 직접 떠안는 선택이므로, 앞서 살펴본 한계를 감수할 수 있는 환경인지 먼저 판단해야 한다.

## 레퍼런스

- [GitHub Actions](https://docs.github.com/ko/actions)
