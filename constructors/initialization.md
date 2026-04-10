# C++ Initialization

## The Core Forms

There are fundamentally four syntactic forms of initialization in C++:

```cpp
int a;              // default initialization
int b = 42;         // copy initialization
int c(42);          // direct initialization
int d{42};          // direct-list-initialization (brace/uniform initialization, C++11)
int e = {42};       // copy-list-initialization (C++11)
```

Each of these follows different rules depending on the type being initialized.

---

## 01 -- Default Initialization

```cpp
int x;              // indeterminate value (garbage) -- NOT zero!
Widget w;           // calls Widget's default constructor

struct Pod {
    int m_a;
    double m_b;
};

Pod p;              // m_a and m_b are indeterminate (garbage)
```

**The rule:** For class types, the default constructor is called. For fundamental types (int, double, pointers, etc.) and POD types, the value is indeterminate -- it is NOT zero.

---

## 02 -- Value Initialization

Syntax: empty parentheses or empty braces.

```cpp
int x {};           // zero
int y = {};         // zero
int z();            // CAREFUL: this is a function declaration, not initialization!

double d {};        // 0.0
bool b {};          // false
int* p {};          // nullptr

Pod pod {};         // m_a = 0, m_b = 0.0 (all members zero-initialized)
```

**The rule:** For fundamental types, value initialization means zero-initialization. For class types with a user-provided default constructor, it calls that constructor. For class types without a user-provided constructor, it zero-initializes all members first, then calls the (possibly compiler-generated) default constructor.

This is the safest default. When in doubt, use `{}` to initialize.

### The most vexing parse

```cpp
Widget w();         // declares a FUNCTION named w that returns Widget
Widget w{};         // creates a default-value-initialized Widget

// This is why braces are generally preferred for initialization.
```

---

## 03 -- Zero Initialization

Zero initialization is not a syntax you write directly (except in a few cases). It's a step that happens as part of other initialization forms:

```cpp
// Explicitly zero-initialized contexts:
static int x;               // zero (static storage duration -> zero-init)
int* p = new int();         // zero (value-init with () on fundamental type)
int* q = new int{};         // zero (value-init with {} on fundamental type)

// Contrast:
int* r = new int;           // garbage! (default-init on fundamental type)
```

**Where zero initialization happens automatically:**
- Static and thread-local variables (before any other initialization)
- As part of value initialization for non-class types
- As part of value initialization for class types without user-provided constructors

```cpp
// Static storage: always zero-initialized before anything else
static int global_count;     // guaranteed 0
static Pod global_pod;       // guaranteed {0, 0.0}

void foo() {
    static int local_static;  // also guaranteed 0
    int local;                // NOT zero -- automatic storage, default-init
}
```

---

## 04 -- Copy Initialization

```cpp
int x = 42;
Widget w = other_widget;       // calls copy constructor
Widget w2 = Widget(42);        // constructs temporary, then copy/move (often elided)
```

**The rule:** The `=` in a declaration is NOT the assignment operator. It's copy initialization syntax. For class types, this calls the copy or move constructor (or the compiler elides the copy entirely since C++17).

The important difference from direct initialization: copy initialization considers `explicit` constructors:

```cpp
class Strict {
public:
    explicit Strict(int value) : m_value(value) {}
private:
    int m_value;
};

Strict a(42);       // OK: direct initialization, explicit is fine
Strict b{42};       // OK: direct-list-initialization, explicit is fine
Strict c = 42;      // ERROR: copy initialization, cannot use explicit constructor
Strict d = {42};    // ERROR: copy-list-initialization, cannot use explicit constructor
```

This is why `explicit` on single-argument constructors matters -- it prevents implicit conversions through copy initialization.

---

## 05 -- Direct Initialization

```cpp
int x(42);
Widget w(arg1, arg2);
Widget w2(other_widget);       // calls copy constructor (direct-init form)
```

**The rule:** Directly calls the matching constructor. `explicit` constructors are considered. This is the "just call the constructor" form.

---

## 06 -- Brace Initialization (List Initialization, C++11)

This is the "uniform initialization" that was supposed to simplify everything. It partially did, but introduced its own quirks.

```cpp
int x {42};                     // direct-list-initialization
int y = {42};                   // copy-list-initialization
std::vector<int> v {1, 2, 3};   // initializer_list constructor
Widget w {arg1, arg2};          // calls matching constructor
```

### Narrowing prevention (the main advantage)

Braces prevent narrowing conversions -- this is the single biggest reason to prefer them:

```cpp
int x(3.14);         // OK: silently truncates to 3 (narrowing)
int y{3.14};         // ERROR: narrowing conversion not allowed
int z = 3.14;        // OK: silently truncates (narrowing)

double d = 3.14;
int a(d);            // OK: narrowing, compiles silently
int b{d};            // ERROR: narrowing from double to int
```

### The std::initializer_list trap

This is the biggest gotcha with brace initialization. If a class has a constructor that takes `std::initializer_list`, braces will **strongly prefer** it over other constructors:

```cpp
std::vector<int> v1(5, 42);    // 5 elements, all 42: {42, 42, 42, 42, 42}
std::vector<int> v2{5, 42};    // 2 elements, 5 and 42: {5, 42}

// v1 calls vector(size_t count, const T& value)
// v2 calls vector(std::initializer_list<int>)

// This is a real source of bugs. The braces change the meaning entirely.
```

Another example:

```cpp
class Ambiguous {
public:
    Ambiguous(int a, int b) {
        std::cout << "two ints: " << a << ", " << b << std::endl;
    }

    Ambiguous(std::initializer_list<int> list) {
        std::cout << "init list with " << list.size() << " elements" << std::endl;
    }
};

Ambiguous x(1, 2);     // "two ints: 1, 2"
Ambiguous y{1, 2};     // "init list with 2 elements"
```

The initializer_list constructor wins whenever braces are used and the conversion is possible, even if another constructor is a "better" match by normal overload resolution rules.

### Empty braces

```cpp
class HasInitList {
public:
    HasInitList() {
        std::cout << "default ctor" << std::endl;
    }

    HasInitList(std::initializer_list<int> list) {
        std::cout << "init list ctor" << std::endl;
    }
};

HasInitList a{};      // "default ctor" -- empty braces call default ctor
HasInitList b({});    // "init list ctor" -- explicitly passing empty init list
```

Empty braces `{}` are special-cased to prefer the default constructor over an empty initializer_list.

---

## 07 -- Aggregate Initialization

Aggregates are structs/classes with no user-provided constructors, no private/protected members, no virtual functions, and no base classes (relaxed in C++17). They get special initialization rules:

```cpp
struct Point {
    double m_x;
    double m_y;
    double m_z;
};

Point p1 = {1.0, 2.0, 3.0};    // aggregate init
Point p2 {1.0, 2.0, 3.0};      // aggregate init (braces)
Point p3 {1.0, 2.0};           // m_z is value-initialized (0.0)
Point p4 {};                   // all zero
Point p5;                      // all garbage (default init)
```

### Designated initializers (C++20)

```cpp
Point p = {.m_x = 1.0, .m_z = 3.0};   // m_y is value-initialized (0.0)

// Must be in declaration order:
// Point bad = {.m_z = 3.0, .m_x = 1.0};  // ERROR in C++ (OK in C)

// Can skip members (they get value-initialized):
struct Config {
    int m_width = 800;
    int m_height = 600;
    bool m_fullscreen = false;
};

Config c = {.m_fullscreen = true};   // m_width = 800, m_height = 600 (defaults)
```

---

## 08 -- Member Initialization

Three ways to initialize class members, in order of precedence:

```cpp
class Widget {
public:
    // 3. Constructor initializer list (highest precedence)
    Widget()
        : m_count(42)         // this wins over the default member initializer
        , m_name("override")  // this wins too
        , m_data()            // value-initialized (empty vector)
    {
        // Assigning in the body is NOT initialization.
        // The member is already constructed by the time you get here.
        // m_count = 42;  // this would be assignment, not initialization
    }

private:
    // 2. Default member initializer (C++11) -- used if not in init list
    int m_count = 0;
    std::string m_name = "default";

    // 1. Default initialization -- used if no default member initializer
    //    and not in init list. For int, this means garbage.
    std::vector<int> m_data;  // default ctor called (empty vector)
};
```

**Critical rule:** Members are initialized in **declaration order**, not in the order they appear in the initializer list. The compiler may warn if these differ:

```cpp
class Dangerous {
public:
    // BUG: m_size is initialized BEFORE m_data because of declaration order,
    // even though m_data appears first in the init list.
    Dangerous(std::size_t size)
        : m_data(new int[size])  // initialized SECOND (despite being listed first)
        , m_size(size)           // initialized FIRST (declared first)
    {
    }

private:
    std::size_t m_size;   // declared first -> initialized first
    int* m_data;          // declared second -> initialized second
};

// In this case it works by accident. But if m_data's init depended
// on m_size, and the order were reversed, you'd read garbage.
```

---

## 09 -- new and Initialization

The initialization form you use with `new` matters:

```cpp
// Fundamental types
int* a = new int;       // default-init: garbage
int* b = new int();     // value-init: zero
int* c = new int{};     // value-init: zero
int* d = new int(42);   // direct-init: 42
int* e = new int{42};   // direct-list-init: 42

// Arrays
int* f = new int[5];    // default-init: all garbage
int* g = new int[5]();  // value-init: all zero
int* h = new int[5]{};  // value-init: all zero
int* i = new int[5]{1, 2};  // 1, 2, 0, 0, 0 (rest value-initialized)

// Class types
Widget* w1 = new Widget;     // default-init: calls default ctor
Widget* w2 = new Widget();   // value-init: may zero-init members first
Widget* w3 = new Widget{};   // value-init: may zero-init members first
```

The difference between `new Widget` and `new Widget()` is subtle. If Widget has a user-provided default constructor, they're the same. If Widget is a POD or aggregate without a user-provided constructor, `new Widget` leaves members as garbage while `new Widget()` zero-initializes them first.

---

## 10 -- Summary Table

| Syntax | Name | int | POD struct | Class with ctor |
|---|---|---|---|---|
| `T x;` | default-init | garbage | garbage members | calls default ctor |
| `T x{};` | value-init | 0 | all members zero | calls default ctor |
| `T x = {};` | copy-list-init | 0 | all members zero | calls default ctor |
| `T x(args);` | direct-init | value | N/A | calls matching ctor |
| `T x{args};` | direct-list-init | value (no narrow) | aggregate init | prefers init_list ctor |
| `T x = val;` | copy-init | value | N/A | no explicit ctors |
| `T x = {args};` | copy-list-init | value (no narrow) | aggregate init | no explicit init_list ctors |

---

## 11 -- Practical Recommendations

**Use `{}` for value initialization (zeroing):**
```cpp
int count{};            // 0, not garbage
double sum{};           // 0.0
Pod data{};             // all members zeroed
```

**Use `()` for constructors where initializer_list ambiguity exists:**
```cpp
std::vector<int> v(5, 42);   // 5 elements of value 42
// NOT: std::vector<int> v{5, 42};  // 2 elements: 5 and 42
```

**Use `{}` with explicit values to catch narrowing:**
```cpp
int x{some_double};    // compile error if narrowing
```

**Always use the member initializer list, never assign in the constructor body:**
```cpp
// Good: initialization
Widget::Widget(int val) : m_value(val) {}

// Bad: default-init then assignment (two steps, possibly wasteful)
Widget::Widget(int val) {
    m_value = val;  // this is assignment, not initialization
}
```

**Use default member initializers for sensible defaults:**
```cpp
class Config {
private:
    int m_width = 1920;
    int m_height = 1080;
    bool m_vsync = true;
    // Constructors only need to override specific members
};
```