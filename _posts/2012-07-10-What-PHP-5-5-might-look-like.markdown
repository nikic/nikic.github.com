---
layout: post
title: What PHP 5.5 might look like
excerpt: PHP 5.5 is still in an early development stage, but there are already many proposals that are being worked on. This post gives some insights into the recent developments.
---
PHP 5.4 was released just four months ago, so it probably is a bit too early to look at the next PHP version. Still I'd
like to give all the people who aren't following the [internals mailing list][mailing_list] a small sneak peak at what
PHP 5.5 might look like.

But be sure to understand this: PHP 5.5 is in an early development stage, so nobody knows how the end result will look
like. All I am talking about here are *proposals*. I'm pretty sure that not all of the things listed below will go into
PHP 5.5, or at least not in their current form.

So, don't get *too* excited about this :)

The list of new features / proposals is rather large and not sorted by significance. So if you don't want to read
through all of it, here are the four features I personally am most excited about:

 * [A simple API for password hashing](#a_simple_api_for_password_hashing)
 * [Scalar typehinting](#scalar_typehinting)
 * [Getters and setters](#getters_and_setters)
 * [Generators](#generators)

Now, without further ado, the list of stuff being worked on for PHP 5.5:

## Backwards compatibility breaks

We'll start off with two changes that already landed in master and represent BC breaks (to some degree at least):

### Windows XP and 2003 support dropped

<small>Status: landed; Responsible: Pierre Joye</small>

PHP 5.5 will no longer support Windows XP and 2003. Those systems are around a decade old, so PHP is pulling the plug
on them.

### /e modifier deprecated

<small>Status: landed; Responsible: myself</small>

The `e` modifier instructs the [`preg_replace`][preg_replace] function to evaluate the replacement string as PHP code
instead of just doing a simple string replacement. Unsurprisingly this behavior is a constant source of problems and
security issues. That's why use of this modifier will throw a deprecation warning as of PHP 5.5. As a replacement you
should use the [`preg_replace_callback`][preg_replace_callback] function. You can find more information on this change
in the [corresponding RFC][e_modifier].

## Function and class additions

Next we'll look at some the planned function and class additions:

### boolval()

<small>Status: landed; Responsible: Jille Timmermans</small>

PHP already implements the `strval`, `intval` and `floatval` functions. To be consistent the `boolval` function is now
added, too. It does exactly the same thing as a `(bool)` cast, but can be used as a callback function.

### hash_pbkdf2()

<small>Status: landed; Responsible: Anthony Ferrara</small>

[PBKDF2][pbkdf2] stands for "Password-Based Key Derivation Function 2" and is - as the name already says - an algorithm
for deriving a cryptographic key from a password. This is required for encryption algorithms, but can also be used for
password hashing. For a more extensive description and usage examples see [the RFC][hash_pbkdf2].

### Intl additions

<small>Status: landed; Responsible: Gustavo André dos Santos Lopes</small>

There have been many improvements to the [intl extension][intl]. E.g. there will be new `IntlCalendar`,
`IntlGregorianCalendar`, `IntlTimeZone`, `IntlBreakIterator`, `IntlRuleBasedBreakIterator`,
`IntlCodePointBreakIterator` classes. I sadly don't know much about the intl extension, so I will just direct you to
the mailing list announcements for [Calendar][intl_calendar] and [BreakIterator][intl_breakiterator], if you want to
know more.

### array_column()

<small>Status: proposed; Responsible: Ben Ramsey</small>

There is [a proposal][array_column] pending for a new `array_column` (or `array_pluck`) function that would behave as
follows:

{% highlight php %}
<?php

$userNames = array_column($users, 'name');
// is the same as
$userNames = [];
foreach ($users as $user) {
    $userNames[] = $user['name'];
}
{% endhighlight %}

So it would be like fetching a column from a database, but for arrays.

### A simple API for password hashing

<small>Status: proposed; Responsible: Anthony Ferrara</small>

The recent password leaks (from LinkedIn etc) have shown that even large websites don't get how to properly hash
passwords. People have been advocating the use of bcrypt for years, but still most people seem to be using completely
unsafe `sha1` hashes.

We figured that the reason for this might be the really hard to use API of the [`crypt`][crypt] function. Thus we would
like to introduce a new, simple API for secure password hashing:

{% highlight php %}
<?php

$password = "foo";

// creating the hash
$hash = password_hash($password, PASSWORD_BCRYPT);

// verifying a password
if (password_verify($password, $hash)) {
    // password correct!
} else {
    // password wrong!
}
{% endhighlight %}

The new hashing API comes with a few more features, which are outlined in [the RFC][password_hash].

## Language changes

Now comes the really interesting stuff: New language features and enhancements.

### Constant dereferencing

<small>Status: landed; Responsible: Xinchen Hui</small>

"Constant dereferencing" means that array operations can be directly applied to string and array literals. Here two
examples:

{% highlight php %}
<?php

function randomHexString($length) {
    $str = '';
    for ($i = 0; $i < $length; ++$i) {
        $str .= "0123456789abcdef"[mt_rand(0, 15)]; // direct dereference of string
    }
}

function randomBool() {
    return [false, true][mt_rand(0, 1)]; // direct dereference of array
}
{% endhighlight %}

I don't think that this feature is of much use in practice, but it makes the language a bit more consistent. See also
[the RFC][const_dereference].

### empty() works with function calls (and other expressions)

<small>Status: landed; Responsible: myself</small>

Currently the `empty()` language construct can only be used on variables, not on other expressions. In particular code
like `empty($this->getFriends())` would throw an error. As of PHP 5.5 this becomes valid code. For more info see [the
RFC][empty_exprs].

### Getting the fully qualified class name

<small>Status: proposed; Responsible: Ralph Schindler</small>

PHP 5.3 introduced namespaces with the ability to alias classes and namespaces to shorter versions. This does not apply
to string class names though:

{% highlight php %}
<?php

use Some\Deeply\Nested\Namespace\FooBar;

// does not work, because this will try to use the global `FooBar` class
$reflection = new ReflectionClass('FooBar');
{% endhighlight %}

To solve this a new `FooBar::class` syntax is proposed, which returns the fully qualified name of the class:

{% highlight php %}
<?php

use Some\Deeply\Nested\Namespace\FooBar;

// this works because FooBar::class is resolved to "Some\\Deeply\\Nested\\Namespace\\FooBar"
$reflection = new ReflectionClass(FooBar::class);
{% endhighlight %}

For more examples see [the RFC][class_name_resolution].

### Parameter skipping

<small>Status: proposed; Responsible: Stas Malyshev</small>

If you have a function accepting multiple optional parameters there is currently no way to change just the last one,
leaving all others at their default.

Taking the example from [the RFC][skip_params], if you have a function like the following:

    function create_query($where, $order_by, $join_type='', $execute = false, $report_errors = true) { ... }

Then there is no way to set `$report_errors = false` without replicating the other two default values. To solve this
a way of skipping parameters is proposed:

    create_query("deleted=0", "name", default, default, false);

Personally I'm not particular fond of this proposal. In my eyes code that needs this feature is just badly designed.
Functions shouldn't have 12 optional parameters.

### Scalar typehinting

<small>Status: proposed; Responsible: Anthony Ferrara</small>

Scalar typehinting was originally planned to go into 5.4, but never made it due to lack of consensus. For more info
on why scalar typehints haven't made it into PHP yet, see: [Scalar typehints are harder than you
think][harder_than_you_think].

For PHP 5.5 the discussion has come up again and I think there is a fairly decent [proposal for scalar typehints with
casting][scalar_typehinting].

It would work by casting the input value to the specified type, but only if the cast can occur *without data loss*. E.g.
`123`, `123.0`, `"123"` would all be valid inputs for an `int` parameter, but `"hallo world"` would not. This matches
the behavior of internal functions.

{% highlight php %}
function foo(int $i) { ... }

foo(1);      // $i = 1
foo(1.0);    // $i = 1
foo("1");    // $i = 1
foo("1abc"); // not yet clear, maybe $i = 1 with notice
foo(1.5);    // not yet clear, maybe $i = 1 with notice
foo([]);     // error
foo("abc");  // error
{% endhighlight %}

### Getters and setters

<small>Status: proposed; Responsible: Clint Priest</small>

If you've never been a fan of writing all those `getXYZ()` and `setXYZ($value)` methods, then this should be a welcome
change for you. The proposal adds a new syntax for defining what should happen when a property is set / read:

{% highlight php %}
<?php

class TimePeriod {
    public $seconds;

    public $hours {
        get { return $this->seconds / 3600; }
        set { $this->seconds = $value * 3600; }
    }
}

$timePeriod = new TimePeriod;
$timePeriod->hours = 10;

var_dump($timePeriod->seconds); // int(36000)
var_dump($timePeriod->hours);   // int(10)
{% endhighlight %}

There are also some more features like read-only properties. If you want to know more, have a look at [the
RFC][getter_setter].

### Generators

<small>Status: proposed; Responsible: myself</small>

Currently custom iterators are used only rarely, because their implementation requires lots of boilerplate code.
Generators solve this issue by providing an easy and boilerplate-free way to create iterators.

For example, this is how you could define the `range` function, but as an iterator:

{% highlight php %}
<?php

function *xrange($start, $end, $step = 1) {
    for ($i = $start; $i < $end; $i += $step) {
        yield $i;
    }
}

foreach (xrange(10, 20) as $i) {
    // ...
}
{% endhighlight %}

The above `xrange` function has the same behavior as the builtin `range` function with one difference: Instead of
returning an array with all the values, it returns an iterator which generates the values on-the-fly.

For more in-depth introduction into the topic see [the RFC][generators].

### List comprehensions and generator expressions

<small>Status: proposed; Responsible: myself</small>

List comprehensions provide a simple way to do small operations on arrays:

    $firstNames = [foreach ($users as $user) yield $user->firstName];

The above list comprehension is equivalent to the following code:

    $firstNames = [];
    foreach ($users as $user) {
        $firstNames[] = $user->firstName;
    }

It is also possible to filter arrays this way:

    $underageUsers = [foreach ($users as $user) if ($user->age < 18) yield $user];

Generator expressions are similar, but return an iterator (which generates the values on-the-fly) instead of an array.

For more examples see [the mailing list announcement][list_comprehensions].

## Wrapping up

As you can see, there is a lot of awesome stuff being worked on for PHP 5.5. But as I already said, PHP 5.5 is still
very young, so we don't know for sure what will get in and what will not.

If you want to stay updated on the new features or want to help out in discussions and/or development, be sure to
[subscribe to the internals mailing list][mailing_list].

Comments welcome!

 [mailing_list]: http://php.net/mailing-lists.php
 [preg_replace]: http://php.net/preg_replace
 [preg_replace_callback]: http://php.net/preg_replace_callback
 [e_modifier]: https://wiki.php.net/rfc/remove_preg_replace_eval_modifier
 [pbkdf2]: http://en.wikipedia.org/wiki/PBKDF2
 [hash_pbkdf2]: https://wiki.php.net/rfc/hash_pbkdf2
 [intl]: http://php.net/intl
 [intl_calendar]: http://markmail.org/thread/b7sldwebskmhbxmx
 [intl_breakiterator]: http://markmail.org/thread/ulnm3hhist5ygido
 [array_column]: https://wiki.php.net/rfc/array_column
 [crypt]: http://php.net/crypt
 [password_hash]: https://wiki.php.net/rfc/password_hash
 [const_dereference]: https://wiki.php.net/rfc/constdereference
 [empty_exprs]: https://wiki.php.net/rfc/empty_isset_exprs
 [class_name_resolution]: https://wiki.php.net/rfc/class_name_scalars
 [skip_params]: https://wiki.php.net/rfc/skipparams
 [harder_than_you_think]: http://nikic.github.com/2012/03/06/Scalar-type-hinting-is-harder-than-you-think.html
 [scalar_typehinting]: https://wiki.php.net/rfc/scalar_type_hinting_with_cast
 [getter_setter]: https://wiki.php.net/rfc/propertygetsetsyntax-as-implemented
 [generators]: https://wiki.php.net/rfc/generators
 [list_comprehensions]: http://markmail.org/thread/uvendztpe2rrwiif