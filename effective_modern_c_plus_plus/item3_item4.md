<!--Meta author:'Jerry' theme:'night' title:Effect Modern C++-->
<!--Meta width:1440 height:900-->
<!--Meta margin:0-->
<!--Meta minScale:0.5-->

## Item 3: Understand decltype
## Item 4: Know ho to view deduced types

### Jerry

---

### decltype Keyword
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

### The decltype of operator[]

* decltype(Container<T>[i]) -> T&
* decltype(vector<bool>[i]) -> not bool& but vector&lt;bool&gt;::reference

``` c
// C++11 only could deduce the return type for single-statement lambdas.
// Use "trailing return type" here
template<typename Container, typename Index>
auto authAndAccess(Container& c, Index i) -> decltype(c[i])
{
  authenticateUser();
  return c[i];
}
```

----

### The decltype of operator[]
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
// recall item 1
// auto for c[i] is just like this template function f
// If c[i] is a reference, ignore the reference part.
template<typename T>
void f(T param);
```

----

### The decltype of operator[]
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

### auto and decltype(auto)
``` c
// C++14
Widget w;
const Widget& cw = w;

auto myWidget1 = cw;            // myWidget1 => Widget

decltype(auto) myWidget2 = cw;  // myWidget2 => const Widget&
```

----

### Final refinement of authAndAccess()
``` c
// C++14
template<typename Container, typename Index>
decltype(auto) authAndAccess(Container& c, Index i);

deque<string> makeStringDeque();
auto s = authAndAccess(makeStringDeque(), 5); // <== error
// rvalue can't bind to lvalue reference.
// Note: rvalue can bind to lvalue-reference-to-const.
```
* Resolution:
  * overloading for lvalue and rvalue reference
  * use universal reference
``` c
// C++14
// use universal reference
template<typename Container, typename Index>
decltype(auto) authAndAccess(Container&& c, Index i)
{
  authenticateUser();
  return forward<Container>(c)[i];  // use forward for universal reference
}
```
