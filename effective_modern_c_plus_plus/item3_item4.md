
## Item 3: Understand decltype
## Item 4: Know ho to view deduced types

### Jerry

---

### Item3: decltype Keyword
* decltype is a keyword used to query the type of an expression

``` c
const int i = 0;          // decltype(i) -> const int
bool f(const Widget& w);  // decltype(w) -> const Widget&
                          // decltype(f) -> bool(const Widget&)
Widget w;
if (f(w)) {               // decltype(f(w)) -> bool
  ....
}
```

----

#### The decltype of operator[]

* decltype(Container&lt;T&gt;[i]) -> T&
* decltype(vector&lt;bool&gt;[i]) -> not bool& but vector&lt;bool&gt;::reference

``` c
// How to get the type of c[i]?
// C++11 only could deduce the return type for single-statement lambdas.
// Use "trailing return type" here.
template<typename Container, typename Index>
auto authAndAccess(Container& c, Index i) -> decltype(c[i])
{
  authenticateUser();
  return c[i];
}
```

----

#### The decltype of operator[]
``` c
// C++14
// Use more powerful auto in C++14?
template<typename Container, typename Index>
auto authAndAccess(Container& c, Index i)
{
  authenticateUser();
  return c[i];  // return type deduced from c[i]
}

deque<int> d;
authAndAccess(d, 5) = 10; // <== error here
                          // return type is int, not int&
                          // It's a rvalue.
```

``` c
// recall item 1:
// auto for c[i] is just like this template function f() as following.
// If c[i] is a reference, ignore the reference part.
template<typename T>
void f(T param);
```

----

#### The decltype of operator[]
``` c
// C++14
// combime decltype and auto
template<typename Container, typename Index>
decltype(auto) authAndAccess(Container& c, Index i)
{
  authenticateUser();
  return c[i];
}
```

----

#### auto and decltype(auto)
``` c
// C++14
Widget w;
const Widget& cw = w;

auto myWidget1 = cw;            // myWidget1 => Widget

decltype(auto) myWidget2 = cw;  // myWidget2 => const Widget&
```

----

#### Final refinement of authAndAccess()
``` c
// C++14
template<typename Container, typename Index>
decltype(auto) authAndAccess(Container& c, Index i);

deque<string> makeStringDeque();
auto s = authAndAccess(makeStringDeque(), 5); // <== error
// rvalue can't bind to lvalue reference.
// makeStringDeque() can't bind to c.
// (rvalue can bind to lvalue-reference-to-const)
```
* Resolution:
  * overloading for lvalue and rvalue reference
  * use universal reference

----

#### Final refinement of authAndAccess()
``` c
// C++14
// use universal reference
template<typename Container, typename Index>
decltype(auto) authAndAccess(Container&& c, Index i)
{
  authenticateUser();
  return forward<Container>(c)[i];  // use forward for universal reference
                                    // (use move for rvalue reference)
}
```
``` c
// C++11
// use universal reference
template<typename Container, typename Index>
auto authAndAccess(Container&& c, Index i) -> decltype(forward<Container>(c)[i])
{
  authenticateUser();
  return forward<Container>(c)[i];  // use forward for universal reference
}
```

----

#### decltype on "name" or other lvalue expression

``` c
decltype(auto) f1()
{
  int x = 0;
  return x; // decltype(x) -> int
}

decltype(auto) f2()
{
  int x = 0;
  return (x); // decltype((x)) -> int&
}
```

---

### Item4: Show the deduced types
* Use IDE
* Use compiler
* typeid operator
* boost::typeindex

``` c
template<typename Type>
class TypeDisplayer;

TypeDisplayer<decltype(x)> xType; // Show the error message with x type.
```

``` c
#include <typeinfo>

cout << typeid(x).name(); // typeid return type_info object which contains type info.
```

----

#### typeid
``` c
vector<Widget> createVec();
const auto vw = createVec();
f(&vw[0]);

template<typename T>
void f(const T& param)
{
  cout << typeid(param).name(); // Param's type is "const Widget* const &" but
                                // get "const Widget*".
                                // typeid is just like the type deducing for
                                // non-reference parameter type. Please check item 1.

}
```

----

#### boost::typeindex
* Since Boost 1.56(Ubuntu 15.10 has boost 1.58 package)

``` c
template<typename T>
void f(const T& param)
{
  cout << boost::typeindex::type_id_with_cvr<T>().pretty_name();
  cout << boost::typeindex::type_id_with_cvr<decltype(param)>().pretty_name();
}
// T -> Widget const*
// param -> const Widget* const &
```
