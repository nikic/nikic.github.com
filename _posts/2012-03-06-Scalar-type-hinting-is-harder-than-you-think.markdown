---
layout: post
title: Scalar type hinting is harder than you think
excerpt: A quick overview of the different scalar type hinting proposals and why PHP is having such a hard time deciding.
---
One of the features originally planned for PHP 5.4 was scalar type hinting. As you know, PHP 5.4 released without them.

Recently the topic has come up again on the mailing list and there has been a hell lot of discussion about it. Yesterday
ircmaxell published a [blog post about his particular proposals][1].

The reactions on [reddit][2] were mixed. On one hand it is clear that people do really want scalar type hints, on the
other hand they didn't seem to like that particular proposal.

[One comment][3] particularly caught my interest:

> Why can't the PHP dev's just get this thing right? Why must they always choose the worst, more useless, most
> convoluted way to implement every new feature! It's disheartening. Especially when there aren't good
> backwards-compatibility reasons for doing it wrong.

In a way, this is a pretty important comment: It shows what people think about how PHP is being developed. They think
that it's a bunch of people that somehow, magically, manage to always do the wrong decisions.

**That's not how it works.** Really. You see, most people, when confronted with scalar type hinting, think "Hey, this is
so simple, just do XYZ", and then wonder why PHP is having such a hard time implementing this most trivial of all
features.

The thing is: It's just not that simple. Most proposals seem so easy and straightforward on first sight, but when it
comes to the details they are not.

In the following I want to introduce you to some of the various proposals that were made in the past and the problems
associated with them. I hope this way people will get a little bit more insight into why this is so hard.

Strict type hinting
-------------------

I'll start with the proposal that I personally dislike most: Strict type hinting, i.e. allowing only the hinted type to
be passed and not any of the types that PHP would normally consider equivalent. See this example:

{% highlight php %}
<?php
function foo(int $i) { /* ... */ }

foo(1);   // works
foo(1.0); // fatal error: int expected, float given
foo("1"); // fatal error: int expected, string given
{% endhighlight %}

I think it is evident that this is not an option. One of PHP's greatest strengths is being a weakly typed language and
this proposal would turn it into a strictly typed one. This goes against the PHP philosophy and also differs from how
pretty much everything else works in PHP.

Unenforced type hinting
-----------------------

Another proposal that was made, is type hinting that isn't enforced by the engine. WTF? What would this be good for?
Basically, it would work just like doc comments (which aren't enforced either ^^), but with nicer syntax.

{% highlight php %}
<?php
function foo(int $i) { /* ... */ }

foo(1);          // works
foo(1.0);        // works
foo("1");        // works
foo(array(123);  // works
foo($iAmObject); // works
// everything works...
{% endhighlight %}

I dislike this proposal, too, for obvious reasons. For me it just doesn't make any sense to have type hints that are
ignored. Doc comments do that already well enough...

Casting weak type hinting
-------------------------

A proposal that came up recently (this is the one [introduced by ircmaxell][1]) is type hinting based on casts.

{% highlight php %}
<?php
function foo((int) $i) {
    var_dump($i);
}
foo(1);       // int(1)
foo(1.0);     // int(1)
foo("1");     // int(1)

foo(1.5);     // int(1)
foo(array()); // int(0)
foo("hi");    // int(0)
{% endhighlight %}

This is one of the more interesting proposals, for several reasons:

Firstly, it is the only proposal that can be implemented without breaking backwards compatibility at all. All other
proposals would at least break existing type hints on classes called "Int" or "String" (believe it or not, but some
people really do use classes like this...)

Secondly, the syntax makes very clear that it will cast the value. It also reuses PHP's normal cast semantics, thus
making it consistent.

But it also has some not insignificant issues:

The first one is that the syntax is simply ugly. Sure, you don't have to tell me, that's subjective and stuff like that,
but honestly, this is an important aspect to consider. Scalar type hinting will be used a lot and we shouldn't make it
ugly.

The second, by far more important issue is that PHP's cast semantics simply suck. In the above example, the first three
casts seem logical, but the three last ones are clearly broken. At least I don't want `"foobar"` or `array()` to be
valid arguments to an integer parameter. It just doesn't make sense. That's why this proposal would go hand in hand with
a change of implicit cast semantics: Whenever such a "strange" cast (one with data loss) occurs, it should throw a
notice. This again would be a controversial change, as it would also change stuff in other places (which is probably a
good thing though ^^). Also I would argue that just a notice is not enough and it should instead throw a recoverable
fatal error.

Also I'm not really sure as to whether we really want incoming parameters to be cast, instead of just being validated.
One argument against it is, that this approach is somewhat different from the existing `array`, `callable` and `Class`
type hints. Especially if you consider that you would now have both an `(array)` and an `array` type hint, the issue
might become more clear: When would you use one over another? When would you want the argument to be cast and when just
checked? Additionally the casts would prevent type hinted parameters to be used by reference in any reasonable manner.

Strict weak type hinting
------------------------

This leads us to another possibility, which is also my favorite: Doing weak type hints, but with stricter input
validation (and without casts).

{% highlight php %}
<?php
function foo(int $i) {
    var_dump($i);
}
foo(1);       // int(1)
foo(1.0);     // float(1.0)
foo("1");     // string(1) "1"

foo(1.5);     // fatal error: int expected, float given
foo(array()); // fatal error: int expected, array given
foo("hi");    // fatal error: int expected, string given
{% endhighlight %}

I like this proposal most, for several reasons: Firstly, it uses the "normal" type hinting syntax, not the strange
cast syntax. Secondly it has stricter validation semantics as it only lets those values through which are representable
in the hinted type without data loss. And thirdly it does not do casts, which is in line with previous type hints and
also allows references.

But it obviously also comes with problems: Again, this won't be backwards compatible as it at least will break existing
`Int` or `String` class type hints. Additionally, depending on the exact implementation, it would also make stuff like
`int` and `string` reserved keywords, thus (maybe) breaking even more stuff. Realistically though I'd say that this will
only affect a small fraction of scripts.

By the way, this proposal would be very similar to how parameters for internal functions are parsed. E.g. an int
argument of an internal function will not accept `"foobar"` and instead throw a warning.

Boxing based type hinting
-------------------------

Another very interesting idea that also came from ircmaxell is type hinting using boxing. Basically this would add
magic methods for casting objects to scalars and the other way around. Something like this (just example names):

    public function __toScalar()
    public static function __fromScalar($value)

Now see how this could be useful:

{% highlight php %}
<?php

class Int {
    protected $value;

    public function __construct($value) {
        if (false === $this->value = filter_var($value, FILTER_VALIDATE_INT)) {
            throw new Exception('Malformed integer'));
        }
    }

    public function __toScalar() {
        return $this->value;
    }

    public function __fromScalar($value) {
        return new static($value);
    }
}

foo(10); // foo() is expecting an Int, so call Int::__fromScalar(10),
         // which returns an Int(10). This is then passed to foo().

foo("hi"); // Int::__fromScalar("hi") is called, which tries to create
           // an Int("hi"), but this throws an Exception.

function foo(Int $i) {
    return str_repeat('*', $i); // $i is Int(10) here, but str_repeat expects an integer,
                                // thus call $i->__toScalar() [or ->__toInt() or whatever],
                                // which returns 10. Pass with to str_repeat().
}
{% endhighlight %}

This basically allows you to implement type hinting in userland code. It also allows to create strict type hinting:

{% highlight php %}
<?php

class StrictInt {
    protected $value;

    public function __construct($value) {
        if (!is_int($value)) {
            throw new Exception('Not an integer'));
        }

        $this->value = $value;
    }

    public function __toScalar() {
        return $this->value;
    }

    public function __fromScalar($value) {
        return new static($value);
    }
}

function foo(StrictInt $i) { /* ... */ }

foo(10); // foo() is expecting a StrictInt, so call StrictInt::__fromScalar(10),
         // which returns a StrictInt(10). This is then passed to foo().

foo("10"); // StrictInt::__fromScalar("10") is called, which tries to create
           // StrictInt("10"), but this raises an exception.
{% endhighlight %}

As you can see this proposal is very powerful as it allows users to define their own type hinting rules (PHP could
obviously still provide some default `Int`, etc classes.)

My main issue with this proposal is that it seems like a little bit overkill to implement type hinting this way (as it
requires to always box and unbox the type). But obviously the internally defined type hinting classes could optimize
this.

But generally this proposal is not yet that mature, so it could well have some more flaws. Still, I think that this is
one of the more interesting approaches, especially as it is so powerful.

We need you!
------------

I hope that I could give you a small overview of the different possible approaches.

Now it's your turn! PHP needs your valuable feedback on the different proposals, so a good decision can be made.

You can give feedback directly on the internals [mailing list][4], but if you don't want to do that, you can also
leave a comment or ping ircmaxell or me in the [StackOverflow PHP chatroom][5].

 [1]: http://blog.ircmaxell.com/2012/03/parameter-type-casting-in-php.html
 [2]: http://www.reddit.com/r/PHP/comments/qiniv/parameter_type_casting_in_php/
 [3]: http://www.reddit.com/r/PHP/comments/qiniv/parameter_type_casting_in_php/c3xyzc5
 [4]: http://php.net/mailing-lists.php
 [5]: http://chat.stackoverflow.com/rooms/11/php