+++
date = '2024-12-04T21:13:41Z'
linktitle = '디버깅 한셀 (3)'
title = '디버깅 한셀 (3) - 데이터의 수명'
weight = 100
+++

## 소유권의 개념

데이터의 수명을 논할 때는 해당 데이터가 어디에 저장되고 어떻게 제거되는지를 따져봐야 합니다. 스택에 저장되는지, 힙에 저장되는지에 따라 메모리가 해제되는 방식도 다르기 때문입니다. 특히 힙의 경우, 한 번의 할당은 정확히 한 번의 해제와 짝을 지어야 하기 때문에 이 clean-up을 담당할 주체 - `소유자` -를 잘 살펴봐야 합니다.

{{% notice style="info" title="소유권이란?" %}}
C/C++ 같이 직접 메모리를 관리하는 언어나 러스트같이 언어 차원에서 소유권 개념을 도입한 언어로 개발해보신 분들이라면 익숙한 개념이겠지만 그렇지 않은 분들을 위해 간단히 요약하자면...

```text
데이터(=값)의 소유권을 가지는 참조자는
1. 다른 참조자에게 소유권을 양도하거나         (= move)
2. 해제를 위한 작업을 수행하고 값을 폐기하거나  (= destroy)
3. 다른 참조자에게 참조할 수 있게 해준다       (= shallow copy / borrow)
```

위 조건들이 잘 지켜진다는 가정 하에 동일 데이터에 대핸 소유권은 오직 하나의 참조자만이 가질 수 있다.  
또한, 소유권이 없는 참조자가 폐기된 값을 참조하는 것은 불법이다.
{{% /notice %}}

앞서 [2편](../debug2/)에서 VB 객체는 `fat pointer` 형식으로 구현되어있을 것이라 추측했으므로 (가상) 스택에 저장되는 개체라도 실질적인 데이터는 힙에 저장될 가능성을 고려해야 한다. 그 외에도 JS처럼 정수형만을 특별히 포인터가 담길 공간에 저장하여 힙 할당을 생략하는 최적화 기법도 적용되었는지 확인해 볼 필요가 있다.

## 실험의 구조

{{< tabs groupid="exp" title="실험 구조" >}}
{{% tab title="소유권 이전" %}}
```vb { lineNos="true" }
Option Explicit
Sub Test()
    Dim arr(0)
    Call Foo(arr)
    MsgBox IsEmpty( arr(0) )
    MsgBox arr(0)
End Sub

Sub Foo(arr)
    Dim x
    x = '실험대상
    arr(0) = x
End Sub
```
{{% /tab %}}

{{% tab title="소유권 불변" %}}
```vb { lineNos="true" }
Option Explicit
Sub Test()
    Dim arr(0)
    Call Foo(arr)
    MsgBox IsEmpty( arr(0) )
    MsgBox arr(0)
End Sub

Sub Foo(arr)
    arr(0) = '실험대상
End Sub
```
{{% /tab %}}

{{% tab title="값 복제" %}}
```vb { lineNos="true" }
Option Explicit
Sub Test()
    Dim arr(0)
    Call Foo(arr)
    MsgBox IsEmpty( arr(0) )
    MsgBox arr(0)
End Sub

Sub Foo(arr)
    Dim x
    x = '실험대상
    arr(0) = x + 0 'identity transformation
End Sub
```
{{% /tab %}}
{{< /tabs >}}


## 정수

{{< tabs groupid="exp" title="정수 실험" >}}
{{% tab title="소유권 이전" %}}
```vb { lineNos="true" }
Sub Foo(arr)
    Dim x
    x = 1
    arr(0) = x
End Sub
```

```text { title="결과" lineNos="true" }
False
​           '또는 런타임 오류 '13': 형식이 일치하지 않습니다
```
{{% /tab %}}

{{% tab title="소유권 불변" %}}
```vb { lineNos="true" }
Sub Foo(arr)
    arr(0) = 1
End Sub
```

```text { title="결과" lineNos="true" }
False
1
```
{{% /tab %}}

{{% tab title="값 복제" %}}
```vb { lineNos="true" }
Sub Foo(arr)
    Dim x
    x = 1
    arr(0) = x + 0
End Sub
```

```text { title="결과" lineNos="true" }
False
1
```
{{% /tab %}}
{{< /tabs >}}

[소유권 이전] 실험에서 `Foo` 함수 내에서 인자로 받은 `arr` 배열에 1을 저장한다면 이 값은 `Foo` 함수가 끝날 때가 아닌 `arr` 배열이 유효할 때까지 값이 존재해야 하는 것이 일반적인 프로그래밍 언어의 상식입니다. 그렇지만 실제로는 `Foo` 함수 지역변수인 `x`와 같은 생명주기를 가지고 `Foo` 함수가 종료된 이후에 `arr` 배열의 정보를 확인해 보면 원소는 존재하지만 실질적인 데이터는 사라진 상태가 됩니다.

나머지 두 실험에서는 값이 정상적으로 출력되는 것으로 보아 리터럴이 종료된 함수 `Foo`에서 선언되었어도 문제가 되지는 않습니다. 단지, 최초로 바인딩된 참조자에 소유권이 주어지며 다른 참조자에 값을 복사하려고 해도 자동 복제가 되지도, 그렇다고 소유권이 이전되지도 않습니다. 값이 현재의 함수를 초월하게 만드려면 직접적으로 복제값을 생성해야 합니다.

러스트조차 간단한 정수형은 `Copy trait`를 구현했는데 이건 좀...

---

가장 간단한 정수도 `복제 안 됨 + 소유권 이전 안 됨`이므로 나머지 간단한 자료형(실수/날짜/시간/화폐)들은 건너뛰고 word 단위 이상의 자료형들을 살펴보겠습니다

---

## 문자열

{{< tabs groupid="exp" title="문자열 실험" >}}
{{% tab title="소유권 이전" %}}
```vb { lineNos="true" }
Sub Foo(arr)
    Dim x
    x = "Hi"
    arr(0) = x
End Sub
```

```text { title="결과" lineNos="true" }
False
​           '또는 런타임 오류 '13': 형식이 일치하지 않습니다
```
{{% /tab %}}

{{% tab title="소유권 불변" %}}
```vb { lineNos="true" }
Sub Foo(arr)
    arr(0) = "Hi"
End Sub
```

```text { title="결과" lineNos="true" }
False
Hi
```
{{% /tab %}}

{{% tab title="값 복제" %}}
```vb { lineNos="true" }
Sub Foo(arr)
    Dim x
    x = "Hi"
    arr(0) = x & ""
End Sub
```

```text { title="결과" lineNos="true" }
False
Hi
```
{{% /tab %}}
{{< /tabs >}}

정수와 동일한 결과입니다. 소유권 이전은 안 되며 복제도 직접 해야만 `Foo` 함수의 생명주기를 초월할 수 있습니다.

## 배열 (변수)

```vb
Option Explicit
Sub Test()
    Dim arr(0)
    Call Foo(arr)
    MsgBox IsEmpty( arr(0) )
    MsgBox IsArray( arr(0) )
    MsgBox Join(arr(0), ",")    '배열을 출력할 수 있게 변경
End Sub
```

{{< tabs groupid="exp" title="배열 변수 실험" >}}
{{% tab title="소유권 이전" %}}
```vb { lineNos="true" }
Sub Foo(arr)
    Dim x(0)
    x(0) = 1
    arr(0) = x
End Sub
```

```text { title="결과" lineNos="true" }
False
True
런타임 오류 '5': 프로시저 호출 또는 인수가 잘못되었습니다.
```
{{% /tab %}}
{{< /tabs >}}

배열 변수는 리터럴이 없고 통으로 복제하는 방법이 없어 첫 실험만 가능하다. 그마저도 역시 `Foo` 함수가 종료되면서 배열이 폐기되어 `Join` 함수가 오류를 일으켰다. 배열 변수도 소유권이 이전되지 않았다.

## 배열 (생성자)

`Test` 함수는 위의 [#배열 (변수)](#배열-변수)를 재사용하면 된다.

{{< tabs groupid="exp" title="배열 생성자 실험" >}}
{{% tab title="소유권 이전" %}}
```vb { lineNos="true" }
Sub Foo(arr)
    Dim x
    x = Array("Hi")
    arr(0) = x
End Sub
```

```text { title="결과" lineNos="true" }
False
False
런타임 오류 '13': 형식이 일치하지 않습니다
```
{{% /tab %}}

{{% tab title="소유권 불변" %}}
```vb { lineNos="true" }
Sub Foo(arr)
    arr(0) = Array("Hi")
End Sub
```

```text { title="결과" lineNos="true" }
False
True
Hi
```
{{% /tab %}}
{{< /tabs >}}

배열 생성자는 복제가 안 되므로 마지막 실험은 할 수 없었다. 결론은 동일하게 소유권 이전이 안 되어 값이 사라지고 `fat pointer`의 남은 부분 (타입 정보 등)만 남은 상태입니다.

여기에서 한 가지 주목할 점은 생성자 또한 값의 생명주기를 늘릴 수 있다는 것이다. 어쩌면 함수의 반환값은 리터럴처럼 소유권이 주어지지 않은 단독적인 값일지도 모른다. 만약 그렇다면 자료구조 자체에서 복제 기능을 제공하지 않을 때 복제하는 함수를 자작해서 대처할 수 있게 된다.

## 함수의 반환값의 소유권

```vb { title="실험 구조" lineNos="true" }
Option Explicit
Sub Test()
    Dim arr(0)
    Call Foo(arr)
    MsgBox IsEmpty( arr(0) )
    MsgBox arr(0)
End Sub

Sub Foo(arr)
    Dim x
    x = '실험대상
    arr(0) = clone(x)
End Sub

Function clone(val)
    Dim cpy
    'val을 cpy로 복제하는 코드
    clone = cpy
End Function
```

{{< tabs title="소유권 이전 실험" >}}
{{% tab title="정수" %}}
```vb { lineNos="true" }
Sub Foo(arr)
    Dim x
    x = 1
    arr(0) = clone(x)
End Sub

Function clone(val)
    Dim cpy
    cpy = val + 0
    clone = cpy
End Function
```

```text { title="결과" lineNos="true" }
False
0
```
{{% /tab %}}

{{% tab title="문자열" %}}
```vb { lineNos="true" }
Sub Foo(arr)
    Dim x
    x = "Hi"
    arr(0) = clone(x)
End Sub

Function clone(val)
    Dim cpy
    cpy = val & ""
    clone = cpy
End Function
```

```text { title="결과" lineNos="true" }
False
Hi
```
{{% /tab %}}

{{% tab title="배열" %}}
```vb { lineNos="true" }
Sub Foo(arr)
    Dim x
    x = Array("Hi")
    arr(0) = clone(x)
End Sub

Function clone(val)
    Dim ub, i
    ub = UBound(val)
    ReDim cpy(ub)
    For i = 0 To ub
        cpy(i) = val(i)
    Next
    clone = cpy
End Function
```

```text { title="결과" lineNos="true" }
False
Hi
```
{{% /tab %}}
{{< /tabs >}}

정수, 문자열, 배열 모두 `clone` 함수를 거치면 복제가 되어 `Foo` 함수가 종료된 이후에도 값이 살아남은 것을 확인할 수 있다.

## 결론

1. 한셀 VBS의 참조자는 한 번 획득한 소유권을 양도하지 않는다
2. 리터럴과 함수의 반환값은 소유권이 없는 것과 마찬가지다
3. 값의 생명주기는 그 값을 소유하는 참조자의 생명주기와 동시에 끝난다
