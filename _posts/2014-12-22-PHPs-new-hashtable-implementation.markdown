---
layout: post
title: PHP's new hashtable implementation
excerpt: In this article we'll explore how the new hashtable implementation used by PHP 7 improved memory usage and performance.
---
About three years ago I wrote an article [analyzing the memory usage of arrays][array_size] in PHP 5. As part of the
work on the upcoming PHP 7, large parts of the Zend Engine have been rewritten with a focus on smaller data structures
requiring fewer allocations. In this article I will provide an overview of the new hashtable implementation and show why
it is more efficient than the previous implementation.

To measure memory utilization I am using the following script, which tests the creation of an array with 100000 distinct
integers:

```php?start_inline=1
$startMemory = memory_get_usage();
$array = range(1, 100000);
echo memory_get_usage() - $startMemory, " bytes\n";
```

The following table shows the results using PHP 5.6 and PHP 7 on 32bit and 64bit systems:

            |   32 bit |    64 bit
    ------------------------------
    PHP 5.6 | 7.37 MiB | 13.97 MiB
    ------------------------------
    PHP 7.0 | 3.00 MiB |  4.00 MiB

In other words, arrays in PHP 7 use about 2.5 times less memory on 32bit and 3.5 on 64bit (LP64), which is quite
impressive.

Introduction to hashtables
--------------------------

In essence PHP's arrays are ordered dictionaries, i.e. they represent an ordered list of key/value pairs, where the
key/value mapping is implemented using a hashtable.

A [Hashtable][wiki_hashtable] is an ubiquitous data structure, which essentially solves the problem that computers can
only directly represent continuous integer-indexed arrays, whereas programmers often want to use strings or other
complex types as keys.

The concept behind a hashtable is very simple: The string key is run through a hashing function, which returns an
integer. This integer is then used as an index into a "normal" array. The problem is that two different strings can
result in the same hash, as the number of possible strings is virtually infinite while the hash is limited by the
integer size. As such hashtables need to implement some kind of collision resolution mechanism.

There are two primary approaches to collision resolution: [Open addressing][wiki_open_addressing], where elements will
be stored at a different index if a collision occurs, and chaining, where all elements hashing to the same index are
stored in a linked list. PHP uses the latter mechanism.

Typically hashtables are not explicitly ordered: The order in which elements are stored in the underlying array depends
on the hashing function and will be fairly random. But this behavior is not consistent with the semantics of PHP arrays:
If you iterate over a PHP array you will get back the elements in the exact order in which they were inserted. This
means that PHP's hashtable implementation has to support an additional mechanism for remembering the order of array
elements.

The old hashtable implementation
--------------------------------

I'll only provide a short overview of the old hashtable implementation here, for a more comprehensive explanation please
see the [hashtable chapter][pib_hashtable] of the PHP Internals Book. The following graphic is a very high-level view of
how a PHP 5 hashtable looks like:

<center><img src="/images/basic_hashtable.svg" height="265px"></center>

The elements in the "collision resolution" chain are referred to as "buckets". Every bucket is individually allocated.
What the image glosses over are the actual values stored in these buckets (only the keys are shown here). Values are
stored in separately allocated zval structures, which are 16 bytes (32bit) or 24 bytes (64bit) large.

Another thing the image does not show is that the collision resolution list is actually a doubly linked list (which
simplifies deletion of elements). Next to the collision resolution list, there is another doubly linked list storing the
order of the array elements. For an array containing the keys `"a", "b", "c"` in this order, this list could look as
follows:

<center><img src="/images/ordered_hashtable.svg" height="255px"></center>

So why was the old hashtable structure so inefficient, both in terms of memory usage and performance? There are a number
of primary factors:

 * Buckets require separate allocations. Allocations are slow and additionally require 8 / 16 bytes of allocation
   overhead. Separate allocations also means that the buckets will be more spread out in memory and as such reduce cache
   efficiency.
 * Zvals also require separate allocations. Again this is slow and incurs allocation header overhead. Furthermore this
   requires us to store a pointer to a zval in each bucket. Because the old implementation was overly generic it
   actually needed not just one, but two pointers for this.
 * The two doubly linked lists require a total of four pointers per bucket. This alone takes up 16 / 32 bytes..
   Furthermore traversing linked lists is a very cache-unfriendly operation.

The new hashtable implementation tries to solve (or at least ameliorate) all of these problems.

The new zval implementation
---------------------------

Before getting to the actual hashtable, I'd like to take a quick look at the new zval structure and highlight how it
differs from the old one. The `zval` struct is defined as follows:

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
		uint32_t next;       /* hash collision chain */
		uint32_t cache_slot; /* literal cache slot */
		uint32_t lineno;     /* line number (for ast nodes) */
	} u2;
};
{% endhighlight %}

You can safely ignore the `ZEND_ENDIAN_LOHI_4` macro in this definition - it is only present to ensure a predictable
memory layout across machines with different endianness.

The zval structure has three parts: The first member is the `value`. The `zend_value` union is 8 bytes large and can
store different kinds of values, including integers, strings, arrays, etc. What is actually stored in there depend on
the zval type.

The second part is the 4 byte `type_info`, which consists of the actual `type` (like `IS_STRING` or `IS_ARRAY`), as well
as a number of additional flags providing information about this type. E.g. if the zval is storing an object, then the
type flags would say that it is a non-constant, refcounted, garbage-collectible, non-copying type.

The last 4 bytes of the zval structure are normally unused (it's really just explicit padding, which the compiler would
introduce automatically otherwise). However in special contexts this space is used to store some extra information. E.g.
AST nodes use it to store a line number, VM constants use it to store a cache slot index and hashtables use it to store
the next element in the collision resolution chain - that last part will be important to us.

If you compare this to the previous zval implementation, one difference particularly stands out: The new zval structure
no longer stores a refcount. The reason behind this, is that the zvals themselves are no longer individually allocated.
Instead the zval is directly embedded into whatever is storing it (e.g. a hashtable bucket).

While the zvals themselves no longer use refcounting, complex data types like strings, arrays, objects and resources
still use them. Effectively the new zval design has pushed out the refcount (and information for the cycle-collector)
from the zval to the array/object/etc. There are a number of advantages to this approach, some of them listed in the
following:

 * Zvals storing simple values (like booleans, integers or floats) no longer require any allocations. So this saves
   the allocation header overhead and improves performance by avoiding unnecessary allocs and frees and improving
   cache locality.
 * Zvals storing simple values don't need to store a refcount and GC root buffer.
 * We avoid double refcounting. E.g. previously objects both used the zval refcount and an additional object refcount,
   which was necessary to support by-object passing semantics.
 * As all complex values now embed a refcount, they can be shared independently of the zval mechanism. In particular it
   is now also possible to share strings. This is important to the hashtable implementation, as it no longer needs to
   copy non-interned string keys.

The new hashtable implementation
--------------------------------

With all the preliminaries behind us, we can finally look at the new hashtable implementation used by PHP 7. Lets start
by looking at the bucket structure:

{% highlight c %}
typedef struct _Bucket {
	zend_ulong        h;
	zend_string      *key;
	zval              val;
} Bucket;
{% endhighlight %}

A bucket is an entry in the hashtable. It contains pretty much what you would expect: A hash `h`, a string key `key` and
a zval value `val`. Integer keys are stored in `h` (the key and hash are identical in this case), in which case the
`key` member will be `NULL`.

As you can see the zval is directly embedded in the bucket structure, so it doesn't have to be allocated separately and
we don't have to pay for allocation overhead.

The main hashtable structure is more interesting:

{% highlight c %}
typedef struct _HashTable {
	uint32_t          nTableSize;
	uint32_t          nTableMask;
	uint32_t          nNumUsed;
	uint32_t          nNumOfElements;
	zend_long         nNextFreeElement;
	Bucket           *arData;
	uint32_t         *arHash;
	dtor_func_t       pDestructor;
	uint32_t          nInternalPointer;
	union {
		struct {
			ZEND_ENDIAN_LOHI_3(
				zend_uchar    flags,
				zend_uchar    nApplyCount,
				uint16_t      reserve)
		} v;
		uint32_t flags;
	} u;
} HashTable;
{% endhighlight %}

The buckets (= array elements) are stored in the `arData` array. This array is allocated in powers of two, with the
size being stored in `nTableSize` (the minimum value is 8). The actual number of stored elements is `nNumOfElements`.
Note that this array directly contains the `Bucket` structures. Previously we used an array of pointers to separately
allocated buckets, which means that we needed more alloc/frees, had to pay allocation overhead and also had to pay for
the extra pointer.

Order of elements
-----------------

The `arData` array stores the elements in order of insertion. So the first array element will be stored in `arData[0]`,
the second in `arData[1]` etc. This does not in any way depend on the used key, only the order of insertion matters
here.

So if you store five elements in the hashtable, slots `arData[0]` to `arData[4]` will be used and the next free slot is
`arData[5]`. We remember this number in `nNumUsed`. You may wonder: Why do we store this separately, isn't it the same
as `nNumOfElements`?

It is, but only as long as only insertion operations are performed. If an element is deleted from a hashtable, we
obviously don't want to move all elements in `arData` that occur after the deleted element in order to have a continuous
array again. Instead we simply mark the deleted value with an `IS_UNDEF` zval type.

As an example, consider the following code:

```php?start_inline=1
$array = [
	'foo' => 0,
	'bar' => 1,
	0     => 2,
	'xyz' => 3,
	2     => 4
];
unset($array[0]);
unset($array['xyz']);
```

This will result in the following `arData` structure:

	nTableSize     = 8
	nNumOfElements = 3
	nNumUsed       = 5

	[0]: key="foo", val=int(0)
	[1]: key="bar", val=int(1)
	[2]: val=UNDEF
	[3]: val=UNDEF
	[4]: h=2, val=int(4)
	[5]: NOT INITIALIZED
	[6]: NOT INITIALIZED
	[7]: NOT INITIALIZED

As you can see the first five `arData` elements have been used, but elements at position 2 (key `0`) and 3 (key `'xyz'`)
have been replaced with an `IS_UNDEF` tombstone, because they were `unset`. These elements will just remain wasted
memory for now. However, once `nNumUsed` reaches `nTableSize` PHP will try compact the `arData` array, by dropping any
`UNDEF` entries that have been added along the way. Only if all buckets really contain a value the `arData` will be
reallocated to twice the size.

The new way of maintaining array order has several advantages over the doubly linked list that was used in PHP 5.x. One
obvious advantage is that we save two pointers per bucket, which corresponds to 8/16 bytes. Additionally it means that
iterating an array looks roughly as follows:

{% highlight c %}
uint32_t i;
for (i = 0; i < ht->nNumUsed; ++i) {
	Bucket *b = &ht->arData[i];
	if (Z_ISUNDEF(b->val)) continue;

	// do stuff with bucket
}
{% endhighlight %}

This corresponds to a linear scan of memory, which is much more cache-efficient than a linked list traversal (where you
go back and forth between relatively random memory addresses).

One problem with the current implementation is that `arData` never shrinks (unless explicitly told to). So if you
create an array with a few million elements and remove them afterwards, the array will still take a lot of memory. We
should probably half the `arData` size if utilization falls below a certain level.

Hashtable lookup
----------------

Until now we have only discussed how PHP arrays represent order. The actual hashtable lookup uses the second `arHash`
array, which consists of `uint32_t` values. The `arHash` array has the same size (`nTableSize`) as `arData` and both
are actually allocated as one chunk of memory.

The hash returned from the hashing function (DJBX33A for string keys) is a 32-bit or 64-bit unsigned integer, which is
too large to directly use as an index into the hash array. We first need to adjust it to the table size using a modulus
operation. Instead of `hash % ht->nTableSize` we use `hash & (ht->nTableSize - 1)`, which is the same if the size is a
power of two, but doesn't require expensive integer division. The value `ht->nTableSize - 1` is stored in
`ht->nTableMask`.

Next, we look up the index `idx = ht->arHash[hash & ht->nTableMask]` in the hash array. This index corresponds to the
head of the collision resolution list. So `ht->arData[idx]` is the first entry we have to examine. If the key stored
there matches the one we're looking for, we're done.

Otherwise we must continue to the next element in the collision resolution list. The index to this element is stored in
`bucket->val.u2.next`, which are the normally unused last four bytes of the `zval` structure that get a special meaning
in this context. We continue traversing this linked list (which uses indexes instead of pointers) until we either find
the right bucket or hit an `INVALID_IDX` - which means that an element with the given key does not exist.

In code, the lookup mechanism looks like this:

{% highlight c %}
zend_ulong h = zend_string_hash_val(key);
uint32_t idx = ht->arHash[h & ht->nTableMask];
while (idx != INVALID_IDX) {
	Bucket *b = &ht->arData[idx];
	if (b->h == h && zend_string_equals(b->key, key)) {
		return b;
	}
	idx = Z_NEXT(b->val); // b->val.u2.next
}
return NULL;
{% endhighlight %}

Lets consider how this approach improves over the previous implementation: In PHP 5.x the collision resolution used a
doubly linked pointer list. Using `uint32_t` indices instead of pointers is better, because they take half the size on
64bit systems. Additionally fitting in 4 bytes means that we can embed the "next" link into the unused zval slot, so we
essentially get it for free.

We also use a singly linked list now, there is no "prev" link anymore. The prev link is primarily useful for deleting
elements, because you have to adjust the "next" link of the "prev" element when you perform a deletion. However, if the
deletion happens by key, you already know the previous element as a result of traversing the collision resolution list.

The few cases where deletion occurs in some other context (e.g. "delete the element the iterator is currently at") will
have to traverse the collision list to find the previous element. But as this is a rather unimportant scenario, we
prefer saving memory over saving a list traversal for that case.

Packed hashtables
-----------------

PHP uses hashtables for all arrays. However in the rather common case of continuous, integer-indexed arrays (i.e. *real*
arrays) the whole hashing thing doesn't make much sense. This is why PHP 7 introduces the concept of "packed
hashtables".

In packed hashtables the `arHash` array is `NULL` and lookups will directly index into `arData`. If you're looking for
the key `5` then the element will be located at `arData[5]` or it doesn't exist at all. There is no need to traverse a
collision resolution list.

Note that even for integer indexed arrays PHP has to maintain order. The arrays `[0 => 1, 1 => 2]` and
`[1 => 2, 0 => 1]` are not the same. The packed hashtable optimization only works if keys are in ascending order. There
can be gaps in between them (the keys don't have to be continuous), but they need to always increase. So if elements are
inserted into an array in a "wrong" order (e.g. in reverse) the packed hashtable optimization will not be used.

Note furthermore that packed hashtables still store a lot of useless information. For example we can determine the index
of a bucket based on its memory address, so `bucket->h` is redundant. The value `bucket->key` will always be `NULL`, so
it's just wasted memory as well.

We keep these useless values around so that buckets always have the same structure, independently of whether or not
packing is used. This means that iteration can always use the same code. However we might switch to a "fully packed"
structure in the future, where a pure zval array is used if possible.

Empty hashtables
----------------

Empty hashtables get a bit of special treating both in PHP 5.x and PHP 7. If you create an empty array `[]` chances are
pretty good that you won't actually insert any elements into it. As such the `arData`/`arHash` arrays will only be
allocated when the first element is inserted into the hashtable.

To avoid checking for this special case in many places, a small trick is used: While the `nTableSize` is set to either
the hinted size or the default value of 8, the `nTableMask` (which is usually `nTableSize - 1`) is set to zero. This
means that `hash & ht->nTableMask` will always result in the value zero as well.

So the `arHash` array for this case only needs to have one element (with index zero) that contains an `INVALID_IDX`
value (this special array is called `uninitialized_bucket` and is allocated statically). When a lookup is performed,
we always find the `INVALID_IDX` value, which means that the key has not been found (which is exactly what you want for
an empty table).

Memory utilization
------------------

This should cover the most important aspects of the PHP 7 hashtable implementation. First lets summarize why the new
implementation uses less memory. I'll only use the numbers for 64bit systems here and only look at the per-element size,
ignoring the main `HashTable` structure (which is less significant asymptotically).

In PHP 5.x a whopping 144 bytes per element were required. In PHP 7 the value is down to 36 bytes, or 32 bytes for the
packed case. Here's where the difference comes from:

 * Zvals are not individually allocated, so we save 16 bytes allocation overhead.
 * Buckets are not individually allocated, so we save another 16 bytes of allocation overhead.
 * Zvals are 16 bytes smaller for simple values.
 * Keeping order no longer needs 16 bytes for a doubly linked list, instead the order is implicit.
 * The collision list is now singly linked, which saves 8 bytes. Furthermore it's now an index list and the index is
   embedded into the zval, so effectively we save another 8 bytes.
 * As the zval is embedded into the bucket, we no longer need to store a pointer to it. Due to details of the previous
   implementation we actually save two pointers, so that's another 16 bytes.
 * The length of the key is no longer stored in the bucket, which is another 8 bytes. However, if the key is actually a
   string and not an integer, the length still has to be stored in the `zend_string` structure. The exact memory
   impact in this case is hard to quantify, because `zend_string` structures are shared, whereas previously hashtables
   had to copy the string if it wasn't interned.
 * The array containing the collision list heads is now index based, so saves 4 bytes per element. For packed arrays it
   is not necessary at all, in which case we save another 4 bytes.

However it should be clearly said that this summary is making things look better than they really are in several
respects. First of all, the new hashtable implementation uses a lot more embedded (as opposed to separately allocated)
structures. How can this negatively affect things?

If you look at the actually measured numbers at the start of this article, you'll find that on 64bit PHP 7 an array with
100000 elements took 4.00 MiB of memory. In this case we're dealing with a packed array, so we would actually expect
32 * 100000 = 3.05 MiB memory utilization. The reason behind this is that we allocate everything in powers of two. The
`nTableSize` will be 2^17 = 131072 in this case, so we'll allocate 32 * 131072 bytes of memory (which is 4.00 MiB).

Of course the previous hashtable implementation also used power of two allocations. However it only allocated an array
with bucket pointers in this way (where each pointer is 8 bytes). Everything else was allocated on demand. So in PHP 7
we loose 32 * 31072 (0.95 MiB) in unused memory, while in PHP 5.x we only waste 8 * 31072 (0.24 MiB).

Another thing to consider is what happens if not all values stored in the array are distinct. For simplicity lets assume
that all values in the array are identical. So lets replace the `range` in the starting example with an `array_fill`:

```php?start_inline=1
$startMemory = memory_get_usage();
$array = array_fill(0, 100000, 42);
echo memory_get_usage() - $startMemory, " bytes\n";
```

This script results in the following numbers:

            |   32 bit |    64 bit
    ------------------------------
    PHP 5.6 | 4.70 MiB |  9.39 MiB
    ------------------------------
    PHP 7.0 | 3.00 MiB |  4.00 MiB

As you can see the memory usage on PHP 7 stays the same as in the `range` case. There is no reason why it would change,
as all zvals are separate. On PHP 5.x on the other hand the memory usage is now significantly lower, because only one
zval is used for all values. So while we're still a good bit better off on PHP 7, the difference is smaller now.

Things become even more complicated once we consider string keys (which may or not be shared or interned) and complex
values. The point being that arrays in PHP 7 will take significantly less memory than in PHP 5.x, but the numbers from
the introduction are likely too optimistic in many cases.

Performance
-----------

I've already talked a lot about memory usage, so lets move to the next point, namely performance. In the end, the goal
of the phpng project wasn't to improve memory usage, but to improve performance. The memory utilization improvement is
only a means to an end, in that less memory results in better CPU cache utilization, resulting in better performance.

However there are of course a number of other reasons why the new implementation is faster: First of all we need less
allocations. Depending on whether or not values are shared we save two allocations per element. Allocations being rather
expensive operations this is quite significant.

Array iteration in particular is now more cache-friendly, because it's now a linear memory traversal, instead of a
random-access linked list traversal.

There's probably a lot more to be said on the topic of performance, but the main interest in this article was memory
usage, so I won't go into further detail here.

Closing thoughts
----------------

PHP 7 undoubtedly has made a big step forward as far as the hashtable implementation is concerned. A lot of useless
overhead is gone now.

So the question is: where we can go from here? One idea I already mentioned is to use "fully packed" hashes for the
case of increasing integer keys. This would mean using a plain zval array, which is the best we can do without starting
to specialize uniformly typed arrays.

There's probably some other directions one could go as well. For example switching from collision-chaining to open
addressing (e.g. using Robin Hood probing), could be better both in terms of memory usage (no collision resolution list)
and performance (better cache efficiency, depending on the details of the probing algorithm). However open-addressing is
relatively hard to combine with the ordering requirement, so this may not be possible to do in a reasonable way.

Another idea is to combine the `h` and `key` fields in the bucket structure. Integer keys only use `h` and string keys
already store the hash in `key` as well. However this would likely have an adverse impact on performance, because
fetching the hash will require an additional memory indirection.

One last thing that I wish to mention is that PHP 7 improved not only the internal representation of hashtables, but
also the API used to work them. I've regularly had to look up how even simple operations like `zend_hash_find` had to be
used, especially regarding how many levels of indirection are required (hint: three). In PHP 7 you just write
`zend_hash_find(ht, key)` and get back a `zval*`. Generally I find that writing extensions for PHP 7 has become quite a
bit more pleasant.

Hopefully I was able to provide you some insight into the internals of PHP 7 hashtables. Maybe I'll write a followup
article focusing on zvals. I've already touched on some of the difference in this post, but there's a lot more to be
said on the topic.


  [array_size]: {{site.url}}/2011/12/12/How-big-are-PHP-arrays-really-Hint-BIG.html
  [wiki_hashtable]: https://en.wikipedia.org/wiki/Hash_table
  [wiki_open_addressing]: https://en.wikipedia.org/wiki/Open_addressing
  [pib_hashtable]: http://www.phpinternalsbook.com/hashtables/basic_structure.html
