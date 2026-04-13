# Linked List

```cpp
#include <iostream>
#include <memory>
#include <iterator>

template <typename T>
class LinkedList {
private:
    // Node structure for the linked list
    struct node {
        T data;
        std::unique_ptr<node> next;
        
        explicit node(const T& value) : data(value), next(nullptr) {}
        explicit node(T&& value) : data(std::move(value)), next(nullptr) {}
    };
    
    std::unique_ptr<node> m_head;
    node* m_tail;
    std::size_t m_size;

public:
    // Forward iterator implementation
    class iterator {
    public:
        using iterator_category = std::forward_iterator_tag;
        using value_type = T;
        using difference_type = std::ptrdiff_t;
        using pointer = T*;
        using reference = T&;
        
    private:
        node* m_current;
        
    public:
        explicit iterator(node* current) : m_current(current) {}
        
        reference operator*() const {
            return m_current->data;
        }
        
        pointer operator->() const {
            return &(m_current->data);
        }
        
        iterator& operator++() {
            m_current = m_current->next.get();
            return *this;
        }
        
        iterator operator++(int) {
            iterator temp = *this;
            m_current = m_current->next.get();
            return temp;
        }
        
        bool operator==(const iterator& other) const {
            return m_current == other.m_current;
        }
        
        bool operator!=(const iterator& other) const {
            return m_current != other.m_current;
        }
    };
    
    // Const iterator implementation
    class const_iterator {
    public:
        using iterator_category = std::forward_iterator_tag;
        using value_type = T;
        using difference_type = std::ptrdiff_t;
        using pointer = const T*;
        using reference = const T&;
        
    private:
        const node* m_current;
        
    public:
        explicit const_iterator(const node* current) : m_current(current) {}
        
        reference operator*() const {
            return m_current->data;
        }
        
        pointer operator->() const {
            return &(m_current->data);
        }
        
        const_iterator& operator++() {
            m_current = m_current->next.get();
            return *this;
        }
        
        const_iterator operator++(int) {
            const_iterator temp = *this;
            m_current = m_current->next.get();
            return temp;
        }
        
        bool operator==(const const_iterator& other) const {
            return m_current == other.m_current;
        }
        
        bool operator!=(const const_iterator& other) const {
            return m_current != other.m_current;
        }
    };
    
    // Default constructor
    LinkedList() : m_head(nullptr), m_tail(nullptr), m_size(0) {}
    
    // Copy constructor
    LinkedList(const LinkedList& other) : m_head(nullptr), m_tail(nullptr), m_size(0) {
        for (const auto& item : other) {
            push_back(item);
        }
    }
    
    // Move constructor
    LinkedList(LinkedList&& other) noexcept 
        : m_head(std::move(other.m_head)), 
          m_tail(other.m_tail),
          m_size(other.m_size) {
        other.m_tail = nullptr;
        other.m_size = 0;
    }
    
    // Copy assignment operator
    LinkedList& operator=(const LinkedList& other) {
        if (this != &other) {
            LinkedList temp(other);
            *this = std::move(temp);
        }
        return *this;
    }
    
    // Move assignment operator
    LinkedList& operator=(LinkedList&& other) noexcept {
        if (this != &other) {
            m_head = std::move(other.m_head);
            m_tail = other.m_tail;
            m_size = other.m_size;
            
            other.m_tail = nullptr;
            other.m_size = 0;
        }
        return *this;
    }
    
    // Destructor (not needed explicitly due to smart pointers)
    ~LinkedList() = default;
    
    // Add an element to the end of the list
    void push_back(const T& value) {
        auto new_node = std::make_unique<node>(value);
        if (!m_head) {
            m_head = std::move(new_node);
            m_tail = m_head.get();
        }
        else {
            m_tail->next = std::move(new_node);
            m_tail = m_tail->next.get();
        }
        ++m_size;
    }
    
    // Add an element to the end with move semantics
    void push_back(T&& value) {
        auto new_node = std::make_unique<node>(std::move(value));
        if (!m_head) {
            m_head = std::move(new_node);
            m_tail = m_head.get();
        }
        else {
            m_tail->next = std::move(new_node);
            m_tail = m_tail->next.get();
        }
        ++m_size;
    }
    
    // Add an element to the front of the list
    void push_front(const T& value) {
        auto new_node = std::make_unique<node>(value);
        if (!m_head) {
            m_head = std::move(new_node);
            m_tail = m_head.get();
        }
        else {
            new_node->next = std::move(m_head);
            m_head = std::move(new_node);
        }
        ++m_size;
    }
    
    // Add an element to the front with move semantics
    void push_front(T&& value) {
        auto new_node = std::make_unique<node>(std::move(value));
        if (!m_head) {
            m_head = std::move(new_node);
            m_tail = m_head.get();
        }
        else {
            new_node->next = std::move(m_head);
            m_head = std::move(new_node);
        }
        ++m_size;
    }
    
    // Remove the first element
    void pop_front() {
        if (!m_head) {
            return;
        }
        
        if (m_head.get() == m_tail) {
            m_head = nullptr;
            m_tail = nullptr;
        }
        else {
            m_head = std::move(m_head->next);
        }
        --m_size;
    }
    
    // Check if list is empty
    bool empty() const {
        return m_head == nullptr;
    }
    
    // Get size of the list
    std::size_t size() const {
        return m_size;
    }
    
    // Access the first element
    T& front() {
        if (!m_head) {
            throw std::out_of_range("List is empty");
        }
        return m_head->data;
    }
    
    // Access the first element (const version)
    const T& front() const {
        if (!m_head) {
            throw std::out_of_range("List is empty");
        }
        return m_head->data;
    }
    
    // Access the last element
    T& back() {
        if (!m_tail) {
            throw std::out_of_range("List is empty");
        }
        return m_tail->data;
    }
    
    // Access the last element (const version)
    const T& back() const {
        if (!m_tail) {
            throw std::out_of_range("List is empty");
        }
        return m_tail->data;
    }
    
    // Clear the list
    void clear() {
        m_head = nullptr;
        m_tail = nullptr;
        m_size = 0;
    }
    
    // Iterator functions
    iterator begin() {
        return iterator(m_head.get());
    }
    
    iterator end() {
        return iterator(nullptr);
    }
    
    const_iterator begin() const {
        return const_iterator(m_head.get());
    }
    
    const_iterator end() const {
        return const_iterator(nullptr);
    }
    
    const_iterator cbegin() const {
        return const_iterator(m_head.get());
    }
    
    const_iterator cend() const {
        return const_iterator(nullptr);
    }
};
```