+++
date = '2024-11-17T14:51:14Z'
linktitle = '초기설정'
title = '개발블로그 시작과정'
weight = 100
+++

## 테마 희망사항

- 웹사이트를 탐색하기에 용이한 사이드바
- 페이지를 카테고리로 분류하고
- 사이드바의 카테고리를 접을 수 있길 바랐다

무조건 최신순이 아니라 직접 페이지를 정리하고 싶었기 때문에 blog 테마보다는 documentation 테마가 더 마음에 들었다.

## 1트: Jekyll Gitbook

처음에는 깃헙과 연동이 잘 되어있는 jekyll을 사용하려고 했다.

그 중 [Gitbook](https://github.com/sighingnow/jekyll-gitbook)가 깔끔해서 이걸 사용하려고 했었다.

그런데 막상 글을 쓰려고 보니 nested collection을 지원하지 않았다.

무슨 말이냐하면 아래의 페이지 구조를 만들 수 없었다:
```
희망사항
/root/
|
+ - blog/
    |
    + - topic1/         -> /blog/topic1.html
        |
        + - page1.md    -> /blog/topic1/page1.html
        + - page2.md    -> /blog/topic1/page2.html

실제로 가능한 것
/root/
|
+ - blog/
    |
    + - page1/          -> blog/page1.html
        |
        + - heading1    -> blog/page1.html#heading1
        + - heading2    -> blog/page1.html#heading2
```

원래의 gitbook은 지원하는 것으로 보이지만 jekyll port는 안 됐기 때문에 다른 대안을 찾기 시작했다.

## 2트: 다른 Jekyll 테마 탐색

Read the Docs 테마도 좋아 보여 jekyll 테마를 찾아보았지만 몇 가지 이유로 선택하지 않았다.

- 활발히 개발되고 있지 않았다
    - 케이스1 (rundocs org): 깃헙 공중분해 (다른 fork도 아직 이 망가진 링크를 참조하는 중)
    - [케이스2](https://github.com/carlosperate/jekyll-theme-rtd): 마지막 commit 3년 전
    - [케이스3](https://github.com/JV-conseil/jekyll-theme-read-the-docs): 최근 commit은 있으나 기능 구현도 아니고 issues/PR도 적다
- 본문이 화면 전체를 다 채우지 않았다
- 참고할 문서 및 예제 부족

다음에 Just the Docs도 봤지만 사이드바가 왼쪽 끝에 붙어있지도 않고 본문도 화면을 안 채워서 선택하지 않았다.

물론 레이아웃을 직접 수정하면 고칠 수는 있겠지만 그런 수고를 최소화하기 위해 SSG + 테마를 사용하는 거기 때문에 가능/불가능은 그닥 중요하지  않았다.

단지 얼마나 사용하기 편한지 뿐.

## 3트: Hugo Relearn

다른 SSG 테마들을 보던 중 [Hugo Themes](https://themes.gohugo.io/tags/docs/)를 보게 됐다.

Docs 테마는 (작성시점 기준) 26개밖에 없었기 때문에 다 볼 수 있었는데 마침 multi-lingual 기능이 보여 요구사항에 추가해버렸다!

추후 작성할 내용들 중 영어로도 작성하고 싶은 내용들이 있었는데 마침 잘 됐다.

그래서 최종적으로 선택하게 된 테마가 바로 Relearn 테마다.

물론, 아직도 조금 아쉬운 부분이 있다. 바로 Incremental Build가  안 된다는 점.

원인은 [사이드바 접기 기능 때문에...](https://mcshelby.github.io/hugo-theme-relearn/configuration/sidebar/menus/index.html#expander-for-submenus)

단순히 생각했을 때 ToC를 따로 랜더링하고 iframe으로 각 페이지에 삽입했다면 됐을 거 같은데... 아닌가..?

아무튼 원하는 기능은 다 있었기 때문에 일단 사용하고 빌드 시간이 문제가 된다면 그건 나중에 생각하기로 했다.
