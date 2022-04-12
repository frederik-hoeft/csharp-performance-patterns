# csharp-performance-patterns

examples for less-known high performance pattern I come across in C#

## Branchless Lazy Getter initialization using delegate pointers

using the normal Lazy Getter initialization syntax produces null check branching to be generated in JIT asm:

```csharp
class LazyInitNormal
{
    private object o;

    public object GetObjectLazy() => o ??= new object();
}
```

JIT asm:

```asm
LazyInitNormal..ctor()
    L0000: ret

LazyInitNormal.GetObjectLazy()
    L0000: push rdi
    L0001: push rsi
    L0002: sub rsp, 0x28
    L0006: mov rsi, rcx
    L0009: mov rax, [rsi+8]
    L000d: test rax, rax         ; <---- branching right here
    L0010: jne short L0033
    L0012: mov rcx, 0x7ffa6c965678
    L001c: call 0x00007ffacc4fb060
    L0021: mov rdi, rax
    L0024: lea rcx, [rsi+8]
    L0028: mov rdx, rdi
    L002b: call 0x00007ffacc4faa00
    L0030: mov rax, rdi
    L0033: add rsp, 0x28
    L0037: pop rsi
    L0038: pop rdi
    L0039: ret
```

in certain scenarios where inlining is not possible and performance is critical this could be mitigated using delegate pointers:

```csharp
class LazyInitBranchless
{
    private object o;

    public delegate*<LazyInitBranchless, object> GetObjectLazy = &InitAndReturn;

    private static object InitAndReturn(LazyInitBranchless target)
    {
        target.GetObjectLazy = &GetObjectOnly;
        return target.o = new object();
    }

    private static object GetObjectOnly(LazyInitBranchless target)
    {
        return target.o;
    }
}
```

The following JIT asm code is generated

```asm
LazyInitBranchless..ctor()
    L0000: mov rax, 0x7ffa78000040
    L000a: mov [rcx+0x10], rax
    L000e: ret

LazyInitBranchless.InitAndReturn(LazyInitBranchless)
    L0000: push rdi
    L0001: push rsi
    L0002: sub rsp, 0x28
    L0006: mov rsi, rcx
    L0009: mov rcx, 0x7ffa78000048
    L0013: mov [rsi+0x10], rcx
    L0017: mov rcx, 0x7ffa6c965678
    L0021: call 0x00007ffacc4fb060
    L0026: mov rdi, rax
    L0029: lea rcx, [rsi+8]
    L002d: mov rdx, rdi
    L0030: call 0x00007ffacc4faa00
    L0035: mov rax, rdi
    L0038: add rsp, 0x28
    L003c: pop rsi
    L003d: pop rdi
    L003e: ret

LazyInitBranchless.GetObjectOnly(LazyInitBranchless)
    L0000: mov rax, [rcx+8]
    L0004: ret
```

no branching :)
