# LTO.what()

When the target program has interprocedural optimizations enabled for their builds, it often translates to link time optimizations, and for MSVC mainstream, it's `/LTCG` and `/GL`, a.k.a. link time code generation and whole program optimizations. This technique feeds the compiler with extra information that allows it make interprocedural optimizations.  

How does it affect our hooking methods and common approaches?  

For MSVC x64 windows targets, we hook by complying with [x64 calling convention](https://learn.microsoft.com/en-us/cpp/build/x64-calling-convention). This is under the assumptions that caller handles the shadow stack and volatile register are safe to use with callee.  

The x64 MSVC ABI considers `rax`, `rcx`, `rdx`, `r8`, `r9`, `r10`, `r11` and `xmm0` to `xmm5` as volatile.  

## LTO.why()

Consider the following call example: 

```asm
mov rdx, [rcx]         // a2
mov rcx, r8            // use r8 value here as a1
call my_func(rcx, rdx) // call boundary
mov r8, 0x100          // using r8 as a free register now
```

After returning from `my_func`, compiler will consider all volatile registers value changed, thus compiler will not reuse `rcx` or `rdx` registers.

**However**, with LTO enabled targets, it may look like this:

```asm
mov rdx, [rcx]         // a2
mov rcx, r8            // use r8 value here as a1
call my_func(rcx, rdx) // call boundary
test r8, r8            // keep using r8
```

With the PGO information from linker, compiler knows that `my_func` did not change `r8`, or it can be optimized to not change `r8`, so compiler lets the caller use `r8` across the call boundary. This effectively reduces register preserving/stack usage, thus optimizations. 

When implementing the hook functions for our plugins, normally there's no way of knowing if all the hooks will change specific registers, and that information cannot be accounted for when dealing with LTO targets.

## Syntax

Currently, `dku::Hook` offers `write_call` variant, `write_call_ex`. This API is designated for preserving regular/sse registers across a hook call boundary and keeping the original LTO code executing.  

```cpp
auto write_call_ex(
    const dku_memory auto src,
    F                     dst,
    enumeration<Register> regs = { Register::NONE },
    enumeration<SIMD>     simd = { SIMD::NONE }
);
```

## Parameter

+ `src` : address of the target callsite
+ `dst` : hook function
+ `regs` : regular registers to preserve as non volatile
+ `simd` : sse registers to preserve as non volatile

You can set multiple registers by passing `{Register, Register, ...}`.

## Example

```cpp{17,20,23,26}
using namespace DKUtil::Alias;

// hook function
bool Hook_123456(void* a_gameInstance)
{
    return func(a_gameInstance);
}

// original function
static inline std::add_pointer_t<decltype(Hook_123456)> func;

// callsite
auto addr = 0x7FF712345678;

// preserve rdx, r9
func = dku::Hook::write_call_ex<5>(addr, Hook_123456, 
    { Reg::RDX, Reg::R9 });
// preserve xmm0, xmm2
func = dku::Hook::write_call_ex<5>(addr, Hook_123456, 
    { Reg::NONE }, { Xmm::XMM0, Xmm::XMM2 });
// preserve rdx, r9, and xmm0, xmm2
func = dku::Hook::write_call_ex<5>(addr, Hook_123456, 
    { Reg::RDX, Reg::R9 }, { Xmm::XMM0, Xmm::XMM2 });
// preserve all
func = dku::Hook::write_call_ex<5>(addr, Hook_123456, 
    { Reg::ALL }, { Xmm::ALL });
```

## Non-Volatile Patch

Sometimes you may not need to commit a `write_call_ex` hook, but could use some boilerplate assemblies for preserving registers and restoring them.

```cpp
auto [prolog, epilog] = dku::Hook::JIT::MakeNonVolatilePatch({Reg::ALL});
Patch userPatch;
// wrap user patch in non-volatile patch generated by DKUtil
userPatch = prolog.Append(userPatch).Append(epilog);
```