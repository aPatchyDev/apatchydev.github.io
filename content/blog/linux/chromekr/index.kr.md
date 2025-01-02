+++
date = '2025-01-02T10:18:47Z'
title = 'Chrome 한글 자모분리'
weight = 100
+++

## 환경

- OS: Fedora 41 KDE
- 언어: en-US

## 문제 발견과 해결 방법

처음으로 한글 자모가 분리되는 것을 발견한 것은 유튜브에서였다. 영상 옆 관련 동영상 제목과 댓글에서 발생했고 해결 과정에서 제목 위에 마우스를 올리면 뜨는 tooltip에서도 같은 문제가 생기는 것을 발견했다. 최종적으로 적용한 해결 방법은 아래와 같다.

### 1. Droid Sans Fallback 폰트 비활성화

`$HOME/.config/fontconfig/fonts.conf`에 아래 내용을 붙여넣기

```
<?xml version="1.0"?>
<!DOCTYPE fontconfig SYSTEM "urn:fontconfig:fonts.dtd">
<fontconfig>

  <!-- settings go here -->

<selectfont>
    <rejectfont>
        <pattern>
            <patelt name="family" >
                <string>Droid Sans Fallback</string>
            </patelt>
        </pattern>
    </rejectfont>
</selectfont>

</fontconfig>
```

[fonts.conf 내용물 출처](https://kldp.org/comment/642958#comment-642958)  
[~/.config의 fonts.conf 경로 출처](https://phoikoi.io/2018/04/27/disable-unwanted-fonts-linux.html)

### 2. export LC_CTYPE=ko_KR.UTF-8

`LC_CTYPE` 환경변수는 character set과 관련된 설정만 변경하기 때문에 `LANG`이랑 다르게 다른 프로그램의 기본 언어를 변경하지 않는다.

하이퍼링크위에 마우스를 올리면 뜨는 tooltip은 이 변수가 한국어로 설정되어 있어야 자모 분리가 안 일어난다.

`.bashrc` 등 사용하는 쉘 설정파일에 `export LC_CTYPE=ko_KR.UTF-8` 한 줄 추가한다.

[출처](https://stackoverflow.com/a/53620854)

### 3. font cache 초기화

`fc-cache -r` 명령어 실행

## 실패한 방법

Chrome 설정 > Customize Font > `standard / serif / sans-serif / fixed-width font`를 `CJK KR` 폰트로 변경하기

`standard font`를 `CJK KR`로 변경하면 유튜브 제목과 댓글의 자모분리 현상은 해소되지만 마우스 오버 시 tooltip은 해결되지 않았다. 나머지 폰트는 변경해도 자모분리 현상에는 영향을 주지 않았다.

Chrome 기본 폰트는 선택할 수가 없어 설정 파일에서 사용자가 지정한 내용을 삭제해야 원복할 수 있다. Chrome 설정은 `$HOME/.config/google-chrome/Default/Preferences` 파일에 저장되어 있으며 `"webkit": {"webprefs": {"fonts": {"fixed": {"Zyyy": "xxx CJK KR"}, "standard": {"Zyyy": "xxx CJK KR"}, ...}}}` 부분을 지워주면 초기화된다.
