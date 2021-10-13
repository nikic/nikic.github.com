---
layout: post
title: How opcache works
excerpt: A brief overview of how the PHP opcache extension works, covering the caching-related aspects only.
---

The opcache PHP extension implements various functionality to speed up PHP in a transparent manner. As the name indicates, its origin and primary purpose is opcode caching, but nowadays it also contains an optimizer and just-in-time compiler. However, this blog post will focus only on the opcode caching aspect.

Opcache has three layers of caches: The original shared memory cache, the file cache introduced in PHP 7, and the preloading functionality added in PHP 7.4. We'll discuss all of these in turn.

While opcache is nominally an independent extension, its functionality is tightly dependent on engine implementation details, and modifications to the engine often require changes to opcache as well. As such, the way opcache works differs significantly between PHP versions. This article describes the state as of PHP 8.1 and highlights some of the changes in this version.

## Shared memory

The primary purposes of opcache is to cache compilation artifacts in shared memory, to avoid the need to recompile PHP scripts on every execution.

On Unix-like systems, a single fixed-size shared memory (SHM) segment is allocated on startup. To handle requests, PHP will then either fork additional processes or spawn additional threads. These processes/threads will see the SHM segment at the same address.

As Windows does not support forking, it is common to instead spawn entirely separate PHP processes, which do not have any shared address space. This is a big problem for opcache, because it requires the SHM segment to be mapped at the same address in each process. Otherwise, pointers into SHM would not be valid across processes.

To make this work, opcache stores the SHM base address, and tries to map the segment at the same address in other processes. If this fails, opcache falls back to using the file cache. However, even if it succeeds, there are limitations: While this guarantees the same address for the SHM segment, the addresses of internal functions/classes may differ between processes due to ASLR. This means that on Windows, it's not possible for cached artifacts to depend on internal functions/classes etc.

Windows is the only platform where two unrelated PHP processes can share the same opcache SHM. For example, it's possible for two concurrent CLI invocations to share the same cache, which is not possible on other operating systems. The `opcache.cache_id` setting exists to force a different cache in this case.

Because maintaining the separate behavior for Windows is something of a pain, opcache may drop support for reattachment from unrelated processes in the future, which means that on Windows, the use of a thread-based rather than process-based SAPI would be required.

## Locking and immutability

When shared memory is in play, it is always important to consider your access model. As we do not want to perform any fine-grained locking operations or atomic reference counting at runtime, opcache's memory model ends up being very simple: Shared memory is immutable.

Opcache essentially only has two locks: One is a write lock, which can be held only by one process allowed to modify SHM. While the write lock is held, other processes are still allowed to read SHM. As such, holding the write lock generally only allows you to allocate new memory in the SHM segment and write to it, but not to modify already allocated and potentially used shared memory (with some exceptions).

The `opcache.protect_memory` option can be used to mprotect the whole SHM segment whenever the write lock is not held, which is useful to detect violations of the immutability invariant (but should not be enabled in production for performance reasons).

The other lock is a read lock that is acquired when a request makes use of SHM for the first time. It does not track what is being used and whether it stops being used. The only purpose is to record that the cache is being used *somehow* in this request.

The purpose of this lock is to facilitate opcache restarts: Because we don't track which parts of the cache are being used in a fine-grained manner, it's not possible to ever remove anything from the opcode cache. When the cache runs full, a restart is scheduled instead.

If a restart is scheduled, then newly started requests will not use the SHM cache (but may fall back to the file cache). When the number of users drops to zero, the entire cache is cleared and we can start from scratch. If the number of users does not drop to zero within `opcache.force_restart_timeout`, then opcache will kill remaining users.

## Map pointers

Some of the structures stored in the SHM cache need (or at least want) to reference per-request data. For example, while a function definition is generally immutable, it may contain static variables, which will be different for each request. Similarly, functions use a run-time cache to cache request-specific symbol resolutions.

As we can't store per-request information in the immutable shared memory cache, we use a "map pointer" indirection instead. Rather than storing a pointer to the static variables, we instead store a reference to where the static variables are going to be stored.

In the current implementation, a map pointer takes one of two forms: Either it is a plain pointer to the actual storage of the pointer, which is the representation used when the structure is not cached in SHM. The indirection pointer is typically arena allocated.

Alternatively, the map pointer only stores an offset from a base address, where the base address is going to be different for each request. This is the representation used for immutable structures in shared memory. We track how large the used map pointer area needs to be and zero it on each request.

```
For mutable memory: map_ptr & 1 == 0

map pointer ----> indirection pointer -----> static variables
                  (arena allocated)


For immutable memory: map_ptr & 1 == 1

map base pointer: slot 0
                  slot 1
    + map offset: slot 2 -----> static variables
                  slot 3
```

While it's clear why we need the indirection in the second case (separate map pointer area for each request), one may wonder what the purpose of the indirection pointer in the first case is: As the memory is mutable, we could store the static variables pointer directly. This is indeed just a historical artifact, and the unnecessary indirection will likely be gone in PHP 8.2.

## Interned strings

At this point, let's take a brief aside to discuss interned strings. Strings in PHP are represented as a reference-counted structure that stores the string length, its contents and its hash. While strings may be shared, there may also be multiple strings with the same content, if they are created independently.

Interned strings are deduplicated: There will only be one interned string with a given content. This saves memory and can make comparison more efficient, because the pointer equality fast-path is more likely to trigger. Interned strings in PHP are also immutable by dint of not being reference-counted.

Without opcache, PHP separates interned strings into persistent and per-request. Persistent interned strings are created during startup, for example for the names of internal classes/functions. Per-request strings are created for symbols and literals in PHP scripts (if no persistent interned string for them exists yet) and discarded at the end of the request.

When opcache is enabled, interned strings are stored in SHM, so they are deduplicated across processes and can be referenced by structures cached in SHM. On startup, opcache will copy persistent interned strings into SHM on a best-effort basis (it may not know about all pointers that are stored somewhere), but this is not important for correctness.

Additionally, creation of interned strings during the request is disabled. Normal, non-interned strings are created instead. Only when the compiled script is cached (and the SHM write lock acquired) do strings get converted into SHM interned strings.

## Class entry cache

PHP scripts contain a lot of references to classes in string form, e.g. `new Foo` or a `Foo $param` type. As the actual identity of `Foo` might differ between requests, it's not possible to compile these down to a direct class reference.

Fetching a class entry from the class name is relatively expensive for how common it is: We need to lower case the string and look it up in the class hash table. For references like `new Foo` this lookup is cached in the function run time cache. However, it's not always possible to use the run time cache. For example, property type checks can't make use of the run time cache and prior to PHP 8.1 used to instead replace a string name with a class entry directly inside the type, which means that the type couldn't live in SHM.

PHP 8.1 introduced a class entry cache, which combines interned strings with map pointers. For interned strings used in certain positions (class declarations and type names) a map pointer slot is allocated, which stores the resolved class entry for this name. To avoid increasing the string size, this uses a trick:

Normally, interned strings always have a reference count of 2. However, the actual reference count doesn't matter, it only needs to be larger than 1 to ensure the string gets duplicated on modification. Strings with refcount 1 can be modified in place. As such, we can use the refcount field to store a map pointer offset to use as the class entry cache.

This does come with some limitations, because it is bound to the interned string mechanism. For example, if opcache is enabled but a script is not cached, then interned strings won't be used and consequently the class entry cache will not be available.

One of the nice things about the class entry cache is that it is fairly generic and not bound to specific language constructs (like the run-time cache). If you write `new ReflectionClass(Foo::class)`, the class lookup can be cached, even though it is happens dynamically.

## Persist

The actual persistence of scripts into shared memory is relatively straightforward. The script is first compiled as usual, apart from some options to make sure no cross-file dependencies are used during compilation. The compilation result is moved out of the global function/class tables into a self-contained persistent script structure.

Then the size of the required shared memory segment is calculated. This step must mirror the logic of the actual persist step exactly, but (mostly) doesn't modify the script. If the shared memory allocation fails, we can still bypass opcache and execute it as usual. The only modification the "persist calc" step does is to convert strings into SHM interned strings if possible, as interned strings are stored in a fixed size segment that is separate from the persisted script. Strings that are succesfully interned do not count towards the script size.

Finally, the persist step copies the script into shared memory and frees the original script. To do so it keeps track of an xlat table, which maps the original pointers to the new pointers in shared memory. This allows resolving repeated uses of the same pointer.

## Inheritance cache

Classes internally come in two forms. Unlinked classes represent a class declaration as you would write it in the code: It contains the methods declared in that class and references dependencies (parent class, interfaces, traits) as strings. Linked classes represent a class declaration that has successfully finished inheritance. It contains inherited methods/properties/etc and references dependencies as resolved class entries.

When looking at a single script, classes usually exist in unlinked form (unless they happen to have no dependencies). Linking classes requires looking at classes in other files. However, the used class declaration might differ from one request to the next.

Prior to PHP 8.1, this meant that only the unlinked class template was cached, and inheritance still has to be performed on each request. As inheritance is a fairly expensive process, this had a non-trivial performance impact. PHP 8.1 addresses this with the introduction of the inheritance cache.

The inheritance cache stores the linked inheritance result for a given set of dependencies. When inheritance is requested at run-time, the class name dependencies are resolved into class entries and if a cache entry for this set of dependencies already exists, it is used. While dependencies *can* differ between requests, in practice they will usually be the same, so inheritance only needs to be performed once.

If no cache entry exists, the unlinked class is copied from SHM into mutable per-process memory and the inheritance process is performed on it (in-place). The result is persisted into the inheritance cache using essentially the normal persistence process, together with the depedencies for which this cache entry is valid.

## Preloading

Preloading is a more radical solution to the inheritance problem: Anything loaded by the preload script will survive across requests. As such, it is safe to make use of cross-script dependencies in this case. The disadvantage is that the preload state cannot be changed without restarting PHP.

Some of the preloading benefit has likely been obsoleted by the inheritance cache in PHP 8.1, though preloading still has some advantages: Classes are available in fully inherited form at the start of the request. The only per-request cost of preloading is clearing the map pointer area. Normal opcache usage still requires going through autoloading, looking up persistent scripts, registering entries in global hash tables, looking up and checking dependencies for the inheritance cache, etc.

Preloading can operate in two modes: When classes are simply loaded using `require`, inheritance will happen as it usually does and preloading can support classes with arbitrarily complex inheritance scenarios (including variance cycles). This also makes it easy to ensure that any necessary dependencies are provided by an autoloader.

Alternatively, it is possible to preload files using `opcache_compile_file()`. In this case opcache will try to preload the class if all dependencies for it are also available. Otherwise, it will throw a warning and cache the script the old-fashioned way. Prior to PHP 8.1 the "all dependencies" requirement was rather problematic.

In earlier PHP versions, unlinked classes were persisted in two parts: One actually immutable one, and another that had to be copied into per-request memory, because it may be modified at run-time. This included property types as well as constant/property initializers. If these could not be fully resolved during preloading, the class couldn't be preloaded, because we cannot perform per-request copies in that case. In PHP 8.1 all remaining run-time modifiable parts were switched to map pointers, thus relaxing the constraint on what counts as a "dependency". Now this only includes parents/interfaces/traits, as well as types necessary to perform variance checks.

The variance checks are another problem: Whether or not an argument/return type is required to perform variance checks is very hard to determine in advance. This depends on whether a method is actually an override (which is non-obvious in the presence of traits) and whether the subtyping relationship can be determined without loading the class (e.g. if the types in the parent and child method are exactly the same). Previous PHP versions solved this heuristically, requiring more dependencies than necessary. PHP 8.1 instead will simply attempt inheritance on a copy of the class and discard it if it fails.

This means that `opcache_compile_file()` based preloading should be a lot more predictable in PHP 8.1.

## File cache

The file cache introduced in PHP 7 can be used either standalone (`opcache.file_cache_only`) or in conjunction with the SHM cache as a second-level cache. In the latter case, it will be used on a cold start, or when the SHM cache is unavailable during an opcache restart. On Windows, the file cache fallback is enabled by default, to make sure that at least some caching is available if SHM reattachment fails.

File cache serialization starts from the persisted representation of the script, either in SHM (second-level) or in a temporary memory region (standalone), but created using the usual persistence mechanism. The actual serialization then replaces all pointers with offsets into the memory region ("pointer unswizzling"). This allows efficient unserialization by adding the new base pointer to all pointers.

The primary complication in this model are interned strings, as these are the only pointers that do not point into the persisted memory region. Referenced interned strings are instead serialized into a separate memory region. On unserialization, an attempt is made to convert these back to SHM interned strings.

Unserialization works by copying the file contents (including the serialized script and the interned string area) into a buffer. In standalone mode, this buffer is non-temporary and unserialization (pointer swizzling) occurs directly in this buffer.

In second-level mode, this buffer is usually temporary. Instead a SHM allocation is made, into which the serialized script is copied and where it is unserialized. In this case all interned strings also need to be converted into SHM interned strings. The temporary buffer can then be discarded. However, if not all interned strings can be inserted due to an interned string buffer overflow, then the SHM segment is abandoned and a per-request unserialization as in the standalone case is performed.
