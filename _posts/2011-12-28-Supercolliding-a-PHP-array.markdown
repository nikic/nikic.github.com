---
layout: post
title: Supercolliding a PHP array
excerpt: Inserting 65536 specially crafted values into a PHP array can take 30 seconds, whereas normally it would only take 0.01 seconds.
---
Did you know that inserting `2^16 = 65536` specially crafted values into a normal PHP array can take
**30 seconds**? Normally this would take only *0.01 seconds*.

This is the code to reproduce it:

{% highlight php %}
<?php echo '<pre>';

$size = pow(2, 16); // 16 is just an example, could also be 15 or 17

$startTime = microtime(true);

$array = array();
for ($key = 0, $maxKey = ($size - 1) * $size; $key <= $maxKey; $key += $size) {
    $array[$key] = 0;
}

$endTime = microtime(true);

echo 'Inserting ', $size, ' evil elements took ', $endTime - $startTime, ' seconds', "\n";

$startTime = microtime(true);

$array = array();
for ($key = 0, $maxKey = $size - 1; $key <= $maxKey; ++$key) {
    $array[$key] = 0;
}

$endTime = microtime(true);

echo 'Inserting ', $size, ' good elements took ', $endTime - $startTime, ' seconds', "\n";
{% endhighlight %}

Try it yourself and wonder (I intentionally don't provide an online demo at the amazing Viper-7
codepad this time so that it doesn't get overloaded.) You may need to adjust the number `16` in the
`$size = pow(2, 16);` line based on what hardware you have. I would start with `14` and increase it
one at a time.

How does this work?
-------------------

PHP internally uses hashtables to store arrays. The above creates a hashtable with 100% collisions
(i.e. all keys will have the same hash).

For those who don't know how hashtables work: When you write `$array[$key]` in PHP the `$key` is run
through a fast hash function that yield an integer. This integer is then used as an offset into a
"real" C array (here "array" means "chunk of memory").

Because every hash function has collisions this C array doesn't actually store the value we want,
but a linked list of possible values. PHP then walks these values one by one and does a full key
comparison until it finds the right element with our `$key`.

Normally there will be only a small number of collisions, so in most cases the linked list will only
have one value.

But the above script creates a hash where *all* elements collide.

How can you make all of them collide?
-------------------------------------

In PHP it's very simple. If the key is an integer the hash function is just a no-op: The hash is the
integer itself. All PHP does is apply a table mask on top of it: `hash & tableMask`.

This table mask ensures that the resulting hash is within the bounds of the underlying real C
array. This underlying C array has always a size which is a power of 2 (this way the hashtable is
always reasonably efficient in both space and time). So if you store 10 elements the real size will
be 16. If you store 33 it will be 64. If you store 63 it will also be 64. The table mask is the size
minus one. So if the size is 64, i.e. `1000000` in binary the table mask will be 63, i.e. `0111111`
in binary.

So basically the table mask removes all bits that are greater than the hashtable size. And this is
what we are exploiting: `0 & 0111111 = 0`, but `1000000 & 0111111 = 0` too, so is
`10000000 & 0111111 = 0` and `11000000 & 0111111 = 0`. As long as we keep those lower bits the same
the result of the hash will always be the same.

So if we insert a total of 64 elements, the first one 0, the second one 64, the third one 128, the
fourth one 192, etc., all of those elements will have the same hash (namely 0) and all will be put
into the same linked list. And that's what the code does. Only not with 64 elements, but with a few
more.

Why is that so abysmally slow?
------------------------------

Well, for every insertion PHP has to traverse the whole linked list, element for element. On the
first insertion it need to traverse 0 element (there is nothing there yet). On the second one it
traverses 1 element. On the third one 2, on the fourth 3 and on the 64th one 63. Those who know
a little bit of math probably know that `0+1+2+3+...+(n-1) = (n-1)*(n-2)/2`. So the number of
elements to traverse is quadratic. For 64 elements it's `62*62/2 = 1953` traversations. For
`2^16 = 65536` it's `65534*65535/2=2147385345`. As you see, the numbers grow fast. And with the
number of iteration grows the execution time.