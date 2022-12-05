---
title: implementation of std::any in libc++
date: 2022-12-04 18:08:01
categories: blog
tags: 
- c++ 
- libc++
---

# What Is `std::any`

`std::any` is a new feature that comes with C++ 17 standard. It's kind of `void*` with type-safety, supporting copy/move/store/get, etc. Some basic usages of it can be found here: [std::any - cppreference](https://en.cppreference.com/w/cpp/utility/any)

Understanding how `std::any` is implemented can be gainful, for it taking advantage of many c++ skills, especially templates.

# Implementation of `std::any`

The source code referred to below is <any> of llvm's libcxx library, revision: [any - libcxx](https://github.com/llvm/llvm-project/blob/eb7d16ea25649909373e324e6ebf36774cabdbfa/libcxx/include/any).

## Class Layout

`std::any` has two data members:

1. `__h_: _HandleFunPtr`, it's a function pointer that points to a static function, as well as the entry of data manipulation including constructing/destroying/copying/access, etc. Prototype of the function is:

   ```cpp
   /*******
   * arg1: Enum, kind of the manipulation, (_Destroy, _Copy, _Move, _Get, _TypeInfo)
   * arg2: Caller's "this" pointer
   * arg3: Destination any, used in copy, move
   * arg4: Runtime type info, always nullptr if rtti is disabled
   * arg5: Fallback type info, described in next chapter
   *******/
   using _HandleFuncPtr =  void* (*)(_Action, any const *, any *, const type_info *,
                                           const void* __fallback_info);
   ```

2. `__s_: _Storage`, stores pointer to managing data. It's declared as a union for separately handling large and small objects.

In conclusion, `std::any` is basically equal to an aggregate of a data block and a predefined manipulation function, which proves the famous saying **Algorithms + Data Structures = Programs** to some extent.



## Skills of Implementation

### Small objects optimization

> Implementations are encouraged to avoid dynamic allocations for small objects.    -- cpp refrence

The data pointer `__s_` is not a `void*` but privately declared as a such union:

```cpp
using _Buffer = aligned_storage_t<3*sizeof(void*), alignment_of<void*>::value>;
union _Storage {
    constexpr _Storage() : __ptr(nullptr) {}
    void *  __ptr;
    __any_imp::_Buffer __buf;
};
```

In a 64-bit machine, `_Storage` occupies 24 bytes. `__buf` is equally a `void*`, used when the contained object is no larger than 24 bytes. Utilizing the benefits of stack memory, constructing or copying these small objects could be more effective. Larger objects, on the other hand, have to be stored on heap memory and allocated dynamically in runtime.

"24 bytes" is a curated threshold that is exactly the size of `std::vector` /`std::string` and many other STL containers in libcxx. This fact means `std::any` can manage these common objects faster, though the memory inside them could still be dynamic.

A similar memory optimization technology is also applied on `std::string`, but subtler. I will introduce it in the future.



### In-place Construct

When a `std::any` object is copied, the object managed by it is also copied. It's pretty straight yet important, simply `memcpy` is not enough because some classes have essential things to do, such as `std::shared_ptr`. `std::any`s copy object with its own copy constructor through allocator. It also applies to move construction.

But what about the constructor itself? Like `emplace_back` for `std::vector`, `std::any` also has an emplace-like constructor, in which the object is directly constructed on `__buf` instead of constructing a temporary object and then moving it.

```cpp
std::any a(std::in_place_type<std::string>, "hello");
std::any b("hello");
```

By inspecting these 2 variables in a debugger, we can acknowledge that `std::in_place_type<std::string>` has two folder meanings, it tells `std::any` constructing a `std::string` instead of a `const char*` and constructing it directly.



### Type to int mapping

Obviously, `std::any` is not a template class itself, but it can throw exceptions when casting it to a different static type even if RTTI(run-time type info) is disabled. The secret of this type-safety is a mapping from type to an integer.

On line 162 of `any.h`, a type-unique template struct is defined as:

```cpp
template <class _Tp>
struct __unique_typeinfo { 
    static constexpr int __id = 0; 
};

// get type id if rtti is disabled
template <class _Tp>
inline constexpr const void* __get_fallback_typeid() {
    return &__unique_typeinfo<remove_cv_t<remove_reference_t<_Tp>>>::__id;
}
```

The static member `__id` of `__unique_typeinfo` is always equal to 0 but is a unique instance corresponding to type `__Tp` due to template specialization. Based on this, `std::any` gets the address of `__id` as a fallback type id if RTTI is disabled (compiling with flag `-fno-rtti`).



# Best Practices

In conclusion, the best practices of `std::any` include:

- To avoid unnecessary copying, use `std::make_any` or `std::in_place_type` to construct.
- Pass `std::any` by reference if possible.
- Use pointer version `std::any_cast<T>(&a)` to avoid copying large objects.
- Let custom objects conform to [the rule of three/five/zero](https://en.cppreference.com/w/cpp/language/rule_of_three) if managed by `std::any`.



# Formatting of `std::any` in LLDB

Both belonging to Project LLVM, LLDB does not provide a formatted display of `std::any` of libcxx (while GDB does with libstdc++). Printing `std::any` in LLDB CLI will get:

```Shell
(lldb) n
Process 76235 stopped
* thread #1, queue = 'com.apple.main-thread', stop reason = step over
    frame #0: 0x0000000100003a3c Play`main at main.cpp:17:5
   14          int main() {
   15              std::any a = 1;
-> 16              return 0;
   17          }
(lldb) fr v a
(std::any) $0 = {
  __h = 0x0000000100003d2c (Play`std::__1::__any_imp::_SmallHandler<int>::__handle(std::__1::__any_imp::_Action, std::__1::any const*, std::__1::any*, std::type_info const*, void const*) at any:350)
  __s = {
    __ptr = 0x0000000000000001
    __buf = (__lx = "\U00000001\0\0\0\0\0\0\0\U00000001\0\xc1\x89FͽV\0:\0\0\U00000001")
  }
}
```

It takes seconds to understand "a" is an int (from _SmallHandler's type) and its value is 0x1 (from first 4 bytes of `__buf`). I write a Python script as a plugin based on LLDB API to print it more intuitively. 

<script src="https://gist.github.com/TsaiHao/a0abaaa7272c917d5fc00e2bdc676969.js"></script>

## Implementation of plugin

There are 2 functions inside this script. `__lldb_init_module` is the entry of this plugin. `handle_std_any` is the processing handle used by LLDB.

A tricky skill is catching the type name inside `_SmallHandler` using regex. After knowing that, we can find the object representing this type in python and then forcibly convert the pointer of buffer to it. This script should work for integers, floats, and `std::string`, but not very robust now. I will continually polish it.

> The same type-catching trick can also be employed in c++ source code. `__PRETTY_FUNCTION__` macro carries type name of a template function, so you can do some static reflections with it. A famous example is [Magic Enum](https://github.com/Neargye/magic_enum).

Another noticeable thing is that public classes of libcxx need special treatment because they have an inline namespace `__1`.