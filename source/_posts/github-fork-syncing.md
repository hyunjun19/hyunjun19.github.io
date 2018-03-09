---
title: 포크한 깃허브 저장소를 원본 저장소와 동기화 하기
date: 2018-03-09 14:35:48
tags:
---

저는 종종 오픈소스를 포크(fork)한 후 수정해서 pull request를 보내거나 개인적으로 사용하고 있습니다.
하지만 원본 저장소의 변경 내용을 포크(fork)한 제 저장소에 반영할 때 마다 방법을 잊어버려서 기록으로 남겨둡니다.

## 절차

- 원본 저장소를 **원본 저장소**라고 하겠습니다.
- 제가 포크한 저장소를 앞으로는 **포크 저장소**라고 하겠습니다.

1. 로컬로 **포크 저장소**를 clone 합니다.
1. 로컬에 있는 **포크 저장소**에 리모트를 설정해 줍니다.
1. **포크 저장소**에 **원본 저장소**를 머지합니다.

## 실전 CLI

- **원본 저장소** => https://github.com/ax5ui/ax5ui-kernel
- **포크 저장소** => https://github.com/hyunjun19/ax5ui-kernel.git

```bash
// 포크 저장소를 로컬로 클론
$ git clone https://github.com/hyunjun19/ax5ui-kernel.git

// 로컬 저장소로 이동
$ cd ax5ui-kernel

// 현재 설정된 리모트 저장소 조회
$ git remote -v
origin	https://github.com/hyunjun19/ax5ui-kernel.git (fetch)
origin	https://github.com/hyunjun19/ax5ui-kernel.git (push)

// 리모트 저장소 추가
$ git remote add upstream https://github.com/ax5ui/ax5ui-kernel

// 리모트 저장소 확인
$ git remote -v
origin	https://github.com/hyunjun19/ax5ui-kernel.git (fetch)
origin	https://github.com/hyunjun19/ax5ui-kernel.git (push)
upstream	https://github.com/ax5ui/ax5ui-kernel (fetch)
upstream	https://github.com/ax5ui/ax5ui-kernel (push)

// 리모트 저장소 fetch
$ git fetch upstream
remote: Counting objects: 793, done.
remote: Total 793 (delta 692), reused 692 (delta 692), pack-reused 101
Receiving objects: 100% (793/793), 215.62 KiB | 144.00 KiB/s, done.
Resolving deltas: 100% (717/717), completed with 127 local objects.
From https://github.com/ax5ui/ax5ui-kernel
...

// 리모트 저장소 merge
$ git merge upstream/master
Updating 9cdedc67..92c52dab
Fast-forward
...

// 포크 저장소로 push
$ git push
Counting objects: 791, done.
Delta compression using up to 4 threads.
Compressing objects: 100% (141/141), done.
Writing objects: 100% (791/791), 212.18 KiB | 53.05 MiB/s, done.
Total 791 (delta 710), reused 723 (delta 650)
remote: Resolving deltas: 100% (710/710), completed with 99 local objects.
To https://github.com/hyunjun19/ax5ui-kernel.git
   9cdedc67..92c52dab  master -> master
```

끄~~~읏
