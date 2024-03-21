# modern-c-features

## Overview

A collection of descriptions along with examples for C language and library features.

C17 contains defect reports and deprecations.

C11 includes the following language features:
- [generic selection](#generic-selection)
- [alignof](#alignof)
- [alignas](#alignas)
- [static_assert](#static_assert)
- [noreturn](#noreturn)
- [unicode literals](#unicode-literals)
- [anonymous structs and unions](#anonymous-structs-and-unions)

C11 includes the following library features:
- [bounds checking](#bounds-checking)
- [timespec_get](#timespec_get)
- [aligned_malloc](#aligned_malloc)
- [char32_t](#char32_t)
- [char16_t](#char16_t)
- [&lt;stdatomic.h&gt;](#stdatomic.h)
    - [Atomic types](#atomic-types)
    - [Atomic flags](#atomic-flags)
    - [Atomic variables](#atomic-variables)
- [&lt;threads.h&gt;](#threads.h)
    - [Threads](#threads)
    - [Mutexes](#mutexes)
    - [Condition variables](#condition-variables)
- [quick exiting](#quick-exiting)
- [exclusive mode file opening](#exclusive-mode-file-opening)

## C11 Language Features

### Generic selection
Using the `_Generic` keyword, select an expression based on the type of a given _controlling expression_. The format of `_Generic` is as follows:
```
_Generic( controlling-expression, T1: E1, ... )
```
where:
* `controlling-expresion` is an expression that yields a type present in the type list (`T1: E1, ...`).
* `T1` is a type.
* `E1` is an expression.

Optionally, specifying `default` in a type list, will match any controlling expression type.
```c
#define abs(expr) _Generic((expr), \
    int: abs(expr), \
    long int: labs(expr), \
    float: fabs(expr), \
    /* ... */ \
    /* Don't call abs for unsigned types, etc. */ \
    default: expr \
)

printf("%d %li %f %d\n", abs(-123), abs(-123l), abs(-3.14f), abs(123u));
// prints: 123 123 3.14 99 123
```



**`_Generic` and overloading**

```c
float minf(float, float);
int mini(int, int);

#define min(a,b) _Generic((a), float: minf(a,b), int: mini(a,b))
```



### alignof

Queries the _alignment requirement_ of a given type using the `_Alignof` keyword or `alignof` convenience macro defined in `<stdalign.h>`.
```c
// On a 64-bit x86 machine:
alignof(char); // == 1
alignof(int); // == 4
alignof(int*); // == 8

// Queries alignment of array members.
alignof(int[5]); // == 4

struct foo { char a; char b; };
alignof(struct foo); // == 1

// 3 bytes of padding between `a` and `b`.
struct bar { char a; int b; };
alignof(struct bar); // == 4
```

`max_align_t` is a type whos alignment is as large as that of every scalar type.

### alignas
Sets the alignment of the given object using the `_Alignas` keyword, or `alignof` macro in `<stdalign.h>`.
```c
struct sse_t
{
    // Aligns `sse_data` on a 16 byte boundary.
    alignas(16) float sse_data[4];
};

struct buffer
{
    // Align `buffer` to the same alignment boundary as an `int`.
    alignas(int) char buf[sizeof(int)];
};
```

`max_align_t` is a type whos alignment is as large as that of every scalar type.

### static_assert
Compile-time assertion either using the `_Static_assert` keyword or the `static_assert` keyword macro defined in `assert.h`.

Ex 1:

```c
#include <assert.h>

// ...

static_assert(sizeof(int) == sizeof(char), "`int` and `char` sizes do not match!");
```

Ex 2:

``` c
#include <assert.h>

int main)void {
    // Test if math works
    static_assert(2+2 == 4, "It works!"); // or _Static_assert(...
    
    // This will producer an error at compile time
    _Static_assert(sizeof(int) < sizeof(char), 
		"this program requires that int `int` is less than `char`");
}
```



### noreturn

Specifies the function does not return. If the function returns, either by returning via `return` or reaching the end of the function body, its behaviour is undefined. `noreturn` is a keyword macro defined in `<stdnoreturn.h>`.
```c
noreturn void foo()
{
    exit(0);
}
```

### Unicode literals
Create 16-bit or 32-bit Unicode string literals and character constants.
```c
char16_t c1 = u'貓';
char32_t c2 = U'🍌';

char16_t s1[] = u"a猫🍌"; // => [0x0061, 0x732B, 0xD83C, 0xDF4C, 0x0000]
char32_t s2[] = U"a猫🍌"; // => [0x00000061, 0x0000732B, 0x0001F34C, 0x00000000]
```

See: [char32_t](#char32_t), [char16_t](#char16_t).

### Anonymous structs and unions
Allows unnamed (i.e. _anonymous_) structs or unions. Every member of an anonymous union is considered to be a member of the enclosing struct or union, keeping the layout intact. This applies recursively if the enclosing struct or union is also anonymous.
```c
struct v
{
   union // anonymous union
   {
       int a;
       long b;
   };
   int c;
} v;
 
v.a = 1;
v.b = 2;
v.c = 3;

printf("%d %ld %d", v.a, v.b, v.c); // prints "2 2 3"
```
```c
union v
{
   struct // anonymous struct
   {
       int a;
       long b;
   };
   int c;
} v;
 
v.a = 1;
v.b = 2;
v.c = 3;

printf("%d %ld %d", v.a, v.b, v.c); // prints "3 2 3"
```



## C11 Library Features

### Bounds checking
Many standard library functions now have bounds-checking versions of existing functions. You can specify a callback function using `set_constraint_handler_s` as a _constraint handler_ when a function fails its boundary check. The standard library includes two constraint handlers: `abort_handler_s` which writes to `stderr` and terminates the program; and `ignore_handler_s` ignores the violation and continue the program.

Standard library functions with bounds-checked equivalents will be appended with `_s`. For example, some bounds-checked versions include `fopen_s`, `get_s`, `asctime_s`, etc.

Example of a custom constraint handler with `get_s`:
```c
void custom_handler_s(const char* restrict msg, void* restrict ptr, errno_t error)
{
    fprintf(stderr, "ERROR: %s\n", msg);
    abort();
}

set_constraint_handler_s(custom_handler_s);
char buffer[BUFFER_SIZE];
gets_s(buffer, BUFFER_SIZE);
printf("Entered: %s\n", buffer);
```
If a user enters a string greater than or equal to `BUFFER_SIZE` the custom constraint handler will be called.

### timespec_get
Populates a `struct timespec` object with the current time given a time base.
```c
struct timespec ts;
timespec_get(&ts, TIME_UTC);
char buff[BUFFER_SIZE];
strftime(buff, sizeof(buff), "%D %T", gmtime(&ts.tv_sec));
printf("Current time: %s.%09ld UTC\n", buff, ts.tv_nsec);
```

### aligned_malloc
Allocates bytes of storage whose alignment is specified.
```c
// Allocate at a 256-byte alignment.
int* p = aligned_alloc(256, sizeof(int));
// ...
free(p);
```

### char32_t
An unsigned integer type for holding 32-bit wide characters.

See: [Unicode literals](#unicode-literals).

### char16_t
An unsigned integer type for holding 16-bit wide characters.

See: [Unicode literals](#unicode-literals).

### &lt;stdatomic.h&gt;

#### Atomic types
The `_Atomic` type specifier/qualifier is used to denote that a variable is an _atomic type_. An _atomic type_ is treated differently than non-atomic types because access to atomic types provide freedom from _data races_. For example, the following code was compiled on x86 clang 4.0.0:
```c
_Atomic int a;
a = 1; // mov     ecx, 1
       // xchg    dword ptr [rbp - 8], ecx

int b;
b = 1; // mov     dword ptr [rbp - 8], 1
```
Notice that the atomic type used a `mov` then an `xchg` instruction to assign the value `1` to `a`, while `b` uses a single `mov`. Other operations may include `+=`, `++`, and so on... Ordinary read and write access to atomic types are _sequentially-consistent_.

See [this StackOverflow post](https://stackoverflow.com/a/26463282/3229983) for more information.

A variety of typedefs exist which expand to `_Atomic T`. For example, `atomic_int` is typedef'd to `_Atomic int`. See [this list](http://en.cppreference.com/w/c/atomic) for a more complete list.

Atomic types can also be tested for lock-freedom by using the `atomic_is_lock_free` function or the various `ATOMIC_*_LOCK_FREE` macro constants. Since `atomic_flag`s are guaranteed to be lock-free, the following example will always return `1`:
```c
atomic_flag flag = ATOMIC_FLAG_INIT;
atomic_is_lock_free(&flag); // == 1
```

#### Atomic flags
An `atomic_flag` is a lock-free (guaranteed), atomic boolean type representing a flag. An example of an atomic operation which uses a flag is test-and-set. Initialize `atomic_flag` objects using the `ATOMIC_FLAG_INIT` macro.

A simple spinlock implementation using `atomic_flag` as the "spin value":
```c
struct spinlock
{
    // false - lock is free, true - lock is taken
    atomic_flag flag;
};

void acquire_spinlock(struct spinlock* lock)
{
    // `atomic_flag_test_and_set` returns the value of the flag.
    // We keep spinning until the lock is free (value of the flag is `false`).
    while (atomic_flag_test_and_set(&lock->flag) == true);
}

void release_spinlock(struct spinlock* lock)
{
    atomic_flag_clear(&lock->flag);
}

void print_foo(void* lk)
{
    struct spinlock* lock = (struct spinlock*) lk;
    acquire_spinlock(lock);
    printf("foo\n");
    release_spinlock(lock);
}

void print_bar(void* lk)
{
    struct spinlock* lock = (struct spinlock*) lk;
    acquire_spinlock(lock);
    printf("bar\n");
    release_spinlock(lock);
}
```
```c
struct spinlock lock = { .flag = ATOMIC_FLAG_INIT };

// In Thread A:
print_foo(&lock);
// ==============
// In Thread B:
print_bar(&lock);
```

#### Atomic variables
An _atomic variable_ is a variable which is declared as an _atomic type_. Atomic variables are meant to be used with the atomic operations which operate on the values held by these variables (except for the flag operations, ie. `atomic_flag_test_and_set`). Unlike `atomic_flag`, these are not guaranteed to be lock-free. Initialize atomic variables using the `ATOMIC_VAR_INIT` macro or `atomic_init` if it has been default-constructed already.

The C11 Atomics library provides many additional atomic operations, memory fences, and allows specifying memory orderings for atomic operations.
```c
struct spinlock
{
    // false - lock is free, true - lock is taken
    atomic_bool flag;
};

void acquire_spinlock(struct spinlock* lock)
{
    bool expected = false;
    // `atomic_compare_exchange_weak` returns `false` when the value of the
    // flag is not equal to `desired`. We keep spinning until the lock is
    // free (value of the flag is `false`).
    while (atomic_compare_exchange_weak(&lock->flag, &expected, true) == false)
    {
        // `expected` will get set to the value of the flag for every call.
        // Reset it since we always "expect" `false`.
        expected = false;
    }
}

void release_spinlock(struct spinlock* lock)
{
    atomic_store(&lock->flag, false);
}

void print_foo(void* lk)
{
    struct spinlock* lock = (struct spinlock*) lk;
    acquire_spinlock(lock);
    printf("foo\n");
    release_spinlock(lock);
}

void print_bar(void* lk)
{
    struct spinlock* lock = (struct spinlock*) lk;
    acquire_spinlock(lock);
    printf("bar\n");
    release_spinlock(lock);
}
```
```c
struct spinlock lock;
atomic_init(&lock.flag, false);

// In Thread A:
print_foo(&lock);
// ==============
// In Thread B:
print_bar(&lock);
```

### &lt;threads.h&gt;
C11 provides an OS-agnostic thread library supporting thread creation, mutexes, and condition variables. Threading library is located in `<threads.h>`. As of September 2023, it is poorly supported in most major compilers.

#### Threads
Creates a thread and executes `print_n` in the new thread.
```c
void print_n(int* n)
{
    printf("%d\n", *n); // prints "123"
}

int n = 123;
thrd_t thr;
const int ret = thrd_create(
    &thr, (thrd_start_t) &print_n, (void*) &n);
thrd_join(thr, NULL);
```

#### Mutexes
C11 provides mutexes that support:
* Timed locking - Given a `timespec` object, blocks the current thread until the mutex is locked or until it times out.
* Recursive locking - Supports recursive locking (e.g. re-locking in recursive functions).
Recursive timed locks can also be created, capturing both these operations.

Specify which type of mutex to be created by passing any of `mtx_plain`, `mtx_timed`, `mtx_recursive`, or any combination of these to `mtx_init`'s `type` parameter.

```c
mtx_t mutex;
const int ret = mtx_init(&mutex, mtx_plain);

// In each thread: =============
mtx_lock(&mutex);
// Critical section
mtx_unlock(&mutex);
// ==============================

mtx_destroy(&mutex);
```

Mutexes must be cleaned up by calling `mtx_destroy`.

#### Condition variables
C11 provides condition variables as part of its concurrency library. Condition variables will block when waiting, support timed-waiting, and can be signalled and broadcasted. Note that spurious wakeups may occur.

Condition variables must be cleaned up by calling `cnd_destroy`.
```c
// Assume remove_from_queue, add_to_queue, and
// can_consume are defined.

mtx_t mutex;
// Initialize mutex...

cnd_t cond;
const int ret = cnd_init(&cond);

// In a consumer thread: =====
// Check in a loop due to spurious wakeups.
while (!can_consume && cnd_wait(&cond, &mutex));
remove_from_queue();
// =========================

// In a producer thread: =====
add_to_queue();
can_consume = true;
cnd_signal(&cond);
// ===========================

cnd_destroy(&cond);
```

### Quick exiting
Causes program termination to occur where clean termination is not possible or impractical; for example if cooperative cancellation between different threads is not possible or cancellation order is unachievable. This would have resulted in some threads trying to access static-duration objects while they are/have been destructed. 

Quick exiting allows applications to register handlers to be called on exit while being able to access static duration objects (unlike `atexit`). Quick exiting does not execute static-duration object destructors in order to achieve this.

Therefore, quick exiting can be thought of as being "in between" choosing to normally terminate a program, and abnormally terminating using `abort`.

```c
void f()
{
    // called first
}
 
void g()
{
    // called second
}
 
int main()
{
    at_quick_exit(f);
    at_quick_exit(g);
    quick_exit(0);
}
```

### Exclusive mode file opening
File access mode flag `x` can optionally be appended to `w` or `w+` specifiers. This flag forces the function to fail if the file exists, instead of overwriting it.
```c
FILE* fp = fopen(fname, "w+x");
if (!fp)
{
    // File either exists or there was an error
}
else
{
    // File opened successfully.
    fclose(fp);
}
```



## C23

Standard:

https://www.open-std.org/jtc1/sc22/wg14/www/docs/n3096.pdf



No one is going to rewrite a billion dollar libraries and software into Rust anyime soon.

https://thephd.dev/c23-is-coming-here-is-what-is-on-the-menu 

https://software.codidact.com/posts/289414

https://hackaday.com/2022/09/13/c23-programming-for-everyone/

https://www.youtube.com/watch?v=lLv1s7rKeCM

https://news.ycombinator.com/item?id=39436623

* http://thradams.com/cake/ownership.html



How to compile:

``` bash
$ gcc modern_c.c -std=C2x -Wall -Wextra -pedantic ./a.out
```

Options found on `info gcc`:

```bash
-std=c99
-std=c11
-std=c17	// Default
-std=gnu17	// Default
-std=c2x
```

GNU provides some extensions over versions. 

### TLDR;



Lists found on the internet:

- `#warning`, `#elifdef`, `#elifndef`
- `__VA_OPT__`
- `__has_include`
- decimal floating point
- arbitrary sized bit-precise integers without promotion
- checked integer math
- guaranteed two's complement
- `[[attributes]]`, including `[[vendor::namespaces ("and arguments")]]`
- proper keywords for `true`, `false`, `atomic`, etc. instead of `_Ugly` defines
- `= {}`
- lots of library fixes and additions
- `0b` literals, bool-typed `true` and `false`
- unicode identifier names
- fixes to regular enums beyond the underlying type syntax, fixes to bitfields



- enumerations improvements (forward declarations, underlying type specification).
- `nullptr`, much better than the `NULL` macro.
- better (but still anaemic) unicode support and `char8_t`.
- `auto` and `typeof`.
- `embed` will be so great but it will kill my [bcc](http://projects.malikania.fr/bcc) software as well :(.



----



### Types

* Decimal floating-point types (for `__STDC_IEC_60559_DFP__`)
  * `_Decimal32`
  * `_Decimal64`
  * `_Decimal128`
* Bit-prcise integers types
  * N: precision in bits
  * N cannot be greater than `BITINT_MAXWIDTH` from `<limits.h>`
  * `_BitInt(N)`
  * `unsigned _BitInt(N)`
* `bool` operation
* `nullptr`-constant and `nullptr_t`-type



### Constant

* Binary numeric constants with prefix `0b` or `0B` : `0b0010`
* Digit separator `'` for improved human readibility: 

    ```c
    int dec_val = 1'000;
    int hex_val = 0xFF'FF'FF'FF;
    ```



* `u8` **character constants** intended for UTF-8 

  ```c
  #define C1   'A'   // int
  #define C2 u8'A'   // char8_t
  #define EUR u8"\u20ac"
  ```

  

  ```c
  u8 my_character = u8'a'; // Guaranteed to be 'a' in UTF-8 encoding 
  printf("ASCII value of 'a': %d\n", my_character);
  ```

  * https://en.cppreference.com/w/c/language/character_constant

* `u8` **string literals** UTF-8, `char8_t` or `unsigned char`:

    ```c
    #include <stdio.h>
    
    int main() {
        // Example u8 string literal
        char8_t str_lit = u8"This is a u8 string literal";
    
        // Accessing characters in the string
        printf("First character: %c\n", (char) u8"Hello"[0]); // Cast to char for printing
    
        // String length (implementation-defined)
        size_t string_length = sizeof(u8"Hello"); // Might not be accurate
    
        // Printing the string (implementation-defined)
        printf("String: %s\n", u8"Hello"); // Might not be supported directly
    
        return 0;
    }
    ```

    * https://en.cppreference.com/w/c/language/string_literal



### Storage-class

#### `constexpr`

* **Compile-time evaluation:** The compiler can calculate the value of a `constexpr` variable at compile time, not during program execution (runtime).
  * Primarily focuses on pre-computing values and expressions solely based on compile-time information, ensuring safety and predictability. 
* **Potential for optimization:** This allows the compiler to directly substitute that value into your code, potentially leading to performance improvements.

* **Restrictions**

  * **Types:** `constexpr` variables must be of a type that can be evaluated at compile time. This excludes complex data structures, runtime-dependent values, etc.
  * **Initializers:** The initial value you provide to a `constexpr` variable must itself be a constant expression.
  * Valid operations:
    * Arithmetic operations (+, -, *, /)
    * Comparisons (==, !=, <, <=, >, >=)
    * Bitwise operations (&, |, ^, ~, <<, >>)
    * Casting between compatible types

**Why is `constexpr` valuable in C23?**

- **Performance:** Replacing certain calculations with pre-computed constants at compile time can improve performance.
- **Code readability:** `constexpr` makes your intentions clear to the compiler and to other developers.
- **Correctness:** Ensures that certain values are determined early and treated as fixed, potentially reducing errors.
- **Metaprogramming:** In more advanced use cases, `constexpr` enables certain kinds of template metaprogramming where calculations happen during compilation.


* https://en.cppreference.com/w/c/language/constexpr



### Attributes

* `deprecated`

* `fallthrough`

* `maybe`

* `nodiscard`

* `noreturn`

* `reproducible`

* `unsequenced`

* `unused`

* Any `attr` can also be written as `__attr__`

  ```c
  [[ deprecated ]] double fun (double);
  [[ __deprecated__ ]] double fun (double);
  ```



### Preprocessor

**Directives**

`#elifdef`

`#elifndef`

`#warning`

`#embed`

* Icons, pictures, etc...



#### Keywords

`static_assert`

`thread_local`

`true`

`false`



### Others

**Unnamed parameters in function definitions**

```c
int f(int, int); 	// declaration
int f(int a, int b) { return 7; }	// definition
int f(int, int) { return 7; }	// error until C23, OK since C23
```

* Interesting for legacy code or to maintain compatibility with functions treated as pointers
* Of course these passed parameters will not be used in the body of the function



**Goto labels followed by declarations**

**Decimal floating points**

* D

???



### Standard library

**Headers**

`<stdbit.h>`

`<stdckdint.h>`

* Set of functions where we have control for some operations to check boundaries ?

`<limits.h>`

`<math.h>`

`<stdint.h>`

* Macrio defining widths of the types defined in this header

`<stdio.h>`

* extensions for function families `fscanf` and ` fprintf`

`<string.h>` 

* `memccpy`
* `strdup`
* `strndup`

`<uchar.h>`

* char8_t
* mbrtoc8()
* c8rtomb()

`<stdatomic.h>`

* atomic_char8_t
* ATOMIC_CHAR8_T_LOCK_FREE

`<time.h>`

* timespec_getres()
* asctime_r()
* ctime_r()
* gmtime_r()
* localtime_r()






----



### Empty-Initialize 

``` c
Type name {0};	// C99 ?
Type name {};	// C23+
```

Works for:

* scalars
* arrays
* structs
* unions



``` c
struct data {
    const char* str;
    int value;
}

struct data my_sata = {};
dobule value = {};
double numbers[42] = {};

```



No need to use `memset` anymore.







### Selective-initialize

**Designated-initalizers: Arrays**

By index or subscript

```c
// Designated initializer
double numbers[42] = { [1] = 2, [5] = 6, [10] = 11 };
// Arbitrary order with index
double numbers[42] = {  [10] = 11 [1] = 2, [5] = 6};
// Positional initializer
double numbers[42] = { 0, 2, 0, 0, 0, 6, 0, 0, 0, 0, 11 };
// Arbitrary order with index
double numbers[42] = { 1, [10] = 1,  [5] = 5, [1] = 2, 3, 4};
```

* The rest is 0

Usecase:

``` c
#define CAN_LEN_MAX 128
typedef enum
{
    START_BIT,
    STOP_BIT = CAN_LEN_MAX - 1,
} CAN_t;
```





**Designated-initalize: Structs **

```c
typedef struct driver {
    const char* name;
    unsigned char data[16];
    uint16_t flags;
    uint16_t flags_ex;
    status_f status;
} driver_s;

driver_s my_serial = {"serial", {0}, 0, 0, &status_serial};
// {0}, 0, 0
```



```c
{.field_name = value }
```

```c
driver_s my_serial1 = { .name = "serial", .status = &status_serial };
// Ordered initializers without the subscript
driver_s my_serial2 = { "serial", .status = &status_serial };
// Arbitrary order with the subscript
driver_s my_serial3 = { .status = &status_serial, .name = "serial" };
// Extended option
driver_s my_serial4 = 
	{ .status = &status_serial, .name = "serial", .flags = 0x1234, 0x4312 };
/*
	name: 	|00|00|00|00|00|00|00|00| 	
	data: 	|00|00|00|00|00|00|00|00| 
			|00|00|00|00|00|00|00|00| 
	flags:	|34|12|21|43|00|00|00|00|	flags + flags_ex
	status	|50|11|40|00|00|00|00|00| 	X padding bytes
	
	Beware of padding into account!
*/

// TODO: What would happen if the extended option is greater than the padding?

```



**Nested initializers: nested structres**

```c
typedef struct flags {
    uint16_t standard;
    uint16_t extended;
}flags_s;

typedef struct driver {
    const char* name;
    unsigned char data[16];
    flags_s flags;
    status_f status;
} driver_s;

driver_s my_serial = {
// A. { ..., .flags = {0x1234, 0x4312} }; 		Nested positional
// B. { ..., .flags = {.extended = 0x4312} };	Nested designated
// C. { ..., .flags.extended = 0x431 };   		Super nested
```



**Mixed nested initialization:**

Nested subscript and field designators can be freely mixed:

```c
#define SIZE (42)

typedef struct data_s {
    char *id;
    double *values;
} data_s;
// ...
data_s arr[SIZE] = { [SIZE-1].id = "last data point"};
```



**Things C++ can not do:**

C wins in matter of flexibility

```c
driver_s my_serial = { .flags = {0x4321}, .name="" };	// C++ error: out of order
driver_s my_serial = { .flags = {0x4321}, 0 };			// C++ error: mixed
driver_s my_serial = { .flags.extended = 0x4321 };		// C++ error: nesting
double numbers[] = {[10] = 55};						// C++ error: No array design init
```



----

### Zeroing, re-assigning and ephemeral `lvalues`

K&R C (Old):

```c
typedef struct point {
	int x;
	int y;
} point_S;

point_s makepoint(int x, int y) {
	point_s temp;
	temp.x = x;
	temp.y = y;
	return temp;
}

point_s p = makepoint(42, 24);
```



C11:

```c
typedef struct point {
	int x;
	int y;
} point_S;

point_s makepoint(int x, int y) {
	point_s tmp = {.x=x, .y=y};
	return tmp;
}

point_s p = makepoint(42, 24);
```



Newest C:  **Compound literals**

```c
typedef struct point {
	int x;
	int y;
} point_S;

point_s makepoint(int x, int y) {
	return (point_s) {.x=x, .y=y};	// Compount literal
}

point_s p = makepoint(42, 24);
```



#### Compound literals

```c
// (type){ /*initializer list*/ }
int arr[N] = {0};
```

* Create unnamed objects of type `type` with static or automatic storage duration
* The unnamed object is an `lvalue`
  * You can assign to it
  * its address can be taken
  * `Type* ptr = &(Type) {/*initializer list*/}`
* Can be used to initialize structs and arrays in-place
  (compilers optimize them away, if needed)
* Can be used as function arguments (either by-value or by-pointer)
* Can enable features like default function arguments




**Storage:**

**Static storage**: For file-scope compound literals

```c
static const double* default_coeffs = (double[]){0.12, 0.32, 0.32, 0.12};
void filter(...){}
```

**Automatic storage**: For block-scope compound literals

```c
typedef struct fir4{
	const double* coeffs;
	double buffer[4];
} fir4_s;

void filter(size_t n, double arr[n]){
	fir4_s fir = { ... };
	// ...
	fir = (fir4_s){.coeffs=default_coeffs, .buffer={0}};	// Assignment
	// ...
}
```

* Assignment: Putting a new value into it



**Applications and usecases**

**Time:** Gets the UNIX time out of it!

* Not like C++ temporary objects! This is valid in C

```c
time_t time = mktime( &(struct tm){.tm_year=2021, .tm_mon=6, .tm_mday=1, .tm_isdst=-1});
```

**Initialization:**

* Compound literal + designated initializers
* `pa->count` and `pa->capacity` are zeroed automatically
* `[static 1]` : `pa` argument is checked against `NULL` 

```c
typedef struct array {
    double* data;
    size_t capacity;
    size_t count;
}array_s;

// Initialization function
array_s* array_init(array_s* pa, size_t capacity);

/* C90 */
array_s* array_init(array_s* pa, size_t capacity){
  	memset(pa, 0, sizeof(*pa));
    pa->data = calloc(capacity, sizeof(*pa->data));
  	
    if (pa->data)
  		pa->capacity = capacity;
	return pa;
}

/* Modern C*/
array_s* array_init(array_s pa[static 1], size_t capacity){
    *pa = (array_s){
        .data = calloc(capacity, sizeof(*pa->data))
    }; // Just "one line"
	
    if (pa->data)
  		pa->capacity = capacity;
	return pa;
};
```



**To zero:**

* Compiler will call `memset` 

```c
/* C90 */
void array_free(array_s* pa){
    if (pa && pa->data){
        free(pa->data);
    }
    
    memset(pa, 0, sizeof(*pa));
    
    return pa;
}

/* Modern C*/
void array_free(array_s pa[static 1]){
    if (pa && pa->data){
        free(pa->data);
    }
   	
    *pa = (array_s){0};
    
    return pa;
}
```



**As function arguments**

```c
/* C90 */
image_s* blur(image_s* img, size_t width, blur_type type, int cwh, _Bool in_place);
```

```c
/* Modern C */

typedef struct blur_params {
	size_t width;
    blur_type type;
    int compute_hw;
    _Bool in_place;
} blur_params_s;

image_s* blur(image_s img[static 1], blur_params_s params[static 1]);
```


If you have to pass more than three or 4 arguments, pass by `struct`. Pass the address of the param structure to the function.

```c
blur_params_s params = { .width=64, .type=box, .compute_hw=0, .in_place=true };
blur(&img, &params);
```

But this is still ugly, so...



The `struct` argument can be created in-place.

```c
image_s* blur(blur_params_s img[static 1], blur_params_s params);
blur(&img, &(blur_params_s){ .width=64, .type=box, .compute_hw=0, .in_place=true });
```

Still not readable...



Both passing by value and by pointer are ok (let the optimizer do its work)

```c
enum blur_type{ box, gauss };
image_s* blur(image_s img[static 1], blur_params_s params[static 1]);

blur(&img, &(blur_params_s){ .width=64, .type=box, .compute_hw=0, .in_place=true });
```



Some arguments can be even skipped - because compound `struct` literals
```c
enum blur_type{ box, gauss };
image_s* blur(image_s img[static 1], blur_params_s params);

blur(&img, (blur_params_s){ .width=64, .in_place=true });
```


The omitted arguments will be zero-initialized



**Function macros for default values**

Adding macro to improve the syntax:

```c
image_s* blur_(image_s img[static 1], blur_params_s params[static 1]);
#define blur(img, ...) blur_((img), &(blur_params_s){__VA__ARGS__})
```

* `...` : Variable number of arguments
* Using `__VA_ARGS__` for variable arguments

```c
blur( &img, .width=64, .in_place=true);
```

* `blur` macro takes an `img`
* and a variable number of arguments hidden behind `...`
* they are expanded in `blur_` function call with `__VA_ARGS__`
* during expansion a `blur_params_s` object is created in-place using those arguments



Arbitrary default values:

```c
enum blur_type{ box, gauss };
image_s* blur_(image_s img[static 1], blur_params_s params[static 1]);
#define blur(img, ...)\
		blur_((img), &(blur_params_s){.width=32, .type=gauss, __VA__ARGS__})

blur( &img, .width=64, .in_place=true);
```

* value of `width` is overridden from default 32 to 64
* `type` defaults to `gauss`
* this will trigger an `initializer override warning`
  * Warning can be silenced with PRAGMAS



The warning can be disabled temporarily. The `_Pragma` operator was introduced in C99 to allow `#pragmas` in macro expansion.

```c
image_s* blur_(image_s img[static 1], blur_params_s params[static 1]);

#define DEF_ARGS_ON \
_Pragma("GCC diagnostic push") \
_Pragma("GCC diagnostic ignored \"-Woverride-init\"")

#define DEF_ARGS_OFF \
_Pragma("GCC diagnostic pop")

#define blur(img, ...) \
DEF_ARGS_ON \
blur_((img), (blur_params_s){.width=64, .type=gauss, __VA_ARGS__}) \
DEF_ARGS_OFF

blur( &img, .width=64, .in_place=true );
```





##### `constexpr`

Illegal

```c
const int n = 5 + 4;
int purrs[n];
```

```c
const int n = 5 + 4;
int purrs[n] = { 0 }; // 💥
```

Correct

```c
constexpr int n = 5 + 4;
int purrs[n] = { 0 }; // ✅
```



----

### Arrays

**Static vs Dynamic**

``` c
/* Static */
#define CONST_SIZE 100
int numbers[CONST_SIZE] = {};

/* Dynamic Size */
size_t dyn_size = 100;
int *numbers = calloc(dyn_size, sizeof(int));	
```

* Instead of malloc, calloc zeroes the allocated block

**C23:`constexpr`**

Before C23

```c
#define CONST_SIZE 100	// Located at the header or .H file
// ...
int numbers[CONST_SIZE];
```

With C23

```c
constexpr size_t CONST_SIZE = {100};	// Located next to the array which is more natural
int numbers[CONST_SIZE];
```



An **Integral Constant Expression** is an expression that can be **evaluated at compile time**, and whose type is integral or an enumeration. The situations that require integral constant expressions include array bounds, enumerator values, case labels, bit-field sizes, static member initializers, and value template arguments.

```c
static const size_t CONST_INT = {100};
void func() {
    int numbers[CONST_INT]
}
```

* Define an array with a constant known size it's illegal
* It compiles but many people think it should not compile



####  VLA: Variable Length Arrays

**Summary**

| What             | Syntax                             | Advice                                                       |
| ---------------- | ---------------------------------- | ------------------------------------------------------------ |
| VLA variable     | `type arr[var_sz]`                 | * Do  not use<br />* Use arrays with known size              |
| VLA arguments    | `void func(size_t n, type arr[n])` | * To use instead of pointer arguments<br />* Communicates intent better |
| Pointers to VLAs | `type (*pArr)[n][n]`               | * To use instead of pointers<br />* Easier to read & write   |



**VLA: Variable Length Arrays** are arrays whose size is determined at **run time** or, in other words, not known at compile time. VLAs can only appear at block scope and in function prototypes.

```c
void filter1(size_t len) {
    // ok - block scope 
    double buffer[len];
}
// ok - function prototype
void filter2(size_t n, double arr[n], size_t len) {
    // ok - block scope 
    double buffer[len];
}

size_t size = 100;
// nope - file scope
double oh_no[size];

```

* Creates an array of unknown size in the stack

* Calculating the RSP register shift in runtime



VLAs need to be initialized after declaration. Cannot be initialized using the brace-enclosed list ` = {0};`

```c
void filter2(size_t n, double arr[n], size_t len) {
    double buffer[len];
    for (size_t i = 0; i < len; i++){
        buffer[i] = arr[i]
    }
}
```



**VLAs assembly overhead**

Calculating the RSP register shift in runtime

Static size variant: `clang -std=c11 -O0`

```assembly
test_known_size:
	push rbp
	mov rbp, rsp
	
	sub rsp, 48

	; init & sum calls
	add rsp, 48
	pop rbp
	ret
```

VLA variant: `clang -std=c11 -O1`

```assembly
test_vla:
	push rbp
	mov rbp, rsp
	
	mov rax, rsp
	lea rdx, [4*rdi + 15]
	and rdx, -16
	mov rcx, rax
	sub	rcx, rdx
	mov	rsp, rcx

	; init & sum calls
	mov rsp, rbp
	pop rbp
	ret
```



**VLA as arguments to functions**

The VLA syntax can be also used to pass arrays to functions!

The variable length argument (`size_t n`) must come before the VLA.

```c
#define MAX_FILTER_LEN (128)

status_e filter(size_t n, double *arr, size_t len){		// <<<
    if (len > MAX_FILTER_LEN)
        return status_fail;
    double buffer[MAX_FILTER_LEN];
    for (size_t i=0; i < len; ++i)
        buffer[i] = arr[i];
    // ...
}

```

Can be used to pass normal arrays

```c
int use_vla(size_t n, int *numbers) {
    return numbers[n/2];
}
```

```c
// Arrays with known size
int use_vla(size_t n, int numbers[n]) {
    return numbers[n/2];
}
```

Compiler can tell you there is a string operation overflow:  `[-Wstringop-overflow=]`

```c
int main() {
    int numbers[] = {1,2,3,4,5};
    return use_vla(6, numbers); // Only 5 elements!!!
}
```

* `[GCC 12] warning: 'use_vla' accessing 24 bytes in a region of size 20 [-Wstringop-overflow=]`

Also works with dynamically allocated arrays:

```c
int main() {
    int* pnumbers = malloc(sizeof(int[5]));
    // init pnumbers
    return use_vla(6, numbers); // Only 5 elements!!!
}
```



**Dynamic multidimensional arrays with VLA syntax**

* VLA declare strong array types (type checked at compile time)


Classical way:

``` c
double* kernel_gauss_create(size_t sz){
	double *pK = malloc(sizeof(double[sz*sz]));
 	if (!pK) return NULL;
  
    for (size_t i=0; i < sz; ++i)
    	for (size_t j=0; j < sz; ++j)
    		*(pK + i*sizeof(double[sz]) + j) = _gcoeff(sz, i, j);	// Hard to understand
    return pK;
}
```

* We write: `*(pK + i*sizeof(double[sz]) + j) = _gcoeff(sz, i, j);`

* We mean `pK[i][j] = _gcoeff(sz,i,j);` **VLA syntax**



Using VLA: The pointer-to-array declaration can contain variable size!

``` c
double* kernel_gauss_create(size_t sz){
	// double *pK = malloc(sizeof(double[sz*sz]));
 	double (*pK)[sz][sz] = malloc(sizeof(double[*pK]));
    // double (*pK)[sz] = malloc(sizeof(*pK) * sz); // is also possible
    
    if (!pK) return NULL;
  
    for (size_t i=0; i < sz; ++i)
    	for (size_t j=0; j < sz; ++j)
    		(*pK)[sz][sz] = _gcoeff(sz, i, j);
    return pK;
}
```

* `pK` pointer of two dimensional array of doubles
* Strongly typed == Statically typed = Types are checked at compile time 

A different variant for a better syntax: `*pK -> pK`. But some type safety is lost.

``` c
double* kernel_gauss_create(size_t sz){
	// double *pK = malloc(sizeof(double[sz*sz]));
    double (*pK)[sz] = malloc(sizeof(*pK) * sz);
    
    if (!pK) return NULL;
  
    for (size_t i=0; i < sz; ++i)
    	for (size_t j=0; j < sz; ++j)
    		pK[sz][sz] = _gcoeff(sz, i, j);	// Better syntax!
    return pK;
}
```

**C23 variant:** Showing off with VLA syntax: `typeof`

``` c
double* kernel_gauss_create(size_t sz){
    
    double (*pK)[sz] = malloc(sizeof( typeof(*pK)[sz] ));
    
    if (!pK) return nullptr; // NULL -> C23: nullptr
  
    for (size_t i=0; i < sz; ++i)
    	for (size_t j=0; j < sz; ++j)
    		pK[sz][sz] = _gcoeff(sz, i, j);	// Better syntax!
    return pK;
}
```

* `typeof(*pK)` : Array of doubles of `[sz][sz]`



------

### Pointers

`T array[] == T* array`

How we pass pointers in ModernC?

**Summary**

| What                                   | How                                                          | Result                                                       |
| -------------------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| A single object,<br />can be null      | `void func(Type* obj);`                                      | func is responsible for<br/>checking obj                     |
| A single object,<br />can not be null  | `void func(Type obj[static 1]);`                             | func is responsible for<br/>checking obj<br />Compilers might emit a warning |
| Multiple objects,<br />can be null     | `void func(Type arr[N]);`<br />`void func(size_t n, Type arr[n]);` | Communicates the intent<br/>clearly<br />Compilers might emit a warning |
| Multiple objects,<br />can not be null | `void func(Type arr[static N]);`<br />`void func(size_t n, Type arr[static n]);` | N is the minimum size<br />Compilers might emit a warning**Pointers to Arrays: `[static n]`** |

* With enough storage for `n` objects
* That not necessarily contains `n` objects

Scenario 1:

* Passing an array of an object
* Enough storage for `n` objects 
* Not necessarily valid objects.

```c
// VLA array syntax
void make_great(int n, char buffer[n]) {
    const char *str = "C is great, long live C!";
    while(--n > 0 && *str){
        *buffer = *str++;
    }
    *buffer = '\0';
}
```

Scenario 2:

* Passing an array of an object
* Enough storage for `n` objects
* Guaranteed to contain `n` valid objects
* `[buffer static n]`
  * Means that it is illegal not to have objects there!
  * Static communicates in the intent
  * Since C11, compilers can check not everything but some part of it
  * Not all compilers implement this
  * Potential flase sense of security since compiler support for static assertions and null pointers can vary.

```c
// VLA array syntax with static qualifier
void make_great( int n, char buffer[static n] ){	
    const char *str = "C is great, long live C!";
    while(--n > 0 && *str){
        *buffer = *str++;
    }
    *buffer = '\0';
}
```



**Pointers to Single objects: `[static 1]`**

If the pointer can be `NULL`, just use a pointer:

```c
time_t time( time_t *arg ){
    time_t result = /* ??? */;
    if (arg) 
        *arg = result;
    return result;
}
```



If the pointer must not be `NULL`, must be always valid, use `array [static 1]` notation:

```c
size_t strlen( const char str[static 1]) {
    size_t len  = 0;
    while( *str++ != '\0' ){
        ++len;
        return len;
    }
}
```



While the `[static n]` syntax might have been introduced with good intentions, its limited reliability and portability make it **less preferable** compared to **explicit null pointer checks** ( `if (pointer == NULL)`)and other safer alternatives for ensuring the validity of array elements and preventing potential runtime issues.



**Array parameters with static sizes: `[static CONST_KNOWN_SIZE]`**

`````c
pair_s solve_quadratic(double coeffs[static 3]){
    // ...
}

pair_s solve_quadratic(double coeffs[static 3]);
`````

* This syntax is only allowed in function prototypes
* It only works for sizes that are constant integer expressions
* It communicates the intent: func expects at least CONST_KNOWN_SIZE objects
* Compilers will often be able to check and emit a warning

```bash
[GCC 11]: argument 1 to 'char[static 1]' is null where non-null expected 
[-Wnonnull]
```



-----



### Memory

#### `memccpy`

```c
memccpy
```



#### Security Development Life-cycle

`memset_s, strcat_s, strcpy_s`



memcpy or memmove used since may be more efficient that strcpy ad they do not repeatedly check for NUL `'\0'` character (less true in modern processors)

* C11 **Annex K**



----



### `embed`





----

### Structures

**Structure definition in the return type directly**

``` C 
struct Ret{ struct {long n;}; } will_it_run( bool yes_or_no) {
    ...
}
```



**Flexible array member**

Less fragile dynamic memory management

* `struct string` has now a flexible array member `arr` that has to be set in runtime
* A flexible array member must be the last member of a `struct`
* `struct string` has an incomplete type
* It cannot be a member of another `struct`

``` c
typedef struct string{
    size_t sz_arr;
    size_t length;
    char arr[];		// Flexible array member
    // char* arr;
} string_s;			// sizeof(string_s) == 16
```

Issue to overcome:

```c
typedef struct string{
    size_t sz_arr;
    size_t length;
    char* arr;
} string_s;			// sizeof(string_s) == 24

string_s* mk_string(const char str[static 1]){
    // Memory allocation #1
	string_s *p = malloc(sizeof(*p)); 		// Size of the same register without the array
    // Check #1
    if (p){
        size_t len = strlen(str);
        *p = (string_s){
            .sz_arr = len + 1,
            .length = len,
    // Memory allocation #2
            .arr = malloc(len + 1)
        };
    // Check #2
        if (p->arr){
            memcpy(p->arr, str, len + 1);
        }else{
  	// Free #2
            free(p);
            p = NULL;
        }
    }
    // Return
    return p;
}

/*
Memory layout of string_s
| | | | | | | | |	size_t sz_arr; 8 bytes
| | | | | | | | |	size_t length; 8 bytes
|A|L|I|C|E|\0| | |	char* arr;	   8 bytes
*/
```



Goal:

* To initialize the array in the `struct` by using only one `malloc` 
* No need to check for NULL
* No need to `free`
* Only 1x `malloc` is needed
* No wasted space overhead: `sizeof("alice\0")` < `sizeof()`

```c


string_s* mk_string(const char str[static 1]){
    size_t len = strlen(str);
    
    // Memory alloc #1
	string_s *p = malloc(sizeof(*p)) + 		// Size of the same register without the array
    				 	 sizeof(char[len + 1])); 	// Size of the arrray    
    
    // Check
    if (p){
    // Initialize using compount literal
    	*p = (string){.sz_arr = len +1, 
                      .lenght = len
                     };
        memcpy(p->arr, str, len + 1);
    }
    
    // Return without freeing
    return p;
}

/*
Memory layout of string_s
| | | | | | | | |	size_t sz_arr; 8 bytes
| | | | | | | | |	size_t length; 8 bytes
|A|L|I|C|E|\0|		char* arr;	   6 bytes
*/
```



Forbidden:

``` c
// A: Arrays of struct string_s are forbidden
#define ARR_SZ[100]
string_s names[ARR_SZ];

// B: struct_s cannot be a member of another struct
struct person{
    string_s name;
    int id;
};

// C: Direct initialization of the flexible array member is not allowed
string_s str = {
    .sz_arr = 0,
    .length = 0,
    .arr = "alice"
};

```



**N3003**
"Two structs with the same tag name and content compatible, this allows generic data structures omit an extra typedef and make the following code legal:"

``` c
#include <stdio.h>
#include <stdlib.h>

#define Vec(T) struct Vec__##T { T *at; size_t _len; }

#define vec_push(a,v) ((a)->at = realloc((a)->at, ++(a)->_len * sizeof *(a)->at), (a)->at[(a)->_len - 1] = (v))
#define vec_len(a) ((a)._len)

void fill(Vec(int) *vec) {
    for (int i = 0; i < 10; i += 2)
        vec_push(vec, i);
}

int main() {
    Vec(int) x = { 0 }; // or = {} in C2x
    // pre C2x you'd need to typedef Vec(int) to make the pointers compatible and use it for `x` and in fill:
    // --v
    fill(&x);
    for (size_t i = 0; i < vec_len(x); ++i)
        printf("%d\n", x.at[i]);
}
```



"Allows to typedef the same struct twice in one translation unit. Your example is not very idiomatic as it uses macros that allows side-effects. Here is a safe way to use this new feature, where multiple headers may include Vec.h with the same i_val defined:"

```c
// Common.h
#define _cat(a, b) a ## b
#define _cat2(a, b) _cat(a, b)
#define MEMB(name) _cat2(Self, name)
#ifndef i_tag
#define i_tag i_val
#endif

// Vec.h
#define Self _cat2(Vec_, i_tag)

typedef i_val MEMB(_val);
typedef struct { MEMB(_val) *at; size_t len, cap; } Self;

static inline void MEMB(_push)(Self* self, MEMB(_val) v) {
    if (self->len == self->cap)
        self->at = realloc(self->at, (self->cap = self->len*3/2 + 4));
    self->at[self->len++] = v;
}
#undef i_tag
#undef i_val
#undef Self
```

```c
#define i_val int
#include "Vec.h"

#define i_val struct Pair { int a, b; }
#define i_tag pair
#include "Vec.h"

// THIS WILL BE FINE IN C23
#define i_val int
#include "Vec.h"

void fill_int(Vec_int *vec) {
    for (int i = 0; i < 10; i += 2)
        Vec_int_push(vec, i);
}

int main() {
    Vec_int iv = {0}; // or = {} in C2x
    fill_int(&iv);

    for (size_t i = 0; i < iv.len; ++i)
        printf("%d ", iv.at[i]);
}
```



#### `typeof`

GNU extension introduced in C23

``` c
int x;         /* Plain old int variable. */
typeof(x) y;   /* Same type as x. Plain old int variable. */
```

One famous example of macro relying on `typeof` is `container_of`.

``` c
#define max(a,b) \
  ({ typeof (a) _a = (a); \
      typeof (b) _b = (b); \
    _a > _b ? _a : _b; })

// This works on C but not on C++
#define max(a,b) \
  ({ __auto_type _a = (a); \
      __auto_type _b = (b); \
    _a > _b ? _a : _b; })
```

Using `__auto_type` instead of `typeof` has two advantages:

- Each argument to the macro appears only once in the expansion of the macro. This prevents the size of the macro expansion growing exponentially when calls to such macros are nested inside arguments of such macros.
- If the argument to the macro has variably modified type, it is evaluated only once when using `__auto_type`, but twice if `typeof` is used.

Example:

``` c
// Helper macro for getting the return type of a function at a given index
#define ReturnType(index) typeof(x[index](0))

// Example function signatures
int addOne(int x) {
    return x + 1;
}
float squareRoot(float x) {
    return x * x;
}
// Enumerated type for function indices
enum FunctionIndex {
    ADD_ONE,
    SQUARE_ROOT
};

// Declare an array of function pointers
// The functions in the array take an integer (int) or float argument and return an integer or float, respectively.
// The array is initialized with pointers to the functions addOne and squareRoot.
int (*x[2])(int) = {addOne, squareRoot};

int main() {
    // Using typeof to get the return type of the function at ADD_ONE index
    ReturnType(ADD_ONE) result1 = x[ADD_ONE](5);	// typeof (x[0](5))
    printf("Result of addOne(5): %d\n", result1);

    // Using typeof to get the return type of the function at SQUARE_ROOT index
    ReturnType(SQUARE_ROOT) result2 = x[SQUARE_ROOT](4.0);	// // typeof (x[1](4.0))
    printf("Result of squareRoot(4.0): %f\n", result2);

    return 0;  
}

```



https://gcc.gnu.org/onlinedocs/gcc/Typeof.html

#### `container_of`



### Generic selection or Function overloading

* Makes writing type-generic macros easy
* Can be nested for multiple selectors
* Suffers from all the macro-shortcomings (e.g. an expression cannot be another macro)
* Allows overloading both on type and number of arguments


Type definition:

``` c
typedef struct Point2D {
	double x, y;
} Point2D_s;

typedef struct Circle {
	Point2D_s center;
	double radius;
} Circle_s;

typedef struct Rectangle {
	Point2D_s center;
	double w, h;
} Rectangle_s;
```



Function overloading In C++:

``` c++
void scale(Circle_s* c, double scale){
c->radius *= scale;
}

void scale(Rectangle_s* r, double scale){
r->w *= scale;
r->h *= scale;
}
```

* But this does not work in C



**Naming approach**

````c
// void scale_circ(Circle_s* c, double scale){
void scale_circ(Circle_s c[static 1], double scale){	// C23
	c->radius *= scale;
}
// void scale_rect(Rectangle_s* r, double scale){
void scale_rect(Rectangle_s r[static 1], double scale){	// C23
	r->w *= scale;
	r->h *= scale;
}
````

* This is ugly and not really overloading



**`_Generic` selection macro**

``` c
#define funcOL(ctrl, funcOL)
_Generic( ctrl,	\
	T1 : expr1, \
	T2 : expr2, \
	T3 : expr3, \
	default: expr_def )
```

* It is a switch statement
  * `x`: Controlling expression
  * `T1, T2, T3`: Type names
  * `default`: Default case
* Works on types and switches based on types
* Heavily used in math library

Applied to our case:

````c
void scale_circ(Circle_s* c, double scale);
void scale_rect(Rectangle_s* r, double scale);

// Generic selection
#define scale(obj, scale)				\
	_Generic((obj),						\	// Function overloading "switch"
			 Rectangle_s* : scale_rect, \
             Circle_s* : scale_circ		\
	) ((obj), (scale))						// Parameters passing
   

void func(){
	Rectangle rect;
	Circle circ;
}

scale(&rect, 5.3);
scale(&circ, 3.5);

````



**Number of arguments overloading**

Goal:

``` c
void scale_circ_1p(Circle_s* c, double scale);
void scale_rect_1p(Rectangle_s* r, double scale);
void scale_rect_2p(Rectangle_s* r, double w_scale, double h_scale);

void func(){
	Rectangle rect1;
	Rectangle rect2;
}

// Number of arguments overloading
scale(&rect1, 5.3);			// scale using two arguments
scale(&rect2, 5.3, 3.5)		// scale using three arguments
```



The bad news:

`_Generic` does not work with macros for expressions! It does not expand with macros within it.

The good news:

You still can do generic selection for `scale2p` and `scale1p` depending on the argument type

```c
#define scale2p(obj, ...)         \
	_Generic( (obj),                \
		Rectangle_s* : scale_rect_2p  \
	)((obj), __VA_ARGS__)
 
#define scale1p(obj, ...)         \
	_Generic( (obj),                \
		Rectangle_s* : scale_rect_1p, \
		Circle_s* : scale_circ_1p     \
	)((obj), __VA_ARGS__)
```



This allows for a classic trick:

Select a name from a list of names

```c
#define INVOKE(_1, _2, _3, NAME, ...) NAME
#define scale(...) INVOKE(__VA_ARGS__, scale2p, scale1p,)(__VA_ARGS__)
```

Invoke takes a number of arguments and depending on the number of arguments you invoke will select different names



## Example: Re-factor

Old Style:

```c
#include <stdlib.h>
#include <stdio.h>

typedef struct vector {
	double * data;
    size_t capacity;
	size_t size;
} vector;

void dump_vec(vector const* const);
#define DEF_CAP (16)

int vec_create(vector ** vec, size_t cap){
	if (*vec != NULL)
		return -1;
	
	*vec = (vector *)malloc(sizeof(**vec));
	if (*vec == NULL)
		return -1;

	cap = cap == 0? DEF_CAP : cap;
	
	(*vec)->data = (double *)malloc(cap * sizeof(double));
	if ((*vec)->data == NULL){
		free(*vec);
		*vec = NULL;
		return -1;
	}
    
    // Initialization	    
    (*vec)->capacity = cap;
	(*vec)->size = 0;

	return 1;
}

int main(void){
    vector* vec = NULL;
    if (vec_create(&vec), 8);
    	printf("Created a vector\n\n");
    	dump_vec(vec);
    	free(vec->data);
    	free(vec);
	}
}
```



C23 style:



```c
#include <stdlib.h>
#include <stdio.h>

typedef struct vector {
	size_t capacity;
	size_t size;
	double data[];	// flexible array member
} vector;

void dump_vec(vector const* const);
// #define DEF_CAP (16)	-> constexpr

bool vec_create(vector ** vec[static 1], size_t cap){
	if (*vec != nullptr)
		return false;	// bool
	
    constexpr size_t def_cap = {16};	// Not #define anymore
    cap = cap == 0? def_cap : cap;
    
    // C23 can add the data member since it is a flexible array
    *vec = malloc(sizeof(**vec) + sizeof(cap * double[cap])); 

    // C23 Initialization
    if (*vec != nullptr)
		**vec = (vector){.capacity = cap};
	
	return *vec != nullptr;	// bool
}

int main(void){
    vector* vec = nullptr;
    if (vec_create(vec), 8);
    	printf("Created a vector\n\n");
    	dump_vec(vec);
    	free(vec);
	}
}
```

* vec_create return type: `int` to `bool`
* `vector ** vec[static 1]`
  * Now compiler will tell notify you if you are trying to pass a NULL
* `NULL` is not strongly typed but it is `nullptr`
* `#define` -> `constexpr` in line
* Initialization of `(*vec)`
  * `size` member will be automatically set to `0`
* 2 mallocs (structure, and data member) can be allocated at once
  * data member as flexible array member, goes at the end: `data[]`
  * No need to free anymore, nor in `dump_vec` nor `main`

