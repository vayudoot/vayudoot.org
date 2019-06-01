---
layout: post
title:  "Memory aligned malloc"
date:   2019-06-01 21:52:07 +0530
categories: cpp programming
---

A very common interview question for systems team is to allocate aligned memory, where size and alignment boundary is provided. Despite being common, I have found that not many people really understand the implementation. This is clearly just an interview question since in real life you would either totally avoid such manual alignment (and rely on machine natural boundary) or use libc's native [aligned_alloc(), or memalign(), or posix_memalign()](https://www.gnu.org/software/libc/manual/html_node/Aligned-Memory-Blocks.html).

The typical case of allocating memory through `malloc` (or its cousins) returns block of memory which is multiple of 8 or 16 for 32 or 64-bit system, resp. On a embedded system, or when implementing your own memory manager, or when need page boundary (4k), or interfacing with certaing I/O hardware, or for CPU cache alignment, `malloc`'s hardcoded alignment might not be enough.

Our first intution might be to allocate requested size plus extra memory to adjust for misaligned address from `malloc`. However, if we return the new aligned address from say, `memalign_malloc()`, using `free` on this address might leave a hole between `malloc` returned address and aligned address, and possibly corrupt memory too. We need a way to save the original address as returned by `malloc` too and it's what should be used by `free` even though we would call `memalign_free`. Most memory allocators keep some metadata associated with each block of allocated memory, which might contain block size, actual address, access/proection info, etc. In our solution, we would do the same storing just the original address as returned by `malloc`. Our complementary `memalign_free()` would know where to pick this address from then call system specific `free`.

How much extra space should we allocate? The address returned by `malloc` could be off by `(alignment - 1)` at most. So, we need to allocate `(alignment - 1)` bytes just for this. Also, we need to save the original address as returned by `malloc`, which will be of size of pointer on machine.

```c
size_t extra = (align - 1) + sizeof(void *);
void *mem = malloc(bytes + extra);
```
<img src="/static/img/memalign-a.png" alt="">

The aligned address is going to be somewhere between `mem` and `mem + (align-1)`. But we also want to save the `mem` and best place would be to save it right before new aligned address, as this is the address which we pass around. So, best would be to find aligned address from `mem + sizeof(void *)` to `mem + (align-1) + sizeof(void *)`. Easiest way to find alignment would be 

```c
address && ~(align - 1)
```

So, if we were trying to align to 16 bytes, ~(16 - 1) will be 0xF. Inverting only these bits (~) and 'and'ing would mean those many last bits would be 0 in address, i.e. new address will be multiple of 16. Note, that due to this calculation the alignment size could be only for power of 2. Our new address will become `(mem + extra) & ~(align-1)`. But that's not enough - `mem` is pointer so we can't do normal arithmatic with it. To circumvent that, we can typecast it to uintptr_t.

```c
void *new_address = ((uintptr_t)mem + extra) & ~(align-1));
```

<img src="/static/img/memalign-b.png" alt="">

Now only remaining thing is how to save `mem`? We already know it must be saved right before new_address for easy acccess as well as to avoid being trampled by data. A neat trick is to treat this block of memory as an array to pointers, and then if arr[0] points to new_address, arr[-1] is where we can store `mem`. When treating block as an array, when we access previous or later element of array, stride will be exactly the size of element (in out case it would be void \*), so now we dont need to typecast, add, subtract, etc. With that new address can also be seen as array of pointers.

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
