---
layout: post
title: Understanding PHP's internal array implementation (PHP's Source Code for PHP Developers - Part 4)
excerpt: The fourth part of the "PHP's Source Code for PHP Developers" series, covering how arrays are internally implemented in PHP and how they are used in the source code.
---
Welcome back to the fourth part of the "PHP's Source Code for PHP Developers" series, in which we'll cover how PHP
arrays are internally represented and used throughout the code base.

In case you missed them, here are the previous parts of this series:

 * [Part 1: Structure of the source code and introduction to C][1]
 * [Part 2: Finding and understanding PHP's internal function definitions][2]
 * [Part 3: PHP's internal value representation: The zval][3]

Everything is a hash table!
---------------------------

Basically, everything in PHP is a hash table. Not only are hash tables used in the underlying implementation of PHP
arrays, they are also used to store object properties and methods, functions, variables and pretty much everything else.

And because the hash table is so fundamental to PHP, it is worth having a deeper look into how it works.

So, what is a hash table?
-------------------------

Remember that in C arrays are basically chunks of memory, which you can access by index. Thus arrays in C only have
integer keys and have to be continuous (i.e. you can't have a key `0` and the next key is `1332423442`). There is no
such thing as an associative array.

And this is where hash tables come in: They convert string keys into normal integer keys using a hash function. The
result can then be used as an index into a normal C array (aka chunk of memory). The problem here obviously is that the
hash function can have collisions, i.e. multiple string keys can yield the same hash. For example in a PHP array with up
to 64 elements the strings `"foo"` and `"oof"` would have the same hash.

This problem is solved by not storing the value directly at the generated index, but storing a linked list of *possible*
values instead.

HashTable and Bucket
--------------------

So, after the basic concept of hash tables is clear, let's have a look at the structures actually used in PHP's hash
table implementation:

The first one is the [`HashTable`][5]:

    typedef struct _hashtable {
        uint nTableSize;
        uint nTableMask;
        uint nNumOfElements;
        ulong nNextFreeElement;
        Bucket *pInternalPointer;
        Bucket *pListHead;
        Bucket *pListTail;
        Bucket **arBuckets;
        dtor_func_t pDestructor;
        zend_bool persistent;
        unsigned char nApplyCount;
        zend_bool bApplyProtection;
    #if ZEND_DEBUG
        int inconsistent;
    #endif
    } HashTable;

Let's quickly go through it:

 * `nNumOfElements` specifies how many values are currently stored in the array. This is also the number that
   `count($array)` returns.

 * `nTableSize` specifies the size of the internal C array. It is always the next power of 2 greater or equal to
   `nNumOfElements`.  E.g. if an array stores 32 elements, the internal C array also has a size of 32. But if one more
   element is added, i.e. the array then contains 33 elements, the internal C array is resized to 64 elements.

   This is done to always keep the hash table efficient in space and time. It is clear that if the internal array is too
   small there will be many collisions and the performance will degrade. If the internal array is too big on the other
   hand, we'd be wasting memory. The power-of-2 size is a good compromise.

 * `nTableMask` is the table size minus one. This mask is used to adjust the generated hashes for the current table
   size. For example the actual hash for `"foo"` (through the [DJBX33A hashing function][4]) is 193491849. If we
   currently have a table size of 64, we obviously can't use that as an index into the array. Instead we only take the
   lower bits of the hash by applying the table mask:

          hash  |   193491849 |   0b1011100010000111001110001001
        & mask  | &        63 | & 0b0000000000000000000000111111
       ---------------------------------------------------------
        = index | =         9 | = 0b0000000000000000000000001001

 * `nNextFreeElement` is the next free integer key, which is used when you append to an array using `$array[] = xyz`.

 * `pInternalPointer` stores the current position in the array. This is used for `foreach` iteration and can be accessed
   using the `reset()`, `current()`, `key()`, `next()`, `prev()` and `end()` functions.

 * `pListHead` and `pListTail` specify the first and last element of the array. Remember: PHP arrays have an order. E.g.
   `['foo' => 'bar', 'bar' => 'foo']` and `['bar' => 'foo', 'foo' => 'bar']` contain the same elements, but have a
   different order.

 * `arBuckets` is the "internal C array" we were always talking about. It's defined as a `Bucket **`, so it can be seen
   as an array of bucket pointers (we'll get to what exactly a Bucket is in a minute).

 * `pDestructor` is the destructor for the values. If a value is removed from the HT this function will be called on it.
   For a normal array the destructor function is `zval_ptr_dtor`. `zval_ptr_dtor` will reduce the reference count of
   the `zval` and, if it reaches 0, destroy and free it.

The last four properties aren't really of interest to us. So let's just say that `persistent` specifies that the hash
table can live between multiple requests, `nApplyCount` and `bApplyProtection` are used to prevent infinite recursion in
some places and `inconsistent` is used to catch incorrect uses of hash tables in debug mode.

Let's move on to the second important structure: [`Bucket`][6]:

    typedef struct bucket {
        ulong h;
        uint nKeyLength;
        void *pData;
        void *pDataPtr;
        struct bucket *pListNext;
        struct bucket *pListLast;
        struct bucket *pNext;
        struct bucket *pLast;
        const char *arKey;
    } Bucket;

 * `h` is the hash (without the table mask applied).

 * `arKey` is used to save string keys. `nKeyLength` is the corresponding length. For integer keys those two aren't
   used.

 * `pData` or `pDataPtr` is used to store the actual value. For PHP arrays that value is a `zval` (but it's also used
   for other things internally.) Don't bother with the fact that there are two properties for this. The difference
   between them is who is responsible for freeing the value.

 * `pListNext` and `pListLast` specify the order of the array elements. If PHP wants to traverse the array it starts off
   at the `pListHead` bucket (specified in the `HashTable` struct) and then always takes the `pListNext` bucket. The
   same works in reverse, by starting at `pListTail` and always following `pListLast`. (You can do this in userland by
   calling `end()` and then always calling `prev()`.)

 * `pNext` and `pLast` form the "linked list of possible values" I mentioned above. The `arBuckets` array stores a
   pointer to the first possible bucket. If that bucket hasn't the right key, PHP will look at the bucket which `pNext`
   points to. This is done until the right bucket is found. `pLast` can be used to do the same in reverse.

As you can see PHP's hash table implementation is fairly complex. This is the price one has to pay for its
ultra-flexible array type.

How are hash tables used?
-------------------------

The Zend Engine defines a large number of API functions for the work with hash tables. An overview of low-level hash
table functions can be found in [`zend_hash.h`][8]. Additionally the ZE defines a set of slightly higher level APIs in
[`zend_API.h`][9].

We don't have time to go through all of these, but we can look at a sample function to see at least some of them in
action. We'll use [`array_fill_keys`][10] as that sample function.

Using the technique outlined in the [second part][11] you should be able to find the function definition in
[`ext/standard/array.c`][7] easily. Let's quickly walk through it now.

As always there's a set of variable declarations and a `zend_parse_parameters` call at the top:

    zval *keys, *val, **entry;
    HashPosition pos;

    if (zend_parse_parameters(ZEND_NUM_ARGS() TSRMLS_CC, "az", &keys, &val) == FAILURE) {
        return;
    }

The `az` obviously means that the first parameter is an **a**rray (fetched into the `keys` variable) and the second one
is an arbitrary **z**val (fetched into the `val` variable).

After the parameters are parsed the array to be returned is initialized:

    /* Initialize return array */
    array_init_size(return_value, zend_hash_num_elements(Z_ARRVAL_P(keys)));

This line contains already three important parts of the array API:

 1. The `Z_ARRVAL_P` macro fetches the hash table from a `zval`.
 2. `zend_hash_num_elements` fetches the number of elements in a hash table (the `nNumOfElements` property).
 3. `array_init_size` initializes an array with a size hint.

So this line initializes an array into `return_value` with the same size as the `keys` array.

The size hint here is just an optimization. The function could have also called just `array_init(return_value)`, in
which case PHP would have to do multiple resizes as more and more elements are added to the array. By specifying an
explicit size, PHP allocates the right amount of memory from the start.

After the return array initialization the function loops through the `keys` array using a `while` loop with roughly this
structure:

    zend_hash_internal_pointer_reset_ex(Z_ARRVAL_P(keys), &pos);
    while (zend_hash_get_current_data_ex(Z_ARRVAL_P(keys), (void **)&entry, &pos) == SUCCESS) {
        // some code

        zend_hash_move_forward_ex(Z_ARRVAL_P(keys), &pos);
    }

This can be translated to PHP code easily:

    reset($keys);
    while (null !== $entry = current($keys)) {
        // some code

        next($keys);
    }

Which is the same as:

    foreach ($keys as $entry) {
        // some code
    }

The only real difference is that the C iteration doesn't use the internal array pointer, but uses it's own `pos`
variable to store the current position.

The code within the loop has two branches: One for integer keys and one for other keys. The integer key branch contains
only two lines:

    zval_add_ref(&val);
    zend_hash_index_update(Z_ARRVAL_P(return_value), Z_LVAL_PP(entry), &val, sizeof(zval *), NULL);

This is pretty straightforward: First the refcount of the value is increased (adding the value to the hash table means
adding another reference to it) and then the value is actually inserted into the hash table. The arguments of the
`zend_hash_index_update` macro are the hash table to update `Z_ARRVAL_P(return_value)`, the integer index
`Z_LVAL_PP(entry)`, the value `&val`, the size of the value `sizeof(zval *)` and the destination pointer (which we don't
care about, thus `NULL`).

The non-integer-key branch is slightly more complicated:

    zval key, *key_ptr = *entry;

    if (Z_TYPE_PP(entry) != IS_STRING) {
        key = **entry;
        zval_copy_ctor(&key);
        convert_to_string(&key);
        key_ptr = &key;
    }

    zval_add_ref(&val);
    zend_symtable_update(Z_ARRVAL_P(return_value), Z_STRVAL_P(key_ptr), Z_STRLEN_P(key_ptr) + 1, &val, sizeof(zval *), NULL);

    if (key_ptr != *entry) {
        zval_dtor(&key);
    }

First the key is converted to a string (unless it already is one) using `convert_to_string`. But before this can be done
the `entry` has to be copied into a new `key` variable. The `key = **entry` line takes care of that. Additionally
`zval_copy_ctor` has to be called, otherwise complex structures (like strings or arrays) won't be copied correctly.

The copy is necessary to ensure that the cast doesn't change the original array. Without the copy the cast would
modify not only our local variable, but also the element in the `keys` array (which obviously would be quite unexpected
to the user).

Obviously the copy has to be removed again after the loop, which is what the `zval_dtor(&key)` line does. The difference
between `zval_ptr_dtor` and `zval_dtor` is that `zval_ptr_dtor` only destroys the zval if the refcount reaches 0,
whereas `zval_dtor` destroys it always, regardless of the refcount. That's why you'll find `zval_ptr_dtor` used with
"normal" values and `zval_dtor` with temporary variables, which aren't used anywhere else anyways. Also `zval_ptr_dtor`
frees the zval after destroying it, while `zval_dtor` does not. As we never `malloc()`d anything, we also don't have
to `free()`, so `zval_dtor` is the right choice in this respect too.

Now, let's look at the last two lines left (the important ones ^^):

    zval_add_ref(&val);
    zend_symtable_update(Z_ARRVAL_P(return_value), Z_STRVAL_P(key_ptr), Z_STRLEN_P(key_ptr) + 1, &val, sizeof(zval *), NULL);

Those are very similar to what was done in the integer-key branch. The difference is that now `zend_symtable_update` is
called instead of `zend_hash_index_update` and the string key and it's length are passed in.

The Symtable
------------

The "normal" function for inserting string keys into a hash table is `zend_hash_update`, but here `zend_symtable_update`
is used instead. What's the difference?

A symtable basically is a special type of hash table, which is used for arrays. The difference to the ordinary hash
table is how it handles numeric string keys: In a symtable the keys `"123"` and `123` are considered identical. So if
you store a value in `$array["123"]`, you can then retrieve it using `$array[123]`.

The underlying implementation could use two ways: Either save both `123` and `"123"` using the `"123"` key or save both
using the `123` key. PHP obviously chooses the latter option (as integers are smaller and faster than strings).

You can have a little bit of fun with symtables, if you manage to somehow insert the `"123"` key without it being cast
to `123`. One way is to exploit the array to object cast:

```php?start_inline=1
$obj = new stdClass;
$obj->{123} = "foo";
$arr = (array) $obj;
var_dump($arr[123]);   // Undefined offset: 123
var_dump($arr["123"]); // Undefined offset: 123
```

Object properties are always saved under string keys, even if they are numbers. So the `$obj->{123} = 'foo'` line
actually saves `"foo"` under the `"123"` index, not the `123` index. When doing the array cast this is not changed. But
as both `$arr[123]` and `$arr["123"]` try to access the `123` index (not the `"123"` index that actually exists), both
throw an error. So, congratulations, you've created a hidden array element!

In the next part
----------------

The next part will again be published on [ircmaxell's blog][12]. It will analyze how objects and classes work
internally.

 [1]: https://blog.ircmaxell.com/2012/03/phps-source-code-for-php-developers.html
 [2]: https://nikic.github.com/2012/03/16/Understanding-PHPs-internal-function-definitions.html
 [3]: https://blog.ircmaxell.com/2012/03/phps-source-code-for-php-developers_21.html
 [4]: https://lxr.room11.org/xref/php-src@5.6/Zend/zend_hash.h#269
 [5]: https://lxr.room11.org/xref/php-src@5.6/Zend/zend_hash.h#67
 [6]: https://lxr.room11.org/xref/php-src@5.6/Zend/zend_hash.h#55
 [7]: https://lxr.room11.org/xref/php-src@5.6/ext/standard/array.c#1551
 [8]: https://lxr.room11.org/xref/php-src@5.6/Zend/zend_hash.h#98
 [9]: https://lxr.room11.org/xref/php-src@5.6/Zend/zend_API.h#364
 [10]: https://php.net/array_fill_keys
 [11]: https://nikic.github.com/2012/03/16/Understanding-PHPs-internal-function-definitions.html
 [12]: https://blog.ircmaxell.com/
