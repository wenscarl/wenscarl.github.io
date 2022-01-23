---
layout: post
title: What does std::forward do?
tag: c++
date: 2022-01-24
published: true
categories: programming


---

`std::forward` is another signature feature of c++11 beside `std::move`. Technically, it does perfect forwarding. But of what and when? In this post, we will explore some use cases and context behind.

<!--more-->

__C++11 lets us perform perfect forwarding, which means that we can forward the parameters passed to a function template to another function call inside it without losing their own qualifiers (const-ref, ref, value, rvalue, etc.).__

# THE standard 

```c++

class Foo{
 public:
  template <typename T>
  foo(T&& p) : _p(std::forward<T>(p)) {}
};

```

The above snippet covers almost 90% occurrence but with lots of pretexts and c++ rules. Boiler first.

   1.___ Reference folding  ___

1. X& &, X& &&, X&& & -> X&
2. X&& && -> X&&

2. A special rule for rvalue reference. When a left value is passed to a function whose argument is a right-valued reference, and this right-valued reference points to a template type argument (T&&), the compiler infers that the template argument type is a left-valued reference to the real parameter, as in 

   ```c++
   template<typename T> 
   void f(T&&);
   int i = 0;
   f(i)
   ```

   T is int& but not int. Again, according to the reference folding rule `void f(int& &&)` will be inferred as `f(int&)`, so f will be instantiated as: `f<int&>(int&)`. That make sense because we never intend to pass in a rvalue.

3. ___`std::forward`cast its parameter(here `p`) to a rvalue only when the parameter binds to a rvalue.___

   Here a example is better to illustrate the idea. For *THE standard* code,

   - pass in as 

     ```c++
     int i=0; foo(i);
     ```

     rule #2 is satisfied and `T = int&`. `foo<int&>(int& && p) -> foo<int&>(int& p)` is called. Here parameter `p` is a rvalue and rvalue reference.

   - pass in as

     ```c++
     foo(2);
     ```

     `T = int`, and `foo<int>(int&& p)` where `p` is a rvalue reference but lvalue itself. Passing to inner function as rvalue reference, we have to do `std::move(p)`. 
     A concise way to combine such 2 cases is `std::forward(p)`. What `std::forward(p)` returns is a lvalue or rvalue exactly the same as whatever the argument is. Let conclude with a complete example.

     ```c++
     #include <iostream>
     #include <type_traits>
     #include <typeinfo>
     #include <memory>
     using namespace std;
     
     class A {
       public:
         A(int&& a) : _a(a) {
             cout << "move , a=" << a << endl;
         }
         A(int& a)  : _a(a) {
            cout << "copy , a=" << a << endl;
         }
     
       private:
        int _a;
     };
     
     class B {
     public:
         template<class T1, class T2, class T3>
         B(T1 && t1, T2 && t2, T3 && t3) :
             a1_(std::forward<T1>(t1)),
             a2_(std::forward<T2>(t2)),
             a3_(std::forward<T3>(t3)) {}
     private:
         A a1_, a2_, a3_;
     };
     
     template <class... U>
     std::unique_ptr<B> make_unique(U&&... u) {
         return std::unique_ptr<B>(new B(std::forward<U>(u)...));
     }
     
     int main() {
         int i = 5;
         auto p1 = make_unique(i, 2, std::move(i));
         return 0;
     }
     // output
     copy , a=5
     move , a=2
     move , a=5
     ```

      