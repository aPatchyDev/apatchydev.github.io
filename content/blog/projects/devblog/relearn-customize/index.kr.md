+++
date = '2024-11-28T07:00:06Z'
title = 'Relearn 커스터마이징'
weight = 100
+++

## 사이드바 메뉴 정렬하기

모든 설정이 끝났을 거라고 생각하고 이 글을 작성했더니 좌측 사이드바 메뉴에서 최신순으로 글이 나열된 것을 발견했다.

오래된순으로 나열되기를 원해 방법을 찾기 시작했다. 물론, 직접 매번 `weight`를 지정하면 원하는대로 정렬이 가능하지만, 그건 너무 번거롭다.

{{% notice style=info title=참고문서 %}}
이 테마가 지원하는 정렬 방법은 [여기](https://mcshelby.github.io/hugo-theme-relearn/configuration/sidebar/menus/index.html#ordering-menu-entries)에 나와있다.
{{% /notice %}}

이 설정은 한 가지 값만 받기 때문에 2가지 기준을 적용하는 유일한 값은 `default`였다. 하지만 Hugo의 `default`는 동일 `weight`에 대해 최신순으로 정렬한다.

원하는 대로 정렬하는 기능을 추가하기 위해서는 기본 레이아웃을 덮어써야 했다.

우선, 최상위 정렬 설정명은 `params.ordersectionsby`이므로 Relearn 레포지토리에서 [검색을 했다](https://github.com/search?q=repo%3AMcShelby%2Fhugo-theme-relearn%20params.ordersectionsby&type=code).

발견된 파일은 단 1개: `layouts/partials/_relearn/pages.gotmpl`

이 파일을 동일한 경로에 저장한 다음, 아래의 내용을 추가했다. [이 블로그 예시](https://github.com/aPatchyDev/apatchydev.github.io/blob/95bee3dea3700b5d8c22ed849726133931684f54/layouts/partials/_relearn/pages.gotmpl#L27-L32)  
[참고자료](https://discourse.gohugo.io/t/sort-posts-by-weight-then-by-title/44937/2)

```diff
@@ -23,8 +23,14 @@
 {{- else if eq $by "length" }}
        {{- $pages = $page.Pages.ByLength }}
 {{- else if eq $by "default" }}
        {{- $pages = $page.Pages }}
+{{- else if eq $by "weighted-old"}}                        # 새로운 정렬 옵션명: "weighted-old"
+       {{- range $page.Pages.GroupBy "Weight" }}
+               {{- range .Pages.ByDate}}
+                       {{- $pages = $pages | append . }}
+               {{ end }}
+       {{ end }}
 {{- else }}
        {{- warnf "%q: Unknown pages sort order '%s'" $page.File.Filename }}
        {{- $pages = $page.Pages }}
 {{- end }}
```

{{% notice style=warning title=경로주의 %}}
`layouts/partials/pages.gotmpl`에 만들면 동작하지 않고 빌드 오류가 발생한다.

반드시 `layouts/partials/_relearn/pages.gotmpl`에 변경된 레이아웃 파일을 저장해야 한다.
{{% /notice %}}

{{% notice style=note title=미지원 %}}
[공식문서](https://mcshelby.github.io/hugo-theme-relearn/configuration/customization/partials/index.html)에서 안전하게 변경해도 되는 파일 목록에 포함되어 있지 않으므로 업데이트 시 주의해야 하며
이 테마의 버전을 직접 지정하는 것을 추천한다.

`go.sum` 삭제 이후 `hugo mod get -u github.com/McShelby/hugo-theme-relearn@<release tag>` 명령어를 실행해 버전을 지정한다.

예: `hugo mod get -u github.com/McShelby/hugo-theme-relearn@7.1.1`
{{% /notice %}}

## Word wrapping

### 본문에서의 단어 단위 word wrap

기본 설정에서는 한글 본문은 글자단위로 word wrap이 이루어진다. 이걸 단어 단위로 변경하려면 아래의 CSS를 추가해줘야 한다.

{{% tab title='layouts/partials/custom-header.html' %}}
```css
<style>
    div {
        word-break: keep-all;
    }
</style>
```
{{% /tab %}}

### 코드 블록에서의 word wrap 해제하기

기본 설정에서는 코드 블록에서 word wrap이 활성화되어 있다. 줄 번호가 꺼져있는 경우(=기본설정)에는 코드를 잘못 이해할 수도 있다.

설정은 `hugo.toml`에서 꺼주면 된다.

{{% tab title='hugo.toml' %}}
```toml
[params]
  highlightWrap = false
```
{{% /tab %}}

## 본문 폭 변경하기

글 본문은 기본으로 최대 폭을 제한하고 있어 큰 모니터에서는 양 옆에 공간이 남는다. 이를 변경하려면 CSS를 추가하면 된다.  
[참고자료](https://mcshelby.github.io/hugo-theme-relearn/configuration/content/width/index.html)

{{% tab title='layouts/partials/custom-header.html' %}}
```css
<style>
    :root {
        --MAIN-WIDTH-MAX: 85rem;
    }
</style>
```
{{% /tab %}}

## h2 글꼴 변경하기

Relearn 테마에서 페이지의 제목은 h1과 같은 글꼴을 공유한다. 가운데 정렬이라 글의 흐름을 깨기 때문에 글 본문에서는 h2부터 쓰는데 h3 이하랑 크기만 달라 구분이 잘 안 되는 것이 불만이었다. 그래서 밑줄을 추가했다.

{{% tab title='layouts/partials/custom-header.html' %}}
```css
<style>
    h2 {
        text-decoration: underline;
    }
</style>
```
{{% /tab %}}
