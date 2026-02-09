---
title: "worktree 세팅/정리 귀찮음 끝: opencode-worktree 플러그인"
date: "2026-02-09"
tag:
  - llm
  - opencode
  - worktree
draft: true
---

나는 작업을 할 때 opencode를 활용하고 있다. 그리고 한 프로젝트에 대해 병렬적으로 여러 작업을 동시에 진행하고 있다. 새로운 피쳐 작업과 운영 이슈 대응 등을 병렬적으로 작업하는 것은 흔한 일이다.

이를 위해 git worktree를 써왔는데, 이 과정에서 느꼈던 번거로운 점과 그 해결책으로 사용중인 opencode-worktree 플러그인에 대해 공유한다.

### 배경: git worktree를 수동으로 쓰면 번거로운 점
같은 프로젝트에서 여러 작업을 병렬로 진행할 때 git worktree가 유용하다.
하지만 수동으로 구성하려면 매번 아래 과정을 반복해야 한다.

1. 워크트리가 생성될 경로를 직접 지정하고 `git worktree`로 워크트리 생성
2. 새 터미널을 열고 생성된 워크트리 디렉토리로 `cd`
3. `.env.local` 복사, `node_modules` 설치 등 개발환경에 필요한 파일 직접 구성
4. opencode를 다시 열어서 작업 시작

추가로,
- worktree 경로/이름을 매번 직접 고민해야 하고
- 작업이 끝나면 worktree 정리(remove/prune 등)도 직접 해야 해서, 병렬 작업 시작/종료가 번거로움
### 해결: opencode-worktree 플러그인 사용

[worktree 플러그인](https://github.com/kdcokenny/opencode-worktree)을 쓰면, **프롬프트만으로 worktree 생성부터 환경 구성, 터미널 오픈까지 자동화**된다.
#### 1) 생성

“이 작업을 할 건데 워크트리 만들어서 작업해줘” 같은 프롬프트만으로:
- 워크트리 네이밍 자동 생성
- 워크트리 생성 + 개발환경에 필요한 파일 구성 (symlink)
- 생성된 워크트리 디렉토리를 기준으로 **새 터미널 자동 오픈**
    - tmux를 쓰는 경우 tmux 탭으로 열어줌
    - 그 외의 경우 새 터미널 탭으로

#### 2) 정리
작업이 끝나면 “워크트리 정리해줘” 같은 프롬프트만으로:
- 해당 워크트리 삭제(delete)