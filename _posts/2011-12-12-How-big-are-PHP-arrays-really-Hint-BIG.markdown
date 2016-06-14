---
layout: post
title: "How big are PHP arrays (and values) really? (Hint: BIG!)"
excerpt: PHP's memory usage might seem atrocious to some - twenty times more than the optimum you would have in C. This post tries to explain those numbers and why they are necessary.
---
**Update** (2016-06-14): This article is about memory usage in PHP 5. Memory usage in PHP 7 is, for the case covered
here, approximately three times lower. For more information, see my article on the [hashtable implementation in PHP 7][16].

Upfront I want to thank [Johannes][7] and [Tyrael][8] for their help in finding some of the more
hidden memory usage.

In this post I want to investigate the memory usage of PHP arrays (and values in general) using the
following script as an example, which creates 100000 unique integer array elements and measures the
resulting memory usage:

{% highlight php startinline %}
$startMemory = memory_get_usage();
$array = range(1, 100000);
echo memory_get_usage() - $startMemory, ' bytes';
{% endhighlight %}

How much would you expect it to be? Simple, one integer is 8 bytes (on a 64 bit unix machine and
using the `long` type) and you got 100000 integers, so you obviously will need 800000 bytes.
That's something like 0.76 MBs.

Now try and run the above code. This gives me `14649024 bytes`. Yes, you heard right, that's 13.97 MB - eighteen times
more than we estimated.

So, where does that extra factor of 18 come from?

Summary
-------

For those who don't want to know the full story, here is a quick summary of the memory usage of
the different components involved:

                                 |  64 bit   | 32 bit
    ---------------------------------------------------
    zval                         |  24 bytes | 16 bytes
    + cyclic GC info             |   8 bytes |  4 bytes
    + allocation header          |  16 bytes |  8 bytes
    ===================================================
    zval (value) total           |  48 bytes | 28 bytes
    ===================================================
    bucket                       |  72 bytes | 36 bytes
    + allocation header          |  16 bytes |  8 bytes
    + pointer                    |   8 bytes |  4 bytes
    ===================================================
    bucket (array element) total |  96 bytes | 48 bytes
    ===================================================
    total total                  | 144 bytes | 76 bytes

The above numbers will vary depending on your operating system, your compiler and your compile
options. E.g. if you compile PHP with debug or with thread-safety, you will get different numbers.
But I think that the sizes given above are what you will see on an average 64-bit production build
of PHP 5.3 on Linux.

If you multiply those 144 bytes by our 100000 elements you get 14400000 bytes, which is 13.73 MB.
That's pretty close to the real number - the rest is mostly pointers for uninitialized buckets, but
I'll cover that later.

Now, if you want to have a more detailed analysis of the values mentioned above, read on :)

The `zvalue_value` union
------------------------

First have a look at how PHP stores values. As you know PHP is a weakly typed language, so it needs
some way to switch between the various types fast. PHP uses a [`union`][3] for this, which is
defined as follows in [`zend.h#307`][4] (comments mine):

{% highlight c %}
typedef union _zvalue_value {
    long lval;                // For integers and booleans
    double dval;              // For floats (doubles)
    struct {                  // For strings
        char *val;            //     consisting of the string itself
        int len;              //     and its length
    } str;
    HashTable *ht;            // For arrays (hash tables)
    zend_object_value obj;    // For objects
} zvalue_value;
{% endhighlight %}

If you don't know C, that isn't a problem as the code is pretty straightforward: A `union` is a
means to make some value accessible as various types. For example if you do a `zvalue_value->lval`
you'll get the value interpreted as an integer. If you use `zvalue_value->ht` on the other hand the
value will be interpreted as a pointer to a hashtable (aka array).

But let's not get too much into this here. Important for us only is that the size of a `union`
equals the size of its largest component. The largest component here is the string struct (the
`zend_object_value` struct has the same size as the `str` struct, but I'll leave that out for
simplicity). The string struct stores a pointer (8 bytes) and an integer (4 bytes), which is 12
bytes in total. Due to memory alignment (structs with 12 bytes aren't cool because they aren't a
multiple of 64 bits / 8 bytes) the total size of the struct will be 16 bytes though and that will
also be the size of the union as a whole.

So now we know that we don't need 8 bytes for every value, but 16 - due to PHP's dynamic typing.
Multiplying by 100000 values gives us 1600000 bytes, i.e. 1.53 MB. But the real value is 13.97 MB,
so we can't be there yet.

The `zval` struct
-----------------

And this is quite logical - the union only stores the value itself, but PHP obviously also needs to
store its type and some garbage collection information. The structure holding this information is
called a `zval` and you may have already have heard of it. For more information on why PHP needs it
I'd recommend to read [an article by Sara Golemon][5]. Anyways, this struct is [defined as
follows][10]:

{% highlight c %}
struct _zval_struct {
    zvalue_value value;     // The value
    zend_uint refcount__gc; // The number of references to this value (for GC)
    zend_uchar type;        // The type
    zend_uchar is_ref__gc;  // Whether this value is a reference (&)
};
{% endhighlight %}

The size of a struct is determined by the sum of the sizes of its components: The `zvalue_value` is
16 bytes (as computed above), the `zend_uint` is 4 bytes and the `zend_uchar`s are 1 byte each.
That's a total of 22 bytes. Again due to memory alignment the real size will be 24 bytes though.

So if we store 100000 elements á 24 bytes that would be 2400000 in total, which is 2.29 MB. The
gap is closing, but the real value is still more than six times larger.

The cycles collector (as of PHP 5.3)
------------------------------------

PHP 5.3 introduced a new [garbage collector for cyclic references][9]. For this to work PHP has to
store some additional data. I don't want to explain how the algorithm works here, you can read that
up on the linked page from the manual. Important for our size calculations is that PHP will wrap
every `zval` into a [`zval_gc_info`][11]:

{% highlight c %}
typedef struct _zval_gc_info {
    zval z;
    union {
        gc_root_buffer       *buffered;
        struct _zval_gc_info *next;
    } u;
} zval_gc_info;
{% endhighlight %}

As you can see Zend only adds a union on top of it, which consists of two pointers. As you hopefully
remember the size of a union is the size of its largest component: Both union components are
pointers, thus both have a size of 8 bytes. So the size of the union is 8 bytes too.

If we add that on top of the 24 bytes we already have we get 32 bytes. Multiply that by the 100000
elements and we get a memory usage of 3.05 MB.

The Zend MM allocator
---------------------

C unlike PHP does not manage memory for you. You need to keep track of your allocations yourself.
For this purpose PHP uses a custom memory manager that is optimized specifically for its needs:
[The Zend Memory Manager][12]. The Zend MM is based on [Doug Lea's malloc][13] and adds some PHP
specific optimizations and features (like memory limit, cleaning up after each request and stuff
like that).

What is important for us here is that the MM adds an allocation header to every allocation done
through it. It is [defined as follows][14]:

{% highlight c %}
typedef struct _zend_mm_block {
    zend_mm_block_info info;
#if ZEND_DEBUG
    unsigned int magic;
# ifdef ZTS
    THREAD_T thread_id;
# endif
    zend_mm_debug_info debug;
#elif ZEND_MM_HEAP_PROTECTION
    zend_mm_debug_info debug;
#endif
} zend_mm_block;

typedef struct _zend_mm_block_info {
#if ZEND_MM_COOKIES
    size_t _cookie;
#endif
    size_t _size; // size of the allocation
    size_t _prev; // previous block (not sure what exactly this is)
} zend_mm_block_info;
{% endhighlight %}

As you can see the definitions are cluttered with lots of compile option checks. So if one of those
options is set the allocation header will be bigger and will be largest if you build PHP with heap
protection, multi-threading, debug and MM cookies.

For this example though we will assume that all those options are disabled. In that case the only
thing left are the two `size_t`s `_size` and `_prev`. A `size_t` has 8 bytes (on 64 bit), so the
allocation header has a total size of 16 bytes - and that header is added on *every* allocation.

So now we need to adjust our `zval` size again. In reality it isn't 32 bytes, but it's 48, due to
that allocation header. Multiplied by our 100000 elements that's 4.58 MB. The real value is 13.97
MB, so we already got approximately a third covered.

Buckets
-------

Until now we have only considered single values. But array structures in PHP also take up lots of
space: "Array" actually is a badly chosen term here. PHP arrays are hash tables / dictionaries in
reality. So how do hash tables work? Basically for every key a hash is generated and that hash is
used as an offset into a "real" C array. As the hashes can clash, all elements that have the same
hash are stored in a linked list. When accessing an element PHP first computes the hash, looks for
the right bucket and the traverses the link list, comparing the exact key, element by element. A
bucket is defined as follows (see [`zend_hash.h#54`][6]):

{% highlight c %}
typedef struct bucket {
    ulong h;                  // The hash (or for int keys the key)
    uint nKeyLength;          // The length of the key (for string keys)
    void *pData;              // The actual data
    void *pDataPtr;           // ??? What's this ???
    struct bucket *pListNext; // PHP arrays are ordered. This gives the next element in that order
    struct bucket *pListLast; // and this gives the previous element
    struct bucket *pNext;     // The next element in this (doubly) linked list
    struct bucket *pLast;     // The previous element in this (doubly) linked list
    const char *arKey;        // The key (for string keys)
} Bucket;
{% endhighlight %}

As you can see one needs to store loads of data to get the kind of abstract array data structure
that PHP uses (PHP arrays are arrays, dictionaries and linked lists at the same time, that sure
needs much info). The sizes of the individual components are 8 bytes for the unsigned long, 4 bytes
for the unsigned int and 7 times 8 bytes for the pointers. That's a total of 68. Add alignment and
you get 72 bytes.

Buckets like zvals need to be allocated on the head, so we need to add the 16 bytes for the
allocation header again, giving us 88 bytes. Also we need to store pointers to those buckets in the
"real" C array (`Bucket **arBuckets;`) I mentioned above, which adds another 8 bytes per element.
So all in all every bucket needs 96 bytes of storage.

So if we need a bucket for every value, that's 96 bytes for the bucket and 48 bytes for the zval,
which is 144 bytes in total. For 100000 elements that's 14400000 bytes aka 13.73 MB.

*Mystery solved.*

Wait, there's another 0.24 MB left!
-----------------------------------

Those last 0.24 MB are due to uninitialized buckets: The size of the real C array storing the
buckets should ideally be approximately the same as the number of array elements stored. This way
you have the least number of collisions (unless you want to waste lots of memory.) But PHP obviously
can't reallocate the whole array every time an element is added - that would be *reeeally* slow.
Instead PHP always doubles the size of the internal bucket array if it hits the limit. So the size
of the array is always a power of two.

In our case it is `2^17 = 131072`. But we need only 100000 of those buckets, so we are leaving
31072 buckets unused. Those buckets will not be allocated (so we don't need to spend the full 96
bytes), but the memory for the bucket pointer (the one stored in the internal bucket array) still
needs to be allocated. So we additionally use 8 bytes (a pointer) * 31072 elements. This is
248576 bytes or 0.23 MB. That matches the missing memory. (Sure, there are still a few bytes
missing, but I don't really want to cover there. They are things like the hash table structure
itself, variables, etc.)

Mystery *really* solved.

What does this tell us?
-----------------------

**PHP ain't C**. That's all this should tell us. You can't expect that a super dynamic language like
PHP has the same highly efficient memory usage that C has. You just can't.

But if you do want to save memory you could consider using an [`SplFixedArray`][15] for large,
static arrays.

Have a look a this modified script:

{% highlight php startinline %}
$startMemory = memory_get_usage();
$array = new SplFixedArray(100000);
for ($i = 0; $i < 100000; ++$i) {
    $array[$i] = $i;
}
echo memory_get_usage() - $startMemory, ' bytes';
{% endhighlight %}

It basically does the same thing, but if you run it, you'll notice that it uses "only" 5600640
bytes. That's 56 bytes per element and thus much less than the 144 bytes per element a normal array
uses. This is because a fixed array doesn't need the bucket structure: So it only requires one zval
(48 bytes) and one pointer (8 bytes) for each element, giving us the observed 56 bytes.

  [3]: http://en.wikipedia.org/wiki/Union_%28computer_science%29
  [4]: http://lxr.php.net/opengrok/xref/PHP_5_4/Zend/zend.h#307
  [5]: http://blog.golemon.com/2007/01/youre-being-lied-to.html
  [6]: http://lxr.php.net/opengrok/xref/PHP_5_4/Zend/zend_hash.h#54
  [7]: http://schlueters.de/blog/
  [8]: http://www.tyrael.hu/
  [9]: http://php.net/manual/en/features.gc.collecting-cycles.php
  [10]: http://lxr.php.net/xref/PHP_5_4/Zend/zend.h#318
  [11]: http://lxr.php.net/opengrok/xref/PHP_5_4/Zend/zend_gc.h#zval_gc_info
  [12]: http://php.net/manual/en/internals2.memory.php
  [13]: http://g.oswego.edu/dl/html/malloc.html
  [14]: http://lxr.php.net/xref/PHP_5_4/Zend/zend_alloc.c#336
  [15]: http://php.net/SplFixedArray
  [16]: https://nikic.github.io/2014/12/22/PHPs-new-hashtable-implementation.html