# Longest Substring Without Repeating Characters

Sliding window: maintain [left, right) as the current window of unique
characters. Use a map from char -> its most recent index. When we see a
duplicate that is inside our current window, jump `left` past it.

```cpp
// =============================================================================
//
// Time:  O(n)  -- each character is visited at most twice (once by right,
//                 once when left jumps past it)
// Space: O(min(n, alphabet_size))
//
// Why char->index map instead of a set?
//   With a set, when we hit a duplicate we must shrink left one step at a
//   time (removing chars from the set) until the duplicate is gone -- still
//   O(n) total but the code is clumsier. With a map we can jump left
//   directly to (last_seen_index + 1).
// =============================================================================

[[nodiscard]] std::size_t longest_unique_substring(const std::string& s)
{
    // char -> most recent index where it appeared
    std::unordered_map<char, std::size_t> last_seen;

    std::size_t max_length = 0;
    std::size_t left = 0;

    for (std::size_t right = 0; right < s.size(); ++right) {
        auto it = last_seen.find(s[right]);

        // If the character was seen before AND its last position is within
        // our current window [left, right), we must shrink.
        if (it != last_seen.end() && it->second >= left) {
            left = it->second + 1;
        }

        last_seen[s[right]] = right;

        std::size_t window_length = right - left + 1;
        if (window_length > max_length) {
            max_length = window_length;
        }
    }

    return max_length;
}

// =============================================================================
// Tests
// =============================================================================

void test_longest_unique_substring()
{
    std::cout << "--- Longest Substring Without Repeating Characters ---\n";

    assert(longest_unique_substring("abcabcbb") == 3);  // "abc"
    assert(longest_unique_substring("bbbbb") == 1);      // "b"
    assert(longest_unique_substring("pwwkew") == 3);     // "wke"
    assert(longest_unique_substring("") == 0);
    assert(longest_unique_substring("abcdef") == 6);     // entire string
    assert(longest_unique_substring("abba") == 2);       // "ab" or "ba"
    assert(longest_unique_substring("tmmzuxt") == 5);    // "mzuxt"

    std::cout << "All Longest Unique Substring tests passed.\n\n";
}
```