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
but overloaded resolution will never prefer one over the other. template 이 다른 변환 유닛으로 보여 진다면 같은 프로그램에서 


### Partial Ordering of Overloading

### Formal Ordering Rules

### Templates and Non-templates

### Variadic Function templates

## Explicit Specialization

### Full Class Template Specialization

### Full Function Template Specialization

### Full Variable Template Specialization

### Full Member Specialization

## Partial Class Templates Specialization

## Partial Variable Template Specialization

## Afternoetes
