---
layout: post
title:  "Memory aligned malloc"
date:   2019-06-01 21:52:07 +0530
categories: cpp programming
---

A very common interview question is to allocate memory aligned to given block size, such as `memalign_malloc(size_t bytes, size_t align)`. Despite being common, I have found that not many people really understand the implementation, including the interviewer. In practice, one would either totally avoid such aligned allocation (relying on machine natural alignment using `malloc`) or use native [aligned_alloc(), or memalign(), or posix_memalign()](https://www.gnu.org/software/libc/manual/html_node/Aligned-Memory-Blocks.html) family.

When allocating memory through `malloc` (or its cousins), it returns a block of memory which is multiple of 8 or 16 for 32 or 64-bit system, resp. However, in some special cases such alignment might not be enough such as in an embedded system, while implementing your own memory manager (e.g. GC), for allocating specifically on page boundary, while interfacing with certaing I/O hardware, or for CPU cache alignment, and other such cases. Hence, there exists `memalign` and similar routines.

To implement `memalign_malloc`, our first intution might be to allocate requested size block plus extra memory to adjust for misaligned address from `malloc`. However, if we return this new aligned address then while using `memalign_free` we might leave a hole between `malloc` returned address and aligned address, and possibly corrupt memory too. We need a place to save the original address as that is what should be used by `free` which would be eventually called by `memalign_free`. Most memory allocators keep some metadata associated with each block of allocated memory which might contain block size, actual address, access/proection info, etc. For `memalign_malloc`, we would only store the original address as returned by `malloc` at some location. Our complementary `memalign_free()` would know where to pick this address from and eventually call `free` using this address.

How much extra space should we allocate? It could be that the address returned by `malloc` needs adjustment by upto `(align - 1)`. So, we need to allocate `(align - 1)` extra bytes just for this adjustment. Also, we need to save the original address as returned by `malloc`, which needs space equal to the size of a pointer for underlying machine. Total memory we allocate is -

```c
size_t extra = (align - 1) + sizeof(void *);
void *mem = malloc(bytes + extra);
```
<img src="/static/img/memalign-a.png" alt="">

The trick for aligning an address is by masking the lower bits of address which correspond to alignment size such as `address & ~(align - 1)`. So, if we were trying to align to 16 bytes, ~(16 - 1) or a mask of 0xF is required, since inverting only those bits and 'and'ing would mean those many last bits would be 0 in address, i.e. new address will be multiple of 16. Note, that due to this calculation the alignment size can only be for power of 2. The new aligned address is going to be somewhere between `mem` and `mem + (align-1)`. But we also need to save address `mem` and best location would be to store it right before new aligned address. So, best would be to find aligned address from `mem + sizeof(void *)` to `mem + extra`. Our new address will become `(mem + extra) & ~(align-1)`. But that's not enough - `mem` is pointer so we can't do normal arithmatic with it. To circumvent that, we can typecast it to uintptr_t.

```c
void *new_address = ((uintptr_t)mem + extra) & ~(align-1));
```

<img src="/static/img/memalign-b.png" alt="">

Now only remaining piece is how to save `mem`? We already know it must be saved right before new_address for easy acccess as well as to avoid being trampled by data. A neat trick is to treat this block of memory as an array to pointers, and then if arr[0] points to new_address, arr[-1] is where we can store `mem`. When treating block as an array, when we access previous or later element of array, stride will be exactly the size of element (in out case it would be void \*), so now we dont need to typecast, add, subtract, etc. With that new address can also be seen as array of pointers.

```c
void **ptr_array = (void **)(((uintptr_t)mem + extra) & ~(align-1));
```

Notice use of `(void **)`. In C, array is nothing but contiguous block of memory. Its only when this block is accessed, one can access it as array (and move as much as size of element) or regular bytes. So array of interegers can be represented as `int arr[]` or `int *arr`. Hence, an array of `(void *)` can be `void *arr[]` or `void **arr` and still one can use arr[0] format. That means in above code snippet, we can use ptr_array[-1] and it'll be essentially `(ptr_array - sizeof(void *))`. That's where we can save `mem`.

```c
ptr_array[-1] = mem;
```
<img src="/static/img/memalign-c.png" alt="">

To free this memory block, all we need to do is `free((void **)addr[-1])`. With that our final code looks like following -

```c
void *memalign_malloc(size_t bytes, size_t align)
{
    size_t extra = (align - 1) + sizeof(void *);
    void *mem = malloc(bytes + extra);
    void **ptr_array = (void **)(((uintptr_t)mem + extra) & ~(align-1));
    ptr_array[-1] = mem;
    return ptr_array;
}

void memalign_free(void *addr)
{
    if (addr) {
        free(((void **)addr)[-1]);
    }
}
```
