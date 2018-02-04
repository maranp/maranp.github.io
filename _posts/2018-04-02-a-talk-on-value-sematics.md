---
published: true
---
## Talks I like - Value semantics: It ain't about the syntax!

Following is my short notes on the great [talk](https://www.youtube.com/watch?v=AL9DBWdj-Pg) by John Lakos.

- 2 objects where one is copy constructed from another
  - both are different objects - objects have different address
  - both are in different state
  - both show differenr behaviour
    - apply same sequence of operations, will the observable behaviour be the same
    - NO. there is an illustration in the talk
  - both should have same "value"

- mathematical type:
  - a set of globally unique values
    - Eg: integer
  - a set of operations on those values

- c++ type
  - may (or may not) be an approximation of mathematical type
    - eg: int in c++
  - an object of C++ type (int) represents one of the (subset of )
    unique values of the mathematical type (integer)
  - the c++ object is another representation of the unique value - eg. 5

- how you present (interface) is important not how you represent (data members and its types)
  - thats why we have abstraction

- salient features of a value type
  - the return value of its get methods, well not all the get methods
    - for eg: capacity() of vector is not its salient feature

- a vector approximates a mathematical sequence

- not all stateful objects have a value
  - eg: a flash light has on/off state but does not have a meaningful value
  - similarly a mutex lock - it does something but does not represent a value

- what types are value types, ie, the type that support value - semantic syntax?
  - will the type have equality comparison?
  - will the type have copy construction?
  - will the type have assignment?

- "mechanisms"
  - stateful objects that do not represent values 
- "valu types"
  - stateful objects that *do* represent values

- a value semantic type defines
  - default construction : T a, b
  - copy construction : T a, b (a)
  - destruction
  - copy assignment : a = b
  - operator==

- member operator== is a bad idea. Make it a free operator== (holds good for all binary operators that take consts)
  - because free operator== is needed to satisfy the following condition
    - a == d compiles <=> d == a compiles
      - where a and d are different types

- a copy constructor may or may not copy non-salient attributes. eg: capacity() of std::vector
  - so, looking at the implementation of copy constructor will not help us to
    identify the salient attributes of the class

- instead, look at the implementation of operator== to identify the
  salient attributes of a class

- guideline for selecting salient attributes
  - illustrated with type Rational

- a value type is
  - regular - if it has operator==
  - semi regular - if it does not have operator==

- when selecting salient attributes of a type, avoid subjective interpretation
  - fractions may be equivalent but not equal
  - graphs may be isomorphic but not equal
  - triangles may be similar but not equal
  - subjective interpretaion of equality should be moved from operator==
    to named functions

Recommendations:
John recommends two books in the talk:
Elements of programming - Alexander Stepanov (hard to read)
From Mathematics to generic programming - Alexander Stepanov (more accessible)

Watch the great talk [here](https://www.youtube.com/watch?v=AL9DBWdj-Pg).
