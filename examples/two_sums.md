# Two Sum

```cpp
// =============================================================================
//
// Walk the array once. For each element, check whether (target - current)
// has already been seen. If yes, return both indices. If no, store the
// current value -> index mapping for future lookups.
//
// Time:  O(n)
// Space: O(n)
//
// Edge cases:
//   - Same element used twice: the map lookup happens BEFORE we insert the
//     current index, so we never match an element with itself.
//   - No solution exists: we return an empty vector (or could throw).
// =============================================================================

[[nodiscard]] std::vector<std::size_t> two_sum(
    const std::vector<int>& nums,
    int target)
{
    // value -> index of first occurrence
    std::unordered_map<int, std::size_t> seen;

    for (std::size_t i = 0; i < nums.size(); ++i) {
        int complement = target - nums[i];
        auto it = seen.find(complement);

        if (it != seen.end()) {
            return {it->second, i};
        }

        // Insert AFTER the lookup so we don't pair an element with itself.
        seen[nums[i]] = i;
    }

    return {};  // no solution found
}

// =============================================================================
// Tests
// =============================================================================

void test_two_sum()
{
    std::cout << "--- Two Sum ---\n";

    // Basic case
    auto result = two_sum({2, 7, 11, 15}, 9);
    assert(result.size() == 2);
    assert(result[0] == 0 && result[1] == 1);

    // Target uses elements later in the array
    result = two_sum({3, 2, 4}, 6);
    assert(result.size() == 2);
    assert(result[0] == 1 && result[1] == 2);

    // Duplicate values (both 3s), must not pair element with itself
    result = two_sum({3, 3}, 6);
    assert(result.size() == 2);
    assert(result[0] == 0 && result[1] == 1);

    // Negative numbers
    result = two_sum({-1, -2, -3, -4, -5}, -8);
    assert(result.size() == 2);
    assert(result[0] == 2 && result[1] == 4);

    // No solution
    result = two_sum({1, 2, 3}, 100);
    assert(result.empty());

    std::cout << "All Two Sum tests passed.\n\n";
}
```