+++
date = '2024-12-04T21:13:41Z'
linktitle = '디버깅 한셀 (3)'
title = '디버깅 한셀 (3) - 데이터의 수명'
weight = 100
+++

## 소유권의 개념

데이터의 수명을 논할 때는 해당 데이터가 어디에 저장되고 어떻게 제거되는지를 따져봐야 한다. 스택에 저장되는지, 힙에 저장되는지에 따라 메모리가 해제되는 방식도 다르기 때문이다. 특히 힙의 경우, 한 번의 할당은 정확히 한 번의 해제와 짝을 지어야 하기 때문에 이 clean-up을 담당할 주체 - `소유자` -를 잘 살펴봐야 한다.

{{% notice style="info" title="소유권이란?" %}}
C/C++ 같이 직접 메모리를 관리하는 언어나 러스트같이 언어 차원에서 소유권 개념을 도입한 언어로 개발해 본 분들이라면 익숙한 개념이겠지만 그렇지 않은 분들을 위해 간단히 설명하자면 소유자는 소유하는 자원에 대한 관리 책임이 있다.

```text
소유자는 소유하는 자원에 대해...
1. 다른 참조자에게 소유권을 양도하거나         (= move)
2. 제거를 위한 작업을 수행하고 값을 폐기하거나  (= destroy)
3. 다른 참조자에게 참조할 수 있게 해준다       (= shallow copy / borrow)
```

동일한 자원에 대한 소유권은 오직 하나의 참조자만이 가질 수 있다. 그리고 참조값만을 가진 참조자가 소유자보다 오래 존재할 경우, 폐기된 값에 접근하게 되는 오류가 발생한다.
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

[소유권 이전] 실험은 `Foo` 함수의 지역변수 `x`에서 인자로 받은 `arr`의 원소로 소유권이 넘어가는지를 확인한다. 만약 값이 항상 남아있다면 소유권이 정상적으로 넘어갔거나 복제되었을 것이고 값이 확률적으로 남아있거나 사라졌다면 소유권은 변경되지 않았다고 볼 수 있다.

[소유권 불변] 실험은 데이터가 상위 스코프로 전달이 될 수 있다는 대조군 실험이다. 여기에서 데이터가 상위 스코프로 전달되지 않는다면 그건 자원 관리가 소유권 모델만으로는 설명할 수 없다는 것을 암시한다.

[값 복제] 실험은 소유권이 이전되지 않을 경우를 대비한 해결책이 의도대로 작동하는지를 확인한다.

이제 위 3개의 실험을 각 자료형별로 `Foo` 함수를 수정하고 결과를 해석하면 된다.


## 정수

{{< tabs groupid="exp" title="정수 실험" >}}
{{% tab title="소유권 이전" %}}
```vb { lineNos="true" lineNoStart="9" }
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
```vb { lineNos="true" lineNoStart="9" }
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
```vb { lineNos="true" lineNoStart="9" }
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

우선 [소유권 불변] 실험을 확인해보면 값이 정상적으로 출력되므로 소유권 모델을 채택해도 됨을 알 수 있습니다.

[소유권 이전] 실험을 보면 확실히 소유권은 넘어오지 않은 것을 알 수 있습니다. 줄 #11에서 정의한 값은 `Foo` 함수가 종료됨과 동시에 제거된 것입니다. 그리고 종종 오류가 나는 것으로 보아 메모리마저 해제된 것으로 보입니다.

러스트조차 간단한 자료형에는 `Copy trait`를 붙여줬는데 이건 좀…

---

가장 간단한 정수도 `복제 안 됨 + 소유권 이전 안 됨`이므로 나머지 간단한 자료형(실수/날짜/시간/화폐)들은 건너뛰고 word 단위 이상의 자료형들을 살펴보겠다

---

## 문자열

{{< tabs groupid="exp" title="문자열 실험" >}}
{{% tab title="소유권 이전" %}}
```vb { lineNos="true" lineNoStart="9" }
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
```vb { lineNos="true" lineNoStart="9" }
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
```vb { lineNos="true" lineNoStart="9" }
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

정수와 동일한 결과다. 소유권 이전은 안 되며 복제도 직접 해야만 `Foo` 함수의 생명주기를 초월할 수 있다.

## 배열 실험

배열은 리터럴이 없고 복제하는 함수 / 연산자가 없어 [소유권 이전] 실험만 가능하다. 전반적인 구조는 위의 실험과 비슷하지만 배열을 다루기 위해 약간의 수정이 필요하다.

```vb { lineNos="true" }
Option Explicit
Sub Test()
    Dim arr(0)
    Call Foo(arr)
    MsgBox IsEmpty( arr(0) )
    MsgBox IsArray( arr(0) )
    MsgBox Join(arr(0), ",")    '배열을 출력할 수 있게 변경
End Sub

Sub Foo(arr)
    Dim x(0)
    x(0) = 1
    arr(0) = x
End Sub
```

기존 실험에서 줄 #6이 추가되고 배열을 출력하기 위해 줄 #7에 Join 함수를 쓰도록 변경됐다.

### 배열 변수

{{% tab title="배열 변수 실험" %}}
```text
실험 코드는 바로 위 참고
```

```text { title="결과" lineNos="true" }
False
True
런타임 오류 '5': 프로시저 호출 또는 인수가 잘못되었습니다.
```
{{% /tab %}}

비록 줄 #5~#6의 출력 결과는 `Foo` 함수의 `x` 배열이 `arr`의 원소로 남아있는 것처럼 보이더라도 같은 스코프에서 정의된 값에 접근할 수 없으므로 이는 use after free 로 인한 결과라고 볼 수 있다. `x` 배열의 원소만 해제됐는지 배열 전체가 해제됐는지의 확인은 `Test` 함수에 `UBound` 값을 출력하도록 추가하면 된다. 쓰레기값이 나오거나 오류가 발생하면 배열 전체가 해제된 것임을 알 수 있다. 이는 이전 편 [Use after free 실험을 위한 primitive 실험 #3](../debug2/#use-after-free-실험을-위한-primitive)에서 다뤘으므로 참고하기 바란다.

### 배열 생성자

{{< tabs groupid="exp" title="배열 생성자 실험" >}}
{{% tab title="소유권 이전" %}}
```vb { lineNos="true" lineNoStart="10" }
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
```vb { lineNos="true" lineNoStart="10" }
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

배열 생성자는 복제가 안 되므로 마지막 실험은 할 수 없었다.

[소유권 이전] 실험에서는 일반 변수와 결과가 동일하므로 추가적인 설명은 생략한다.

[소유권 불변] 실험에서는 생성자가 반환한 값이 상위 스코프에서 받은 `arr`에게 소유권이 주어진 것을 볼 수 있다. 배열의 생성자는 리터럴이 아니라 함수이므로 여기에서 어쩌면 함수의 반환값은 리터럴처럼 소유자가 지정되지 않은 것이라 예측해 볼 수 있다.

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
```vb { lineNos="true" lineNoStart="9" }
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
```vb { lineNos="true" lineNoStart="9" }
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
```vb { lineNos="true" lineNoStart="10" }
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

정수, 문자열, 배열 모두 `clone` 함수를 거치면 복제가 되므로 함수의 반환값은 소유권이 지정되지 않은 상태임을 알 수 있다.

흥미로운 점은 함수의 반환값을 직접 `clone = <복제된 값>` 으로 하지 않고 지역변수에 정의하고 그 변수를 반환값으로 지정해도 된다는 점이다.

## 결론

1. 한셀 VBS에서 참조자는 한 번 획득한 소유권을 양도하지 않는다
2. 리터럴과 함수의 반환값은 소유권이 없는 초기 상태이다
