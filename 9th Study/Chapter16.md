# Specialization and Overloading

이 챕터에서는 template Specialization 와 function template 의 overloading 에 대하여 알아 볼것 이다.

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

```c++
template<typename T>
void quickExchange( T * a, T * b )      // #1
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

## Overloading Function Templates

### Signatures

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
