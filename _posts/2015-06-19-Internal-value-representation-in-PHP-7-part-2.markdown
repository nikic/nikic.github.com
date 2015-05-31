---
layout: post
title: Internal value representation in PHP 7 - Part 2
excerpt: Covers the implementation of complex types like strings and objects in PHP 7, as well as a number of special types like indirect zvals.
---
In the [first part][part1] of this article, high level changes in the internal value representation between PHP 5 and
PHP 7 were discussed. As a reminder, the main difference was that zvals are no longer individually allocated and don't
store a reference count themselves. Simple values like integers or floats can be stored directly in a zval, while
complex values are represented using a pointer to a separate structure.

The additional structures for complex zval values all use a common header, which is defined by `zend_refcounted`:

{% highlight c %}
struct _zend_refcounted {
    uint32_t refcount;
    union {
        struct {
            ZEND_ENDIAN_LOHI_3(
                zend_uchar    type,
                zend_uchar    flags,
                uint16_t      gc_info)
        } v;
        uint32_t type_info;
    } u;
};
{% endhighlight %}

This header now holds the `refcount`, the `type` of the value and cycle collection info (`gc_info`), as well as a slot
for type-specific `flags`.

In the following the details of the individual complex types will be discussed and compared to the previous
implementation in PHP 5. One of the complex types are references, which were already covered in the previous part.
Another type that will not be covered here are resources, because I don't consider them to be interesting.

Strings
-------

PHP 7 represents strings using the `zend_string` type, which is defined as follows:

{% highlight c %}
struct _zend_string {
    zend_refcounted   gc;
    zend_ulong        h;        /* hash value */
    size_t            len;
    char              val[1];
};
{% endhighlight %}

Apart from the refcounted header, a string contains a hash cache `h`, a length `len` and a value `val`. The hash cache
is used to avoid recomputing the hash of the string every time it is used to look up a key in a hashtable. On first use
it will be initialized to the (non-zero) hash.

If you're not familiar with the quite extensive lore of dirty C hacks, the definition of `val` may look strange: It is
declared as a char array with a single element - but surely we want to store strings longer than one character? This
uses a technique called the "struct hack": The array is declared with only one element, but when creating the
`zend_string` we'll allocate it to hold a larger string. We'll still be able to access the larger string through the
`val` member.

Of course this is technically undefined behavior, because we end up reading and writing past the end of a
single-character array, however C compilers know not to mess with your code when you do this. C99 explicitly supports
this in the form of "flexible array members", however thanks to our dear friends at Microsoft, nobody needing
cross-platform compatibility can actually use C99.

The new string type has some advantages over using normal C strings: Firstly, it directly embeds the string length. This
means that the length of a string no longer needs to be passed around all over the place. Secondly, as the string now
has a refcounted header, it is possible to share a string in multiple places without using zvals. This is particularly
important for sharing hashtable keys.

The new string type also has one large disadvantage: While it is easy to get a C string from a zend_string (just use
`str->val`) it is not possible to directly get a zend_string from a C string -- you need to actually copy the string's
value into a newly allocated zend_string. This is particularly inconvenient when dealing with literal strings (constant
strings occurring in the C source code).

There are a number of flags a string can have (which are stored in the GC flags field):

{% highlight c %}
#define IS_STR_PERSISTENT           (1<<0) /* allocated using malloc */
#define IS_STR_INTERNED             (1<<1) /* interned string */
#define IS_STR_PERMANENT            (1<<2) /* interned string surviving request boundary */
{% endhighlight %}

Persistent strings use the normal system allocator instead of the Zend memory manager (ZMM) and as such can live longer
than one request. Specifying the used allocator as a flag allows us to transparently use persistent strings in zvals,
while previously in PHP 5 a copy into the ZMM was required beforehand.

Interned strings are strings that won't be destroyed until the end of a request and as such don't need to use
refcounting. They are also deduplicated, so if a new interned string is created the engine first checks if an interned
string with the given content already exists. All strings that occur literally in PHP source code (this includes string
literals, variable and function names, etc) are usually interned. Permanent strings are interned strings that were
created before a request starts. While normal interned strings are destroyed on request shutdowns, permanent strings
are kept alive.

If opcache is used interned strings will be stored in shared memory (SHM) and as such shared across all PHP worker
processes. In this case the notion of permanent strings becomes irrelevant, because interned strings will never be
destroyed.

Arrays
------

I will not talk about the details of the new array implementation here, as this is already covered in a [previous
article][ht7]. It's no longer accurate in some details due to recent changes, but all the concepts are still the same.

There is only one new array-related concept I'll mention here, because it is not covered in the hashtable post:
Immutable arrays. These are essentially the array equivalent of interned strings, in that they don't use refcounting
and always live until the end of the request (or longer).

Due to some memory management concerns, immutable arrays are only used if opcache is enabled. To see what kind of
difference this can make, consider the following script:

{% highlight php startinline %}
for ($i = 0; $i < 1000000; ++$i) {
    $array[] = ['foo'];
}
var_dump(memory_get_usage());
{% endhighlight %}

With opcache the memory usage is 32 MiB, but without opcache usage rises to a whopping 390 MB, because each element of
`$array` will get a new copy of `['foo']` in this case. The reason an actual copy is done here (instead of a refcount
increase) is that literal VM operands don't use refcounting to avoid SHM corruption. I hope we can improve this
currently catastrophic case to work better without opcache in the future.

Objects in PHP 5
----------------

Before considering the object implementation in PHP 7, let's first walk through how things worked in PHP 5 and highlight
some of the inefficiencies: The zval itself used to store a `zend_object_value`, which is defined as follows:

{% highlight c %}
typedef struct _zend_object_value {
    zend_object_handle handle;
    const zend_object_handlers *handlers;
} zend_object_value;
{% endhighlight %}

The `handle` is a unique ID of the object which can be used to look up the object data. The `handlers` are a VTable of
function pointers implementing various behaviors of an object. For "normal" PHP objects this handler table will always
be the same, but objects created by PHP extensions can use a custom set of handlers that modifies the way it behaves
(e.g. by overloading operators).

The object handle is used as an index into the "object store", which is an array of object store buckets defined as
follows:

{% highlight c %}
typedef struct _zend_object_store_bucket {
    zend_bool destructor_called;
    zend_bool valid;
    zend_uchar apply_count;
    union _store_bucket {
        struct _store_object {
            void *object;
            zend_objects_store_dtor_t dtor;
            zend_objects_free_object_storage_t free_storage;
            zend_objects_store_clone_t clone;
            const zend_object_handlers *handlers;
            zend_uint refcount;
            gc_root_buffer *buffered;
        } obj;
        struct {
            int next;
        } free_list;
    } bucket;
} zend_object_store_bucket;
{% endhighlight %}

There's quite a lot of things going on here. The first three members are just some metadata (whether the destructor
of the object was called, whether this bucket is used at all and how many times this object was visited by some
recursive algorithm). The following union distinguishes the case where the bucket is currently used or whether it is
part of the bucket free list. Important for use is the case where `struct _store_object` is used:

The first member `object` is a pointer to the actual object (finally). It is not directly embedded in the object store
bucket, because objects have no fixed size. The object pointer is followed by three handlers managing destruction,
freeing and cloning. Note that in PHP destruction and freeing of objects are distinct steps, with the former being
skipped in some cases ("unclean shutdown"). The clone handler is virtually never used. Because these storage handlers
are not part of the normal object handlers (for whatever reason) they will be duplicated for every single object, rather
than being shared.

These object store handlers are followed by a pointer to the ordinary object `handlers`. These are stored in case the
object is destroyed without a `zval` being known (which usually stores the handlers).

The bucket also contains a `refcount`, which is somewhat odd given how in PHP 5 the zval already stores a reference
count. Why do we need another? The problem is that while usually zvals are "copied" simply by increasing their refcount,
there are also cases where a hard copy occurs, i.e. an entirely new zval is allocated with the same `zend_object_value`.
In this case two distinct zvals end up using the same object store bucket, so it needs to be refcounted as well. This
kind of "double refcounting" is one of the inherent issues of the PHP 5 zval implementation. The `buffered` pointer into
the GC root buffer is also duplicated for similar reasons.

Now let's look at the actual `object` that the object store points to. For normal userland objects it is defined as
follows:

{% highlight c %}
typedef struct _zend_object {
    zend_class_entry *ce;
    HashTable *properties;
    zval **properties_table;
    HashTable *guards;
} zend_object;
{% endhighlight %}

The `zend_class_entry` is a pointer to the class this object is an instance of. The two following members are used for
two different ways of storing object properties. For dynamic properties (i.e. ones that are added at runtime and not
declared in the class) the `properties` hashtable is used, which just maps (mangled) property names to their values.

However for declared properties an optimization is used: During compilation every such property is assigned an index and
the value of the property is stored at that index in the `properties_table`. The mapping between property names and
their index is stored in a hashtable in the class entry. As such the memory overhead of the hashtable is avoided for
individual objects. Furthermore the index of a property is cached polymorphically at runtime.

The `guards` hashtable is used to implement the recursion behavior of magic methods like `__get`, which I won't go into
here.

Apart from the double refcounting issue already previously mentioned, the object representation is also heavy on memory
usage with 136 bytes for a minimal object with a single property (not counting zvals). Furthermore there is a lot of
indirection involved: For example, to fetch a property on an object zval, you first have to fetch the object store
bucket, then the zend object, then the properties table and then the zval it points to. As such there are already four
levels of indirection at a minimum (and in practice it will be no fewer than seven).

Objects in PHP 7
----------------

PHP 7 tries to improve on all of these issues by getting rid of double refcounting, dropping some of the memory bloat
and reducing indirection. Here's the new `zend_object` structure:

{% highlight c %}
struct _zend_object {
    zend_refcounted   gc;
    uint32_t          handle;
    zend_class_entry *ce;
    const zend_object_handlers *handlers;
    HashTable        *properties;
    zval              properties_table[1];
};
{% endhighlight %}

Note that this structure is now (nearly) all that is left of an object: The `zend_object_value` has been replaced with a
direct pointer to the object and the object store, while not entirely gone, is much less significant.

Apart from now including the customary `zend_refcounted` header, you can see that the `handle` and the `handlers` of the
object value have been moved into the `zend_object`. Furthermore the `properties_table` now also makes use of the struct
hack, so the `zend_object` and the properties table will be allocated in one chunk. And of course, the property table
now directly embeds zvals, instead of containing pointers to them.

The `guards` table is no longer directly present in the object structure. Instead it will be stored in the first
`properties_table` slot if it is needed, i.e. if the object uses `__get` etc. But if these magic methods are not used,
the guards table is elided.

The `dtor`, `free_storage` and `clone` handlers that were previously stored in the object store bucket have now been
moved into the `handlers` table, which starts as follows:

{% highlight c %}
struct _zend_object_handlers {
    /* offset of real object header (usually zero) */
    int                                     offset;
    /* general object functions */
    zend_object_free_obj_t                  free_obj;
    zend_object_dtor_obj_t                  dtor_obj;
    zend_object_clone_obj_t                 clone_obj;
    /* individual object functions */
    // ... rest is about the same in PHP 5
};
{% endhighlight %}

At the top of the handler table is an `offset` member, which is quite clearly not a handler. This offset has to do with
how internal objects are represented: An internal object always embeds the standard `zend_object`, but typically also
adds a number of additional members. In PHP 5 this was done by adding them after the standard object:

{% highlight c %}
struct custom_object {
    zend_object std;
    uint32_t something;
    // ...
};
{% endhighlight %}

This means that if you get a `zend_object*` you can simply cast it to your custom `struct custom_object*`. This is the
standard means of implementing structure inheritance in C. However in PHP 7 there is an issue with this particular
approach: Because `zend_object` uses the struct hack for storing the properties table, PHP will be storing properties
past the end of `zend_object` and thus overwriting additional internal members. This is why in PHP 7 additional members
are stored *before* the standard object instead:

{% highlight c %}
struct custom_object {
    uint32_t something;
    // ...
    zend_object std;
};
{% endhighlight %}

However this means that it is no longer possible to directly convert between a `zend_object*` and a
`struct custom_object*` with a simple cast, because both are separated by a offset. This offset is what's stored in the
first member of the object handler table. At compile-time the offset can be determined using the `offsetof()` macro.

You may wonder why PHP 7 objects still contain a `handle`. After all, we now directly store a pointer to the
`zend_object`, so we no longer need the handle to look up the object in the object store.

However the handle is still necessary, because the object store still exists, albeit in a significantly reduced form. It
is now a simple array of pointers to objects. When an object is created a pointer to it is inserted into the object
store at the `handle` index and removed once the object is freed.

Why do we still need the object store? The reason behind this is that during request shutdown, there comes a point where
it is no longer safe to run userland code, because the executor is already partially shut down. To avoid this PHP will
run all object destructors at an early point during shutdown and prevent them from running at a later point in time. For
this a list of all active objects is needed.

Furthermore the handle is useful for debugging, because it gives each object a unique ID, so it's easy to see whether
two objects are really the same or just have the some content. HHVM still stores an object handle despite not having a
concept of an object store.

Comparing with the PHP 5 implementation, we now have only one refcount (as the zval itself no longer has one) and the
memory usage is much smaller: We need 40 bytes for the base object and 16 bytes for every declared property, already
including its zval. The amount of indirection is also significantly reduced, as many of the intermediate structure were
either dropped or embedded. As such reading a property is now only a single level of indirection, rather than four.

Indirect zvals
--------------

At this point we have covered all of the normal zval types, however there are a couple of additional special types that
are used only in certain circumstances. One that was newly added in PHP 7 is `IS_INDIRECT`.

An indirect zval signifies that its value is stored in some other location. Note that this is different from the
`IS_REFERENCE` type in that it *directly* points to another zval, rather than a `zend_reference` structure that embeds
a zval.

To understand under what circumstances this may be necessary, consider how PHP implements variables (though the same
also applies to object property storage):

All variables that are known at compile-time are assigned an index and their value will be stored at that index in the
compiled variables (CV) table. However PHP also allows you to dynamically reference variables, either by using variable
variables or, if you are in global scope, through `$GLOBALS`. If such an access occurs, PHP will create a symbol table
for the function/script, which contains a map from variable names to their values.

This leads to the question: How can both forms of access be supported at the same time? We need table-based CV access
for normal variable fetches and symtable-based access for varvars. In PHP 5 the CV table used doubly-indirected `zval**`
pointers. Normally those pointers would point to a second table of `zval*` pointer that would point to the actual zvals:

    +------ CV_ptr_ptr[0]
    | +---- CV_ptr_ptr[1]
    | | +-- CV_ptr_ptr[2]
    | | |
    | | +-> CV_ptr[0] --> some zval
    | +---> CV_ptr[1] --> some zval
    +-----> CV_ptr[2] --> some zval

Now, once a symbol table came into use, the second table with the single `zval*` pointers was left unused and the
`zval**` pointers were updated to point into the hashtable buckets instead. Here illustrated assuming the three
variables are called `$a`, `$b` and `$c`:

    CV_ptr_ptr[0] --> SymbolTable["a"].pDataPtr --> some zval
    CV_ptr_ptr[1] --> SymbolTable["b"].pDataPtr --> some zval
    CV_ptr_ptr[2] --> SymbolTable["c"].pDataPtr --> some zval

In PHP 7 using the same approach is no longer possible, because a pointer into a hashtable bucket will be invalidated
when the hashtable is resized. Instead PHP 7 uses the reverse strategy: For the variables that are stored in the CV
table, the symbol hashtable will contain an `INDIRECT` entry pointing to the CV entry. The CV table will not be
reallocated for the lifetime of the symbol table, so there is no problem with invalidated pointers.

So if you have a function with CVs `$a`, `$b` and `$c`, as well as a dynamically created variable `$d`, the symbol
table could looks something like this:

    SymbolTable["a"].value = INDIRECT --> CV[0] = LONG 42
    SymbolTable["b"].value = INDIRECT --> CV[1] = DOUBLE 42.0
    SymbolTable["c"].value = INDIRECT --> CV[2] = STRING --> zend_string("42")
    SymbolTable["d"].value = ARRAY --> zend_array([4, 2])

Indirect zvals can also point to an `IS_UNDEF` zval, in which case it is treated as if the hashtable does not contain
the associated key. So if `unset($a)` writes an `UNDEF` type into `CV[0]`, then this will be treated like the symbol
table no longer having a key `"a"`.

Constants and ASTs
------------------

There are two more special types `IS_CONSTANT` and `IS_CONSTANT_AST` which exist both in PHP 5 and PHP 7 and deserve a
mention here. To understand what these do, consider the following example:

{% highlight php startinline %}
function test($a = ANSWER,
              $b = ANSWER * ANSWER) {
    return $a + $b;
}

define('ANSWER', 42);
var_dump(test()); // int(42 + 42 * 42)
{% endhighlight %}

The default values for the parameters of the `test()` function make use of the constant `ANSWER` - however this constant
is not yet defined when the function was declared. The constant will only be available once the `define()` call has run.

For this reason parameter and property default values, constants and everything else accepting a "static expression"
have the ability to postpone evaluation of the expression until first use.

If the value is a constant (or class constant), which is the most common case for late-evaluation, this is signaled
using an `IS_CONSTANT` zval with the constant name. If the value is an expression, a `IS_CONSTANT_AST` zval pointing to
an abstract syntax tree (AST) is used.

And this concludes our walk through the PHP 7 value representation. Two more topics I'd like to write about at some
point are some of the optimizations done in the virtual machine, in particular the new calling convention, as well as
the improvements that were made to the compiler infrastructure.


  [part1]: //nikic.github.io/2015/05/05/Internal-value-representation-in-PHP-7-part-1.html
  [ht7]: http://nikic.github.io/2014/12/22/PHPs-new-hashtable-implementation.html
