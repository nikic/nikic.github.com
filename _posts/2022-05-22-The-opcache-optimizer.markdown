---
layout: post
title: The opcache optimizer
excerpt: An introduction to the inner workings of the opcache optimizer, or at least some parts of it.
---
In a [previous article]({% link _posts/2021-10-13-How-opcache-works.markdown %}), I described how the caching related functionality in opcache works. However, opcache also exposes an optimizer, which is the subject of this article.

Previously, the optimizer was part of opcache proper. Since PHP 8.1, it is part of the core engine. However, the engine only exposes an internal API, which can currently only be enabled through the opcache extension. In the future, the optimizer may be exposed independently of opcache. Some extensions like pcov also make use of the internal API.

## Options

Assuming opcache is loaded and enabled (don't forget `opcache.enable_cli=1` if relevant), the optimizer is enabled using the `opcache.optimization_level` ini setting, which defaults to `0x7FFEBFFF`. This setting is a bit mask of enabled optimization passes defined in [zend_optimizer.h](https://github.com/php/php-src/blob/cc506a81e17c3e059d44b560213ed914f8199ed5/Zend/Optimizer/zend_optimizer.h). As of PHP 8.1, the meaning of the bits is as follows:

 * 0x0001 ([pass1.c][pass1]): Simple peephole optimizations.
 * 0x0002: Unused.
 * 0x0004 ([pass3.c][pass3]): Simple jump optimization.
 * 0x0008 ([optimize_func_calls.c][pass4]): Call optimization.
 * 0x0010 ([block_pass.c](https://github.com/php/php-src/blob/cc506a81e17c3e059d44b560213ed914f8199ed5/Zend/Optimizer/block_pass.c)): CFG-based optimization.
 * 0x0020 ([dfa_pass.c](https://github.com/php/php-src/blob/cc506a81e17c3e059d44b560213ed914f8199ed5/Zend/Optimizer/dfa_pass.c)): SSA-based optimization.
 * 0x0040: Whether call graph should be used for SSA-based optimizations.
 * 0x0080 ([sccp.c](https://github.com/php/php-src/blob/cc506a81e17c3e059d44b560213ed914f8199ed5/Zend/Optimizer/sccp.c)): Sparse conditional constant propagation.
 * 0x0100 ([optimize_temp_vars_5.c][optimize_temp_vars]): Temporary variable optimization.
 * 0x0200 ([nop_removal.c][nop_removal]): Removal of NOP opcodes.
 * 0x0400 ([compact_literals.c][compact_literals]): Literal optimization.
 * 0x0800: Call stack adjustment.
 * 0x1000 ([compact_vars.c][compact_vars]): Unused variable removal.
 * 0x2000 ([dce.c](https://github.com/php/php-src/blob/cc506a81e17c3e059d44b560213ed914f8199ed5/Zend/Optimizer/dce.c)): Dead code elimination.
 * 0x4000: Collect and substitute constant declarations (**unsafe**).
 * 0x8000: Trivial function inlining (part of call optimization).
 * 0x10000: Ignore possibility of operator overloading (**unsafe**).

The default enables everything apart from the unsafe settings. The unsafe settings can either cause program behavior to change (constant substitution) or PHP to crash outright (ignore operator overloading).

The other useful setting is `opcache.opt_debug_level`, which is a debugging option that produces an opcode dump after individual passes. It accepts the bit flags from `opcache.optimization_level` if they correspond to an optimization pass. Additionally, it's possible to produce a dump at a few more points, with the most useful being:

 * 0x10000: Dump before optimizer.
 * 0x20000: Dump after optimizer.
 * 0x40000: Dump before CFG optimizations.
 * 0x80000: Dump after CFG optimizations.
 * 0x200000: Dump before SSA optimizations.
 * 0x400000: Dump after SSA optimizations.

When debugging, I will usually just set `opcache.opt_debug_level=-1` to get a full trace and not bother picking out specific passes.

## Pass two

Before we talk about individual optimization passes, we first need to talk about "pass two". The name is a bit confusing in that this refers to a post-processing step in the compiler, not an optimization pass. It brings opcodes into the final form expected by the virtual machine.

Among other things, pass two performs the following changes:

  * The opcode handler pointer is assigned, based on the opcode, operand kinds and other specialization information.
  * Variable references are converted from variable numbers into offsets from the start of the stack frame.
  * Instruction references are converted from instruction numbers into offsets relative to the current opline.
  * The opline array and the literal array are combined into a single allocation, and literal references are converted into opline-relative offsets as well.
  * Liveness ranges for temporary variables are calculated.

To make changes easier, the optimizer partially reverts pass two: In particular, the opline array and literal array are split again, and literals are referenced using indices into the literals table. This means that literal references use indices, while variable and instruction references use frame-relative or opline-relative offsets.

Additionally, some invariants are not upheld during optimization and instead recomputed later. This includes the opcode handler, the liveness ranges, as well as smart branch flags (which determine whether a comparison can be fused with a following jump).

## Liveness range calculation

While liveness ranges are recomputed after optimization, we can unfortunately not ignore them entirely. Liveness ranges are computed using a single-pass CFG-less algorithm, which requires certain invariants to be upheld during the elimination of unreachable code.

As a reminder, the purpose of live ranges is to determine which temporary variables need to be freed when an exception is thrown. Consider the following example (with a before optimizer dump):

```php?start_inline=1
foreach ($array as $value) {
    throw $ex;
}

0000 V3 = FE_RESET_R CV0($array) 0004
0001 FE_FETCH_R V3 CV1($value) 0004
0002 THROW CV2($ex)
0003 JMP 0001
0004 FE_FREE V3
0005 RETURN int(1)
LIVE RANGES:
     3: 0001 - 0004 (loop)
```

The live range intervals are half open, so what this tells us is that an exception thrown at instructions `0001`, `0002` or `0003` needs to free the foreach loop variable `V3`.

Live ranges are calculated using a single backwards scan which remembers the last use of all variables and then emits a live range when it encounters the corresponding definition. Things are a bit more complicated than that, for example due to instructions that don't consume their operands, but that's the basic idea. In particular, we need an actual use of the variable to emit a live range. Consider the following code:

```php?start_inline=1
calc() + throw $ex;

0000 INIT_FCALL_BY_NAME 0 string("calc")
0001 V2 = DO_FCALL_BY_NAME
0002 THROW CV0($ex)
0003 T1 = ADD V2 bool(true)
0004 FREE T1
0005 RETURN int(1)
LIVE RANGES:
     2: 0002 - 0003 (tmp/var)
```

The shown opcodes are after optimization. Note that instructions `0003`, `0004` and `0005` are unreachable due to the throw at `0002` and would usually be removed by CFG optimization. However, this would remove the only use of `V2` in instruction `0003`, which means that no live range would be emitted and the variable would leak.

To prevent this, expression use of `throw` (or `exit`) gets a special flag that tells the optimizer to pretend that execution continues after the instruction, for the sole purpose of preserving live ranges.

The other and somewhat more annoying case we need to deal with are loop variable frees (where switch and match are also "loops"):

```php?start_inline=1
switch (calc()) {
case 0:
    throw $ex1;
default:
    throw $ex2;
}
return 1;

0000 INIT_FCALL_BY_NAME 0 string("calc")
0001 V2 = DO_FCALL_BY_NAME
0002 T3 = CASE V2 int(0)
0003 JMPZ T3 0005
0004 THROW CV0($ex1)
0005 THROW CV1($ex2)
0006 FREE V2
LIVE RANGES:
     2: 0002 - 0006 (tmp/var)
```

Again, the `return` at the end is unreachable, because the switch will definitely throw in both branches. And indeed, you can see that the `RETURN` instruction has been optimized away. However, the `FREE` of the loop variable `V2` has been preserved for the purpose of live range construction, even though it is equally unreachable. CFG optimization knows the block is unreachable, but treats it as a special "unreachable free block".

It may be worthwhile to make live range construction CFG-based, at least if the optimizer is used, to remove the need for these kinds of hacks.

## Pass 1: Peephole optimizations

With that said, we can take a look at the [first pass][pass1], which does simple peephole optimizations:

The first optimization is evaluating instructions like `ADD` with constants operands. An important caveat is that operations that would throw an exception or warning are not evaluated. If the result of the opcode can be determined, then users of the result are replaced with a constant operand and the original opcode is converted into a `NOP`.

PHP's representation of instructions does not permit to efficiently remove (or add) instructions, as they are stored in a flat array, and contain offset references besides. For this reason, optimizations will replace instructions with NOPs instead, which are then eliminated by dedicated NOP removal passes that can move instructions and update references to them.

Replacing uses of a temporary with constants is less straightforward than it may sound. Not all instructions support constant operands, and some instructions impose additional requirements. For example, while an object property read `FETCH_OBJ_R` would accept an arbitrary variable property name, a constant property name must be a string, and requires allocation of run-time cache slots. These rules are encoded in the [`zend_optimizer_update_opN_const()`](https://github.com/php/php-src/blob/cc506a81e17c3e059d44b560213ed914f8199ed5/Zend/Optimizer/zend_optimizer.c#L277) functions.

The [replacement](https://github.com/php/php-src/blob/cc506a81e17c3e059d44b560213ed914f8199ed5/Zend/Optimizer/zend_optimizer.c#L620) happens by scanning forward in the instruction stream, finding a use of the variable and updating the operand to a constant. This assumes that there is a only a single use of the temporary. This is usually the case, as temporary variables are understood to be consumed by the using instruction. For instructions that keep their operands alive, we continue scanning to replace later uses of the same temporary.

If replacement fails, for example because the temporary is used in an instruction that does not support constant operands, the instruction is instead replaced with a `QM_ASSIGN`. The "QM" stands for question mark, and dates back to its original use in the implementation of the ternary operator. Nowadays, this is used as a generic assignment to a temporary.

A second implicit assumption involved here is that the opcode defining the temporary dominates all uses. For example, if we had the following structure:

```php?start_inline=1
if (...) {
    T1 = ...
} else {
    T1 = ADD 1, 2
}
use(T1)
```

Then replacing T1 with the result of `ADD 1, 2` would be incorrect, as the value is only produced in one branch. This early in the optimization pipeline, this invariant does hold, because the compiler would actually generate something along these lines instead:

```php?start_inline=1
if (...) {
    T1 = ...
    T2 = QM_ASSIGN T1
} else {
    T3 = ADD 1, 2
    T2 = QM_ASSIGN T3
}
use(T1)
```

In output produced by the compiler, only a few special instructions like `QM_ASSIGN` may assign the same temporary from multiple control flow branches. However, this invariant is no longer upheld starting with the block pass.

Next to simple operations like binary operators and casts, calls to certain functions like `function_exists()` are evaluated. Of course, opcache cannot statically determine that a function will not exist at runtime, but it can ascertain the existence of extension-defined functions. A caveat is that on Windows the cached script may be reused in a process with different loaded extensions and thus different available functions. Similar optimizations can be performed for `extension_loaded()` and system `ini_get()`.

Conditional jump instructions with constant operands are converted into either an unconditional jump or a NOP.

Finally, this pass can collect constants defined using `const` or `define()` and replace later uses. Constant definitions are only collected before the first control flow.

The constants definition collection is guarded by a default-disabled optimization option, because this optimization is not always legal. Consider the following code:

```php?start_inline=1
// file1.php
const X = 1;
require __DIR__ . '/file2.php';

// file2.php
const X = 2;
var_dump(X);
```

The redeclaration of `X` in file2.php will throw a warning and retain the previously declared value 1. However, the optimization will think that the actual value is 2, resulting in different program behavior.

Most of the pass 1 functionality is repeated in later passes. For example, both the block pass and the SCCP pass perform constant folding, with the latter solving the problem in full generality. As such, I'm not sure how practically important the functionality in pass 1 really is. It's possible that there are some phase-ordering considerations involved here.

## Pass 3: Jump optimizations

Next up is [pass 3][pass3], which deals with jump optimizations. If you're wondering about the odd numbering, there used to be a pass 2, whose functionality was later merged into pass 1.

This is a good point to review the jump instructions supported by the PHP virtual machine. The simplest are the unconditional jump (`JMP`) and the conditional jump on zero (`JMPZ`) and non-zero (`JMPNZ`), where "zero" should be read as "false after a bool cast".

The `JMPZ`/`JMPNZ` conditional jumps will fall through to the following instruction if the condition is not met. There also used to be a `JMPZNZ` instruction that specified two target labels, but I removed it in the process of writing this blog post.

The `JMPZ_EX` and `JMPNZ_EX` variants additionally return a temporary with the boolean value of the condition. These instructions are used in the implementation of the short-circuiting `&&` and `||` operators.

```php?start_inline=1
$x = f1() && f2();

0000 INIT_FCALL_BY_NAME 0 string("f1")
0001 V1 = DO_FCALL_BY_NAME
0002 T2 = JMPZ_EX V1 0006
0003 INIT_FCALL_BY_NAME 0 string("f2")
0004 V3 = DO_FCALL_BY_NAME
0005 T2 = BOOL V3
0006 ASSIGN CV0($x) T2
```

If `f1()` is falsy, then the `JMPZ_EX` will populate `T2` with false and jump directly to the `ASSIGN`. If `f1()` is truthy, then we fall through to the call to `f2()`, and assign the result of a bool cast to `T2`. The implementation for `||` is basically the same with `JMPNZ_EX`.

The `JMP_SET` is similar to `JMPNZ_EX`, but returns the original value of the condition, without a bool cast. It is used for the implementation of the shorthand ternary operator:

```php?start_inline=1
$x = f1() ?: f2();

0000 INIT_FCALL_BY_NAME 0 string("f1")
0001 V1 = DO_FCALL_BY_NAME
0002 T2 = JMP_SET V1 0006
0003 INIT_FCALL_BY_NAME 0 string("f2")
0004 V3 = DO_FCALL_BY_NAME
0005 T2 = QM_ASSIGN V3
0006 ASSIGN CV0($x) T2
```

If `f1()` is truthy, `T2` is set to its value and we jump directly to the `ASSIGN`. Otherwise `f2()` is evaluated, and the result is moved to the right temporary using `QM_ASSIGN`.

Another close relative is `COALESCE`, which returns the condition value verbatim just like `JMP_SET`, but performs the jump on non-null, rather than on truthy. Unsurprisingly, it is used to implement the null-coalesce operator `??`.

While there are more control-flow instructions than that (e.g. for jumptables and for exceptional control flow), these are the ones that pass 3 deals with.

Returning to the jump optimization pass, the transforms it performs fall into a few broad categories, though there are quite a few combinations of different jumps. The first one is that a jump to the next instruction can be converted into a NOP. For conditional jumps on a temporary, we replace it with a `FREE` of the temporary instead.

A peculiar case is a conditional jump on a compiled variable (CV). Compiled variables are not consumed by using instructions, so it is not necessary to explicitly `FREE` them. However, entirely dropping the instruction may still be incorrect, because it might optimize away an "undefined variable" warning. The virtual machine has a special `CHECK_VAR` instruction for this purpose, which only performs an undefined variable check. It is another instruction that is not emitted by the compiler and only created during optimization:

```php?start_inline=1
if ($a) {}
echo "x";

# Original
0000 JMPZ CV0($a) 0001
0001 ECHO string("x")

# After jump optimization
0000 CHECK_VAR CV0($a)
0001 ECHO string("x")
```

The second category of optimizations are sequences of jumps. For example, if a `JMPZ` target is another `JMP`, then we can go directly to the `JMP` target. If both jumps involved are conditional, then such optimizations are only possible if the conditions are correlated:

```php?start_inline=1
if ($x && $y) {
    echo "Test";
}

# Original
0000 T2 = JMPZ_EX CV0($x) 0002
0001 T2 = BOOL CV1($y)
0002 JMPZ T2 0004
0003 ECHO string("Test")

# After jump optimization
0000 T2 = JMPZ_EX CV0($x) 0004
0001 T2 = BOOL CV1($y)
0002 JMPZ T2 0004
0003 ECHO string("Test")
```

The original target of the `JMPZ_EX` is the `JMPZ`, which uses the `T2` result value of the `JMPZ_EX`. We know that the `JMPZ` will be taken if the `JMPZ_EX` is taken, so it's fine to replace the `JMPZ_EX` target with the `JMPZ` target. You may note that the result is still quite sub-optimal, and will be optimized to only two `JMPZ` instructions by the block pass.

Something that jump optimization needs to be careful about are infinite loops. Consider the following example:

```php?start_inline=1
while ($x) {}

# Original
0000 JMP 0001
0001 JMPNZ CV0($x) 0001

# After jump optimization
0000 NOP
0001 JMPNZ CV0($x) 0001
```

When trying to optimize the `JMPNZ`, jump optimization would naively try to follow the correlated jump target `0001`, which happens to be the same instruction. The cycle is very straightforward in this case, but might span across multiple instructions more generally.

Jump optimization avoids infinite optimization loops by remembering a "hit list" of targets it has already followed while trying to optimize the current instruction. It stops following the target chain once a cycle has been encountered.

It's somewhat tedious to enumerate all optimizations this pass performs, so you'll have to read the code for an exhaustive list. Similar to pass 1, I'm not sure how valuable pass 3 still is, as it only performs a subset of optimizations the block pass is capable of.

## Pass 4: Call optimization

[Pass 4][pass4] is responsible for call optimization. Calls in PHP generally use a sequence of `INIT_*` to determine the called function and allocate a call frame, a number of `SEND_*` instructions to push arguments to the call stack, and finally a `DO_*` opcode to perform the actual function call. These sequences may be nested for nested calls like `a(b())`.

The possible `INIT_*` opcodes are relatively straightforward:

* `INIT_FCALL`: Call to a statically known function. The necessary stack size is pre-calculated.
* `INIT_FCALL_BY_NAME`: Call to a function with resolved name (either outside a namespace or fully qualified / imported).
* `INIT_NS_FCALL_BY_NAME`: Call to an unqualified function in a namespace, will try `NS\name()` and `name()`.
* `INIT_METHOD_CALL`: Call to an instance method.
* `INIT_STATIC_METHOD_CALL`: Either a call to a static method or a scoped instance call.
* `INIT_DYNAMIC_CALL`: Call of a dynamic value like `$fn()`.
* `INIT_USER_CALL`: Statically known call of `call_user_func()` or `call_user_func_array()`.
* `NEW`: Creation of an object, will call `__construct()` if one exists.

The situation is significantly more complicated when it comes to `SEND_*` opcodes. There are many different opcodes depending on the kind of the argument and the value being passed to it:

* `SEND_VAL`: Send a constant or temporary to a known value argument.
* `SEND_VAR`: Send a variable to a known value argument.
* `SEND_REF`: Send a variable to a known reference argument.
* `SEND_VAR_NO_REF`: Send the result of a function call to a known reference argument, if the function returns by reference. Otherwise throw a notice and send a dummy reference.
* `SEND_VAL_EX`: For a value argument, perform `SEND_VAL`, otherwise error.
* `SEND_VAR_EX`: For a value argument, perform `SEND_VAR`, otherwise send the variable by reference.
* `SEND_VAR_NO_REF_EX`: For a reference argument, perform a `SEND_VAR_NO_REF`, otherwise send the call result by value.
* `SEND_FUNC_ARG`: Send a complex variable either by value or by reference. The send type is determined by an earlier `CHECK_FUNC_ARG` for the argument, and the variable is produced by a sequence of `FETCH_*_FUNC_ARG` opcodes that will either perform a read or a write fetch. For example, depending on the argument kind, `f($array[0])` either needs to read `$array[0]` (and diagnose a non-existent index) or write `$array[0]` (and create a non-existent index).
* `SEND_UNPACK`: Send zero or more arguments using `...$args` syntax.
* `SEND_ARRAY`: Send zero or more arguments using `call_user_func_array()`.
* `SEND_USER`: Send arguments using `call_user_func()`.

Finally, the `DO_*` opcodes also come in flavors that differ by their degree of specialization:

* `DO_FCALL`: This is the generic function call opcode that can be used for all functions.
* `DO_FCALL_BY_NAME`: This can be used for both internal and userland functions if neither the `execute_ex` nor `execute_internal` hooks are in use, and the functions do not use trampolines. Additionally, internal functions cannot be deprecated and cannot have `$this`.
* `DO_ICALL`: Same as `DO_FCALL_BY_NAME` but limited to internal functions.
* `DO_UCALL`: Same as `DO_FCALL_BY_NAME` but limited to userland functions.
* `CALLABLE_CONVERT`: This is used for first-class callable syntax `foo(...)` introduced in PHP 8.1 and is not a real call. It converts the call frame into a `Closure` object.

The primary purpose of the call optimization pass is to relax `INIT`/`SEND`/`DO` opcodes into cheaper forms. If the function being called can be determined, then we can perform the following relaxations:

* `INIT_FCALL_BY_NAME` / `INIT_NS_FCALL_BY_NAME` -> `INIT_FCALL`
* `SEND_*_EX` -> `SEND_*`
* `FETCH_*_FUNC_ARG` -> `FETCH_*_R` or `FETCH_*_W`
* `DO_FCALL` -> `DO_FCALL_BY_NAME` / `DO_ICALL` / `DO_UCALL`

We can see this in action with the following example:

```php?start_inline=1
test($a[$b]);

function test($byval) {
    return $byval;
}

# Original
0000 INIT_FCALL_BY_NAME 1 string("test")
0001 CHECK_FUNC_ARG 1
0002 V2 = FETCH_DIM_FUNC_ARG CV0($a) CV1($b)
0003 SEND_FUNC_ARG V2 1
0004 DO_FCALL_BY_NAME

# After call optimization
0000 INIT_FCALL 1 96 string("test")
0001 NOP
0002 V2 = FETCH_DIM_R CV0($a) CV1($b)
0003 SEND_VAR V2 1
0004 DO_UCALL
```

The function is declared after the call, so the compiler is not aware of it at the time the call is compiled. However, the optimizer does know the definition of `test()` and is able to relax the `*_BY_NAME` opcodes, as well as determine that the `*_FUNC_ARG` send is actually by value.

When determining the called function, we generally distinguish between knowing the exact function being called and only knowing a "prototype function". In the latter case, we might actually be calling an overriding method from a child class. Even if we only have a prototype function, our variance checks ensure that certain properties will hold for child methods as well. For example, if an argument of the prototype function is by-value, then it cannot be changed to a by-reference argument in a child class.

The other optimization performed by this pass in inlining, for a very generous interpretation of the term: In particular, only functions that return a constant value can be inlined, and nothing else.  This transform additionally needs to ensure that the call cannot trigger any warnings or exceptions, for example through type checks or default constant expression evaluation. I'd consider this optimization largely useless in practice.

## Pass 9: Temporary variable optimization

We're going to jump over a whole lot of passes here, namely the entire CFG and SSA based optimization pipeline. While these are arguably the most interesting passes, they are also the most complex and would blow the size of this article beyond all reasonable bounds.

Instead, let's continue with a number of late cleanup passes, starting with [temporary variable optimization][optimize_temp_vars]. The allocation of temporaries in the PHP compiler itself is naive: For each instruction result, a new temporary will be used:

```php?start_inline=1
return $a + $b + $c + $d + $e + $f;

# Compiler-produced opcodes
0000 T6 = ADD CV0($a) CV1($b)
0001 T7 = ADD T6 CV2($c)
0002 T8 = ADD T7 CV3($d)
0003 T9 = ADD T8 CV4($e)
0004 T10 = ADD T9 CV5($f)
0005 RETURN T10
```

Here we use five different temporaries T6-T10 even though most of these are not live at the same time. Temporary optimization reduces this to two temporary variables, which reduces the size of the stack frame and improves access locality:

```php?start_inline=1
# After temporary optimization
0000 T6 = ADD CV0($a) CV1($b)
0001 T7 = ADD T6 CV2($c)
0002 T6 = ADD T7 CV3($d)
0003 T7 = ADD T6 CV4($e)
0004 T6 = ADD T7 CV5($f)
```

At this point one might wonder whether we can reduce this example down to a single temporary instead:

```php?start_inline=1
# After temporary optimization
0000 T6 = ADD CV0($a) CV1($b)
0001 T6 = ADD T6 CV2($c)
0002 T6 = ADD T6 CV3($d)
0003 T6 = ADD T6 CV4($e)
0004 T6 = ADD T6 CV5($f)
```

This would indeed work in this specific case, but generally opcodes might write to the result operand before they finish reading all input operands, in which case reusing a temporary for inputs and outputs would not be legal. The optimizer does not bother to model this accurately, as this would usually save only one additional temporary per function.

The compaction algorithm is straightforward: We walk backwards through the opcodes. When we see a use of a temporary that has not been assigned a new temporary yet, we assign the first one that is free. When we see a definition of the temporary, we mark the new temporary as free again.

Of course, there are a few caveats here: Some temporaries are defined multiple times (in different control-flow paths, e.g. different branches of a ternary operator), and we only want to consider the temporary as dead when we hit the earliest possible definition point. This is done by collecting these definition points in a separate pass upfront.

The other caveat are instructions with special semantics. For example, `ROPE_*` instructions don't work on a single temporary, but rather `N` consecutive temporaries that can hold `2*N` strings. We need to make sure that a contiguous range of temporaries is allocated for these. Additional care also has to be taken for instructions that are part of a return sequence that may jump into a finally block. Temporary reuse is avoided for these, to avoid conflicts with temporaries used in the finally block.

## Pass 10: NOP removal

Because instructions and side-tables encode either absolute or relative references to other instructions, it is not straightforward to remove an instruction from the opcode stream. Instead, optimizations will replace dead instructions with a NOP, and rely on the [NOP removal pass][nop_removal] to eliminate these later.

NOP removal happens in two phases. We first iterate over the opcodes while skipping NOPs, and copy opcodes backwards to fill the resulting holes. While doing these copies, jumps are "[migrated](https://github.com/php/php-src/blob/cc506a81e17c3e059d44b560213ed914f8199ed5/Zend/Optimizer/zend_optimizer.c#L705)", which means that opline-relative instruction references are updated to account for the movement. For example, if an instruction moves backwards two instructions, opline-relative references need to be increased by two. It should be emphasized that this update only accounts for the movement of the jump instruction itself, but not for the movement of the target it is pointing to.

While doing this, we populate a shiftlist, which stores how many steps backwards each instruction has been moved. Then, in a second iteration, we "[shift](https://github.com/php/php-src/blob/cc506a81e17c3e059d44b560213ed914f8199ed5/Zend/Optimizer/zend_optimizer.c#L748)" jumps by looking up the jump targets in the shiftlist and computing their new position. The same is also done for the try/catch sidetable, which stores offsets of try, catch and finally regions.

NOP removals are also performed as part of other passes (e.g. the CFG and SSA based optimization passes) and work similarly, but usually need to update more auxiliary data structures using the shiftlist.

## Pass 11: Literal compaction

The [literal compaction pass][compact_literals] deduplicates literals and cache slots used by instructions. It operates in multiple phases:

First, instructions are scanned to find all referenced literals. This serves the dual purpose of detecting which literals are entirely unused and can be dropped, and determining how many "related" literals there are. Opcodes that involve symbol references (e.g. calls or constant fetches) will often use multiple consecutive literals. For example, the `INIT_NS_FCALL` opcode uses three literals that hold the lowercased name in the global namespace, the lowercased name in the current namespace, as well as the original spelling of the function name for use in error messages. These related literals need to be treated as an atomic group for the purposes of deduplication.

The second pass is a walk over the literals table, which performs the actual deduplication. Literals are copied in-place to their new position, with dead or duplicate literals being dropped. A map keeps track which old position maps to which new position. For null, false, true and empty arrays, there is only a single possible value, so deduplication just involves remembering the first occurrence of such a literal.

For strings, a hash table is used to store the new literal position for a given string. The main interesting bit is how related literals are handled: In this case the cache key is produced by concatenating all the related literals. However, doing just that could result in collisions between strings with different numbers of related literals. For example, a string `"Aa"` might collide with the pair of literals that an `a()` call using `INIT_FCALL_BY_NAME` would produce.

To avoid this, a clever hack is used: We add the number of related literals as a bias to the hash that is stored inside the string structure. Because hash table lookups will compare both the hash and the string contents, this ensures that cache keys for different numbers of related literals will never collide. Theoretically, there is still a risk of collisions between strings consisting of the same number of literals, for example `"abc"` could be derived from literal pairs `["a", "bc"]` or `["ab", "c"]`. This cannot happen in practice due to the limited ways in which related literals are used.

Integers are deduplicated using the same hash table. There is only one case where integers can have related literals, which is array accesses of the form `$foo["1"]`, where a pair of literals `[1, "1"]` is used. This is done because plain arrays will convert `"1"` to `1`, while overloaded objects using `ArrayAccess` must be passed the original `"1"` value. In this case `"1"` is used as a cache key, but once again with a hash bias that distinguishes it from both ordinary strings, and strings with related literals.

Finally, floating-point numbers also use string keys with a biased hash. In this case the string contains the raw binary value of the double. This is done because PHP's hash tables do not natively support double keys, so we are forced to represent them as strings instead.

The last phase walks over the instructions again and replaces the literal references. Additionally, this phase performs deduplication of runtime cache slots. This is not strictly related to literal compaction, but does rely on literals having already been deduplicated. There is also the historical background here that runtime cache slot offsets used to be stored in the u2 space of literals. Later, they were moved into either operands of the opcodes, or their extended value space. This avoids an indirection through the literal table when accessing the runtime cache.

This phase reallocates all runtime cache slots. For cases that cannot be deduplicated (or where we don't bother), this just involves using the current cache size, and then incrementing it by the number of cache slots used by the instruction. For cases that can be deduplicated, we keep a number of different tables to remember an already allocated cache slot. The tables are separated by symbol kind (e.g. function, class, property) and indexed by the number of the literal associated with the cache slot. This is the part that relies on literals already being deduplicated and allows us to use simple indexing, rather than dealing with biased hash tables.

The only case where we do use a hash table are static member cache slots, because these are derived from two literals, the class name and the member name (which could be a method, property or constant). In this case we use a `"Class::member"` style hash key, once again with a bias to distinguish different member kinds.

The end result of this pass is that we use less memory for both literals and cache slots, and thus also have better cache locality. Additionally, having a single cache slot for a given symbol reference means that it only needs to be populated once, rather than separately for each of the references.

## Pass 13: Variable compaction

The [variable compaction pass][compact_vars] removes variables that are completely unused. Unlike the temporary optimization pass, it also supports compiled variables (i.e. named variables like `$x`). We do not attempt to reuse CV variables with disjoint lifetimes for a number of reasons: It would make for a confusing debug experience (as variable names would be reused), is often not possible due to possible destructor effects, and would also be substantially more complicated from a technical perspective, because CV lifetimes don't have the convenient structure of temporary lifetimes.

Compaction works by first walking over all opcodes and collecting all referenced variables into a bitset. Then we compute the mapping from old variable numbers to new numbers (skipping the dead ones) and walk the instructions again to update all references. Finally, the mapping from variable numbers to names is updated to drop the dead variables as well.

## Finalization

As mentioned at the very start of this article, once all optimizations are done, we have redo parts of pass two. This involves combining the opcode array and literals table back into one allocation and making constant references opline-relative, recomputing the opcode handler, as well as smart branch flags and live ranges.

Once all op arrays have been optimized, one final optimization is applied in a separate phase: `INIT_FCALL` opcodes include a pre-calculated stack size that should be reserved for the call. This pass recalculates the stack size, as it might have been reduced by optimizations (e.g. due to temporary optimization). This runs as a separate phase, so that all callees have already been optimized.

And with that we're done! Of course, I did skip over the most interesting bits, which are the CFG and SSA based optimizations. Hopefully, we can take a look at these another time.


[pass1]: https://github.com/php/php-src/blob/cc506a81e17c3e059d44b560213ed914f8199ed5/Zend/Optimizer/pass1.c
[pass3]: https://github.com/php/php-src/blob/cc506a81e17c3e059d44b560213ed914f8199ed5/Zend/Optimizer/pass3.c
[pass4]: https://github.com/php/php-src/blob/cc506a81e17c3e059d44b560213ed914f8199ed5/Zend/Optimizer/optimize_func_calls.c
[optimize_temp_vars]: https://github.com/php/php-src/blob/cc506a81e17c3e059d44b560213ed914f8199ed5/Zend/Optimizer/optimize_temp_vars_5.c
[nop_removal]: https://github.com/php/php-src/blob/cc506a81e17c3e059d44b560213ed914f8199ed5/Zend/Optimizer/nop_removal.c
[compact_literals]: https://github.com/php/php-src/blob/cc506a81e17c3e059d44b560213ed914f8199ed5/Zend/Optimizer/compact_literals.c
[compact_vars]: https://github.com/php/php-src/blob/cc506a81e17c3e059d44b560213ed914f8199ed5/Zend/Optimizer/compact_vars.c
