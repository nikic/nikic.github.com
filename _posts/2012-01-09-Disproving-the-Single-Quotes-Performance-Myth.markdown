---
layout: post
title: Disproving the Single Quotes Performance Myth
excerpt: One of the oldest myths around PHP is that single quotes are faster than double quotes. And. It. Is. Not. True.
---
If there is one PHP related thing that I *really* hate, then it is definitely the Single Quotes
Performance Myth. Let's do a random Google search for ["PHP single quotes performance"][1]: [You][2]
[will][3] [get][4] [many][5] [results][6] telling you that single quotes are faster than double
quotes and that string interpolation is much slower than string concatenation. Most of them advise
to use single quotes and concatenation to improve the performance of your application.

Let's be clear here: **This is pointless.**

I hold the opinion that microoptimization like that is just pointless. I doubt that string parsing
performance is the bottleneck of even a single PHP application out there. [This comment][7] nails it
pretty well: "And, on top of that, by using less pixels, you reduce greenhouse emissions." I think
that everybody will agree that using single quotes won't save any significant greenhouse emissions.
Still everybody seems to believe that it saves significant execution time. Strange.

But okay, some say "Well, it doesn't cost me anything to write single quotes, so why not just do
it?"

Sounds plausible. So I decided to look into how big the difference really is. And it turned out:
There is none.

Strings at runtime
------------------

What many people don't realize is that strings are parsed during lexing, not during execution. When
a PHP script runs all strings are already parsed.

A simple proof:

{% highlight php startinline %}
$x = 'Test';
$y = "\x54\x65\x73\x74";
{% endhighlight %}

If you have a look at the opcodes that PHP generates for this they will look like this:

    compiled vars:  !0 = $x, !1 = $y
    line     # *  op                           fetch          ext  return  operands
    ---------------------------------------------------------------------------------
       3     0  >   ASSIGN                                                 !0, 'Test'
       4     1      ASSIGN                                                 !1, 'Test'
             2    > RETURN                                                 1

You don't need to understand the complete output, but you should still see the main point: Both
`'Test'` and `"\x54\x65\x73\x74"` compiled down to the very same opcode.

What does this mean? That the used quote type and the use of escape sequences does not affect
runtime, not at all.

Strings at compile time
-----------------------

Now some may argue: PHP is an interpreted language, so compile time actually also happens at
runtime. My response to that would normally be: Use [APC][8]. APC will cache the generated opcodes
so they don't need to be created on every single request. This can greatly improve page loading time
and CPU load, so if you aren't using APC yet you might try as well now ;)

But, hypothetically, let use assume that we are not using APC. So let's see whether single quoted
strings are actually lexed faster than double quoted ones.

For this we will use a handy function called [`token_get_all`][9]: It lexes a PHP string into a
token array. The function will return the token texts plainly and not preparsed, but the internal
string parsing routines still will get called, so we can use this as a good approximation to the
real numbers.

For testing I use the following script:

{% highlight php startinline %}
const NUM = 10000000;

$singleQuotedStringCode = "<?php '" . str_repeat('x', NUM) . "';";
$doubleQuotedStringCode = '<?php "' . str_repeat('x', NUM) . '";';

$startTime = microtime(true);
token_get_all($singleQuotedStringCode);
$endTime = microtime(true);

echo 'Single quotes: ', $endTime - $startTime, ' seconds', "\n";

$startTime = microtime(true);
token_get_all($doubleQuotedStringCode);
$endTime = microtime(true);

echo 'Double quotes: ', $endTime - $startTime, ' seconds', "\n";
{% endhighlight %}

It creates two strings with *ten million* `x` characters and lexes them. Here's what I get:

    Single quotes: 0.061846971511841 seconds
    Double quotes: 0.061599016189575 seconds

Namely: **No difference.** The Single Quotes Performance Myth thus is just a big lie: Single
quotes are neither faster at runtime nor at compile time.

String interpolation
--------------------

So the next question is, what is faster: String concatenation or string interpolation?

Let's consider this simple case:

{% highlight php startinline %}
$world = 'World';
'Hallo ' . $world;
"Hallo $world";
{% endhighlight %}

If you [try running the corresponding benchmark][10] you will get numbers similar to this:

    Concatenation: 0.015104055404663 seconds
    Interpolation: 0.016894817352295 seconds

So interpolation seems *slightly* slower than concatenation, doesn't it? Well, not exactly, but more
on that later. Let's look at the opcodes first:

    compiled vars:  !0 = $world
    line     # *  op                           fetch          ext  return  operands
    ---------------------------------------------------------------------------------
       2     0  >   ASSIGN                                                   !0, 'World'
       3     1      CONCAT                                           ~1      'Hallo+', !0
             2      FREE                                                     ~1
       4     3      ADD_STRING                                       ~2      'Hallo+'
             4      ADD_VAR                                          ~2      ~2, !0
             5      FREE                                                     ~2
             6    > RETURN                                                   1

Here you see the reason why concatenation is slightly faster than interpolation in the above
example:

Concatenation uses a single `CONCAT` opcode with `'Hallo+', !0` (`!0` is `$world` here). All
`CONCAT` basically does is create a new string buffer with the new length, copy `'Hallo+'` into it
and then copy `$world` into it.

Interpolation on the other hand needs two opcodes: First `ADD_STRING` is called, which in this case
basically copies `'Hallo+'` into a new string buffer. Then `ADD_VAR` is called with `~2, !0` (`~2`
is the buffer that `'Hallo+'` was copied to and `!0` is still `$world`), which basically does the
same as `CONCAT`.

If you read closely the difference is clear: Interpolation copies the `'Hallo+'` part twice,
concatenation only copies it one time. (Disclaimer: I'm actually not perfectly sure that this is
really the reason, but it would be my guess. It could also be just the overhead of another opcode.)

So, interpolation is slower after all, isn't it? Well, not really, let's consider this more
realistic example:

{% highlight php startinline %}
$name  = 'Anonymous';
$age   = 123;
$hobby = 'nothing';

'Hi! My name is ' . $name . ' and I am ' . $age . ' years old! I love doing ' . $hobby . '!';
"Hi! My name is $name and I am $age years old! I love doing $hobby!";
{% endhighlight %}

If you [run the benchmark][11] you will get numbers looking approximately like this:

    Concatenation: 0.053942918777466 seconds
    Interpolation: 0.049801111221313 seconds

And... interpolation is faster now! Why so? Let's have a look at the new opcodes:

    compiled vars:  !0 = $name, !1 = $age, !2 = $hobby
    line     # *  op                           fetch          ext  return  operands
    ---------------------------------------------------------------------------------
       2     0  >   ASSIGN                                                   !0, 'Anonymous'
       3     1      ASSIGN                                                   !1, 123
       4     2      ASSIGN                                                   !2, 'nothing'
       6     3      CONCAT                                           ~3      'Hi%21+My+name+is+', !0
             4      CONCAT                                           ~4      ~3, '+and+I+am+'
             5      CONCAT                                           ~5      ~4, !1
             6      CONCAT                                           ~6      ~5, '+years+old%21+I+love+doing+'
             7      CONCAT                                           ~7      ~6, !2
             8      CONCAT                                           ~8      ~7, '%21'
             9      FREE                                                     ~8
       7    10      ADD_STRING                                       ~9      'Hi%21+My+name+is+'
            11      ADD_VAR                                          ~9      ~9, !0
            12      ADD_STRING                                       ~9      ~9, '+and+I+am+'
            13      ADD_VAR                                          ~9      ~9, !1
            14      ADD_STRING                                       ~9      ~9, '+years+old%21+I+love+doing+'
            15      ADD_VAR                                          ~9      ~9, !2
            16      ADD_CHAR                                         ~9      ~9, 33
            17      FREE                                                     ~9
            18    > RETURN                                                   1

As you can see the interpolation opcodes are more optimal here: For once interpolation distinguishes
between adding a single character, a string and a variable. Also it reuses the same temporary
variable for the whole interpolation (`~9`), whereas concatenation uses six of them (`~3`, `~4`,
`~5`, `~6`, `~7`, `~8`).

So interpolation becomes more efficient the more is interpolated. For the simple string + var case
it is slower, but for more realistic cases with multiple variables it becomes faster.

Lesson learned
--------------

At this point I want to emphasize again that the above has no practical impact. All above demos use
high iteration counts and any differences in execution time are completely negligible.

Does this still teach us something? Yes, it does: "Never trust a statistic you didn't forge
yourself." Amen.

  [1]: https://www.google.com/search?q=php+single+quotes+performance
  [2]: http://stackoverflow.com/questions/482202/is-there-a-performance-benefit-single-quote-vs-double-quote-in-php
  [3]: http://stackoverflow.com/questions/3316060/single-quotes-or-double-quotes-for-variable-concatenation
  [4]: http://atomized.org/2005/04/php-performance-best-practices/
  [5]: https://github.com/fabpot/Twig/issues/407
  [6]: http://classyllama.com/development/php/php-single-vs-double-quotes/
  [7]: http://stackoverflow.com/questions/482202/is-there-a-performance-benefit-single-quote-vs-double-quote-in-php#comment-299001
  [8]: http://php.net/manual/en/book.apc.php
  [9]: http://php.net/token_get_all
  [10]: http://codepad.viper-7.com/tBx3TZ
  [11]: http://codepad.viper-7.com/p4aUGN