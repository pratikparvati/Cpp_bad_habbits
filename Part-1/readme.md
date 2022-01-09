#C++ bad habits - How to make mistakes! 

When you study programming language, learn to make mistake. Learn to program means to understand what you should not do and how you recognize a mistake and how to debug it. Writing a program is the easy part; learning to program means to find a bug if a program seems to be correct but fails from time to time in an arbitrary way. Then you need to understand what mistake leads into this behavior and how to debug this bug. Therefore you should know as much mistakes as possible.

> **_NOTE:_** You should not avoid mistakes. You should do them all and learn from them.

So the first mistake you should avoid is to avoid mistakes while being a C++ beginner in order to be able to avoid mistakes as an advanced C++ developer.

## Do not mix overloading with overriding

In C++, two functions can have the same name if the number and/or type of arguments passed is different. These functions having the same name but different arguments are known as overloaded function. Overriding means same method name and same parameter occur in different class that has inheritance relationship. Before we get to the problem, lets understand scoping rule in C++.

#### Scoping 

The basic to complex scoping rules are explained at cppreference site [here](https://en.cppreference.com/w/cpp/language/scope). The inner scope declarations hide the declarations in outer scope regardless of the type of language elements(object, type, function etc).

Similarly, a class name can be hidden by an explicit declaration of the same name - as an object, function or enumeration in a nested declarative region or derived class. The class name is hidden whenever the object, function or enumeration name is visible. This process is referred to as [name hiding](https://isocpp.org/wiki/faq/strange-inheritance#hiding-rule).

For instance:

```C++
#include <iostream>
#include <typeinfo>

struct Base
{
  char x;
};

struct Derived: Base
{
  int x;  // char x is hidden as per name hiding rule
};

int main()
{
  Derived d;
  std::cout << typeid(d.x).name() << std::endl; // prints "int"
}
```
Since the members of `class Base` are visible in `class Derived`, a declaration of an object named x in  `class Derived` will hide `char x` in `class Base`.

##### why name hiding was actually designed into C++?

You probably know that in C++ overload resolution works by choosing the best function from the set of candidates; this is done by matching the types of arguments to the types of parameters. Hence, adding new functions to a set of previously existing ones might result in a rather drastic shift in overload resolution results.

For instance:

```C++
#include <iostream>
#include <typeinfo>

struct Base
{
  void foo(char f) { std::cout << "Base::foo(char)\n"; }
};

struct Derived : Base
{
  void foo(int f) { std::cout << "Derived::foo(int)\n"; }
  void bar() { foo('c'); }
};

int main()
{
  Derived d;
  d.bar(); // prints "Derived::foo(int)"
}
```
Although `Base::foo(char)` is a better match for the call `foo('c')` name lookup stops after finding `Derived::foo(int)` and so the program prints `Derived::foo(int)`. If the member functions weren't hidden, that would mean name lookup in class scope behaved differently where the different versions of `foo()` is carried forward over the scope of inherited functions with the same name. This type of behavior was then considered to be undesirable by the C++ standard writers, so they implemented name hiding, which effectively gives each class a "clean slate" or a new, clean class.

#### Why not mix overloading with overriding?

Functions in Derived classes with the same name as the functions in the Base classes, but that do not override the Base class function are considered to be bad practice because it can result in errors.

For instance:

```C++
#include <iostream>
#include <typeinfo>
#include <string>

struct Base
{
  void foo(std::string f) { std::cout << "Base::foo(std::string)\n"; }
};

struct Derived : Base
{
  void foo(int f) { std::cout << "Derived::foo(int)\n"; }
};

int main()
{
  Derived d;
  d.foo("xyz");
}

/* ERROR

Output of x86-64 gcc 11.2 (Compiler #1)x86-64 gcc 11.2 (C++, Editor #1, Compiler #1)
<source>: In function 'int main()':
<source>:18:9: error: invalid conversion from 'const char*' to 'int' [-fpermissive]
   18 |   d.foo("xyz");
      |         ^~~~~
      |         |
      |         const char*

*/
```
The error occurs because `foo(string)` is hidden in Derived class scope. This behaviour makes sense because it prevents ambiguities in the inheritance process. Suppose, we had a function `Base::foo(float)` and a function `Derived::foo(double)`. If `Base::foo(float)` was not hidden by default in `Derived`, we would call the base class function when calling `d.foo(0.f)`, even though a `float` can be promoted to a `double`.

The real fun with these ambiguities would start when a `0` is used instead of a `nullptr` in C++11 — since a function with an integral parameter will always be a better match than a function taking a pointer parameter, this would result in agonizing, hard-to-trace bugs.

## Use "override" for overridden functions

In C++, the virtual methods are introduced with the virtual keyword. However, when creating overrides in derived classes, the keyword `virtual` is optional, which might cause difficulty when working with large classes or hierarchies. To determine whether a function is virtual or not, you may need to traverse through the hierarchy all the way to the base class. On the other hand, sometimes, it is useful to make sure that a virtual function or even a derived class can no longer be overridden or derived further. In this recipe, we will see how to use C++11 special identifiers `override` and `final` to declare virtual functions or classes.

Following below two rules would ensure correct declaration of virtual methods both in derived and base classes and also increase readability.

- Always use the `virtual` keyword when declaring virtual functions in derived classes that are supposed to override virtual functions from a base class, and
- Always use the [`override`](https://en.cppreference.com/w/cpp/language/override) special identifier after the declarator part of a virtual function declaration or definition.

For instance:

```C++
#include <iostream>
#include <typeinfo>
#include <string>

struct Base
{
  virtual void foo(double f) {std::cout << "Base::foo(double)\n";};
};

struct Derived : Base
{
  virtual void foo(int f) { std::cout << "Derived::foo(int)\n"; } // intention here is to override Base::foo(double)
};

int main()
{
  Derived d;
  d.foo(10.0); // prints "Derived::foo(int)"
}
```
Even though the use is intended to call overridden function of `Base::foo(double)`; the function `Derived::foo(int)` (by impilicit type coversion to `int`) is called which is actually a overloaded function that hides `foo(double)` inherited from base class. The compiler, unaware that it is intending to write a previous method, simply adds it to the class as a new method.

Adding `override` clearly disambiguate this: through this, one is telling the compiler that three things are expecting:
- There is a method with the same name in the base class
- This method in the base class is declared as `virtual` (that means, intended to be rewritten)
- The method in the base class has the same signature as the method in the derived class (the rewriting method)

If any of these is false, then an error is signaled.

```C++
#include <iostream>
#include <typeinfo>
#include <string>

struct Base
{
  virtual void foo(double f) {std::cout << "Base::foo(double)\n";};
};

struct Derived : Base
{
  virtual void foo(int f) override { std::cout << "Derived::foo(int)\n"; }
};

int main()
{
  Derived d;
  d.foo(10.0);
}

/*ERROR
<source>:12:16: error: 'virtual void Derived::foo(int)' marked 'override', but does not override
   12 |   virtual void foo(int f) override { std::cout << "Derived::foo(int)\n"; }
      |                ^~~
ASM generation compiler returned: 1
<source>:12:16: error: 'virtual void Derived::foo(int)' marked 'override', but does not override
   12 |   virtual void foo(int f) override { std::cout << "Derived::foo(int)\n"; }
      |                ^~~
Execution build compiler returned: 1
*/
```

To ensure that functions cannot be overridden further or classes cannot be derived any more, use the `final` special identifier:

```C++
#include <iostream>
#include <typeinfo>
#include <string>

struct Base
{
  virtual void foo(double f) { std::cout << "Base::foo(double)\n"; };
};

struct Derived final : Base
{
  virtual void foo(double f) override { std::cout << "Derived::foo(int)\n"; }
};

struct Derived1 : Derived
{
};

int main()
{
  Derived d;
  d.foo(10.0);
}

/*ERROR
<source>:15:8: error: cannot derive from 'final' base 'Derived' in derived type 'Derived1'
   15 | struct Derived1: Derived {};
      |        ^~~~~~~~
ASM generation compiler returned: 1
<source>:15:8: error: cannot derive from 'final' base 'Derived' in derived type 'Derived1'
   15 | struct Derived1: Derived {};
      |        ^~~~~~~~
Execution build compiler returned: 1
*/
```

## Don't specify default value on function overrides

Default arguments are mostly syntactic sugar and get determined at compile time. Virtual dispatch, on the other hand, is a run-time feature. i.e, virtual functions are dynamically bound, but default parameter values are statically bound. Therefore, the default parameter is selected by the compiler using the static type of the object a member function is called upon.

For instance:

```C++
#include <iostream>
#include <typeinfo>
#include <string>

struct Base
{
  virtual void foo(double f = 10) { std::cout << "Base::foo(double) --> " << f << "\n"; };
};

struct Derived final : Base
{
  virtual void foo(double f = 20) { std::cout << "Derived::foo(double) --> " << f << "\n"; }
};

int main()
{
  Base *b = new Derived{};
  b->foo(30); // prints "Derived::foo(double) --> 30" which is fine
  b->foo(); // prints "Derived::foo(double) --> 10" ??? expect to print 20 instead of 10!!!
}
```
Virtual functions are dynamically bound, meaning that the particular function called is determined by the dynamic type of the object through which it's invoked:

```C++
b->foo(30) calls Derived::foo(30)
```

However, when you consider virtual functions with default parameter values; you may end up invoking a virtual function defined in a derived class but using default parameter value from a base class.

```C++
b->foo() calls Derived::foo(10)!!
```
In `Derived::foo()`, the default parameter value is `20`. Since the `b`'s static type is `Base*`, the default parameter value for this function call is taken from the `Base` class, not the `Derived` class! The result is a call consisting of a strange and almost certainly unanticipated combination of the declarations for `foo()` in both the `Base` and `Derived` classes.

Never redefine an inherited default parameter value, because default parameter values are statically bound, while virtual functions the only functions you should be overriding are dynamically bound.

> **_NOTE:_** Parameters in an overriding virtual function shall either use the same default arguments as the function they override, or else shall not specify any default arguments

Always use inheritance and virtual functions carefully, commit mistakes but learn from them. That's all I have in this blog, I will keep writing similar points and my learnings in my next coming blogs.

Thanks for reading,
Pratik