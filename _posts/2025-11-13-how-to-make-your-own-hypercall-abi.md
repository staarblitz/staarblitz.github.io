---
title: How to make your own HyperCall ABI
description: How did I make my own HyperCall ABI in hxposed.
date: 2025-11-13 13:07:00 +0800
categories: [Virtualization, Windows]
tags: [virtualization, kernel, x86, asm]
---

# So you've got into vmcalls and stuff huh?
Providing services to non-root software is cool. But there is something very important.
**How** you are going to do it?

`vmcall` is a single instruction. No operands. No nothing. Just like a call, but without specifying a procedure.
So how do we pass parameters? How do we make sure we don't get exploited? And how to allow flexibility?

For all of those purposes, we need to define what a single call contains.

## I'm calling you, somehow.
Take a look at what I did in HxPosed's `HypervisorCall` structure:
```rust
#[bitfield(u32)]
pub struct HypervisorCall {
    #[bits(16)]
    pub func: ServiceFunction,
    pub is_fast: bool,
    pub ignore_result: bool,
    pub buffer_by_user: bool,
    pub yield_execution: bool,
    pub is_async: bool,

    #[bits(11)]
    pub async_cookie: u16,
}
```

The fields are very self explanatory. But let's go over what I aimed.
`func` - Most self explanatory one. The unique identifier of the service that guest software asks hypervisor to do.
`is_fast` - A fastcall. Currently reserved.
`ignore_result` - The hypervisor returns nothing.
`buffer_by_user` - Hypervisor expects the buffer to be pre-allocated by the caller. If not, the function fails.
`yield_execution` - Hypervisor yields execution to next thread after its done executing the service function.
`is_async` - Hypervisor puts the request into a queue to execute later.
`async_cookie` - The cookie of the call, which will be used to notify caller it has been done.

While constructing this ABI, I decided to NOT use stack. Since the memory is slow, and a vmexit makes it even slower.
We need to pick a register to store the hypercall result and the call details. I picked RSI for that purpose. And picked R8, R9, and R10 for 3 args.

But what if I we neeed to pass more than 3 arguments?

## I cannot call you.
We know Nt and Zw functions that take enormous amount of arguments. But that is not good. It takes up our precious (kernel) stack space. Have you ever wondered why you always pass a pointer to `OBJECT_ATTRIBUTES` instead of passing `OBJECT_ATTRIBUTES` directly?

To save kernel stack space and make the call faster.

In Microsoft x64 calling convention, any argument after 4th is passed on to stack.
```c
example_call(1,2,3,4, "I'm on stack!", "Me too!");
```

I decided that I don't want nonsense in my hypervisor. So I did the easiest one. Always a fixed number of registers, for every function.
To pass more than 3 arguments, we can simply construct a struct and pass its pointer through registers.
We can also use the famous cbSize WinAPI uses to specify the version of structure that is being passed, if it ever gets changed, to support compatibility with older callers.

## Did you get it?
We also need to make sure that hypervisor catches our call. For that, I picked the register RCX for its special behavior on CPUID instruction.
When CPUID is executed with wrong leaf/sub-leaf, it resets the RCX to 0. That will be to our use.

If hypervisor succesfully catches the trap, it will set RCX to 0x2009, which we will know that hypervisor got our trap.

Here is a simple snippet that shows catching this trap in hxposed:
```rust
...
fn handle_vmcall<T: Guest>(guest: &mut T, info: &InstructionInfo) {
    let call = HypervisorCall::from_bits(guest.regs().rsi as _);
    unsafe { SHARED_HOST_DATA.get_unchecked().vmcall_handler.unwrap()(guest, call) }
    guest.regs().rip = info.next_rip;
    guest.regs().rcx = 0x2009; // This leaf indicates that CPUID was handled by the hypervisor.
}

fn handle_cpuid<T: Guest>(guest: &mut T, info: &InstructionInfo) {
    let leaf = guest.regs().rax as u32;
    let sub_leaf = guest.regs().rcx as u32;
    log::trace!("CPUID {leaf:#x?} {sub_leaf:#x?}");

    if sub_leaf == 0x2009 {
        // Our CPUID trap
        handle_vmcall(guest, info);
        return;
    }
...
```
