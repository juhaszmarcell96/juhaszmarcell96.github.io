# Binary Search on Rotated Sorted Array

```cpp
// =============================================================================
//
// The array was sorted then rotated, e.g. [4,5,6,7,0,1,2].
// At each step of binary search, at least one half [left..mid] or
// [mid..right] is sorted. We figure out which half is sorted, then
// check if the target falls within that sorted range.
//
// How to determine which half is sorted:
//   - If nums[left] <= nums[mid], the left half is sorted.
//   - Otherwise the right half is sorted.
//
// Time:  O(log n)
// Space: O(1)
//
// Edge case: this assumes no duplicates. With duplicates, worst case
// degrades to O(n) because you can't distinguish which half is sorted
// when nums[left] == nums[mid] == nums[right].
// =============================================================================

[[nodiscard]] std::optional<std::size_t> search_rotated(
    const std::vector<int>& nums,
    int target)
{
    if (nums.empty()) {
        return std::nullopt;
    }

    std::size_t left = 0;
    std::size_t right = nums.size() - 1;

    while (left <= right) {
        std::size_t mid = left + (right - left) / 2;

        if (nums[mid] == target) {
            return mid;
        }

        if (nums[left] <= nums[mid]) {
            // Left half [left..mid] is sorted.
            if (target >= nums[left] && target < nums[mid]) {
                // Target is in the sorted left half.
                right = mid - 1;
            }
            else {
                // Target must be in the right half.
                left = mid + 1;
            }
        }
        else {
            // Right half [mid..right] is sorted.
            if (target > nums[mid] && target <= nums[right]) {
                // Target is in the sorted right half.
                left = mid + 1;
            }
            else {
                // Target must be in the left half.
                if (mid == 0) {
                    break;  // prevent underflow on size_t
                }
                right = mid - 1;
            }
        }
    }

    return std::nullopt;
}

// =============================================================================
// Tests
// =============================================================================

void test_search_rotated()
{
    std::cout << "--- Binary Search on Rotated Array ---\n";

    auto result = search_rotated({4, 5, 6, 7, 0, 1, 2}, 0);
    assert(result.has_value() && result.value() == 4);

    result = search_rotated({4, 5, 6, 7, 0, 1, 2}, 3);
    assert(!result.has_value());

    result = search_rotated({4, 5, 6, 7, 0, 1, 2}, 4);
    assert(result.has_value() && result.value() == 0);

    result = search_rotated({4, 5, 6, 7, 0, 1, 2}, 2);
    assert(result.has_value() && result.value() == 6);

    // Not rotated at all
    result = search_rotated({1, 2, 3, 4, 5}, 3);
    assert(result.has_value() && result.value() == 2);

    // Single element
    result = search_rotated({1}, 1);
    assert(result.has_value() && result.value() == 0);

    result = search_rotated({1}, 0);
    assert(!result.has_value());

    // Two elements
    result = search_rotated({3, 1}, 1);
    assert(result.has_value() && result.value() == 1);

    // Empty
    result = search_rotated({}, 5);
    assert(!result.has_value());

    std::cout << "All Rotated Array Search tests passed.\n\n";
}
```