# First Non-Repeating Character

```cpp
// =============================================================================
// 
// Two passes:
//   1) Count frequency of every character using a fixed-size array (256
//      entries covers all unsigned char values).
//   2) Walk the string again and return the first character with count == 1.
//
// Time:  O(n)
// Space: O(1) -- the frequency array is bounded by the alphabet size (256),
//                independent of input length.
//
// We return std::optional so the caller can distinguish "not found" cleanly
// without sentinel values.
// =============================================================================

[[nodiscard]] std::optional<char> first_non_repeating(const std::string& s)
{
    // Using std::array instead of a C-style array; value-initialized to 0.
    std::array<int, 256> freq{};

    // First pass: count frequencies.
    for (unsigned char ch : s) {
        ++freq[ch];
    }

    // Second pass: find the first character with frequency 1.
    for (unsigned char ch : s) {
        if (freq[ch] == 1) {
            return static_cast<char>(ch);
        }
    }

    return std::nullopt;
}

// =============================================================================
// Tests
// =============================================================================

void test_first_non_repeating()
{
    std::cout << "--- First Non-Repeating Character ---\n";

    auto result = first_non_repeating("leetcode");
    assert(result.has_value() && result.value() == 'l');

    result = first_non_repeating("loveleetcode");
    assert(result.has_value() && result.value() == 'v');

    result = first_non_repeating("aabb");
    assert(!result.has_value());  // all repeated

    result = first_non_repeating("");
    assert(!result.has_value());

    result = first_non_repeating("z");
    assert(result.has_value() && result.value() == 'z');

    std::cout << "All First Non-Repeating Character tests passed.\n\n";
}
```