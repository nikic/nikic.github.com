---
layout: post
title: "PHP internals: When does foreach copy?"
excerpt: PHP's foreach language construct sometimes copies the array it iterates and sometimes does not. This post analyzes when and why this happens.
---
**Important**: This post requires some knowledge of PHP's internal workings, namely `zval`s,
`refcount`s and copy-on-write behavior. If those terms don't mean anything to you, you will need to
read up on them in oder to understand this post. I would recommend [an article by Sara Golemon][0].

PHP's `foreach` is a very neat and to-the-point language construct. Still some people don't like to
use it, because they think it is slow. One reason usually named is that `foreach` copies the array
it iterates. Thus some people recommend to write:

{% highlight php startinline %}
$keys = array_keys($array);
$size = count($array);
for ($i = 0; $i < $size; $i++) {
    $key   = $keys[$i];
    $value = $array[$key];

    // ...
}
{% endhighlight %}

Instead of the much more intuitive and straightforward:

{% highlight php startinline %}
foreach ($array as $key => $value) {
    // ...
}
{% endhighlight %}

There are two problems with this:

 1. [Microoptimization is evil][1]. Usually it only wastes your time and doesn't give any measurable
    performance improvements.
 2. The copying behavior of `foreach` is somewhat more complicated than most people think. Often the
    "optimized" variant happens to be slower than the original.

When does foreach copy?
-----------------------

Whether or not `foreach` copies the array and how much of it depends on three things: Whether the
iterated array is referenced, how high its `refcount` is and whether the iteration is done by
reference.

### Not referenced, refcount == 1

In the below code `$array` is not referenced and has a refcount of `1`. In this case `foreach` will
**not** copy the array ([proof][2]) - contrary to the popular belief that `foreach` always copies
the iterated array if it isn't referenced.

{% highlight php startinline %}
test();
function test() {
    $array = range(0, 100000);
    foreach ($array as $key => $value) {
        // ...
    }
}
{% endhighlight %}

The reason is simple: Why should it? The only thing that `foreach` modifies about `$array` is it's
internal array pointer. This is expected behavior and thus doesn't need to be prevented.

### Not referenced, refcount > 1

The following code looks very similar to the previous one. The only difference is that the array is
now passed as an argument. This seems like an insignificant difference, but it does change the
behavior of `foreach`: It now **will** copy the array structure, but **not** the values ([proof][3];
if you want to see that this is really only the structure being copied compare [this][8] and [that][9]
script. The first only copies the structure, the second copies both).

{% highlight php startinline %}
$array = range(0, 100000);
test($array);
function test($array) {
    foreach ($array as $key => $value) {
        // ...
    }
}
{% endhighlight %}

This might seem odd at first: Why would it copy when the array is passed through an argument, but
not if it is defined in the function? The reason is that the array `zval` is now shared between
multiple variables: The `$array` variable outside the function and the `$array` variable inside it.
If `foreach` would iterate the array without copying its structure it would not only change the array
pointer of the `$array` variable in the function, but also the pointer of the `$array` variable outside
the function. Thus `foreach` needs to copy the array structure (i.e. the hash table). The values on
the other hand still can share zvals and thus don't need to be copied.

### Referenced

The next case is very similar to the previous one. The only difference is that the array is passed
by reference. In this case the array again will **not** be copied ([proof][4]).

{% highlight php startinline %}
$array = range(0, 100000);
test($array);
function test(&$array) {
    foreach ($array as $key => $value) {
        // ...
    }
}
{% endhighlight %}

In this case the same reasoning applies as with the previous case: The outer `$array` and the inner
`$array` share `zval`s. The difference is that they now are references (`isref == 1`). Thus in this
case it is expected that any change to the inner array will also be done to the outer array. So if
the array pointer of the inner array is changed, the array pointer of the outer array should change,
too. That's why `foreach` doesn't need to copy.

### Iterated by reference

The above examples were all iterating by value. For iteration by reference the same rules apply, but
the additional value reference changes copying behavior of the array *values* (the behavior about
*structure* copying stays).

The case "Not referenced, refcount == 1" doesn't change. By reference iteration means we want to
change the original array if there are any changes to the `$value`, so the array isn't copied
([proof][5]).

The case "Referenced" also stays the same, as in this case a change to `$value` should change all
variables referencing the iterated array ([proof][6]).

Only the "Not referenced, refcount > 1" case changes, as now both the array structure and its values
need are be copied. The array structure because otherwise the array pointer of the `$array` variable
outside the function would change and the values because a change to `$value` would also change the
outside `$array` values ([proof][7]).

Summary
-------

To summarize:

 * `foreach` will copy the array *structure* if and only if the iterated array is not referenced and
   has a `refcount > 1`
 * `foreach` will additionally copy the array *values* if and only if the previous point applies and
   the iteration is done by reference


 [0]: http://blog.golemon.com/2007/01/youre-being-lied-to.html
 [1]: http://blog.ircmaxell.com/2011/08/on-optimization-in-php.html
 [2]: http://codepad.viper-7.com/MFNrmk
 [3]: http://codepad.viper-7.com/mixX1H
 [4]: http://codepad.viper-7.com/9bKciB
 [5]: http://codepad.viper-7.com/VkqVQ1
 [6]: http://codepad.viper-7.com/qKMkDo
 [7]: http://codepad.viper-7.com/O584we
 [8]: http://codepad.viper-7.com/nNX1Rt
 [9]: http://codepad.viper-7.com/lQC1K6