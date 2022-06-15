# Coding Guidelines for the C programming language

C is and has been a staple in Systems Programming, and it does not seem
to be leaving us anytime soon. In its core it is a simple language, yet a
powerful tool.
But over the years it has become infamous because of how easy it is to misuse
the language and its standard library.

## Why Coding Guidelines?

The aims of this set of guidelines center around **managing the error-proneness**
of the language and its standard library, working towards a codebase free from UB
(undefined behaviour). Aswell as **providing a common framework for programmers**
to work and reason about the codebase.
With the hope of making projects' codebase easier to manage, easier to interact with,
and increasing the productivity of the project members.
While imposing reasonable restriction.

This guidelines are a **constant work in progress** and any rule may be changed,
dropped, or added as the project members see fit. Of course, given proper justification
(and alternatives if needed).

Note: **this does not, and does not attempt to, replace other best practices** like
testing and static analysis. And rather boosts the efficiency of other practices
like code-reviews and peer-programming by giving a set of bullet points to
look-out for when interacting with the codebase.

## Table of Contents

Section
--- |
[Goals](#goals-of-the-coding-guidelines)
[C Version](#c-version)
[Formatting](#formatting)
[Header Files](#header-files)
[Scoping](#scoping)
[Functions](#functions)
[Error Handling](#error-handling)
[Macros](#macros)
[C Features](#c-features)
[Doxygen](#doxygen)
[Exceptions to the Rules](#exceptions-to-the-rules)
[TBD](#to-be-determined)
[References](#references)

## Goals of the Coding Guidelines

- **The presence of a rule must outweigh its absence**: It's benefit must be large
enough to justify its usage. Every rule must have a reason to be. The benefit
is measured with the problems that would arise with its absence.

- **Optimizing for the reader, not the writer**: The code is read fare more
often than it is written. It is preferable to enhance the reading, maintaining,
and debugging experience.

- **Avoiding dangerous practices**: As mentioned
[previously](#coding-guidelines-for-the-c-programming-language), it is easy to misuse
the C programming language and its standard library, ending up with security, safety,
and overall unpleasent bugs worth ours of work. This guidelines try to push for
type and memory safety practices, among other "good" practices.

- **Optimizing when necessary**: Performance optimizations can sometimes be
necessary and appropriate, even when they conflict with other principles of
this document. Justifications must accompany this optimizations (including measurements
of the obtained benefit/s).

## C Version

The project code should be **C17 compliant** (also refered as C18 sometimes).
It is fully supported as of GCC 8.2 and Clang 6 (versions old enough for anyone
to have availability in their prefered operating system).

## Formatting

Spaces, tabs, bracket placing, are tedious enough to have in mind when trying
to solve a difficult problem. Formatting is left to tooling. This project uses
[Clang Format](https://clang.llvm.org/docs/ClangFormat.html). Which is a big enough
tool to have integrated support on most text editors and IDEs out there.

**TBD**: Specific styling by clang format (default -llvm-, some other styling
offered by clang-format, or customized styling)

## Header Files

### Header file associated implementation files

Every .h file should have at least one .c file that implements it.

### Header Guards

The header files should have #define guards to prevent multiple inclusion. The format
of the symbol name should be \<PATH\>_\<FILE\>\_H\_, replacing the folder separator
in PATH with a lower dash '\_'. For example, the file foo/bar/baz.h should have
the following guard:

```C
#ifndef FOO_BAR_BAZ_H_
#define FOO_BAR_BAZ_H_

...

#endif // FOO_BAR_BAZ_H
```

### Include Everything You Use

If a source or header file refers to a symbol defined elsewhere, the file should
directly include the header file which provides the declaration or definition
of that symbol.

Do NOT rely on transitive inclusions. This allows people to remove no longer
needed #include statements from their headers without breaking clients. This applies
to related headers; foo.c should include bar.h if it uses a symbol from it even
if foo.h includes bar.h (no double inclusion will happen, bar.h has a header guard).

```C
// ===== foo.h =====
#ifndef FOO_H_
#define FOO_H_

#include "bar.h"

...

#endif // FOO_H_

// ===== foo.c =====

#include "foo.h"

... // other headers inclusion in proper order

#include "bar.h" // includes bar.h even if foo.h already includes it

... // rest of foo.c
```

### Forward Declarations

Avoid using forward declarations where possible. Include the headers instead.

**Definition:** a "forward declaration" is a declaration of an entity without
an associated definition

```C
void FooInB(); // forward declaration of function otherwise declared in B.h
int main() { return FooInB(); }
```

**Pros:**

- May save compile time, since the alternative #include, would force the compiler
to open the included file (more files opened simultaneously).

**Cons:**:

- The forward declared entity, may change in the header file, but
the forward declaration will not change along with it.
- Tooling may have a difficult time finding declarations.

**Prefered approach**: Include the header file.

```C
#include "B.h"
int main() { return FooInB(); }
```

### Inline Functions

Define functions inline only when they are small. Reference length: 10 lines or fewer.

**Definition:**: You can declare functions in a way that allows the compiler to
expand them inline rather than calling them through the usual function call mechanism.

**Pros:**

- May lead to better efficiency when the function is small (bigger functions
may end-up in instruction cache-thrashing, depending on the size of the function
and the cache size).
- Type safe alternative to preprocessor macros.

**Cons:**

- Bigger code size (limited in embedded systems).

**Prefered Approach**: Definitly prefered over macros since inlined functions
retain source code view between programmer, tooling, and compiler. Tooling like linters,
static-analysis and formatters, may not perform macro expansion. Making it
impossible for some of those tools to catch programmer errors in the usage of
macros (such as, but not only, type errors).

### \#include Style

The prefered #include order is:

1. Associated header file
1. C system header files
1. Other libraries headers
1. Project headers.

With each grouping separated from the next one by an empty line.

## Scoping

### Global Variables

#### Definition of Global Variables

Defined in one and only one implementation file. Usage of extern declaration in
associated header file.

**Cons:**

- Initialization of static and global variables in different files has an
undefined order.

**Pros:**

- It is the method supported by C (not a convincing pro but meh).

#### Avoidance of Global Variables

Overall, avoid polluting the global namespace. Which in C, save for
[internal linkage](#internal-linkage), and local variables, it's the only namespace.

**Pros:**

- Easy to reason about functions that do not depend on globally shared
resources (pure functions).
- Unpolluted global namespace guards against linkage errors.
- No race-condition ever happened on non-shared resources.

### Internal Linkage

When definitions outside of a .c file do not need to be referenced outside that file,
give them internal linkage by declaring them static. Do NOT use this construct
in .h files

### Local Variables

Initialize variables in the declaration (this is not C89).

```C
size_t i;
for (i = 0; i < n; ++i) // WRONG
  ...

for (size_t i = 0; i < n; ++i) // GOOD (existence of i is limited to the loop scope)
  ...

char k;
... // bunch of lines of code where k has an undefined value, accessing k is UB
k = '\0'; // WRONG

char k = '\0'; // GOOD
... // bunch of lines of code where k is '\0' :)
```

## Functions

Functions are the main abstraction of our C project. Simple and easy to reason
functions allow for testeable, safe, and composable code.

### Side Effects

Software is all about side effects. Programs must read input (files, networking,
sensors) and generate output, somehow. Otherwise the program is just heating
the processor.
However, the introduction of side effects (where access to global resources - including
variables - happen) complicates reasoning even in single threaded programs.
The change of a global variable in a function may affect several if statements
in many other functions, reading a file or requesting memory may throw an error.
And it becomes even worst when this resources become shared among threads.
Data races and synchronization quickly become common problems spreaded all over
the source code.

Simple functions, the kind that only compute output from given inputs, do not
access globally shared resources, and where the same input always results in
the same output (*pure functions*) are encouraged for testeability, safety, performance,
and ease of reasoning. Since side effects are a must to make something useful,
encapsulation of side effects in their own abstractions is encouraged.

Many rules in this guideline center around the desirable code properties explained
in this section.

### Inputs and Outputs

The output of a function is provided via a return value, and sometimes via
output parameters. There are also in/out parameters.

- Paramenter order: input parameters first, output parameters second, and in/out
parameters last.
- Prefer returning via return value.
- If output parameters are needed, then the name of the output variable in the
function declaration must be prefixed with '\_O'. That way the function
user can know what to expect from reading the declaration.
- If in/out parameters are needed, then the name of the in/out variable in the
function declaration must be prefixed with '\_I\_O'. That way the function user
can know what to expect from reading the declaration.
- Input parameters can be named according to the project stablished naming convention.
- Input parameters with pointer types, MUST be const qualified. This ensures
that the value pointed to is NOT going to change, as it is only input. Feel free
to const non pointer types aswell if your function does not plan to change the value
(more of a documentation manner, rather than an actual
visible effect by the caller, since the argument is copied).

Prefixes for both output and in/out parameters are purposely
tedious to write and clear to the reader.
Since the C language does not provide us with a built-in way to express this desired
functionality for the parameters, this tedious approach seems an appropriate
way to make this special parameters stand out above the crowd.

```C
// takes 2 input parameters by value and returns by value. const is optional here.
int multiply(const int x, const int y) { return x * y; }

// takes 1 input parameter by reference, it must be a pointer to a constant char.
size_t str_len(const char *str) {
  if (str == NULL) return 0;
  size_t length = 0;
  ... // length computation
  return length;
}

// takes 1 input parameter by reference and 1 output parameter.
void str_len_2(const char *str, size_t *_O_length) {
  if (str == NULL || _O_length == NULL) return;
  size_t length = 0;
  ... // length computation
  *_O_length = length; // used only as output
}

// takes 1 input parameter by reference and 1 in/out parameter.
void str_len_3(const char *str, size_t *_I_O_length) {
  if (_I_O_length == NULL) return;
  if (*_I_O_length != 0) *_I_O_length = 0; // used as input
  while (*str++ != '\0') ++*_I_O_length;
  // _I_O_length used as output. exits with the length of the string
}
```

### Passing by reference or by copy

Commonly, copying is perceived as an expensive operation, but passing by reference
(by pointer when talking C) has its costs. Passing by reference means that
the passed object must have an identity in memory. Sometimes that is desired
(ex.: objects representing devices). But many times it gets in the way and complicates
ease of reasoning over our code. It is very easy to reason about a variable that
is going to die at the end of the scope. Not so for a memory address that may be
used for *something else* outside of the function. And compilers also have this
problem of lack of contextual information.

What's more, copying is easy to deal with for the C programming language since,
unlike in C++, all types in C are what's called [POD types](https://en.cppreference.com/w/cpp/named_req/PODType).
Meaning their initialization, copy, and destruction is *trivial*.

Summarizing:

1. Passing by value allows for simpler, more declarative APIs and programming
style overall.
1. Automatic variables are easy to manage (automatic, duh).
1. Nowadays, compilers are good at optimizing passing by value
and returing by value (one example is [Copy Elision](https://en.cppreference.com/w/cpp/language/copy_elision)).
1. Even if there are no optimizations taking place, copying may not be as
expensive as you think it is.
1. Choose judiciously. If on dobut, inspect generated code and/or measure.

- video recommendation: [Modern C and What We Can Learn About It](https://youtu.be/QpAhX-gsHMs)

### Write Short Functions

Prefer small and focused functions. If possible, pure functions as described
[previously](#side-effects). Transform inputs into outputs, avoid depending
on global resources. Functions constructed this way, are easy to reason about
and to test.

If your functions end up big and complex, consider refactoring into several
simpler functions.

Function inlining is the compilers' most powerful optimization. Whenever possible,
function inlining will happen. And function inlining likes short, easy to reason
functions quite a bit. It is **wrong** to avoid having too many small functions
because of "too many function calls".

It is foolish to think that this can be followed on every case. The eventual
big function with loads of side effects will show up. But it is far better for those
(hard to tame) beasts to be isolated cases.

- video recommendation: more on the interaction between memory and function
abstractions in this great [talk by Chandler Cnuth](https://youtu.be/FnGCDLhaxKU)

### Array downcasting to pointer

When C arrays are used as function arguments, these are downcasted to pointers.
With no access to length attributes by any mean, this can easily end up in a nice
SEGFAULT. If the project counts with data structures that provide a safer alternative,
consider using that, otherwise, pay particular care or look for alternatives in your
function interface (adding a parameter for length, is just as easy to misuse for
the distracted caller).

## Error Handling

## Macros

Macros are very useful tool in the C programming language. However, macros
change the source code such that what the programmer sees is not what the compiler
sees (since compiling happens after macro-expansion).

Being the C language, with it's lack of support of parametric or ad-hoc polymorphism
that provide generic programming, macros are often the goto alternative.
Use marcros judiciously, think of macros carefully, and use them only when
there is truly no cleaner alternative to achieve the goal.

### Macro Naming

- Unsafe macros (macros with side effects): All uppercase.
- Constant values: All uppercase.
- Function-like macros: All lowercase.
- Macro as inline version of an already existing function: Same name as the
function but uppercasing.

### Global Constants

It is prefered to define global constants as const qualified varialbes (saddly
there is no constexpr in C). This retains type-safety. The global variable has
a well defined type, unlike a macro. Performance-wise there is no meaningful difference
(if there is a difference at all).

```C
// silly example demonstrating the type-safety issue with macros

#define loop_counter_const 10;
const unsigned loop_counter = 10;

int main() {
  for (int i = 0; i < loop_counter_const; ++i) // BAD: the compiler totally
    ;                                          // misses a type mismatch comparisson

  for (int i = 0; i < loop_counter; ++i) // GOOD: the compiler catches
    ;                                    // the mismatching int and unsigned comparisson
}
```

By the way, constant global strings are defined as either
`const char *const hello_world = "Hello, World!\n"` or
`const char hello_world[] = ...`. The pointer example reads from
right to left 'a constant pointer to a char that is constant'. More
on the usage of const in [const Usage section](#const-usage)

### Parenthesis for argument usages

This is a common one. Make sure to use parenthesis when using arguments
of a function like macro so that the macro can behave as expected.

```C
#define mul (x, y) x * y // WRONG. Calling mul(2+5, 3) expands to 2+5*3

#undef mul

#define mul (x, y) (x) * (y) // GOOD. Calling mul(2+5, 3) expands to (3+5)*3
```

### *do { ... } while (0)*

For macros containing multiple statements, make sure to enclose them with
do..while(0) constructs, so that the macro can behave as expected on if, while,
and for loops.

```C
// ===== BAD example =====
#define some_macro(x, y)      \
  variable = (x) + (y);       \
  (y) += 2;

bool some_predicate() { return true; }

int foo ()
{
  int a = 0;
  int b = 1;
  if (some_predicate() == true)
    some_macro(a, b);
    // (y) += 2 is outside of the if statement after macro expansion
}

// ===== GOOD example =====
#define some_macro(x, y)      \
{                             \
  variable = (x) + (y);       \
  (y) += 2;                   \
} while (0)

int foo() 
{
  int a = 0;
  int b = 1;
  if (some_predicate() == true)
    some_macro(a, b); // the whole macro is inside the if statement after macro expansion
}
```

### Conditional Compilation

Wherever possible, don't use preprocessor conditional in .c files. This
makes the horder so much harder to read and follow the logic. Instead,
use these conditionals in a header file defining functions for use in those .c files,
providing no-op stub versions in the #else case. That way, functions can be called
unconditionally from .c files. The compiler will avoid producing any code for the
stub functions of the #else branch anyways, but the logic will be much easier
to read. This also helps on isolating conditional code in its own function.

## C Features

### Standard Library Usage

Avoid libc. It is famously unsafe, slow, and just a bad API design. It is understandable
for 1978, but we are aiming for modern C.

There are some useful parts of it:

- stddef.h and [stdint.h](#integer-types-and-stdinth).
- memcpy, memset, memmove
- math.h

It is discouraged libc usage specially on string manipulation functions
(anything with str). It is famously slow, non-intuitive, and hence unsafe to use.
Prefere our own project provided library for such cases.

### Asserts

*assert.h* header file has several utilities for regular asserts
and static asserts.

Asserts are a feature that is only active on debug builds. Asserts allow
checking certain conditions either at runtime (regular assert() calls)
or at compile time (static_assert() calls).

**Pros**:

- The use of assertions for function pre-conditions checking serves as great
documentation for future changes to the function.
- Tooling for static analysis commonly use asserts to improve their analaysis.
- Asserts are only included in debug builds.

**Prefered apporach**: Usage of assertions where the programmers' see fit is
highly encouraged.

### Integer Types and *stdint.h*

Of the built-in C integer types, only *int* and *unsigned* should be used.
The C standard leaves built-in integer types to be architecture and implementation
dependant.

It is important to know that **signed mixings with unsigned integers are
a programming error**.

1. Usage of *int* and *unsigned* is prefered when the actual size of the integer
type is not important (when you just want a signed or unsigned integer type no
matter its width).
1. For array length, loop indexing, byte sizes, size_t is a type able to represent
the size of any object in bytes. And widely use in libraries to represent sizes
and counts.
1. For the rest of use cases, *stdint.h* defines plenty of integer types with
a fixed width. Such as int8_t, int16_t, int32_t, int64_t, and their uint_
equivalents.
1. If size is critical, *stdint.h* provides with int_least8_t, int_least16_t, etc,
which ensure that no other integer type exists with lesser size for the
specified width.
1. If performance is critical, *stdint.h* provides with int_fast8_t,
int_fast16_t, etc, which ensure that the integer type is at least as fast as
any other integer type with at least the specified width.

**Pros:**

- Uniformity of declaration

**Cons:**

- Hopefully does not lead to misusage of the int_fastx_t and int_leastx_t.

### Type-casting

The C programming language does not provide generic programming capabilities
nor safer type-erasure alternatives.

The *type system* is our strongest tool to ensure a properly working codebase.
Casting loses original type information for both programmers and compilers.
void, char, uint8_t, and casting in general **is discouraged**. Usually these
castings are a dirty work-around or a quick-fix.

C allows for interesting casting tricks specially with pointers. And eventual need
for those tricks is a probability when working on a C project. But these tricks
MUST be centralized, in common libraries for the project.

**Pros:**

- Some interesting casting tricks will eventually show-up (but these have to be centralized).

**Cons:**

- Loses type-safety.
- Performance penalty (the common case after losing type information).

**Prefered approach**: Casting is the absolute last resource.
Behind macros even (some level of generic programming can be achieved with
macros if absolutely needed).

### *0*, *'\0'*, and *NULL*

- 0 for integer types.
- '\0' for char. '\0' provides type-safety over 0.
- NULL for pointers. NULL looks like a pointer to the reader, even if `NULL == 0`
behind the scenes.

### Designated Initializers

Usage of designated initializers is fine on certain cases. It allows us to
initialize POD (basically any data structure in C language) by naming its fields
explicitly. Every field, including nested PODs, not explicitly initialized in the
designated initializer, is *zero-initialized*. This is called Zero Is Initialization
in C standard (ZII).

The usage of designated initializers without naming struct fields is discouraged,
since it will require the reader to find the struct definition and check value orderings.

```C
typedef struct Point { float x, y, z; } Point;

typedef struct Foo { Point pa; Point pb; float a; } Foo;

typedef struct TenPoints { Point points[10]; } TenPoints;

int main() {
  Point p = { .x=0.0f, .y=1.0f }; // z is zero-initialized
  Foo f = { .pa.x=0.0f } // f.pa.y, f.pa.z, f.pb.x, f.pb.y, f.pb.z, and f.a are zero-initialized
  TenPoints tp = { .points[0].x=0.0f } // tp.points[0].y, tp.points[0].z, and every
                                    // other Point in TenPoints tp, is zero-initialized

  Point p1 = { 0.0f, 1.0f }; // BAD: initializes x to 0, y to 1, and z to 0,
                             // but it is not explicit about what fields are set
                             // and requires the reader to see the struct
                             // declaration to know what is happening
  ...

  return 0;
}
```

### Compound Literals

[Struct and array literals](https://en.cppreference.com/w/c/language/compound_literal)
are possible as of C11. Programmers may use it judiciously if they find this useful.

```C
typedef struct Point { float x, y; } Point;

void print_point(struct Point p) {
  printf("(x, y) = (%f, %f)", p.x, p.y);
}

int main() {
  print_point((Point){.x = 1.0f, .y = 2.0f }); // struct literal
  // array literals are possible too.
  //      example of array literal: (const char []){"Hello, World!\n"}
}
```

### C Keywords

Keywords often allow us to express explicitly in code the intent of certain constructs.
Keywords feed programmers and compiler with information to make better decisions
about our code.

But keyword misusage (looking at you *volatile*) can end up in miss-behaved programs
and confusing code.

#### *const* Usage

*const* is a variable qualification that indicates that certain variable is not
expected to change. The code will not compile if an attempt to change such value
is made.

- On function parameters, use const as described in [function arguments' section](#inputs-and-outputs).
- For global variables, prefer global const qualified variables to macros, as
described [here](#global-constants).
- Use *const* wherever it fits. If a variable (local, static, global, etc.) is
expected not to change, then it is sane for *const* to be used. This can be used
as documentation and avoid future unexpected changes.

```C
unsigned x = 0; // non-constant unsigned integer, it can change
const unsinged y = 0; // constant unsigned integer
const unsignez z = 0; // another constant unsigned integer

// if the const keyword is after the pointer '*', then the pointer
// is constant. if the const keyword is before the pointer '*', then the pointed
// type is what's constant. As shown in the following examples:

unsigned *const x_ptr = &x; // const pointer to non-constant integer
*x_ptr = 1; // OK: x_ptr points to non-constant integer, the value can change
x_ptr = &y; // ERROR: x_ptr is a constant pointer

const unsigned *y_ptr = &y; // non-constant pointer to constant unsigned integer
y_ptr = &z; // OK: y_ptr is a non-constant pointer, so it can change what it
            // points to
*y_ptr = 1; // ERROR: y_ptr points to a constant unsigned, the pointed value
            // cannot change

const unsigned *const k_ptr = &y; // constant pointer to a constant unsigned
k_ptr = &z; // ERROR: k_ptr is a constant pointer
*k_ptr = 1; // ERROR: k_ptr points to a constant unsigned integer
```

#### *restrict* Usage

One problem with functions expecting *different* pointers as arguments, is
that theese pointers might point to the same address. The function interface has
no way of telling the client of such pre-requisite (that these pointers must point
to different addresses). This lack of information also may force the
compiler to be conservative and generate code for the case where the pointers
point to the same address. That generated code would be just an unnecessary
burden if there was a way to explicitly tell the compiler and the function
clients, that pointers pointing to non-interleaved addresses are expected. This
is what the *restrict* keyword does.

As a clear example, take memcpy's signature from the man pages
`void *memcpy(void *restrict dest, const void *restrict src, size_t n);`
if dest and src are equal, or the range of addresses covered by dest+n and
src+n interleaves, this function has undefined behaviour. And that undefined behaviour
is made explicit by the usage of **restrict**. Now both function users and compiler,
know what's expected of this function, and any other usage is
*clearly a misusage from the client*.

#### *volatile* Usage

This is probably the most misused keyword.

*volatile* is a variable qualifier (just like *const*). Declaring a variable
as volatile (or having object with a nested volatile) is
*purposely underspecified* by the C standard. Accessing a volatile object is treated
as a visible side-effect (like IO or networking). This means that the compiler
cannot optimize away its usages.

```C
int c = 0;
volatile int c_volatile = 0;

void foo() {
  c = 1; // the compiler can optimize this away, since there is an assignment
         // c = 2 and c is not used between the 2 assignments
  c = 2;

  c_volatile = 1; // the compiler cannot optimize this assignment alway
                  // this is seen as having unknown side-effects
  c_volatile = 2;
  return;
}
```

#### Confusion of volatile with atomics

*volatile* qualified variables are still subject to race conditions on multithreaded
environments.

**Prefered approach**: *volatile* keyword is useful for constructs representing
an actual side-effect (is the whole reason why this keyword exists), such as
device registers.

#### *typedef* Usage

Typedef only in header files.

Typedef usages:

- Totally opaque objects (where the typedef is actively used to hide what the
object is).
- When you use sparse to literally create a new type for type-checking.
- typedef struct defined in headers.

Never typedef:

- Pointer types.

Regarding typedef struct definitions in header files, the typedef must have
the same name as the struct:

```C
typedef struct Point2 { float x, y, z; } NotPoint2; // WRONG, typedef is not
                                                    // the same as the struct name
                                                    // ("Point2" != "NotPoint2")

typedef struct Point { float x, y, z; } Point; // GOOD ("Point" == "Point")

```

Regarding the last item. This should be done only to struct types, never
typedef pointers

## Doxygen

## Exceptions to the Rules

### Pre-existing Code

If files of pre-existing code already follow some styling,
use the styling of the pre-existing code while working in those files.

## To be Determined

1. [Error Handling](#error-handling) (proposal: monadic error handling)
1. [Doxygen](#doxygen)
1. [Exceptions to the Rules](#exceptions-to-the-rules)
1. Naming conventions.

## References

- [Google C++ Style Guide](https://google.github.io/styleguide/cppguide.html)
- [LLVM Coding Style](https://llvm.org/docs/CodingStandards.html)
- [FreeBSD kernel Coding Style](https://manpages.debian.org/buster/freebsd-manpages/style.9freebsd.en.html)
- [Linux Kernel Coding Style](https://github.com/torvalds/linux/blob/master/Documentation/process/coding-style.rst)
- [Modern C and What We Can Learn About It](https://youtu.be/QpAhX-gsHMs)
