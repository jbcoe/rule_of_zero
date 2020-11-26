# Value semantics and the rule of zero in modern C++

C++ has a rule of zero that suggests that users should not be writing 
compiler-generated functions like  copy constructors, destructors, assignment 
operations and the like. 

The rule is easy to state but hard to follow. I want to fix this and have 
proposed two new class-templates polymorphic_value, indirect_value for 
addition to the C++23 standard library.

## Values vs References

## Compiler-generated special member functions in C++

## The Rule of Zero

## Pointers as members

## Node-like structures

## PImpl pattern

## Hot-cold splitting

## Polymorphism

### Compile-time polymorphism

### Closed-set polymorphism

### Open-set polymorphism

## Problems with pointers

### Const-propagation

### Deep copies

## Fixing const-propagation: `std::experimental::propagate_const`

## `polymorphic_value` and `indirect_value`

## Proposed for C++23

## Implemented on GitHub

## Implementing `polymorphic_value` with type-erasure


https://www.fluentcpp.com/2019/04/23/the-rule-of-zero-zero-constructor-zero-calorie/
wg21.link/p0201
wg21.link/p1950

I will present motivation and implementation details for these classes."