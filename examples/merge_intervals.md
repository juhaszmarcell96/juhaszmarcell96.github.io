# Merge Intervals

{% raw %}
```cpp
// =============================================================================
//
// Sort by start time, then sweep through. If the current interval overlaps
// with the last merged one (i.e. current.start <= merged_back.end), extend
// the merged interval's end. Otherwise start a new merged interval.
//
// Time:  O(n log n) for the sort
// Space: O(n) for the output (O(log n) for the sort itself)
//
// Edge cases:
//   - Empty input -> empty output.
//   - Intervals that touch exactly (e.g. [1,3] and [3,5]) -> merged,
//     because start <= previous end.
//   - One interval completely inside another (e.g. [1,10] and [2,3]).
// =============================================================================

struct Interval {
    int start;
    int end;
};

[[nodiscard]] std::vector<Interval> merge_intervals(std::vector<Interval> intervals)
{
    if (intervals.empty()) {
        return {};
    }

    // Sort by start time; break ties by end time (not strictly necessary
    // but makes the behavior deterministic).
    std::sort(intervals.begin(), intervals.end(),
        [](const Interval& a, const Interval& b) {
            if (a.start != b.start) {
                return a.start < b.start;
            }
            return a.end < b.end;
        });

    std::vector<Interval> merged;
    merged.push_back(intervals[0]);

    for (std::size_t i = 1; i < intervals.size(); ++i) {
        if (intervals[i].start <= merged.back().end) {
            // Overlapping or touching -- extend the end.
            merged.back().end = std::max(merged.back().end, intervals[i].end);
        }
        else {
            // No overlap -- start a new merged interval.
            merged.push_back(intervals[i]);
        }
    }

    return merged;
}

// =============================================================================
// Tests
// =============================================================================

void test_merge_intervals()
{
    std::cout << "--- Merge Intervals ---\n";

    // Basic overlapping
    auto result = merge_intervals({{1, 3}, {2, 6}, {8, 10}, {15, 18}});
    assert(result.size() == 3);
    assert(result[0].start == 1 && result[0].end == 6);
    assert(result[1].start == 8 && result[1].end == 10);
    assert(result[2].start == 15 && result[2].end == 18);

    // Touching intervals should merge
    result = merge_intervals({{1, 3}, {3, 5}});
    assert(result.size() == 1);
    assert(result[0].start == 1 && result[0].end == 5);

    // Nested interval (one inside another)
    result = merge_intervals({{1, 10}, {2, 3}, {4, 5}});
    assert(result.size() == 1);
    assert(result[0].start == 1 && result[0].end == 10);

    // Already non-overlapping
    result = merge_intervals({{1, 2}, {4, 5}, {7, 8}});
    assert(result.size() == 3);

    // Empty
    result = merge_intervals({});
    assert(result.empty());

    // Single interval
    result = merge_intervals({{5, 10}});
    assert(result.size() == 1);

    std::cout << "All Merge Intervals tests passed.\n\n";
}
```
{% endraw %}