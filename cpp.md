# c++

In which I keep track and categorize non-trivial things I've learned about how c++ works or good practices

### General

- `const`
  * Every variable that isn't modified should be marked `const`
  * `const` is not deep: pointers and references in a `const` object only have `const` addressess, and the objects they refer to can still be modified through them
  * Use `const` for function parameters, but in definitions only (recognized by the compiler as being the same function, doesn't leak implementation detail to the interface):
    ```
    // Declaration in header
    void f(int i);

    // Definition in implementation file
    void f(int const i)
    {
        ...
    }
    ```
  * Some people are a bit confused by the syntax of `const` when it relates to pointers. The general rule is :
    > `const` applies to the thing to its left. If there is nothing to its left, then it applies to the thing to its right.

    Personnally I find it less confusing to always write `const` to the right of what it applies to
- `auto`
  * **Always use auto to declare local variables**
  * Using `auto` prevents mistakes and implicit conversions, and also more generally allows to make the code more robust to implementation changes
  * `int i;` may lead to using `i` uninitialized, but `auto i;` doesn't compile by itself
  * `auto s = myvector.size();` is correct even though `size()` returns an implementation-dependent type (type alias in the template)
  * `auto i = 0u;` is better than `unsigned int i = 0;` (which has the wrong type on the right side)
  * `auto i = unsigned int(0);` is even better, and more generic since you can also write `auto i = uint32_t(0);`
  * `auto myVar = static_cast<MyType>(getVal());` makes it clear that a cast is happening when the type you get is not the type you want (ex: proxy classes)
    + **WARNING:**
      - Beware of hidden proxy classes. They're not meant to be anything other than temporaries
      - More generally, you need to make sure of the return type of the functions you are calling

- Implicit conversions between integers
  * Integral promotion
    + Integer types have "ranks" based on their width (*e.g.* `int` ranks higher than `short` and lower than `long`)
    + Arithmetic operations do not accept bools, enums, etc, or types smaller than `int` as argument
    + Passing values of such types as arguments automatically casts them to a higher-rank integer type that can hold all the values of that type (signed if both signed and unsigned work)
    + For example, this means `int8_t + int8_t = int` and `uint8_t + uint8_t = int`
    + Note that this impacts both size and signedness (but does not affect the value)
    + Given that `int` can have an arbitary size, this may affect all enums, fixed-width types, etc
  * Arithmetic conversion
    + Arithmetic operations take parameters with the same type on both side
    + After integral promotion, if the operands do not have the same type, another implicit conversion occurs
    + If both operands have the same signedness, things work well and the smaller type is cast into the bigger type
    + Otherwise conversion rules apply that always preserve the unsigned value but often (though not always) convert the signed one to unsigned, which usually leads to unwanted behavior
    + For example, `5u > -1` evaluates to `false`
  * Conclusion
    + Always make sure your integers have the correct signedness (including literals)
    + Never compare signed and unsigned
    + If the unsigned type you're using isn't `unsigned int`, `unsigned long` or `unsigned long long`, handle the case in which the result of the operation is signed (*e.g.* `uint8_t + uint8_t = int` as mentioned above)

- `new`/`delete`
  * Calling `delete` on a null pointer is allowed (does nothing), so no need to check that it's not null beforehand
  * operator `new`
    + refers to `void* operator new(size_t size)`
    + It's the c++ equivalent to `malloc`
    + Allocates memory and returns a pointer to it
    + Deallocation is done by "operator `delete`", i.e. by `void operator delete(void* memory)`
    + Managing memory can be done by overloading operator `new` and operator `delete`
  * "the `new` operator" (not the same thing)
    + Called through `new` statements, *i.e.* `A* a = new A;`
    + Calls operator `new` based on `A`'s size, uses `A`'s constructor to build the object, and casts the pointer to the correct type
      > Note: Calls operator `delete` if the construction throws an exception so no memory is leaked

    + "the `delete` operator" similarly allows to perform the reverse opertion through `delete` statements (`delete a;`), which calls the object's destructor then operator `delete`
  * "the `new[]` operator"
    + `A* a = new A[10];`
    + Same as above but builds several objects
    + The reverse operation is `delete[] a;`, using the `delete[]` operator
       > Note: Remember to always use delete[] if new[] was used

  * Placement `new` (part of the standard library)
    + `void* operator new(size_t, void* location)`
    + Same as operator new but takes an address as argument instead of allocating the memory through operator `new`
    + Corresponding `new` statement: `A* a = new (location) A;`
    + Doesn't allocate memory, but uses the constructor to build the object at the given location
    + The reverse operation is `a->~A();` (not `delete`, since no memory was allocated!)
    + One way to manually manage memory is to use operator `new` to get a big chunk of memory, then to manually build objects with placement `new`

  * If you provide a custom `new` for your type
    + Provide all standard forms as well
      ```
      class MyClass
      {
          ...
          static void* operator new(std::size_t, MyCustomAllocator&);       // Called through T* p = new(alloc) T;
          ...
          static void* operator new(std::size_t);                           // Plain new
          static void* operator new(std::size_t, std::nothrow_t) noexcept;  // Nothrow new
          static void* operator new(std::size_t, void*);                    // Placement new
          ...
      };
      ```
    + Provide a matching `delete` for each `new`
      ```
      static void* operator delete(void*, std::size_t, MyCustomAllocator&) noexcept;
      static void* operator delete(void*, std::size_t);
      static void* operator delete(void*, std::size_t, std::nothrow_t) noexcept;
      static void* operator delete(void*, std::size_t, void*) noexcept;
      ```
    + To provide the functions, you can write `using Base::operator new;` in a derived class, otherwise you need to explicitly call the matching function from the global namespace with `::new(...)`
  * Same thing for `new[]`

- `using`
  * Type alias: `using A = B;`
    + Can be templated, unlike `typedef`
  * `using` declaration: `using A::f;` is a
    + Makes the function `A::f` visible in the current namespace as if it was declared there
    + Very useful in implementation files to make things more readable
    + Must not be used in the global scope in header files except if you explicitly want to make things visible
    + Can also be used within a derived class scope to introduce members of the base class and avoid hiding overloads (see related entry)
  * `using` directive: `using namespace A;`
    + Like a `using` declaration but on everything from a given namespace
    + Usually a bad idea because there may be things in that namespace that you don't know are there and have the same name as others

- Local static objects
  * Refers to variable declared at block scope, with the `static` keyword
  * Thread-safe since C++11
  * Constructor called only by the first thread, while others wait
  * Local static objects are destroyed at the end of the execution in the reverse order in which they were constructed

- Non-local static objects
  * Refers to global variables, static member variables of a class, etc
  * They are initialized is an undefined order, which can be bad if one depends on another
  * Replace their use by function calls that return references or `const`references to local static objects, so that each is initalized when the function is first called and relative initialization order can be enforced

- Evaluation order
  * `foo(shared_ptr<A>(new A), bar());`
  * Since c++17 each argument is fully evaluated before evaluating another
  * Before c++17, evaluations could happen in any order, including `new A` -> `bar()` -> `shared_ptr<A>()`, thus leaking resources if `bar()` threw an exception. Avoiding that issue required to use single statements when putting resources in managers, or to use a function like `make_shared()` to do it for you

- `return {b, c};`
  * Same as `return A{b, c};` in a function returning an object of type `A`

- `auto f() -> A;`
  * Since C++11, valid syntax to declare functions
  * Can be particularly useful with templates, to make full use of `decltype` :
    ```c++
    template<T>
    auto f(T t) -> decltype(g(t));
    ```
    (which can't be done with the other syntax)
  * Since C++14, the `-> A` can even be omitted when
    the body of the function is available to deduce the return type
    + Can have trouble with recursion
  * If `decltype(t)` is `T`, then `decltype((t))` is `T&`, so be careful not to write `auto f() { A a; return (a); }`

- `switch`
  * Always have a `default` to be sure to cover each path
  * If it's supposed to be unreachable, write it explicitly with `std::unreachable()` (since C++23, otherwise create a custom `unreachable()` calling `std::terminate()`)

- `goto`
  * Usually considered a bad practice
  * There are certain cases in which I don't have better alternatives at the moment:
    + Breaking from multiple nested loops (instead of one `if (...) { break; }` per loop)
    + Replace `break` in a `switch` (more granular than falling through)
    + More generally factoring code when the control structure is complex

- `dynamic_cast`
  * Relies on the vtable
  * Therefore not usable if there is no virtual function

- `const_cast`
  * Basically the only case in which it can be legitimately used is to pass `const` variables to legacy APIs which should have been tagged `const` but weren't
  * Modifying a `const` string literal would be bad, because string literals are allowed to overlap in memory
  * More generally, modifying anything which is created (*i.e.* declared) `const` is undefined behavior

- Sensitive data
  * If sensitive data is stored in dynamically allocated memory, then that memory should be set to 0 before being deallocated (including when a `vector` is reallocated to increase its capacity, etc)
  * Similarly, static memory holding sensitive data must also be cleared before the program exits
  * Make sure the compiler doesn't optimize away those calls


### Advanced concepts

- Most vexing parse
  * `B myObject(A());`
    + Might be intended to build `B myObject`, passing it as argument a default-constructed `A`
    + Is parsed as if it was `B myObject(A myFunction());`
    + That's the declaration of a function called `myObject` which takes as argument a pointer to a function `myFunction` (which takes no argument and returns an `A`) and returns a `B`
  * `MyClass myObject();`
     + Will also be parsed as a function declaration
  * Since c++11, this can be avoided through `B myObject(A{})` and `MyClass myObject{};`
  * Before c++11, avoiding that issue was done through `B myObject((A()))` (a formal parameter declaration cannot be placed within parentheses) and `MyClass myObject;`

- Koenig Lookup = Argument-Dependent Lookup (ADL)
  * `f();`
    + Is an unqualified call (which `f` is that?)
    + The compiler looks for a function named `f` in the immediate scope (i.e. the one in which the function is used)
    + If it doesn't find one, it goes "outwards" (i.e. the scope in which the previous scope was declared, possibly the global scope), and continues outwards until it either finds at least one function with that name or runs out of scopes
    + If it finds a function with that name in a scope, then it stops looking further
    + If the signature or access rights (private/public) of the candidate functions don't match, the code won't compile, even if an outer scope has the correct fonction
    + That's why overriding (or shadowing) a function in a derived class hides the other overloads
      - A `using` declaration (`using Base::f;` within the derived class scope) can be used to make them visible
      - Without it, a qualified call is necessary: `myObject.Base::f();`
  * General case
    + The compiler looks at the immediate scope, then goes outwards
    +  In parallel to that, if there are arguments passed to the function (including the `myObject` when calling `myObject.f()`), it also looks at the scopes containing the types of those arguments (hence the name "Argument-Dependent Lookup") and goes outwards as well
    + This means that
      ```c++
      namespace A {
          struct X { ... };
          void f(X) { ... }
      }

      namespace B {
          void f(X) { ... }
          A::X x;
          f(x);
      }
      ```
      is ambiguous and won't compile, because `f` is found in the scope of the call (`B`) and in the scope containing the argument type (`A`)
    + Adding something to a library's namespace can therefore make lots of code using that library ambiguous and prevent it from compiling
    + Similarly
      ```c++
      namespace N {
          class C { ... };
      }

      int operator+(int, N::C) {...}

      N::C c;
      int res = 1+c; // This is an unqualified call!
      ```
      won't necessarily compile, depending on what else is in the namespace `N` (namely, if any `operator+()` is declared, even if its signature doesn't match the argument types)

- Forwarding, forwarding references
  * How do we make a wrapper `f()` that seemlessly passes its arguments to a function `g()` ("perfect forwarding")?
    + `template <typename T> void f(T t){g(t);}` takes things by value, which induces a copy
    + `template <typename T> void f(T const& t){g(t);}` can only handle `g()`'s `const` arguments
    + that also leaves out rvalues
  * `template <typename T> void f(T&& t)` is called a forwarding reference (or universal reference)
    + Reference collapsing rules:
      - && + && = &&
      - & + && = &
      - && + & = &
      - & + & = &
    + Therefore
      - `T` = `U` => `T&&` = `U&&`
      - `T` = `U const&` => `T&&` = `U const&`
      - `T` = `U&` => `T&&` = `U&`
      - `T` = `U&&` => `T&&` = `U&&`
    + A forwarding reference therefore defines the function for the 3 relevant overloads
  * `auto&&` is also a forwarding reference
  * If we pass an rvalue to `f()`, we want it to pass an rvalue to `g()`
    + The rvalue reference (`U&&`) is itself an lvalue
    + Where we would usually call `g(t)`, specifically for rvalue references we need to call `g(std::move(t));`
    + `std::forward()` is a templated function in the standard library allowing a conditional `std::move()`
    + We can therefore perform perfect forwarding like this:
      ```c++
      template <typename T>
      f(T&& t)
      {
          g(std::forward<T>(t));
      }
      ```
  * Since c++14 `auto&&` can be used as a type parameter for lambdas, which makes it really convenient for forwarding (with `std::forward<decltype(param)>(param)`)
  * Since C++14, the syntax
    ```c++
    decltype(auto) f() {return getVal();}
    ```
    allows also allows to return `auto`, `auto&` or `auto&&` (exact same type returned by `getVal()`)
  * Overloads
    + Don't try to overload when there's perfect forwarding
      * `template <typename T> f(T&& var)` will catch everything, so an `f(int)` overload would require that exact type and passing it a short would not call the overload
      * There'll also be trouble if the function is actually a constructor, because the template's existence won't prevent the automatic definition of the implicit copy/move constructors
    + Use tag dispatch instead
      * Have `f()` inline-call `f_impl(var, std::is_integral_type<std::remove_reference_t<T>>)`
      * `template <typename T> f_impl(T&& var, std::true_type)` and `template <typename T> f_impl(T&& var, std::false_type)` are the actual overloads
    +  Another solution is to use `std::enable_if` to constrain the template


### Classes

- `explicit`
  * Don't forget to use it by default on single-argument constructors to prevent them from becoming conversion constructor (allowing implicit conversion)
  * May be used when implementing conversion operators (`explicit operatorMyType() const`) to allow `static_cast` but not to implicitly convert when returning it or passing it as argument
    + Doesn't prevent implicit conversions in other context; for example `std::shared_ptr` has `explicit operator bool() const noexcept` but we can still write `if (myPointer)`

- Inheritance
  * Add the `final` specifier if the class isn't meant to be derived from
  * If a base class isn't meant to be instanciated, make its constructor `protected`, then either make the destructor `protected` too to forbid calling `delete` on a base class pointer, or make it public and virtual
  * Keep in mind that when constructing a derived class object, the base class constructor is called first, and same in reverse with the destructor of the base class being called afterwards. Therefore in the base class constructor/destructor
    + The dynamic type is that of the base class even when building a derived object
    + Calling a pure virtual function is therefore undefined behavior
    + Calling a virtual function will not call its derived-class overload
    + To be safe, never call virtual functions from a constructor/destructor
  * When a base class is not made to be instanciated, it should be made uncopyable and unmovable to avoid slicing a derived class in a way that doesn't make sense

- Inheritance hierarchy
* When you already have a concrete class `B` and you need to create a concrete class `C` which is a specialized version of `B`, it's tempting to make `C` inherit from `B`
* Instead, make them both inherit from an abstract class `A`
  + Not very different, `B` can just use the default implementations from `A`'s pure virtual functions
  + Avoids heterogenous assignments (`B b = c;` should not compile)
  + Leaves room to add stuff in `B` not needed in `C`
* Concrete classes should be the leaves of the inheritance hierarchy

- Destructor
  * `virtual`
    + Only necessary if `delete` might be called on a `Base*` pointing to an instance of a derived class
    + Conversely means that you should not do that with a base class that doesn't have a virtual destructor
  * `noexcept`
    + Destructors are automatically called when an exception is thrown, and they should not throw exceptions themselves (otherwise crash or undefined behaviour)
    + All destructors must therefore be `noexcept` (and are tagged as such by default)

- Constructor
  * Derived class constructors
    + It's possible to write `using Base::Base` to import all constructors from a base class into a derived class, with all additional members being default constructed
    + If you don't call the base class's constructor in the derived one's, the base class's default constructor is called, so don't forget to call the base class's copy constructor through `Derived(Derived const& foo) : Base(foo) { ... }` when writing the derived class's
  * Constructor delegation
    + Since C++11, it's possible to write
      ```c++
      MyClass(A a)
        : MyClass(a[0],a[1])
      {
          ...
      }
      ```
      i.e. one constructor leaves the construction to another one
    + It makes it easier to avoid duplicate code and to ensure consistency
    + The object's lifetime starts when the first constructor finishes

- Overriding
  * An override shadows all overloads at once (ex: overriding `f(int)` also hides `f(double)`)
  * Solution:
    ```c++
    class Derived
    {
        ...
        using Base::f;
        virtual void f(int) override;
        ...
    };
    ```
    This imports `f(double)` and overrides `f(int)`

- Non-Virtual Interface (NVI) principle
  * Virtual functions are an implementation detail, so virtuality should be kept out of the public interface
  * Public virtual functions should be replaced with normal functions calling the virtual ones
    ```c++
    class MyClass
    {
    public:
        void f() { fImplementation(); }
    private:
        virtual void fImplementation();
    };
    ```
    > Yes, a derived class can override the base class's private virtual functions, even if it can't call them

  * This also allows to add common code before/after the part that varies, and presents a good place to put default parameter values
    > calling `base->f();` where `base` is a base-class pointer to a derived-class object, will call the derived override of `f` with the default values from the base class... it's better to not use default parameters with virtual functions)

- Virtual inheritance
  * We have a base class `B`, 2 derived classes `D0` and `D1`, and want to write a class `DD` inheriting from both `D0` and `D1`
  * There would normally be 2 copies of `B` within `DD`, and refering to any of `B`'s members (including member functions) would become ambiguous when not fully qualified
  * ```c++
    class D0 : virtual public B { ... };
    class D1 : virtual public B { ... };
    class DD : public D0, public D1 { ... };
    ```
    allows `DD` to have only one copy of `B` shared by `D0` and `D1`
  * This comes at a cost, though: takes space, big overhead, and which class initializes `B` becomes complicated (same for assignments)

- `const`-ness
  * Don't forget to tag all possible class method `const`
  * Return types of can be made `const` too to avoid things like `(a*b)=c;`, but is usualy not worth it since it prevents the value from being used as an rvalue
  * `const` member functions returning a reference to some internals should usually return a `const` reference (*e.g.* `A const& operator[](unsigned int i) const;`)
  * Use `const_cast` to avoid code duplication when you know what you're doing:
    ```c++
    A& operator[](unsigned int i)
    {
        return const_cast<A&>(static_cast<MyClass const&>(*this)[i]);
    }
    ```
    (calls the `const` version of `operator[]` and removes `const`-ness). Since the `const` overload might use functions that are also overloaded over `const`-ness, you really need to make sure this leads to the correct behavior.
  * Calling a `const` member function (or accessing a `const` member) is a read operation, so users will assume you can do it in many threads at once: **`const` implies thread-safe**

- `mutable`
  * Can be used to tag members that can be changed even when the object is `const`, usually by `const` member functions (*e.g.* for lazy access, mutexes that need to be locked/unlocked)
  * Since `const` implies thread-safe, **access to `mutable` members must be thread-safe too**

- Special member functions
  * The functions which are implicitly defined (*i.e.* created by the compiler)
    + the default constructor (if no other constructor)
    + the destructor (if no destructor)
    + the copy constructor (if no move constructor and no move assignment operator)
    + the copy assignment operator (if no move constructor and no move assignment operator)
    + the move constructor (if no destructor, no copy constructor and no copy nor move assignment operator)
    + the move assignment operator (if no destructor, no copy constructor and no copy nor move assignment operator)
  * Can be explicitly defaulted or deleted
    + Since c++11 through `= default`/`= delete`
    + Before c++11 through private declarations without definitions
    + **Always use `= default`/`= delete` for those not explicitly declared** to make it easier to understand which are or aren't available. It prevents breaking things by mistake (in particular, template classes have looser rules, so it's even more important)
  * Think about removing the copy ones and/or the move ones if the class is meant to be non-copyable and/or non-movable
  * "Rule of 3"/"Rule of 5"
    + Besides the default constructor, if you need to explicitly write one of the special functions, you most likely need to explicitly write all of them
    + Before c++11: a nothrow `swap` function should also be implemented so that copy-and-swap can be used to provide strong exception guarantee
    + Since c++11: make sure the move constructor and assignment operator are nothrow, for the same reason (`std::swap` relies on them)

- `= delete;`
  * Can be used for more than special member functions
    + With overloads: prevents implicit conversion
    + With template specialization: forbids some types

- Proxy classes
  * Returning something (and in particular a reference to a member), means losing control over that something
  * Wrapping it in a class can give us control over what's done with it (*e.g.* making `a[i] = b;` valid or not, maybe with a custom `operator=()`). We call that a proxy class
  * How to make it invisible to the user
    + Implicit conversion operator to the type the client believes `a[i]` to be
    + All member functions that should normally able to be called with that type (including operators, especially `operator&()`) should also work with the proxy class (usually can't be done automatically when templating)
  * Even then, not perfect because implicit cast doesn't work
    + When non-`const` references are needed
    + When an implicit cast from the resulting type is needed


### Templates

- In a scope with `template <typename T>`
  * `typename T::U` needed for `U` to be recognized as a nested type
  * `t.template f(someParam)` needed for `f` to be recognized as a template function (whose template parameters will need to be deduced from `someParam`)

- `this->baseClassMethod();` needed to call a base class method when inheriting from a template class (the compiler doesn't really know where the function is from before instanciation)

- When a template class is instanciated
  * everything in it gets instanciated too
  * functions that don't actually use the templating should be kept outside the class to avoid ending up with 3+ copies in the binary

- Expression templates
  * Templates representing an expression, such as `M = M0 + M1 + M2;` (sum of vectors or matrices)
  * Usually, each addition would involve a loop over all elements, but using expression templates it's actually possible to achieve loop fusion, *i.e* have the compiler perform only one loop of `M[i] = M0[i] + M1[i] + M2[i];`

- Static polymorphism
  * Done through Curiously Recurring Template Pattern (CRTP)
  * ```c++
    template <typename T>
    MyClassInterface
    {
        void f()
        {
            static_cast<T*>(this)->fImplementation();
        }
    };

    MyClass : public MyClassInterface<MyClass>
    {
        friend MyClassInterface<MyClass>;
    private:
        void fImplementation() { ... }
    };
    ```
  * No polymorphism, no vtable, no runtime overhead, just an interface
  * A default implementation
    + Can be given in `MyClassInterface` as a protected function that `MyClass` will have to explicitly call (to do the same with dynamic polymorphism, the trick is to provide an implementation for a pure virtual function, which will then have to be explicitly called through `Base::myFunction` by the derived class)
    + Or through shadowing (`fImplementation()` in the base class hidden by a function of the same name in the derived class, which is the static equivalent of an override)

- `extern` (since C++11)
   * `extern template class std::vector<MyClass>;` tells the compiler that the template is instanciated in another compilation unit and that it'll have access to it when linking
   * Allows to reduce compilation time, but there needs to be an instanciation in one of the compilation unit

- Templated variables (since C++14)
  * This can now be written
    ```c++
    template<typename T>
    constexpr T pi = T(3.14159265358979323846);
    ```
  * Which allows for things like this:
    ```c++
    template<class T>
    T area_of_circle_from_radius(T r)
    {
        return pi<T> * r * r;
    }
    ```
  * Or as a lambda, this:
    ```c++
    auto area_of_circle_from_radius = [](auto r)
    {
        return pi<decltype(r)> * r * r;
    };
    ```

- Variadic templates (since C++11)
  * It is possible to use a template parameter pack, i.e. an unspecified number of template arguments, if necessary with the matching function parameter pack:
    ```c++
    template<typename... ParamTypes>
    void f(A a, ParamTypes... params, B b);
    ```
  * `sizeof...(ParamTypes)` returns the number of template parameters in the pack
  * The main way to use this is with recursion
    ```c++
    template<typename T>
    constexpr T adder(T v)
    {
        return v;
    }

    template<typename T, typename... Args>
    constexpr T adder(T first, Args... args)
    {
        return first + adder(args...);
    }

    int sum = adder(1, 2, 3, 8, 7);
    ```
  * This is what allowed the implementation of `std::tuple`
  * Pack expansion:
    ```c++
    template<typename... ParamTypes>
    void f(ParamTypes... params)
    {
        g(h<ParamTypes>(params)...);
    }
    ```
    here `...` tells the compiler to expand the expression `h<ParamTypes>(params)` into comma-separated values: `h<T0>(p0), h<T1>(p1), h<T2>(p2), etc` (can only be used as function parameters because of the commas)
  * `std::integer_sequence` (since C++14)
    + Can be used to create compile-time arrays of integers (ex: `std::integer_sequence<int, 9, 2, 5, 1>{}`)
    + In particular, `std::index_sequence` is an alias when the type is `size_t`
    + There are some helper
    + `std::index_sequence_for` can be used to attribute indexes to parameters in a parameter pack


### Operators

- Binary operators can be declared either as member functions or friend global functions
  * Declaring both is forbidden (ill-formed)
  * The global version is better because it allows both operands to be casted into the correct types (instead of only one, which isn't symmetrical)
    > Note: casting has an overhead, so use overloads when efficiency is needed

- Prefix vs postfix
  * `++i;` calls the prefix operator: `MyClass& operator++();`
  * `i++;` calls the postfix operator: `MyClass& operator++(int);`
    > Note: compiler warnings about unused parameters can be avoided by not naming said parameters

- `operator=()`
  * Make sure it always handle `a = a;`
  * It's an edge case that can easily happen when references are involved
  * More generally, always assume a function which takes several objects of the same type might be called with the same object twice or more


### Exception safety

- Guarantees
  * Basic guarantee
    + If an exception is thrown, everything remains in a valid state (no corruption, everything internally consistent)
  * Strong guarantee
    + If an exception is thrown, the state of the program is unchanged (none of the arguments should be modified, including the object on which it's called if it's a class method, as if the function was never called)
    + Put everything that can throw at the beginning of the code, and everything that can modify the data under that
    + Sometimes possible without a cost in efficiency, sometimes not. By default, only write code that gives the strong guarantee when it's possible to do it at no cost)
    + Can always be achieved by copy & swap... but the user can do it themselves so it does not need to be provided
  * Nothrow guarantee
    + The function always does what it's supposed to do, and never throws an exception
    + Most often it's not possible, since even allocating memory or locking a mutex can throw

- `noexcept` (since c++11)
  * Can be used to signal that a function gives the nothrow guarantee
  * Usually also implies that the function has no precondition, otherwise throwing a "precondition was violated" exception might be needed in the future. Mostly used for functions that cannot or must not fail
  * Not enforced by the compiler, it's the programmer's responsibility
  * It's part of a function's interface, can't be removed lightly
  * Destructors are `noexcept` by default, and so are implicitly defined special member functions (except if the same special functions from base classes/members aren't)
  * Needs to be used as much as possible (which is not often) to allow the compiler to forego keeping the stack unwindable and optimize. It also allows to use `std::move` as much as possible through `std::move_if_noexcept()`
  * `void f() noexcept;` is actually a shorthand for `void f() noexcept(true);`, while `noexcept(false)` explicitly does not give the nothrow guarantee. The flag can therefore be set through template metaprogramming

- What should never throw (and be marked noexcept)
  * destructors
  * `swap()`
  * move construction/assignment (used for `swap()`, but can allow to give strong guarantee without copy & swap)

- Function-try-block
  * Looks like this
    ```c++
    void f()
    try
    {
        // Do stuff
    }
    catch(...)
    {
        // Do stuff
    }
    ```
  * Catches exceptions throws anywhere within the function
  * This is the only way for a constructor to also catch exceptions thrown by its members' constructors:
    ```c++
    MyClass::MyClass(ParamType param)
    try : _param(param)
    {
        // Do stuff
    }
    catch (...)
    {
        // Do stuff
    }
    ```
  * However, it does not allow to differentiate between those constructors

- Exceptions within a constructor
  * `try`/`catch` often necessary because **the destructor won't be called**: the lifetime of an object starts when the constructor finishes (first constructor if one calls another)

- Cases in which a catch-all `catch(...)` is necessary
  * Around a callback function (to handle the user passing a function that throws)
  * In a function passed to `std::thread`
  * Inside destructors as mentioned above



### Literals

- Basics
  * Integers
    + `1` is an `int`
    + Add suffix `u` for `unsigned`
    + Add suffix `l`/`L` for `long`, twice for `long long`
    + Add prefix `0` to parse as octal
    + Add prefix `0x`/`0X` to parse as hexadecimal
    + Add prefix `0b` to parse as binary (since C++14)
    + Digits can be freely separated by `'`
  * Floating-point numbers
    + `1.` is a `double`
    + Add suffix `f` for `float`
  * Strings
    + `'l'` is a `char`
    + `"lorem ipsum"` is a `char[N]` (including termination character)
    + Add prefix `u8` to parse string as UTF-8
    + Prefixes `u`/`U`/`L` exist for UTF-16 / UTF-32 / wchar with bigger underlying types
- User-defined literal operators (since c++11)
  * Should be placed in their own namespace, and imported with `using` declarations
  * Cooked literal operator
    + Parse the literal normally then pass it to a custom function (called at runtime except if it is `constexpr`)
    +  The suffix name must start with `_`
    + Syntax:
      ```c++
      constexpr A operator"" _mysuffix(long long int value)
      {
          A a = do_stuff(value);
          return a;
      }

      constexpr B operator"" _mysuffix(long double value)
      {
          B b = do_stuff(value);
          return b;
      }

      constexpr C operator"" _mysuffix(char const* str, size_t len)
        {  
          C c = do_stuff(str, len);  
          return c;  
      }
      ```
    + The space between `""` and `_mysuffix` is essential to avoid it being interpreted as an empty literal
    + Integers are always positive, negative ones require overloading `operator-()`
  * Raw literal operator:
    + Replaces normal parsing of the integer or double literal
    + The suffix name must start also with `_`
    + Syntax:
      ```c++
      constexpr A operator"" _mysuffix(const char* str)
      {
          A a = do_stuff(str);
          return a;
      }

      template <char... STR>
      constexpr A operator"" _mysuffix()
      {
          return doStuff(STR...);
      }
      ```
    + With the variadic template form, all chars are template arguments, no end of string character
      ```c++
      a = 0.1_a;
      b = 11011_b;
      c = 0x01AD_c;
      // is equivalent to
      a = operator"" _a<'0', '.', '1'>();
      b = operator"" _b<'1', '1', '0', '1', '1'>();
      c = operator"" _c<'0', 'x', '0', '1', 'A', 'D'>();
      ```


### Standard library

- Standard suffixes (since C++14)
  * `s` is a suffix of `std::string` (ex: `"lorem ipsum"s`)
  * `h`, `min`, `s`, `ms`, `us`, `ns` are suffixes for `std::chrono::duration` (ex: `1s`, `1ms`)
  * `if`, `i`, `il` for `std::complex<float/double/long double>`

- `std::endl`
  * Adds `\n` then flushes
  * Flushing has a cost, hence why there's buffering
  * Prefer `\n` when the timing of the printing doesn't matter

- `begin()`, `end()`
  * `std::begin(container)` returns `container.begin()`
  * Also works for arrays
  * If you write `MyClass container; auto it = begin(container);` and `mynamespace::MyClass::begin` doesn't exist but there is a function `mynamespace::begin(MyClass&)`, ADL guarantees that it will be found
  * In a templated context, the generic way to get the iterator is
  ```
  using std::begin;
  auto it = begin(container);
  ```
  (the `using` declaration means `std::begin()` can be found through Koenig lookup, but the compiler will use the best match and use the user-defined overload if there is one rather than instanciating the template)
  * Same thing with `end()`

- `swap()`
  * `std::swap(a,b);` swaps `a` and `b`
  * May be overloaded for more efficiency, which is similar to the `std::begin()` case above. In a templated context, the generic way to swap things is
  ```
  using std::swap;
  swap(a,b);
  ```
  * Before c++11, it's good practice to implement a `namespace::swap()` overload
    + Calling a `swap()` member function rather than being a friend of the class (for access to internals)
    + Since c++11, `std::swap()` relies on move-constructors, so overloading the function makes less sense.

- Smart pointers
  * The basic idea is RAII
    + Resource/Responsibility Acquisition Is Initialization: here the memory allocation corresponds to an initialization, and the deallocation is done by the destructor
    + The smart-pointer classes can usually be passed a deleter which is called instead of `delete`, which allows to use the classes' functionality (non-copyability or reference counting) for other RAII stuff than just memory
  * **Don't focus on allocation/deallocation, it's about ownership**, not about how the pointer is stored
    + If a function needs access to a resource managed by a smart pointer but won't delete it or anything, then it should not take a smart pointer as argument but just a raw pointer or a reference
    + If there is a need to maintain that access over time:
      - If the resource is guaranteed to outlive the code accessing it: use a raw pointer or a reference
      - If the resource may be released at any time: use an `std::weak_ptr`
      - If the resource's lifetime must be extended to provide that guarantee: use an `std::shared_ptr`
  * `std::unique_ptr`
    + Usually doesn't cost more memory/process than raw pointers (except if they store a function pointer or if the lambda used as deleter captures something)
    + By default, if you need to allocate memory, put it in a `std::unique_ptr`
    + Returning a `std::unique_ptr` from a factory function is a good way to tell the user that he's given both a resource and ownership of that resource (and if needed it can be efficiently converted to a `std::shared_ptr`)
  * `std::shared_ptr`
    + They take more place because they also store a pointer to a reference count, which usually needs to be dynamically allocated, and is atomic for thread-safety (so incrementing/decrementing is rather slow), so use them sparingly
    + Avoid unnecessary copies (moving is faster, for a start)
  * `std::weak_ptr`
    + They're used when there's a need to keep a pointer to something that might not exist at some point, which requires the ability to test whether the pointer is still valid (ex: you may have `vector<shared_ptr<Employee>>` and `vector<shared_ptr<Project>>`, with each employee having a `vector<weak_ptr<Project>>` for the projects they work on, and vice versa)
    + `std::weak_ptr<A> ptr(mySharedPtr);` to create them from a `std::shared_ptr`, and `ptr.lock();` to convert them back to `std::shared_ptr` (a null one if it's expired already)
    + **WARNING:** `std::shared_ptr` and `std::weak_ptr` share the same control block, which is only deallocated when all of them are destroyed (there are separate counts for both types of pointers); since memory for both the object and the control block is usually allocated at one contiguous block when calling `make_shared()`, this can get expensive

- `std::set`, `std::map`, `std::priority_queue`
  * Those are optimized for random insertions, lookups or erasures (one at a time, in a somewhat random order)
  * They are not optimized for batches (ex : big setup -> lots of lookups -> big changes -> lots of lookup -> ...)
    + Using a sorted vector (and then `std::binary_search` or something) can be better
    + In some cases, `std::unordered_set` or `std::unordered_map` can be more appropriate (since they offer average constant time instead of logarithmic)
  * `myMap[key]`
    + Creates an entry and returns a reference to the default-constructed value if there was no corresponding entry
    + This behavior can be useful if the value is a container that needs to be filled or a counter that needs to be incremented
    + It also means that `myMap[key] = val;` calls have an overhead (default-construction then assignment, instead of copy-construction)
      - Use `insert` when the entry is known to be absent
      - Use `operator[]` for updating (more efficient)
      - When it is unknown whether the key is present or not in the map, use `std::lower_bound` or `std::upper_bound` to know where the key should be, compare it to the key, and either update the value or use insert with a hint

- Reverse iterators
  * They usually need to be converted into normal iterators for insert or erase operations
  * **Using `base()` to get the matching iterator doesn't return an iterator pointing to the same element:** `rbegin()` is made to match `end()`, which is outside the container, so there is an offset
    + ex: [1|2|3|4|5]
    + if `rIt` points to 3, then `rIt.base()` points to 4
    + using `rIt.base()` works when inserting (insertion happens before the iterator)
    + when erasing elements, we need an iterator that points to the element to be erased, so `(++rIt).base()` should be used

- Algorithms with an output
  * `std::transform()`, `std::copy`, and many other algorithms, take their input from a range and write their output to a range
  * They take begin/end iterators and don't have access to an underlying container, so they work by assigning things to the output range (**no insertion, no deletion, the size of the container doesn't change**)
  * Always check how an algorithm works before using it
  * Adapters exist to modify what the assignments do
    + `std::move_iterator` can be used to use move semantics for the assignments
    + `std::inserter` (built from the container) can be used to replace assignments by calls to `insert()`
    + `std::back_inserter` (built from the container) can be used to replace assignments by calls to `push_back()`
    + `std::front_inserter` (built from the container) can be used to replace assignments by calls to  `push_front()`; this means the elements are inserted from back to front, so it might be relevant to go through them in reverse order too, with a reverse iterator

- Algorithms which remove things
  * Same as above, they take begin/end iterators and don't have access to an underlying container, so they work by assigning things (**no insertion, no deletion, the size of the container doesn't change**)
  * Always check how an algorithm works before using it
  * In particular, `std::unique()` or `std::remove()` don't actually remove elements from the container
  * What they actually do is go through all values from the range, put the values that need to be "kept" in the first positions of the range, and return an iterator to the "new end"
  * The correct way to use them with a container is for example
    ```
    myVector.erase(std::remove(myVector.begin(), myVector.end(), val), myVector.end());
    ```
  * If the container class provides a dedicated `remove()` or `erase()` method, use that directly
  * **Keep in mind that removing/erasing things impact the validity of iterators**
    + `erase(it)` usually makes `it` invalid, and following with ``++it;`` is undefined behaviour
    + When `erase` returns an iterator, that's the iterator that should be used to continue looping
    + It's important to be careful while looping and erasing stuff

- Algorithms and conditions of use
  * Some algorithm require random access iterators, some only work on a sorted range
  * Always check how an algorithm works before using it

- Algorithms and sorting
  * Sorting requires random-access iterators
  * Sometimes fully sorting all elements with `std::sort()` is overkill
    + `std::partial_sort()` has the correct `n` first/smallest elements at the front but leaves the rest unsorted
    + `std::nth_element()` has the `n`-th smallest element at the correct place, with unsorted smaller elements before it and unsorted bigger elements after it
    + `std::partition()` has all elements satisfying some criterium (without specifying how many they are) at the front and the rest at the back
  * Once a range is fully sorted, `std::lower_bound()`, `std::upper_bound()` and `std::equal_range()` can be used
    + Given any value `val`, the sorted range can be seen as the succession of three subranges
      - `[first, lb)`, the range of all values `< val`
      - `[lb, ub)`, the range of all values equivalent to `val` (the poorly named "equal range")
      - `[ub, last)`, the range of all values `> val`
    + The functions
      - `std::lower_bound(first, last, val)` returns `lb`
      - `std::upper_bound(first, last, val)` returns `ub`
      - `std::equal_range(first, last, val)` returns both
    + Searching for `val` in `[first, last)` is done by calling `std::equal_range()`, then testing `if (lb != ub)` to make sure `[lb, ub)` is not empty
    + Inserting values equivalent to val can be done:
      - At the beginning of `[lb, ub)` with the `lb` iterator (which points to the first element `>= val`)
      - At the end of `[lb, ub)` with the `ub` iterator (which points to the first element `> val`)

- Algorithms to keep in mind
  * `std::find()` and `std::count()`
    + `find()` stops at the first occurrence and returns the iterator
    + `count()` returns the number of occurrences
    + Use `find()` when checking for existence (testing against the end of the range) except for sets or maps (in which case `count()` is idiomatic and there can be only one)
  * `std::accumulate()`
    + Can be passed a function `f` (default: `+`)
    + Iterates over a range (which it should not modify) and calls
      ```
      cumulativeValue = f(cumulativeValue, newElement);
      ```
  * `std::inner_product()`
    + Can be passed functions `f` and `g` (default: `+` and `*`)
    + Iterates over two ranges (which it should not modify) and calls
      ```
      cumulativeValue = f(cumulativeValue, g(newElement0,newElement1));
      ```
  * `std::transform()`
    + Must be passed a function `f`
    + Either iterates over one input range and one output range, calling
      ```
      *outputIt = f(*inputIt)
      ```
    + Or iterates over two input ranges and one output range, calling
      ```
      *outputIt = f(*inputIt0, *inputIt1)
      ```
    + Could for example be used to implement `operator+=()` between vectors

- Initializer lists
   * `MyClass(std::initializer_list<T> aList);` allows to use the syntax `MyClass var{a, b, c};` or `MyClass var({a, b, c});`
   * Write `A a({});` to call a constructor with an empty initializer list (since `A a{};` calls the default constructor)
   * `void f(std::initializer_list<T> list);` also allows to use the syntax `f({a, b, c});`

- `std::atomic`
  * Is usually more efficient than `std::mutex` with built-in types
  * Use `std::atomic` to make read/write operations on a single variable thread-safe, and `std::mutex` when several variables are involved

- `std::lock_guard`
  * Has less overhead than `std::unique_lock`, but can be locked only once
  * Cannot be used with condition variables, or in other situations when unlocking/relocking is needed

- `std::exchange` (since C++14)
  * Calling `std::exchange(b,c);` sets the value of `b` to `c` and returns the old value
  * Example:
    ```c++
    // Before: a = x, b = y, c = z
    a = std::exchange(b,c);
    // After: a = y, b = z, c = z
    ```
    (it's kind of a half `std::swap`, in which one of the values is discarded and replaced by something else)
  * Can be used in move constructor/assignment
    ```c++
    data = std::exchange(other.data, nullptr);
    ```

- Attributes
  * `[[noreturn]]` for a function that does not return
  * `[[deprecated]]` or `[[deprecated("use bar instead")]]` (since C++14) for a deprecated function
  * `[[fallthrough]]` (since C++17) when falling through in a switch
  * `[[maybe_unused]]` (since C++17) to avoid warnings for unused arguments
  * `[[nodiscard]]` or `[[nodiscard("reason")]]` (c++20) to print a warning if the return value is discarded
  * `[[likely]]` and `[[unlikely]]` to help the compiler optimize a path of execution

- `std::thread`
  * Only use it when necessary (e.g. to use the native handle to set the thread's priority), otherwise use `std::async`
  * Make sure not to destroy a joinable `std::thread` (e.g. end of scope), because that would terminate the program
    + Whatever execution path is taken (including if there's an exceptions), all `std::threads` (besides the default-constructed and moved from ones) must be joined or detached
    + There should be an RAII way to do this, but the order in which destructors are called may involve a risk of getting stuck trying to join a thread which is waiting for a mutex to be released or something
    + Ideally we'd probably prefer to be able to interrupt the thread if not joined or detached, but it's not a feature supported in the standard library yet

- `std::async()`
  * Use `std::async()` over `std::thread` (task-based approach)
  * The abstraction lets the STL and compiler deal with de details of making things happen efficiently, and makes it less likely to get oversubscription (= more threads than what can be handled, which would lead to an exception)
  * Fall back on `std::thread` when low-level control of what's happening is needed, possibily in a platform-specific way (e.g. to use the native handle to set the thread's priority)
  * Returns a `std::future`, which needs to be bound to a reference or moved from (otherwise its destructor is blocking and waits for the function to complete, so the code stays synchronous)
  * Modes
    + `std::launch::async`
      - The task will be run concurrently (e.g. to run a GUI concurrently with the application)
    + `std::launch::deferred`
      - The task will be run when the `std::future` it produces is accessed
      -  A task run with `std::launch::deferred` whose `std::future` is not accessed may not run at all
      - This is a simple way to perform lazy computation
    + `std::launch::async|std::launch::deferred`
      - Default mode
      - Adapts to the circumstances at runtime
      - If you want to be sure that the task is run, make sure to use `myFuture.get()` or `myFuture.wait()` at some point
      - If you want to wait for a task to finish with
        ```
        while(myFuture.wait_for(duration) == std::future_status::ready)
        ```
        (or same thing with `wait_until()`), check
        ```
        myFuture.wait_for(0s) != std::future_status::differed
        ```
        first to make sure you won't get into an infinite loop

- `std::future`
  * If `std::async()` was called in `std::launch::async` mode, and we reach the `std::future`'s destructor (or, in the case of `shared_futures`, the destructor of the last one) before calling `get()` or `wait()`, said destructor will join the thread
  * If that is an issue, pass a `std::packaged_task` to a `std::thread` instead
  * A `std::packaged_task` is basically a `std::function` linked to a `std::future`, and joining or detaching has to be done directly on the `std::thread` without automatically joining when the `std::future` is destroyed (but calling `get()` or `wait()` on the future still joins if necessary)
  * The use of a `std::future` which you don't `get()` is unclear though, and `std::packaged_task` is mostly a lower-level feature used to implement `std::async`

- `std::promise`
  * Can be used in conjunction with `std::future` for inter-thread communication without mutexes and condition variables, when it's relevant
  * The future object must be obtained by moving from `myPromise.get_future()`, so that it exists even if `myPromise` is destroyed (in which case the call will throw a `std::broken_promise` exception)
  * Example: two threads do some work, and one needs to wait for the other one to have reached a certain point before continuing
    + Can be implemented with a `std::promise<void>` and a `std::future<void>`
    + `myFuture.wait();` can only return after the other thread has called `myPromise.set_value();`
    + If needed, a value can be passed while doing so by using another type than `<void>`
  * In particular, that allows to suspend a thread (or several using `std::shared_futures`) and have them run after setting them up (*e.g.* adding a priority through the native handle or something)
  * It's necessary to be careful though: one needs to think about what happens if an exception is thrown before `std::promise` is set
