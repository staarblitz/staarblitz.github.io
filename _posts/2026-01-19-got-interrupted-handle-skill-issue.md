---
title: Got interrupted? Part 2. Tales of two handle tables.
description: How to give a middle finger to the object manager
date: 2026-01-19 00:33:00 +0800
categories: [Virtualization, Windows]
tags: [virtualization, windows-internals, kernel]
---

## Disappointment
After the disappointment from the first post, and after *having* to redesign the kernel-side of the async system, I decided to give a part 2 for this "got interrupted" series. Which we will be investigating ways to go our way around IRQL limitations in VMEXITs. See the related issue [here](https://github.com/staarblitz/hxposed/issues/8)

### How to NOT deal with high IRQL
What I did for HxPosed was simply a global queue which defined which task to execute in which process context. It went well, until it didn't. That was when I was making the callbacks feature. Which were simply `PsSetCreateProcessNotifyRoutineEx` wrappers that queue a command to be queued (yeah, I regret that) to be executed.

The order was simple for *almost* any async function.
1. Check if call is async.
2. If async, construct the `InsertNameHereAsyncCommand` object.
3. Box it.
4. Acquire global lock queue.
5. Queue it.
6. Return an `EmptyResponse`.

And a worker thread running at `PASSIVE_LEVEL` IRQL was using `KeDelayExecutionThread` with 250ms delay. It was messy.
1. Wait for 250ms.
2. Acquire lock for global async queue.
3. Check if there is a new AsyncCommand.
4. If no, go to step 1.
5. If yes, drop the lock, switch to context of process made the call.
6. "Work" the command.
7. Switch context to the process made the async call. (Yes! Twice!)
8. Probe the shared memory region.
9. If writable, write result to the memory region.
10. Set the event using `ZwSetEvent`.

Do you see the problem here? There are multiple.
1. Use after free.
2. Heavy reliance on process contexts.
3. Poll-based queue. Not event based. Slowwwww.

But what I could do? The async was simply passing a shared memory region and a handle to the event.

No more.

#### UAF
This was not present until the callback system arrived. Which heavily relies on time outs.

When the user allocates the buffer and sends the request, but cancels it (via dropping the Future), it does not inform hypervisor that the request is cancelled. The memory is freed. Thus, hypervisor writes to it.

But how does `ProbeForWrite` return Ok then?

Because memory is still writable. That memory was from heap, and even though it was freed, it wasn't unmapped, it was still in process's address space. Thus, perfectly valid. Except that it isn't.

So boom. Heap corruption. We need MDLs.

### Refactoring the async system.
So we had to get rid of excessive `KeStackAttachProcess` calls and a way to stop the UAF.

So I came with this idea:
Using MDLs and opening the native kernel event object instead of using `Zw*` functions.

But how?

## Back to the IRQL.
So, we need to open the object itself from handle. Easy thing. We have a function exactly made for that purpose. `ObReferenceObjectByHandle`. And guess what?

![MSDN](/assets/img/uploads/img9.png)

`PASSIVE_LEVEL`. I don't understand Microsoft's desire to page everything. Hell, they even have a function named as `MmPageEntireDriver`. Just... What is the point? Why hurt the SSD? Just keep things in memory.

Anyway, after ranting enough. I decided to take a deep dive into Windows internals to figure out how to get the object by myself. After all, a handle is just an index in a per process table, right?

## The tale of 2 handle tables.
A process' handle table is stored at `ObjectTable` field of `EPROCESS` structure. Which itself is a `PHANDLE_TABLE`. It's a complex structure by itself. It grows depending on the demand of handles and has 3 states. Small, medium, large (large can hold up to 2 million handles!).[^1]

So let's see what that bad boy has to offer for us.


[^1]: I'm not exactly sure if that was the case. I have read this from some obscure PDF. But yeah, probably that's correct since we have `TableCode` field and its bitmasked usage via `ExpLookupHandleTableEntry`.

![Fields of _HANDLE_TABLE](/assets/img/uploads/img10.png)

Uh huh. Huh huh. Mhm. Yes. We can work with that.

So we have a `HandleTableList` structure which is *supposedly* holding a doubly linked list to `_HANDLE_TABLE_ENTRY` structures. Which itself is extremely messy.

We can see that it's not <u>correct</u> given that `_HANDLE_TABLE_ENTRY` does not have a `ListEntry` field or such.

![No ListEntry field](/assets/img/uploads/img11.png)

### Traversing the handle table manually
To get an inspiration of what to do, let's take a look at what `ObReferenceObjectByHandle` is doing.

![Interesting call](/assets/img/uploads/img12.png)

That might be what we are just looking for.

![Decompilation](/assets/img/uploads/img13.png)

After some digging through WRK, we can extract out its parameters and return value. It 

`ExpLookupHandleTableEntry` takes a pointer to `_HANDLE_TABLE` and an `EXHANDLE`.
`EXHANDLE` is just a fancy way of "HANDLE" with some extra flags like kernel or some. It doesn't bother us. We can use a normal handle value as an `EXHANDLE`. And it returns an `_HANDLE_TABLE_ENTRY`

#### But wait, that's not manual!
It's not. Because traversing handle table manually is pain. Somehow I was never getting the correct entry no matter what I do. I did a lot of research on that, but didn't turn out to be right. So yeah, we will use `ExpLookupHandleTableEntry`. It's *safe* to assume that this won't change in future versions of Windows. Since it's been pretty stable so far. And this saves us from having to get offsets or define `_HANDLE_TABLE`. Yay!

### So how do we get the object?
As we've seen from above, _HANDLE_TABLE_ENTRY has an *interesting* field named as `ObjectPointerBits`. It also has a `GrantedAccessRights`, which was a nice target for a lot of people. But not what our point is. HxPosed gives you kernel access anyway. Who cares about handles?

Ehm. Anyway. As you see from definition, bits 20-44 is what we are looking for. We can quickly whip up a function to read those bits.

There is a caveat tho. This is NOT the full address. We have to add 0xffff's to beginning of it. So we have to do some math `object_pointer << 4 | 0xffff000000000000`.

### All good, for now.

The ObjectPointerBits field returns the `OBJECT_HEADER`. Not the object itself. Nice for us, the object body itself is **exactly** right after the `OBJECT_HEADER` structure. So if we advance 0x30 bytes from the `OBJECT_HEADER`, we will get the object!

```rust
let exhandle = _EXHANDLE {
    Value: handle
};
let handle_table_entry = unsafe{ExpLookupHandleTableEntry(table, exhandle)};
if handle_table_entry.is_null() {
    // invalid handle
    return Err(());
}

let object_pointer = unsafe{*(handle_table_entry)}.get_bits(20..64);
let object_header = (object_pointer << 4 | 0xffff000000000000) as *mut u64; // decode bitmask to get real ptr

// object body is always after object header. so we add sizeof(OBJECT_HEADER) which is 0x30 to get object itself
let object_body = unsafe{object_header.byte_offset(0x30)} as *mut T;
```

### Not all good.

We referenced the object, without actually referencing it. We have the object but we didn't tell Windows that we are using it. We need to modify the `PointerCount` field of the `OBJECT_HEADER` structure. So Windows won't delete it mid-operation.

```rust
// TODO: Make this atomic.
pub fn increment_ref_count(object: *mut u64) {
    let header = unsafe{object.offset(-0x30)};
    unsafe{header.write(*header +1)};
}

pub fn decrement_ref_count(object: *mut u64) {
    let header = unsafe{object.offset(-0x30)};
    unsafe{header.write(*header -1)};
}
```

Simple as that.

### The Real Deal

After testing, we see that we easily get a pointer to the object, in a VMEXIT! Yuppy!