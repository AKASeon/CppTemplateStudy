# Specialization and Overloading

이 챕터에서는 template Specialization 와 function template 의 overloading 에 대하여 알아 볼것 입니다.

## When "Generic code" Doest't Quite

Generic Code 가 안 좋은 경우도 존재 합니다.

```c++
template<typename T>
class Array
{
private:
    T * data;
    ...
public:
    Array( Array<T> cont & );
    Array<T> & operator= ( Array<T> const & );

    T & operator[] ( std::size_t k )
    {
        return data[k];
    }
    ...
};
```

```c++
template<typename T>
inline void exchange( T * a, T * b )
{
    T tmp( * a );     // copy-constructor(복사 생성자) 호출
    * a= * b;         // copy-assignment(복사 대입연산자) 호출
    * b = tmp;        // copy-assignment(복사 대입연산자) 호출
}
```

Generic code 로 생성한 exchange 함수의 경우 Array 의 모든 데이를 이동 시키면 Array 개수 만큼 exchange 함수가 호출되어야 하고 exchange 함수는 copy constructor 가 1번 copy-assignment 2번 호출 되어야 합니다.
비용으로 보자면 (copy constructor 가 1번 + copy-assignment 2번) * (array 의 element 개수 ) 입니다.

```c++
...
template<typename T>
class Array
{
    ...
    void exchangeWith( Array<T> * b )
    {
        T * tmp = data;     
        data = b->data;     
        b->data = tmp;      
    }
};
```

위와 같은 함수를 이용하여 array 의 element 를 교환할수 있는 함수를 제공하면 generic code 로 생성한 exchange 함수를 이용하는것 보다 훨씬 효율적입니다.

### Transparent Customization

방금 전 예제에서 exchangeWidth 함수는 exchage 함수에 비해 효율적이지만 다른 함수를 사용해야 하는 점이 아래와 같은 점때문에 불편할수 있습니다.
1. Array class 사용자는 추가 interface 에 대하여 기억하고 있어야 하며 가능한 경우에만 사용할수 있도록 주의를 기울여야 합니다.
2. Generic 알고리즘은 일반적으로 다양한 가능성 사이에서 구분할수 없습니다.

```c++
template<typename T>
void genericAlgorithm( T * x, T * y )
{
...
    exchange( x, y );   // how do we select the right algorithm?
...
}
```

이러한 점때문에 C++ template 은 투명하게(transparently) function template 와 class template 를 사용자가 정의할수 있는 방법을 제공 합니다. function template 의 경우 overloading 기법을 통하여 수행할수 있습니다.

```c++
template<typename T>
void quickExchange( T * a, T * b )                  // #1
{
    T tmp( * a );
    * a = * b;
    * b = tmp;
}

template<typename T>
void quickExchange( Array<T> * a, Array<T> * b )    // #2
{
    a->exchangedWith( b );
}

void demo( Array<int> * p1, Array<int> * p2 )
{
    int x = 42, y = -7;
    quickExchange( &x, &y );        // uses #1
    quickExchange( p1, p2 );        // uses #2
}
```

### Semantic Transparency

이전 챕터에서 보여준 overloading 의 사용이 instantiation process 의 투명한 사용자 정의(tranparent customization)화 하는데 아주 많은 도움을 주었지만 구현의 세부사항에 크게 의존한다는 것을 아는것이 중요합니다.

```c++
struct S
{
    int x;
} s1, s2;

void distinguish( Array<int> a1, Array int a2 )
{
    int * p = &a1[0];
    int * q = &s1.x;

    a1[0] = s1.x = 1;
    a2[0] = s2.x = 2;

    quickExchange( &a1, &a2 );      // * p == 1 after this
                                    // 16.1.1 의 #2 함수를 호출
    quickExchange( &s1, &s2 );      // * q == 2 after this
                                    // 16.1.1 의 #1 함수를 호출
}
```
quickExchange 함수 호출이후에 pointer 를 swap 하는 것과 실제 데이터를 복사로 swap 하는것에 따라서 차이가 발생합니다. 이러한 차이는 template 구현체를 사용하는 사용자에게 혼란을 줄수 있습니다.

quck_ 접두사는 원하는 작업을 수행하기 위한 지름길이라는 사실에 관심을 가지는데 도움을 줄수 있습니다. 그러나 기본 generic exchange() template 은 Array<T> 에 유용한 최적화를 가질수 있습니다.

```c++
template<typname T>
void exchange( Array<T> * a, Array<T> * b )
{
    T * p = &(* a)[0];
    T * q = &(* b)[0];

    for ( std::size_t k = a->size(); k-- != 0; )
    {
        exchange( p++, q++ );
    }
}
```

generic code 를 이용한 이 버젼의 이점은 큰 임시 Array<T>가 필요로 하지 않습니다. exchange() template 은 재귀로 호출되기때문에 Array<Array<char>> 와 같은 타입 조차 좋은 성능을 보여줍니다. 또한 template 의 좀더 specialized 된 버젼의 경우 많은 양의 일을 저절로 하지만 inline 으로 선언되지 않습니다. 기본 generic 구현의 경우에는 몇가지 작업만 수행하기 때문에 inline 화 됩니다.

## Overloading Function Templates

전 세션에서 같은 이름을 사용하는 두가지 function template이 인스턴스화 되어 둘다 동일한 parameter 타입을 가질수 있더라도 공존할수 있음을 보았습니다.

```c++
template<typename T>
int f( T )
{
    return 1;
}

template<typename T>
int f( T * )
{
    return 2;
}
```

처음 template 에서 T 는 int * 로 치환될때 두번째 template 에서 T가 int 로 치환되어 얻는 것과 같은 parameter(그리고 return) 타입들을 포함하는 함수를 얻을수 있습니다. 이런 template 이 공존할수 있을 뿐만 아니라 동일한 parameter 와 return 타입을 가지고 있더라도 각각 공존할수 있습니다.

```c++
#include <iostream>
#include "funcoverload1.hpp"

int main()
{
    std::cout << f<int*>( (int*)nullptr );    // calls f<T>(T)
    std::cout << f<int>( (int*)nullptr );     // calls f<T>(T*)
}
```

f<int*>() 는 template argument 추론에 의존하지 않고 template f 함수의 처음 template parameter를 int* 로 치환하는것을 나타냅니다.
이 경우 하나 이상의 template f() 가 존재하므로 template 으로 부터 생성된 두개의 함수를 포함하는 overload set 이 만들어 집니다.: ( f<int*>(int*) (처음 template 으로 부터 생성된)와 f<int*>(int**)(두번째 template 으로 부터 생성된) )
(int*)nullptr 으로 호출하는 argument 는 타입 int * 이다. 이것은 처음 template 으로 부터 생성된 함수만 일치하므로 결국 그 함수가 호출됩니다.
반면에 두번째 호출은 f<int>(int) (처음 template 으로 생성된) 와 f<int>(int*) (두번째 template 으로 생성된)) 포함된 overloading 이 만들어 지고 두번째 template 만 일치 합니다.

### Signatures

구별되는 signature 를 가지고 있는 두개의 함수는 공존할수 있습니다. 아래와 같은 정보로 함수의 signature 를 정의할수 있습니다.

1. The unqualified name of the function ( 또는 생성된 function template 의 이름 )
2. 이름의 class 또는 namespace 범위를 선언
3. 함수의 const, volatile 또는 const volatile qualification ( 만약 그런 qualifier 를 포함한 멤버 function 이면 )
4. 함수의 & 또는 && qualification ( 만약 그런 qualifier 를 포함한 멤버 function 이면 )
5. function parameter 의 타입( function template 으로 부터 생성된 함수의 경우 template parameter 가 치환되기전 )
6. function template 으로 부터 생성된 함수면 그것의 return 타입
7. function template 으로 부터 생성된 함수면 template parameter 와 template arguments

원칙적으로 다음 template 과 그것의 인스턴스화는 같은 프로그램에서 공존할수 있습니다.

```c++
template<typename T1, typename T2>
void f1 ( T1, T2 );

template<typename T1, typename T2>
void f1 ( T2, T1 );

template<typename T>
long f2( T );

template<typename T>
char f2( T );
```

그러나 같은 범위에서 선언될때 모두다 인스턴스화하면 overload 가 모호하게 되기 때문에 항상 사용할수 없습니다. 예를 들면 위의 template 처럼 선언되어 있을때 f2(42) 를 호출하면 모호하게 생성될것 입니다.

```c++
#include <iostream>

template<typename T1, typename T2>
void f1( T1, T2 )
{
    std::cout << "f1( T1, T2 )\n";
}

template<typename T1, typename T2>
void f1( T2, T1 )
{
    std::cout << "f1( T2, T1 )\n";
}

// fine so far
int main()
{
    f1<char, char>( 'a', 'b' );     // Error: ambigous
}
```

`f1<T1 = char, T2 = char>(T1, T2)` 함수와 'f1<T1 = char, T2 = char>(T2, T1)' 함수는 같이 사용할수 있습니다.
but overloaded resolution will never prefer one over the other. template 이 다른 변환 유닛으로 보여 진다면 같은 프로그램안에 존재 할수 있습니다.( linker 는 인스턴스화의 signature가 구분되기 때문에 중복 정의에 대하여 오류를 출력하지 않습니다.)

```c++
// translation unit1:
#include <iostream>

template<typename T1, typename T2>
void f1( T1, T2 )
{
    std::cout << "f1( T1, T2 )\n";
}

void g()
{
    f1<char, char>( 'a', 'b' );
}

// translation unit 2:
#include <iostream>
template<typename T1, typename T2>
void f1( T2, T1 )
{
    std::cout << "f1( T2, T1 )\n" );
}

extern void g();        // defined in translation unit 1

int main()
{
    f1<char, char>( 'a', 'b'b );
    g();
}
```

위 코드는 문제가 없는 코드이고 그 결과는 아래와 같습니다.

```c++
f1( T2, T1 )
f1( T1, T2 )
```

### Partial Ordering of Overloading

저의 예제를 다시 보면 : 주어진 template argument lists(<int*> 와 <int>)으로 치환된후에 overload resolution 이 호출할 알맞는 함수를 하는것으로 끝났습니다.

```c++
std::cout << f<int*>((int*)nullptr);    // calls f<T>(T)
std::cout << f<int>((int*)nulltpr);     // calls f<T>(T*)
```

그러나 explicit template argument가 제공되지 경우에도 function 는 선택됩니다. 이 경우에 template argument 추론이 실행됩니다. 이 mechanism 를 이야기 하기 위하여 저의 예제에서 main() 함수를 조금 수정합니다.

```c++
#include <iostream>

template<typename T>
int f( T )
{
    return 1;
}

template<typename T>
int f( T* )
{
    return 2;
}

int main()
{
    std::cout << f( 0 );            // calls f<T>( T )
    std::cout << f( nullptr );      // calls f<T>( T )
    std::cout << f( (int*)nulltpr ) // calls f<T>( T* )
}
```

처음 호출된 f(0) 를 보면 : argument 의 타입은 int 입니다. 그래서 T 가 int 로 치환된다면 첫 template 이 일치 합니다. 그러나 두번째 template 의 parameter 타입은 항상 pointer 이므로 추로후에 첫 template 에서 생성된 인스턴스만 호출될수 있는 후보 입니다. 이 경우에 overload resolution 은 별거 없습니다.
두번째 호출인 f( nullptr )도 동일하게 접근하면: argument 타입은 std::nullptr_t 이고 이것 역시 첫 template 와 일치 합니다.
세번째 호출되는 f( (int*)nullptr )은 조금더 흥미롭습니다. argument 추론이 f<int*>(int*)와 f<int>(int*) 로 나타 냅니다. 전통적인 overload resolution 측면에서는 둘다 int* 를 argument 로 호출되는 좋은 함수이지만, 이런 호출은 모호 합니다.
그러나 이런경우에는 추가적인 overload resolution 기준이 적용됩니다. 좀더 specialized template 으로 부터 생성된 function 이 선택됩니다. 두번째 template이 조금더 specilized 하고 그런 이유로 이 예제에 대한 결과는 112 입니다.

### Formal Ordering Rules

마지막 예제에서 첫번째 template은 모둔 argument 타입에 대하여 수용하고 있지만 두번째 template 은 pointer 타입에 대해서만 허용하기 때문에 두번째 template 이 첫 번째 template 보다 더 specialize 하는 것이 매우 직관적으로 보일수 있습니다. 그러나 다른 예제들은 반드시 직관적인것은 아닙니다. 다음은 overload set 에 포함된 하나의 function template이 다른것들 보다 더 specialize 되어 있는지 확인하는 정확한 절차에 대하여 이야기 할것 입니다. 이것들은 partial oerdering rule 입니다.: It is possible that given two templates, neither can be considered more specialized than the other. overload resolution 이 2개의 template 사이에서 선택한다면, 결정이 되지 않습니다. 그리고 프로그램은 모호한 문제를 포함하고 있습니다. 주어진 function call 이 실행가능한 것 처럼 보이는 2개의 동일한 이름의 function template 를 비교한다고 가정합니다. overload resolution 은 아래처럼 결정 됩니다.
- default argument 와 줄임표 parameter로 이루어진 function call parameter 는 다음항목에서 무시 됩니다.
- 다음과 같이 모든 template parameter 치환된 두가지 argument 타입의 artificial 목록으로 합성 합니다.
1. 각 template 타입 parameter 는 고유한 invented type 으로 대체
2. 각 template의 template parameter 는 고유한 invented class template 으로 대체
3. 각 non-type tempate parameter 는 적합한 타입의 고유한 invented value 로 대체
( 이 context 에서 invented 된 type, template 나 values는 다른 context 에서 프로그래머가 사용하거나 complier 가 합성한 다른 어떤 type template, value 와 구분된다.)
- 첫 합성 argument type 의 리스트에 대하여 두번째 template 의 template argument 추론이 정확하게 일치하여 성공할 경우, 그 반대는 아니지만, 첫 template 은 두번째 보다 좀더 specialize 합니다. 반대로 두번째 합성 argument type 의 리스트에 대하여 첫 번째 template argument 추론이 정확하게 일치 한다면 처음것 보다 두번째 template 이 좀더 specialize 합니다. 그렇지 않으면 (추론이 성공하지 못하거나, 둘다 성공하면) 두 template 사이에는 ordering이 존재하지 않습니다. 마지막 예에서 두 template 에 적용하여 구체적으로 설정합니다. 이 두 template으로 부터 앞에서 설명한 template parameter를 대체하여 argument type 의 두 리스트를 합성합니다. (A1) 과 (A2*) ( where A1 and A2 are unique make up type ) 명백하게 두번째 argument type 의 리스트에 대한 첫번째 template 추론은 T가 A2* 로 치환되어 성공 합니다. 그러나 두번째 template 의 T* 을 첫 리스트 타입에서 non-pointer 타입 A1 를 일치시킬 방법이 없습니다. 따라서 처음것 보다 두번째 template 이 좀더 specialize 되어 있다라고 결론을 내릴수 있습니다.

여러 function parameter 를 포함한 좀더 복잡한 예제를 고려 하여야 합니다.

```c++
template<typename T>
void t( T*, T const * = nullptr, ... );

template<typename T>
void t( T const*, T*, T* = nullptr );

void example( int * p )
{
    t( p, p );
}
```

먼저 실제 호출에서 첫 template 에서 ellipsis parameter와 default argument로 선언되어 두번째 template 의 마지막 parameter는 사용되지 않기 때문에 이런 parameter들은 partial ordering 에서 무시 됩니다. 첫 template 의 default argument 는 사용되지 않습니다. 따라서 해당 parameter 는 ordering 에 참여하게 됩니다. 합성된 argument type 의 리스트는 (A1* A1 const*) 와 (A2 const*, A2 \*) 입니다. 두번째 template 에 비하여 (A1\*, A1 const \*)의 template argument 추론은 T 가 A1 const 로 치환되는것으로 성공합니다. 그러나 (A1\*, A1 const\*) 타입의 argument 가 t<A1 const>(A1 const\*, A1 const\*, A1 const\* = 0) 이 호출하기 위하여 qualification adjustment 이 필요로 하기 때문에 결과 일치가 정확하지 않습니다. 비슷하게 argument 타입 list( A2 const\*, A2 \* ) 에서 첫 template 의 template argument 추론하여 정확한 일치를 찾을수 없습니다. 그러므로 두 template 사이의 관계에는 ordering 이 없고 호출은 모호 합니다. formal ordering rule 은 function template 의 직관적으로 선택합니다. 그러나 가끔씩 rule 이 직관적인 선택을 채택하지 않는 예가 있습니다. 그러므로 앞으로는 이런 예제를 수용하여 rule 의 변경될 가능성이 있습니다.

### Templates and Non-templates

function template 은 non-template function 으로 overload 될수 있습니다. 호출되는 실제 함수를 선택할때에  non-template function 이 선호됩니다. 다음 예제를 보면..

```c++
#include <string>
#include <iostream>

template<typeame T>
std::string f(T)
{
    return "Template";
}

std::string f(int &)
{
    return "Nontemplate";
}

int main()
{
    int x = 7;
    std::count << f(x) << '\n';     // prints : Nontemplates
}
```
위 코드에 대한 결과는 Nontemplate 입니다.

그러나 const 와 reference qualifier 가 다를 경우 overload resolution 의 우선 순위는 변경 될수 있습니다. 예를 들면 :

```c++
#include <string>
#include <iostream>

template<typename T>
std::string f( T& )
{
    return "Template";
}

std::string f( int const & )
{
    return "Nontemplate";
}

int main()
{
    int x = 7;
    std::cout << f( x ) << '\n';    // prints: Template
    int const c = 7;
    std::cout < f (c ) <<'\n';      // prints: Nontemplate
}
```

위 코드에 대한 결과는 Template NonTemplate 입니다.

non constant int 를 그대로 보낼때 function template f<>(T&) 은 조금 더 일치 합니다. The reason is that for an int the instantiated f<>(int&) is better match than f(int const&). 하나의 함수는 template 이고 다른것은 template 이 아니것이 차이점입니다. 이 case 에서 overload resolution 의 일반적인 rule 이 적용됩니다(Section C.2 on page 682). int const 를 f() 호출할때 int const & type 를 둘가 가지고 있으므로 nontemplate 이 선호 됩니다.
이런 이유로 아래 처럼 member function template 을 선언하는 것은 좋은 생각 입니다.

```c++
template<typename T>
std::string f( T const \& )
{
    return "Template";
}
```

그럼에도 불구 하고 이런 효과는 실수를 쉽게 발생시킬수 있으며 copy 또는 move constructor 처럼 같은 argument가 허용되는 member function 을 정의 할때 놀라운 행동을 발생시킵니다. 예를 들면:

```c++
#include <string>
#include <iostream>

class C{
public:
    C() = default;
    C ( C const & )
    {
        std::cout << "copy constructor\n";
    }
    C( C&& )
    {
        std::cout << "move constructor\n";
    }
    template<typename T>
    C( T&& )
    {
        std::count << "template constructor\n";
    }
};

int main()
{
    C x;
    C x2{ x };              // prints: template constructor
    C x3{ std::move(x) }    // prints: move constructor
    C const c;
    C x4{ c }               // prints: copy constructor
    C x5{ std::move(c) }    // prints: template constructor
}
```

위 코드의 결과는 아래와 같습니다.

```
template constructor
move constructor
copy constructor
template constructor
```

이와 같이 member function template 은 copy constructor 보다 C 를 복사하는것이 더 일치 합니다. 그리고 std::move(c), which yields type C const && (a type that is possible but usually doesn't have meaningful semantics, 의 경우 move constrcutor 보다 member function template 이 더 일치 합니다.
이런 이유로 copy 나 move constructor 를 숨길 필요가 있을때 그런 member function template 를 부분적으로 비활성화 해야 합니다. 이것은 Section 6.4 on page 99 에서 설명 하였습니다.

### Variadic Function templates

parameter pack(Section 15.5 on page 275) 추론이 여러 argument 를 하나의 parameter 와 동일시 하기 때문에 Variadic function template( Section 12.4 on page 200) 은 partial ordering 하는 동안 약간의 특별한 처리가 필요로 합니다. 이런 처리는 function template ordering 의 흥미로운 상황에 대하여 소개 합니다. 다음 예제를 보면..

```c++
#include <iostream>

template<typename T>
int f( T* )
{
    return 1;
}

template<typename... Ts >
int f( Ts... )
{
    return 2;
}

template<typename... Ts>
int f( Ts* ...)
{
    return 3;
}

int main()
{
    std::cout << f( 0, 0.0 );                           // calls f<>(Ts...)
    std::cout << f( (int*)nulltpr, (double*)nullptr );  //calls f<>(Ts* ...)
    std::cout << f( (int*)nullptr );                    // calls f<>(T*)
}
```

위코드의 결과는 231 입니다.

처음 호출된 f(0, 0.0) 은 각 function template 이름 f 가 고려되어야 합니다. template parameter 가 추론될수 없고 이 non-variadic function template의 parameter 보다 argument 많 때문에 첫 function template f(T*) 은 추론이 실패 합니다. 두번째 function template f(Ts...)은 variadic 입니다. 이 케이스에서 추론은 두 agrument의 타입(int, double)과 function parameter pack(Ts)의 pattern 를 비교 합하여 Ts 를 sequence( int, double )로 추론합니다. 세번째 function template f(Ts*)에서 추론은 parameter pack Ts* 의 패턴과 argument 타입을 비교 합니다. 이 추론을 실패 되고 두번째 function template 만 유효 합니다. 그래서 function template ordering 은 필요 없습니다.

두번째 호출 f( int*)nullptr, (double*)nullptr ) 은 좀더 흥미롭습니다. parameter 보다 argument 가 많기 때문에 first template 은 추론에 실패 합니다. 그러나 두번째와 세번째 template 은 추론에 성공 합니다. 명시적으로 작성된 결과는 다음과 같습니다.

```c++
f<int *, double*>( (int*)nullptr, (double*)nullptr) // for second template
```
```c++
f<int, double>( (int*)nullptr, (double*)nullptr )   // for third template
```

partial oerdering 으로 두번째와 세번째 template 을 고려 하여야 합니다. 둘다 다음과 같이 variadic 합니다.: variadic template 에 Section 16.2.3. 에서 언급한 formal ordering rule 를 적용할때 각 template parameter pack 은 single make-up type, class template 이나 value 로 변경 된다. 예를들면
552

## Explicit Specialization

### Full Class Template Specialization

full specialization 은 세개의 토큰 \( template, <, > \) 으로 소개 된다. 추가적으로 class 이름 다음에는 specialization 으로 선언되 template argument가 나타납니다.

```c++
template<typename T>
class S {
public:
    void info()
    {
        std::cout << "generic \( S<T>::info\(\) \)\n";
    }
};

template<>
class S<void> {
public:
    void msg()
    {
        std::cout << "fully specialized \( S<void>::msg\(\) \)\n";
    }
};
```

full specialization 의 구현이 일반적인 정의와 어떤 관계도 어떻게 관련없는지에 유의해야 합니다. 이것은 다른 이름을 memeber function 에게 줄수 있도록 합니다. class template 의 이름으로만 연결점이 결정 되어 집니다. 특정된 template argument 리스트들은 template parameter 의 리스트와 일치해야만 합니다. 예를 들면 template type parameter 에 nontype value 를 명시하는것은 유효하지 않습니다. 그러나 default template argument 들을 가지고 있는 parameter 의 template argument들은 선택 사항 입니다.

```c++
template<typename T>
class Types
{
public:
    using I = int;
};

template<typename T, typename U = typename Types<T>::I>
class S;                // #1

template<>
class S<void>           // #2
{
public:
    void f();
};

template<>
class S<char, char>     // #3

template<>
class S<char, 0>;       // ERROR:0 cannot substitute U

int main()
{
    S<int> * pi;        // OK: uses #1, no definition needed
    S<int> e1;          // ERROR: uses #1, but no definition available
    S<void> * pv;       // OK: uses #2
    S<void, int>    sv; // OK: uses #2 definition avaiable
    S<void, char>   e2; // ERROR: uses #1, but no definition available
    S<char, char>   e3; // ERROR: uses #3, but no definition available
}

template<>
class S<char, char>     // definition for #3
{
};
```
이예제 또한 full specializatiion 의 선언이 꼭 정의될 필요는 없습니다. 그러나 full specialization 이 선언될때 지정된 template argument 에 대하여 generic definition 를 사용하지 않습니다. 이런 이유로 제공되지 않는 definition이 필요로 할때 프로그램은 에러를 발생 시킵니다. class template specialization은 때때로 상호 의족적인 type 를 구성할수 있는 forward declare type 를 유용하게 사용할수 있습니다. full specialization 선언은 normal class 선언과 동일 합니다. 하나의 차이점은 구문과 선언이 이전 template 선언과 일치 해야 한는것 입니다. template 선언이 아니기 때문에 full class template specialization 의 member 는 ordinary out-of-class member definition syntax를 사용하여 정의 됩니다.( 다른말로 하면 template<> 접두사는 명시될수 없습니다.)

```c++
template<typename T>
class S;

template<>
class S<char**>
{
public:
    void print() const;
};

void S<char**>::print() const
{
    std::cout << "pointer to pointer to char\n";
}
```

보다 복잡한 예제는 이 개념을 강화 할수 있습니다.

```c++
template<typename T>
class Outsize
{
public:
    template<typename U>
    class Inside
    {
    };
};

template<>
class Outsize<void>
{
    // there is no special connection between the following nested class
    // and the one defined in the generic template
    template<typename U>
    class Inside
    {
    private:
        static int count;
    };
};

// the following definition cannot be preceded by template<>
template<typename U>
int Outsize<void>::Inside<U>::count = 1;
```

full specialization 은 특정 generic template 의 인스턴스화를 대체 합니다. 그리고 동일한 프로그램에 있는 template 의 명시적인 버젼과 생성된 버젼을 모두 사용할수 없습니다. 같은 파일에서 둘다 사용하면 일반적으로 컴파일러에 의하여 알수 있습니다.

```c++
template<typename T>
calss Invalid
{
};

Invalid<double> x1;     // causes the instantiation of Invalid<double>

template<>
class Invalid<double>;  // Error: Invalid<double> already instantiated
```

불행히도 다른 translation unit 에서의 사용은 문제를 그리 쉽게 알수 없습니다. 다음의 잘못된 c++ 예제는 2개의 파일과 컴파일러 많은 구현체를 링킹하는것으로 구성 됩니다.

```c++
// translation unit 1:
template<typename T>
class Danger
{
public:
    enum
    {
        max = 10
    };
};

char buffer[Danger<void>::max]; // use generic value

extern void clear( char* );

int main()
{
    clear( buffer );
}

// translation unit 2:
template<typename T>
class Danger;

template<>
class Danger<void>
{
public:
    enum
    {
        max = 100
    };
};

void clear( char * buf )
{
    // mismatch in array bound:
    for( int k = 0; k < Danger<void>::max; ++k )
    {
        buf[k] = '\0';
    }
}
```
이 예제는 짮게 구현이 되었지만 일반 template 의 모든 사용자는 specialization 의 선언을 볼수 있도록 주위를 기울려야 하는것을 보여 줍니다. 실직적으로 specialization 의 선언은 그 header 파일의 template 선언을 따라야 합니다. generic 구현을 외부소스(과련된 header 파일을 수정할수 없는)에서 하는 경우 이것은 실용적이지 않다. 그러나 찾기 어려운 오류를 피하기 위하여 generic template 을 포함하는 header를 생성한 다음 specialization 의 선언을 하는것이 좋습니다. 명백하게 목적을 위하여 설계된 것으로 명확하게 표시되지 않는한 외부 소스의 spcializing template 을 피하는것이 좋습니다.

### Full Function Template Specialization

full function template specialization 뒤에 syntax 와 principle 은 full class template specialization 과 매무 많이 유사 합니다. 그러나 but overloading are argument deduction come into play. specialize된 template 은 argument 추론( argument type 으로 선언에 제공된 parameter type 을 사용할때 )과 partial ordering 으로 결정될때 Full specialization 선언은 명시적으로 template argument 를 생략할수 있습니다.


```c++
template<typename T>
int f( T )                  // #1
{
    return 1;
}

template<typename T>        // #2
int f( T* )
{
    return 2;
}

template<> int f( int )     // OK: specialization of #1
{
    return 3;
}

template<> int f( int* )    // OK: specialization of #2
{
    return 4;
}
```

full function template specialization 은 default argument value 을 가질수 없습니다. 그러나 specialize화된 template 으로 지정된 어떤 default argument 은 명시적 specialization 에 적용할수 있습니다.

```c++
template<typename T>
int f( T, T x = 42 )
{
    return x;
}

template<> int f( int, int = 35 ) // ERROR
{
    return 0;
}
```

(full specialization 은 대안적인 정의를 제공하지만 대안적인 선언을 제공하기 때문이다. function template 를 호출할때 function template 기반으로 문제를 해결 합니다.) full specialization 은 일반적인 선언과 매우 많이 비슷 합니다. 특히 template 을 선언하지 않으므로 non-line full function template specialization의 정의가 프로그램에 나타납니다. 그러나 일반적으로 full specialization 의 선언은 두개의 파일로 구성 됩니다.

- interface 파일은 primary template 와 partial specialization의 정의 를 포함되지만 full specialization 을 선언합니다.

```c++
#ifndef TEMPLATE_G_HPP
#define TEMPLATE_G_HPP

// template definition should appear in header file:
template<typename T>
int g( T, T x = 42 )
{
    return x;
}

// specialization declaration inhibits instantiations of the template:
// definition should not appear here to avoid multiple definition errors
template<> int g( int, int y );

#endif
```

- 해당하는 구현 파일은 full specialization 을 정의 합니다.

```c++
#include "template_g.hpp"

template<> int g( int, int y)
{
    return y/2;
}
```

또는 specialization 은 inline 으로 만들수 있고 이 경우 그것의 정의는 header 파일에 위치합니다.

### Full Variable Template Specialization

variable template 은 fully specialize 될수 있습니다. 지금쯤이면 syntax 는 직관적일수 있습니다.

```c++
template<typename T> constexpr std::size_t SZ = sizeof(T);
template<> constexpr std::size_t SZ<void> = 0
```

분명히 specialization 은 template 의 으로 부터 발생되는 것과 구분되는 initializer 를 제공할수 있습니다. 흥미롭게도 variable template specialization 은 specialize 된 template과 일차하는 타입을 가질 필요는 없습니다.

```c++
template<typename T> typename T::iterator null_iterator;
tempate<> BitIterator null_iterator<std::bitset<100>>;
// BitIterator doesn't match T::iterator, and ths is fine
```

### Full Member Specialization

member template 뿐만 아니라 class template 의 일반 statis data member 와 member function 도 fully specialize 될수 있습니다. The syntax requires template<> prefix for every enclosing class template. member template 가 specialize 되어 있을 경우 tempate<> 은 추가하여 specailize 가 될어 있다는것을 나타내어야 합니다. To illustrate the implications of this, let's assume the following declarations:

```c++
template<typename T>
class Outer                 // #1
{
public:
    template<typename U>
    class Inner             // #2
    {
    private:
        static int count;   // #3
    };

    static int code;        // #4

    void print() const      // #5
    {
        std::cout << "generic";
    }
};

template<typename T>
int Outer<T>::code = 6      // #6

template<typename T> template<typename U>
int Outer<T>::Inner<U>::count = 7;  // #7

template<>
class Outer<bool>               // #8
{
public:
    tempate<typename U>
    class Inner                 // #9
    {
    private:
        static int count;       // #10
    };

    void print() const          // #11
    {
    }
};
```

\#4의 일반적인 member code 와 \#1의 generic outer template \#1 의 \#5에 위치한 print() 함수는 single enclosing class template 을 가지고 있습니다. 따라서 완전히 template argument 의 set 을 specific 를 하기 위해서는 template<> prefix 가 필요로 합니다.

```c++
tempate<>
int Outer<void>::code = 12;

tempate<>
void Outer<void>::print() const
{
    std::cout << "Outer<void>";
}
```

이런 정의는 Outer<void> class 에 #4 와 #5 의 일반적인 정의가 사용 되었습니다. 그러나 Outer<void> class 의 다른 member 들은 #1 지점의 template 으로 부터 생성됩니다. 이런 정의이후에 Outer<void> 의 명시적인 specialize 을 제공하지 않습니다. full function template specialization 과 마찬가지로 정의를 지정하지 않은 class template 의 ordinary member 의 specialization 을 선언하는 방법이 필요로 합니다.( multiple 정의를 파하기 위해서). 비록 c++ 에서 member function 과 statis data member에 대해 정의되지 않은 out-of-class 의 선언은 사용될수 없지만 class template  의 memeber 를 specialize 할때는 괜찮습니다. The previous definitions could be declared with

```c++
template<>
int Outer<void>::code;

template<>
void Outer<void>::print() const;
```

Outer<void::code 의 full specialization의 정의되지 않은 선언이 default constructor 로 초기화하는 정의와 같게 보입니다. 이것은 실제로 그렇지만 그런 선언은 정의되지 않은 선언으로 항상 해석 됩니다. For a full specialization of a static data member, with a type that can only be initialized using a default constructor, we must resort to initializer list syntax.

```c++
class DefaultInitOnly
{
public:
    DefaultInitOnly() = default;

    DefaultInitOnly( DefaultInitOnly const &) = delete;
};

template<typename T>
class Statics
{
private:
    static T sm;
};
```

```c++
template<>
DefaultInitOnly Statics<DefaultInitOnly>::sm;
```

다음은 default constructor 를 호출하는 정의 입니다.

```c++
template<>
DefaultInitOnly Statics<DefaultInitOnly> sm{};
```

c++11 이전에는 이것은 가능하지 않습니다. 그런 specialization 에서 default initialization를 사용할수 없습니다.

```c++
template<>
DefaultInitOnly Statics<DefaultInitOnly>::sm = DefaultInitOnly();
```

불행히도 copy constructor 를 삭제하였기 때문에 이 예제는 가능하지 않습니다. 그러가 c++17 은 copy constructor 호출이 더이상 관련되지 않기 때문에 가능하게 할수 있는 copy-elision rule 이 있습니다.
member template Outer<T>::Inner는 Outer<T> 의 특정 인스턴스화된 다른 member 에 영향 없이 주어진 template argument 에 대해 specialize 할수 있습니다. 다시말하면 하나의 enclosing template 이기 때문에 template<> prefix 가 필요합니다. 그 결과의 코드는 아래와 같습니다.

```c++
template<>
template<typname X>
class Outer<wchar_t>::Inner{
public:
    static long count;  // member type changed
};

template<>
template<typename X>
long Outer<wchar_t>::Inner<X>::count;
```

Outer<T>::Inner template 은 fully specialize 될수 있지만 Outer<T> 의 주어진 인스턴스에 대해서만 specialize 될수 있습니다. 두개의 template<> prefix 가 있습니다. 하나는 enclosing class 이기 때문이고 다른 하나는 fully specializing template 하기 때문입니다.

```c++
template<>
template<>
class Outer<char>::Inner<wchar_t>
{
public:
    enum
    {
        count = 1
    };
};

// following is not valid C++
// template<> cannot follow a template parameter list
template<typename X>
template<>
class Outer<X>::Inner<void>;    // Error
```

Outer<bool> 의 member template 의 specialize 와 차이가 있습니다. 후자는 이미 fully specialize 되어 있기 때문에 enclosing template 가 없습니다. 그리고 하나의 templata<> prefix 가 존재 합니다.

```c++
template<>
class Outer<bool>::Inner<wchar_t>
{
public:
    enum
    {
        count = 2
    };
};
```

## Partial Class Templates Specialization

Full template specialization 은 종종 유용하지만 하나의 특정 template argument 의 집합 보다는 class template 이나 template argument 의 군을 위하여 class template 이나 variable template 를 specialize 하는것이 자연 스럽습니다. 예를 들면 linked list 을 class template 으로 구현 하였다고 가정해 봅시다.

```c++
template<typename T>
class List                                  // #1
{
public:
    ...
    void append( T const& );
    inline std::size_t length() const;
    ...
};
```

이 template 를 사용하는 큰 프로젝트에서는 많은 타입의 member 들을 인스턴스화 할것 입니다. inline( List<T>::append() ) 화 되지 않는 member function 들 때문에 object 코드가 눈에 띄게 증가할수 있습니다. 그러나 low-level 에서는 List<int*>::apend() 와 List<void*)::append 의 코드가 같다는것을 알수 있습니다. 다른말로 하면 모든 pointer 의 list 는 구현을 공유하도록 지정하려고 합니다. 비록 이것이 c++ 에서 표현할수 없다고 하더라도 모든 포인터의 List 들은 다른 template 정의로 부터 인스턴스화 되는것을 지정함으로써 아주 근접하게 만들수 있다.

```c++
template<typename T>
class List<T*>
{
private:
    List<void*> impl;
    ...
public:
    ...
    inline void append( T* p )
    {
        impl.append( p );
    }
    inline std::size_t length() const
    {
        return impl.length();
    }
    ...
};
```

이 context 에서 \#1 에서 지정된 기본 template은 primary template 이라고 부르고 뒷 부분 정의는 partial specialization 이라고 부릅니다.(이 template 정의를 사용해야하는 template argument 가 부분적으로 지정되었기 대문이다.) partial specialization 을 특징하는 syntax 는 class template 이름에 명시적으로 지정된 template argument(<T*>) 와 template parameter 리스트 선언( template<...> )과의 조합입니다.
List<void*> 같은 List<void*) 타입을 재귀적으로 포함하기 때문에 문제가 있습니다. 이 싸이클을 없애기 위하여 full specialization을 전 partial specialization 보다 먼저 하여야 합니다.

```c++
template<>
class List<void*>       // #3
{
    ...
    void append( void * p );
    inline std::size_t length() const;
};
```

partial specialization 보다 full specialization 보다 선호되기 때문에 효과가 있습니다. 결과적으로 모든 list 의 member function 은 list<void*> 의 구현체로 전달됩니다. 이것은 코드가 커지는것을 방지하는데 효과적입니다.

partial specialization 선언의 parameter 와 argument 리스트을 제한하는것이 몇가지 존재한다.
1. partial specialization 의 arguement는 primary template 의 parameter 와 종류(type, nontype, template)가 일치해야 합니다.
2. partial specialization 의 parameter 리스트는 default argument 를 가질수 없습니다. primary class template 의 default argument가 대신 사용 됩니다.
3. partial specialization 의 nontype argument 는 nondependent value 이거나 plain nontype template parameter 입니다. They cannot be more complex dependent expression like 2*N (where N is a template parameter).
4. partial specialization 의 template arguement 리스트들은 primary template 의 parameter 리스트와 동일 해서는 안됩니다.(이름 변경은 무시 합니다.)
5. template argument 중 하나가 pack expansion 인 경우 template argument list 의 끝에 있어야 합니다.

다음 예제는 이런 제한들을 보여주고 있습니다.

```c++
template<typename T, int I = 3>
class S;                // primary templates

template<typename T>
class S<int, T>;        // ERROR : parameter kind mismatch

template<int T = int>
class S<T, 10>;         // ERROR : no default arguments

template< int I>
class S<int, I*2>;      // ERROR : no nontype expression

template<typename U, int K>
class S<U, K>;          // ERROR : no significant difference from primary templates

template<typename... Ts>
class Tuple;

template<typename Tail, typename... Ts>
class Tuple<Ts... , Tail>;  // ERROR : pack expansion not at the ends

template<typename Tail, typename... Ts>
class Tuple<Tuple<Ts..>, Tail>; // OK : pack expansion is at the end of a
                                // nested template arguments
```

모든 partial specialization 은 모든 full specialization 처럼 primary template 과 관련이 있습니다. template 이 사용 될때 primary template 은 항상 검색되고 있지만

function template argument 추론을 사용하는 SFINAE 원리가 여기에 적용 됩니다.




## Partial Variable Template Specialization

## Afternoetes
