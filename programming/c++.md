The C++ Programming Language
============================

## Initializations

- Static Initializations
  - Zero Initialization
    - for scalars, initialized to `0` casted to the types
    - for others, recursively zero-initialized with padding bits zeroed
  - Constant Initialization
    - initialize to a constexpr
- Dynamic Initializations
  - Default Initialization
    - `T obj;` or `new T;`, etc.
    - for classes, call the default constructor
    - for arrays, recursively default-initialized
    - otherwise, nothing.  Initial value is indeterminate.
  - Value Initialization
    - `T obj();` or `new T();`, etc.
    - for classes with implicitly-defined or defaulted default constructors,
      zero-initialized then default-initialized
    - for other classes (e.g., with user-provided or no default constructors),
      default-initialized
    - for arrays, recursively value-initialized
    - otherwise, zero-initialized
  - Direct Initialization
    - `T obj(args);` or `new T(args)`, etc.
    - for classes, call the suitable constructor
    - otherwise, initialized to the values casted to the types
  - Copy Initialization
    - `T obj = other;` or function parameters, etc.
    - for classes and the values are of compatible types, call the suitable
      converting constructor
    - for classes and the values are incompatible classes, or for non-classes
      and the values are classes, direct-initialized with the values converted
      to the types
    - otherwise, initialized to the values casted to the types
  - List Initialization
    - `T obj{args};` or `new T{args};` or `T obj = {args};`, etc.
  - Aggregate Initialization
    - `T obj{args};` or `T obj = {args};`
    - only when T is an aggregate (arrays or POD classes)
  - Reference Initialization
    - when T is a reference

## Caveats

* `friend` is a sign of ill-designed interface
  * use it no more frequent than `goto`
* reference and pointers
  * pointers are discouraged
  * Never use `void *`.  If it is an opaque object, we should at least know to
    operate on it.  Define a base class for the objects that `void *` may
    point to.
    * Another way to put it is that `void *` semantically means a pointer to
      an uninitialized memory
* Some projects may require
  * do not use RTTI
    * roll your own version if you need one
  * do not use exception
  * do not use STL
  * do not use template
  * do not use default argument
  * do not overload function

## Interface

* A class consists of only pure virtual methods
* it must have a virtual destructor that does nothing
  * imagine `Interface *p = new Implementation;`
  * deleting p would call `Interface`'s synthesized dtor instead of
    `Implementation`'s.

## overloading, overriding, and hiding

* A function is overloaded if there are more than one function with the same
  names but different arguments
* A method in a base class can be overridden if it is virtual and there is a
  derived class that defines the same method with the same arguments
  * the method in the derived class is also virtual, with or without `virtual`
    keyword
* A method in a base class can be hidden if there is a derived class that
  defines the same method, and
  * the method is not virtual
  * the method is virtual, but the derived class defines the method with
    different arguments

## References

* <http://www.cplusplus.com/doc/tutorial/>

## Misc

* There can be multiple characters in a character constant.  The value is
  impl-defined.  In gcc, the values are concatenated.
  * e.g., `'abc' != "abc"`
* placement new
  * `#include <new>`
  * `void *buf = malloc(sizeof(MyClass)); MyClass *c = new(buf) MyClass;`
  * `buf` is used to hold `MyClass`.

## Dynamic Memory

* There are two syntaxes to allocate/deallocate memories
  * `Type *a = new Type` and `delete a`
  * `Type *a = new Type[num]` and `delete [] a`
  * each element will be constructed and destructed
  * On error, `bad_alloc` is thrown
* Another way is to use `nothrow`
  * `Type *a = new (nothrow) Type`
  * On error, `NULL` is returned
* they cannot be mixed with `malloc/free`.
  * where constructor and destructor are not called

## POD type

* POD stands for `plain old data`
* It provides compatibility between C and C++ types
  * it covers all built-in types
  * POD objects share 4 characteristics with their C equivalents
    * initialization, copying, layout, and addressing
* For non-POD type `T`, `new T` and `new T()` will default-initialize the
  object.  `new T(x)` will initialize the object with constructor.
  * default-initilization will call the default-constructor for non-POD types
  * if no constructor is defined for `T`, a synthesized default constructor is
    generated.  The synthesized default constructor calls the default
    constructors of its non-POD members.
* For POD type `T`, `new T()` and `new T(x)` is the same as above.  But,
  `new T` will not initialize the object.
  * default-initialization is zero-initialization for POD types
* POD objects can be `memcpy()`ed.

## Template

* `template <class identifier>` is equivalent to `template <typename identifier>`
  * A entirely different use of `typename`, which is the origin of the keyword
    is to tell the C++ parser that the next token is a type.
    * imagine `template <class T> void foo() { typename T::iterator *iter; ... }`
    * if T is `class Bar { static int iterator; };`, without `typename`, the
      first line becomes a statement
* A simple template function looks like
  `template <class Type> Type max(Type a, Type b) { return (a > b) ? a : b; }`
  * it can be called like `max<int>(a, b)`.  Or simply `max(a, b)`.  The
    compiler will derive the type automatically.
* Template specialization
  * it is possible to specialize a template function/class for certain types
  * `template <> char max<char>(char a, char b) { return a;}`.  Note the empty
    angle brackets.
* Non-type template parameters
  * `template <class T, int N> class array { T buf[N]; };`
  * It is also possible to give template parameters default values
    `template <class T=unsigned char, int N=10> class array { T buf[N]; };`


## Class

* class is a struct whose members are private by default.
* typedef is implied
  * When `class A` is declared, `typedef class A A` is implied.
* Constructor
  * There are 4 special member funcions that the compiler will generate if none
    defined.
    * Default constructor
    * Copy constructor
    * Copy assignment operator
    * Destructor
  * A default constructor is a constructor that takes no arguments
    * Or every argument has a default
  * A constructor with a single argument may be used for implicit conversion
    * `MyClass a(3); a = 4;` may be equal to
      `MyClass a(3); a = static_cast<MyClass>(4);`
    * If it is not desired, one should use `explicit` keyword when defining the
      constructor.
  * When no constructor is defined by user, a default constructor will be
    generated by the compiler.
    * It is called synthesized default constructor
    * I am not sure if its behavior is defined.  But it may not initialize the
      built-in types in the class.
  * `class A a` and `class A a = A()` is equivalent.
    * so does `class A a()`.
  * `class A a[3] = { 1, A("la") }` will initialize `a[0] = A(1)`,
    `a[1] = A("la")`, and `a[2] = A()`.
  * `class A { A() : _b(B()) { } };` is better than
    `class A { A() { _b = B() }};`.
    * The latter initialize `_b` twice.
  * The only way to fail a constructor is to throw an exception
* Copy constructor
  * `Type(const Type &rhs);`
  * other than default constructor, compiler will generate a copy constructor
    for us if none is defined.
    * the generated one performs shallow copy
* Rule of three
  * destructor
  * copy constructor
  * copy assignment operator
  * if any of the above is defined, one should define them all.
* Initialization order
  * Base class objects are initialized first
    * the default construtors of the parent classes are called
    * unless `ParentClass(params)` is given in the initialization-list
  * Member data objects are then initialized
    * again, members' default constructors are called.
    * unless they are assigned in the initialization-list.
  * The constructor is finally called.
  * Base class objects and member objects are initialized in declaration order.
    Not in initilaization-list order.
  * google for `static initialization order fiasco` as well
* A member function can have `const` in the end
  `int memb_func(int) const;`
  * it means `this` can be constant
  * a constant member function cannot modify any of the members
* Protection
  * a member that is `private` or `protected` cannot usually be accessed outside
    the class
    * unless it is declared as a `friend`
    * Suppose `void Func(A a)` is a normal function.  By adding
      `friend void Func(A a);` to `class A` gives the function access to the
      private/proctected members of `class A`.  This is seldom used.
    * It is also possible to grant a class, `class B`, access to everything in
      another class, `class A` by addming `friend class B;` to `class A`.
    * A friend of a friend is not a friend.
  * A `protected` member can also be accessed by the inheriting classes.
  * `class Child : protected class Parent` makes all members inherited from
    `Parent` at best `protected`.  A private member in `Parent` is still
    private.  But a `public` member in `Parent` becomes `protected`.
* static member
  * static class data cannot be defined in the header.  They are declared in
    the header, but defined in the C++ source.
  * static class functions do not have access to non-static data.  There is no
    `this`.
    * in other word, `this` is passed to all non-static member functions, and
      it has to.  static member function could be faster.  Of course,
      optimization can eliminate the overhead.
* a member of a constant object may still be changed if it is qualified with
  `mutable` keyword.
  * Can be used to implement reference counting.  That is, to increase or
    decrease the reference count of a constant object

## Polymorphism

* A derived class pointer can be implicitly converted to a base class pointer
  * `class Child : public class Parent {};`
  * `Child child; Parent *parent = &child;`
  * The problem is, if both parent and child defines `do_something()`.
    `child.do_something()` and `parent->do_something()` will call the respetive
    definitions.
  * The solution is to add `virtual` to parent's `do_something()`.
  * In such case, both parent and child are called polymorphic classes.
* If a virtual function does not have a implementation, and instead, has ` = 0`
  appended, it becomes pure virtual function.
  * A class with at least one pure virtual function is called an abstract base
    class.  It cannot be instanciated.

## Multiple Inheritance and vtable

* `vtable`
  * every class that uses virtual functions or is derived from a class that
    uses virtual functions has `vtable`
  * `vtable` has one entry point for each virtual functions.  Each entry is
    simply a function pointer pointing to the most-derived function.
  * the compiler also inserts a pointer to the most-base class, making the
    class the size of a pointer larger
    * the pointer points to the `vtable` of the class
* Suppose `Top`, `Left`, `Right`, and `Bottom` form a diamond inheritance.
* Given `Bottom *b = new Bottom`, `Left *l = b`, and `Right *r = b`
  * The address of `l` is equal to `b`; the address of `r` is _not_ equal to `b`
* When a class has virtual functions or has virtual inheritance(s), there will
  be an associated set of vtables.
  * There are more than one vtable for a particular class, if it is used as a
    base class for other classes.
* Each instance is inserted a set of vtable pointers.
* Suppose `Left` and `Right` are derived from `Top` virtually,
  * The address of `l` is still equal to `b`; the address of `r` is still
    _not_ equal to `b`
  * `l` points to the vtable of `Left` in `Bottom`; `r` points to the vtable of
    `Right` in `Bottom`.

## Namespace

* `namespace identifier { }`
* `using namespace identifier;`
* `namespace alised_name = name`
  * this is called namespace aliasing.
* there is an implicit global namespace whose variable can be accessed with
  `::variable`
  * this is used if a namespace also has `variable` and we need to access the
    global one
* unnamed namespace `namespace {}`
  * this has the effect of `static`ized all variables/types in the unnamed space
  for the file scope

## Type Casting

* `static_cast<T>(expr)` is the same as performing `T t(expr);`.  The result is an
  lvalue if T is a reference type, and an rvalue otherwise.
* Explicit type conversion (cast notation)
  * `(T) expr` results in an expression of type T.  It is an lvalue if T is a
    reference type, otherwise, the result is an rvalue.
  * It is equivalent to
    * `const_cast`
    * `static_cast`
    * `static_cast followed by const_cast`,
    * `reinterpret_cast`
    * `reinterpret_cast followed by const_cast`
* Explicit type conversion (funtional notation)
  * `T(expr)` is equivalent to `(T) expr`
  * `T(expr1, expr2, ...)` calls a constructor to contruct a temporary rvalue
  * `T()` also creates an rvalue, which is value-initialized
* Implicit conversion
  * It is done implicitly when one, say, assign a double to an int.
* Explicit conversion
  * C-style: `(new_type) expression`
  * C++-style: `new_type(expression)`
  * They are equivalent.  They are also equivalent to who ever succeeds first
    below
    * `const_cast`
    * `static_cast`
    * `static_cast followed by const_cast`,
    * `reinterpret_cast`
    * `reinterpret_cast followed by const_cast`
* Typecast operators

    dynamic_cast <new_type> (expression)
    reinterpret_cast <new_type> (expression)
    static_cast <new_type> (expression)
    const_cast <new_type> (expression)
* `reinterpret_cast` casts between pointers to different classes, or between
  integer and pointer.
  * It is seldom used.
* `static_cast` casts bwtween base class and derived class, or where implicit
  conversion is possible (e.g., `void *` to `char *`).
  * In other words, it can perform implicit conversions and the reverse of them.
* `const_cast` adds or removes the const-ness.
* Cast generally creates a temporary object
  * suppose `class A { A(int); };`
  * `static_cast<A>(3)`, `(A) 3`, and `A(3)` all create a temporary object of
    type A

## Initialization

* Let's define three types of initialization
  * zero-initialize
    * if T is a scalar type, the object is set to 0 (converted to T)
    * if T is a non-union class type, each non-static member and each base
      class is zero-initialized
    * ...
  * default-initialize
    * if T is a non-POD class type, the default constructor is called
    * if T is an array type, each element is default-initialized
    * otherwise, the object is zero-initialized
  * value-initialize
    * ... (almost the same as default-initialize)
* Each object of static storage is zero-initialized
* An object whose initializer is `()` should be value-initialized
* If no initializer is specified for an object, the object is
  default-initialized if it is non-POD, and has undetermined intermediate
  value otherwise
* Direct-initializaion is the initialization occurs in `new`, `static_cast`,
  functional notation type conversions, and base and member initializaion.
  * it is equivalent to `T x(a);`
  * it finds the matching constructor and calls the constructor to initialize
    the object
* Copy-initialization is the initialization occurs in argument passing,
  function return, and etc.
  * it is equivalent to `T x = a;`
  * if `a` is of type T or a derived class of T, it is the came as
    direction-initialization
  * otherwise, it finds the matching user-defined conversion and calls the
    conversion.  The result of the convesion is then used to direct-initialize
    the destination.  This calls the copy constructor, but an implementaiton
    can optimize this out.

## Operator overloading

* Some operators are automatically generated.  To disable them,
  * e.g., put `Type &operator=(const Type &rhs);` in `private`.
  * same trick to disable constructor `Type(const Type &rhs);`
* `const Type operator+(const Type &rhs) const;`.

## reference counting

* Example

    class ReferenceCount {
    public:
        ReferenceCount() : count(1) {}
        virtual ~ReferenceCount() {} // so that the destructor of the
                                     // most-derived class is called
        void ref() const { count++; }
        void unref() const { count--; if (count == 0) delete this; }

    private:
        mutable int count; // must be mutable so that we can manipulate the
                           // reference count a const object
    };

## One Definition Rule (ODR)

* In any translation unit, a template, type(/class), function or
  object(/variable) can have no more than one definition.
  * `class C; class C;` is fine because it is just two declarations
  * `class C {}; class {};` is wrong because it is two definitions
* In the entire program, an object or non-inline function cannot have more
  than one definition.
  * But a template or type can.  That is why we can define classes in the
    headers, but have to define class methods in the sources.
  * Of course unless the class methods are inlined.  Note that a class method
    defined within the class definition is implicitly inlined
* static const integral members does not require a definition
  * `class C { static const int N = 10; };` is as if `C::N` is a `#define` or
    `enum`.

## Effective STL

* STL has containers, iterators, algorithms, and function objects.  The most
  important thing one is the containers.
* Containers
  * sequence containers: vector, string, deque, and list (item 1)
  * associative containers: set, multiset, map, and multimap (item 1)
  * there is no container-independent code.  Different containers are
    different (item 2)
  * As the choice of containers may change and it is not realistic to write
    container-independent code, do this (item 2)

    class Obj { ... };
    // typedefs that allow the container type to be more easily changed
    typedef std::vector<Obj> ObjContainer;
    typdef ObjContainer::iterator ObjIterator;

  * typedefs also save lots of typings (item 2)
  * objects in containers are copy in, copy out mostly.  Make copy cheap and
    correct.  Or, store pointers to objects in the containers (item 3)
  * because of copying, inserting a derived object in a container of base
    objects is almost always an erro (item 3)
  * call empty() instead of size() == 0 (item 4)
  * list.splice() is efficient (item 4)
  * use assign(), range insert(), range erase(), or range construction instead
    of a loop (item 5)
  * use `for_each` (and a function object) to loop a container (item 7)
  * remember to delete newed objects when the containter
    stores pointers to those objects (or use `boost::shared_ptr`) (item 7)
  * containers of `auto_ptr` are prohibited (item 8)
  * learn erase(), remove(), std::remove(), and `std::remove_if` (item 9)
  * STL allocator is weird.  Be careful when writing a custom one (item 10)
  * RAII example: Lock class (item 12)
  * use vector (or string) to replace arrays (item 13)
  * size() tells how many elements are in a container, capacity() tells how
    many elements the container can hold, resize() changes the number of
    elements the container holds, reserve() changes capacity() (item 14)
  * `&vec[0]` or `str.c_str()` to pass vectors and strings to C (item 16)
  * avoid `vector<bool>` as `vector<bool> v; bool *p = &v[0]` does not work.
    Use `bitset` or `deque<bool>` instead. (item 18)
  * associative containers use equivalence, not equality (item 19)
  * standard associate containers are sorted (item 19)
  * anytime an associative container of pointers is created, you almost always
    want to specify the comparison type (item 20)
  * comparisons should have strict weak ordering (item 21)
    * partial ordering (i.e., less than or equal to)
      * `f(x, x) must be true`
      * `f(x, y) and f(y, x) imply x is y`
      * `f(x, y) and f(y, z) imply f(x, z)`
    * strict partial ordering (i.e. less than)
      * `f(x, x) must be false`
      * `f(x, y) implies !f(y, x)`
      * `f(x, y) and f(y, z) imply f(x, z)`
    * a partial ordering becomes total when all elements are comparable
    * there is a one-to-one relationhsip between partial and strict partial
      orderings
    * a strick weak ordering is a strict partial ordering plus
      * `we say that x and y are equivalent if f(x, y) and f(y, x) are both
        false.  Then if x is equivalent to y, and y is equivalent to z, then x
        is equivalent to z`
  * map is sorted by the keys, not values (item 22)
  * map keys are const, set values are not (item 22)
  * do not change (at least the key part of) elements in a set (item 22)
  * std associate container lookups are logarithmic-time (item 23)
  * consider using a sorted vector or non-std hash table for lookups (item 23)
  * for a map, use `operator[]` to update and `insert` to add elements (item
    24)
  * use the hinted version of `insert` or `erase` (item 24)
    * `insert(hint, ...)` adds the element before `hint`
  * prefer `iterator` over `const_iterator` (item 26)
  * you cannot cast `const_iterator` to `iterator` (item 27)
  * replace `istream_iterator` by `istreambuf_iterator` for char-by-char input
    (item 29)
  * it is `transform(in.begin(), in.end(), back_inserter(out), func)`, not
    `transform(in.begin(), in.end(), out.end(), func)` (item 30)
  * `sort`, `stable_sort`, `partial_sort`, `nth_element`, `partition` (item 31)
  * `remove`, `remove_if`, `unique` (item 32)
  * `InputIterator`, `ForwardIterator`, `BidirectionalIterator`, and
    `RandomAccessIterator` (item 34)
  * `mismatch` and `lexicographical_compare` (item 35)
  * functors passed to all algorithms except `for_each` should not have side
    effects (item 37)
  * functors are passed by values.  Make copy cheat and avoid virtual
    functions (use pimpl idiom) (item 38)
  * make predicates pure (item 39)
  * function objects are also known as functors.  A funcion object may or may
    not be adaptable. (item 40)
  * `for_each(begin, end, func)` calls `func(*it)` on each iterator.  If
    `func` is a member function of elements in the container, it would not
    compile because we would need `*it.func()` or `*it->func()`.  The way to
    resolve this is to use `mem_func_ref` or `mem_func` (item 41)
  * `transform(in, in + size, inserter(out, out.begin()), bind2nd(plus<int>(), num))`
    (item 43)
  * use algorithms to replace loops, unless that results in many more lines of
    code (item 43)
  * `count`, `find`, `equal_range`, `upper_bound`, `lower_bound` (item 45)
  * function objects allow inlining! prefer them to function pointers (item 46)

## Effective C++

* `iterator`s are modeled on pointers.  `const iterator` thus means
  `T * const`.  We need `const_iterator` to mean `const T *` (item 3)
* Return value of `operator*` and others should be const (item 3)
  * to avoid `(A * B) = C`
* `mutable` can be used for cached values.  For example, the `getSize()`
   member method should be const.  But if calculating the size costs much, we
   want to cache the calculated result (item 3)
* casting is bad (item 3)
* use initialization list to initialize _all_ members.  If there are multiple
  constructors, consider a private init function to avoid repetition (item 4)
* Order of initialization: base class, derived class, and then data members
  one by one (item 4)
* avoid non-local static objects.  The order of initialization is
  unpredictable.  Use the singleton pattern instead (item 4)
* compiler may generate default ctor, copy ctor, dtor, and copy assignment
  operator (item 5)
* declare compiler-generated methods private to disable the generation (item 6)
  * e.g., since every person is unique, a `Persion` class should not have copy
    ctor nor copy assignment operator.  Declaring them private will make all
    uses of them generate a compile-time error.
  * However, if they are called in another member function, it only gives
    link-time undefined reference error.  To make it compile-time error, see
    `boost::noncopyable`
* polymorphic base classes should declare virtual destructors.  If a class has
  any virtual function, it is likely to be a polymorphic base class (item 7)
* If a base class is intended to be abstract but there is no virtual
  functions, declare the dtor pure virtual (item 7)
  * being abstract means being polymorphic, thus needing a virtual dtor
  * having dtor pure virtual makes the class abstract
  * but since the compiler will call the base class dtor, we need to define
    `Abstract::~Abstract() {}`
* destructors should not emit exceptions (item 8)
* never call virtual functions in ctors or dtors (item 9)
  * as they will call the ones in the base class
* return a reference to `*this` in assignment operators (item 10)
  * so that `x = y = z = blah;` works
* handle assignment to self in operator=.  make sure exception does not result
  in partially assigned object (item 11)
* copy members and call base class's in copy ctor and copy assignment operator
  (item 12)
* RAII and use `std::tr1::shared_ptr` for heap-based resources (item 13)
* an RAII object should prohibit copying, reference-count the resource on
  copying, copy the resource on copying, or transfer ownership on copying
  (item 14)
* RAII object will provide `get()` to return the underlying resource (item 15)
  * sometimes, an implicit type conversion (operator TYPE) may be desired
* `new` allocates an object and calls its constructor.  `delete` does the
  reverse.  `new []` allocates an array of objects and calls their
  constructors.  `delete []` does the reverse, but how does it know how many
  objects there are given just a pointer?  Usually, the memory layout of
  `new []` is `N obj1 obj2 ... objN` (item 16)
* `std::vector` makes the need for `new []` nearly zero (item 16)
* `func(std::tr1::shared_ptr<Obj>(new Obj), some_func_that_throws())` is wrong
  because it may happen that the object is created but `shared_ptr` has not
  constructed when the second function throws (item 17)
* questions to ask when defining a class (item 19)
  * how should initialization (ctor) differ from assighment (operator=)?
  * how should pass by value (copy ctor) work?
  * how should type conversion work?
    * `operator T2() const` defines how to convert to T2
  * what operators and functions are supported?  functions may be member or
    non-member
  * what standard functions should be disallowed?  E.g. copy ctor
* pass-by-reference-to-const instead of pass-by-value for user-defined types
  (item 20)
  * reference may be implemented by pointers.  For built-in types such as int,
    pass-by-value is more efficient
  * STL iterators or functors are designed to be passed by value too.
* all data members should be private (item 22)
* If a member function does not provide capabilities that cannot be achieved
  other public member functions (e.g. a helper member funcion), then it should
  be moved out of the class (item 23)
* declare a non-member function when type conversions should apply to all
  parameters (item 24)
  * suppose you have `class Rational;` for rational numbers
  * if the class has `operator*` as a member function, `rational * 2` works
    but `2 * rational` does not because `2.operator*(rational)` is not
    declared.  You want to declare `operator*` as a non-member function in the
    namespace of `class Rational;`
* consider support an efficient and non-throwing swap (item 25)
* minimize cast (item 27)
* call `Base::Method` instead of `static_cast<Base>(*this).Method` (item 27)
* avoid returning handles (references, pointers, iterators) to the internal
  data members (item 28)
* strive for exception-safe code (item 29)
* inline is a hint.  member functions defined within the class definition is
  implicitly inlined (item 30)
* compilers do inlining at compile time.  compilers do template instantiating
  at compile time.  As such, inline functions and templates should generally
  to be defined in the header files (item 30)
* consider pimpl idiom (handle class) or abstract class (interface class) to
  reduce header file dependencies (item 31)
* make sure public inheritance models "is-a" (item 32)
  * however in the sense that a square is not a rectangle.  For if a square
    was a rectangle, we would be able to create a square whose width != height
* remember a variable in the local scope can shadow a global variable of the
  same name, even their types differ?  A member function always shadows the
  member function, or overloaded functions, in the base class of the same
  name, disgarding the function arguments, virtual or not.
  * shadowing includes member data, member functions, enums, nested classes,
    typedefs, and etc.
  * un-hide the base class name by declaring `using Base::method;` in the
    derived class
  * in public inheritance, which models "is-a", one should unhide all names.
* you can implement a pure virtual function in the base class.  It needs to be
  invoked like `obj->BaseClass::PureVirtualFunction()` (item 34)
* a member function that should be invariant over specialization should not be
  virtual (item 34)
* a derived class inherits the function interface of a pure virtual function; it
  inherits the function interface as well as a default implementation of a
  virtual function;  it inherits the function interface as well as the
  mandatory implementation (item 34)
  * a non-pure virtual function in the base class serves as a common
    implementation.  But as not every derived class wants it, one can make the
    function pure, and provide a protected method, `DefaultDoSomething()`, to
    be called by derived classes who want.  The upside is that a new derived
    not wanting the common implementaiton is less likely to use it
    accidentally.
  * another alternative is to make the function pure virtual, but still
    provide the definition.
* alternatives to virtual functions (item 35)
  * non-virtual interface (NVI) idiom, aka., template method design pattern:
    provide non-virtual public member functions that wrap private virtul ones
    * it allows you to to pre- and post- work before calling the real function
  * stratgy pattern: provide function pointer to the constructor
    * it allows you to use different impl for different objects of the same
      type, or change the impl at runtime
* never redefine an inherited non-virtual funciton (item 36)
* never redefine a function's inherited default parameter value (item 37)
  * Or better off, do not assign default parameter values to virtual
    functions
* "has-a" and "is-implemented-in-terms-of" (item 38)
  * a set can be implemented by a list.  If the set inherits the list
    ("is-a"), as the list allows duplicated values, the set has incorrect
    behavior.  The set should "has-a" list, or in another word,
    "is-implemented-in-terms-of" a list.
* private inheritance means "is-implemented-in-terms-of" (item 39)
  * no software design meaning, but software impl meaning
  * you want private inheritance, instead of compoisition, so that you can 
    access the protected members or redefine virtual functions.
  * but a better approach is to have a private nested class that publicly
    inherits the base class, and use composition
    * better because you do not want others to inherit you and
      redefine the virtual functions
* private inheritance should be used only when the base class has zero-size
  (item 39)
  * that is, when empty base optimization (EBO) applies
* don't use virtual base classes.  If you have to, the base class should have
  not data members (item 40)
* class has explicit interface and runtime polymorphism
* template has implicit interface and compile-time polymorphism (item 41)
  * the interface is described by valid expressions
  * e.g., a template function supports any type whose interface makes the
    expressions in the template function valid
  * it shows compile-time polymorphism in that the compiler decides the type
    and the function to be called at the compile time
* nested dependent name (e.g., `T::iterator`) is assume to be a value in a
  tepmlate.  Prefix it by `typename` to make it a type (item 42)
* in derived template class, refer to names in base class templates via
  a `this->` prefix or via `using` declarations (item 43)
* watch out for object size bloat when using templates (item 44)
* When writing a class template that offers functions related to the template
  that support implicit type conversions on all parameters, define those
  functions as friends inside the class template (item 46)
* traits, using iterator and `advance` as an example (item 47)
* template metaprogramming (item 48)
* curiously recurring template pattern (CRTP) (item 49)
* conventions for overriding operator new and operator delete (item 51)
* when `operator new` takes extra parameters other than the size, it is known
  as the placement version of new (item 52)
  * it is invoked as `new (extra arguments) Type`
  * people usually refer to `void *operator new(size_t, void *);` as _the_
    placement new.  It is used by STL containers.
* `namespace tr1 = ::boost;` makes tr1 an alais for boost (item 54)
* TR1 has hash tables (item 54)
* Embedded
  * virtual function size and speed penalties (slide 24)
  * an object has multiple addresses under multiple inheritance (slide 25)
  * no-cost c++ features (slide 33)
    * do not use default parameters!  just overload
  * exceptions, MI, RTTI cost (slide 36, 42)
  * template code is conventionally in the header (slide 61)
  * interface-based programming (slide 77)
    * runtime polymorphism: inheritance and virtual functions. most flexible
      and most expensive (vptr, vtbl, non-inline function calls)
    * link-time polymorphism: no virtual functions, but still not allowing
      inlining.
      * pimpl idiom
    * compile-time polymorphism: no virtual functions, inline-able
      * traits
  * dynamic memory management concerns (slide 98)
    * speed: how fast is new/delete? is it deterministic?
    * fragmentation: the heap consists of unusable small chunks
    * leak: forget to free?
    * OOM: what should happen?
  * memroy allocation strategies (slide 99)
    * fully static: everything is either on stack or in static storage
      * useful when the max number of objects can be statically determined
      * all concerns are irrelevant
    * LIFO: allocate a buffer statically, have a stack-like allocator class
      for the buffer
    * pool: for objects of a fixed-size.
    * block:
    * region:
  * modeling memory-mapped IO (slide 140)
