# Constructors

```cpp
// --- DANGEROUS: assignment on unconstructed memory ---
void* raw = std::malloc(sizeof(Widget));

// This is undefined behavior!
// No Widget object exists at 'raw' yet.
// The assignment operator reads from *this (garbage) and
// may free resources that were never allocated.
auto* ptr = static_cast<Widget*>(raw);
*ptr = some_widget;                  // <-- UB: calls operator= on non-object UB

std::free(raw);                      // <-- also UB: no dtor called
```