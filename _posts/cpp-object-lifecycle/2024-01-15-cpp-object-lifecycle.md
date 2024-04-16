---
title: RAII and The C++ Object Lifecycle
date: 2024-04-10 12:00:00 +00:00
modified: 2024-04-10 12:00:00 +00:00
tags: [c++, RAII, memory]
description: An analysis of the C++ Object Lifecycle
image: "/cpp-object-lifecycle/omen.jpg"
image_caption: Omen
---

Most Discussions around RAII/C++ Objects don't discuss the implicit contracts required to maintain the object's validity. These contracts are required when implementing your custom container types, working with custom memory allocators, tag discriminated unions (i.e. `Result<T, E>` and `Option<T>`, `std::variant`), etc.
These are typically termed as 'unsafe' operations as they do require an understanding of the Object lifetime invariants or lifecycle.
I would assume some basic familiarity with assembly as it is difficult to make sense of most of the article's experiments without them.

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

A violation of this lifecycle **WILL** lead to undefined behavior, typically: memory leak, double-free, uninitialized memory read/write, unaligned read/writes, nullptr dereference, out-of-bound read/writes, etc.

A general rule of thumb for testing lifecycle violations in containers is to ensure the number of constructions is equal to the number of destructions, which is the core idea behind RAII.
The types we will be using for demonstrating some of these concepts are defined as follows:

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

The object's placement memory address **MUST** be sized to _at least_ the object's size and the object placement address within the memory **MUST** be aligned to a multiple of the object's alignment. If an object is constructed at a memory location not properly sized for it, it can lead to Undefined Behaviour (out-of-bound reads). Non-suitably aligned placement memory can lead to unaligned read & writes (undefined behavior, which on some CPU architectures can crash your application with a `SIGILL` or just lead to degraded performance).
Reading an uninitialized/non-constructed object is Undefined Behaviour and catastrophic.

Placement-new serves some important purposes:

- initializing virtual function dispatch table for virtual (base and inherited) classes. (`reintepret_cast` + trivial construction i.e. `memset` or `memcpy` is not enough). i.e.
- initializing the class/struct, its base classes, and its members

Let's look at these in practice:

```cpp
// https://godbolt.org/z/fq9KdP1eo
int * x = (int*) malloc(4);
(*x)++; // undefined behaviour
```

The code above invokes undefined behavior due to an uninitialized read of an `int` at memory x.
With optimizations enabled, the compiler can aggressively decide to totally ignore the increment operation.

To fix:

```cpp
// https://godbolt.org/z/fq9KdP1eo
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

```cpp
// https://godbolt.org/z/Kn3bccore
Obj* obj = (Obj*) malloc(sizeof(Obj));
obj->data++; // undefined behaviour, data is random value
printf("data: %" PRIu32 "\n", obj->data);
counter.log(); // num_constructs = 0, num_destructs = 0
```

From the log above, you can see the Object is never constructed at address `obj`, so, no object of type `Obj` exists at `obj` yet and it is undefined behavior to use/destroy the object in that state.
This could lead to a number of contract violations/undefined behavior like double-free, out-of-bounds reads/writes.

To fix:

```cpp
// https://godbolt.org/z/1M58e85Mh
Obj* obj = (Obj*) malloc(sizeof(Obj));
new (obj) Obj{};  // constructs object of type Obj at the address
obj->data++;  // ok: data is increased from default value of 1 to 2
printf("data: %" PRIu32 "\n", obj->data);
counter.log();  // num_constructs = 1, num_destructs = 0
```

The placement new constructs the object of type `Obj` at address `obj`, and now contains valid member `data`.

Placement-new also serves to initialize the virtual function table pointers for the object to be usable in virtual dispatch.
The compiler's reachability analysis **MIGHT** decide an object doesn't exist at a memory address if it is not constructed with placement new and thus invoke undefined behavior. To illustrate:

```cpp
// https://godbolt.org/z/aMMGe1n8o
Cat * cat = (Cat*) malloc(sizeof(Cat));
memset(cat, 0, sizeof(Cat));
cat->react(); // static dispatches to Cat::react()
Animal * animal = cat;
animal->react(); // undefined behaviour
```

Calling `cat->react()`, correctly calls `Cat::react` via static dispatch. However with dynamic dispatch from its Base class method `Animal::react` via the call `animal->react()`, this would lead to undefined behavior (a segmentation fault if in debug mode or compiler's reachability analysis doesn't see the `memset`. otherwise, the compiler **CAN** decide to simply ignore it).

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

For virtual dispatch to occur, the function pointer `Animal::react` would need to be called, but in the former example `Animal::react` would have been initialized to `0` by the `memset` call which is undefined behavior when `Animal::react` is invoked.

To fix our previous example, we would need to correctly initialize the implementation-defined virtual function dispatch table via the operator-new call, i.e:

```cpp
// https://godbolt.org/z/z3rds6hPc
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

Copy and Move assignment requires that an object already exists at a memory address and we would like to assign another object to it. Meaning both the source and destination addresses contain valid intialized objects.
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

Deallocating memory requires that any object on the placement memory has been destroyed. The memory is returned to its allocator and **SHOULD** no longer be referenced nor used.

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
Whilst the implications of this at scale is non-obvious, it becomes drastic when the compiler's reachability analysis can't prove they are distinct objects.
Consider the reasonable contract that type `A` can not alias (`reinterpret_cast`) type `B` then we can always perform an optimization and assume both objects are different, therefore mutations to type `B` can not be observed from type `A`.
However, we would still need a backdoor in case we need to copy byte-wise from `a` to `b`, the exception to the contract being that `char`, `unsigned char`, and `signed char` can alias any object, otherwise encapsulated by [`std::bit_cast`](https://en.cppreference.com/w/cpp/numeric/bit_cast), this implies we can alias any object of any type from a `char`, `unsigned char`, or `signed char`, this is called [the strict aliasing rule](https://gist.github.com/shafik/848ae25ee209f698763cffee272a58f8).

To illustrate the strict aliasing rule, let's look at the generate assembly for:

```cpp
// https://godbolt.org/z/M18z53b35
A * a = get_A();
B * b = get_B();

a->value = 6;
b->value = 2;
return a->value;

```

We can see from the example above that the compiler is able to perform a **Dead-Load Optimization** on the expression `a->value` and just assumes the value remains 6, which wouldn't have been possible if `a` could alias `b`.

However, if we **really**, **really**, needed to alias both types, we could use the strangely-named function `std::launder` which would interfere with the compiler's reachability analysis.

```cpp
// https://godbolt.org/z/8rM6YbM75
A* a = get_A();
B* b = get_B();
B* a_b = std::launder<B>((B*)a);
a_b->value = 6;
b->value = 2;
return a->value;
```

From the generated assembly, the compiler is forced to perform the redundant load from `a_b`, because it _could_ be an alias of `b` because it's origin has been hidden by `std::launder`. Which is just like laundering money, hence, the name `:)`.
Some languages/dialects have more aggressive form of this aliasing optimization/rule around mutability and aliasing, i.e. Rust's mutable reference (`& mut`) and Circle's mutable reference which requires only one mutable reference can be binded to an object at once. Which allow for more controversial and aggressive optimizations even across objects of the same type.
This is comparable to the non-standard `restrict` qualifier (GCC/Clang: `__restrict__`, and MSVC: `__restrict`).

To illustrate:

```cpp
// https://godbolt.org/z/ahd6xT8Gx

int fn(A* a1, A* a2) {
    a1->value = 6;
    a2->value = 2;
    return a1->value;
}

```

As we said earlier, `a1` could alias/overlap with `a2` since they are the same type and there's no restrictions around mutability even within the same types, therefore the read expression `a1->value` would not optimized and we would still need to load the value, which would be redundant if we can ascertain that the objects do not infact overlap. Whilst the effect of this would likely go unnoticed on small objects, it would be noticable on an array of multiple elements due to the data dependency and cause a drastic slowdown.
To optimize this we would use the `restrict` attribute, which implies objects with the attribute/qualifier do not alias other objects within that scope.

```cpp
// https://godbolt.org/z/TK94KjTjx
int fn(A* RESTRICT a1, A* RESTRICT a2) {
    a1->value = 6;
    a2->value = 2;
    return a1->value;
}

```

#### Unions

Whilst most "modern C++" codebases would outright ban unions due to the difficulty of their constraints or how easy it is to create bugs with them, they still remain an essential component of many data structures like `Option<T>`, `Result<T, E>`.

Unions are used in scenarios where one of multiple objects can exist at a location. Effectively giving room to constrained object dynamism.
Given only one object can exist at a union's address, the Object lifetime contracts still apply:

- At least one of the specified variants must exist in the union
- Any accessed object must have first been constructed
- Only one object must exist or be constructed on the union at a point in time, for another object to be constructed in the union, the previously constructed object must first have been destroyed.

Even though the variant types are clearly possibly aliasing, the strict aliasing rules still apply to them, i.e. variant type `A` can not alias variant type `B`.

i.e.:

```cpp
// https://godbolt.org/z/jYMozMnTx
union Which {
    char c = 0;
    Cat cat;
};

void react(Animal* a) { a->react(); }

Which w;  // only c is initialized
react(&w.cat);  // SISGSEGV beacause we accessed `cat` without initializing it

```

To fix:

```cpp
// https://godbolt.org/z/7G8s7vTP9

Which w;  // only c is initialized
// w.c.~char() - trivial, but char doesn't have a destructor
new (&w.cat) Cat{};  // now cat is initialized, we can access it
react(&w.cat);       // purr...
```

As you can see in the example above, we can't simply pretend to use the union's other variant, we need to maintain the object lifecycle, by first deleting `c` (trivial in this case, so no-op), then constructing `cat` using operator `new` (non-trivial).
This is a common footgun for C developers thinking C++ unions function similar to C's.
The v-table for the `Cat` object needs to be initialized, by using `operator new`.

#### `std::aligned_storage` (deprecated in C++ 23)

Aligned Storage is meant to be a byte-wise representation of an object, with the object's lifetime context managed externally or determined by an external source of truth, thus still requiring the represented object's lifetime to be managed explicitly and correctly by the user.
Aligned storage is commonly used for implementing container types, specifically when containing both initialized and uninitialized objects, i.e. Open-Addressing (Linear-Probing Hashmaps), (ECS) Sparse Sets, Static-Capacity Vectors, Stack-allocated vectors, pre-allocated/bump/arena allocators.

**NOTE**: `std::aligned_storage` was [deprecated in C++23 (P1413R3)](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2021/p1413r3.pdf)

#### `Option<T>` (`std::optional<T>`)

`Option<T>` implies an object of type `T` may or may not exist, this means the object is either initialized or not initialized at the placement address and its existence is recognized by a discrimating enum/boolean.
Implementing `Option<T>` would require that the lifecycle of the value type `T` is maintained correctly. i.e. number of constructions is same as the number of destructions, the object's constructor is called before being regarded as existing in the `Option`.

#### `Result<T, E>` (`std::expected<T, E>`)

`Result<T, E>` implies an object of type `T` or type `E` exists at the placement address of the `Result`, it is discriminated by an enum or boolean value.
Just like `Option<T>`, `Result<T, E>` maintains the lifecycle of the value type `T` and `E`.

#### Trivial Relocation

Trivial relocation implies the object can be safely moved from one memory address to another uninitialized memory address without invoking the object's move constructor and destructor. This means we can instead use a bitewise copy, typically via `memcpy` or `memmove`, essentially being "trivial", as long as we don't access the moved object's representation from the source memory address.

consider:

```cpp
struct Buff{
char * data_ = nullptr;
size_t num_ = 0;
Buff(){}
Buff(Buff const&)=delete;
Buff(Buff && b): data_{b.data_}, num_{b.num_} {
    b.data_ = nullptr;
    b.num_ = nullptr;
}
Buff& operator=(Buff const&) = delete;
Buff& operator=(Buff && b) {
    std::swap(data_, b.data_);
    std::swap(num_, b.num_);
    return *this;
}
char * data() {
    return data_;
}
char const * data() const {
    return data_;
}
size_t size() const {
    return num_;
}
};

```

When the object is moved from it still needs to be left in a valid state for its destructor to be called.

For `std::vector` and `std::string` types,

[P2786R0](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2023/p2786r0.pdf)
[STL algorithms for trivial relocation](https://quuxplusone.github.io/blog/2023/03/03/relocate-algorithm-design/)

#### Container Types
