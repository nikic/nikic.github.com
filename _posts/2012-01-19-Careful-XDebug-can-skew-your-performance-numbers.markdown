---
layout: post
title: "Careful: XDebug can skew your performance numbers"
excerpt: In some cases XDebug can significantly skew your benchmarking and profiling numbers. So make sure that you do measurements without it.
---
[XDebug][1] is a great tool and a really big aid in debugging.

But running it incurs a certain overhead on many common operations - like calling functions. Due to
this having XDebug enabled can skew your benchmarking and profiling numbers. ("Enabled" here means
just enabled, no debugging going on or such.)

This has an important implication: If you optimize a script while XDebug is enabled it can actually
turn out that it gets **slower** on your production server (as you *hopefully* don't have XDebug
enabled there).

An example: "Optimizing" a lexer wrapper
----------------------------------------

For my [PHP parser][2] I need a smaller wrapper around PHP's [`token_get_all`][3] function. This
wrapper (called `PHPParser_Lexer`) only does nothing more than a little bit of normalization,
filtering and mapping on the tokens.

There are basically two ways how this wrapper can work:

 1. For every source code to be parsed a new `Lexer` instance is created and passed into the parser,
    which then does multiple calls to `->lex()` to fetch one token at a time.
 2. A `Lexer` is creates once and reused for all source codes. Always only a single call to `->lex()`
    is made, which then returns all the tokens at once.

So, what I did is implement both approaches and see how much time is spent lexing the whole Symfony
tree. I got the following results:

    Scenario 1 on 5.3.8 with XDebug enabled: 16.939509868622 seconds
    Scenario 2 on 5.3.8 with XDebug enabled:  6.104107856751 seconds

That's an impressive saving of 10 seconds (aka 60%) right there! is what I though.

But then I tried the same thing without XDebug:

    Scenario 1 on 5.3.8 with XDebug disabled: 3.656718969345 seconds
    Scenario 2 on 5.3.8 with XDebug disabled: 3.195748090744 seconds

Now that XDebug is disabled the numbers are drastically lower and also much closer. The 17 seconds
from above are only 3.7 seconds now. The time for the 6 seconds case halved.

Why is there such a drastic change on the first number, and a less pronounced one on the second? The
first scenario needs approximately 70000 `->lex()` calls, whereas the second one only needs 2000.
XDebug adds a quite large overhead to function calls, so the scenario with 70000 calls is impacted
much more.

Finally, let's look at the numbers for PHP 5.4.0 without XDebug:

    Scenario 1 on 5.4.0 with XDebug disabled: 2.8629820346832 seconds
    Scenario 2 on 5.4.0 with XDebug disabled: 3.0883010864258 seconds

And here the numbers are now actually turned around. PHP 5.4 got some optimizations that made the
seconds variant slower.

Another example: Micro benchmarking
-----------------------------------

As pointless as they are, people still love micro benchmarks, I do too. You'll find lots of blog
posts comparing the performance of something vs. something else. The problem is: Most of these are
done on development machines, with XDebug enabled.

As an example this recent [blog post about the performance of exceptions][4] will serve. Here are
the numbers the author measured for exceptions on PHP 5.3/5.4 for a certain script:

    PHP 5.3: 1.1479668617249
    PHP 5.4: 0.1864490332

Looking at those you might say: Damn, that's a pretty impressive improvement!

Well, not really: Seeing these numbers I immediately had the suspicion that the author actually
measured two different things: 5.3 with XDebug and 5.4 without it.

So I tested both versions without XDebug and got 0.14 seconds for PHP 5.3 and 0.09 seconds for PHP
5.4, i.e. a much smaller difference. You can find more detailed results in [edorian's blog post][5]
on the topic.

I think many micro benchmarking posts you'll find on the net are affected by this. I was also
trapped by this multiple times (e.g. see [this question I asked about function call performance][6]).

Conclusion
----------

The conclusion from the above is: If you do benchmarking, benchmark on a machine that's configured
like your production machine. Otherwise you might actually be anti-optimizing.

Also: Low level optimizations like function inlining may actually measurably improve performance on
your development machine (like in my case), but have little effect in the production environment.
So, you can safely go for clean code with small functions and not fear about performance
degradation.

  [1]: http://xdebug.org/
  [2]: https://github.com/nikic/PHP-Parser
  [3]: https://php.net/token_get_all
  [4]: http://gonzalo123.wordpress.com/2012/01/16/checking-the-performance-of-php-exceptions/
  [5]: http://edorian.posterous.com/never-trust-other-peoples-benchmarks-a-recent
  [6]: https://stackoverflow.com/q/3691625/385378