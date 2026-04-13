# Matrix

```cpp
#include <cstdint>
#include <iostream>
#include <vector>
#include <string>
#include <algorithm>
#include <stdexcept>

struct Dimensions {
    std::size_t row { 0 };
    std::size_t col { 0 };
};

/*
 * Row-major flat storage: element (i,j) lives at index [i * num_cols + j].
 *
 *   2x3 example
 *   ┌───────────────────────────────┐
 *   │ a00  a01  a02 │ a10  a11  a12 │   <- contiguous in memory
 *   └───────────────────────────────┘
 */

template<typename T>
class Matrix {
private:
    std::size_t m_rows;
    std::size_t m_cols;
    std::vector<T> m_data;           // flat row-major buffer

    std::size_t idx(std::size_t i, std::size_t j) const { return i * m_cols + j; }

public:
    Matrix(std::size_t num_rows, std::size_t num_cols)
        : m_rows{num_rows}, m_cols{num_cols}, m_data(num_rows * num_cols, T{})
    {
        if (num_rows == 0) { throw std::invalid_argument("number of rows cannot be 0"); }
        if (num_cols == 0) { throw std::invalid_argument("number of columns cannot be 0"); }
    }

    Matrix() = delete;
    ~Matrix() = default;
    Matrix(const Matrix&) = default;
    Matrix(Matrix&&) noexcept = default;
    Matrix& operator=(const Matrix&) = default;
    Matrix& operator=(Matrix&&) noexcept = default;

    // ── Dimensions ──────────────────────────────────────────────

    std::size_t get_num_rows() const { return m_rows; }
    std::size_t get_num_cols() const { return m_cols; }
    Dimensions  get_dimensions() const { return {m_rows, m_cols}; }

    // ── Raw data access ─────────────────────────────────────────

    T*       data()       { return m_data.data(); }
    const T* data() const { return m_data.data(); }

    // ── Element access ──────────────────────────────────────────

    T  operator()(std::size_t i, std::size_t j) const { return m_data[idx(i, j)]; }
    T& operator()(std::size_t i, std::size_t j)       { return m_data[idx(i, j)]; }

    // Row span (pointer + length) - replaces operator[] returning vector&
    const T* row(std::size_t i) const { return m_data.data() + i * m_cols; }
    T*       row(std::size_t i)       { return m_data.data() + i * m_cols; }

    // ── Arithmetic ──────────────────────────────────────────────

    Matrix operator+(const Matrix& rhs) const {
        check_same_dims(rhs);
        Matrix result{m_rows, m_cols};
        for (std::size_t k = 0; k < m_data.size(); ++k) {
            result.m_data[k] = m_data[k] + rhs.m_data[k];
        }
        return result;
    }

    Matrix operator-(const Matrix& rhs) const {
        check_same_dims(rhs);
        Matrix result{m_rows, m_cols};
        for (std::size_t k = 0; k < m_data.size(); ++k) {
            result.m_data[k] = m_data[k] - rhs.m_data[k];
        }
        return result;
    }

    // matrix * scalar
    Matrix operator*(const T& scalar) const {
        Matrix result{m_rows, m_cols};
        for (std::size_t k = 0; k < m_data.size(); ++k) {
            result.m_data[k] = m_data[k] * scalar;
        }
        return result;
    }

    // scalar * matrix
    friend Matrix operator*(const T& scalar, const Matrix& mat) {
        return mat * scalar;
    }

    Matrix operator/(const T& scalar) const {
        Matrix result{m_rows, m_cols};
        for (std::size_t k = 0; k < m_data.size(); ++k) {
            result.m_data[k] = m_data[k] / scalar;
        }
        return result;
    }

    // scalar / matrix (element-wise)
    friend Matrix operator/(const T& scalar, const Matrix& mat) {
        Matrix result{mat.m_rows, mat.m_cols};
        for (std::size_t k = 0; k < mat.m_data.size(); ++k) {
            result.m_data[k] = scalar / mat.m_data[k];
        }
        return result;
    }

    /* A_nxm * B_mxr = C_nxr */
    Matrix operator*(const Matrix& rhs) const {
        if (m_cols != rhs.m_rows) {
            throw std::invalid_argument(
                "inner dimensions must match for multiplication ("
                + std::to_string(m_cols) + " vs " + std::to_string(rhs.m_rows) + ")");
        }
        Matrix result{m_rows, rhs.m_cols};
        for (std::size_t i = 0; i < m_rows; ++i) {
            for (std::size_t k = 0; k < m_cols; ++k) {
                const T a_ik = m_data[idx(i, k)];
                for (std::size_t j = 0; j < rhs.m_cols; ++j) {
                    result.m_data[result.idx(i, j)] += a_ik * rhs.m_data[rhs.idx(k, j)];
                }
            }
        }
        return result;
    }

    // ── Comparison ──────────────────────────────────────────────

    friend bool operator==(const Matrix& a, const Matrix& b) {
        return a.m_rows == b.m_rows
            && a.m_cols == b.m_cols
            && a.m_data == b.m_data;
    }

    friend bool operator!=(const Matrix& a, const Matrix& b) {
        return !(a == b);
    }

    // ── I/O ─────────────────────────────────────────────────────

    friend std::ostream& operator<<(std::ostream& os, const Matrix& mat) {
        os << "dimensions : " << mat.m_rows << "x" << mat.m_cols << '\n';
        for (std::size_t i = 0; i < mat.m_rows; ++i) {
            os << "[ ";
            for (std::size_t j = 0; j < mat.m_cols; ++j) {
                os << mat.m_data[mat.idx(i, j)] << ' ';
            }
            os << "]\n";
        }
        return os;
    }

private:
    void check_same_dims(const Matrix& rhs) const {
        if (m_rows != rhs.m_rows || m_cols != rhs.m_cols) {
            throw std::invalid_argument("matrices must have the same dimensions");
        }
    }
};
```