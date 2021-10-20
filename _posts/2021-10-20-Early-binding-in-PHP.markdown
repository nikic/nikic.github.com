---
layout: post
title: Early binding in PHP
excerpt: Early binding allows using a class before its declaration in the same file. However, the precise behavior is rather arcane.
---
PHP allows using a class before its declaration in the same file -- sometimes. Internally we call this "early binding". However, the precise behavior is rather arcane and not well documented. After reading this blog post, you'll probably appreciate why bug reports related to early binding go right on the "won't fix" pile.

## Early binding != class hoisting

The most basic example of early binding is the following:

```php?start_inline=1
$test = new Test;
class Test {
    /* ... */
}
```

This code works fine, even though the class `Test` is used before its declaration. Now, one could assume that this works because PHP performs class hoisting, i.e. implicitly converts the code to:


```php?start_inline=1
class Test {
    /* ... */
}
$test = new Test;
```

However, the reality is more complicated than this. Consider the following example:

```php?start_inline=1
# a.php
class Test {}
require 'b.php';

# b.php
if (class_exists(Test::class)) {
    return;
}
class Test {}
```

If the class declaration in `b.php` were hoisted to the top of the file, this would result in a class redeclaration error. To still support this kind of early-exit code, we need to make early binding behave as follows:

> If the class does not exist at the start of the script execution, declare it then. Otherwise, declare it at its original point of declaration.

However, this isn't all. Consider the following example:

```php?start_inline=1
# a.php
require 'b.php';
class Test2 extends Test {}

# b.php
class Test {}
```

If the class declaration in `a.php` were hoisted to the top of the file, it would occur before the `require` which pulls in a necessary dependency. As such, we need to adjust our rule:

> If the class does not exist at the start of the script execution and all dependencies of the class are available, declare it then. Otherwise, declare it at its original point of declaration.

## Limitations

The "all dependencies are available" requirement is tricky, because variance rules make it hard to determine this a priori. For example, is `A` a required dependency in the following code?

```php?start_inline=1
class Test2 extends Test {
    public function method(): A {}
}
```

The answer to that question depends on how `Test` looks like. If `Test` is declared as follows, then `A` is not a dependency, because we can establish a subtyping relationship without loading the class. `A` is always a subtype of `A`, regardless of its actual identity.

```php?start_inline=1
class Test {
    public function method(): A {}
}
```

However, if the class declaration instead looks as follows, then both `A` and `B` would be required dependencies, as we need the class declarations of `A` and `B` to determine if these classes are in a subtyping relationship:

```php?start_inline=1
class Test {
    public function method(): B {}
}
```

This is relatively simple to check when working on a class that has at most a parent class. It's easy to determine which method would override which parent method and perform a variance check. If the variance check comes back unresolved, we can't early bind and should fall back to late binding.

The situation quickly deteriorates once interfaces and especially traits get involved. Trait use adaptations like aliases make it hard to determine what overrides what and which subtyping relationships need to be proven for inheritance to succeed.

At that point, the only really robust way to determine whether inheritance will succeed is to actually do it. That is, create a copy of the class, perform inheritance on it and discard the result if it fails. While we use this approach for preloading since PHP 8.1, I'm rather leery of using it for early binding. Inheritance in PHP is not designed to fail gracefully. Maybe it would be worthwhile to change it in that direction (with the added benefit that inheritance errors could throw an exception rather than a fatal error), but that's not what we have right now.

The end result is that early binding is not performed if the class implements interfaces or uses traits:

> If the class does not exist at the start of the script execution, the class does not implement interfaces or use traits, and all dependencies of the class are available, declare it then. Otherwise, declare it at its original point of declaration.

It is worth noting that "implements interfaces" also includes implicitly implemented interfaces. Classes implementing `__toString()` are not eligible for early binding, because they implement `Stringable`. Similarly, enums are not early bound, because they implement `UnitEnum` or `BackedEnum`.

## Delayed early binding

The situation becomes more complicated once opcache gets involved. Normally, early binding is attempted directly during compilation. If it succeeds, the class is registered, otherwise a `DECLARE_CLASS` opcode is emitted.

This wouldn't work for opcache, because cached scripts must be isolated and cannot use dependencies from other scripts. A dependency might not be available during initial compilation (should not early bind) but be available during a later use of the cached script (should early bind).

Opcache solves this through "delayed early binding". `DECLARE_CLASS_DELAYED` opcodes are emitted for classes that are potentially eligible for early binding and placed in a linked list. When a script is loaded, opcache walks the `DECLARE_CLASS_DELAYED` opcodes and tries to perform early binding. If it succeeds, it marks the class as already declared in the run-time cache, so that another declaration is not attempted when the class declaration opcode is executed through normal control flow.

Of course, it turns out that this approach is still too simplistic. Consider the following variant:

```php?start_inline=1
if (true) {
    return;
}
class Test2 extends Test {}
```

In this case, the early return is always taken, and the optimizer will detect this and remove any following code -- including the `DECLARE_CLASS_DELAYED` opcode. However, per our early binding semantics, the class should still be delared if `Test` is available at the time the script is executed.

This particular problem is only fixed in PHP 8.2, where classes that require delayed early binding are now tracked in a separate structure that is independent of the `DECLARE_CLASS_DELAYED` opcodes that may be optimized away.

Despite all this, the behavior of early binding with and without opcache is still not quite the same. Consider the following example:

```php?start_inline=1
new Test2;
class Test2 extends Test {}
class Test {}
```

Without opcache this will error: Class `Test2` is not early bound, because the dependency `Test` is not available at the time of declaration. However, `Test` is early bound. With opcache `Test` is early bound and `Test2` queued for delayed early binding. At the time early binding is actually attempted `Test` does exist and the code will finish without error. Effectively this means that without opcache, there is a single early binding pass, while with opcache there are two.

## Conclusion

There are two sensible ways in which class declarations could have worked:

1. The class could be declared exactly where it was written in the code. This is nice and simple, but may be something of an inconvenience for code not following the "single class per file" style.
2. All (top-level) class declarations could be hoisted to the start of the script. This would also be easy to support, including reordering of class declarations to account for dependencies.

Of course, PHP picked the third way: "Why don't we have both?" Now we have a complicated system that only works some of the time.
