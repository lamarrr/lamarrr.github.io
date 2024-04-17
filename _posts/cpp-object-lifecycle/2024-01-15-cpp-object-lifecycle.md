---
title: RAII and The C++ Object Lifecycle
date: 2024-04-17 12:00:00 +00:00
modified: 2024-04-17 12:00:00 +00:00
tags: [c++, RAII, memory]
description: An analysis of the C++ Object Lifecycle
image: "/cpp-object-lifecycle/omen.jpg"
image_caption: Omen
---

Most Discussions around RAII/C++ Objects don't discuss the implicit contracts required to maintain the object's validity. These contracts are required when implementing your custom container types, working with custom memory allocators, tag discriminated unions (i.e. `Result<T, E>` and `Option<T>`, `std::variant<T...>`), etc.
These are typically termed as 'unsafe' operations as they do require an understanding of the Object lifetime invariants or lifecycle.
I would assume some basic familiarity with assembly as it is difficult to make sense of some of the article's experiments without them.

**NOTE**: We will not discuss exceptions nor the corner cases, unnecessary complexities, code path explosions, and limitations they introduce.

The lifecycle of a C++ Object is illustrated as:

```txt
  allocate placement memory
             ||
             ||                       ==============
             \/                       ||          ||
====>  construct object  =====> assign object <=====
||           ||                       ||
||           \/                       ||
====== destruct object  <===============
             ||
             \/
 deallocate placement memory
```

A violation of this lifecycle **WILL** lead to undefined behavior, typically: memory leak, double-free, uninitialized memory read/write, unaligned read/writes, `nullptr` dereference, out-of-bound read/writes, etc.

A rule of thumb I use for testing lifecycle violations in containers is to ensure the number of constructions is equal to the number of destructions. The types we will be using for demonstrating some of these concepts are defined as follows:

```cpp

struct Counter {
    uint32_t num_constructs = 0;
    uint32_t num_destructs = 0;

    void log() {
        printf("num_constructs = %" PRIu32 " \nnum_destructs =  %" PRIu32 "\n",
               num_constructs, num_destructs);
    }
} counter;

struct Obj {
    // default-construction
    Obj() { counter.num_constructs++; }
    // copy-construction
    Obj(Obj const& t) : data{t.data} { counter.num_constructs++; }
    // move-construction
    Obj(Obj&& t) : data{t.data} { counter.num_constructs++; }
    // copy-assignment
    Obj& operator=(Obj const& t) {
        data = t.data;
        return *this;
    }
    // move-assignment
    Obj& operator=(Obj&& t) {
        data = t.data;
        return *this;
    }
    // destruction
    ~Obj() { counter.num_destructs++; }
    uint32_t data = 1;
};


```

```cpp

struct Animal {
    virtual void react() = 0;
};

struct Cat : Animal {
    void react() override { printf("purr...\n"); }
};

struct Dog : Animal {
    void react() override { printf("woof!\n"); }
};

```

##### Allocate Memory

An object's memory **can** be sourced from the stack (i.e. `alloca`, `malloca`) or heap (i.e. `sbrk`, `malloc`, `kalloc`) and have some base requirements for objects to be placed on them:

- On successful allocation, the memory returned by allocators **MUST** be valid and not be already in use.
  This prevents catastrophic failures like double-free (double-object destructor calls).

**SEE**: GNUC's [`__attribute__((malloc(...)))`](https://gcc.gnu.org/onlinedocs/gcc/Common-Function-Attributes.html) and MSVC's [`__restrict`](https://learn.microsoft.com/en-us/cpp/cpp/restrict?view=msvc-170) return attributes which enables global aliasing optimizations for the compiler's reachability analysis.

**NOTE**: `malloc(0)` and `realloc(ptr, 0, 0)` are not required to return `nullptr` and is implementation-defined behavior. An implementation **MIGHT** decide to return the same or different non-null (possibly sentinel) memory address for a 0-sized allocation.

- General-purpose allocators **SHOULD** support at least alignment of `alignof(max_align_t)` where [`max_align_t`](https://en.cppreference.com/w/c/types/max_align_t) is mostly either `double` (8 bytes) or `long double` (16 bytes), as in the case of `malloc`. `max_align_t` is a maximum-aligned integral scalar type.

**NOTE**: C11 introduced [`aligned_alloc`](https://en.cppreference.com/w/c/memory/aligned_alloc) for over-aligned allocations (beyond `alignof(max_align_t)`) which is typically required for SIMD vector operations (SSE/AVX's 128-bit, 256-bit, and 512-bit extensions) as the SIMD's wide registers operate on over-aligned memory addresses. `MSVC`'s C runtime doesn't support `aligned_alloc` yet but provides [`_aligned_malloc`](https://learn.microsoft.com/en-us/cpp/c-runtime-library/reference/aligned-malloc?view=msvc-170) and [`_aligned_free`](https://learn.microsoft.com/en-us/cpp/c-runtime-library/reference/aligned-free?view=msvc-170).

#### Construct Object

This is where the lifecycle of an object begins. For non-trivially constructible types this implies a placement new of the object on the placement memory and for trivially constructible types, any memory write operation on the object's placement memory.

The object's placement memory address **MUST** be sized to _at least_ the object's size and the object placement address within the memory **MUST** be aligned to a multiple of the object's alignment. If an object is constructed at a memory location not properly sized for it, it can lead to Undefined Behavior (out-of-bound reads). Non-suitably aligned placement memory can lead to unaligned read & writes (undefined behavior, which on some CPU architectures can crash your application with a `SIGILL` or just lead to degraded performance).
Reading an uninitialized/non-constructed object is Undefined Behavior and catastrophic.

Placement-new serves some important purposes:

- initializing virtual function dispatch table for virtual (base and inherited) classes. (`reintepret_cast` + trivial construction i.e. `memset` or `memcpy` is not enough). i.e.
- initializing the class/struct, its base classes, and its members

Let's look at these in practice:

[_Godbolt_](https://godbolt.org/z/fq9KdP1eo)

```cpp
int * x = (int*) malloc(4);
(*x)++; // undefined behavior
```

The code above invokes undefined behavior due to an uninitialized read of an `int` at memory x.
With optimizations enabled, the compiler can aggressively decide to ignore the increment operation.

To fix:

[_Godbolt_](https://godbolt.org/z/fq9KdP1eo)

```cpp
int * x = (int*) malloc(4);
* x = 0;
(*x)++;
```

Because `int` is a trivially constructible type (i.e. no special construction semantics) with no invariants, it can be constructed simply by writing to the memory address, and an `int` "object" would implicitly exist at memory address `x`.
To construct an `int` or trivially constructible object at address `x` You can also use:

- `operator new`
- `memcpy`/`memmove`
- `memset`/`memset_explicit`

Now, let's take a look at a type with a more complex construction semantic (non-trivially-constructible):

[_Godbolt_](https://godbolt.org/z/Kn3bccore)

```cpp
Obj* obj = (Obj*) malloc(sizeof(Obj));
obj->data++; // undefined behavior, data is random value
printf("data: %" PRIu32 "\n", obj->data);
counter.log(); // num_constructs = 0, num_destructs = 0
```

From the log above, you can see the Object is never constructed at address `obj`, so, no object of type `Obj` exists at `obj` yet and it is undefined behavior to use/destroy the object in that state.
This could lead to a number of contract violations/undefined behavior like double-free, out-of-bounds reads/writes.

To fix:

[_Godbolt_](https://godbolt.org/z/1M58e85Mh)

```cpp
Obj* obj = (Obj*) malloc(sizeof(Obj));
new (obj) Obj{};  // constructs object of type Obj at the address
obj->data++;  // ok: data is increased from default value of 1 to 2
printf("data: %" PRIu32 "\n", obj->data);
counter.log();  // num_constructs = 1, num_destructs = 0
```

The placement new constructs the object of type `Obj` at address `obj`, and now contains valid member `data`.

Placement-new also serves to initialize the virtual function table pointers for the object to be usable in virtual dispatch.
The compiler's reachability analysis **MIGHT** decide an object doesn't exist at a memory address if it is not constructed with placement new and thus invoke undefined behavior. To illustrate:

[_Godbolt_](https://godbolt.org/z/aMMGe1n8o)

```cpp
Cat * cat = (Cat*) malloc(sizeof(Cat));
memset(cat, 0, sizeof(Cat));
cat->react(); // static dispatches to Cat::react()
Animal * animal = cat;
animal->react(); // undefined behavior
```

Calling `cat->react()`, correctly calls `Cat::react` via static dispatch. However with dynamic dispatch from its Base class method `Animal::react` via the type-erased call `animal->react()`, this would lead to undefined behavior (a segmentation fault if in debug mode or compiler's reachability analysis doesn't see the `memset`. Otherwise, the compiler **CAN** decide to simply ignore it given it is undefined behavior).

To examine why this happens, let's take a look at implementing implement a virtual class with a custom dynamic dispatch/v-table:

```cpp
struct Animal{
 void (*react)(void *);
};

struct Cat{
  Animal animal{
    .react = &react
  };
 static void react(void *);
};

```

For virtual dispatch to occur, the function pointer `Animal::react` would need to be called, but in the former example `Animal::react` would have been initialized to `0` by the `memset` call which is undefined behavior when `animal->react()` is invoked.

To fix our previous example, we would need to correctly initialize the implementation-defined virtual function dispatch table via the operator-new call, i.e:

[_Godbolt_](https://godbolt.org/z/z3rds6hPc)

```cpp
Cat * cat = (Cat*) malloc(sizeof(Cat));
new (cat) Cat{}; // initializes v-table
cat->react(); // static dispatches to Cat::react()
Animal * animal = cat;
animal->react(); // OK
```

The virtual function call `animal->react()` now correctly dispatches to `Cat::react`.

**NOTE**: The C++ standard doesn't specify how virtual dispatch/virtual function tables should be implemented, so there's no portable way to reliably manipulate the runtime's virtual function table.

Copy and Move construction implies the source address is already constructed with an object, and the destination address is a scratch memory containing uninitialized object/memory that needs to be initialized. Note that Copy and Move construction **SHOULD** not call the destructors of either the source or destination objects.

Object construction is also split into several categories, namely:

- [non-trivial construction](https://en.cppreference.com/w/cpp/types/is_constructible)
- [non-trivial copy construction](https://en.cppreference.com/w/cpp/types/is_copy_constructible)
- [non-trivial move construction](https://en.cppreference.com/w/cpp/types/is_move_constructible)
- [trivial construction](https://en.cppreference.com/w/cpp/types/is_constructible)
- [trivial copy construction](https://en.cppreference.com/w/cpp/types/is_copy_constructible)
- [trivial move construction](https://en.cppreference.com/w/cpp/types/is_move_constructible)

#### Assign Object

Copy and Move assignment requires that an object already exists at a memory address and we would like to assign another object to it. Meaning both the source and destination addresses contain valid initialized objects.
Object Assignment is split into several categories, namely:

- [copy assignment (`T& operator=(U const&)`)](https://en.cppreference.com/w/cpp/types/is_copy_assignable)
- [move assignment (`T& operator=(U &&)`)](https://en.cppreference.com/w/cpp/types/is_move_assignable)
- [trivial copy assignment](https://en.cppreference.com/w/cpp/types/is_copy_assignable)
- [trivial move assignment](https://en.cppreference.com/w/cpp/types/is_move_assignable)

Assignment being trivial means the objects can be assigned to another object without a special operation, this means it can be copied bytewise, i.e. via a `memcpy` or `memmove`.

#### Destruct Object

Destruction requires that a valid object exists at a memory location.
Destructing an object at a memory address implies that no object exists at that memory address and the memory is now left in an uninitialized state.
Unlike trivial constructions and assignments, trivial destruction implies a no-op.

- [non-trivial destruction (`~T()`)](https://en.cppreference.com/w/cpp/types/is_destructible)
- [trivial destruction](https://en.cppreference.com/w/cpp/types/is_destructible)

#### Deallocate Memory

Deallocating memory requires that any object on the placement memory has been destroyed. The memory is returned to its allocator and **SHOULD** no longer be referenced or used.

## Applications

#### Strict Aliasing, Dead-store, and Dead-load Optimizations

Strict aliasing is a very important contract that enables an aggressive compiler optimization called dead-store and dead-load optimizations

consider:

```cpp
struct A {
    int value = 0;
};

struct B {
    int value = 0;
};

A * a = get_A();
B * b = get_B();

a->value = 6;
b->value = 2;
return a->value;

```

Here, we first write to `a` and then to `b`, Consider that `a` might be a `reinterpret_cast` of `b`, then we wouldn't be able to assume the value of `a` is still 6 because there's a possibility both are pointing to either the same or different objects.
Whilst the implications of this at scale are non-obvious, it becomes drastic when the compiler's reachability analysis can't prove they are distinct objects.
Consider the reasonable contract that type `A` can not alias (`reinterpret_cast`) type `B` then we can always perform an optimization and assume both objects are different, therefore mutations to type `B` can not be observed from type `A`.
However, we would still need a backdoor in case we need to copy byte-wise from `a` to `b`, the exception to the contract being that `char`, `unsigned char`, and `signed char` can alias any object, otherwise encapsulated by [`std::bit_cast`](https://en.cppreference.com/w/cpp/numeric/bit_cast), this implies we can alias any object of any type from a `char`, `unsigned char`, or `signed char`, this is called [the strict aliasing rule](https://gist.github.com/shafik/848ae25ee209f698763cffee272a58f8).

To illustrate the strict aliasing rule, let's look at the generated assembly for:

[_Godbolt_](https://godbolt.org/z/M18z53b35)

```cpp
A * a = get_A();
B * b = get_B();

a->value = 6;
b->value = 2;
return a->value;

```

We can see from the example above that the compiler is able to perform a **Dead-Load Optimization** on the expression `a->value` and just assumes the value remains 6, which wouldn't have been possible if `a` could alias `b`.

However, if we **really**, **really**, needed to alias both types, we could use the strangely-named function `std::launder` which would interfere with the compiler's reachability analysis.

[_Godbolt_](https://godbolt.org/z/8rM6YbM75)

```cpp
A* a = get_A();
B* b = get_B();
B* a_b = std::launder<B>((B*)a);
a_b->value = 6;
b->value = 2;
return a->value;
```

From the generated assembly, the compiler is forced to perform the redundant load from `a_b`, because it _could_ be an alias of `b` because it's origin has been hidden by `std::launder`. Which is just like laundering money, hence, the name `:)`.
Some languages/dialects have a more aggressive form of this aliasing optimization/rule around mutability and aliasing, i.e. Rust's mutable reference (`& mut`) and [Circle's](https://github.com/seanbaxter/circle) mutable reference which requires only one mutable reference can be binded to an object at once, Which allow for more controversial and aggressive optimizations even across objects of the same type within a scope.
This is comparable to the non-standard `restrict` qualifier (GCC/Clang: `__restrict__`, and MSVC: `__restrict`).

To illustrate:

[_Godbolt_](https://godbolt.org/z/ahd6xT8Gx)

```cpp
int fn(A* a1, A* a2) {
    a1->value = 6;
    a2->value = 2;
    return a1->value;
}
```

As we said earlier, `a1` could alias/overlap with `a2` since they are the same type and there are no restrictions around mutability even within the same types, therefore the read expression `a1->value` would not optimized and we would still need to load the value, which would be redundant if we can ascertain that the objects do not in fact alias/overlap. Whilst the effect of this would likely go unnoticed on small objects, it would be noticeable on an array of multiple elements due to the data dependency and cause a drastic slowdown.
To optimize this we would use the `restrict` attribute, which implies objects with the attribute/qualifier do not alias other objects within that scope.

[_Godbolt_](https://godbolt.org/z/TK94KjTjx)

```cpp
int fn(A* RESTRICT a1, A* RESTRICT a2) {
    a1->value = 6;
    a2->value = 2;
    return a1->value;
}
```

#### Unions

Whilst most "modern C++" codebases would outright ban unions due to the difficulty of their constraints or how easy it is to create bugs with them, they remain an essential component of many data structures like `Option<T>`, `Result<T, E>`.

Unions are used in scenarios where one of multiple objects can exist at a location. Effectively giving room to constrained object dynamism/polymorphism.
Given only one object can exist at a union's address, the Object lifetime contracts still apply:

- At least one of the specified variants must exist in the union
- Any accessed object must have first been constructed
- Only one object must exist or be constructed in the union at a point in time, for another object to be constructed in the union, the previously constructed object must first have been destroyed.

Even though the variant types are possibly aliasing, the strict aliasing rules still apply to them, i.e. variant type `A` can not alias distinct variant type `B`.

i.e.:

[_Godbolt_](https://godbolt.org/z/jYMozMnTx)

```cpp
union Which {
    char c = 0;
    Cat cat;
};

void react(Animal* a) { a->react(); }

Which w;  // only c is initialized
react(&w.cat);  // SISGSEGV because we accessed `cat` without initializing it

```

To fix:

[_Godbolt_](https://godbolt.org/z/7G8s7vTP9)

```cpp
Which w;  // only c is initialized
// w.c.~char() - trivial, but char doesn't have a destructor
new (&w.cat) Cat{};  // now cat is initialized, we can access it
react(&w.cat);       // purr...
```

As you can see in the example above, we can't simply pretend to use the union's other variant, we need to maintain the object lifecycle, by first deleting `c` (trivial in this case, so no-op), then constructing `cat` using `operator new` (non-trivial) which would solve the UB by initializing the v-table for the `Cat` object.
This is a common footgun for C developers thinking C++ unions function similarly to C's.
Also, note that if the union contains non-trivial types, the construction, destruction, assignment, and move operations, need to be manually and explicitly implemented.

#### `std::aligned_storage` (deprecated in C++ 23)

Aligned Storage is meant to be a byte-wise representation of an object, with the object's lifetime context managed externally or determined by an external source of truth, thus still requiring the represented object's lifetime to be managed explicitly and correctly by the user, they work similar to unions but have the caveat that they are untyped.
Aligned storage is commonly used for implementing container types, specifically when containing both initialized and uninitialized objects, i.e. Open-Addressing (Linear-Probing Hashmaps), (ECS) Sparse Sets, Static-Capacity Vectors, Stack-allocated vectors, pre-allocated/bump/arena allocators.

**NOTE**: `std::aligned_storage` was [deprecated in C++23 (P1413R3)](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2021/p1413r3.pdf)

#### `Option<T>` (`std::optional<T>`)

`Option<T>` implies an object of type `T` may or may not exist, this means the object is either initialized or not initialized at the placement address and its existence is recognized by a discriminating enum/boolean.
Implementing `Option<T>` would require that the lifecycle of the value type `T` is maintained correctly. i.e. the number of constructions is same as the number of destructions, the object's constructor is called before being regarded as existing in the `Option`.

#### `Result<T, E>` (`std::expected<T, E>`)

`Result<T, E>` implies an object of type `T` or type `E` exists at the placement address of the `Result`, it is discriminated by an enum or boolean value.
Just like `Option<T>`, `Result<T, E>` maintains the lifecycle of the value type `T` and `E`.

#### Trivial Relocation

Trivial relocation is an upcoming C++ 26 Feature I'm really excited about that further extends the C++ object lifecycle and gives room for further optimizations.
Relocation is a combination of move from source object to uninitialized destination and destruction of the object left in the source (_destructive move<sup>TM</sup>_).

i.e:

```cpp
void relocate(A * src, A * dst){
    new (dst) A{ std::move(*src) };
    src->~A();
}
```

Trivial relocation implies the object can be safely moved from one memory address to another uninitialized memory address without invoking the object's move constructor and destructor, essentially capturing the "move to destination and destroy source" operation. This means we can instead use a bitewise copy, typically via `memcpy` or `memmove`, essentially being "trivial", as long as we don't treat the source memory address as containing a valid object after the relocation.

i.e:

```cpp
void trivial_relocate(A * src, A * dst){
    memcpy(dst, src, sizeof(A));
}
```

Note that trivial relocation doesn't always imply the move constructor and destructors are trivial.

i.e:

```cpp
struct MyStr {
    char* data_ = nullptr;
    size_t size_ = 0;
    MyStr() {}
    MyStr(char* data, size_t num) : data_{(char*)malloc(num)}, size_{num} {}
    MyStr(MyStr const&) = delete;
    MyStr& operator=(MyStr&&) = delete;
    MyStr& operator=(MyStr const&) = delete;
    MyStr(MyStr&& a) : data_{a.data_}, size_{a.size_} {
        a.data_ = nullptr;
        a.size_ = 0;
    }
    ~MyStr() { free(data); }
};

```

`MyStr` like many container types don't have trivial move constructors and destructors, but their object representation can be trivially relocated.
For small locally contained objects, non-trivial-relocation may not affect performance much as the compiler would typically be able to optimize the code generated by the move constructor and destructor, but for implementing generic container types like `std::vector` where a large number of these objects are frequently shifted around (i.e. during `push_back`, `insert`, move of elements from one container to another), trivial relocation (`memcpy`/`memmove`) would perform better than executing the non-trivial move constructor and destructor that would produce redundant operations, like the setting of `MyStr::num_` to `nullptr` and `MyStr::size_` to `0` (as in `std::vector<MyStr>` by `MyStr::MyStr(Mystr &&)`). This is a consequence of the C++ object model requiring move constructors to leave the source object in a _valid<sup>TM</sup>_ but _unspecified_ state for the destructor to still run correctly.
Also note that if your allocator supports `realloc`, trivial relocations means `grow`'ing your `vector` type's capacity could be collapsed into a zero-cost `realloc` (the Operating System would often just need to extend the allocation's entry if there's enough space within the page) rather than allocating a new separate memory, moving the objects to that memory, destroying the residual objects in the source memory and then free-ing the source memory.

This would end up extending our C++ object lifecycle to (from C++ 26):

```txt
  allocate placement memory
             ||
             ||
             ||   ====================================> relocate object
             ||   ||                                          ||
             ||   ||                  ==============          ||
             \/   ||                  ||          ||          ||
====>  construct object  =====> assign object <=====          ||
||           ||                       ||                      ||
||           \/                       ||                      ||
====== destruct object  <===============                      ||
             ||                                               ||
             \/                                               ||
 deallocate placement memory <==================================
```

- [P2786R0: Trivial relocatability options,
  Proposal for an alternative approach to trivial relocatability](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2023/p2786r0.pdf)
- [STL algorithms for trivial relocation](https://quuxplusone.github.io/blog/2023/03/03/relocate-algorithm-design/)
- [C++ Trivial Relocation Through Time - Mungo Gill - ACCU 2023](https://www.youtube.com/watch?v=DZ0maTWD_9g)
