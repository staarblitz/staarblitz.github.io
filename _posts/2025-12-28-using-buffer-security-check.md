---
title: Using Buffer Security Checks to your advantage.
description: How I patched ACPI.sys's GsDriverEntry to redirect control flow to HxPosed driver.
date: 2025-12-28 13:07:00 +0800
categories: [Internals, Windows, Boot]
tags: [windows-internals, rust, x86, asm, uefi, acpi]
---

## HxPosed needs to be loaded.
So the problem was simple. HxPosed, being an unsigned driver, has to be loaded. It's easy. Just fire up WinDbg and a VM. Yeah? Yeah. But not for average guy who wants to enjoy HxPosed.

So I made HxLoader. It maps and runs HxPosed by patching Windows's boot. And how I did that?

### UEFI Bootkits for Windows. A summary.
*WARNING: In no way do I, the author and coder, condone any tendency to harm a computer, person or entity. This blog and all my work are for educational purposes only. Individuals who misuse this blog and my work for malicious purposes are solely responsible for any harm they cause.*

It's a *de facto* standard how bootkits for Windows work nowadays.
1. Load Bootmgfw to memory.
2. Hook `ImgArchStartBootApplication`.
3. *On `ImgArchStartBootApplication`,* Get base address and size of Winload.
4. Hook `OslFwpKernelSetupPhase1` and `BlImgAllocateImageBuffer`.
5. *On `BlImgAllocateImageBuffer`,* allocate a buffer for the driver to be mapped.
6. *On `OslFwpKernelSetupPhase1`,* Map the driver and patch a driver that is in `_LOADER_PARAMETER_BLOCK`, redirect the execution flow.

Then, what is so interesting in my case, you might ask. That is because I have done it in rust and in a way abusing Gs for its exact opposite purpose.

### /GS flag. Buffer Security Check.
Microsoft describes the purpose of this flag very well on MSDN. [Here take a look](https://learn.microsoft.com/en-us/cpp/build/reference/gs-buffer-security-check?view=msvc-170).

In short, this feature creates a new entry point for the application. (GsXxxxx). Where it places a stack security cookie. Most functions have a piece of code inserted on them that checks for this cookie. If the cookie is gone, or overwritten the application assumes its execution has been hijacked and terminates itself.

That is whatt all those `__security_init_cookie` things you see are for.

It looks like this on IDA.

![GsDriverEntry on IDA Pro](/assets/img/uploads/img6.png)

And a function that performs stack security check.

![Function performing stack security check](/assets/img/uploads/img8.png)

But in my case, its used as a "buffer" between HxPosed and the real ACPI.sys entry point.

My first thought was patching this function like I did with any other using the `Detour` structure in HxLoader. But when the OS exits UEFI boot services, it will be dangerous trying to access the global variables of HxLoader from HxPosed. So that was a no.

I did something else instead, "What if I use `GsDriverEntry` to call the real driver entry?" and that worked. But with some caveats which I had to deal with.

### GsDriverEntry stack is NOT aligned.
My first prototype was this:
```asm
mov rax, <addr>
call rax
ret
```

But then I saw a weird behavior. Suddenly, `to_unicode_string` was giving me access violations. `KeSetUserAffinityThread` and so on do too. Not even bugchecks. Just straight out access violations that are catched by WinDbg. I was like "okay, maybe its because they are not meant to be used at boot-time". But the problem was way more deep than that.

I analyzed the problem. My first assumption is that anything that touched SIMD registers crashed. Which they were. All of those functions I aforementioned crashed while working on `XMM` registers. "How is that even possible" I thought to myself. I kinda blurry remembered something in `CR` registers that enabled SIMD instructions. But that didn't seem like the cause.

I continued to analyze the problem. Then, looked at the assembly code which caused the crash:

![movaps and movups](/assets/img/uploads/img7.png)

The line just before `movaps` (Move Aligned Packed Single Precision Floating-Point Values), that uses `movups` (Move Unaligned Packed Single Precision Floating-Point Values) worked. Immediately thought of alignment. I took the stack register, and looked it on calculator. Last 4 bits were not 0.

Gotcha.

So I had to manually align the stack. But this is a dangerous operation. My first idea was like this:
```asm
sub rsp, 0x100
and rsp, 0xFFFFFFFF0
call aligned_driver_entry
; uh?
```

That uh explains everything. After changing the `RSP`, how do you get it back? You don't. We don't have an idea what could the `RSP` be now. And the registers we want to put it into are volatile or non-volatile. Both cases we are not allowed to use. So this wouldn't work.

I looked at it. Subtracting just 8 from `RSP` gave us a perfectly aligned value. So we can do that. The `GsDriverEntry` itself does it with 0x20. Why not 8?

Then I realized I can use the `GsDriverEntry` to do that. Which I did.

```asm
mov rax, <addr>
sub rsp, 8
call rax
add rsp, 8
ret
```

### ACPI is useful, you know.

I was about to pat myself on the back and call it a day. I just pressed "g" on WinDbg and prayed it would work. It did. Windows was booting!

It was booting...
Come on... boot...
Crash.

We forgot something very important. We weren't calling the real driver entry!

So did a quick patch....
```asm
mov rax, <addr>
sub rsp, 8
call rax
nop
nop
nop
nop
call 0x160 ; call from GsDriverEntry -> DriverEntry
add rsp, 8
ret
```

And it crasheed... again? Do you see the problem here?

We are trashing `RCX` and `RDX` when we call our own driver entry. And its foolish to think they are preserved, (they are not).

SO at the last, this was what I did.

So did a quick patch....
```asm
mov rax, <addr>
sub rsp, 8
push rcx
push rdx
call rax
pop rdx
pop rcx
call 0x160 ; call from GsDriverEntry -> DriverEntry
add rsp, 8
ret
```

Thanks `GsDriverEntry` for its beautiful buffer area to place our hook into. Without that, we would had to save those bytes somewhere, restore them later and all of that bullshit.

#### Benefits
- No restoration
- No prologue/epilogue for the detour

So 2 birds at one.

Thanks for reading!