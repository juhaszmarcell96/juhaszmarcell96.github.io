# Container With Most Water

```cpp
// =============================================================================
//
// Two pointers start at both ends. The area between them is:
//   min(height[left], height[right]) * (right - left)
//
// We always move the pointer with the shorter height inward. Why?
// The area is limited by the shorter side. If we move the taller side
// inward, the width decreases and the height can't increase beyond
// the shorter side -- so area can only decrease or stay the same.
// Moving the shorter side at least gives a chance of finding a taller
// line that increases the area.
//
// Time:  O(n)
// Space: O(1)
// =============================================================================

[[nodiscard]] int max_water_area(const std::vector<int>& heights)
{
    if (heights.size() < 2) {
        return 0;
    }

    std::size_t left = 0;
    std::size_t right = heights.size() - 1;
    int max_area = 0;

    while (left < right) {
        int width = static_cast<int>(right - left);
        int height = std::min(heights[left], heights[right]);
        int area = width * height;
        max_area = std::max(max_area, area);

        // Move the shorter side inward.
        if (heights[left] < heights[right]) {
            ++left;
        }
        else {
            --right;
        }
    }

    return max_area;
}

// =============================================================================
// Tests
// =============================================================================

void test_max_water()
{
    std::cout << "--- Container With Most Water ---\n";

    assert(max_water_area({1, 8, 6, 2, 5, 4, 8, 3, 7}) == 49);
    assert(max_water_area({1, 1}) == 1);
    assert(max_water_area({4, 3, 2, 1, 4}) == 16);
    assert(max_water_area({1, 2, 1}) == 2);
    assert(max_water_area({}) == 0);
    assert(max_water_area({5}) == 0);

    std::cout << "All Container With Most Water tests passed.\n\n";
}
```