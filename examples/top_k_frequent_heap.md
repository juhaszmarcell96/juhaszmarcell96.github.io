# Top K Frequent Elements

```cpp
// =============================================================================
//
// Approach A (min-heap, O(n log k)):
//   Count frequencies with a hash map. Maintain a min-heap of size k.
//   Push each (frequency, value) pair; if size exceeds k, pop the
//   smallest. After processing all elements, the heap contains the
//   k most frequent.
//
// Approach B (bucket sort, O(n)):
//   Count frequencies, then create buckets indexed by frequency (max
//   possible frequency is n). Walk buckets from highest to lowest,
//   collecting elements until we have k.
//
// Both are implemented below. The heap approach is more commonly
// expected; the bucket sort is a good follow-up to impress.
// =============================================================================

// Approach A: min-heap
[[nodiscard]] std::vector<int> top_k_frequent_heap(
    const std::vector<int>& nums,
    std::size_t k)
{
    // Step 1: count frequencies.
    std::unordered_map<int, int> freq;
    for (int num : nums) {
        ++freq[num];
    }

    // Step 2: min-heap of (frequency, value).
    // std::priority_queue is a max-heap by default, so we use
    // std::greater to make it a min-heap.
    using Entry = std::pair<int, int>;  // (frequency, value)
    std::priority_queue<Entry, std::vector<Entry>, std::greater<Entry>> min_heap;

    for (const auto& [value, count] : freq) {
        min_heap.push({count, value});
        if (min_heap.size() > k) {
            // Pop the least frequent -- the heap always retains the
            // k most frequent seen so far.
            min_heap.pop();
        }
    }

    // Step 3: extract results (order doesn't matter for this problem).
    std::vector<int> result;
    result.reserve(k);
    while (!min_heap.empty()) {
        result.push_back(min_heap.top().second);
        min_heap.pop();
    }

    return result;
}

// Approach B: bucket sort -- O(n) time
[[nodiscard]] std::vector<int> top_k_frequent_bucket(
    const std::vector<int>& nums,
    std::size_t k)
{
    // Step 1: count frequencies.
    std::unordered_map<int, int> freq;
    for (int num : nums) {
        ++freq[num];
    }

    // Step 2: create buckets. Index = frequency, value = list of elements
    // with that frequency. Max possible frequency is nums.size().
    std::vector<std::vector<int>> buckets(nums.size() + 1);
    for (const auto& [value, count] : freq) {
        buckets[static_cast<std::size_t>(count)].push_back(value);
    }

    // Step 3: walk from highest frequency bucket downward.
    std::vector<int> result;
    result.reserve(k);
    for (std::size_t i = buckets.size(); i > 0 && result.size() < k; --i) {
        for (int value : buckets[i - 1]) {
            result.push_back(value);
            if (result.size() == k) {
                break;
            }
        }
    }

    return result;
}

// =============================================================================
// Tests
// =============================================================================

void test_top_k_frequent()
{
    std::cout << "--- Top K Frequent Elements ---\n";

    // Helper: check that result contains exactly the expected elements
    // (order doesn't matter).
    auto contains_all = [](std::vector<int> result, std::vector<int> expected) {
        std::sort(result.begin(), result.end());
        std::sort(expected.begin(), expected.end());
        return result == expected;
    };

    // Heap approach
    assert(contains_all(
        top_k_frequent_heap({1, 1, 1, 2, 2, 3}, 2),
        {1, 2}));

    assert(contains_all(
        top_k_frequent_heap({1}, 1),
        {1}));

    assert(contains_all(
        top_k_frequent_heap({4, 4, 4, 1, 1, 2, 2, 2, 3}, 2),
        {4, 2}));

    // Bucket sort approach -- same test cases
    assert(contains_all(
        top_k_frequent_bucket({1, 1, 1, 2, 2, 3}, 2),
        {1, 2}));

    assert(contains_all(
        top_k_frequent_bucket({1}, 1),
        {1}));

    assert(contains_all(
        top_k_frequent_bucket({4, 4, 4, 1, 1, 2, 2, 2, 3}, 2),
        {4, 2}));

    std::cout << "All Top K Frequent tests passed.\n\n";
}
```