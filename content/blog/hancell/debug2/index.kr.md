+++
date = '2024-11-28T18:37:30Z'
linktitle = '디버깅 한셀 (2)'
title = '디버깅 한셀 (2) - 지역변수의 수명'
weight = 100
+++

이번 편은 한셀 버전에 따라 결과가 다를 수 있다. 군대에서 먼저 실험을 하고 이 글을 작성하면서 한 번 더 실행해봤더니 몇몇 실험들의 결과가 달랐다.

## 선언문 종류

한셀에서 지역변수를 선언할 때는 3가지 선언문과 3가지 변수 종류가 있다.  
선언문은 `Dim / ReDim / ReDim Preserve`가 있다.  
변수 종류는 `배열 변수 / 객체 변수 / 배열도 객체도 아닌 변수`가 있다. 편의상 `배열도 객체도 아닌 변수`는 `일반 변수`라고 하겠다.

예전에 한창 한셀을 뜯어볼 때 객체 변수의 존재를 잊어버려서... 이 글에서는 생략하겠다. 선언문에서는 일반 변수와 차이가 없고 정의문에서만 `Set <변수명> = <객체>` 형태를 가질 뿐이라 일반 변수랑 같을 거라 예상은 하지만 궁굼한 사람들은 여기 있는 실험들을 응용해서 한번 직접 검증해 볼 것을 추천한다.

## Dim 일반 변수

```vb { title="실험 1" lineNos="true" }
Option Explicit
Sub Test()
    Call Foo()
    Call Foo()
End Sub

Sub Foo()
    Dim a
    MsgBox a
    a = "Hi"
End Sub
```

```text { title="실험 결과 1" lineNos="true" }
​
​
```

[실험1]의 출력 결과가 두 번 다 빈 문자열인 것에 대한 설명은 2가지가 가능하다.

1. `Dim` 선언문은 선언과 동시에 기본값으로 초기화한다
2. `Dim`으로 선언된 변수는 함수가 종료됨과 동시에 소멸하여 초기화된다

각 가설 단독으로도 설명이 가능하고 두 가설이 동시에 참일 수도 있다. 단, 가설 #2 단독일 경우에는 매크로 실행 시 메모리는 쓰레기값이 없도록 초기화되어야 한다는 조건이 붙는다.

한셀이 클래스를 지원했다면 destructor가 존재하기 때문에 정확한 소멸 시점이 중요하기도 하고, 검증하기도 조금 더 쉬웠을 것이다. 하지만 지금의 한셀에서는 어느 가설(들)을 채택하든 실질적으로는 차이가 없다. 물론 검증은 시도할 것이다. 다만, 일반 변수에 대해서는 검증하기 어려워서 뒤에서 배열 변수로 검증하겠다.

## Dim 배열 변수

{{< tabs groupid="exp2" title="실험 코드" >}}
{{% tab title="2-A" %}}
```vb { lineNos="true" }
Option Explicit
Sub Test()
    Call Foo()
    Call Foo()
End Sub

Sub Foo()
    Dim arr(0)
    MsgBox arr(0)
    arr(0) = "Hi"
End Sub
```
{{% /tab %}}

{{% tab title="2-B" %}}
```vb { lineNos="true" }
Option Explicit
Sub Test()
    Call Foo()
    Call Foo()
End Sub

Sub Foo()
    Dim arr
    If not IsArray(arr) Then
        arr = Array("Hi")
        MsgBox "Assigned array"
    End If
    MsgBox arr(0)
End Sub
```
{{% /tab %}}
{{< /tabs >}}

{{< tabs groupid="exp2" title="실험 결과" >}}
{{% tab title="2-A" %}}
```text { lineNos="true" }
​
​
```
{{% /tab %}}

{{% tab title="2-B" %}}
```
Assigned array
Hi
Assigned array
Hi
```
{{% /tab %}}
{{< /tabs >}}

[실험2-A]는 [실험1]과 동일하게 해석하면 된다. 단, 이번 실험의 대상은 `arr` 배열 그 자체이며 `arr(0)`에 담긴 문자열은 그저 배열이 초기화되는지를 확인하기 위한 수단일 뿐이다. 배열 변수에 대한 해석을 배열의 원소에까지 확대하기에는 근거가 부족하다.

[실험2-B]는 일반 변수처럼 선언된 변수에 배열을 저장하는 경우를 다룬다.  
줄 #13을 제외하면 [실험2-A]와 동일하게 해석해도 된다. 선언문과 정의문 사이에 이전 값을 확인하고 이후 값을 정의하는 구조가 동일하기 때문이다. 대신 이번에는 초기화될지도 모를 값을 출력하는 것이 아니라 if 문으로 탐지 후 줄 #11이 출력되면 초기화된 것으로 간주한다. 줄 #13은 일반 변수처럼 선언한 변수에 배열을 저장해도 배열 변수처럼 동일하게 취급할 수 있음을 보여준다.

### Dim 배열 변수의 초기화 시점

변수의 초기화 시점을 검증하기 위해서는 시작하기 직전인 함수의 지역변수가 할당된 메모리 또는 종료된 함수의 지역변수가 할당될 메모리에 접근할 필요가 있다. 전자를 위해서는 double free 취약점을 찾아 함수 호출 전에 미리 할당을 받고 함수가 호출될 때 같은 공간이 재할당되어야 한다. 후자를 위해서는 use after free 취약점을 찾아 함수가 종료된 메모리 공간을 다시 할당받아 남아있는 흔적을 부검해야 된다. Double free 취약점은 찾지 못했지만 use after free 취약점은 찾았기 때문에 이걸 이용한 실험의 구조를 먼저 설명하겠다. 간단한 설명을 위해 C++ 문법을 사용했다.

```c++ { title="C++ 실험 설계도" lineNos="true" }
void main() {
    VBObject *obj = null;
    foo(&obj);
    bar(&obj);
}

void foo(VBObject **ptr) {
    VBObject *a = new VBObject();   // 지역변수가 할당됨
    a->valueX = 123;    // 임의의 값
    a->valueY = "Hi";   // 임의의 값
    *ptr = a;           // 지역변수를 가리키는 포인터를 상위 스코프에 저장
    delete a;           // 지역변수가 소멸됨
}

void bar(VBObject **ptr) {
    VBObject *x = new VBObject();   // 지역변수가 할당됨
    x->valueX = 456     // foo 함수와는 다른 임의의 값
    if(x->valueX == (*ptr)->valueX) {
        std::cout << "Use after free 성공\n";
        std::cout << x->valueY << std::endl;    // foo 함수에서 정의한 임의의 값이 남아있으면 초기화가 안 됨을 확인
    } else {
        std::cout << "Use after free 실패" << std::endl;
    }
    delete x;   // 지역변수가 소멸됨
}
```

각 함수의 인자로 전달되는 `ptr`는 `foo` 함수의 메모리를 가리키는 dangling pointer를 제공한다. 줄 #9, #17~#18은 `foo` 함수와 `bar` 함수에서 할당된 메모리 공간이 동일한지 확인한다. `bar` 함수에서 새로 할당받은 메모리 영역에 줄 #17에서 쓴 값이 `ptr`를 통해 접근한 `foo` 함수의 메모리에도 동일하게 반영된다면 같은 공간임을 알 수 있다.

같은 메모리 공간임을 확인한 다음에는 남아있는 데이터를 부검해야 한다. 이를 위해 줄 #10에서 임의의 데이터를 쓰고 줄 #20에서 그 값이 남아있는지 확인한다.

#### Use after free 실험을 위한 primitive

이 실험을 하기 위해 필요한 primitive는 다음과 같다.

- 메모리 공간을 가리키는 pointer
- 메모리 공간을 초기화하거나 값을 덮어쓰지 않고 생으로 할당하는 방법
- 메모리 공간 일부에 해제된 이후에도 읽을 수 있는 값을 지정하는 방법 (partial write)
- 해제된 메모리에서도 값을 읽는 방법

배열은 포인터가 없는 언어에서 참조값으로 전달되는 대표적인 자료형이다. 일부 현대 언어의 경우, clone / copy-on-write로 처리할 수도 있지만 VB는 오래된 언어인 만큼 고려하지 않아도 될 것이다.

포인터 대신 배열을 사용할 경우, 필요한 나머지 primitive를 찾기 위해 아래와 같은 실험을 할 수 있다.

```vb { title="실험 3" lineNos="true" }
Option Explicit
Sub Test()
    Dim ptr(0)
    Call Foo(ptr)
    
    MsgBox IsEmpty( ptr(0) )
    MsgBox IsArray( ptr(0) )
    MsgBox TypeName( ptr(0) )
    MsgBox UBound( ptr(0) )
    MsgBox ptr(0)(0)
End Sub

Sub Foo(ptr)
    Dim arr(0)
    arr(0) = "Hi"
    ptr(0) = arr
End Sub
```

[실험3]을 반복해서 실행하면 아래의 두 결과 중 하나가 랜덤으로 나온다.

{{< tabs title="실험3 결과" >}}
{{% tab title="Case 1" %}}
```text { lineNos="true" }
False
True
Variant()
418977394       -> 랜덤 쓰레기값
런타임 오류 '9': 아래 첨자 사용이 잘못되었습니다. / 런타임 오류 '5': 프로시저 호출 또는 인수가 잘못되었습니다.
```
{{% /tab %}}

{{% tab title="Case 2" %}}
```text { lineNos="true" }
False
True
Variant()
런타임 오류 '9': 아래 첨자 사용이 잘못되었습니다.
```
{{% /tab %}}
{{< /tabs >}}

{{% notice style="info" title="실험3 설계 해설" %}}
`Foo` 함수의 지역변수 `arr`를 `Test` 함수의 `ptr` 배열에 저장하면 `ptr`를 종료된 함수 `Foo`의 메모리 공간을 가리키는 dangling pointer로 활용할 수 있다. 줄 #6~#10은 여러가지 방법으로 이 dangling pointer를 통해 어떤 값을 읽고 출력한다.

이때 `arr`를 배열 변수로 선언한 이유는 OOP에서의 객체 구현 방식과 일반적인 인터프리터가 자료형을 표현하는 방법을 고려했기 때문이다.

OOP에서 객체를 구현하는 구조체는 이런 형태를 가진다


```c { title="객체 구조체" }
struct Object {
    void **vtable;  // OOP 가상 매소드를 구현하기 위한 사실상의 유일한 방법
    void superclass_properties;
    void subclass_properties;
    ...
};
```

`vtable`은 함수 포인터의 배열로, 클래스 당 1번만 생성되는 static (전역)변수다. 해제될 일이 없으니 이후 메모리 취약점을 이용할 때는 고려하지 않아도 된다.
상속이 가능한 클래스는 상위 클래스의 속성을 앞에 두어 upcast되어도 offset이 일정하게 유지된다.

인터프리터에서 자료형을 표현하는 방식은 CPython을 참고했다. 구현할 언어의 자료형은 하나의 공통 클래스로부터 상속되며 이때 자료형의 구조체에는 인터프리터 구현을 위한 내부 정보와 자료형의 타입 정보, 자료형이 담는 실질적인 값 등이 저장된다. 한셀의 VBS 인터프리터도 이와 같은 방식으로 구현되었다면 대략적으로 이런 형태를 가질 것이다.


```c { title="자료형 구조체" }
struct VbObject {
    void **vtable
    …    // 구현을 위한 기타 내부 정보
    void type_info    // 자료형 타입 정보
    void inst_prop    // 개체 속성들 - 문자열/배열 길이 등 추가 정보 필요시
    void inst_data    // 개체 값(들)
};

```

인터프리터에 따라 가성 머신처럼 별도의 콜스택을 만들고 `arena allocator`처럼 메모리를 직접 관리하며 사용할 수도 있다. 이렇게 되면 `VbObject` 객체는 free되지 않기 때문에 이 구조체의 정적인 속성들은 보편적인 메모리 취약점 공격에 쓰기 어렵다. 그렇기 때문에 같은 타입이라도 크기가 달라질 수 있어 힙 메모리를 추가로 할당해야만 하는 배열을 사용한 것이다.

이 조건만 보면 문자열도 가능할 것 같지만, 이후 use after free 취약점을 활용할 때 필요한 primitive를 전부 충족시키지 못하기 때문에 문자열은 적합하지 않다. 문자열은 선언문만으로 타입을 명시할 수 없어 문자열 관련 함수가 제대로 데이터를 읽을 수 있을지도 확실하지 않고, 일부의 문자만 수정할 수도 없다. 동일한 메모리 공간인지 확인하기 위해 값을 덮어쓴다면 메모리 공간 전체가 변경되어 이후에 부검할 수가 없다. 그렇다고 메모리 공간 확인을 생략한다면 결과에 확신을 가질 수 없다.
{{% /notice %}}

[실험3]의 결과를 해석해 보면...  
`IsEmpty`의 결과가 `False`이므로 `ptr(0)`에 `arr`의 잔재가 남아있는 것을 확인할 수 있다.  
`IsArray`와 `TypeName`으로 이 잔재에는 배열이었던 타입 정보가 남아있는 것을 확인할 수 있다.  
`UBound`는 쓰레기값을 반환하거나 오류를 일으킨다.  
`arr`을 역참조하여 원소에 접근할 수 없다.

값이 남아있는 경우와 변경되거나 사라지는 경우가 동시에 존재하므로 가상 스택과 힙이 있으며 객체의 일부만이 힙에 할당되고 나머지는 스택에 저장되는 것으로 보인다. 항상 일정한 값을 반환하는 함수는 스택에 저장되는 값을 읽어드리는 것이고 값이 변경되거나 오류를 일으키는 경우에는 힙에서 읽어드리는 것이라고 추측할 수 있다.

UBound가 쓰레기값을 반환하거나 오류를 일으키는 것으로 보아 배열의 길이는 배열 원소와 함께 fat pointer로 힙에 저장된다고 유추할 수 있다. 오류가 나는 이유는 아마도 allocator가 메모리를 해제하면서 남긴 연결 리스트 포인터가 UBound 함수가 해석할 수 있는 범위를 벗어나서 그런 것 같다.

이것으로 필요한 primitive가 전부 마련됐다.

| 필요한 primitive | 사용할 VBS 기능 |
| --- | --- |
| 메모리 공간을 가리키는 포인터 | 배열의 원소 |
| 메모리 공간을 초기화하지 않고 할당 | 배열 변수 선언 |
| 메모리 공간 일부에 값을 지정하는 방법 | 선언문에서 배열 변수의 길이 |
| 해제된 메모리에서 값을 읽는 방법 | `UBound` 함수 |

#### Use after free 기반 초기화 시점 검증

```vb { title="실험 4" lineNos="true" }
Option Explicit
Sub Test()
    Dim ptr(1)
    ptr(1) = "Hi"
    Call Foo(ptr)
    MsgBox UBound( ptr(0) )
    Call Bar(ptr)
End Sub

Sub Foo(ptr)
    Dim arr(123)
    arr(0) = ptr(1)
    ptr(0) = arr
End Sub

Sub Bar(ptr)
    Dim x(456)
    If UBound( ptr(0) ) = UBound(x) Then
        MsgBox x(0)
        x(0) = "Bye"
        MsgBox x(0)
        MsgBox ptr(0)(0)
    Else
        MsgBox "Failed to re-use memory"
    End If
End Sub
```

[실험4]를 반복해서 실행하면 아래의 두 결과 중 하나가 랜덤으로 나온다.

{{< tabs title="실험 결과" >}}
{{% tab title="4 성공" %}}
```text { lineNos="true" }
123

Bye
Bye
```
{{% /tab %}}

{{% tab title="4 실패" %}}
```text { lineNos="true" }
123
Failed to re-use memory
```
{{% /tab %}}
{{< /tabs >}}

{{% notice style="info" title="실험4 설계 해설" %}}
[C++ 실험 설계도]에 빗대어 보면...

- `ptr`는 `&obj`에 해당한다
- `ptr(0)`는 `obj`에 해당한다
- `arr` 배열의 길이는 `a->valueX`에 해당한다
- `arr(0)`는 `a->valueY`에 해당한다
- `ptr(1)`는 `arr(0)`에 저장할 임의의 값에 해당한다
    - 리터럴을 쓰지 않고 상위 스코프에서 값을 받아오는 이유는 실험 도중에 불필요한 메모리의 할당과 해제를 줄이기 위해서다
- 줄 #6은 디버깅을 위해 해제된 `arr` 변수의 길이 정보가 남아있는 것을 확인한다
- 줄 #20~#22는 정말 같은 메모리가 할당된 것을 강조하기 위해 `a->valueY`의 값을 또 다른 임의의 값으로 변경하고 양쪽의 포인터를 통해 역참조하는 엑스트라다

Foo 함수에서 리터럴을 쓰지 않고 상위 스코프에서 값을 받아오는 이유는 실험 도중에 불필요한 메모리의 할당과 해제를 줄이기 위함이다.

---

사실 memory allocator가 같은 공간을 즉시 재할당하기 위해서는 같은 크기의 메모리를 요청하는 것이 유리하다. 그 이유는 빠른 할당을 위해 대부분의 memory allocator가 크기에 따라 할당할 수 있는 메모리 청크를 스택으로 관리하기 때문이다. 메모리 할당 방법에는 `first fit / best fit` 등의 전략이 있다. 그러나 두 번째에 더 큰 공간을 요청하는 [실험4]는 어느 전략에도 잘 맞지 않는 것처럼 보인다.

아마도 충분히 큰 공간을 할당했기 때문에 파편화된 힙의 앞부분에서는 만족하는 청크가 없어 아직 건드리지 않은 힙 뒷부분에서 청크를 잘라내어 할당하고, 추가로 할당하지 않았기 때문에 해제된 뒤에는 힙 뒷부분과 바로 병합이 됐다가 다시 큰 공간을 요청하자 같은 위치에서 크기만 키운 청크로 다시 잘라내어 할당한 것으로 추측하고 있다.

솔직히 `large bin`를 재사용해야겠다는 생각으로 첫 배열의 크기만 크게 잡고 두 번째 배열은 적당히 큰 숫자를 넣고 되길래 넘어갔다가 거의 1년이 지나 이 글을 작성하면서 다시 보니 `이게 왜 되지..?` 고민하다 내린 추론이다...
{{% /notice %}}

[실험4]의 줄 #19에서 `arr`의 잔재가 사라진 것을 보면 Dim은 선언 시 초기화하는 것을 알 수 있다.  
덤으로 줄 #6에서 `arr`의 잔재가 남아있는 것을 보면 함수가 종료된 이후에는 초기화하지 않는 것을 알 수 있다.

#### Double free 시도

Double free를 이용한 실험도 시도해보려 했으나 double free 취약점을 찾지 못했다.

{{< tabs groupid="exp5" title="실험 코드" >}}
{{% tab title="5-A" %}}
```vb { lineNos="true" }
Option Explicit
Sub Test()
    Dim ptr(0)
    Call Foo(ptr)
End Sub

Sub Foo(ptr)
    Dim x(123), y
    y = x
    ptr(0) = y
    MsgBox UBound(y)
    MsgBox UBound( ptr(0) )
End Sub
```
{{% /tab %}}

{{% tab title="5-B" %}}
```vb { lineNos="true" }
Option Explicit
Sub Test()
    Dim ptr(0)
    Call Foo(ptr)
End Sub

Sub Foo(ptr)
    Dim x(123), y
    y = x
    ptr(0) = x
    MsgBox UBound(y)
    MsgBox UBound( ptr(0) )
End Sub
```
{{% /tab %}}
{{< /tabs >}}

{{< tabs groupid="exp5" title="실험 결과" >}}
{{% tab title="5-A" %}}
```text { lineNos="true" }
123
런타임 오류 '51': 내부 오류입니다
```
{{% /tab %}}

{{% tab title="5-B" %}}
```text { lineNos="true" }
​123
123
```
{{% /tab %}}
{{< /tabs >}}

[실험5-A]와 [실험5-B]의 차이는 줄 #9밖에 없다. 다른 변수로 alias를 만들면 raw-pointer인 경우에 double free가 가능할 수도 있고 smart-pointer면 정상적으로 동작할 거라 예상한 것과는 다르게 오류가 발생한다. 그것도 전혀 엉뚱한 곳에서... 이 실험 결과는 어떻게 해석해야 할지 전혀 모르겠다.

## ReDim 일반 변수

{{< tabs groupid="exp6" title="실험 코드" >}}
{{% tab title="6-A" %}}
```vb { lineNos="true" }
Option Explicit
Sub Test()
    ReDim a
End Sub
```
{{% /tab %}}

{{% tab title="6-B" %}}
```vb { lineNos="true" }
Option Explicit
Sub Test()
    ReDim Preserve a
End Sub
```
{{% /tab %}}
{{< /tabs >}}

{{< tabs groupid="exp6" title="실험 결과" >}}
{{% tab title="6-A" %}}
```text
컴파일 오류:

변수를 찾을 수 없습니다.
```
{{% /tab %}}

{{% tab title="6-B" %}}
```text
컴파일 오류:

변수를 찾을 수 없습니다.
```
{{% /tab %}}
{{< /tabs >}}

일반 변수의 선언에는 ReDim을 사용할 수 없다.

## ReDim 배열 변수

```vb { title="실험 7" lineNos="true" }
Option Explicit
Sub Test()
    Call Foo()
    Call Foo()
End Sub

Sub Foo()
    ReDim arr(0)
    MsgBox arr(0)
    arr(0) = "Hi"
End Sub
```

{{< tabs groupid="exp7" title="실험 결과 1회차" >}}
{{% tab title="7 ver군대" %}}
```text { lineNos="true" }
​
Hi
```
{{% /tab %}}

{{% tab title="7 ver민간" %}}
```text { lineNos="true" }
​
​
```
{{% /tab %}}
{{< /tabs >}}

{{< tabs groupid="exp7" title="실험 결과 2회차 이상" >}}
{{% tab title="7 ver군대" %}}
```text { lineNos="true" }
Hi
Hi
```
{{% /tab %}}

{{% tab title="7 ver민간" %}}
```text { lineNos="true" }
​
​
```
{{% /tab %}}
{{< /tabs >}}

{{< tabs groupid="exp7" title="실험 결과 해설" >}}
{{% tab title="7 ver군대" %}}
처음 `Foo`가 호출됐을 때 종료 직전에 정의한 값이 다음 `Foo`가 호출될 때 존재하는 것을 보아 함수가 종료되어도 ReDim으로 선언한 배열은 초기화는 커녕, 있는 그대로 재사용되고 있다. Dim이 선언 시 초기화하는 것과 반대일 뿐만 아니라 선언문인데도 새로운 객체가 생성되지 않았다.

ReDim으로 동적배열을 생성할 경우에는 선언+삭제+선언 패턴으로 완전 초기화를 해야지만 새로운 객체를 할당받을 수 있다. 선언을 2번 하는 이유는 선언문 없이 `Erase`문을 사용할 수 없기 때문이다.

여기에서 호기심에 실험을 재실행했더니 결과가 달랐다. ReDim 배열 변수는 단순히 함수가 종료되는 것을 초월할 뿐만 아니라 모든 매크로가 실행을 완료한 뒤에도 다음 매크로가 실행될 때까지 계속 살아남는다. 다시 1회차처럼 동작하게 초기화하는 방법은 매크로 코드가 변경되거나 한셀 프로그램을 재시작하면 된다.

스크립트 창에서 코드에 주석을 추가하거나 개행 문자를 추가해서 코드는 변경하지 않은 체 텍스트만 수정해도 한셀 매크로 엔진이 재시작한다. 흥미로운 점은 무언가를 바꾸었다가 다시 되돌린 다음에 스크립트를 재실행하면 엔진이 재시작하지 않는다. `Ctrl z` 되돌리기가 아니라 변경한 내용을 수동으로 되돌려도 그런 것으로 보아 변경되는 순간 dirty 체크를 하는 것이 아닌 변경 전과 후를 diff하는 것처럼 보인다.
{{% /tab %}}

{{% tab title="7 ver민간" %}}
`Dim`으로 선언한 변수와 동일하게 해석하면 된다.
{{% /tab %}}
{{< /tabs >}}

## ReDim Preserve 배열 변수

[실험7] 줄 #8을 `ReDim arr(0)` -> `ReDim Preserve arr(0)`로 변경하면 된다.

결과도 [실험7]과 모든 경우에 다 동일하기 때문에 추가 설명은 생략한다.

## 번외

`Dim`으로 선언한 변수를 `ReDim`으로 재선언할 수 없다. VBS 공식 문서에서는 동적배열을 예시로 들며 이게 가능한 것으로 설명하지만 적어도 한셀에서는 불가능하다.

`ReDim 배열 변수`는 매크로가 종료되어도 계속 살아남기 때문에 메모리 누수가 발생할 수 있을 거라 생각해 한계까지 배열을 생성해 봤다. 작업 관리자 기준 약 10MB를 늘리는데 성공했지만 그 이상은 늘지 않아 심각한 결함 같지는 않다.
