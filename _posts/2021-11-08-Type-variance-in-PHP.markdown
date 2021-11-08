---
layout: post
title: Type variance in PHP
excerpt: Type variance allows types to change during inheritance in a way that is compatible with the Liskov substitution principle. This article describes how this works on a technical level.
---
Inheritance in PHP enforces the [Liskov substitution principle](https://en.wikipedia.org/wiki/Liskov_substitution_principle) (LSP) to make sure that an instance of the parent class can be replaced with an instance of the child class. Of course, PHP cannot enforce this in full generality -- it can only detect definitely incompatible cases by inspecting class member declarations. For example, these two classes are clearly incompatible:

```php?start_inline=1
class A {
    public function method(string $arg) {}
}
class B extends A {
    public function method(int $arg) {}
}
```

An instance of `A` cannot be replaced by `B`, because `A::method()` will accept `"foobar"`, while `B::method()` will not.

Prior to PHP 7.4, the LSP verification was rather simplistic and rejected many LSP-compatible overrides. PHP 7.4 [introduced](https://wiki.php.net/rfc/covariant-returns-and-contravariant-parameters) covariant return types and contravariant argument types, which are most concisely illustrated with nullable types:

```php?start_inline=1
class A {
    public function method(T $arg): ?T {}
}
class B extends A {
    public function method(?T $arg): T {}
}
```

For return types, LSP requires that the child return type is the same or "more specific". If a caller could handle the `?T` return type of `A::method()`, it will also be able to handle the `T` return type of `B::method()`. It has to deal with strictly less possible values. Conversely, the child argument type must be the same or "more general". `B::method()` accepts anything that `A::method()` does, and more.

## Subtyping

Formally, we define type variance in terms of subtyping relationships:

* Covariance (return types): The child type must be a subtype of the parent type.
* Contravariance (argument types): The parent type must be a subtype of the child type.
* Invariance (property types): The child type must be a subtype of the parent type and the parent type must be a subtype of the child type. This implies that the types are equal (in a certain sense).

I will use the notation `A <= B` to denote that `A` is a subtype of `B` in the following. So, what are PHP's subtyping rules? Let's build them up one by one:

> 1\. A <= A

Subtyping is reflexive: A type is always a subtype of itself.

> 2\. A <= B if exists X s.t. A <= X and X <= B

Subtyping is transitive.

> 3\. A <= B if A extends B or A implements B

Together with the previous rule, this means that A is a subtype of B if it extends from B or implements B anywhere higher up in the hierarchy. It does not necessary have to extend/implement it directly.

> 3\. A <= object forall A where A is a class/interface type \
> 4\. static <= self

Any class type is a subtype of `object`, and `static` is a subtype of `self`.

> 5\. A <= mixed forall A \
> 6\. never <= A forall A

The `mixed` type is the top type, and the `never` type is the bottom type.

> 7\. iterable == array\|Traversable

`iterable` is effectively an alias for `array|Traversable`.

> 8\. U<sub>1</sub>\|...\|U<sub>n</sub> <= V if forall U<sub>i</sub> we have U<sub>i</sub> <= V \
> 9\. U<sub>1</sub>&...&U<sub>n</sub> <= V if exists U<sub>i</sub> s.t. U<sub>i</sub> <= V \
> 10\. U <= V<sub>1</sub>\|...\|V<sub>n</sub> if exists V<sub>i</sub> s.t. U <= V<sub>i</sub> \
> 11\. U <= V<sub>1</sub>&...&V<sub>n</sub> if forall V<sub>i</sub> we have U <= V<sub>i</sub>

Now to the interesting cases involving union and intersection types. Some examples:

`A|B` is a subtype of `object`, because `A` is a subtype of `object` and `B` is a subtype of `object`.

`A&B` is a subtype of `A`, because `A` is a subtype of `A`.

`static` is a subtype of `self|null`, because `static` is a subtype of `self`.

If class `X` implements interfaces `A` and `B`, then `X` is a subtype of `A&B`, because `X` is a subtype of `A` and `X` is a subtype of B.

From this, we can derive rules for how two union/intersection types interact (these are the rules we implement in practice):

> 8'\. U<sub>1</sub>\|...\|U<sub>n</sub> <= V<sub>1</sub>\|...\|V<sub>n</sub> if forall U<sub>i</sub> exists V<sub>j</sub> s.t. U<sub>i</sub> <= V<sub>j</sub> \
> 9'\. U<sub>1</sub>&...&U<sub>n</sub> <= V<sub>1</sub>&...&V<sub>n</sub> if forall V<sub>j</sub> exists U<sub>i</sub> s.t. U<sub>i</sub> <= V<sub>j</sub> \
> 10'\. U<sub>1</sub>\|...\|U<sub>n</sub> <= V<sub>1</sub>&...&V<sub>n</sub> if forall U<sub>i</sub> forall V<sub>j</sub> we have U<sub>i</sub> <= V<sub>j</sub> \
> 11'\. U<sub>1</sub>&...&U<sub>n</sub> <= V<sub>1</sub>\|...\|V<sub>n</sub> if exists U<sub>i</sub> exists V<sub>j</sub> s.t. U<sub>i</sub> <= V<sub>j</sub>

The reason why it's worth explicitly considering these is that forall and exists quantifiers do not commute. That is, you could also construct the following two variants with forall/exists swapped:

> 8''\. U<sub>1</sub>\|...\|U<sub>n</sub> <= V<sub>1</sub>\|...\|V<sub>n</sub> if exists V<sub>j</sub> forall U<sub>i</sub> s.t. U<sub>i</sub> <= V<sub>j</sub> \
> 9''\. U<sub>1</sub>&...&U<sub>n</sub> <= V<sub>1</sub>&...&V<sub>n</sub> if exists U<sub>i</sub> forall V<sub>j</sub> s.t. U<sub>i</sub> <= V<sub>j</sub>

However, we can discard these two rules because they are strictly weaker than 8' and 9'. For example, according to 8' `static|null` is a subtype of `self|null`, because `static` is a subtype of `self` and `null` is a subtype of `null`.

When using 8'' instead, the precondition is not satisfied, because `static|null` is a neither a subtype of `self`, nor of `null`. Because the precondition is not satisfied, the rule makes no statement on whether it is a subtype or not.

The reason we use rules 8'-11' is that they are actually equivalences, i.e. we could replace "if" with "if and only if". That means that we don't need to check multiple variants and determine whether any one of them succeeds.

Okay, with the subtyping relationship out of the way, surely we have solved the hard part of the problem? Unfortunately, this is just where the real fun begins...

## Cyclic dependencies

The problem that makes variance checks hard are cyclic dependencies. Consider the following example:

```php?start_inline=1
class A {
    public function method(): A {}
}
class B extends A {
    public function method(): C {}
}
class C extends B {
}
```

These classes satisfy LSP, because C is a subtype of A. However, at the time class B is declared, PHP doesn't know anything about class C yet, so it doesn't know that this is actually the case. This code will thus result in an inheritance error:

> Fatal error: Could not check compatibility between B::method(): C and A::method(): A, because class C is not available

There is also no way in which we could reorder the declarations to make the code compile: A cyclic graph cannot be sorted topologically.

However, PHP does support these kinds of cyclic references if an autoloader is used, as is customary in modern code. The following code runs without error:

```php?start_inline=1
spl_autoload_register(function(string $class) {
    if ($class === 'A') {
        class A {
            public function method(): A {}
        }
    } else if ($class === 'B') {
        class B extends A {
            public function method(): C {}
        }
    } else if ($class === 'C') {
        class C extends B {
        }
    }
});
$b = new B;
```

Going through an autoloader allows us to load additional dependencies during inheritance, though the particulars of how this works are somewhat involved.

## Early class registration

Prior to PHP 7.4, we would first run inheritance on a class, and then insert it into the class table, making the symbol visible. With the introduction of type variance, we need to register the symbol for the "unlinked" class first, and then run inheritance ("linking").

The preceding example should make it clear why this is the case: We start off running inheritance on class `B`, but then need to load `C` to determine a subtyping relationship. But `C` in turn depends on `B`, which should be the `B` we're currently trying to link, not a separate copy (otherwise we'd just recurse infinitely).

At the same time, we can't allow actually using the class in most circumstances, because it hasn't finished inheritance yet. Apart from a few whitelisted usages related to inheritance, we will pretend that the class does not exist.

This gives us Schrödinger's class. On the one hand, `new B` in this code will claim that class `B` doesn't exist:

```php?start_inline=1
spl_autoload_register(function(string $class) {
    if ($class === 'A') {
        new B; // Class "B" not found
    } else if ($class === 'B') {
        class B extends A {
        }
    }
});
$b = new B;
```

On the other hand, `class B {}` in this code will claim that `B` is already declared:

```php?start_inline=1
spl_autoload_register(function(string $class) {
    if ($class === 'A') {
        class B {} // Cannot declare class B, because the name is already in use
    } else if ($class === 'B') {
        class B extends A {
        }
    }
});
$b = new B;
```

Both statements are somewhat true: The class *is* declared, but only visible for certain symbol references.

There are two more subtleties here. The first is that we actually distinguish three states: unlinked, nearly linked and linked. A nearly linked class has completed inheritance apart from potentially outstanding variance checks. Unlinked classes can only be used during subtyping checks, while nearly linked classes can be used as parents or interfaces.

We have to allow the use of unlinked classes for subtyping, because we can have cyclic dependencies there. However, the actual class hierarchy itself can never be cyclic. You can't construct something like "A extends B extends A". As such, allowing only "nearly linked" classes is sufficient here (and prevents recursive inheritance).

The second subtlety relates to error handling. Normally, inheritance errors always result in a fatal error, because the inheritance process is implemented in a destructive fashion (the unlinked class template is modified in-place). However, due to certain requirements of the Symfony framework, we treat exceptions thrown while autoloading the parent class or an implemented interface specially. We make sure to load parent/interface dependencies before doing any destructive changes, which allows us to recover from a failure by simply unregistering the class again.

Unfortunately, unlinked uses introduce a complication: We allow unlinked classes to participate in subtyping checks, on the premise that in the end either inheritance on all involved classes will succeed (and as such, the subtyping results were correct) or else inheritance will fail with a fatal error. If we were to unregister a class that participated in a subtyping check, this logic would no longer hold. Consider the following contrived example:

```php?start_inline=1
spl_autoload_register(function(string $class) {
    class X {
        public function test(): I {}
    }
    class Y extends X { 
        public function test(): B {}
    }
    throw new Exception("Class $class not found");
});

interface I {}
class B extends A implements I {}
```

An exception is thrown while loading the parent class `A`. If we were to simply unregister `B` and let it bubble up, the programmer could catch the exception and then declare `B` with a different hierarchy:

```php?start_inline=1
try {
    class B extends A implements I {}
} catch (Exception) {
    class B {}
}
```

However, we have already made use of the fact that `B` is a subtype of `I` when linking the classes `X` and `Y`. If we allowed this kind of code, the earlier subtyping assumption would no longer hold.

For this reason we have to track whether a class has had any "unlinked uses", in which case we must convert the autoloader exception into a fatal error:

> Fatal error: During inheritance of B with variance dependencies: Uncaught Exception: Class A not found

This is a compromise to make Symfony's optional dependency hacks work for the common case, while preserving correctness in the general case.

## Delayed variance obligations

Subtyping checks can have three possible results: Success, failure and unresolved. The subtyping check is performed without autoloading, but allowing the use of unlinked classes. If it's not possible to produce a definitive result, then all unknown classes mentioned in the two types are registered as "delayed autoloads" and the unresolved state is returned.

It is important that we try to determine the subtyping relationship without loading additional classes first. The by far most common case is that types during inheritance are unchanged, and we don't need class loading to determine that `A` is a subtype of `A`. We also don't need class loading to know that `A` is a subtype of `?A`, and so on.

Triggering class loading for trivial subtyping relationships like these would greatly increase the number of loaded classes: Types mentioned in methods that are not otherwise used in the current request would have to be loaded. Class loading is expensive, and we want to minimize it as much as possible.

Now, if we have received an unresolved variance result, then we will register a delayed variance obligation for later processing. There are three kinds of obligations:

  * Method compatibilty obligation: Requires that a certain child method is compatible with a certain parent method.
  * Property compatibility obligation: Requires that a certain child property is compatible with a certain parent property.
  * Dependency obligation: Requires that a parent class or implemented interface will be linked. While we can use a "nearly linked" class during inheritance, all dependencies must be fully linked for the class to be fully linked.

If the class has any open variance obligations after inheritance is otherwise finished ("nearly linked"), we will load all delayed autoloads and then process the obligations again. Dependency obligations are handled by checking obligations on the dependency first. If any obligations are still not satisfied, then inheritance fails with a fatal error, either because of an actual variance violation, or because a dependency is still missing despite the autoloading attempt.

An important aspect of the variance implementation is that it ensures that classes are usable immediately after the class declaration. Consider the following variant of our motivating example:

```php?start_inline=1
function printAvailable() {
    if (class_exists(A::class, false)) echo "A";
    if (class_exists(B::class, false)) echo "B";
    if (class_exists(C::class, false)) echo "C";
    echo "\n";
}

spl_autoload_register(function(string $class) {
    if ($class === 'A') {
        class A {
            public function method(): A {}
        }
        echo "After A: "; printAvailable();
    } else if ($class === 'B') {
        class B extends A {
            public function method(): C {}
        }
        echo "After B: "; printAvailable();
    } else if ($class === 'C') {
        class C extends B {
        }
        echo "After C: "; printAvailable();
    }
});
$b = new B;

// Prints:
// After A: A
// After C: ABC
// After B: ABC
```

Here we check which classes are available after each class declaration. The general user expectation is that a class is usable immediately after declaring it. While style guides generally discourage mixing declarations with code, supporting this is important for some use cases, such as calling `class_alias()` directly after a declaration, or calling a static initializer on the class.

The order of the prints gives us a hint on how class linking plays out in this case:

1. B is loaded and registered.
2. A is loaded and linked (it is a leaf class), and `new A` is printed.
3. C is added to the delayed autoloads and a method compatibility obligation between `B::method()` and `A::method()` is registered.
4. After B is nearly linked, C is loaded via delayed autoload. It will use B while it is nearly linked, adding a dependency obligation.
5. After C is nearly linked, there are no delayed autoloads and variance obligations on C are processed. There is a single dependency obligation on B.
6. Thus variance obligations on B are processed first. The single method compatibility obligation is satisfied now that C is available. B becomes fully linked.
7. As the dependency obligation has been satisfied, C becomes fully linked as well, and `new C` is printed.
8. At this point we return to linking of B, but it is already linked now, so we're done and print `new B`.

The main peculiar bit here is that class B becomes fully linked (and thus fully available to code) during the declaration of class C. What happens if we tweak the previous code to make `C` the loaded root class instead?

```php?start_inline=1
$c = new C;

// Prints:
// After A: A
// After B: AB
// After C: ABC
```

This case is simpler, because the classes are loaded in a nice chain, so no delayed variance obligations are involved here. We do make use of the fact that C is a subtype of A while C is still unlinked though.

The interesting bit about this example is that class B becomes fully linked (and available) while C remains unlinked. Doesn't this break variance, as linking of class C might still fail later? This is fine, based on the following reasoning:

Before class C is fully linked, a call to `B::method()` cannot return successfully, because the `C` return type is not available yet. As such, it's not possible to observe a broken variance promise on a call to this method. Lateron, either C will become fully linked and all is good, or C will fail inheritance and throw a fatal error. Even code executing after the fatal error (e.g. in a shutdown function) will not be able to register a new C with a different subtyping relationship, because the old stub class remains registered.

## Conclusion

The addition of type variance support is one of the prime examples where a feature has been accepted without anything even approaching a complete understanding of its technical implications. The feature is conceptually simple, but the devil is very much in the details.

Now, I do believe that this is a necessary feature, despite the complexity. However, none of this complexity was described in the original RFC, and it took significant effort to nail down the semantics after the fact.
