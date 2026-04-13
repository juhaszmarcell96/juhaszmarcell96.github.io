# Struct Padding / Alignment

```cpp
// =============================================================================
//
// Rules on typical x86-64 (LP64) platforms:
//
//   a) Each member is aligned to its own natural alignment:
//      - char:      1 byte
//      - short:     2 bytes
//      - int/float: 4 bytes
//      - long/double/pointer: 8 bytes
//
//   b) The compiler inserts padding BEFORE a member if the current offset
//      is not a multiple of that member's alignment.
//
//   c) The struct's total size is padded at the END to a multiple of the
//      LARGEST member's alignment. This ensures arrays of structs keep
//      every element properly aligned.
//
// The takeaway for interviews: reordering members from largest to smallest
// (or grouping same-sized members) minimizes wasted padding.
// =============================================================================

// --- Bad layout: 12 bytes ---
//
// Offset  Member    Size   Padding before
// 0       a (char)  1      0
// 1       (pad)     3      3 bytes to align int to offset 4
// 4       b (int)   4      0
// 8       c (char)  1      0
// 9       (pad)     3      3 bytes tail padding (struct aligns to 4)
//                   ---
// Total:            12
struct BadLayout {
    char a;   // offset 0, size 1
    int b;    // offset 4, size 4  (3 bytes padding after a)
    char c;   // offset 8, size 1  (3 bytes tail padding)
};
// sizeof = 12

// --- Good layout: 8 bytes ---
//
// Offset  Member    Size   Padding before
// 0       b (int)   4      0
// 4       a (char)  1      0
// 5       c (char)  1      0
// 6       (pad)     2      2 bytes tail padding (struct aligns to 4)
//                   ---
// Total:            8
struct GoodLayout {
    int b;    // offset 0, size 4
    char a;   // offset 4, size 1
    char c;   // offset 5, size 1  (2 bytes tail padding)
};
// sizeof = 8

// --- More complex example: 24 bytes ---
//
// Offset  Member          Size   Padding before
// 0       a (char)        1      0
// 1       (pad)           7      7 bytes to align double to offset 8
// 8       b (double)      8      0
// 16      c (char)        1      0
// 17      (pad)           3      3 bytes to align int to offset 20
// 20      d (int)         4      0
// 24      (pad)           0      but wait -- struct alignment is 8 (from double)
//                                24 is already a multiple of 8, so no tail pad
//                   ---
// Total:            24
struct ComplexBad {
    char a;       // offset 0
    double b;     // offset 8   (7 bytes padding)
    char c;       // offset 16
    int d;        // offset 20  (3 bytes padding)
};
// sizeof = 24

// --- Reordered: 16 bytes ---
//
// Offset  Member          Size
// 0       b (double)      8
// 8       d (int)         4
// 12      a (char)        1
// 13      c (char)        1
// 14      (pad)           2      tail padding to multiple of 8
//                   ---
// Total:            16
struct ComplexGood {
    double b;     // offset 0
    int d;        // offset 8
    char a;       // offset 12
    char c;       // offset 13  (2 bytes tail padding)
};
// sizeof = 16

// --- Tricky: nested struct and arrays ---
//
// A good interview follow-up: "what about arrays inside structs?"
// An array of char[5] has alignment 1, so it doesn't force padding
// by itself -- but the NEXT member might.
struct WithArray {
    char tag[5];  // offset 0, size 5, alignment 1
    int value;    // offset 8 (3 bytes padding to reach alignment 4)
    char flag;    // offset 12
                  // tail padding: 3 bytes (struct alignment = 4)
};
// sizeof = 16

// --- alignas: explicit alignment ---
struct Cacheline {
    alignas(64) int data;  // forces struct alignment to 64
    char flag;
};
// sizeof = 64 (tail padding to multiple of 64)

// =============================================================================
// Tests
// =============================================================================

void test_struct_padding()
{
    std::cout << "--- Struct Padding ---\n";

    std::cout << "  BadLayout    (char, int, char):        "
              << sizeof(BadLayout) << " bytes\n";
    assert(sizeof(BadLayout) == 12);

    std::cout << "  GoodLayout   (int, char, char):        "
              << sizeof(GoodLayout) << " bytes\n";
    assert(sizeof(GoodLayout) == 8);

    std::cout << "  ComplexBad   (char, double, char, int): "
              << sizeof(ComplexBad) << " bytes\n";
    assert(sizeof(ComplexBad) == 24);

    std::cout << "  ComplexGood  (double, int, char, char): "
              << sizeof(ComplexGood) << " bytes\n";
    assert(sizeof(ComplexGood) == 16);

    std::cout << "  WithArray    (char[5], int, char):      "
              << sizeof(WithArray) << " bytes\n";
    assert(sizeof(WithArray) == 16);

    std::cout << "  Cacheline    (alignas(64) int, char):   "
              << sizeof(Cacheline) << " bytes\n";
    assert(sizeof(Cacheline) == 64);

    // Verify offsets with offsetof
    assert(offsetof(BadLayout, a) == 0);
    assert(offsetof(BadLayout, b) == 4);
    assert(offsetof(BadLayout, c) == 8);

    assert(offsetof(GoodLayout, b) == 0);
    assert(offsetof(GoodLayout, a) == 4);
    assert(offsetof(GoodLayout, c) == 5);

    std::cout << "All Struct Padding assertions passed.\n\n";
}
```