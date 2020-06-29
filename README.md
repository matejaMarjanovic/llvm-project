
# [PROPOSAL] [Salvage Debug Info] Support IR instructions with two SSA values

### Description

Intrinsics like llvm.dbg.value and llvm.dbg.declare have three
arguments: variable info (file and line in which it was declared,
variable name, etc.), register who holds the value which combined
with DIExpression gives us the value of the variable that is being
described and the DIExpression which holds DWARF operaions that are
used to calculate the value of the variable.

Without optimizations one llvm.dbg.value intrinsic call would describe
one variable (DIExpression would be empty).
With optimizations some variables are being optimized out and they
wouldn't exist during the execution of the program. If we run
the program in a debugger and would like to read the values of the
variables, in some cases that wouldn't be possible.

When we salvage an instruction which has a register and a constant as
it's operands, the register in the llvm.dbg.value intrinsic call will
become the register that is the operand in the instruction, and the
appropriate DWARF operations will be added in DIExpression.

### The problem

The problem arises when we try to salvage an instruction with two virtual
registers as it's operands, beacuse there is no DWARF operation that
represents a register, like there is that represents a constant. And not
only that, even if there was a DWARF operation like that, we wouldn't know
which register should we represent as the register from the instruction is
still virtual and we don't know in which physical register will it be mapped.


### The solution

Add another operand in llvm.dbg.value intrinsic call. That operand will be a
register that holds the value of the second register in case of a binary
instruction that has only registers as it's operands.
It will have undef value by default.

Now that we have an extra register in llvm.dbg.value intrinsic call we should
examine the behaviour if we try to salvage the instruction of the first register.
If the instruction has a register and a constant as it's register it will behave
as before, replacing the current register (the first) with a new register, and
adding DWARF operaions to the DIExpression, without changing the second register
in llvm.dbg.value.
If the instruction has two registers, it will cause undefined behaviour.

But what if we salvage the instruction of the second register in llvm.dbg.value?
Will we add DWARF operaions in the DIExpression like in the previous case?
How will the compiler know on which register to apply which DWARF operations
from the DIExpression?

The solution to that problem is adding another DIExpression, which is initially
empty (DIExpression()).
When we salvage the second register, the appropriate DWARF operaions will be pushed
to the second DIExpression, so we could calculate the values separetely.

Besides the DWARF operations that we would push to the DIExpression, in case
of an instruction with two registers as it's operands, we would also push another
new LLVM internal DWARF operation DW_OP_reg_* (plus, minus, mul, div, etc.) which
will in the ASM printer be translated to a standard DWARF operation consisting of
the value of the first DIExrpression and the value of the second DIExpression.

### Example

```
int main (int argc, char** argv) {
    char c1 = argc + 1;
    char c2 = argc - 1;
    char c3 = c1 + c2;
    char c4 = c1 + c2;
    char c5 = c4 * 4;

    if (argc % 2)
        printf("Value = %d\n", c3);
    else
        printf("Value = %d\n", c5);

    return 0;
}
```

The following command line will generate an .ll file with llvm.dbg.value intrinsic
which has three arguments and because of that the debugger won't be able to read
the values of the variables when they are still live:

`$ ./build-clean/bin/clang -g -O2 test.c -emit-llvm -S -o test-clean.ll`

```
define dso_local i32 @main(i32 %argc, i8** nocapture readnone %argv) local_unnamed_addr #0 !dbg !7 {
entry:
  call void @llvm.dbg.value(metadata i32 %argc, metadata !15, metadata !DIExpression()), !dbg !22
  call void @llvm.dbg.value(metadata i8** %argv, metadata !16, metadata !DIExpression()), !dbg !22
  call void @llvm.dbg.value(metadata i8 undef, metadata !17, metadata !DIExpression(DW_OP_plus_uconst, 1, DW_OP_stack_value)), !dbg !22
  call void @llvm.dbg.value(metadata i8 undef, metadata !18, metadata !DIExpression(DW_OP_constu, 1, DW_OP_minus, DW_OP_stack_value)), !dbg !22
  call void @llvm.dbg.value(metadata i8 undef, metadata !19, metadata !DIExpression()), !dbg !22
  call void @llvm.dbg.value(metadata i8 undef, metadata !20, metadata !DIExpression()), !dbg !22
  call void @llvm.dbg.value(metadata i8 undef, metadata !21, metadata !DIExpression()), !dbg !22
  %0 = and i32 %argc, 1, !dbg !23
  %tobool = icmp eq i32 %0, 0, !dbg !23
  %.sink = select i1 %tobool, i32 27, i32 25, !dbg !25
  %sext22 = shl i32 %argc, %.sink, !dbg !22
  %conv13 = ashr exact i32 %sext22, 24, !dbg !22
  %call14 = tail call i32 (i8*, ...) @printf(i8* nonnull dereferenceable(1) getelementptr inbounds ([12 x i8], [12 x i8]* @.str, i64 0, i64 0), i32 %conv13), !dbg !26
  ret i32 0, !dbg !27
}
```

If we compile it and then run gdb on the generated executable file
we will get the following:

`$ ./build-clean/bin/clang -g -O2 test.c -o test-clean`
```
$ gdb --args test-clean 3 4 5 6
...
(gdb) b 9
Breakpoint 1 at 0x400513: file test.c, line 9.
(gdb) r
Starting program: /home/rtrk/llvm/test/test-clean 3 4 5 6

Breakpoint 1, main (argc=5, argv=<optimized out>) at test.c:10
10	    if (argc % 2)
(gdb) p c1
$1 = <optimized out>
(gdb) p c2
$2 = <optimized out>
(gdb) p c3
$3 = <optimized out>
(gdb) p c4
$4 = <optimized out>
(gdb) p c5
$5 = <optimized out>
(gdb) p argc
$6 = 5
```

In the case of llvm.dbg.value with five arguments the outcome will be
different:
`$ ./build-dev/bin/clang -g -O2 test.c -emit-llvm -S -o test-dev.ll`

```
define dso_local i32 @main(i32 %argc, i8** nocapture readnone %argv) local_unnamed_addr #0 !dbg !7 {
entry:
  call void @llvm.dbg.value(metadata i32 %argc, metadata !15, metadata !DIExpression(), metadata i32 undef, metadata !DIExpression()), !dbg !22
  call void @llvm.dbg.value(metadata i8** %argv, metadata !16, metadata !DIExpression(), metadata i8** undef, metadata !DIExpression()), !dbg !22
  call void @llvm.dbg.value(metadata i32 %argc, metadata !17, metadata !DIExpression(DW_OP_LLVM_convert, 8, DW_ATE_signed, DW_OP_LLVM_convert, 32, DW_ATE_signed, DW_OP_plus_uconst, 1, DW_OP_stack_value), metadata i8 undef, metadata !DIExpression()), !dbg !22
  call void @llvm.dbg.value(metadata i32 %argc, metadata !18, metadata !DIExpression(DW_OP_LLVM_convert, 8, DW_ATE_signed, DW_OP_LLVM_convert, 32, DW_ATE_signed, DW_OP_constu, 1, DW_OP_minus, DW_OP_stack_value), metadata i8 undef, metadata !DIExpression()), !dbg !22
  call void @llvm.dbg.value(metadata i32 %argc, metadata !19, metadata !DIExpression(DW_OP_constu, 24, DW_OP_shl, DW_OP_plus_uconst, 16777216, DW_OP_constu, 24, DW_OP_shr, DW_OP_LLVM_reg_plus, DW_OP_LLVM_convert, 8, DW_ATE_signed, DW_OP_LLVM_convert, 32, DW_ATE_signed, DW_OP_stack_value), metadata i32 %argc, metadata !DIExpression(DW_OP_constu, 24, DW_OP_shl, DW_OP_constu, 16777216, DW_OP_minus, DW_OP_constu, 24, DW_OP_shr)), !dbg !22
  call void @llvm.dbg.value(metadata i32 %argc, metadata !20, metadata !DIExpression(DW_OP_constu, 24, DW_OP_shl, DW_OP_plus_uconst, 16777216, DW_OP_constu, 24, DW_OP_shr, DW_OP_LLVM_reg_plus, DW_OP_LLVM_convert, 8, DW_ATE_signed, DW_OP_LLVM_convert, 32, DW_ATE_signed, DW_OP_stack_value), metadata i32 %argc, metadata !DIExpression(DW_OP_constu, 24, DW_OP_shl, DW_OP_constu, 16777216, DW_OP_minus, DW_OP_constu, 24, DW_OP_shr)), !dbg !22
  call void @llvm.dbg.value(metadata i32 %argc, metadata !21, metadata !DIExpression(DW_OP_constu, 25, DW_OP_shl, DW_OP_constu, 22, DW_OP_shra, DW_OP_LLVM_convert, 8, DW_ATE_signed, DW_OP_LLVM_convert, 32, DW_ATE_signed, DW_OP_stack_value), metadata i8 undef, metadata !DIExpression()), !dbg !22
  %0 = and i32 %argc, 1, !dbg !23
  %tobool = icmp eq i32 %0, 0, !dbg !23
  %.sink = select i1 %tobool, i32 27, i32 25, !dbg !25
  %sext22 = shl i32 %argc, %.sink, !dbg !22
  %conv13 = ashr exact i32 %sext22, 24, !dbg !22
  %call14 = tail call i32 (i8*, ...) @printf(i8* nonnull dereferenceable(1) getelementptr inbounds ([12 x i8], [12 x i8]* @.str, i64 0, i64 0), i32 %conv13), !dbg !26
  ret i32 0, !dbg !27
}
```

`$ ./build-dev/bin/clang -g -O2 test.c -o test-dev`
```
$ gdb --args test-dev 3 4 5 6
...
(gdb) b 9
Breakpoint 1 at 0x400513: file test.c, line 9.
(gdb) r
Starting program: /home/rtrk/llvm/test/test-dev 3 4 5 6

Breakpoint 1, main (argc=5, argv=<optimized out>) at test.c:10
10	    if (argc % 2)
(gdb) p c1
$1 = 6 '\006'
(gdb) p c2
$2 = 4 '\004'
(gdb) p c3
$3 = 10 '\n'
(gdb) p c4
$4 = 10 '\n'
(gdb) p c5
$5 = 40 '('
(gdb) p argc
$6 = 5
```

![test-dev test-clean compare](https://i.imgur.com/5xsdUVY.png)


### Concern

When compiling ./llvm/lib/Target/AMDGPU/MCTargetDesc/R600MCCodeEmitter.cpp the memory
gets filled due to too many generated llvm.dbg.value intrinsic calls.
The fix limits the number of DbgUsers and DbgValues to 50.
