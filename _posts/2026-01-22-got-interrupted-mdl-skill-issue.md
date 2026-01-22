---
title: Got interrupted? Part 3. Page table traversal
description: How to give a middle finger to the memory manager
date: 2026-01-22 00:33:00 +0800
categories: [Virtualization, Windows, Memory]
tags: [virtualization, windows-internals, memory, x86, kernel]
---

## So anyway, about the async

In the [second part](https://staarblitz.github.io/posts/got-interrupted-handle-skill-issue/) of the series "Got interrupted? Skill issue", we have covered how to get objects by their handles and create handles refererring to the objects in VMEXITs, without any call to the object manager.

I realized something, if I can get objects and their handles, and even create them, there is no need for async in HxPosed. Async is nothing but trouble. So I removed everything async. But while refactoring the code, I hit a major roadblock. Memory services.


This article makes a great overview about x86_64 memory management. But if you are unfamiliar with paging concept, I highly recommend you to read AMD64 Architecture Programmer’s Manual, volume 2, page translation and protection.

### Memory Services - Overview.

HxPosed's memory services offer ability to allocate, map, free, unmap, protect, read, write to physical or virtual allocations. You can allocate from non paged pool, map it, write to it, then map it to another process. Or you can read from kernel, or anything you could ask for.

The problem is that memory manager is **highly** IRQL dependent. `MmMapLockedPagesSpecifyCache` for example, requires `APC_LEVEL` for `UserMode` mappings. Which is a luxury we cannot afford at a VMEXIT.

Since removing memory services from HxPosed was not an option, I decided to take a different route.

## First attempt - VADs

VAD stands for Virtual Address Descriptor. It's a weird windows thingy that involes balanced AVL trees and so on. This is how Windows NT describes memory in process address space. When you use `MmMapLockedPagesSpecifyCache`, you can see in `!vad` command in WinDbg that, you have a new VAD that describes the mapped pages.

I won't go into more detail because its really complicated. I first tried to create VADs myself. As you can guess, after 2 days of work and emotional crisis, I gave up. This was simply not feasible. And still IRQL dependent if I were to rely on NT API.

So I decided to do it in the _real_ way. Modifying the page tables.

### What about EPT?

Since HxPosed is a hypervisor, it might seem like a nice option to use EPTs or NPTs. But that is not a good idea. EPTs introduce overhead, and they manage physical -> physical translations. Not virtual -> physical translations which is what we would really need to say for example, map a page.

## Page Tables

Microsoft Windows (generally) uses 4 level paging. Which consist of:

```
CR3 -> Page Map Level 4 -> Page Directory Pointer Table -> Page Directory -> Page Table -> Physical address
```

Each virtual address actually has a meaning.
A virtual address of _X_, is divided into 5 (or 6 if 5-level paging is on) sections.

1. Sign - Rest, unused.
2. PML5 Index - 9 bits (56-48)
3. PML4 Index - 9 bits (39-47)
4. PDP Index - 9 bits (38-30)
5. PD Index - 9 bits (21-29)
6. PT Index - 9 bits (20-12)
7. Physical offset - 12 bits (11-0)

If the system uses 4 level paging, PML5 index field is always 0 and is not walked.

PML5, PML4, and PDP are defined as such:

```c
typedef union _PAGING_ENTRY {
  struct {
    UINT64 Present : 1;
    UINT64 Write : 1;
    UINT64 User : 1;
    UINT64 Pwt : 1;
    UINT64 Pcd : 1;
    UINT64 Accessed : 1;
    UINT64 Ignored1 : 1;
    UINT64 LargePage : 1;
    UINT64 Ignored2 : 3;
    UINT64 Global : 1;
    UINT64 Address : 40;
    UINT64 Reserved : 11;
    UINT64 ExecuteDisable : 1;
  } Flags;
  Value;
} PAGING_ENTRY, *PPAGING_ENTRY;
```

- `Present` - If set, the page is valid.
- `Write` - If set, page is writable.
- `User` - If set, page can be accessed from user mode.
- `Pwt` - (Page Write-Through). If set, write-through caching is enabled. If not, write-back caching is used.
- `Pcd` - (Page Cache Disable). If set, page is not cached.
- `Accessed` - Set by CPU when page is accessed. OS unsets it if it wants to.
- `LargePage` - If set, the page is 4 MiB big.
- `Global` - If set, the processor does not invalidate this page's cache when a MOV-to-CR3 happens.
- `Address` - Page Frame Number of the next entry.
- `ExecuteDisable` - If set, trying to execute code in this page causes a #GP.

PD and PTE have a very similar structure, too. The difference is that it also contains a "dirty" bit.

### Virtual to physical address translation

A virtual address of `0x7ff80bca0000` actually represents these fields:

1. PML5 Index: `0`
2. PML4 Index: `FF`
3. PDP Index: `1E0`
4. PD Index: `5E`
5. PT Index: `A0`
6. Physical offset: `0`

So what does CPU do when it needs to read that address? Simple.

If you want a more visual look, check the little tool I made that gives you more info about pages by their values [PageDisplay](https://github.com/staarblitz/PageDisplay).

1. First, CPU reads the `CR3` register. Which contains physical base to PML4 (or PML5 if 5-level paging) table.
2. To calculate the address from `CR3` register, it masks out the last 12 bits.
3. Then, it calculates the address of which field it should read. `BASE + (8 * PML4Index)`, and reads it from that memory address. This is our PML4.
4. Then, it calculates the address of which field it should read. `BASE + (8 * PDP)`, and reads it from that memory address. This is our PDP.
5. Then, does the same and goes to PD and PT.
6. The `Address` field of PT is the real physical address which the operation will happen in.

Depending on the current privilege level and bits that are set, CPU raises exceptions, or allows the operation.

## Mapping Physical Addresses to Virtual Addresses.

Who cares about VADs when you can modify the `CR3`? Am I wrong?

So, to map the physical address `X` to virtual address `Y`, we need to traverse the page table just like CPU does. But this time, filling it ourselves.

Let's say, we want to map `0x940000` to virtual address `0x1000`. For that case, we need to decode the virtual address. Everything except the PT field is 0, and PT is equal to 1.

So that means, (in a 4 level paging system), we have to do these:

1. Read `CR3`, calculcate the address. Go to there.
2. Calculate which address to read `BASE + (8 * 0)`, so read first entry.
3. This is our PML4. Check if `Present` bit is set.
4. If its not set, allocate from physical memory. If it's set, go to step 7.
5. Calculate the PFN of allocated address. For that, `Physical >> 12`.
6. Set the fields. (User = 1, Write = 1, Address = pfn).
7. Read the `Address` field and extract full address. For that, `Physical << 12`.
8. Calculate which address to read. Again.
9. This is our PML4. check `Present` bit.
10. Boring repeat steps.
11. At last, Set the PT's `Address` field to newly allocated physical memory.
12. Invalidate the page. So we can see it right away. `__invlpg((PVOID)0x1000)`.
13. Boom, memory mapped at high IRQL!

### Mappy mapped?

So writing the prototype code to map our physical address. It should look like this:

Initialization + Pml4. (I have no idea why formatting is so messy, cant't seem to fix)
```c
    CR3 Cr3;
	Cr3.Value = __readcr3(); // from <intrin.h>
    PHYSICAL_ADDRESS TablePA;
    // very important. Cr3's Address field is a PFN. Not the address. We have to shift it.
	TablePA.QuadPart = Cr3.Flags.Address << 12;

    PPAGING_ENTRY Pml4 = (PPAGING_ENTRY)MmGetVirtualForPhysical(TablePA);

    if (!Pml4[0].Flags.Present) {
		PVOID NewTable = ExAllocatePool2(POOL_FLAG_NON_PAGED, PAGE_SIZE, 0x2009);
		PHYSICAL_ADDRESS PhysicalAddress = MmGetPhysicalAddress(NewTable);

        // very important. The Address field, just like CR3, is not the address. It's the PFN.
		ULONG64 Pfn = GET_PFN(PhysicalAddress);

	    Pml4[0].Flags.Write = 1;
	    Pml4[0].Flags.Present = 1;
	    Pml4[0].Flags.User = 1;
		Pml4[0].Flags.Address = Pfn;
	}
```

For Pdpt:
```c
    TablePA.QuadPart = Pml4[0].Flags.Address << 12;

	PPAGING_ENTRY Pdpt = (PPAGING_ENTRY)MmGetVirtualForPhysical(TablePA);

	if (!Pdpt[0].Flags.Present) {
		PVOID NewTable = ExAllocatePool2(POOL_FLAG_NON_PAGED, PAGE_SIZE, 0x2009);
		PHYSICAL_ADDRESS PhysicalAddress = MmGetPhysicalAddress(NewTable);
		ULONG64 Pfn = GET_PFN(PhysicalAddress);

	    Pdpt[0].Flags.Write = 1;
	    Pdpt[0].Flags.Present = 1;
	    Pdpt[0].Flags.User = 1;
		Pdpt[0].Flags.Address = Pfn;
	}
```

For Pd:
```c
    TablePA.QuadPart = Pdpt[0].Flags.Address << 12;

	PPAGING_ENTRY Pd = (PPAGING_ENTRY)MmGetVirtualForPhysical(TablePA);

	if (!Pd[0].Flags.Present) {
		PVOID NewTable = ExAllocatePool2(POOL_FLAG_NON_PAGED, PAGE_SIZE, 0x2009);
		PHYSICAL_ADDRESS PhysicalAddress = MmGetPhysicalAddress(NewTable);
		ULONG64 Pfn = GET_PFN(PhysicalAddress);

    	Pd[0].Flags.Write = 1;
	    Pd[0].Flags.Present = 1;
	    Pd[0].Flags.User = 1;
		Pd[0].Flags.Address = Pfn;
	}
```

And last but not least, the Pd:
```c
    TablePA.QuadPart = Pd[0].Flags.Address << 12;

	PPAGING_ENTRY Pt = (PPAGING_ENTRY)MmGetVirtualForPhysical(TablePA);

	if (!Pt[1].Flags.Present) {
		PHYSICAL_ADDRESS PhysicalAddress = MmGetPhysicalAddress(MyLovelyMemoryAddress);
		ULONG64 Pfn = GET_PFN(PhysicalAddress);

        Pt[1].Flags.Write = 1;
        Pt[1].Flags.Present = 1;
        Pt[1].Flags.User = 1;
        Pt[1].Flags.Address = Pfn;

		__invlpg((PVOID)0x1000);
	}
```

Of course, you have to decode the target address in code as well, but since this is prototype, I went with hardcoding those values instead.

When we run this we get a bugcheck right after setting the address of first entry. Why is that?

### Map Mappy

When we set the address of first entry, we also make it present. So CPU thinks its indeed a valid address, meanwhile we still haven't finished constructing the other entries.

The solution is deferring the setting the Present bit at the end of the function, in reverse order.

I also set other bits because I'm paranoid. But you don't have to.
```c
// do it in reverse order, or else cpu reads our entries and takes whole system down.

Pt[1].Flags.Write = 1;
Pt[1].Flags.Present = 1;
Pt[1].Flags.User = 1;

Pd[0].Flags.Write = 1;
Pd[0].Flags.Present = 1;
Pd[0].Flags.User = 1;

Pdpt[0].Flags.Write = 1;
Pdpt[0].Flags.Present = 1;
Pdpt[0].Flags.User = 1;

Pml4[0].Flags.Write = 1;
Pml4[0].Flags.Present = 1;
Pml4[0].Flags.User = 1;
```

## Real Deal
When we execute `dq 0x1000`, we can see that WinDbg succesfully reads from the target and we get our zeroes.

This way, we can do most of the stuff we have to (except reading and writing, which can be handled via mapping), in high IRQLs, without using memory manager at all!

Thanks for reading!
