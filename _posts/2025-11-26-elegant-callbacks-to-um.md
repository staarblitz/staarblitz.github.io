---
title: KeUserModeCallback. Elegant way to wait for kernel.
description: How to utilize (hijack) ntdll to get elegant and nice callbacks from your kernel.
date: 2025-11-26 13:07:00 +0800
categories: [Internals, Windows]
tags: [windows-internals, c, x86, asm]
---

## It always have been an issue
Long gone the days of sharing event handles between UM and KM. Let alone inverted call model OSR suggests us to use. Don't you want something more.... *elegant*?

## The hidden gem in PEB
The PEB has a lot of juicy stuff we can utilize. But something I always wondered about (and never actually researched about) is the `KernelCallbackTable`. It is an array of PVOIDs that each is a redirector by their indices, which are the API numbers.

So to find pointer to the callback by it's API number, we just have to do simple math:
```math
APINUM * 8 + CallbackTableAddress
```

Where the `APINUM` is the index of the function we want to call. `CallbackTableAddress` is value of the `KernelCallbackTable` field of PEB (no shit).
The expression can be simplified in WinDbg using the `apfnDispatch` symbol, exported in `user32.dll`:
```math
u poi((@r8 * 8) + user32!apfnDispatch)
```

R8 register will contain the API number that will be loaded from stack at `ntdll!KiUserCallbackDispatcher`, right before call to `KiUserCallForwarder`.

## Why, though?
Win32k GUI driver needs a way to inform the graphical applications (WndProc, especially) about graphics event that happen. If you do a `dps user32!apfnDispatch` (don't forget to force load user symbols with `.reload /user /f`!) when you are in the context of a GUI process, you will see that the callback table is populated by callbacks to user32.dll.

Microsoft makes all their gem hidden beneath undocumented functions. Sigh.

## Elegancy, someone said?
Of course, we can be a good boy, allocate virtual memory, copy the table, add our entry, replace the `KernelCallbackTable` field, and release the lock to add our own callback into the table. But it has its quirks.

### GUI applications need to register their callbacks
Let's find a process that does not have user32.dll loaded.
![cmd.exe's Modules tab from System Informer](/assets/img/uploads/img.png)

Mmmm.. Clean and beautiful. Just like how we like it.
Let's see its KernelCallbackTable with help of WinDbg.
![cmd.exe's PEB](/assets/img/uploads/img2.png)

Let alone having user32 callbacks, it isn't even allocated yet!

This brings us to an issue, do we have to check for PEB periodically, or use `WaitOnAddress` with a worker thread, or some weirder solution?

Because when user32 loads (implicitly or explicitly), it's going to DESTROY our custom, beautiful little table and put its weirdo win32k callbacks.

### Our callback's API num will not be steady

After user32 loads, we need to reallocate the callback table. And thus, first few ten entries are unusable for us. Of course, we can get around it by allocating more than we need. But who knows how big the number of APIs for win32k will be?


Given this information, we can see that "good boy" tactics are impractiical. Thus, we will use a more sophisticated (and worse) manuaver to make the event-based communication between kernel to user mode possible.

## Hooking ntdll!KiUserCallbackDispatcher
Because why not?
If we look at memory section where ntdll is located in, we will see that its protected with write copy access rights.
![cmd.exe's Memory tab](/assets/img/uploads/img3.png)

What this means is (with bonus info):

- Ntdll is allocated ONCE and for all.
- If a process wishes to change ntdll’s contents, its reallocated at its address space and changes only affect the current process.
- We can hook ntdll :lol:

So how? It’s no different than our usual hooking procedure. Of course, you can waste some of your hours (or take your existing work) for trampoline hooking a function. But I won’t, and I will use MinHook instead. It’s a great library. And that is all we need.