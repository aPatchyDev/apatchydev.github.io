+++
date = '2025-02-28T05:40:04Z'
title = '홈 피드 제작'
weight = 100
+++

블로그 첫 화면인 '홈' 페이지가 너무 휑해서 무언가로 채워넣으려고 블로그 글 피드를 만들기로 했다.

## 기본 기능 검토

우선 사용하고 있는 SSG (`Hugo`)와 테마(`ReLearn`)이 기본으로 제공하는 shortcode를 확인해 봤다.

`Relearn` 테마에서는 `children` shortcode를 제공하는데 이는 현재 페이지 기준으로 모든 하위 페이지를 목록으로 보여준다.

이 기능을 사용하기에는 몇 가지 아쉬운 점들이 있었다.

1. 홈 페이지 기준이 아니라 블로그 페이지 기준의 하위 페이지들만 보고 싶다
2. 블로그 카테고리 페이지는 빼고 글을 작성한 포스트만 보고 싶다
3. 하위 페이지가 보여지는 형식을 변경하고 싶다
4. 제목 외에 표시되는 포스트별 추가 정보를 제어하고 싶다

설명을 조금 더 보태자면...

1. 현재 웹사이트 구조상 '홈' 페이지는 가장 상위의 루트 페이지이므로 `소개 / 링크`와 같은 `블로그` 외의 페이지까지 포함하게 된다
    - 내부용 가짜 페이지를 만들고 참조하면 가능하겠지만 다른 단점들을 무시할 만큼 간단하지도 않아서 제외
2. 블로그 내 실제 포스트는 최하단 페이지 뿐이며 중간의 `Projects, Projects/Devblog`와 같은 카테고리 렌딩 페이지는 별 내용이 없어 피드에 포함시킬 이유가 없다
3. `children` shortcode는 목록 형태로만 나열하기 때문에 다른 블로그 플랫폼에서 제공하는 미리보기 카드 형식보다 멋없고 가독성 떨어진다
    - 원하는 `html` 태그를 사용하도록 변경할 수는 있지만 삽입되는 `html`의 구조는 변경할 수 없는 단순 치환에 불과하다
4. `children` shortcode는 제목에 설명을 추가할 수 있다. 이 기능을 제대로 활용하기 위해서는 각 포스트마다 직접 요약본을 삽입해줘야 한다. 원문만 작성하는데도 시간이 걸리는데 추가로 요약하는 수고가 드는 것도 싫고 같은 내용이 중복되면 수정하면서 sync가 안 맞게 될 수도 있다
    - 요약본이 없는 경우에는 기본으로 첫 70자가 요약본으로 활용되지만 글 초반에 테이블이나 이미지가 있으면 요약본이 너무 길어진다

이런 단점들 때문에 결국 `Hugo`가 제공하는 `custom shortcode`를 사용하게 되었다

## Custom Shortcode

`Hugo`에서 제공하는 `custom shortcode`는 `Go template` 기반으로 만들어졌다. [`Hugo` 공식문서](https://gohugo.io/templates/shortcode/)를 참고하면서 만들었다.

### 요구사항

- 원하는 디렉토리의 모든 최하단 페이지만을 모아서
- 마지막 갱신일 기준으로 정렬하고
- 미리보기 카드 형식으로 나열하기

서버에서 동적으로 컨탠츠를 불러오는 피드였다면 Pagination을 추가했겠지만 어차피 SSG로 생성되는 정적 사이트라 페이지 로딩에 차이가 생기는 것도 아니니 생략했다. 대신 최대 카드 개수를 지정할 수 있도록 구현했다.

### 최하단 페이지 탐색

먼저 [`children` shortcode의 구현체](https://github.com/McShelby/hugo-theme-relearn/blob/f39096c62ba3057ba2082e140809b752f88b3cb7/layouts/partials/shortcodes/children.html)를 살펴봤다.

이 구현체는 재귀로 사이드 메뉴를 탐색하면서 하위 페이지들을 찾고 있는 것 같다. `Go template` 문법을 잘 몰라서 정확히 어떻게 선언되어있는지는 모르겠지만 `L30-L56` 사이에서 함수 body와 재귀 호출이 일어나고 있는 것 같다.

`children` shortcode는 사이드 메뉴처럼 목차를 생성하는 것이 목적이므로 해당 메뉴를 순회하도록 작성된 듯하지만 다른 기준으로 정렬을 한다면 DFS / BFS로 순회하는 로직이 더 간단해 보였다.

`Go template`에서는 배열 대신 `slice`를 제공하고 `range`로 루프를 돌릴 수 있게 해준다. 이를 믿고 나이브하게 BFS를 구현하면 아래와 같이 실패하게 된다.

```go-template
{{ $posts := slice }}
{{ $pages := slice .Page }}
{{ range $pages }}
    {{ if eq (len .Pages) }}
        {{ $posts = $posts | append . }}
    {{ else }}
        {{ $pages = $pages | append .Pages }}
    {{ end }}
{{ end }}
```

`range`가 실행된 이후에 원소를 추가해도 처음에 있던 원소만 순회하고 종료된다. 따라서 BFS에서 깊이가 1 증가할 때마다 queue를 교체하거나 DFS에서 재귀로 구현하는 방법밖에 없다. 그러나 재귀를 쓸 정도로 `Go template`에 익숙하지도 않고 어차피 template parser에도 최대 스택 깊이 제한이 걸려있기 때문에 간단하게 BFS로 구현했다.

```go-template
{{ $posts := slice }}
{{ $pages := slice .Page }}
{{ range seq 999 }}
    {{ $nxt := slice }}
    {{ range $pages }}
        {{ if eq (len .Pages) 0 }}
            {{ $posts = $posts | append . }}
        {{ else }}
            {{ $nxt = $nxt | append .Pages }}
        {{ end }}
    {{ end }}

    {{ if eq (len $nxt) 0 }}
        {{ break }}
    {{ else }}
        {{ $pages = $nxt }}
    {{ end }}
{{ end }}
```

덤으로 `Go template`에서 변수는 블록 단위 스코프를 가지며 `:=`를 사용하면 초기화하면서 값을 지정하기 때문에 상위 블록의 변수값을 변경할 때는 `=`을 써야 하며 `seq 9999`는 최대 허용범위를 넘어간다.

### 마지막 갱신일 기준으로 정렬하기

사이드 메뉴에는 순서를 직접 지정한 경우를 제외하고는 파일을 생성한 시간을 기준으로 정렬된다. 블로그를 둘러볼 때는 가장 직관적이지만 나중에 수정을 할 때는 업데이트가 된 사실이 어딘가에는 노출되었으면 하는데 그 어딘가가 피드인 게 가장 적절한 것 같다.

정렬하기 위해서는 당연히 먼저 각 포스트의 마지막 갱신일을 조회해야 한다. 수정할 때마다 직접 날짜를 업데이트할 수 있지만, 잊어버릴 수도 있고 귀찮다. `Hugo`는 가장 최근에 해당 파일을 변경한 커밋의 커밋 시간을 `lastmod` 속성으로 지정하는 기능이 있다. 이를 위해서는 몇 가지 작업을 해줘야 한다.

우선 `Hugo`가 깃 정보를 조회할 수 있게 설정을 업데이트 한다.

```toml{ title="hugo.toml" }
enableGitInfo = true
```

그 다음에 배포 환경에 따라 깃 커밋 정보가 deep clone 되도록 CI/CD 파이프라인을 업데이트해 준다. [참고자료](https://gohugo.io/methods/page/gitinfo/#hosting-considerations)

```diff{ title=".github/workflows/deploy.yaml" }
@@ -20,6 +20,7 @@ jobs:
       - uses: actions/checkout@v4
         with:
           submodules: true
+          fetch-depth: 0
       - name: Setup Hugo
         uses: peaceiris/actions-hugo@v3
         with:
```

마지막으로 앞에서 만든 shortcode에 `Lastmod` 속성 기준으로 정렬하는 코드를 추가한다.

```go-template
{{ $posts := slice }}
{{ $pages := slice .Page }}
{{ range seq 999 }}
    ...
{{ end }}

{{ $posts = sort $posts "Lastmod" "desc" }}
```

### 미리보기 카드 만들기

#### 모양 만들기

디자인은 기본으로 제공되는 `notice` shortcode에서 뜯어왔다.

{{% notice title="test" color="darkslategray" %}}
이렇게 `notice` 블록을 하나 만들어 개발자 도구로 `HTML/CSS` 클래스와 스타일을 알아냈다.

```html
<!-- notice css color를 지정하지 않으면 inline style 속성이 사라지고 class에 default가 추가된다  -->
<details open="" class=" box cstyle notices" style="--VARIABLE-BOX-color: darkslategray;">
    <summary class="box-label">
        Title
    </summary>
    <div class="box-content">
        Body
    </div>
</details>
```
{{% /notice %}}

태그만 전부 `<div>`로 변경해주면 모양은 얼추 완성되지만 `box` 클래스에는 `pointer-events: none;` 스타일이 적용되어 링크를 클릭할 수 없다. 그렇다고 `box` 클래스를 제외하면 테두리와 색상 등 다른 스타일도 없어진다. 복수의 속성을 복제하기보단 문제가 되는 속성 하나만 덮어쓰는 것이 효율적이므로 `pointer-events: unset;` 스타일을 inline html에 추가했다.

```html
<div class="box cstyle default" style="pointer-events: unset;">
    <div class="box-label">제목</div>
    <div class="box-content">미리보기</div>
</div>
```

#### 카드 전체를 링크로 변환

포스트 제목에 `<a>` 태그를 걸면 링크의 기능으로는 완성되지만 카드 크기에 비해 누를 수 있는 유효 공간이 작아 사용성은 꽝이다. 그러나 이미지도 아니고 간단한 기능을 위해 JS를 삽입하고 싶지는 않았다. 평볌한 HTML이면 거부감이 적었겠지만 탬플릿 안에서 작업하려니까 escape character 등으로 인해 가독성이 떨어져서 탬플릿 외의 로직은 최대한 단순하게 유지하고 싶었다. 다행이 HTML만으로 해결할 수 있는 방법을 [stackoverflow](https://stackoverflow.com/a/3494108)
에서 찾았다.

```html
<!-- a 태그의 position: absolute를 담기 위해 부모 태그에는 position: relative를 추가해야 한다 -->
<div class="box cstyle default" style="pointer-events: unset; position: relative;">
    <div class="box-label">제목</div>
    <div class="box-content">미리보기</div>
    <a href="링크">
        <span style="position: absolute; width: 100%; height: 100%; top: 0; left: 0; z-index: 1;"></span>
    </a>
</div>
```

만약 바깥 `<div>`를 `<a>`로 바꾸면 비슷하게 동작하지만 카드 위에 링크 레이어를 덮는 것이 아닌 제목과 미리보기 자체에 링크가 걸린다. 마우스를 위에 올릴 때 글에 밑줄이 그어지는 등의 사소한 차이가 발생한다. CSS로 숨길 수는 있지만 굳이..?

어느 방식을 선택해도 글을 선택할 수 없는 단점이 있는데 아직 해결 방법은 못 찾았다.

#### Grid 레이아웃 적용

한 줄에 카드 하나만 넣기에는 공간이 아깝고, 보기에도 못생겼다. 그래서 모든 카드를 담을 `<div>`에 `display: grid` 스타일을 적용시키기로 했다.

```html
<div style="display: grid; grid-gap: 1lh 2em; grid-template-columns: repeat(auto-fit, minmax(20em, 1fr));">
    <!--   탬플릿이 생성할 각 각의 카드   -->
</div>
```

Grid 레이아웃을 적용하고 나니 2가지 문제가 드러났다.

1. 바깥 `<div>`에서 `grid-gap`의 위아래 간격이 잘 맞지 않는다
2. 같은 줄에 있는 카드의 내용의 길이가 다른 경우 더 짧은 미리보기 카드의 배경이 남는 공간을 채우지 않는다

[]()

1. 각 카드의 `margin`을 0으로 설정하면 된다
2. 카드의 바깥 `<div>`를 `flexbox`로 변경하고 `<div class="box-content">`에 `flex: 1`을 적용하면 된다

```html{ title="카드 HTML" }
<div class="box cstyle default" style="pointer-events: unset; position: relative; margin: 0; display: flex; flex-direction: column;">
    <div class="box-label">제목</div>
    <div class="box-content" style="flex: 1;">미리보기</div>
    <a href="링크">
        <span style="position: absolute; width: 100%; height: 100%; top: 0; left: 0; z-index: 1;"></span>
    </a>
</div>
```

#### 카드 내용 채우기

포스트 미리보기인만큼 카드에 담을 내용은 `블로그 카테고리, 제목, 요약` 정도가 된다.

블로그 카테고리는 최하단 카테고리만 출력하려면 `.Parent.Title`을 하면 된다. 그러나 블로그에서의 전체 경로를 출력하려면 부모 트리를 거슬러 올라가야 한다.

```go-template
{{ $src := /* 블로그 최상단 경로 */ }}

{{ $parents := slice }}
{{ range .Ancestors }}
    {{ if eq . $src }}
        {{ break }}
    {{ end }}
    {{ $parents = $parents | append .Title }}
{{ end }}
{{ $parents = delimit ($parents | collections.Reverse) "/" }}
```

카드의 제목란에 카테고리, 미리보기로 포스트 제목만 놓차니 너무 휑하다. 그러나 `Hugo의 Description / Summary`는 쓰고 싶지 않았다. 그래서 [Page의 메소드 목록](https://gohugo.io/methods/page/)을 보니 `TableOfContents`가 보였다. 이거야말로 글의 내용을 적절히 요약해주지 않을까 싶어 선택했다.

글이 길어지고 모든 heading을 다 포함하면 미리보기도 길어지겠지만 기본 설정은 h3까지만 보여주기 때문에 본문에는 h2부터 사용하는 이 블로그에 안성맞춤이다.

하지만 각 heading에 걸려있는 링크는 거슬렸기 때문에 아래와 같이 빼버렸다.

```go-template
{{ .TableOfContents | strings.ReplaceRE "</?a.*?>" "" | safeHTML }}
```

마지막에 `safeHTML`을 안 쓰면 `Go template`에서 xml injection으로 간주하고 `ZgotmplZ`로 바꿔버린다. 따라서 안전한 코드라고 알려주기 위해 추가해야 한다.

미리보기에 목록이 해당 글의 목차임을 밝히는 prefix도 추가했다.

### 카드 최대 개수 제한 구현

개수를 제한하는 기능은 `slice.First`로 간단하게 구현할 수 있다. 단, 피드에서 생략이 된 카드가 있다는 정보를 효과적으로 전달할 필요가 있다. 그러기 위해서 생략된 카드의 개수도 기록해야 한다.

```go-template
{{ $maxcnt:= /* 카드 최대 개수 */ }}
{{ $posts = ... }}

{{ $truncated := 0 }}
{{ if gt $maxcnt 0 }}
    {{ $truncated = sub (len $posts) $maxcnt }}
    {{ $posts = $posts | first $maxcnt }}
{{ end }}

{{/* 카드 생성 */}}
```

생략된 카드의 개수도 또 하나의 카드처럼 만들면 새로 디자인할 수고를 줄일 수 있으나 진짜 포스트와 구분이 안 되니 제목과 미리보기로 나누지 않고 제목과 같은 색의 단색 카드로 만들었다.

```go-html-template
{{/* 카드 생성 */}}

{{ if gt $truncated 0 }}
    <div class="box cstyle default" style="pointer-events: unset; position: relative; margin: 0; display: flex; flex-direction: column;">
        <h1 style="height: 100%; text-align: center; align-content: center;">+ {{ $truncated }}</h1>
    </div>
{{ end }}
```

### 최종 shortcode

모든 컴포넌트를 합치고 shortcode로 쓰기 위한 옵션과 인자 처리를 마치면 아래와 같이 완성된다.  
'홈' 페이지에서는 아래와 같이 shortcode로 사용할 수 있다.  
- 덤으로 이 글에서 코드 블럭에 shortcode 예제를 삽입하기 위해 `{{</*/* 이렇게 작성해야 */*/>}}` 아래처럼 변환된다. [출처](https://discourse.gohugo.io/t/a-way-to-mark-plain-text-and-stop-hugo-from-interpreting/1325/4)

```go-template
{{/* source: 절대경로 또는 현 페이지 기준 상대경로 */}}
{{/* tocprefix: 미리보기 목차 위에 삽입할 문자열 */}}
{{/* color: CSS color */}}
{{/* limit: 최대 카드 개수 */}}

{{</* feed source="/blog" tocprefix="[목차]" color="darkslategray" limit=10 */>}}
```

```go-html-template{ lineno=True title="layouts/shortcodes/feed.html" }
{{/*
    $src: Root page of feed
    $tocprefix: Text to prepend in each feed entry's ToC
    $color: CSS color value for styling feed entry title
    $maxcnt: Max number of feed entry
*/}}
{{ $src := .Get "source" | default "." | .Page.GetPage }}
{{ $tocprefix := .Get "tocprefix" | default "" }}
{{ $color := .Get "color" | default "" }}
{{ $maxcnt:= .Get "limit" | default 0 | int }}

{{/*
Perform BFS with high max depth since while loop cannot be used
    $posts: Queue for holding leaf pages
    $pages: Queue for holding intermediate pages
*/}}
{{ $posts := slice }}
{{ $pages := slice $src }}
{{ range seq 999 }}
    {{ $nxt := slice }}
    {{ range $pages }}
        {{ if eq (len .Pages) 0 }}
            {{ $posts = $posts | append . }}
        {{ else }}
            {{ $nxt = $nxt | append .Pages }}
        {{ end }}
    {{ end }}

    {{ if eq (len $nxt) 0 }}
        {{ break }}
    {{ else }}
        {{ $pages = $nxt }}
    {{ end }}
{{ end }}

{{/*
Sort by last modified date and apply limit if necessary
    $truncated may be negative if entry count is less than a positive $maxcnt
        Check for truncation by using greater-than rather than equals
*/}}
{{ $posts = sort $posts "Lastmod" "desc" }}
{{ $truncated := 0 }}
{{ if gt $maxcnt 0 }}
    {{ $truncated = sub (len $posts) $maxcnt }}
    {{ $posts = $posts | first $maxcnt }}
{{ end }}

{{/*
Generate feed
*/}}
<div style="display: grid; grid-gap: 1lh 2em; grid-template-columns: repeat(auto-fit, minmax(20em, 1fr));">
{{ $boxstyle := "pointer-events: unset; position: relative; margin: 0; display: flex; flex-direction: column;" }}
{{ range $posts }}
    {{/*
    Get relative path to $src
    */}}
    {{ $parents := slice }}
    {{ range .Ancestors }}
        {{ if eq . $src}}
            {{ break }}
        {{ end }}
        {{ $parents = $parents | append .Title }}
    {{ end }}
    {{ $parents = delimit ($parents | collections.Reverse) "/" }}

    {{/*
    Generate preview entry
    */}}
    {{ if eq $color "" }}
        {{ (printf `<div class="box cstyle default" style="%s">` $boxstyle) | safeHTML }}
    {{ else }}
        {{ (printf `<div class="box cstyle" style="%s --VARIABLE-BOX-color: %s;">` $boxstyle $color) | safeHTML }}
    {{ end }}
        <div class="box-label">
            [{{ $parents }}] {{ .Title }}
        </div>
        <div class="box-content" style="flex: 1;">
            {{ if ne $tocprefix "" }}
                <p>{{$tocprefix}}</p>
            {{ end }}
            {{ .TableOfContents | strings.ReplaceRE "</?a.*?>" "" | safeHTML }}
        </div>
        <a href="{{ .RelPermalink }}">
            <span style="position: absolute; width: 100%; height: 100%; top: 0; left: 0; z-index: 1;"></span>
        </a>
    </div>
{{ end }}

{{ if gt $truncated 0 }}
    {{/*
    Display truncation info if applicable
    */}}
    {{ if eq $color "" }}
        {{ (printf `<div class="box cstyle default" style="%s">` $boxstyle) | safeHTML }}
    {{ else }}
        {{ (printf `<div class="box cstyle" style="%s --VARIABLE-BOX-color: %s;">` $boxstyle $color) | safeHTML }}
    {{ end }}
        <h1 style="height: 100%; text-align: center; align-content: center;">+ {{ $truncated }}</h1>
    </div>
{{ end }}
</div>
```
