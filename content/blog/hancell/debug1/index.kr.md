+++
date = '2024-11-28T11:51:43Z'
linktitle = '디버깅 한셀 (1)'
title = '디버깅 한셀 (1) - 지역변수의 유효범위'
weight = 100
+++

언어에 따라 지역변수의 유효범위가 다른 경우가 일부 존재한다. 일반적으로는 선언문부터 유효하며 선언문이 속한 블록, 또는 함수가 종료된 이후에는 유효하지 않다. Python의 경우에는 선언문이 따로 없어 처음으로 값을 정의할 때 선언을 동시에 하며 Javascript의 경우에는 선언문을 함수 최상단으로 올려버리는 `hoisting` 특징이 있다.

{{% notice style="note" title="주의할 점" %}}
이번 편에서 다루는 `유효범위`란 변수가 잘 정의되어 있는 (well-defined) 범위를 의미한다.  
그것이 특수값이든, 역참조 시 오류를 던지든, 일정하면 된다.  
데이터가 실제로 생성되고 폐기되는 정확한 시점인 `수명`은 다음 편에서 다룰 예정이다.
{{% /notice %}}

## 호이스팅 검증

### 함수 단위의 호이스팅

{{< tabs groupid="exp1" title="실험 코드" >}}
{{% tab title="1-A" %}}
```vb { lineNos="true" }
Option Explicit
Sub Test()
    MsgBox a
    Dim a
    a = "Hi"
End Sub
```
{{% /tab %}}

{{% tab title="1-B" %}}
```vb { lineNos="true" }
Option Explicit
Sub Test()
    MsgBox "Test"
    On Error Resume Next
    MsgBox a
    Dim a
    a = "Hi"
End Sub
```
{{% /tab %}}

{{% tab title="1-C" %}}
```vb { lineNos="true" }
Option Explicit
Sub Test()
    Dim a
    MsgBox a
    a = "Hi"
End Sub
```
{{% /tab %}}
{{< /tabs >}}

{{< tabs groupid="exp1" title="실험 결과" >}}
{{% tab title="1-A" %}}
```
컴파일 오류:

a(은)는 정의되지 않았습니다.
```
{{% /tab %}}

{{% tab title="1-B" %}}
```
컴파일 오류:

a(은)는 정의되지 않았습니다.
```
{{% /tab %}}

{{% tab title="1-C" %}}
```text { lineNos="true" }
​
```
{{% /tab %}}
{{< /tabs >}}

[실험1-A] 줄 #3을 주석처리하면 오류가 안 나는 것으로 보아 해당 줄이 오류의 원인임을 확인할 수 있다. 여기서 알 수 있는 사실은 선언문 이전에는 변수에 `접근`할 수 없다는 점이다. 여기에서 다시 3가지 가설로 나눌 수 있다:

1. 호이스팅은 이루어지지만 선언문의 원위치 이전에 접근 시 런타임 오류를 던진다
2. 호이스팅은 이루어지지만 정의문 이전에 접근 시 런타임 오류를 던진다 (JS `let`와 동일)
3. 호이스팅이 없어서 실행되기 전에 문법 오류를 던진다.

이 가설들은 [실험1-B]와 [실험1-C]로 검증할 수 있다. `On Error Resume Next`는 VBS의 `try/catch`에 해당한다. 단, 별도의 `catch` 없이 모든 에러를 무시하고 실행을 이어간다.

가설 #1이 맞을 경우, [실험1-B]에서는 최소한 줄 #3에서 `Test`가 출력이 된 다음에 종료되어야 한다. 그러나 아무런 출력 없이 [실험1-A]와 동일한 오류가 발생하므로 이 가설은 틀렸다.

가설 #2이 맞을 경우, [실험1-C]에서도 오류가 발생해야 한다. 그러나 오류가 발생하지 않고 기본값 (빈 문자열)이 출력되므로 이 가설 또한 틀렸다.

따라서 가설 #3 - `호이스팅이 없다`가 정답일 것이다.

엄밀히 말하자면 `Superscalar / Out-of-order execution이 없거나 최소한 사용자에게는 그 효과가 노출되지 않는다`는 전제가 깔려있지만 병렬처리하는 것도 아닌 순차 실행에서조차 이 전제를 깨는 프로그래밍 언어는 모르기 때문에 굳이 검증하지는 않겠다.

### 블록 단위의 호이스팅

호이스팅의 대표적인 예가 JS라서 함수 단위를 먼저 살펴봤지만 블록 단위로도 비슷한 실험을 할 수 있다.

{{< tabs groupid="exp2" title="실험 코드" >}}
{{% tab title="2-A" %}}
```vb { lineNos="true" }
Option Explicit
Sub Test()
    Dim a
    MsgBox a < 1
    MsgBox a > 1
End Sub
```
{{% /tab %}}

{{% tab title="2-B" %}}
```vb { lineNos="true" }
Option Explicit
Sub Test()
    If Date() > #1/1/2000# Then '항상 True
        MsgBox "Check"
        Msgbox a
        Dim a: a = 2    '선언과 정의를 같은 줄에 쓰는 VB 문법
    End If
    
    MsgBox "Done"
End Sub
```
{{% /tab %}}

{{% tab title="2-C" %}}
```vb { lineNos="true" }
Option Explicit
Sub Test()
    If a > 0 Then
        Dim a: a = 2    '선언과 정의를 같은 줄에 쓰는 VB 문법
    End If
    
    MsgBox "Done"
End Sub
```
{{% /tab %}}
{{< /tabs >}}

{{< tabs groupid="exp2" title="실험 결과" >}}
{{% tab title="2-A" %}}
```text { lineNos="true" }
True
False
```
{{% /tab %}}

{{% tab title="2-B" %}}
```
컴파일 오류:

a(은)는 정의되지 않았습니다.
```
{{% /tab %}}

{{% tab title="2-C" %}}
```
컴파일 오류:

a(은)는 정의되지 않았습니다.
```
{{% /tab %}}
{{< /tabs >}}

[실험2-A]는 정의되지 않은 변수를 정수와 대소비교를 한 결과를 보여준다. 오류가 나지 않는다는 것만 보인다면 [실험2-C]에서 사용할 수 있으므로 충분하다. 만약 undefined behavior 라고 간주하고 어떤 결과든 발생할 수 있다고 치면 이미 그 시점에서 호이스팅이 없다고 하는 것과 마찬가지다

[실험2-B]는 if문 안에서 선언된 변수가 if문의 상단까지 호이스팅되는지 확인하는 실험이다.  
덤으로 [실험1-B]와 비슷하게 해당 블록이 실제로 실행되어야 오류가 발생하는지도 같이 확인한다. 결과는 블록 단위 역시 실행되지 않아도 오류를 던지는 것으로 보아 호이스팅은 없다.  
줄 #3 if 조건절을 위와 같이 작성한 이유는 `if True`가 최적화로 if 블록이 제거되는 것을 방지하기 위해서 값이 런타임에 결정되도록 한 것이다.

[실험2-C]는 if문 안에서 선언된 변수가 if문의 조건절까지 호이스팅되는지 확인하는 실험이다.

{{% notice style="note" title="호이스팅 정리" %}}
1. 선언문 호이스팅은 없다.
2. 선언문 이전에 변수에 접근하면 문법 오류로 매크로가 실행이 안 된다.
3. 선언문과 정의문 사이에서 변수에 접근하면 오류 없이 기본값을 반환한다. (또는 undefined behavior)
{{% /notice %}}

## 선언문은 실행되어야 하는가..?

```vb { title="실험 3" lineNos="true" }
Option Explicit
Sub Test()
    If Date() < #1/1/2000# Then '항상 False
        Dim a: a = "Hi"
    End If
    
    MsgBox a
    a = "Hello"
    MsgBox a
End Sub
```

```text { title="실험 결과 3" lineNos="true" }
​
Hello
```

`Option Explicit`이 있기 때문에 변수 `a`는 줄 #7 이전에 선언이 되었음을 보장받는다. 그러므로 설령 정의문 이전에 접근하는 것이 undefined일지라도 줄 #8~#9는 합법이라는 말이다. 선언문은 실행되지 않아도 존재하기만 하면 된다.

어메이징!

## 선언문은 어디까지 유효한가..?

일반적인 언어에서 지역변수는 선언된 블록 안에서만 사용할 수 있고 해당 블록이 종료되면 지역변수 또한 사용할 수 없다. 하지만 이미 [실험3]에서 선언된 `if` 블록이 종료된 이후에도 유효함을 보였다. 선언문과 동일한 블록은 당연히 유효할 것이므로 남은 것은 분기점에서 선언문과 다른 실행 경로의 블록밖에 없다.

한셀이 지원하는 VBS의 블록 구문은 `if / select / with`밖에 없다. 이 중 분기가 없는 `with`를 제외하면 `if / select` 두 개다.

{{< tabs groupid="exp4" title="실험 코드" >}}
{{% tab title="4-A" %}}
```vb { lineNos="true" }
Option Explicit
Sub Test()
    If Date() < #1/1/2000# Then '항상 False
        Dim a: a = 2
    ElseIf a < 1 Then
        MsgBox a
    Else
        MsgBox "Else"
    End If
End Sub
```
{{% /tab %}}

{{% tab title="4-B" %}}
```vb { lineNos="true" }
Option Explicit
Sub Test()
    Select Case 3
        Case 1
            Dim a: a = "Hi"
        Case 2
            a = "Hello"
        Case Else
            MsgBox a
    End Select
End Sub
```
{{% /tab %}}
{{< /tabs >}}

{{< tabs groupid="exp4" title="실험 결과" >}}
{{% tab title="4-A" %}}
```text { lineNos="true" }
​
```
{{% /tab %}}

{{% tab title="4-B" %}}
```text { lineNos="true" }
​
```
{{% /tab %}}
{{< /tabs >}}

[실험4-A]에서 오류가 발생하지 않기 때문에 상위 스코프인 줄 #5도 유효하고 다른 형제 블록인 줄 #6도 유효하다.  
[실험4-B]에서도 오류가 발생하지 않기 때문에 형제 블록인 case 블록에서도 유효하다.

줄 #5만 아니였다면 AST로 변환하면서 어쩌고... 억지로 이해해보려 할 수 있었겠지만 상위 스코프조차 유효하면 이야기가 달라진다. [실험2-C]에서 이미 상위 스코프인 조건절에는 호이스팅이 되지 않는 것을 확인했는데 이번에도 상위 스코프인 조건절임에도 유효하다.  
차이점이라면 줄이 위에 있는가, 아래에 있는가.

[실험2-C]와 거의 동일한 실험이므로 어느 하나를 더 우선하기는 어렵다. 하지만 다른 [실험2]들을 고려하면 선언문이 이동하지는 않았을 확률이 더 높다. 그렇다면 자연스럽게 `(같은 함수 내에서) 선언문보다 아래에 위치하기만 하면 유효하다`는 결론이 도출된다.

여기에서 공식자료를 참고해보면 딱히 더 자세히 명시하지 않는다.  
[https://learn.microsoft.com/en-us/previous-versions/t7zd6etz(v=vs.85)#scope-and-lifetime-of-variables](https://learn.microsoft.com/en-us/previous-versions/t7zd6etz(v=vs.85)#scope-and-lifetime-of-variables)

> A variable's scope is determined by where you declare it. When you declare a variable within a procedure, only code within that procedure can access or change the value of that variable. It has local scope and is a procedure-level variable. If you declare a variable outside a procedure, you make it recognizable to all the procedures in your script. This is a script-level variable, and it has script-level scope.

변수의 유효범위에 대한 정보는 이 문단이 끝이다. 맥락상 전역변수와 지역변수를 구분할 뿐, 구체적인 유효범위를 명시한 것으로는 보이지 않는다.
