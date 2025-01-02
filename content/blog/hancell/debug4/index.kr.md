+++
date = '2024-12-14T09:13:40Z'
linktitle = '디버깅 한셀 (4)'
title = '디버깅 한셀 (4) - 전역변수'
weight = 100
+++

## 전역변수 문법

전역변수 문법에 관한 자료를 찾기가 어려워 예전에 어느 stackoverflow 질문글에서 찾은 예제는 아래와 같다.  
링크를 저장 안 했더니 다시 찾지는 못했다.

{{< tabs title="전역변수 예제" >}}
{{% tab title="Module1" %}}
```vb { lineNos="true" }
Option Explicit
Dim a
Public b
Private c
Public Const d = #12/31/2024#
Private Const e = "Hi"

Sub Test()
    a = 1
    b = True
    c = #1:00:00 PM#
    MsgBox Join(Array(a,b,c,d,e), vbNewLine)
End Sub
```
{{% /tab %}}
{{< /tabs >}}

전역변수 선언문에서 사용할 수 있는 modifier는 다음과 같다.

| Modifier | Const 가능여부 | 설명 |
| --- | --- | --- |
| Public | 가능 | 다른 모듈에서도 사용할 수 있는 전역변수 |
| Private | 가능 | 현재 모듈에서만 사용할 수 있는 전역변수 |
| Dim | 불가능 | 현재 모듈에서만 사용할 수 있는 전역변수 |

`Dim vs Private`의 기능이 겹치는 이유는 VB 언어가 발전하면서 access modifier가 뒤늦게 추가되었기 때문이라고 한다. 한셀과 같이 모듈을 지원하는 프로그램에서는 전역변수에서 `Public / Private`만 쓸 것을 권장한다는 [stackoverflow 답변](https://stackoverflow.com/a/23911728)이 있다.

함수 밖에서는 변수 정의문을 쓸 수 없다. 따라서 한 줄에 선언과 정의를 같이하는 `Dim x: x = 1` 같은 코드 또한 지역변수에서만 사용 가능하고 전역변수는 안 된다.

상수는 함수로부터 값을 받을 수 없다. 따라서 `Array(원소1, 원소2, ...)` 같이 원소가 포함된 배열이나 `CreateObject(...)` 같은 객체를 생성하여 할당할 수는 없다.

## 선언 위치

변수나 상수를 선언할 때는 필요한 코드에 최대한 인접한 곳에서 선언하는 것이 일반적이다.

{{< tabs groupid="exp1" title="실험 코드" >}}
{{% tab title="1-A" %}}
```vb { lineNos="true" }
Option Explicit
Sub Test()
    MsgBox "Test"
    Call Foo()
End Sub

Private Const x = "Hi"

Sub Foo()
    MsgBox x & " Foo"
End Sub
```
{{% /tab %}}

{{% tab title="1-B" %}}
```vb { lineNos="true" }
Option Explicit
Private Const x = "Hi"

Sub Test()
    MsgBox "Test"
    Call Foo()
End Sub

Sub Foo()
    MsgBox x & " Foo"
End Sub
```
{{% /tab %}}

{{% tab title="1-C" %}}
```vb { lineNos="true" }
Option Explicit
Sub Test()
    MsgBox "Test"
    Call Foo()
End Sub

Private Const x = "Hi"
```
{{% /tab %}}
{{< /tabs >}}

{{< tabs groupid="exp1" title="실험 결과" >}}
{{% tab title="1-A" %}}
```
컴파일 오류:
End Sub, End Function 또는 End Property 다음에는 주석만 나타날 수 있습니다.
```
{{% /tab %}}

{{% tab title="1-B" %}}
```
Test
Hi Foo
```
{{% /tab %}}

{{% tab title="1-C" %}}
```
컴파일 오류:
End Sub, End Function 또는 End Property 다음에는 주석만 나타날 수 있습니다.
```
{{% /tab %}}
{{< /tabs >}}

[실험1-A]와 [실험1-B]는 전역변수 `x`의 선언 위치만 다를 뿐이다. [실험1-A]는 `Test` 함수와 `Foo` 함수 사이에, [실험1-B]는 최상단에 선언되었다. 그러나 [실험1-A]는 오류로 실행조차 되지 않느다.

믿을 수가 없어 [실험1-C]를 전행했으나 역시 [실험1-A]와 결과가 같다. 전역변수는 그 어떤 함수보다 위에 선언해야만 된다.

```
호이스팅은 셀프입니다~ 🙏
```

## Shadowing

{{< tabs title="실험 2" >}}
{{% tab title="Module1" %}}
```vb { lineNos="true" }
Option Explicit                 '전역변수 예제와 동일함
Dim a
Public b
Private c
Public Const d = #12/31/2024#
Private Const e = "Hi"

Sub Test()
    a = 1
    b = True
    c = #1:00:00 PM#
    MsgBox Join(Array(a,b,c,d,e), vbNewLine)
End Sub
```
{{% /tab %}}

{{% tab title="Module2" %}}
```vb { lineNos="true" }
Option Explicit
Sub Foo()
    MsgBox b
    b = "Hi"
    Dim b
    b = "Hello"
    MsgBox b
    Call Bar()
End Sub

Sub Bar()
    MsgBox b
End Sub
```
{{% /tab %}}
{{< /tabs >}}

`Foo` 함수 실행으로 실험한다.

{{% tab title="실험 결과 1회차" %}}
```text { lineNos="true" }
​
Hello
Hi
```
{{% /tab %}}

{{% tab title="실험 결과 2회차 이상" %}}
```text { lineNos="true" }
Hi
Hello
Hi
```
{{% /tab %}}

`Module2` 줄 #3~#4는 `Module1`에서 선언한 전역변수를 기리키고  
`Module2` 줄 #5~#7은 `Foo` 함수에서 선언한 지역변수를 가리킨다.

이때 `Foo` 함수의 지역변수가 전역변수를 덮어씌우는지, shadowing만 하는지 확인하기 위해 `Bar` 함수에서 변수 `b`를 출력해보면 `Module2` 줄 #4에서 출력한 값과 같게 나오는 것으로 보아 shadowing만 발생한다는 것을 알 수 있다.

`Foo` 함수를 다시 실행하면 `Module` 줄 #3에서 전역변수가 이전 회차에서 정의한 값을 가지고 있는 것으로 보아 전역변수는 `한셀 ver군대` 지역변수의 `ReDim` 변수처럼 ([이전 2편 참고](../debug2/#redim-배열-변수)) 매크로가 종료되어도 한셀 매크로 엔진이 재시작할 때까지 값이 남는 것을 확인할 수 있다.
