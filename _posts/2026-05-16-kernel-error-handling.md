---
layout: post
title: "Linux Kernel Error Handling"
date: 2026-05-15 12:00:00 -0000
categories: kernel
---

## Kernel error codes
Today's topic is a short but important one. Recently I have been dabbling into linux kernel drivers and I find the error code convention very exciting. In linux kernel the error codes are defined in:

- https://github.com/torvalds/linux/blob/master/include/uapi/asm-generic/errno-base.h
- https://github.com/torvalds/linux/blob/master/include/uapi/asm-generic/errno.h

The ones I use on a regular basis are as follows: EPERM, EIO, ENOMEM, ENODEV, EINVAL and ENXIO. An important point to note is that the error codes defined here are positive values and hence when returning these codes we need to negate them.

## Transactional behaviour
Another important point to keep in mind is that when a driver hits any of these error values, we must ensure that the driver is reverted to a healthy state. An obvious example is when a driver requests memory but the system cannot fulfill the request. In this case, along with returing ENOMEM, we must free any previous allocation that was made as a part of this call. 
In other words, a linux kernel should treat a call like a transaction and if a call fails, the kernel must gaurantee a return to the initial state. To implement this developers often use the `goto` keyword after an error is detected.

```
int my_driver_init(struct my_device *dev) {
    int ret;

    ret = allocate_resource_A(dev);
    if (ret) return ret; // No cleanup needed yet

    ret = allocate_resource_B(dev);
    if (ret) goto err_free_A;

    ret = allocate_resource_C(dev);
    if (ret) goto err_free_B;

    return 0; // Success!

err_free_B:
    free_resource_B(dev);
err_free_A:
    free_resource_A(dev);
    return ret;
}
```

## Handling NULL pointer errors
Functions that return a pointer often return a NULL value to indicate a failure. This is eventually handled by the original caller, however, the caller does not get a fine-grained failure cause since each type of failure will return a NULL. Linux kernel has the following functions that can help us make these more meaningfull:
- `void *ERR_PTR(long error);`: Converts err codes to a pointer
- `long IS_ERR(const void *ptr);`: To check if the pointer returned is an error pointer
- `long PTR_ERR(const void *ptr);`: To convert the err pointer to an error code

Using the methods declared above a kernel function can encode the return pointer to carry one of the fine grained error messages we discussed in the above sections and the caller method can identofy and decode it using `IS_ERR` and `PTR_ERR` respectively.
There is another method worth mentioning here: `long IS_ERR_OR_NULL(const void *ptr)`. This is a blanket check that returns true if the pointer is either NULL or is an error pointer described above.

## How does ERR_PTR work?
This is a theoretical dive into the mechanics of ERR_PTR, in practice I don't think this matters. Given the disclaimer, its a pretty exciting mechanism. 
The kernel reserves the very highest page of the virtual address space (typically the last 4096 bytes, from `0xfffff000` to `0xffffffff`) specifically for error codes.
Because this memory is reserved, no valid pointer will ever point there. IS_ERR() simply checks if the pointer's address falls within this specific, tiny range.

---
> This blog post is inspired from reading chapter 2 of "Linux Device Driver Developement" by John Madieu!



