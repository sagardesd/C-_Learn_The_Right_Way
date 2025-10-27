# C++ Topics questions

## Table of Contents
1. [Copy Constructor](#copy-constructor)
2. [RAII (Resource Acquisition Is Initialization)](#raii-resource-acquisition-is-initialization)
3. [Smart Pointers in C++11](#smart-pointers-in-c11)
4. [Move Semantics and Move Constructors](#move-semantics-and-move-constructors)
5. [C++11 Features](#c11-features)
6. [nullptr in C++11](#nullptr-in-c11)
7. [Lambda Expressions](#lambda-expressions)
8. [Final Keyword](#final-keyword)
9. [STL Containers](#stl-containers)

---

<a id="copy-constructor"></a>
## 1. Copy Constructor

### What is a Copy Constructor?

A **copy constructor** is a special constructor that creates a new object as a copy of an existing object. It takes a reference to an object of the same class and initializes a new object with the same values.

### Syntax

```cpp
class ClassName {
public:
    ClassName(const ClassName& other);  // Copy constructor
};
```

### Why is it Needed?

Copy constructors are needed for:

1. **Creating a copy of an object** when passing by value
2. **Returning objects from functions** by value
3. **Initializing an object with another object** of the same type
4. **Deep copying** when the class manages dynamic resources

### Why `const ClassName&` as Parameter?

The copy constructor takes `const ClassName&` for several important reasons:

#### 1. **Reference (`&`) - Prevents Infinite Recursion**

```cpp
// ❌ WRONG: Would cause infinite recursion
Foo(Foo obj);  // Pass by value

// When you call: Foo obj2 = obj1;
// Compiler needs to copy obj1 to pass it to constructor
// But to copy, it needs to call the copy constructor again
// Which needs to copy again... INFINITE LOOP!
```

```cpp
// ✓ CORRECT: Reference avoids copying
Foo(const Foo& obj);  // Pass by reference

// No copy needed - just pass the address
// No recursion!
```

#### 2. **`const` - Preserves Original Object**

```cpp
class Foo {
    int x;
public:
    // ✓ GOOD: const reference - can't modify original
    Foo(const Foo& obj) : x(obj.x) { }
    
    // ❌ BAD: non-const reference - could modify original
    // Foo(Foo& obj) : x(obj.x) {
    //     obj.x = 999;  // Accidentally modifying the original!
    // }
};

int main() {
    Foo obj1;
    Foo obj2 = obj1;  // Should NOT modify obj1
}
```

#### 3. **`const` - Allows Copying from Temporary Objects**

```cpp
class Foo {
public:
    Foo(const Foo& obj);  // ✓ Can accept temporaries
    // Foo(Foo& obj);     // ❌ Cannot accept temporaries
};

Foo createFoo() {
    return Foo();  // Returns temporary object
}

int main() {
    Foo obj = createFoo();  // ✓ Works with const&
                            // ❌ Fails with non-const&
}
```

### Complete Example

```cpp
#include <iostream>
#include <cstring>

class String {
private:
    char* data;
    size_t length;
    
public:
    // Regular constructor
    String(const char* str) {
        length = strlen(str);
        data = new char[length + 1];
        strcpy(data, str);
        std::cout << "Constructor called\n";
    }
    
    // Copy constructor - Deep copy
    String(const String& other) {
        length = other.length;
        data = new char[length + 1];
        strcpy(data, other.data);
        std::cout << "Copy constructor called\n";
    }
    
    // Destructor
    ~String() {
        delete[] data;
        std::cout << "Destructor called\n";
    }
    
    void print() const {
        std::cout << "String: " << data << std::endl;
    }
};

int main() {
    String str1("Hello");
    str1.print();
    
    String str2 = str1;  // Copy constructor called
    str2.print();
    
    return 0;
}
```

#### Output
```
Constructor called
String: Hello
Copy constructor called
String: Hello
Destructor called
Destructor called
```

### Shallow Copy vs Deep Copy

#### Shallow Copy (Default behavior)
```cpp
class Shallow {
    int* data;
public:
    Shallow(int val) : data(new int(val)) {}
    // No copy constructor - compiler generates one
    // It just copies the pointer, not the data!
    ~Shallow() { delete data; }
};

int main() {
    Shallow obj1(10);
    Shallow obj2 = obj1;  // Both point to same memory!
    // ❌ CRASH: Double delete when both destructors run
}
```

#### Deep Copy (Custom copy constructor)
```cpp
class Deep {
    int* data;
public:
    Deep(int val) : data(new int(val)) {}
    
    // Deep copy constructor
    Deep(const Deep& other) {
        data = new int(*other.data);  // Allocate new memory
    }
    
    ~Deep() { delete data; }
};

int main() {
    Deep obj1(10);
    Deep obj2 = obj1;  // ✓ Each has its own memory
}
```

[↑ Back to Table of Contents](#table-of-contents)

---

<a id="raii-resource-acquisition-is-initialization"></a>
## 2. RAII (Resource Acquisition Is Initialization)

### What is RAII?

**RAII** is a C++ programming idiom where resource management is tied to object lifetime:
- **Acquire resources in constructor** (initialization)
- **Release resources in destructor** (cleanup)

### Key Principle

> "Resource lifetime = Object lifetime"

When an object goes out of scope, its destructor automatically releases resources—no manual cleanup needed!

### Why RAII?

- **Automatic resource management** - No manual cleanup
- **Exception safety** - Resources released even if exceptions occur
- **No memory leaks** - Guaranteed cleanup
- **Simplifies code** - Less error-prone

### RAII Example: File Handling

```cpp
#include <fstream>
#include <iostream>

class FileHandler {
    std::ofstream file;
    
public:
    // Constructor: Acquire resource
    FileHandler(const std::string& filename) {
        file.open(filename);
        if (!file.is_open()) {
            throw std::runtime_error("Failed to open file");
        }
        std::cout << "File opened\n";
    }
    
    // Destructor: Release resource
    ~FileHandler() {
        if (file.is_open()) {
            file.close();
            std::cout << "File closed\n";
        }
    }
    
    void write(const std::string& data) {
        file << data;
    }
};

int main() {
    try {
        FileHandler fh("data.txt");
        fh.write("Hello, RAII!");
        // No need to manually close file!
        // Destructor automatically closes it
    } catch (const std::exception& e) {
        std::cout << "Error: " << e.what() << std::endl;
    }
    // File is closed here automatically
}
```

### RAII Example: Dynamic Memory

```cpp
class Buffer {
    int* data;
    size_t size;
    
public:
    // Constructor: Acquire resource
    Buffer(size_t s) : size(s) {
        data = new int[size];
        std::cout << "Memory allocated\n";
    }
    
    // Destructor: Release resource
    ~Buffer() {
        delete[] data;
        std::cout << "Memory freed\n";
    }
    
    int& operator[](size_t index) {
        return data[index];
    }
};

int main() {
    {
        Buffer buf(100);
        buf[0] = 42;
        // No need to manually delete[]
    }  // Destructor automatically frees memory here
}
```

### Common RAII Resources

1. **Dynamic Memory** - `new`/`delete`
2. **File Handles** - `fopen`/`fclose`
3. **Mutex Locks** - `lock`/`unlock`
4. **Network Sockets** - `open`/`close`
5. **Database Connections** - `connect`/`disconnect`

[↑ Back to Table of Contents](#table-of-contents)

---

<a id="smart-pointers-in-c11"></a>
## 3. Smart Pointers in C++11

### What are RAII Features in C++11?

C++11 introduced **Smart Pointers** as RAII wrappers for dynamic memory management:

- `std::unique_ptr<T>` - Exclusive ownership
- `std::shared_ptr<T>` - Shared ownership
- `std::weak_ptr<T>` - Non-owning reference

### `std::unique_ptr<T>`

**Exclusive ownership** - Only one `unique_ptr` can own a resource at a time.

#### Characteristics:
- ✓ Cannot be copied (only moved)
- ✓ Automatically deletes the resource when it goes out of scope
- ✓ Zero overhead (same as raw pointer)
- ✓ Use for exclusive resource ownership

```cpp
#include <memory>
#include <iostream>

class Resource {
public:
    Resource() { std::cout << "Resource acquired\n"; }
    ~Resource() { std::cout << "Resource destroyed\n"; }
    void use() { std::cout << "Using resource\n"; }
};

int main() {
    // Create unique_ptr
    std::unique_ptr<Resource> ptr1(new Resource());
    // Or better (C++14):
    auto ptr2 = std::make_unique<Resource>();
    
    ptr1->use();
    
    // ❌ Cannot copy
    // std::unique_ptr<Resource> ptr3 = ptr1;  // ERROR
    
    // ✓ Can move
    std::unique_ptr<Resource> ptr3 = std::move(ptr1);
    // Now ptr1 is nullptr, ptr3 owns the resource
    
    // No need to delete - automatic cleanup
}  // Resource destroyed here
```

### `std::shared_ptr<T>`

**Shared ownership** - Multiple `shared_ptr` can own the same resource.

#### Characteristics:
- ✓ Can be copied and shared
- ✓ Uses reference counting
- ✓ Resource deleted when last `shared_ptr` is destroyed
- ✓ Thread-safe reference counting
- ✗ Slight overhead (reference count storage)

```cpp
#include <memory>
#include <iostream>

class Resource {
public:
    Resource() { std::cout << "Resource acquired\n"; }
    ~Resource() { std::cout << "Resource destroyed\n"; }
    void use() { std::cout << "Using resource\n"; }
};

int main() {
    // Create shared_ptr
    std::shared_ptr<Resource> ptr1 = std::make_shared<Resource>();
    std::cout << "Count: " << ptr1.use_count() << "\n";  // 1
    
    {
        // Copy shared_ptr - both own the resource
        std::shared_ptr<Resource> ptr2 = ptr1;
        std::cout << "Count: " << ptr1.use_count() << "\n";  // 2
        
        ptr2->use();
    }  // ptr2 destroyed, count decreases
    
    std::cout << "Count: " << ptr1.use_count() << "\n";  // 1
    
}  // Resource destroyed here (last shared_ptr destroyed)
```

#### Output
```
Resource acquired
Count: 1
Count: 2
Using resource
Count: 1
Resource destroyed
```

### Difference Between `unique_ptr` and `shared_ptr`

| Feature | `unique_ptr` | `shared_ptr` |
|---------|-------------|--------------|
| **Ownership** | Exclusive (single owner) | Shared (multiple owners) |
| **Copyable** | ❌ No (move-only) | ✓ Yes |
| **Reference Counting** | No | Yes |
| **Overhead** | Zero (same as raw pointer) | Small (ref count storage) |
| **Use Case** | Single owner, clear lifetime | Multiple owners, unclear lifetime |
| **Thread Safety** | N/A | Reference count is thread-safe |
| **Performance** | Faster | Slightly slower |

```cpp
// unique_ptr - Exclusive ownership
std::unique_ptr<int> up1(new int(10));
// std::unique_ptr<int> up2 = up1;  // ❌ ERROR: Cannot copy
std::unique_ptr<int> up2 = std::move(up1);  // ✓ OK: Transfer ownership

// shared_ptr - Shared ownership
std::shared_ptr<int> sp1 = std::make_shared<int>(10);
std::shared_ptr<int> sp2 = sp1;  // ✓ OK: Both share ownership
```

### `std::weak_ptr<T>`

**Non-owning reference** - Observes a `shared_ptr` without affecting its reference count.

#### Use Cases for `weak_ptr`:

1. **Breaking Circular References**
2. **Caching** (check if resource still exists)
3. **Observer Pattern** (observe without owning)

#### Characteristics:
- ✓ Doesn't increase reference count
- ✓ Can check if resource still exists
- ✓ Must convert to `shared_ptr` to access resource
- ✓ Prevents circular reference memory leaks

### Problem: Circular Reference

```cpp
class Node {
public:
    std::shared_ptr<Node> next;
    ~Node() { std::cout << "Node destroyed\n"; }
};

int main() {
    auto node1 = std::make_shared<Node>();
    auto node2 = std::make_shared<Node>();
    
    node1->next = node2;
    node2->next = node1;  // ❌ Circular reference!
    
}  // Memory leak! Neither node is destroyed
```

### Solution: Use `weak_ptr`

```cpp
class Node {
public:
    std::shared_ptr<Node> next;
    std::weak_ptr<Node> prev;  // ✓ Use weak_ptr
    ~Node() { std::cout << "Node destroyed\n"; }
};

int main() {
    auto node1 = std::make_shared<Node>();
    auto node2 = std::make_shared<Node>();
    
    node1->next = node2;
    node2->prev = node1;  // ✓ No circular reference!
    
}  // Both nodes destroyed properly
```

### Using `weak_ptr`

```cpp
#include <memory>
#include <iostream>

int main() {
    std::weak_ptr<int> weak;
    
    {
        auto shared = std::make_shared<int>(42);
        weak = shared;  // weak observes shared
        
        std::cout << "Shared count: " << shared.use_count() << "\n";  // 1
        std::cout << "Weak count: " << weak.use_count() << "\n";      // 1
        
        // Access through weak_ptr
        if (auto locked = weak.lock()) {  // Convert to shared_ptr
            std::cout << "Value: " << *locked << "\n";
        }
    }  // shared destroyed
    
    // Check if resource still exists
    if (weak.expired()) {
        std::cout << "Resource no longer exists\n";
    }
}
```

#### Output
```
Shared count: 1
Weak count: 1
Value: 42
Resource no longer exists
```

### Smart Pointer Best Practices

✓ **DO:**
- Use `std::make_unique` and `std::make_shared`
- Prefer `unique_ptr` by default
- Use `shared_ptr` only when needed
- Use `weak_ptr` to break circular references

✗ **DON'T:**
- Mix smart pointers with raw `new`/`delete`
- Create `shared_ptr` from raw pointers multiple times
- Use `shared_ptr` when `unique_ptr` suffices

```cpp
// ✓ GOOD
auto ptr = std::make_unique<MyClass>();

// ❌ BAD
MyClass* raw = new MyClass();
std::unique_ptr<MyClass> ptr(raw);
// ...
delete raw;  // Double delete!

// ❌ BAD
MyClass* raw = new MyClass();
std::shared_ptr<MyClass> sp1(raw);
std::shared_ptr<MyClass> sp2(raw);  // Two independent ref counts!
```

[↑ Back to Table of Contents](#table-of-contents)

---

<a id="move-semantics-and-move-constructors"></a>
## 4. Move Semantics and Move Constructors

### What is Move Semantics?

**Move semantics** allows transferring resources from one object to another without copying. Instead of duplicating data, we "move" ownership.

### Why Move Semantics?

- **Performance** - Avoid expensive copies
- **Efficiency** - Transfer resources instead of duplicating
- **Enable move-only types** - `unique_ptr`, file handles, etc.

### Lvalues vs Rvalues

```cpp
int x = 10;      // x is an lvalue (has a name, addressable)
int y = x + 5;   // (x + 5) is an rvalue (temporary, no name)

std::string s1 = "Hello";           // s1 is lvalue
std::string s2 = s1 + " World";     // (s1 + " World") is rvalue
```

- **Lvalue** - Has a name, can take address (`&x`)
- **Rvalue** - Temporary, cannot take address

### Move Constructor

A **move constructor** transfers resources from a temporary (rvalue) to a new object.

#### Syntax

```cpp
class ClassName {
public:
    ClassName(ClassName&& other) noexcept;  // Move constructor
};
```

- `&&` - Rvalue reference (binds to temporaries)
- `noexcept` - Promises no exceptions (enables optimizations)

### Complete Example: Move Constructor

```cpp
#include <iostream>
#include <cstring>

class String {
private:
    char* data;
    size_t length;
    
public:
    // Constructor
    String(const char* str) {
        std::cout << "Constructor\n";
        length = strlen(str);
        data = new char[length + 1];
        strcpy(data, str);
    }
    
    // Copy constructor - Deep copy
    String(const String& other) {
        std::cout << "Copy constructor\n";
        length = other.length;
        data = new char[length + 1];
        strcpy(data, other.data);
    }
    
    // Move constructor - Transfer ownership
    String(String&& other) noexcept {
        std::cout << "Move constructor\n";
        // Steal resources
        data = other.data;
        length = other.length;
        
        // Leave other in valid but empty state
        other.data = nullptr;
        other.length = 0;
    }
    
    // Destructor
    ~String() {
        std::cout << "Destructor\n";
        delete[] data;
    }
    
    void print() const {
        if (data) {
            std::cout << "String: " << data << std::endl;
        } else {
            std::cout << "String: (empty)\n";
        }
    }
};

String createString() {
    return String("Temporary");  // Returns rvalue
}

int main() {
    std::cout << "=== Copy ===\n";
    String s1("Hello");
    String s2 = s1;  // Copy constructor
    s1.print();
    s2.print();
    
    std::cout << "\n=== Move ===\n";
    String s3 = createString();  // Move constructor
    s3.print();
    
    std::cout << "\n=== Explicit Move ===\n";
    String s4("World");
    String s5 = std::move(s4);  // Move constructor
    s4.print();  // s4 is now empty
    s5.print();
    
    return 0;
}
```

#### Output
```
=== Copy ===
Constructor
Copy constructor
String: Hello
String: Hello

=== Move ===
Constructor
Move constructor
Destructor
String: Temporary

=== Explicit Move ===
Constructor
Move constructor
String: (empty)
String: World
Destructor
Destructor
Destructor
Destructor
```

### Move Assignment Operator

```cpp
class String {
public:
    // Move assignment operator
    String& operator=(String&& other) noexcept {
        std::cout << "Move assignment\n";
        
        if (this != &other) {
            // Free existing resources
            delete[] data;
            
            // Steal resources
            data = other.data;
            length = other.length;
            
            // Leave other in valid state
            other.data = nullptr;
            other.length = 0;
        }
        
        return *this;
    }
};
```

### `std::move` Function

`std::move` casts an lvalue to an rvalue reference, enabling move semantics.

```cpp
std::vector<int> v1 = {1, 2, 3, 4, 5};
std::vector<int> v2 = std::move(v1);  // Move, not copy

// v1 is now in valid but unspecified state (usually empty)
// v2 now owns the data
```

### Rule of Five

If you define any of these, you should define all:

1. Destructor
2. Copy constructor
3. Copy assignment operator
4. Move constructor
5. Move assignment operator

```cpp
class MyClass {
public:
    ~MyClass();                                    // Destructor
    MyClass(const MyClass& other);                 // Copy constructor
    MyClass& operator=(const MyClass& other);      // Copy assignment
    MyClass(MyClass&& other) noexcept;            // Move constructor
    MyClass& operator=(MyClass&& other) noexcept; // Move assignment
};
```

### When Does Move Happen?

1. **Returning from function** (automatic)
```cpp
std::vector<int> createVector() {
    std::vector<int> v = {1, 2, 3};
    return v;  // Move, not copy
}
```

2. **Passing temporaries**
```cpp
void process(std::vector<int> v) { }

process({1, 2, 3});  // Move temporary
```

3. **Explicit `std::move`**
```cpp
std::vector<int> v1 = {1, 2, 3};
std::vector<int> v2 = std::move(v1);  // Explicit move
```

### Move-Only Types

Some types can only be moved, not copied:

```cpp
std::unique_ptr<int> up1 = std::make_unique<int>(42);
// std::unique_ptr<int> up2 = up1;  // ❌ ERROR: Cannot copy
std::unique_ptr<int> up2 = std::move(up1);  // ✓ OK: Move

std::thread t1([]{ });
// std::thread t2 = t1;  // ❌ ERROR: Cannot copy
std::thread t2 = std::move(t1);  // ✓ OK: Move
```

[↑ Back to Table of Contents](#table-of-contents)

---

<a id="c11-features"></a>
## 5. C++11 Features

### Highlighted Features Introduced in C++11

#### 1. **Auto Type Deduction**
```cpp
auto x = 42;              // int
auto y = 3.14;            // double
auto z = "Hello";         // const char*
auto v = std::vector<int>{1, 2, 3};
```

#### 2. **Range-Based For Loop**
```cpp
std::vector<int> v = {1, 2, 3, 4, 5};
for (auto x : v) {
    std::cout << x << " ";
}
```

#### 3. **Lambda Expressions**
```cpp
auto add = [](int a, int b) { return a + b; };
std::cout << add(3, 4);  // 7
```

#### 4. **Smart Pointers**
```cpp
std::unique_ptr<int> up = std::make_unique<int>(42);
std::shared_ptr<int> sp = std::make_shared<int>(42);
```

#### 5. **Move Semantics**
```cpp
std::vector<int> v1 = {1, 2, 3};
std::vector<int> v2 = std::move(v1);  // Move, not copy
```

#### 6. **Nullptr**
```cpp
int* ptr = nullptr;  // Better than NULL
```

#### 7. **Uniform Initialization**
```cpp
int x{42};
std::vector<int> v{1, 2, 3, 4, 5};
MyClass obj{arg1, arg2};
```

#### 8. **Deleted and Defaulted Functions**
```cpp
class NonCopyable {
public:
    NonCopyable() = default;
    NonCopyable(const NonCopyable&) = delete;  // Prevent copying
};
```

#### 9. **Override and Final**
```cpp
class Base {
    virtual void foo() { }
};

class Derived : public Base {
    void foo() override { }  // Explicitly override
};

class Final final {  // Cannot be inherited
    virtual void bar() final { }  // Cannot be overridden
};
```

#### 10. **Strongly-Typed Enums**
```cpp
enum class Color { Red, Green, Blue };
Color c = Color::Red;
```

#### 11. **Variadic Templates**
```cpp
template<typename... Args>
void print(Args... args) {
    (std::cout << ... << args) << '\n';
}

print(1, 2.5, "Hello");  // 12.5Hello
```

#### 12. **Static Assert**
```cpp
static_assert(sizeof(int) == 4, "int must be 4 bytes");
```

#### 13. **Constexpr**
```cpp
constexpr int square(int x) {
    return x * x;
}

constexpr int result = square(5);  // Computed at compile time
```

#### 14. **Threading Support**
```cpp
std::thread t([]{ std::cout << "Hello from thread\n"; });
t.join();
```

#### 15. **Regular Expressions**
```cpp
std::regex pattern("\\d+");
std::string text = "123 abc 456";
std::smatch match;
if (std::regex_search(text, match, pattern)) {
    std::cout << match[0];  // 123
}
```

[↑ Back to Table of Contents](#table-of-contents)

---

<a id="nullptr-in-c11"></a>
## 6. nullptr in C++11

### What is `nullptr`?

`nullptr` is a keyword introduced in C++11 that represents a null pointer constant. It has its own type: `std::nullptr_t`.

### How is it Different from `NULL`?

#### Traditional `NULL` (C++98)

```cpp
#define NULL 0  // Usually defined as integer 0
```

**Problems with `NULL`:**

1. **Ambiguity in function overloading**
```cpp
void foo(int x) { std::cout << "int version\n"; }
void foo(char* ptr) { std::cout << "pointer version\n"; }

foo(NULL);  // ❌ Calls foo(int), not foo(char*)!
            // NULL is 0, which is an integer
```

2. **Type confusion**
```cpp
int* ptr = NULL;  // Works, but NULL is actually 0 (int)
bool b = (NULL == 0);  // true - NULL is just 0
```

#### Modern `nullptr` (C++11)

```cpp
void foo(int x) { std::cout << "int version\n"; }
void foo(char* ptr) { std::cout << "pointer version\n"; }

foo(nullptr);  // ✓ Calls foo(char*) correctly!
               // nullptr is a pointer type
```

### Key Differences

| Feature | `NULL` | `nullptr` |
|---------|--------|-----------|
| **Type** | Integer (`int` or `long`) | `std::nullptr_t` |
| **Overload Resolution** | May call wrong function | Always correct |
| **Implicit Conversion** | Converts to any integer | Only converts to pointer types |
| **Type Safety** | Weak | Strong |
| **C++ Version** | C++98 and earlier | C++11 and later |

### Examples

```cpp
// 1. nullptr has its own type
auto ptr = nullptr;  // Type is std::nullptr_t

// 2. Can be assigned to any pointer type
int* p1 = nullptr;
char* p2 = nullptr;
void* p3 = nullptr;

// 3. Cannot be assigned to non-pointer types
// int x = nullptr;  // ❌ ERROR

// 4. Works correctly in templates
template<typename T>
void process(T* ptr) {
    if (ptr == nullptr) {
        std::cout << "Null pointer\n";
    }
}

process<int>(nullptr);  // ✓ Works perfectly

// 5. Better in comparisons
if (ptr == nullptr) {  // ✓ Clear intent
    // Handle null pointer
}

if (ptr == NULL) {  // Works but less clear
    // Handle null pointer
}

if (ptr == 0) {  // ❌ Confusing - is it really a null check?
    // Handle null pointer
}
```

### Complete Example

```cpp
#include <iostream>

void process(int value) {
    std::cout << "Processing int: " << value << "\n";
}

void process(int* ptr) {
    if (ptr == nullptr) {
        std::cout << "Processing null pointer\n";
    } else {
        std::cout << "Processing pointer to: " << *ptr << "\n";
    }
}

int main() {
    process(42);       // Calls process(int)
    
    // With NULL (C++98)
    // process(NULL);  // ❌ Ambiguous or calls process(int)!
    
    // With nullptr (C++11)
    process(nullptr);  // ✓ Correctly calls process(int*)
    
    int x = 100;
    process(&x);       // Calls process(int*)
}
```

#### Output
```
Processing int: 42
Processing null pointer
Processing pointer to: 100
```

### Best Practices

✓ **DO:**
- Always use `nullptr` instead of `NULL` in C++11 and later
- Use `nullptr` for pointer initialization
- Use `nullptr` in pointer comparisons

✗ **DON'T:**
- Use `NULL` in new C++ code
- Use `0` to represent null pointers
- Mix `NULL` and `nullptr` in the same codebase

```cpp
// ✓ GOOD
int* ptr = nullptr;
if (ptr == nullptr) { }

// ❌ BAD (in modern C++)
int* ptr = NULL;
if (ptr == 0) { }
```

[↑ Back to Table of Contents](#table-of-contents)

---

<a id="lambda-expressions"></a>
## 7. Lambda Expressions

### What are Lambda Expressions?

**Lambda expressions** (or lambda functions) are anonymous functions that can be defined inline. They're useful for short callbacks, predicates, and functional programming.

### Basic Syntax

```cpp
[capture](parameters) -> return_type { body }
```

- **`[capture]`** - Capture clause (variables from outer scope)
- **`(parameters)`** - Function parameters
- **`-> return_type`** - Return type (optional, usually deduced)
- **`{ body }`** - Function body

### Simple Examples

#### 1. Basic Lambda
```cpp
auto hello = []() {
    std::cout << "Hello, Lambda!\n";
};

hello();  // Call the lambda
```

#### 2. Lambda with Parameters
```cpp
auto add = [](int a, int b) {
    return a + b;
};

std::cout << add(3, 4);  // 7
```

#### 3. Lambda with Explicit Return Type
```cpp
auto divide = [](int a, int b) -> double {
    return static_cast<double>(a) / b;
};

std::cout << divide(5, 2);  // 2.5
```

### Capture Clauses

The capture clause `[...]` specifies which variables from the outer scope are available inside the lambda.

#### Types of Capture

| Capture | Description | Example |
|---------|-------------|---------|
| `[]` | Capture nothing | `[]() { }` |
| `[=]` | Capture all by value | `[=]() { }` |
| `[&]` | Capture all by reference | `[&]() { }` |
| `[x]` | Capture `x` by value | `[x]() { }` |
| `[&x]` | Capture `x` by reference | `[&x]() { }` |
| `[=, &x]` | Capture all by value, `x` by reference | `[=, &x]() { }` |
| `[&, x]` | Capture all by reference, `x` by value | `[&, x]() { }` |
| `[this]` | Capture `this` pointer | `[this]() { }` |

### 1. Capture by Value

Captures a **copy** of the variable. The lambda has its own copy, and modifications don't affect the original.

```cpp
#include <iostream>

int main() {
    int x = 10;
    
    // Capture x by value
    auto lambda = [x]() {
        std::cout << "Inside lambda: " << x << "\n";
        // x = 20;  // ❌ ERROR: Cannot modify captured-by-value variable
                    // (unless lambda is mutable)
    };
    
    lambda();  // Prints: Inside lambda: 10
    
    x = 20;  // Change original
    lambda();  // Still prints: Inside lambda: 10 (has its own copy)
    
    return 0;
}
```

#### Output
```
Inside lambda: 10
Inside lambda: 10
```

#### Mutable Lambda (Modify Captured Values)
```cpp
int x = 10;

auto lambda = [x]() mutable {
    x = 20;  // ✓ OK: Can modify the copy
    std::cout << "Inside lambda: " << x << "\n";
};

lambda();  // Prints: Inside lambda: 20
std::cout << "Original x: " << x << "\n";  // Prints: Original x: 10
```

### 2. Capture by Reference

Captures a **reference** to the variable. The lambda accesses the original variable directly.

```cpp
#include <iostream>

int main() {
    int x = 10;
    
    // Capture x by reference
    auto lambda = [&x]() {
        std::cout << "Inside lambda: " << x << "\n";
        x = 20;  // ✓ Modifies the original
    };
    
    lambda();  // Prints: Inside lambda: 10
    std::cout << "After lambda: " << x << "\n";  // Prints: After lambda: 20
    
    x = 30;  // Change original
    lambda();  // Prints: Inside lambda: 30 (sees the change)
    std::cout << "After lambda: " << x << "\n";  // Prints: After lambda: 20
    
    return 0;
}
```

#### Output
```
Inside lambda: 10
After lambda: 20
Inside lambda: 30
After lambda: 20
```

### Comparison: Capture by Value vs Reference

```cpp
#include <iostream>

int main() {
    int value = 100;
    int reference = 200;
    
    // Capture 'value' by value, 'reference' by reference
    auto lambda = [value, &reference]() {
        std::cout << "Value: " << value << "\n";
        std::cout << "Reference: " << reference << "\n";
        
        // value = 999;      // ❌ ERROR: Cannot modify
        reference = 999;     // ✓ OK: Modifies original
    };
    
    std::cout << "=== Before lambda ===\n";
    std::cout << "value = " << value << "\n";
    std::cout << "reference = " << reference << "\n";
    
    lambda();
    
    std::cout << "\n=== After lambda ===\n";
    std::cout << "value = " << value << "\n";         // Still 100
    std::cout << "reference = " << reference << "\n"; // Changed to 999
    
    return 0;
}
```

#### Output
```
=== Before lambda ===
value = 100
reference = 200
Value: 100
Reference: 200

=== After lambda ===
value = 100
reference = 999
```

### Various Types of Lambda Expressions

#### 1. No Capture
```cpp
auto lambda = []() {
    std::cout << "No capture\n";
};
```

#### 2. Capture All by Value
```cpp
int x = 10, y = 20;
auto lambda = [=]() {
    std::cout << x << ", " << y << "\n";  // Can access x and y
};
```

#### 3. Capture All by Reference
```cpp
int x = 10, y = 20;
auto lambda = [&]() {
    x = 30;  // Modifies original
    y = 40;  // Modifies original
};
```

#### 4. Mixed Capture
```cpp
int x = 10, y = 20, z = 30;
auto lambda = [=, &y]() {  // x and z by value, y by reference
    std::cout << x << ", " << y << ", " << z << "\n";
    y = 99;  // Modifies original y
};
```

#### 5. Capture `this` (In Class Methods)
```cpp
class MyClass {
    int value = 42;
    
public:
    void doSomething() {
        auto lambda = [this]() {
            std::cout << value << "\n";  // Access member variable
        };
        lambda();
    }
};
```

#### 6. Generic Lambda (C++14)
```cpp
auto lambda = [](auto x, auto y) {
    return x + y;
};

std::cout << lambda(3, 4) << "\n";      // 7 (int)
std::cout << lambda(3.5, 4.5) << "\n";  // 8.0 (double)
```

#### 7. Lambda with STL Algorithms
```cpp
#include <vector>
#include <algorithm>

std::vector<int> v = {1, 2, 3, 4, 5};

// for_each with lambda
std::for_each(v.begin(), v.end(), [](int x) {
    std::cout << x << " ";
});

// find_if with lambda
auto it = std::find_if(v.begin(), v.end(), [](int x) {
    return x > 3;
});

// sort with lambda
std::sort(v.begin(), v.end(), [](int a, int b) {
    return a > b;  // Descending order
});
```

#### 8. Lambda Returning Lambda
```cpp
auto makeAdder = [](int x) {
    return [x](int y) {
        return x + y;
    };
};

auto add5 = makeAdder(5);
std::cout << add5(10);  // 15
```

### Complete Example: Various Lambda Uses

```cpp
#include <iostream>
#include <vector>
#include <algorithm>

int main() {
    std::vector<int> numbers = {5, 2, 8, 1, 9, 3};
    
    // 1. Print all elements
    std::cout << "Original: ";
    std::for_each(numbers.begin(), numbers.end(), [](int n) {
        std::cout << n << " ";
    });
    std::cout << "\n";
    
    // 2. Count elements greater than 5
    int threshold = 5;
    int count = std::count_if(numbers.begin(), numbers.end(), 
        [threshold](int n) {
            return n > threshold;
        }
    );
    std::cout << "Count > " << threshold << ": " << count << "\n";
    
    // 3. Sort in descending order
    std::sort(numbers.begin(), numbers.end(), [](int a, int b) {
        return a > b;
    });
    
    std::cout << "Sorted desc: ";
    for (int n : numbers) {
        std::cout << n << " ";
    }
    std::cout << "\n";
    
    // 4. Transform each element (multiply by 2)
    std::transform(numbers.begin(), numbers.end(), numbers.begin(),
        [](int n) {
            return n * 2;
        }
    );
    
    std::cout << "Doubled: ";
    for (int n : numbers) {
        std::cout << n << " ";
    }
    std::cout << "\n";
    
    // 5. Accumulate with lambda
    int sum = 0;
    std::for_each(numbers.begin(), numbers.end(), [&sum](int n) {
        sum += n;
    });
    std::cout << "Sum: " << sum << "\n";
    
    return 0;
}
```

#### Output
```
Original: 5 2 8 1 9 3 
Count > 5: 2
Sorted desc: 9 8 5 3 2 1 
Doubled: 18 16 10 6 4 2 
Sum: 56
```

### Lambda Best Practices

✓ **DO:**
- Use lambdas for short, simple functions
- Capture by reference when you need to modify variables
- Capture by value for thread safety
- Use `[=]` or `[&]` when capturing many variables

✗ **DON'T:**
- Capture by reference if lambda outlives the scope
- Write overly complex lambdas (use regular functions instead)
- Capture unnecessary variables

[↑ Back to Table of Contents](#table-of-contents)

---

<a id="final-keyword"></a>
## 8. Final Keyword

### What is the `final` Keyword?

The `final` keyword (introduced in C++11) is used to **prevent inheritance** or **prevent overriding** of virtual functions.

### Two Uses of `final`:

1. **Prevent class inheritance** - Class cannot be used as a base class
2. **Prevent function overriding** - Virtual function cannot be overridden in derived classes

### 1. Final Class (Prevent Inheritance)

```cpp
class Base final {  // Cannot be inherited
    // ...
};

// class Derived : public Base {  // ❌ ERROR: Cannot inherit from final class
// };
```

#### Example

```cpp
#include <iostream>

class FinalClass final {
public:
    void display() {
        std::cout << "This class cannot be inherited\n";
    }
};

// Attempting to inherit will cause compilation error
// class Derived : public FinalClass {  // ❌ ERROR
// };

int main() {
    FinalClass obj;
    obj.display();
    return 0;
}
```

### 2. Final Function (Prevent Overriding)

```cpp
class Base {
public:
    virtual void foo() final {  // Cannot be overridden
        std::cout << "Base::foo\n";
    }
};

class Derived : public Base {
    // void foo() override {  // ❌ ERROR: Cannot override final function
    // }
};
```

#### Example

```cpp
#include <iostream>

class Animal {
public:
    virtual void makeSound() {
        std::cout << "Some sound\n";
    }
    
    virtual void eat() final {  // Cannot be overridden
        std::cout << "Animal is eating\n";
    }
};

class Dog : public Animal {
public:
    void makeSound() override {  // ✓ OK: Can override
        std::cout << "Woof!\n";
    }
    
    // void eat() override {  // ❌ ERROR: Cannot override final function
    // }
};

int main() {
    Dog dog;
    dog.makeSound();  // Woof!
    dog.eat();        // Animal is eating
    return 0;
}
```

### Complete Example: Using `final`

```cpp
#include <iostream>

// Base class
class Shape {
public:
    virtual void draw() {
        std::cout << "Drawing a shape\n";
    }
    
    virtual void resize() final {  // Cannot be overridden
        std::cout << "Resizing shape\n";
    }
};

// Intermediate class - can be inherited
class Circle : public Shape {
public:
    void draw() override {
        std::cout << "Drawing a circle\n";
    }
    
    // Cannot override resize() - it's final
};

// Final class - cannot be inherited
class SpecialCircle final : public Circle {
public:
    void draw() override {
        std::cout << "Drawing a special circle\n";
    }
};

// This would cause compilation error:
// class VerySpecialCircle : public SpecialCircle {  // ❌ ERROR
// };

int main() {
    Shape shape;
    Circle circle;
    SpecialCircle special;
    
    shape.draw();    // Drawing a shape
    circle.draw();   // Drawing a circle
    special.draw();  // Drawing a special circle
    
    shape.resize();   // Resizing shape
    circle.resize();  // Resizing shape (inherited, cannot override)
    special.resize(); // Resizing shape (inherited, cannot override)
    
    return 0;
}
```

### Why Use `final`?

#### Benefits:
1. **Design intent** - Explicitly show inheritance/override is not allowed
2. **Performance** - Compiler can optimize (devirtualization)
3. **Prevent misuse** - Avoid unintended overriding
4. **API stability** - Lock down behavior in library code

#### Use Cases:
- **Utility classes** that shouldn't be inherited
- **Critical functions** that must not be changed
- **Performance-critical code** where devirtualization helps
- **API design** to prevent extension

[↑ Back to Table of Contents](#table-of-contents)

---

<a id="stl-containers"></a>
## 9. STL Containers

### What Containers are Available in STL?

The C++ Standard Template Library (STL) provides several container types:

#### Sequence Containers
- **`std::vector`** - Dynamic array
- **`std::deque`** - Double-ended queue
- **`std::list`** - Doubly-linked list
- **`std::forward_list`** - Singly-linked list
- **`std::array`** - Fixed-size array (C++11)

#### Associative Containers
- **`std::set`** - Sorted unique elements
- **`std::multiset`** - Sorted elements (duplicates allowed)
- **`std::map`** - Sorted key-value pairs (unique keys)
- **`std::multimap`** - Sorted key-value pairs (duplicate keys allowed)

#### Unordered Associative Containers (C++11)
- **`std::unordered_set`** - Hash table of unique elements
- **`std::unordered_multiset`** - Hash table (duplicates allowed)
- **`std::unordered_map`** - Hash table of key-value pairs
- **`std::unordered_multimap`** - Hash table (duplicate keys allowed)

#### Container Adaptors
- **`std::stack`** - LIFO (Last In First Out)
- **`std::queue`** - FIFO (First In First Out)
- **`std::priority_queue`** - Priority-based queue

---

### What is `std::map` ?

**`std::map`** is an associative container that stores key-value pairs in sorted order (by key).

#### Characteristics:
- Keys are **unique** (no duplicates)
- Keys are **sorted** (typically in ascending order)
- Implemented as a **Red-Black Tree** (balanced BST)
- Time complexity: **O(log n)** for insert, find, erase

#### Basic Usage

```cpp
#include <iostream>
#include <map>

int main() {
    // Create a map
    std::map<std::string, int> ages;
    
    // Insert elements
    ages["Alice"] = 30;
    ages["Bob"] = 25;
    ages["Charlie"] = 35;
    
    // Or using insert
    ages.insert({"David", 28});
    ages.insert(std::make_pair("Eve", 32));
    
    // Access elements
    std::cout << "Alice's age: " << ages["Alice"] << "\n";
    
    // Find element
    auto it = ages.find("Bob");
    if (it != ages.end()) {
        std::cout << "Found: " << it->first << " = " << it->second << "\n";
    }
    
    // Iterate through map (sorted by key)
    std::cout << "\nAll ages:\n";
    for (const auto& pair : ages) {
        std::cout << pair.first << ": " << pair.second << "\n";
    }
    
    // Erase element
    ages.erase("Charlie");
    
    // Size
    std::cout << "\nSize: " << ages.size() << "\n";
    
    return 0;
}
```

#### Output
```
Alice's age: 30
Found: Bob = 25

All ages:
Alice: 30
Bob: 25
Charlie: 35
David: 28
Eve: 32

Size: 4
```

---

### How can you use an User-Defined Objects as Map Keys ?

To use a user-defined object as a key in `std::map`, you must provide a way to **compare** keys (for sorting).

#### Problem

```cpp
class Foo {
private:
    int x;
public:
    Foo(int val) : x(val) {}
};

std::map<Foo, int> myMap;  // ❌ ERROR: No way to compare Foo objects
```

#### Solution 1: Overload `operator<`

The map needs to know how to compare keys. Provide `operator<`:

```cpp
#include <iostream>
#include <map>

class Foo {
private:
    int x;
public:
    Foo(int val) : x(val) {}
    
    // Overload operator< for comparison
    bool operator<(const Foo& other) const {
        return x < other.x;
    }
    
    int getValue() const { return x; }
};

int main() {
    std::map<Foo, std::string> myMap;
    
    myMap[Foo(10)] = "Ten";
    myMap[Foo(5)] = "Five";
    myMap[Foo(15)] = "Fifteen";
    
    // Iterate (sorted by Foo::x)
    for (const auto& pair : myMap) {
        std::cout << pair.first.getValue() << ": " << pair.second << "\n";
    }
    
    return 0;
}
```

#### Output
```
5: Five
10: Ten
15: Fifteen
```

#### Solution 2: Provide Custom Comparator

```cpp
#include <iostream>
#include <map>

class Foo {
private:
    int x;
public:
    Foo(int val) : x(val) {}
    int getValue() const { return x; }
};

// Custom comparator
struct FooComparator {
    bool operator()(const Foo& a, const Foo& b) const {
        return a.getValue() < b.getValue();
    }
};

int main() {
    std::map<Foo, std::string, FooComparator> myMap;
    
    myMap[Foo(10)] = "Ten";
    myMap[Foo(5)] = "Five";
    myMap[Foo(15)] = "Fifteen";
    
    for (const auto& pair : myMap) {
        std::cout << pair.first.getValue() << ": " << pair.second << "\n";
    }
    
    return 0;
}
```

#### Solution 3: Use Lambda as Comparator (C++11)

```cpp
auto comp = [](const Foo& a, const Foo& b) {
    return a.getValue() < b.getValue();
};

std::map<Foo, std::string, decltype(comp)> myMap(comp);
```

---

### What is Difference Between `std::map` and `std::set` ?

| Feature | `std::map` | `std::set` |
|---------|-----------|-----------|
| **Stores** | Key-value pairs | Only keys (values) |
| **Access** | `map[key]` returns value | Check existence with `find()` |
| **Use Case** | Associate data with keys | Store unique sorted elements |
| **Example** | `map<string, int>` (name → age) | `set<int>` (unique numbers) |

#### Example: `std::set`

```cpp
#include <iostream>
#include <set>

int main() {
    std::set<int> numbers;
    
    // Insert elements
    numbers.insert(5);
    numbers.insert(2);
    numbers.insert(8);
    numbers.insert(2);  // Duplicate - ignored
    
    // Iterate (sorted)
    std::cout << "Set: ";
    for (int n : numbers) {
        std::cout << n << " ";  // 2 5 8
    }
    std::cout << "\n";
    
    // Check existence
    if (numbers.find(5) != numbers.end()) {
        std::cout << "5 is in the set\n";
    }
    
    return 0;
}
```

#### Output
```
Set: 2 5 8 
5 is in the set
```

#### Comparison Example

```cpp
// map: Associate names with ages
std::map<std::string, int> ages;
ages["Alice"] = 30;
ages["Bob"] = 25;
std::cout << "Alice's age: " << ages["Alice"] << "\n";

// set: Store unique numbers
std::set<int> unique_numbers;
unique_numbers.insert(5);
unique_numbers.insert(2);
unique_numbers.insert(5);  // Ignored (duplicate)
// Cannot do: unique_numbers[5]  // No indexing
```

---

### What is Difference Between `std::vector`, `std::list`, and `std::deque` ?

| Feature | `std::vector` | `std::list` | `std::deque` |
|---------|--------------|------------|-------------|
| **Structure** | Dynamic array | Doubly-linked list | Double-ended queue |
| **Random Access** | ✓ O(1) | ❌ O(n) | ✓ O(1) |
| **Insert/Remove at End** | ✓ O(1) amortized | ✓ O(1) | ✓ O(1) |
| **Insert/Remove at Beginning** | ❌ O(n) | ✓ O(1) | ✓ O(1) |
| **Insert/Remove in Middle** | ❌ O(n) | ✓ O(1) | ❌ O(n) |
| **Memory** | Contiguous | Non-contiguous | Chunked |
| **Cache Performance** | ✓ Excellent | ❌ Poor | ~ Moderate |
| **Iterator Invalidation** | Yes (on reallocation) | No (except erased) | Yes (on insert/erase) |

#### `std::vector` - Dynamic Array

**Best for:** Sequential access, frequent access by index, adding to end

```cpp
#include <vector>

std::vector<int> v = {1, 2, 3, 4, 5};

// Fast random access
std::cout << v[2] << "\n";  // O(1)

// Fast push_back
v.push_back(6);  // O(1) amortized

// Slow insert at beginning
v.insert(v.begin(), 0);  // O(n)
```

#### `std::list` - Doubly-Linked List

**Best for:** Frequent insertions/deletions anywhere, no random access needed

```cpp
#include <list>

std::list<int> l = {1, 2, 3, 4, 5};

// Slow random access
// l[2]  // ❌ Not available

// Fast insert anywhere
auto it = l.begin();
std::advance(it, 2);
l.insert(it, 99);  // O(1) once you have iterator

// Fast push_front and push_back
l.push_front(0);  // O(1)
l.push_back(6);   // O(1)
```

#### `std::deque` - Double-Ended Queue

**Best for:** Adding/removing from both ends, random access

```cpp
#include <deque>

std::deque<int> d = {1, 2, 3, 4, 5};

// Fast random access
std::cout << d[2] << "\n";  // O(1)

// Fast push/pop at both ends
d.push_front(0);  // O(1)
d.push_back(6);   // O(1)
d.pop_front();    // O(1)
d.pop_back();     // O(1)

// Slower insert in middle
d.insert(d.begin() + 2, 99);  // O(n)
```

#### Complete Comparison Example

```cpp
#include <iostream>
#include <vector>
#include <list>
#include <deque>

int main() {
    std::vector<int> vec = {1, 2, 3, 4, 5};
    std::list<int> lst = {1, 2, 3, 4, 5};
    std::deque<int> deq = {1, 2, 3, 4, 5};
    
    // Random access
    std::cout << "Vector[2]: " << vec[2] << "\n";  // ✓ Fast
    // std::cout << "List[2]: " << lst[2] << "\n";  // ❌ Not available
    std::cout << "Deque[2]: " << deq[2] << "\n";   // ✓ Fast
    
    // Insert at beginning
    vec.insert(vec.begin(), 0);  // ❌ Slow O(n)
    lst.push_front(0);           // ✓ Fast O(1)
    deq.push_front(0);           // ✓ Fast O(1)
    
    // Insert at end
    vec.push_back(6);   // ✓ Fast O(1)
    lst.push_back(6);   // ✓ Fast O(1)
    deq.push_back(6);   // ✓ Fast O(1)
    
    return 0;
}
```

### When to Use Which?

✓ **Use `std::vector`:**
- Default choice for most cases
- Need random access
- Mostly adding to end
- Cache-friendly performance matters

✓ **Use `std::list`:**
- Frequent insertions/deletions in middle
- Don't need random access
- Iterator stability important

✓ **Use `std::deque`:**
- Adding/removing from both ends
- Need random access
- Growing from both sides

[↑ Back to Table of Contents](#table-of-contents)

---

## Summary

This guide covers essential advanced C++ topics:

1. **Copy Constructors** - Deep copying and why `const&` is used
2. **RAII** - Resource management tied to object lifetime
3. **Smart Pointers** - `unique_ptr`, `shared_ptr`, `weak_ptr` for automatic memory management
4. **Move Semantics** - Efficient resource transfer without copying
5. **C++11 Features** - Modern C++ improvements
6. **nullptr** - Type-safe null pointer
7. **Lambda Expressions** - Anonymous functions with various capture modes
8. **final Keyword** - Prevent inheritance and overriding
9. **STL Containers** - Different container types and their use cases

These concepts form the foundation of modern C++ programming!

## Questions

1. Copy constructor:
    1. What is a copy constructor ?
    2. Why its needed ?
    3. Explain why copy constructor takes:  const Foo& for the below class Foo as argument.
   
  	class Foo {
		public:
			Foo(const Foo& obj); // Copy constructor // Why constant reference
  	};

2. What is RAII ? (Resource acquisition is initialization) ?

3. What are RAII features available in C++11 ?
  Ans: Smart pointers

4. What are smart pointers introduced in C++11 ?
  Ans: unique_ptr<T>, shared_ptr<T>, weak_ptr<T>
  
5.  What is the difference between unique_ptr and shared_ptr ?

6. What is the use case for weak_ptr ?

  7. What is move semantics and move constructors, write the move constructor for a class ?

8. What are some of the highlighted features introduced in C++11 ?

9. What is nullptr in C++11 ? How its different from traditional NULL ?

10. What are lambda expressions ? Write the various types of lambda expressions example ?
    1. Following questions what is capture by value ?
    2. What is capture by reference ?
11. What is the new C++11 feature that restricts a class from being inherited into a derived class as a base class.

STL: 
1. What is the different containers available in STL ?
2. What is a std::map ?
3. What special handling has to be done if I need to keep a user defined object as key of a map ?
  For example:
  Class Foo {
  	private:
		int x;   }
  std::map<Foo, int> : Here key is a user defined object.
 4. What is the difference between a map and a set ?
 5. What is difference between a vector, list and duque ?
 



  






