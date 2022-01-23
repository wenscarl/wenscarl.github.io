---
layout: post
title: What does std::move do at parameter binding?
tag: c++
date: 2022-01-23
published: true
categories: programming

---

`std::move` is the signature feature of c++11 and often used in resource management so called RAII(resource acquisition is initialization). In this post, we just focus on 1 scenario that appears frequently in factory mode and demonstrate how compiler treats it at parameter binding. 

<!--more-->

Some term clarification:

```
void foo(int p1);
int a = 1;
foo(a);
```
we refer `p1` as parameter and `a` as argument of `foo`. 

Say we have a resource class as

```
Struct Resource {};
```

and a handle class to hold the resource as,

```c++
class handle {
 public:                                                     
  handle () { res = new Resource(); }                               
  ~handle () { if (res) delete res; }
  handle (const handle & other){
    cout << "copy constructor" << endl;
    cout<<"At entrance: other.res="<<other.res<<endl;
    res = new Resource(*other.res);
    cout<<"I am set to="<<res<<endl;
  }
                                                        
  handle (handle && other) {
    cout << "move constructor" << endl;
    res = other.res;
    other.res = nullptr;
  }
                                                        
  handle & operator=(handle && other){
    cout << "move assignment" << endl;
    std::swap(res, other.res);
    return *this;
  }
  Resource* get_res() const { return res; }
 private:
  Resource* res;
};
```

Then we need a wrapper class as an aggregate of couple of handles like this

```c++
class wrapper{
 public:
  wrapper(handle &&handle) :
   _handle_1(std::move(handle)) {
       cout<<"private member  holds resource "<<_handle_1.get_res()<<"; parameter holds " <<
           handle.get_res()<<endl;}  
    
  handle* get_handle()  {return &_handle_1;}
  wrapper(wrapper&& x) = default;
 private:
  handle _handle_1;
};
```

 Assume there is a user case where the ownership needs to be passed around.

```
handle h4;
cout <<"h4 res="<<h4.get_res()<<endl;
unique_ptr<wrapper> m(new wrapper(std::move(h4)));
cout <<"h4 res="<<h4.get_res()<<endl;
cout <<"m res="<<instance->get_holder()->get_res()<<endl;
```

It's nearly a common ground that we need to use `std::move(h4)` as argument to construct `wrapper()`, but the key is how to write the initialization list of the constructor for class wrapper. We have the following 4 cases.

1. `wrapper(handle &&handle) : _handle_1(std::move(handle))`
2. `wrapper(handle &&handle) : _handle_1(handle)`
3. `wrapper(handle handle) : _handle_1(std::move(handle))`
4. `wrapper(handle handle) : _handle_1(handle)`

- ```c++
  // output #1
  h4 res=0x55fbc9929eb0
  move constructor
  private member holds resource 0x55fbc9929eb0; parameter holds 0
  h4 res=0
  m res=0x55fbc9929eb0
  Success! It's what we want and  only 1 move constructor is called. The compiler it smart enough to `swap(h4, _handl_1)`.
      
  // output #2
  h4 res=0x564af9158eb0
  copy constructor
  At entrance: other.res=0x564af9158eb0
  I am set to=0x564af9159300
  private member holds resource 0x564af9159300; parameter holds 0x564af9158eb0
  h4 res=0x564af9158eb0
  m res=0x564af9159300
  Failure. No move at all because we never mean  to `move` explicitly inside wrapper. Parameter handle is a lvalue though itself is a rvalue reference. No wonder class handle invokes its copy constructor. So std::move(handle) is indispensable. Luckily, `h4` is not tampered and `h4->temp` is skipped or optimized by compiler. With std::forward() syntex, perfect forwarding, we can preserve the qualifiers (const-ref, ref, value, rvalue, etc.). of parameter and will be discusssed in another post.
      
  // output #3
  h4 res=0x555ca6e7ceb0
  move constructor
  move constructor
  private member holds resource 0x555ca6e7ceb0; parameter holds 0
  h4 res=0
  m res=0x555ca6e7ceb0
  Success! But 2 moves. Interestingly, no copy constructor! `h4->temp->_handle_1`
      
  // output #4
  h4 res=0x56100474deb0
  move constructor
  copy constructor
  At entrance: other.res=0x56100474deb0
  I am set to=0x56100474e300
  private member holds resource 0x56100474e300; parameter holds 0x56100474deb0
  h4 res=0
  m res=0x56100474e300
  Failure. Really bad because `h4->temp` and gets lost!


Based on analysis above, I would actually recommend case 3 over 1 though one more time of `move`. Reason is case 3 is more general in which case the input argument can be rvalue or lvalue. Moreover, `move` is implemented by `swap` under the hood(usually) thus not too much overheads. 
