# Maximum Subarray (Kadane's Algorithm)

```cpp
// =============================================================================
//
// At each position, decide: is it better to extend the running subarray
// or start fresh from the current element?
//
//   current_sum = max(nums[i], current_sum + nums[i])
//
// This works because if the running sum is negative, it can only drag
// down whatever comes next, so we discard it.
//
// Time:  O(n)
// Space: O(1)
//
// Edge cases:
//   - All negative numbers: the answer is the single largest (least
//     negative) element. Kadane handles this naturally.
//   - Single element: trivially correct.
// =============================================================================

[[nodiscard]] int max_subarray_sum(const std::vector<int>& nums)
{
    if (nums.empty()) {
        throw std::invalid_argument("array must not be empty");
    }

    int current_sum = nums[0];
    int max_sum = nums[0];

    for (std::size_t i = 1; i < nums.size(); ++i) {
        // Extend the current subarray, or start fresh here.
        current_sum = std::max(nums[i], current_sum + nums[i]);
        max_sum = std::max(max_sum, current_sum);
    }

    return max_sum;
}

// Bonus: return the actual subarray bounds [left, right] (inclusive).
struct SubarrayResult {
    int sum;
    std::size_t left;
    std::size_t right;
};

[[nodiscard]] SubarrayResult max_subarray_with_bounds(const std::vector<int>& nums)
{
    if (nums.empty()) {
        throw std::invalid_argument("array must not be empty");
    }

    int current_sum = nums[0];
    int max_sum = nums[0];
    std::size_t temp_left = 0;
    std::size_t best_left = 0;
    std::size_t best_right = 0;

    for (std::size_t i = 1; i < nums.size(); ++i) {
        if (nums[i] > current_sum + nums[i]) {
            // Starting fresh is better -- new subarray begins here.
            current_sum = nums[i];
            temp_left = i;
        }
        else {
            current_sum = current_sum + nums[i];
        }

        if (current_sum > max_sum) {
            max_sum = current_sum;
            best_left = temp_left;
            best_right = i;
        }
    }

    return {max_sum, best_left, best_right};
}

// =============================================================================
// Tests
// =============================================================================

void test_max_subarray()
{
    std::cout << "--- Maximum Subarray (Kadane) ---\n";

    assert(max_subarray_sum({-2, 1, -3, 4, -1, 2, 1, -5, 4}) == 6);
    assert(max_subarray_sum({1}) == 1);
    assert(max_subarray_sum({-1}) == -1);
    assert(max_subarray_sum({-3, -2, -1}) == -1);  // all negative
    assert(max_subarray_sum({5, 4, -1, 7, 8}) == 23);

    // Test bounds tracking
    auto res = max_subarray_with_bounds({-2, 1, -3, 4, -1, 2, 1, -5, 4});
    assert(res.sum == 6);
    assert(res.left == 3 && res.right == 6);  // subarray [4, -1, 2, 1]

    std::cout << "All Maximum Subarray tests passed.\n\n";
}
```