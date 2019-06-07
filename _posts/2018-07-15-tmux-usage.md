---
layout: post
title: Tmux 사용 방법
author-id: sungup
tags: [tmux, linux, tools]
---

Terminal 상에서 다수의 Window와 Panel 을 만들어 활용하는 Linux 유틸리티입니다.
이름의 유래는 Terminal + Mux 로 여기 Article 은 Tmux 를 활용하면서 유용한
기능들에 대해 정리해 두었습니다.

## Tmux scrolling 하는 방법

Console 상에서 대규모 출력이 발생하는 경우 위아래 스크롤링 하는 방법

* Ctrl-b + [ 로 scrolling mode 로 이동
  * 이 경우 오른쪽 위에 [Back scroll lines / Total lines] 로 표시됨
  * 화살표 방향, Page Up/Down으로 이동
  * q 키로 빠져나올 수 있음.

*reference: [tmuxでconsoleのスクロール(not mouse)を行う方法](https://qiita.com/sutoh/items/41ddd9bdbc9e23746c9d)*
