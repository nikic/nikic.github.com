---
layout: post
title: Methods on primitive types in PHP
excerpt: This article discusses the merits of allowing method calls on primitive PHP types, like strings and arrays.
---
A few days ago Anthony Ferrara wrote down some thoughts on [the future of PHP][ircmaxell_opinion]. I concur with most of
his opinions, but not all of them. In this post I'll focus on one particular aspect: Turning primitive types like strings
or arrays into "pseudo-objects" by allowing to perform method calls on them.

Lets start off with a few examples of what this entails:

{% highlight php startinline %}
$str = "test foo bar";
$str->length();      // == strlen($str)        == 12
$str->indexOf("foo") // == strpos($str, "foo") == 5
$str->split(" ")     // == explode(" ", $str)  == ["test", "foo", "bar"]
$str->slice(4, 3)    // == substr($str, 4, 3)  == "foo"

$array = ["test", "foo", "bar"];
$array->length()       // == count($array)             == 3
$array->join(" ")      // == implode(" ", $array)      == "test foo bar"
$array->slice(1, 2)    // == array_slice($array, 1, 2) == ["foo", "bar"]
$array->flip()         // == array_flip($array)        == ["test" => 0, "foo" => 1, "bar" => 2]
{% endhighlight %}

Here `$str` is just a normal string and `$array` just a normal array - they aren't objects. We just give them a bit of
object-like behavior by allowing to call methods on them.

Note that this isn't far off dreaming, but something that already exists right now. The [scalar objects][scalar_objects]
PHP extension allows you to define methods for the primitive PHP types.

The introduction of method-call support for primitive types comes with a number of advantages that I'll outline in the
following:

An opportunity for a cleaner API
--------------------------------

The likely most common complaint you get to hear about PHP is the inconsistent and unclear naming of functions in the
standard library, as well as the equally inconsistent and unclear order of parameters. Some typical examples:

{% highlight php startinline %}
// different naming conventions
strpos
str_replace

// totally unclear names
strcspn                  // STRing Complement SPaN
strpbrk                  // STRing Pointer BReaK

// inverted parameter order
strpos($haystack, $needle)
array_search($needle, $haystack)
{% endhighlight %}

While this issue is often overemphasized (we *do* have IDEs), it is hard to deny that the situation is rather
suboptimal. It should also be noted that many functions exhibit problems that go beyond having a weird name. Often
edge-case behaviors were not properly considered, thus creating the need to specially handle them in the calling code.
(For the string functions edge-cases usually involve empty strings or offsets that are at the very end of a string.)

A common suggestion is to just add a huge number of function aliases in PHP 6, which will unify the function names and
parameter orders. So we'd have `string\pos()`, `string\replace()`, `string\complement_span()` or something in that
direction. Personally (and this seems to be the opinion of many php-src devs) this makes little sense to me. The current
function names are deeply ingrained into the muscle memory of any PHP programmer and applying a few trivial cosmetic
changes to them just doesn't seem worth it.

The introduction of an OO API for primitive types on the other hand offers an opportunity of an API redesign as a side
effect of switching to a new paradigm. It also offers a truly clean slate, without the need to meet any expectations
coming with the old procedural API. Two examples:

 * I would very much like to have the `$string->split($delimiter)` and `$array->join($delimiter)` methods, which are the
   universally accepted names for this functionality (as opposed to `explode` and `implode`). On the other hand I would
   be very uncomfortable to have a `string\split($delimiter)` function with this behavior, because the existing
   `str_split` function does something completely different (chunking).
 * I would certainly like the new API to use exceptions for error reporting. As this is an OO API that is automatically
   a given. Exceptions could of course also be used with a renamed procedural API, however this goes against the current
   convention where all procedural functions use warnings for error handling. That's not set in stone, but it certainly
   is a discussion I would like to avoid ;)

That's my *main* motivation for an OO API on primitive types: A clean slate, allowing us to implement a set of properly
designed APIs. But of course that's not the only advantage of such a move. The OO syntax offers a number of further
benefits, discussed in the following.

Improved readability
--------------------

Procedural calls commonly do not chain well. Consider the following example:

{% highlight php startinline %}
$output = array_map(function($value) {
    return $value * 42;
}, array_filter($input, function($value) {
    return $value > 10;
});
{% endhighlight %}

At a glance, what are `array_map` and `array_filter` applied to? In what order are the calls happening? The variable
`$input` is hidden somewhere in the middle of two closures and the function calls are written in the reverse order of
how they are actually applied. Now the same example using an OO syntax:

{% highlight php startinline %}
$output = $input->filter(function($value) {
    return $value > 10;
})->map(function($value) {
    return $value * 42;
});
{% endhighlight %}

I daresay that in this case the order of operations (first filter then map) and the source array `$input` are a lot more
obvious.

The example is of course somewhat contrived, because `array_map` and `array_filter` are another example of functions
with swapped parameter order (which is why the input array ends up in the middle). Another example (this time from real
code) where the input parameter stays in the same position:

{% highlight php startinline %}
substr(strtr(rtrim($className, '_'), '\\', '_'), 15);
{% endhighlight %}

In this case you end up with a string of additional parameters `'_'), '\\', '_'), 15`, which are hard to associate with
the corresponding function calls. Compare this to the version using methods:

{% highlight php startinline %}
$className->trimRight('_')->replace('\\', '_')->slice(15);
{% endhighlight %}

Here the operations and their arguments are tightly grouped and once again the order of the method calls matches the
order in which they are executed.

Another readability benefit that can be derived from this syntax is the absence of the needle/haystack problem. While
aliasing lets us resolve this issue by introducing some uniform parameter order convention, the problem simply does not
exist in the first place with an OO API:

{% highlight php startinline %}
$string->contains($otherString)
$array->contains($someValue)

$string->indexOf($otherString)
$array->indexOf($someValue)
{% endhighlight %}

Here there can be no confusion as to which part takes which role.

Polymorphism
------------

PHP currently provides a `Countable` interface, which can be implemented by classes to customize the output of
`count($obj)`. Why is this needed? Because we don't have polymorphism for functions. We do however have polymorphism
for methods:

If arrays implement `$array->count()` as a (pseudo-)method, the code doesn't actually care that `$array` is an array. It
could be any other object implementing the `count()` method. This basically gives us the same behavior as `Countable`,
just without the engine hackery it requires.

This is also a much more general solution. For example you could implement a `UnicodeString` class, which implements all
the methods of the string type, and then use normal strings and a `UnicodeString`s interchangeably. Well, at least
that's the theory. This would obviously only work as long as the usage is limited to just the string methods, and would
fail once the concatenation operator is employed (full operator overloading is currently only supported for internal
classes).

Still, I hope it's clear that this is a rather powerful concept. The same also applies to arrays, e.g. you could have
an `SplFixedArray` behave the same way as an array, by implementing the same interface.

Now that we've covered some of the advantages of this approach, lets also consider some problems it faces:

Loose Typing
------------

Quoting from Anthony's blog post:

> [S]calars are not objects, but more importantly they are not any type. PHP relies on a type-system that truly believes
> that strings are integers. A lot of the flexibility to the system is based that any scalar can be converted to any
> other scalar with ease. \[...\]
>
> More importantly though, due to this loose type system, you can't 100% of the time know what type a variable will be.
> You can tell how you want to treat it, but you can't tell what it is under the hood. Even with casting or scalar type
> hinting it isn't a perfect situation since there are cases where types can still change.

To illustrate the issue, consider the following example:

{% highlight php startinline %}
$num = 123456789;
$sumOfDigits = array_sum(str_split($num));
{% endhighlight %}

Here `$num` is treated as a string of digits, which is split apart using `str_split` and then summed using `array_sum`.
Now try the same using methods:

{% highlight php startinline %}
$num = 123456789;
$sumOfDigits = $num->chunk()->sum();
{% endhighlight %}

Here the `chunk()` method from the string type is called on a number. What happens? Anthony suggests one solution:

> So that means that all scalar operations would need to be bound to all scalar types. Which leads to an object model
> where scalars have all of the math methods, as well as all of the string methods. What a nightmare...

As the quote already says, that's by no means an acceptable solution. However I think that we can absolutely get away
with just throwing an error (exception!) in this case. To explain why this is viable, lets take a look at what types a
PHP value can have.

Primitive types in PHP
----------------------

Apart from objects, PHP has the following variable types:

    null
    bool
    int
    float
    string
    array
    resource

Now, lets think about which of these actually could have meaningful methods: We'll drop `resource` from consideration
right away (that's a legacy type) and have a look at the rest. Null and bool obviously have no need for methods, unless
you want to invent abominations like `$bool->invert()`.

The vast majority of math functions don't do well as methods either. Consider:

{% highlight php startinline %}
log($n)        $n->log()
sqrt($n)       $n->sqrt()
acosh($n)      $n->acosh()
{% endhighlight %}

I hope you agree that math functions read a lot better in function notation. There are of course some few methods you
could reasonably apply to the number type. For example `$num->format(10)` reads quite nicely. However, that's about it.
There's no real need for an OO number API, as there's little functionality you could include. (Furthermore the current
math API is not so problematic in terms of naming, as math operation names are pretty standardized.)

This leaves us with just strings and arrays. We've already seen that there are many nice APIs for those two types. But
what does all this have to do with the loose typing issue? The important point is the following:

While it is very common to treat strings as if they were integers (e.g. coming from HTTP or DB), the inverse is not
true: It is very uncommon to directly use an integer as a string. For example, the following code would really confuse
me:

{% highlight php startinline %}
strpos(54321, 32, 1);
{% endhighlight %}

As treating numbers as strings in this way is a rather weird operation, I think it's totally okay to require a cast in
this case. Using the original sum-of-digits example:

{% highlight php startinline %}
$num = 123456789;
$sumOfDigits = ((string) $num)->chunk()->sum();
{% endhighlight %}

Here we have clarified that, yes, we actually do want to treat that number as a string. To me this is acceptable for the
cases where you want to make use of a hack like this.

For arrays the situation is even easier: It doesn't make any sense to apply an array operation to something that isn't
an array.

Another factor that ameliorates this issue are scalar typehints (which I totally assume to be present in any PHP version
this gets in - really embarrassing that we still don't have them). If you typehint against `string`, the input you'll
get to see *will* be a string (even if the value passed to the function wasn't - depending on the details of the
typehinting implementation).

However, I don't want to imply that there is no problem here at all. Due to incorrect function design, it can sometimes
happen that an unexpected type sneaks into the code. For example `substr($str, strlen($str))` will, in its wisdom,
return `bool(false)` instead of `string(0) ""`. (However, that's really just an issue with `substr`. The method API
won't have that issue, so you won't run into it.)

Object passing semantics
------------------------

Apart from the loose-typing problem, there is another semantic issue with pseudo-methods on primitive types: Objects in
PHP have different passing semantics than other types (somewhat similar to references). If we start allowing method
calls on strings and arrays, they'll start to look like objects and some people might expect them to have object
passing semantics because of that. This issue applies both to strings and arrays:

{% highlight php startinline %}
function change($arg) {
    echo $arg->length(); // $arg looks like object
    $arg[0] = 'x';       // but doesn't have object passing semantics
}

$str = 'foo';
change($str); // $str stays the same

$array = ['f', 'o', 'o'];
change($array); // $array stays the same
{% endhighlight %}

We could of course change the passing semantics. In my eyes passing large structures like arrays by-value was a pretty
bad idea in the first place and I would prefer them to be passed by-object. However, that would be a pretty big
backwards-compatibility break and one that's not easy to refactor automatically (at least that would be my assumption,
I did not perform tests to determine the actual impact of such a change). For strings on the other hand, by-object
passing would be catastrophic, unless we make the strings fully immutable at the same time, giving up the local
mutability we currently have (which I personally find quite handy - go and try to change a character in a Python
string).

I don't know if there is some nice way to resolve this issue of expectations, apart from emphasizing in our
documentation that strings and arrays are only *pseudo-objects* with methods, not actual objects.

This issue can also be expanded to other object-related features. E.g. you could ask whether something like
`$string instanceof string` would start to work. I'm not yet sure just how far the whole thing should go. It might be
best to strictly stick with just methods, to emphasize that these are not actual objects. It might however also be good
to support further features of the OO system. That point deserves further thought.

Current state
-------------

In conclusion, this approach does have a number of problems, but I don't view them as particularly critical. At the same
time this offers a great opportunity to introduce clean APIs for our most basic types and improve the readability (and
writability) of code performing operations on them.

So what's the state of this idea? From what I gathered, the folk on internals is not particularly opposed to this
approach and prefers it to aliasing all the things. The main thing that's missing to move this forward is an API
proposal.

For this purpose I created the [scalar objects][scalar_objects] extension, which implements this functionality as a PHP
extension. It allows you to register a class which will handle method calls for the respective primitive type. An
example:

{% highlight php startinline %}
class StringHandler {
    public function length() {
        return strlen($this);
    }

    public function contains($str) {
        return false !== strpos($this, $str);
    }
}

register_primitive_type_handler('string', 'StringHandler');

$str = "foo bar baz";
var_dump($str->length());          // int(11)
var_dump($str->contains("bar"));   // bool(true)
var_dump($str->contains("hello")); // bool(false)
{% endhighlight %}

I have started working on a [string handler][string_handler] including an [API specification][string_api] some time ago,
but never really finished that project (I hope I'll find the motivation to pick it up again sometime soon). There are
also a number of other projects working on such APIs.

So, this is one of the things I'd like to see for PHP 6. I may write another post for my other plans in that direction.

  [ircmaxell_opinion]: http://blog.ircmaxell.com/2014/03/an-opinion-on-future-of-php.html
  [scalar_objects]: https://github.com/nikic/scalar_objects
  [string_handler]: https://github.com/nikic/scalar_objects/blob/master/handlers/string.php
  [string_api]: https://github.com/nikic/scalar_objects/blob/master/doc/string_api.md
