# Simple data move with type conversion
```c
// 0.c
void foo(unsigned char *x, long *y) {
  *y = (long) *x;
}
```

### Excerpt of asm created via `gcc -S -nostdlib -nostartfiles -O3 0.c`
```s
# 0.s
foo:
	movzbl	(%rdi), %eax
	movq	%rax, (%rsi)
	ret
```

## Main Question
**Why `movzbl` and not `movzbq`?** **Shouldn't we clear the upper 32 bits of `%rax`?**

Section `3.4.1.1` of `Intel x86_64 Software Developer's Manual` answers the question:
> When in 64-bit mode, operand size determines the number of valid bits in the destination general-purpose
register:
> * 64-bit operands generate a 64-bit result in the destination general-purpose register.
> * 32-bit operands generate a 32-bit result, zero-extended to a 64-bit result in the destination general-purpose
register.
> * 8-bit and 16-bit operands generate an 8-bit or 16-bit result. The upper 56 bits or 48 bits (respectively) of the
destination general-purpose register are not modified by the operation. If the result of an 8-bit or 16-bit
operation is intended for 64-bit address calculation, explicitly sign-extend the register to the full 64-bits.

**For 32-bit operands, zero-extension of upper 32 bits is done implicitly by hardware.**
**Thus `movzbl (%rdi) %eax` achieves the same data movement as `movzbq (%rdi) %rax`.**

Using `objdump -d` to compare encoding of the 2 cases:

* When using `movzbq`
```
0000000000000000 <foo>:
   0:	f3 0f 1e fa          	endbr64 
   4:	48 0f b6 07          	movzbq (%rdi),%rax
   8:	48 89 06             	mov    %rax,(%rsi)
   b:	c3                   	ret
```

* When using `movzbl`
```
0000000000000000 <foo>:
   0:	f3 0f 1e fa          	endbr64 
   4:	0f b6 07             	movzbl (%rdi),%eax
   7:	48 89 06             	mov    %rax,(%rsi)
   a:	c3                   	ret
```

**Using `movzbl` saves 1 byte of encoding**

## Follow-up Questions

### Q: Does `movzbl` always take 1 less byte of encoding (vs. `movzbq`)?
* **A: Depends on the operands of `movzbl`.**
  `movzbl (%rdi) %eax` doesn't reference registers R8-R15, so REX prefix is not prepended to instruction encoding.
  In contrast, `movzbl (%rdi),%r8d` references register R8 and thus the REX prefix is needed (encoded to 4 bytes; 44 0f b6 07)

### Q: Why implicitly zero-extend upper 32 bits for 32-bit operands?

### Q: When would we need `movzbq`? Do we need it at all?
