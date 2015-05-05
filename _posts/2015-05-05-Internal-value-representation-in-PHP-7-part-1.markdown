---
layout: post
title: Internal value representation in PHP 7 - Part 1
excerpt: Describes and compares the zval implementations used by PHP 5 and PHP 7, including a discussion of references. Other types are covered in the next part.
---
My last article described the [improvements to the hashtable implementation][ht7] that were introduced in PHP 7. This
followup will take a look at the new representation of PHP values in general.

Due to the amount of material to cover, the article is split in two parts: This part will describe how the zval (Zend
value) implementation differs between PHP 5 and PHP 7, and also discuss the implementation of references. The second
part will investigate the realization of individual types like strings or objects in more detail.

Zvals in PHP 5
--------------

In PHP 5 the zval struct is defined as follows:

{% highlight c %}
typedef struct _zval_struct {
    zvalue_value value;
    zend_uint refcount__gc;
    zend_uchar type;
    zend_uchar is_ref__gc;
} zval;
{% endhighlight %}

As you can see, a zval consists of a `value`, a `type` and some additional `__gc` information, which we'll talk about in
a moment. The `value` member is a union of different possible values that a zval can store:

{% highlight c %}
typedef union _zvalue_value {
    long lval;                 // For booleans, integers and resources
    double dval;               // For floating point numbers
    struct {                   // For strings
        char *val;
        int len;
    } str;
    HashTable *ht;             // For arrays
    zend_object_value obj;     // For objects
    zend_ast *ast;             // For constant expressions
} zvalue_value;
{% endhighlight %}

A C union is a structure in which only one member can be active at a time and those size matches the size of its largest
member. All members of the union will be stored in the *same* place in memory and will be interpreted differently
depending on which one you access. If you read the `lval` member of the above union, its value will be interpreted as a
signed integer. If you read the `dval` member the value will be interpreted as a double-precision floating point
number instead. And so on.

To figure out which of these union members is currently in use, the `type` property of a zval stores a type tag, which
is simply an integer:

{% highlight c %}
#define IS_NULL     0      /* Doesn't use value */
#define IS_LONG     1      /* Uses lval */
#define IS_DOUBLE   2      /* Uses dval */
#define IS_BOOL     3      /* Uses lval with values 0 and 1 */
#define IS_ARRAY    4      /* Uses ht */
#define IS_OBJECT   5      /* Uses obj */
#define IS_STRING   6      /* Uses str */
#define IS_RESOURCE 7      /* Uses lval, which is the resource ID */

/* Special types used for late-binding of constants */
#define IS_CONSTANT 8
#define IS_CONSTANT_AST 9
{% endhighlight %}

Reference counting in PHP 5
---------------------------

Zvals in PHP 5 are (with a few exceptions) allocated on the heap and PHP needs some way to keep track which zvals are
currently in use and which should be freed. For this purpose reference counting is employed: The `refcount__gc` member
of the zval structure stores how often a zval is currently "referenced". For example in `$a = $b = 42` the value `42`
is referenced by two variables, so its refcount is 2. If the refcount reaches zero, it means a value is unused and can
be freed.

Note that the references that the refcount refers to (how many times a value is currently used) have nothing to do with
PHP references (using `&`). I will always using the terms "reference" and "PHP reference" to disambiguate both concepts
in the following. For now we'll ignore PHP references altogether.

A concept that is closely related to reference counting is "copy on write": A zval can only be shared between multiple
users as long as it isn't modified. In order to change a shared zval it needs to be duplicated ("separated") and the
modification will happen only on the duplicated zval.

Lets look at an example that shows off both copy-on-write and zval destruction:

{% highlight php startinline %}
$a = 42;   // $a         -> zval_1(type=IS_LONG, value=42, refcount=1)
$b = $a;   // $a, $b     -> zval_1(type=IS_LONG, value=42, refcount=2)
$c = $b;   // $a, $b, $c -> zval_1(type=IS_LONG, value=42, refcount=3)

// The following line causes a zval separation
$a += 1;   // $b, $c -> zval_1(type=IS_LONG, value=42, refcount=2)
           // $a     -> zval_2(type=IS_LONG, value=43, refcount=1)

unset($b); // $c -> zval_1(type=IS_LONG, value=42, refcount=1)
           // $a -> zval_2(type=IS_LONG, value=43, refcount=1)

unset($c); // zval_1 is destroyed, because refcount=0
           // $a -> zval_2(type=IS_LONG, value=43, refcount=1)
{% endhighlight %}

Reference counting has one fatal flaw: It is not able to detect and release cyclic references. To handle this PHP uses
an additional [cycle collector][cycle]. Whenever the refcount of a zval is decremented and there is a chance that this
zval is part of a cycle, the zval is written into a "root buffer". Once this root buffer is full, potential cycles will
be collected using a mark and sweep garbage collection.

In order to support this additional cycle collector, the actually used zval structure is the following:

{% highlight c %}
typedef struct _zval_gc_info {
    zval z;
    union {
        gc_root_buffer       *buffered;
        struct _zval_gc_info *next;
    } u;
} zval_gc_info;
{% endhighlight %}

The `zval_gc_info` structure embeds the normal zval, as well as one additional pointer - note that `u` is a union, so
this is really just one pointer with two different types it may point to. The `buffered` pointer is used to store where
in the root buffer this zval is referenced, so that it may be removed from it if it's destroyed before the cycle
collector runs (which is very likely). `next` is used when the collector destroys values, but I won't go into that here.

Motivation for change
---------------------

Let's talk about sizes a bit (all sizes are for 64-bit systems): First of all, the `zvalue_value` union is 16 bytes
large, because both the `str` and `obj` members have that size. The whole `zval` struct is 24 bytes (due to padding) and
`zval_gc_info` is 32 bytes. On top of this, allocating the zval on the heap adds another 16 bytes of allocation
overhead. So we end up using 48 bytes per zval - although this zval may be used by multiple places.

At this point we can start thinking about the (many) ways in which this zval implementation is inefficient. Consider the
simple case of a zval storing an integer, which by itself is 8 bytes. Additionally the type-tag needs to be stored in
any case, which is a single byte by itself, but due to padding needs another 8 bytes.

To these 16 bytes that we really "need" (in first approximation), we add another 16 bytes handling reference counting
and cycle collection and another 16 bytes of allocation overhead. Not to mention that we actually have to perform that
allocation and the subsequent free, both being quite expensive operations.

This raises the question: Does a simple integer value *really* need to be stored as a reference-counted,
cycle-collectible, heap-allocated value? The answer to this question is of course, no, this doesn't make sense.

Here is a summary of the primary problems with the PHP 5 zval implementation:

 * Zvals (nearly) always require a heap allocation.
 * Zvals are always reference counted and always have cycle collection information, even in cases where sharing the
   value is not worthwhile (an integer) and it can't form cycles.
 * Directly refcounting the zvals leads to double refcounting in the case of objects and resources. The reasons behind
   this will be explained in the next part.
 * Some cases involve quite an awesome amount of indirection. For example to access the object stored in a variable, a
   total of four pointers need to be dereferenced (which means following a pointer chain of length four). Once again
   this will be discussed in the next part.
 * Directly refcounting the zvals also means that values can only be shared between zvals. For example it's not possible
   to share a string between a zval and hashtable key (without storing the hashtable key as a zval as well).

Zvals in PHP 7
--------------

And this brings us to the new zval implementation in PHP 7. The fundamental change that was implemented, is that zvals
are no longer individually heap-allocated and no longer store a refcount themselves. Instead any complex values they
may point to (like strings, arrays or objects) will store the refcount themselves. This has the following advantages:

 * Simple values do not require allocation and don't use refcounting.
 * There is no more double refcounting. In the object case, only the refcount in the object is used now.
 * Because the refcount is now stored in the value itself, the value can be shared independently of the zval structure.
   A string can be used both in a zval and a hashtable key.
 * There is a lot less indirection, i.e. the number of pointers you need to follow to get to a value is lower.

Now lets take a look at how the new zval is defined:

{% highlight c %}
struct _zval_struct {
    zend_value value;
    union {
        struct {
            ZEND_ENDIAN_LOHI_4(
                zend_uchar type,
                zend_uchar type_flags,
                zend_uchar const_flags,
                zend_uchar reserved)
        } v;
        uint32_t type_info;
    } u1;
    union {
        uint32_t var_flags;
        uint32_t next;                 // hash collision chain
        uint32_t cache_slot;           // literal cache slot
        uint32_t lineno;               // line number (for ast nodes)
        uint32_t num_args;             // arguments number for EX(This)
        uint32_t fe_pos;               // foreach position
        uint32_t fe_iter_idx;          // foreach iterator index
    } u2;
};
{% endhighlight %}

The first member stays pretty similar, this is still a `value` union. The second member is an integer storing type
information, which is further subdivided into individual bytes using a union (you can ignore the `ZEND_ENDIAN_LOHI_4`
macro, which just ensures a consistent layout across platforms with different endianness). The important parts of this
substructure are the `type` (which is similar to what it was before) and the `type_flags`, which I'll explain in a
moment.

At this point there exists a small problem: The `value` member is 8 bytes large and due to struct padding adding even a
single byte to that grows the zval size to 16 bytes. However we obviously don't need 8 bytes just to store a type. This
is why the zval contains the additional `u2` union, which remains unused by default, but can be repurposed by the
surrounding code to store 4 bytes of data. The different union members correspond to different usages of this extra data
slot.

The `value` union looks slightly different in PHP 7:

{% highlight c %}
typedef union _zend_value {
    zend_long         lval;
    double            dval;
    zend_refcounted  *counted;
    zend_string      *str;
    zend_array       *arr;
    zend_object      *obj;
    zend_resource    *res;
    zend_reference   *ref;
    zend_ast_ref     *ast;

    // Ignore these for now, they are special
    zval             *zv;
    void             *ptr;
    zend_class_entry *ce;
    zend_function    *func;
    struct {
        ZEND_ENDIAN_LOHI(
            uint32_t w1,
            uint32_t w2)
    } ww;
} zend_value;
{% endhighlight %}

First of all, note that the value union is now 8 bytes instead of 16. It will only store integers (`lval`) and doubles
(`dval`) directly, everything else is a pointer. All the pointer types (apart from those marked as special above) use
refcounting and have a common header defined by `zend_refcounted`:

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

Of course the structure contains a refcount. Additionally it contains a `type`, some `flags` and `gc_info`. The `type`
just duplicates the zval type and allows the GC to distinguish different refcounted structures without storing a zval.
The `flags` are used for different purposes with different types and will be explained for each type separately in the
next part.

The `gc_info` is the equivalent of the `buffered` entry in the old zvals. However instead of storing a pointer into the
root buffer it now contains an index into it. Because the root buffer has a fixed size (10000 elements) it is enough to
use a 16 bit number for this instead of a 64 bit pointer. The `gc_info` info also encodes the "color" of the node, which
is used to mark nodes during collection.

Zval memory management
----------------------

I've mentioned that zvals are no longer individually heap-allocated. However they obviously still need to be stored
*somewhere*, so how does this work? While zvals are still mostly part of heap-allocated structures, they are directly
embedded into them. E.g. a hashtable bucket will directly embed a `zval` instead of storing a pointer to a separate
zval. The compiled variables table of a function or the property table of an object will be zval arrays that are
allocated in one chunk, instead of storing pointers to separate zvals. As such zvals are now usually stored with one
level of indirection less. What was previously a `zval*` is now a `zval`.

When a zval is used in a new place, previously this meant copying a `zval*` and incrementing its refcount. Now it means
copying the contents of a `zval` (ignoring `u2`) instead and *maybe* incrementing the refcount of the value it points
to, if said value uses refcounting.

How does PHP know whether a value is refcounted? This cannot be determined solely based on the type, because some types
like strings and arrays are not always refcounted. Instead one bit of the zvals `type_info` member determines whether or
not the zval is refcounted. There are a number of other bits encoding properties of the type:

{% highlight c %}
#define IS_TYPE_CONSTANT            (1<<0)   /* special */
#define IS_TYPE_IMMUTABLE           (1<<1)   /* special */
#define IS_TYPE_REFCOUNTED          (1<<2)
#define IS_TYPE_COLLECTABLE         (1<<3)
#define IS_TYPE_COPYABLE            (1<<4)
#define IS_TYPE_SYMBOLTABLE         (1<<5)   /* special */
{% endhighlight %}

The three primary properties a type can have are "refcounted", "collectable" and "copyable". You already know what
refcounted means. Collectable means that the zval can participate in a cycle. E.g. strings are (often) refcounted, but
there's no way you can create a cycle with a string in it.

Copyability determines whether the value needs to copied when a "duplication" is performed. A duplication is a hard
copy, e.g. if you duplicate a zval that points to an array, this will not simply increase the refcount on the array.
Instead a new and independent copy of the array will be created. However for some types like objects and resources even
a duplication should only increment the refcount - such types are called non-copyable. This matches the passing
semantics of objects and resources (which are, for the record, not passed by reference).

The following table shows the different types and what type flags they use. "Simple types" refers to types like integers
or booleans that don't use a pointer to a separate structure. A column for the "immutable" flag is also present, which
is used to mark immutable arrays and will be discussed in more detail in the next part.

                    | refcounted | collectable | copyable | immutable
    ----------------+------------+-------------+----------+----------
    simple types    |            |             |          |
    string          |      x     |             |     x    |
    interned string |            |             |          |
    array           |      x     |      x      |     x    |
    immutable array |            |             |          |     x
    object          |      x     |      x      |          |
    resource        |      x     |             |          |
    reference       |      x     |             |          |

At this point, lets take a look at two examples of how the zval management works in practice. First, an example using
integers based off the PHP 5 example from above:

{% highlight php startinline %}
$a = 42;   // $a = zval_1(type=IS_LONG, value=42)

$b = $a;   // $a = zval_1(type=IS_LONG, value=42)
           // $b = zval_2(type=IS_LONG, value=42)

$a += 1;   // $a = zval_1(type=IS_LONG, value=43)
           // $b = zval_2(type=IS_LONG, value=42)

unset($a); // $a = zval_1(type=IS_UNDEF)
           // $b = zval_2(type=IS_LONG, value=42)
{% endhighlight %}

This is pretty boring. As integers are no longer shared, both variables will use separate zvals. Don't forget that these
are now embedded rather than allocated, which I try to signify by writing `=` instead of a `->` pointer. Unsetting a
variable will set the type of the corresponding zval to `IS_UNDEF`. Now consider a more interesting case where a complex
value is involved:

{% highlight php startinline %}
$a = [];   // $a = zval_1(type=IS_ARRAY) -> zend_array_1(refcount=1, value=[])

$b = $a;   // $a = zval_1(type=IS_ARRAY) -> zend_array_1(refcount=2, value=[])
           // $b = zval_2(type=IS_ARRAY) ---^

// Zval separation occurs here
$a[] = 1   // $a = zval_1(type=IS_ARRAY) -> zend_array_2(refcount=1, value=[1])
           // $b = zval_2(type=IS_ARRAY) -> zend_array_1(refcount=1, value=[])

unset($a); // $a = zval_1(type=IS_UNDEF) and zend_array_2 is destroyed
           // $b = zval_2(type=IS_ARRAY) -> zend_array_1(refcount=1, value=[])
{% endhighlight %}

Here each variable still has a separate (embedded) zval, but both zvals point to the same (refcounted) `zend_array`
structure. Once a modification is done the array needs to be duplicated. This case is similar to how things work in PHP
5.

Types
-----

Lets take a closer look at what types are supported in PHP 7:

{% highlight c %}
// regular data types
#define IS_UNDEF                    0
#define IS_NULL                     1
#define IS_FALSE                    2
#define IS_TRUE                     3
#define IS_LONG                     4
#define IS_DOUBLE                   5
#define IS_STRING                   6
#define IS_ARRAY                    7
#define IS_OBJECT                   8
#define IS_RESOURCE                 9
#define IS_REFERENCE                10

// constant expressions
#define IS_CONSTANT                 11
#define IS_CONSTANT_AST             12

// internal types
#define IS_INDIRECT                 15
#define IS_PTR                      17
{% endhighlight %}

This list is quite similar to what was used in PHP 5, however there are a few additions:

 * The `IS_UNDEF` type is used in places where previously a `NULL` zval pointer (not to be confused with an `IS_NULL`
   zval) was used. For example, in the refcounting examples above the `IS_UNDEF` type is set for variables that have
   been `unset`.
 * The `IS_BOOL` type has been split into `IS_FALSE` and `IS_TRUE`. As such the value of the boolean is now encoded in
   the type, which allows the optimization of a number of type-based checks. This change is transparent to userland,
   where this is still a single "boolean" type.
 * PHP references no longer use an `is_ref` flag on the zval and use a new `IS_REFERENCE` type instead. How this works
   will be described in the next section.
 * The `IS_INDIRECT` and `IS_PTR` types are special internal types.

The `IS_LONG` type now uses a `zend_long` value instead of an ordinary C long. The reason behind this is that on 64-bit
Windows (LLP64) a `long` is only 32-bit wide, so PHP 5 ended up always using 32-bit numbers on Windows. PHP 7 will allow
you to use 64-bit numbers if you're on an 64-bit operating system, even if that operating system is Windows.

Details of the individual `zend_refcounted` types will be discussed in the next part. For now we'll only look at the
implementation of PHP references.

References
----------

PHP 7 uses an entirely different approach to handling PHP `&` references than PHP 5 (and I can tell you that this change
is one of the largest source of bugs in PHP 7). Lets start by taking a look at how PHP references used to work in PHP 5:

Normally, the copy-on-write principle says that before modifying a zval it needs to be separated, in order to make sure
you don't end up changing the value for every place sharing the zval. This matches by-value passing semantics.

For PHP references this does not apply. If a value is a PHP reference, you *want* it to change for every user of the
value. The `is_ref` flag that was part of PHP 5 zvals determined whether a value is a PHP reference and as such whether
it required separation before modification. An example:

{% highlight php startinline %}
$a = [];  // $a     -> zval_1(type=IS_ARRAY, refcount=1, is_ref=0) -> HashTable_1(value=[])
$b =& $a; // $a, $b -> zval_1(type=IS_ARRAY, refcount=2, is_ref=1) -> HashTable_1(value=[])

$b[] = 1; // $a = $b = zval_1(type=IS_ARRAY, refcount=2, is_ref=1) -> HashTable_1(value=[1])
          // Due to the is_ref=1 PHP will *not* separate the zval
{% endhighlight %}

One significant problem with this design is that it's not possible to share a value between a variable that's a PHP
reference and one that isn't. Consider the following example:

{% highlight php startinline %}
$a = [];  // $a         -> zval_1(type=IS_ARRAY, refcount=1, is_ref=0) -> HashTable_1(value=[])
$b = $a;  // $a, $b     -> zval_1(type=IS_ARRAY, refcount=2, is_ref=0) -> HashTable_1(value=[])
$c = $b   // $a, $b, $c -> zval_1(type=IS_ARRAY, refcount=3, is_ref=0) -> HashTable_1(value=[])

$d =& $c; // $a, $b -> zval_1(type=IS_ARRAY, refcount=2, is_ref=0) -> HashTable_1(value=[])
          // $c, $d -> zval_1(type=IS_ARRAY, refcount=2, is_ref=1) -> HashTable_2(value=[])
          // $d is a reference of $c, but *not* of $a and $b, so the zval needs to be copied
          // here. Now we have the same zval once with is_ref=0 and once with is_ref=1.

$d[] = 1; // $a, $b -> zval_1(type=IS_ARRAY, refcount=2, is_ref=0) -> HashTable_1(value=[])
          // $c, $d -> zval_1(type=IS_ARRAY, refcount=2, is_ref=1) -> HashTable_2(value=[1])
          // Because there are two separate zvals $d[] = 1 does not modify $a and $b.
{% endhighlight %}

This behavior of references is one of the reasons why using references in PHP will usually end up being slower than
using normal values. To give a less-contrived example where this is a problem:

{% highlight php startinline %}
$array = range(0, 1000000);
$ref =& $array;
var_dump(count($array)); // <-- separation occurs here
{% endhighlight php %}

Because `count()` accepts its value by-value, but `$array` is a PHP reference, a full copy of the array is done before
passing it off to `count()`. If `$array` weren't a reference, the value would be shared instead.

Now, let's switch to the PHP 7 implementation of PHP references. Because zvals are no longer individually allocated, it
is not possible to use the same approach that PHP 5 used. Instead a new `IS_REFERENCE` type is added, which uses the
`zend_reference` structure as its value:

{% highlight c %}
struct _zend_reference {
    zend_refcounted   gc;
    zval              val;
};
{% endhighlight %}

So essentially a `zend_reference` is simply a refcounted zval. All variables in a reference set will store a zval with
type `IS_REFERENCE` pointing to the same `zend_reference` instance. The `val` zval behaves like any other zval, in
particular it is possible to share a complex value it points to. E.g. an array can be shared between a variable that is
a reference and another that is a value.

Lets go through the above code samples again, this time looking at the PHP 7 semantics. For the sake of brevity I will
stop writing the individual zvals of the variables and only show what structure they point to.

{% highlight php startinline %}
$a = [];  // $a                                     -> zend_array_1(refcount=1, value=[])
$b =& $a; // $a, $b -> zend_reference_1(refcount=2) -> zend_array_1(refcount=1, value=[])

$b[] = 1; // $a, $b -> zend_reference_1(refcount=2) -> zend_array_1(refcount=1, value=[1])
{% endhighlight %}

The by-reference assignment created a new `zend_reference`. Note that the refcount is 2 on the reference (because two
variables are part of the PHP reference set), but the value itself only has a refcount of 1 (because one
`zend_reference` structure points to it). Now consider the case where references and non-references are mixed:

{% highlight php startinline %}
$a = [];  // $a         -> zend_array_1(refcount=1, value=[])
$b = $a;  // $a, $b,    -> zend_array_1(refcount=2, value=[])
$c = $b   // $a, $b, $c -> zend_array_1(refcount=3, value=[])

$d =& $c; // $a, $b                                 -> zend_array_1(refcount=3, value=[])
          // $c, $d -> zend_reference_1(refcount=2) ---^
          // Note that all variables share the same zend_array, even though some are
          // PHP references and some aren't.

$d[] = 1; // $a, $b                                 -> zend_array_1(refcount=2, value=[])
          // $c, $d -> zend_reference_1(refcount=2) -> zend_array_2(refcount=1, value=[1])
          // Only at this point, once an assignment occurs, the zend_array is duplicated.
{% endhighlight %}

The important difference to PHP 5 is that all variables were able to share the same array, even though some were PHP
references and some weren't. Only once some kind of modification is performed the array will be separated. This means
that in PHP 7 it's safe to pass a large, referenced array to `count()`, it is not going to be duplicated. References
will still be slower than normal values, because they require allocation of the `zend_reference` structure (and
indirection through it) and are usually not handled in the fast-path of engine code.

Wrapping up
-----------

To summarize, the primary change that was implemented in PHP 7 is that zvals are no longer individually heap-allocated
and no longer store a refcount themselves. Instead any complex values they may point to (like strings, array or objects)
will store the refcount themselves. This usually leads to less allocations, less indirection and less memory usage.

In the second part of this article the remaining complex types will be discussed.

  [ht7]: http://nikic.github.io/2014/12/22/PHPs-new-hashtable-implementation.html
  [cycle]: http://php.net/manual/en/features.gc.collecting-cycles.php
