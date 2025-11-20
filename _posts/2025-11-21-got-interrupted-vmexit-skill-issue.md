---
title: Got interrupted? Skill issue. How to deal with 0xFF IRQL on VMEXITs.
description: How to cope with inability to call any NT function in a VMEXIT.
date: 2025-11-21 00:33:00 +0800
categories: [Virtualization, Windows]
tags: [virtualization, kernel, x86]
---

## What is IRQL, even?
IRQL stands for Interrupt Request Level, is an NT thingy. It's stored on CR8 and those fancy functions' (`KeRaiseIrql`, `KeLowerIrql`) purpose is to write to it.

An IRQL essentailly defines how "important" the currently running code is. For example, after IRQLs `DISPATCH_LEVEL` and above, Windows Scheduler cannot interrupt the running core.

Some most commons are (ordered by prioirity):
1. `PASSIVE_LEVEL` - lowest level of IRQL. Every user-moded thread runs in this IRQL.
2. `DISPATCH_LEVEL` - Some kind of "mid" level. The task is not that important but important enough to know that it should not be scheduled in half.
3. `HIGH_LEVEL` - highest level of IRQL (known to Windows). Very, very important tasks (no examples coming to my mind)

After `DISPATCH_LEVEL` and above, paging is disabled since page fault recovery requires a low IRQL. This is why touching paged memory in high IRQLs often end in tears.

## So what do we do about it?
In a VMEXIT, as Intel manual states, every kind of interrupt is disabled. LAPIC, disabled, device interrupts, gone, clock interrupts, gone.
Even IPIs are gone. I think you know what I mean now.

This level of IRQL restricts **almost** every useful NT function. From `PspTerminateProcess` to `ZwOpenProcess`, as I stated in [this issue](https://github.com/staarblitz/hxposed/issues/2). 
Unfortunately, the world isn’t all flowers and rainbows, and you can’t just call `KeLowerIrql` whenever you feel like it. Even if you *somehow* managed to do it, remember that we are in a VMEXIT.

So what is our best bet? Pending them.

## Cancel safe queues.... But not for IRPs.
Cancel safe queues are amazing. It's a way for user-mode applications to wait for kernel mode driver to respond until something interesting happens. You can learn more about it [here](https://learn.microsoft.com/en-us/windows-hardware/drivers/kernel/cancel-safe-irp-queues#ddk-implementing-the-cancel-safe-irp-queue-kg).

What makes them interesting to us is the logic Microsoft uses in their sample code for cancel safe queues. [Here](https://github.com/microsoft/Windows-driver-samples/blob/main/general/cancel/sys/cancel.c) it is.

What? You didn't think I was going to setup an I/O Csq for a hypervisor, right?

Right?

## Worker threads in kernel mode
First, as it looks like in the polling thread code, we simply neeed a queue (which Rust provides, saving us from `LINKED_LIST`s), a worker thread in `PASSIVE_LEVEL`.
But which primitive shall we use?

## Sync primitives in NT
Basically, in NT architecture, we have waaaay more sync primitives than any of us could ever wish. Ready for your any weird IRQL levels and speed requests.
MSDN provides a document we can read to learn about them in detail. [Here](https://learn.microsoft.com/en-us/previous-versions/windows/hardware/design/dn613998(v=vs.85)?redirectedfrom=MSDN).

But if you hate Microsoft documentation (I do), the OSR also provides a very, very good explanation of them at [here](https://www.osr.com/nt-insider/2015-issue3/the-state-of-synchronization/).

### Choosing the right type.
The section "asking the right questions" is exactly what you should do when pickingg a sync primitive in NT. Let's see what our questions are and what are their answers.

*Is it okay to use a wait based lock?* - No. The VMEXIT cannot be waited.

*Do I need to support recursion in my synchronization logic?* No. A core can only be executing in host mode or guest mode. Not both.

*Will the data sometimes be concurrently accessed only for read semantics, with only some uses requiring write semantics, or are all concurrent accesses likely to require modification of the data?* - All concurrent access requires modification of data. Since the worker thread has to remove or mark the pending "command" as done.

*Is heavy contention expected on the lock, or is it likely that only one thread or processor will ever need the lock?* - Depends. Most likely no for our purpose. Our hypervisor does not hypervise the system. It simply provides nice mechanics to extend NT functionality.

*Does the lock require fairness?* - See answer above. (No)

*Are there CPU cache, NUMA latency, or false sharing considerations that will impact performance?* - No.

*Is the lock global, or is it highly instantiated/replicated throughout memory?* - Global. More than you could ever wish for.


Since we cannot do wait-based locking on VMEXITs and other high IRQL situations, we must go with the standard spin-based locks. So `KeAcquireSpinLock` and `KeReleaseSpinLock` will do very fine for our purpose.
Note that, this is for hardcore C devs and Rust already provides `spin` crate that does the hard work for us.

## Anything else?
Using `KeDelayExecutionThread` to await before every iteration of while loop will be a good idea.

An example implementation will be on [hxposed](https://github.com/staarblitz/hxposed) soon!


Yours truly.
