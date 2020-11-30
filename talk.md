## Value semantics and the rule of zero in modern C++

C++ has a rule of zero that suggests that users should not be writing 
compiler-generated functions like  copy constructors, destructors and 
assignment operations.

The rule is easy to state but hard to follow. I want to fix this and have 
proposed two new class-templates `polymorphic_value` and `indirect_value` 
for addition to the C++ standard library.

---

## Data encapsulation with structs and classes

C++, like C, lets us group associated data together into structs and classes.

Members of a struct or class can be primitives like pointers, integers and 
floating point numbers:

```~cpp
struct Date {
    int day;
    int month;
    int year;
};
```

---

Members of a struct or class can also be other structs or classes:

```~cpp
struct Employee {
    char[64] name;
    Date start_date;
    Date birthday;
};
```

---

## Copies

In C, when an instance of a struct is copied the copy will be a bit-for-bit 
copy. 

In C++, when a class (or struct) is copied the compiler generated copy 
constructor will perform a member-wise copy for each member of the class. 

In C++, a class author is able to specify a copy constructor for a class which 
will then be called when an instance of that class is copied.

The distinction between bit-for-bit copies and member-by-member copies becomes 
significant when members of a class are pointers to other data and ownership of 
that data must be considered.

```~cpp
struct Node {
    int data;
    Node* next;
};
```

---

## Destructors

When an object goes out of scope it is destroyed and memory used by its members 
will be freed up.

In C++ one can specify a destructor for a class that determines what else happens 
when an instance of that class is destroyed. 

One could, for instance, destroy the objects pointed to by pointer-members 
of the class. 

If the pointees are owned by the class instance, it would make sense to delete 
them as they should not have a lifetime that extends beyond the object they are 
logically part of.

---

## Moving

C++11 and later allow objects to be moved. 

A move constructor will be called in contexts where an  object will not be used 
again (either explicitly or determined by the compiler). 

This allows potentially expensive customised copying operations to be skipped 
over altogether.

If pointed-to data will not be used again then it can be re-used by the 
moved-to object rather than copied.

---

```~cpp
class Matrix {
    double* data_;
    // other members and functions omitted 
 
 public:
    // move constructor
    Matrix(Matrix&& m) {
        data_ = m.data_;
        m.data_ = nullptr; // Does not delete the data.
    }

    ~Matrix() {
        delete data_;
    }
};
```

---

## The Rule of Zero

C++ will generate some, none or all of the special member functions under a set 
of somewhat complicated conditions. 

There is an easy rule to remember though - the rule of zero:

Either you specify all of the special member functions or you specify none of 
them.

If a class requires special logic to be copied or moved or deleted then chances 
are it will need some special logic for copies, moves _and_ deletion.

---

## Pointers as members

If we never need pointers as members of a class then compiler-generated functions 
will serve us well.

There are unavoidably some circumstances in which pointer members are needed.

* Node-like structures

* Hot-cold splitting

* PImpl pattern

* Open-set polymorphism

---

## Node-like structures

A tree or linked-list can be implemented as a node-based container. Nodes would 
contain data and pointer(s) to other nodes so that the list or tree can be 
navigated

```~cpp
class ForwardListNode {
    int data_;
    ForwardListNode* next_;
}; 

class ForwardList {
    ForwardListNode* head_;
public:
    // ...
};
``` 

---

## Hot-cold splitting

We can design a class so that data is segregated according to how frequently 
it is accessed. This keeps more frequently accessed data in cache:

```~cpp
class Element {
  SmallData frequently_accessed_data;
  LargeData* infrequently_accessed_data;
};
```

Keeping `LargeData` as a pointer keeps the object size down and reduces the 
frequency of cache evictions.

---

## PImpl pattern

The handle-body idiom was used to keep compilation times down and will probably 
see a resurgance in use as people struggle to continue to improve performance 
while keeping ABI stable.

```~cpp
class Widget {
public:
    Widget();
    ~Widget();

    ReturnType DoThing(Some args);
    AnotherReturnType DoOtherThing(SomeOther args);
private:
    class Impl* pimpl_;
};
```

---

The implementation of `Widget` will contain a definition of `Widget::Impl` 
which will be used to implement `Widget`'s member functions. 

This class definition allows details of the implementation to be changed 
without users of the class needing to recompile code.

---

## Polymorphism

It may be desirable for an interface to act on objects of different types. 

C++ supports multiple forms of polymorphism with language or library features.

We have 

* compile-time polymorphism

* closed-set polymorphism

* open-set polymorphism

---

## Compile-time polymorphism

Compile-time polymorphism allows multiple types to be supported provided that
in any instance the type is known at compile time. Compile time polymorphism 
can be achieved with function overloads or templates.

```~cpp
void doThing(Type1 t);

void doThing(Type2 t);

template <typename class>
void doThing(T t);
```

---

## Closed-set run-time polymorphism

When the type of an object may be only determined at run-time 
but the number of types is known, an interface can take a variant.

```~cpp
using Instrument = std::variant<Stock, Bond, Future>;

double current_value(const Instrument& i)
```

---

## Open-set run-time polymorphism

When the type of an object may only be determined at run-time
and the interface must support an open set of types 
(such as types that inherit from a framework-defined base class)
 an interface can take a reference or pointer to a base class.

 ```~cpp
 class GameObject;

 void Update(GameObject& game_object);
 ```

---

## Polymorphic components

When a class needs a polymorphic member we can pick the most suitable
form of polymorphism available.

This may result in an open-set polymorphic member variable being needed.
We can only store this member as a pointer as its size cannot be determined
at compile time.

```~cpp
struct FileSystem {
  virtual ~FileSystem();
  virtual Write(std::string_view) = 0;
};

class FileWriter {
    FileSystem* file_system;
};
```

---

## Compiler generated functions with pointer members

When a class has a pointer member, the compiler-generated copy constructor will 
copy the pointer, not the pointee.

A compiler-generated move constructor will copy the pointer leaving both the 
copy and the moved from object with a member pointer that points to the same 
object.

Similarly the compiler-generated destructor will not free the memory that the 
pointer points to. 

If the data pointed to by a pointer member is logically part of the object then 
we will need to implement all the special member functions ourselves to ensure 
correct behaviour.

---

## Const-propagation

C++ added the `const` keyword which was then back-ported into C.

An object accessed through a `const` access path cannot be modified.

This const-ness is shallow:

While the compiler prevents modifications to a member pointer in a `const` 
object it does not propagate this constness to the pointee. We cannot point 
at different data but we are free to modify the data.

If the pointed-to data is logically part of an object then it should not be 
possible to modify it when the object is accessed through a `const` access path.

Without compiler help, we'll need to be careful and ensure that no `const` 
member functions modify pointed-to objects through pointer members.

---

## Do one job and do it well

The separation of concerns is a principle of sotware design that says each 
section of a program should address a single concern. 

If a class has ownership logic that requires user-specified special member 
functions, then all it should do is control ownership of and access to its 
members. 

---

## Values and references

The C++ standard library contains classes that are implemented with pointers 
but have value semantics.

`std::vector` and `std::map` do not allow members of `const` containers to be 
modified.

Copies of standard library constainers are distinct objects. Modifications 
to a copy would not be reflected in the original.

The C++ standard library is missing value-types that are suitable for storing 
polymorphic or indirect member variables.

---

## `polymorphic_value`

The class template, `polymorphic_value`, confers value-like semantics on a freestore 
allocated object. 

A `polymorphic_value<T>` may hold an object of a class publicly derived from `T`, 
and copying the `polymorphic_value<T>` will copy the object of the derived type.

When `polymorphic_value` is accessed through a `const` access path, only `const` 
access to the held object is possible.

---

## `polymorphic_value` interface 

```~cpp
  template <class T>
  class polymorphic_value {
    ~polymorphic_value() = default;

    // Some constructors omitted

    polymorphic_value(const polymorphic_value& p);
    polymorphic_value(polymorphic_value&& p) noexcept;

    explicit polymorphic_value(const polymorphic_value<U>& p);
    explicit polymorphic_value(polymorphic_value<U>&& p);

    polymorphic_value& operator=(const polymorphic_value& p);
    polymorphic_value& operator=(polymorphic_value&& p) noexcept;

    explicit operator bool() const;

    const T* operator->() const;
    const T& operator*() const;

    T* operator->();
    T& operator*();
  };
```

---

## `indirect_value`

The class template, `indirect_value`, confers value-like semantics on a 
free-store allocated object. 

An `indirect_value<T>` may hold an object of class `T`, copying the 
`indirect_value` will copy the object. 

When `indirect_value` is accessed through a `const` access path, only 
`const` access to the held object is possible.

---

##  `indirect_value` interface

```~cpp
  template <class T>
  class indirect_value {
    ~indirect_value() = default;

    // Some constructors omitted

    indirect_value(const indirect_value& p);
    indirect_value(indirect_value&& p) noexcept;

    indirect_value& operator=(const indirect_value& p);
    indirect_value& operator=(indirect_value&& p) noexcept;

    explicit operator bool() const;

    const T* operator->() const;
    const T& operator*() const;

    T* operator->();
    T& operator*();
  };
```

---

## Implementing `indirect_value`

Implementation of `indirect value` is relatively simple.

The class contains a pointer-member, user-defined special member 
functions and suitable `const`-qualified overloads for accessors.

---

## Implementing `polymorphic_value` with type-erasure

Implementation of `polymorphic_value` is more involved.

Like `indirect value`, the class contains a pointer-member, 
user-defined special member functions and suitable 
`const`-qualified overloads for accessors.

The class also contains a type-erased control block that is able 
to perform correct copies of derived types.

---

## Type erasure

Type erasure uses a combination of generics and virtual dispatch
to turn structural subtypes into an object hierarchy.

```~cpp
class TypeErasedWriter {
    struct BaseWriter {
        virtual void Write(std::string_view s) = 0;
        virtual ~BaseWriter() = default;
    }

    template <class T>  
    // T needs a `Write` method but no base class.
    struct GenericWriter{
        T t_;
        GenericWriter(T&& t) : t_(std::move(t)) {}
        void Write(std::string_view s) override { t_.Write(); }
    }
    
    BaseWriter* writer_;
 
 public:
    template <class T> 
    TypeErasedWriter(T&& t) {
        writer_ = new GenericWriter<T>(std::move(t));
    }
    
    void Write() { writer_.Write(); }
};
```

---

## Standardisation

`polymorphic_value` has been approved by the C++ Library Evolution working 
group and reviewed by the Library working group and needs approval in a 
plenary meeting to be included in a future standard.

http://wg21.link/p0201

`indirect_value` has been forwarded by the C++ Library Incubator working group
for review by the Library Evolution working group.

http://wg21.link/p1950

The authors hope to see both included in a future C++ standard library.

---

## Acknowledgements

A great many folk have contributed to discussions around the design of 
these class templates.

I'd like to explicitly acknowledge the contribution of my co-authors 
Sean Parent (at Adobe) and Antony Peacock (at Maven Capital Partners).

---

## Questions?
