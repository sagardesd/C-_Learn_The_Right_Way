# C++ Constructors and Destructors - Complete Guide

## What are Constructors?
Constructors are special member functions that share the same name as their class.

### Common Misconception
Many people believe constructors create objects, but this isn't accurate.

### What Constructors Actually Do
Constructors are special functions designed to **initialize** an object immediately after it has been created. 
When an object is instantiated, memory is first allocated for it, 
and then the constructor is automatically invoked to set up the object's initial state(assigning values to member variables), 
allocating resources, or performing any other setup operations needed before the object is ready to use.

### Key Points:
- **Object creation** (memory allocation) happens first
- **Constructor invocation** (initialization) happens immediately after
- Constructors ensure objects start in a valid, well-defined state
- They are called automatically—you don't invoke them manually

---

## What are Destructors?
Destructors are special member functions that have the same name as the class, but prefixed with a tilde (`~`).

### What Destructors Actually Do
Destructors are special functions designed to **clean up** an object just before it is destroyed. When an object goes out of scope or is explicitly deleted, the destructor is automatically invoked to perform cleanup operations—releasing dynamically allocated memory, closing file handles, releasing locks, or performing any other necessary cleanup before the object's memory is deallocated.

### Key Points:
- **Destructor invocation** (cleanup) happens first
- **Object destruction** (memory deallocation) happens immediately after
- Destructors ensure proper resource cleanup and prevent memory leaks
- They are called automatically when an object goes out of scope or is deleted
- A class can have only **one destructor** (no overloading, no parameters)
- Destructors are called in **reverse order** of object creation

---

## Example
```cpp
#include <iostream>

class Foo {
    private:
        int member;
    public:
        /* Default Constructor */
        Foo() { 
            std::cout << "Foo() invoked\n"; 
        }
        
        /* Parameterized constructor */
        Foo(int a) {
            this->member = a;
            std::cout << "Foo(int a) invoked\n";
        }
        
        /* Destructor */
        ~Foo() {
            std::cout << "~Foo() invoked\n";
        }

        void print_obj() {
            std::cout << "Object Add: " << this << ": member : " << this->member << std::endl;
        }
};

int main(int argc, char* argv[]) {
    Foo obj1; // Default constructor invoke
    obj1.print_obj();
    
    Foo obj2(2); // Explicitly Parameterized constructor invoked
                 // Explicit conversion
    obj2.print_obj();
    
    Foo obj3 = 10; // Parameterized constructor will be invoked 
                   // Implicit type conversion from int to Foo Type 
    obj3.print_obj();
    
    return 0;
    // Destructors are called here automatically in reverse order: obj3, obj2, obj1
}
```

```
output:
➜  g++  -O0 -fno-elide-constructors  constructor_example.cpp
➜  ./a.out       
Foo() invoked
Object Add: 0x16f64704c: member : 1
Foo(int a) invoked
Object Add: 0x16f647038: member : 2
Foo(int a) invoked
~Foo() invoked
Object Add: 0x16f647034: member : 10
~Foo() invoked
~Foo() invoked
~Foo() invoked
```
