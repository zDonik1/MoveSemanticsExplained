# MoveSemanticsExplained

This reposity contains my explanation of lvalues, rvalues, references, move semantics and perfect forwarding.

Note that when I am talking about references here, I mean both lvalue and rvalue references. If a specific reference is discussed, it is explicitly written.

## Table of contents

> 1. [lvalues and rvalues](#lvalues-and-rvalues)
> 1. [lvalue and rvalue references](#lvalue-and-rvalue-references)
> 1. [Move semantics](#move-semantics)
> 1. [Forwarding references](#forwarding-references)
> 1. [Perfect forwarding](#perfect-forwarding)
> 1. [Rare use cases](#rare-use-cases)
>
> [References](#references)

<a name="lvalues-and-rvalues"></a>
## 1. lvalues and rvalues

**lvalue** (locator value) is a type of expression that has a specific address in memory and can be identified through that memory address.

Examples of lvalues include:
* any variables `int var;` or `int *ptr;` or `int &ref;` or `int &&rref`
* dereferenced addresses `*ptr` or `*(ptr + 1)`
* string literals `"Hello world!"`  

**rvalue** are all other expressions that are not lvalues, meaning that they have no memory address and therefore cannot be located.

Examples of rvalues include:
* literals (non-string) `10`
* temporary objects `MyClass()`
* expressions `(x + 10)`
* addresses `&var`

Typical lvalues stand on the left side of an assignment expression while typical rvalues stand on the right.

```
int a = 1;     // a is an lvalue, 1 is an rvalue
int b = 2;     // b is an lvalue, 2 is an rvalue
int c = a + b; // + needs rvalues, so a and b are converted to rvalues and an rvalue is returned
```

As seen in the previous code block, lvalues such as `a` and `b` can be implicitly converted to rvalues, however the opposite cannot be true, meaning that rvalues cannot be converted to lvalues.

Having said that, lvalues can be produced from a rvalues and vice versa is also true.

```
int arr[] = {1, 2};
int* p = &arr[0];
*(p + 1) = 10;   // OK: p + 1 is an rvalue, but *(p + 1) is an lvalue

int var = 10;
int* bad_addr = &(var + 1); // ERROR: lvalue required as unary '&' operand
int* addr = &var;           // OK: var is an lvalue
&var = 40;                  // ERROR: lvalue required as left operand of assignment
```

Previous block shows that dereferencing produces an lvalue from an rvalue, and getting the address of an lvalue produces an rvalue (the address).

Expressions that calls a function `foo()` can be an lvalue or an rvalue based on what it returns. If the return type is a reference, then it is an lvalue, otherwise it is an rvalue. Functions that return `void` cannot be resolved into an expression.

<a name="lvalue-and-rvalue-references"></a>
## 2. lvalue and rvalue references

lvalue references "point" or refer to an lvalue. Similarly, rvalue references refer to an rvalue.

```
int var = 10;
int &ref = var;     // lvalue reference, referring to var
int &&rref = 10;    // rvalue reference, referring to 10 (memory allocated for 10)
```

The expression being classified as an rvalue or an lvalue doesn't depend on its type. A quick example is that a type `int` can be either a lvalue using a variable `var` or a rvalue using a literal `10`. The continuing the last example, the following code shows different expression results and what their type is

```
ref;                         // lvalue of type lvalue reference
rref;                        // lvalue of type rvalue reference
static_cast<Int &&>(var);    // rvalue of type rvalue reference
```

Since references return an address (have an identifiable memory location), when an rvalue reference is created for a literal (for example an integer `10`), memory is allocated for the literal (or expression that resolves into a constant), therefore making the identifer a lvalue, even though its type is rvalue reference. Also, when an rvalue reference is created for a temporary object, the lifetime of the temporary object is extended until the rvalue reference is destroyed (for example when it falls out of scope).

Having said that, an rvalue expression of type lvalue reference cannot exist, and the expression becomes a lvalue, which means that `static_cast<Int &>(var)` is a lvalue. The `static_cast<Int &&>(var)` generates a rvalue temporary object of rvalue reference type. This specifically is used in movement semantics, and that's how `std::move` is implemented. A function `void foo(Int &&)` needs to bind to a rvalue, and the cast returns the rvalue that is wanted.

However, that begs the question as to why `static_cast<Int>(var)` cannot be used. The problem is that while casting, a temporary object is created, and a temporary of type `Int` means that variable `var` gets copied into the temporary object (copy constructor is called). Therefore the type for casting to move should specifically be a rvalue reference. As mentioned earlier, casting to lvalue reference gives a lvalue.

The following piece of code might be interesting to look at.

```
int var = 10;
int &ref = var;
int &&rref = 10;
int *ptr = &var;

std::cout << ptr;      // returns address of var
std::cout << &ptr;     // returns address of ptr
std::cout << &ref;     // returns address of var (address of ref is inaccessible)
std::cout << &rref;    // returna address of where 10 was allocated (address of rref is inaccessible)
```

References will always return the address of what is being referred.

Since the const lvalue reference isn't changing what it is referring to, it could get assigned with lvalues and rvalues, although neither being modifiable. This means that const rvalue references are redundant and shouldn't generally be used.

```
int var = 10;

int &refToLvalue = var;           // OK
int &refToRvalue = 10;            // ERROR: rvalue cannot be assigned to an lvalue reference
const int &cRefToLvalue = var;    // OK
const int &cRefToRvalue = 10;     // OK: rvalue can be assigned to a const lvalue reference 

int &&rrefToLvalue = var;         // ERROR: lvalue cannot be assigned to an rvalue reference
int &&rrefToRvalue = 10;          // OK
const int &&cRrefToLvalue = var;  // ERROR: lvalue cannot be assigned to an rvalue reference
const int &&cRrefToRvalue = 10;   // OK: but redundant, can use const lvalue reference
```

### Functions and references

Functions should accept values in the following ways

```
void foo1(int var);              // by value, if copying isn't a problem (for plain data types)
void foo2(const int var);        // by const value, if you want to strictly follow the rule of immutable parameters
void foo3(MyClass &obj);         // by lvalue reference, if object should be mutable
void foo4(const MyClass &obj);   // by const lvalue reference, if object should be only read (can accept rvalues)
void foo5(MyClass &&obj);        // by rvalue reference, if rvalues that are mutable should be accepted
```

Mutable rvalues are obviously not necessary since we can't use the rvalue later and the rvalue will get destroyed, however they are useful in cases where movement should ocurr (which will be talked about later on).

Most of the time only parameters are passed by value or const lvalue reference, however sometimes rvalue references are necessary, like in the following example

```
class MyClass1
{
private:
    std::vector<std::string> strings;

public:
    void addString(const std::string &str) {
        strings.push_back(str);
    }
    
    void addString(std::string &&str) {
        strings.push_back(std::move(str));    // move should be used here, since str is an lvalue, 
                                              // ... and without moving, it will call push_back(const T &)
    }
};

class MyClass2
{
private:
    std::vector<std::string> strings;

public:
    void addString(std::string str) {
        strings.push_back(std::move(str));    // always moves the str
    }
};

int main() {
    MyClass1 obj1;
    obj1.addString("Hello World!");    // the string literal is implictly converted to std::string, 
                                       // ... which is then passed as rvalue, so push_back(T &&) is called
    
    std::string str1 = "Hello World!";
    obj1.addString(str1);              // an lvalue string is passed, so push_back(const T &) 
                                       // ... is called and the string is copied
    obj1.addString(std::move(str1));   // the string is converted to an rvalue using move, 
                                       // ... so push_back(T &&) is called and the string is moved
    
    // do not use str1 from here on, since it has been moved
    
    MyClass2 obj2;
    obj2.addString("Hello World!");    // the string literal is implictly converted to std::string, 
                                       // ... which is an rvalue so it is moved to parameter variable
    
    std::string str2 = "Hello World!";
    obj2.addString(str2);              // an lvalue string is passed, so the string is copied
                                       // ... to the parameter variable
    obj2.addString(std::move(str2));   // the string is converted to an rvalue using move, 
                                       // ... so it is moved to parameter variable
                                       
    return 0;
}
```

In the previous example it is important to note 2 things. 

First is that `push_back(T &&)` has this signature, and you will later see that `T &&` is a forwarding reference, however forwarding references apply only to function templates, while in the case of class `std::vector<T>` the T comes from the class template. You can read about forwarding references [later](#forwarding-references) on. 

The second thing to note is that in the second case, where the string is passed by value, for lvalues the string is copied to the parameter variable and then moved into the `std::vector`, while for rvalues the string is moved into the parameter variable and then moved into the `std::vector`. As you can see there is an extra move operation for both lvalues and rvalues, while in the first case, there was only one operation, either a copy or a move. However moves are usually very cheap to do, so the first case should be used only when the performance gain is significant, so much so that it overweighs the disadvantage of code duplication, and also when there aren't many parameters (preferably use only if 1 parameter). Each new parameter for the function would add an extra variation and the number of variations should be 2^n, where n is the number of parameters.

Returning values from functions should be done in the following ways

```
MyClass foo1();            // by value, if temporary objects or plain data types are returned (expression is an rvalue)
MyClass &foo2();           // by lvalue reference, if the object should be mutable 
                           // ... (only useful in class methods, expression is an lvalue)
const MyClass &foo3();     // by const lvalue reference, if the object is used only to be read 
                           // ... (only useful in class methods, expression is an lvalue)

```

If you have questions on functions that return rvalue references, refer [here](#return-rvalue-reference).

Temporary variables created in the function (or rvalues and temporary objects) should be returned by value, since Return Value Optimization (RVO) automatically constructs the temporary object in place of where the function is being called (no copies are made).

```
MyClass foo1() {
    MyClass instance;
    
    // do some operations on instance
    
    return instane;    // returning an lvalue, RVO activated
}

MyClass foo2() {
    return {};     // returning an rvalue, RVO activated
}
```

<a name="move-semantics"></a>
## 3. Move semantics

Movement semantics works by using rvalue references. A temporary object will be deleted anyways, so instead of copying its contents, a move can be applied, stealing its contents for a new object.

```
MyClass obj;
MyClass obj2{obj};              // copy constructor called
MyClass obj3{std::move(obj)};   // move constructor called
// obj should not be used since it was moved
```

Here is how it works

```
class MyClass
{
private:
    std::string label;
    int *data;
    int size;
    
public:
    // ...
    MyClass(MyClass &&rhs)
        : label(std::move(rhs.label)    // important that string was moved, since rhs.label is lvalue
                                        // ... and it should be converted to rvalue
    {
        std::swap(data, rhs.data);
        std::swap(size, rhs.size);
    }
    
    MyClass &operator=(MyClass &rhs)
    {
        label = std::move(rhs.label);   // important that string was moved, since rhs.label is lvalue
                                        // ... and it should be converted to rvalue
        std::swap(data, rhs.data);
        std::swap(size, rhs.size);
};
```

It is important to note that passing a const lvalue to the `std::move` function will convert it to a const rvalue, which cannot be moved, since the only const lvalue reference can bind to a const rvalue, which results in copy constructor getting called.

The `std::move` function is just a cast which accepts a forwarding reference (lvalue ref or rvalue ref) and casts it to a rvalue reference, resulting in a rvalue, which is then used in a move operation. A function that returns a rvalue reference (as `std::move` does) results in a rvalue expression.

Keep in mind that generally move constructors and move assignment operators do not have to be written out, unless any one of the 5 special functions are declared. Read more about the Rule of Zero, Rule of Five and Rule of All or Nothing in this [article](https://www.fluentcpp.com/2019/04/23/the-rule-of-zero-zero-constructor-zero-calorie/).

<a name="forwarding-references"></a>
## 4. Forwarding references

Forwarding references are references that can bind to anything, either a lvalue or rvalue. It becomes a lvalue reference when bound to lvalues and rvalue reference when bound to  rvalues.

```
int var = 10;
auto &&ref = var;            // ref is a forwarding reference, which in this case becomes an lvalue reference

template<typename T>
void foo(T &&param);         // param is a forwarding reference, can either become a lvalue ref or rvalue ref

template<class ...Args>
void print(Args &&...params);  // params is a forwarding reference
```

Forwarding references come in only 2 forms, as shown in the previous example. They are either `auto &&` or `T &&`, and `T` can be any name for the template. Even `const T &&` is not a forwarding reference, it is an rvalue reference. Basically forwarding references come about only when the type is deduced. In the given example, `foo` can accept both lvalues and rvalues, and the type of param will become either an lvalue reference or an rvalue reference, depending on what was given.

Here are some examples of when what seems like a forwarding reference is actually a rvalue reference.

```
template<typename T>
void f(std::vector<T> &&param);     // param is of type rvalue reference

template<typename T>
void f(const T &&param);            // param is of type rvalue reference

template<typename T>
class MyClass
{
    void foo(T &&param);            // param is of type rvalue reference
};
```

In the last case, where a class is used, `T &&` is not a forwarding reference and instead a rvalue reference. The way this works is that `T` is already known when for when the function gets used, therefore no type deduction is taking place.

### Reference collapsing

There is a problem with references - there are no references to references. Consider the following example

```
int& &var = 10;      // ERROR: no reference to reference
int& &&var = 10;     // |
int&& &var = 10;     // |
int&& &&var = 10;    // |
```

All of the above examples give an error, since there are no references to references in C++.

```
template<typename T>
void foo(T &&param);
```

In the case of forwarding references, as shown in the above example, if a rvalue of type `int` is passed to the reference, `T` is deduced as `int`, which makes the result type `int &&`. Meanwhile, if a lvalue of type `int` is passed to the reference, `T` is deduced as `int &`, which makes the result type `int & &&`. At first glance, this might seem illegal, however we know that forwarding references can accept lvalues, and it works due to reference collapsing.

The reference collapsing rules are as follows:
* if there is a rvalue reference to rvalue reference, it is resolved to rvalue reference (`int && &&` -> `int &&`)
* otherwise it is resolved to lvalue reference (`int & &`, `int & &&`, `int && &` -> `int &`)

```
int var = 10;
foo(10);         // parameter deduced as int &&
foo(var);        // parameter deduced as int & (collapsed from int & &&)
```

In the case that references are given as the parameter to the function `foo`, the reference part is stripped out and since the expression itself is an lvalue, the type is deduced as `int &`.

```
int &ref = var;
int &&rref = var;
foo(ref);        // parameter deduced as int & (collapsed from int & &&)
foo(rref);       // parameter deduced as int & (collapsed from int & &&)
```

Using `auto` for type deduction is essentially the same as type deduction in templates, therefore the same rules of reference collapsing apply to `auto &&`.

After learning about forwarding references, it is important to note that forwarding references are essentially rvalue references with reference collapsing together. Forwarding references are only concepts, and what really is happening is that rvalue references are getting collapsed, resulting in either rvalue reference or lvalue reference, thus giving the ability to bind to rvalues and lvalues.

<a name="perfect-forwarding"></a>
## 5. Perfect forwarding

Now that we know about forwarding reference, we can continue with perfect forwarding. Let's take a look at this code sample

```
template<typename T>
class MyClass
{
private
    std::vector<T> list;

public:
    template<typename T2>
    void add(T2 &&param);
};
```

We know what param can have either lvalue reference or rvalue reference as its type, but param itself will always be an lvalue. In case it's type is a rvalue reference, which means that a rvalue was passed into the function when called, we would like to treat param as a rvalue (for example to for moving), meanwhile lvalues shouldn't be changed, that is they should remain as lvalues.

```
template<typename T2>
void MyClass::add(T2 &&param)
{
    list.push_back(param);              // param is always treated as lvalue, so copy ocurrs,
                                        // ... even if rvalue was passed to function
    // OR
    list.push_back(std::move(param));   // param (after move) is always treated as rvalue,
                                        // ... lvalue that was passed will become unusable
}
```

To solve this issue where we want to keep the rvalue (or lvalue), `std::forward` can be used. It will preserve the value that was passed into the function `add` and `param` becomes that value.

```
template<typename T2>
void MyClass::add(T2 &&param)
{
    list.push_back(std::forward<T2>(param));    // correctly casts param to rvalue or lvalue
}
```

`std::forward` conditionally casts `param` to either an lvalue or rvalue, based on `T2`. If param was initialized with lvalue, then `T2` will be of type `T &`, and if param was initialized as rvalue, then `T2` will be of type `T`. Based on this `std::forward` knows when to cast to a rvalue reference, and when to cast to lvalue reference.

<a name="rare-use-cases"></a>
## 6. Rare use cases

### rvalue references as variables

Usually rvalue reference variables shouldn't be used, however they can be used to bind a temporary object which is of non-copyable and non-movable type, as shown in the following example

```
class MyClass 
{
    MyClass(const obj &) = delete; 
    MyClass(obj &&) = delete;
}; 

MyClass foo() { 
    return {};  // returns default constructed temporary object
} 

int main() {
    MyClass&& var = foo();
    return 0;
}
```

In other cases simply use ```MyClass var = foo();``` since it allows for RVO activation.

<a name="return-rvalue-reference"></a>
### Functions returning rvalue references

```
MyClass &&foo1();          // by rvalue reference, if the object should be moved out 
                           // ... (was temporary and is not used anymore, expression is a rvalue)
const MyClass &&foo2();    // by const rvalue reference, pretty much useless, however due to bug code can break 
                           // ... (was temporary and is not used anymore, expression is a rvalue)
```

Take care with functions that return rvalue references, such functions should not be called from classes that are not rvalues (when instanced), since moving a member out and then using that member later is considered bad code. Such functions have only a few use cases that ocurr rarely in general code. 
You can read this [article](https://www.foonathan.net/2018/03/rvalue-references-api-guidelines/) for more details.

### Reference qualified functions

```
class MyClass
{
    void foo() &;
    void foo() &&;
};
```

Functions can have reference qualifiers, and be overloaded on them. The function that is called will depend on what the object value was, if the object is lvalue, lvalue reference qualifed function is called, and the same for rvalue.

```
MyClass obj;
obj.foo();          // lvalue ref qualified function called
MyClass().foo();    // rvalue ref qualified function called
```

<a name="references"></a>
## References

* **https://www.fluentcpp.com/2018/02/06/understanding-lvalues-rvalues-and-their-references/**
* **https://www.fluentcpp.com/2019/04/23/the-rule-of-zero-zero-constructor-zero-calorie/**
* **https://eli.thegreenplace.net/2011/12/15/understanding-lvalues-and-rvalues-in-c-and-c/**
* **https://www.foonathan.net/2018/03/rvalue-references-api-guidelines/**
* **https://stackoverflow.com/questions/19593552/c11-whats-the-use-of-a-rvalue-reference-initialization**
* **https://eli.thegreenplace.net/2011/12/15/understanding-lvalues-and-rvalues-in-c-and-c/**
* **https://drewcampbell92.medium.com/understanding-move-semantics-and-perfect-forwarding-987cf4dc7e27**
* **http://www.vishalchovatiya.com/21-new-features-of-modern-cpp-to-use-in-your-project/#Uniform_initialization_Non-static_member_initialization**
* **https://isocpp.org/blog/2012/11/universal-references-in-c11-scott-meyers**
* **Effective Modern C++ by Scott Meyers (O’Reilly). Copyright 2015 Scott Meyers, 978-1-491-90399-5**
