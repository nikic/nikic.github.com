---
layout: post
title: "Don't be STUPID: GRASP SOLID!"
excerpt: "Introducing STUPID: Singleton, Tight Coupling, Untestability, Premature Optimization, Indescriptive Naming, Duplication."
---
Ever heard of [SOLID][1] code? Probably: It is a term describing a collection of design principles
for "good code" that was coined by Robert C. Martin (aka ["uncle bob"][2]), our beloved evangelist
of clean code.

Programming is full of acronyms like this. Other examples are [DRY][3] (Don't Repeat Yourself!) and
[KISS][4] (Keep It Simple, Stupid!). But there obviously are many, many more...

So, why not approach the problem from the other side for once? Looking at what makes up **bad**
code.

Sorry, but your code is STUPID!
-------------------------------

Nobody likes to hear that their code is stupid. It is offending. Don't say it. But honestly: Most
of the code being written around the globe is an unmaintainable, unreusable mess.

What characterizes such code? What makes code **STUPID**?

 * ***S***ingleton
 * ***T***ight coupling
 * ***U***ntestability
 * ***P***remature Optimization
 * ***I***ndescriptive Naming
 * ***D***uplication

Do you agree with this list? Yes? Great. No? I'll explain the individual points in greater detail
in the following, so you can better understand why exactly those patterns were chosen.

Singleton
---------

{% highlight php startinline %}
class DB {
    private static $instance;

    public static function getInstance() {
        if (!isset(self::$instance)) {
            self::$instance = new self;
        }

        return self::$instance;
    }

    final private function __construct() { /* something */ }
    final private function __clone() { }

    /* actual methods here */
}
{% endhighlight %}

The above is the typical database access implementation you will find in pretty much any PHP
tutorial. I actually used something similar to this myself not too long ago.

Now you wonder: What's wrong with that? You can easily access the DB from anywhere using
`DB::getInstance()` and the code also ensures that you only have one database connection open at a
time. What could be bad about that?

Well, yeah, I thought that too ^^ "I only need one connection." When the application grew larger it
turned out that I actually needed a second connection to a different database. And that's where the
mess began. I changed the singleton to have a `->getSecondInstance()`, thus converting the singleton
to a .. erm .. dupleton? Instead I should have realized that the database connection *simply isn't a
singleton* and thus can't be sanely implemented as one. The same also applies to any other use of
singletons that you will find. The request object surely is a singleton! Ever heard of subrequests?
But the logger definitely is! Never wanted to log different things differently?

And that was only *one* issue. Another important issue is that by using `DB::getInstance()` in your
code you are binding your code to the `DB` classname. This means: You can't extend the `DB` class.
At some point I wanted to optionally log query performance data to APC. But the tight coupling to
the class name did not allow me to. If my application had used [dependency injection][8] at that
time I could have easily extended `DB` and passed the new instance. But the singleton prevented.
What I did instead was somethink looking roughly like this:

{% highlight php startinline %}
// original DB class
class _DB { /* ... */ }

// extending class
class DB extends _DB { /* ... */ }
{% endhighlight %}

One word: Ugly. One could add some other words like Hacky, Unmaintainable, Crap. Or STUPID.

One last point to consider: Remember when I said "You can easily access the DB from anywhere using
`DB::getInstance()`". Well, actually that's a bad thing too. Read "from anywhere" as "globally" and
translate to "A singleton is a global variable with a fancy name." When you learned PHP you were
probably told that it's evil to use the `global` keyword. But by using a singleton you are doing
just that: creating global state. This creates non-obvious dependencies and thus makes your app hard
to reuse and test.

Tight coupling
--------------

You can actually generalize the Singleton issue to `static` methods and properties in general.
Whenever you are writing `Foo::bar()` in your code you are tightly coupling your code to the `Foo`
class. This makes extending `Foo`s functionality impossible and thus makes your code hard to reuse
and hard to test.

Similarly most other plain uses of class names are code smell too. This also applies to the `new`
operator:

{% highlight php startinline %}
class House {
    public function __construct() {
        $this->door   = new Door;
        $this->window = new Window;
    }
}
{% endhighlight %}

How would you replace the door or the window in your house now? Simple: You can't. As a good
developer you will obviously find some dirty hack that *does* allow you to somehow replace the
door or a window. But why not simply write this instead:

{% highlight php startinline %}
class House {
    public function __construct(Door $door, Window $window) { // Door, Window are interfaces
        $this->door   = $door;
        $this->window = $window;
    }
}
{% endhighlight %}

This way one can easily create houses with different doors and windows. The code is easy to extend,
easy to reuse and easy to test. What could you want more?

The above code is actually "dependency injection" in a nutshell. Many people associate DI with
some Dependency Injection Container (DIC) like Symfony's, but the actual concept of DI is so much
simpler.

Untestability
-------------

[Unit testing][5] is important. If you don't test your code, you are bound to ship broken code. But
still most people don't properly cover their code with tests. Why? Mostly because their code is
*hard to test*. What makes code hard to test? Mainly the previous point: Tight coupling. Unit
testing - it might seem obvious - tests units of code (usually classes). But how can you test
individual classes if they are tightly coupled? You probably can somehow, with even more dirty
hacks. But people usually don't do those efforts and just leave their code untested and broken.

Whenever you don't write unit tests because you "don't have time" the real cause probably is that
your code is bad. If your code is good you can test it in no time. Only with bad code unit testing
becomes a burden.

Premature Optimization
----------------------

Here is a code snippet from an old version of a website I wrote:

{% highlight php startinline %}
if (isset($frm['title_german'][strcspn($frm['title_german'], '<>')])) {
    // ...
}
{% endhighlight %}

Guess what it does!

How long did it take you to realize that this code does nothing more than check whether the German
title contains the `<` or `>` character? Did you realize it at all?

Let me explain: If `<` and `>` are both not contained, [`strcspn`][7] will return the length of the
string. So the code will basically be `isset($str[strlen($str)])`. As the maximum offset set in a
string is the length minus one, this will return `false`. If one of the characters is contained
the function will return a number smaller than the length of the string and thus the whole
expression will be `true`.

So why did I write this unintelligible piece of code? Why didn't I write this instead:

{% highlight php startinline %}
if (strlen($frm['title_german']) == strcspn($frm['title_german'], '<>'))) {
    // ...
}
{% endhighlight %}

Because some days ago I read that `isset` is so much faster than `strlen`... But that code still is
not particularly intelligible, because you need to know the exact semantics of the `strcspn`
function (which probably most PHP programmers do not). So why not just write this:

{% highlight php startinline %}
if (preg_match('(<|>)', $frm['title_german'])) {
    // ...
}
{% endhighlight %}

Because I read that regex is slow... (Which by the way is mostly a lie: regular expressions are
much faster and much more powerful than you think.)

So, what did I gain from these "optimizations", apart from unreadable code? Nothing. Even now, when
this site attracts approximately fourty million pageviews every month (at the time I wrote this code
it was faaar less), even now this micro-optimization won't make *any* difference. Simply because it
is not the bottleneck of the application. The actual bottleneck is a tripple `JOIN` in the
most-frequently accessed controller (the bottleneck of your application probably is similar).

You will find many more micro-optimization tips like that on the internet. Like "use single quotes,
they are faster". Don't listen. Most of the advice is plain wrong and even if it isn't, it won't
make your code measurably faster, it'll only waste your time.

Indescriptive Naming
--------------------

By the way, do you know what the [`strpbrk`][6] PHP function does? No? You didn't even know it
existed? I'm not surprised. Nobody who wants to search a string for a list of characters looks for
a function named `strpbrk`. Where did that name come from anyways? This function was inherited from
C and its name stands for "string pointer break". Yeah, really nice, especially in a language that
doesn't actually have pointers (I mean PHP, not C).

Oh, and reading the code snippet in the above section, did you know what the [`strcspn`][7]
function does off the top of your head? No? Again, I'm not surprised. It's short for "string
complement span", just in case you didn't know.

The lesson from this: Please, name your classes, methods, variables properly, so that people
actually know what you mean. I'm not arguing about variables like `$i`, those are short, but still
self-explanatory. The problem is functions like the ones named above. Functions like `strpbrk` or
variables like `$yysstk` may be obvious the author, but also *only* to the author.

Duplication
-----------

I think everybody agrees that code that is particularly short, concise and pertinent is considered
especially elegant (oh, and I *don't* mean the Perl/Ruby style "short"). On the other hand most
people would consider long and redundant code rather ugly. This is what the aforementioned DRY
(Don't Repeat Yourself!) and KISS (Keep It Simple, Stupid!) design principle want to teach you.

So where does code duplication come from? Programmers are lazy animals, so it would be only natural
to type as little code as possible. Still duplication prevails.

I think the most common reason for duplication is following the second STUPID principle: Tight
Coupling. If your code is tightly coupled, you just can't reuse it. And here comes your duplication.

Don't be STUPID: GRASP SOLID!
-----------------------------

So what are the alternatives to writing STUPID code? Simple, [SOLID][1] and [GRASP][9].

SOLID deciphers to:

 * ***S***ingle responsibility principle
 * ***O***pen/closed principle
 * ***L***iskov substitution principle
 * ***I***nterface segregation principle
 * ***D***ependency inversion principle

GRASP stands for General Responsibility Assignment Software Principles, which are the following:

 * Information Expert
 * Creator
 * Controller
 * Low Coupling
 * High Cohesion
 * Polymorphism
 * Pure Fabrication
 * Indirection
 * Protected Variations

Happy coding and a happy new year to you!

PS: If you wonder where STUPID comes from: The idea [came up][13] on the [PHP chatroom on
StackOverlow][10] and the individual components were formed by edorian ([StackOverflow][10],
[Github][11], [Twitter][12]), [James][14] and me. Gordon ([StackOverflow][16], [Github][17],
[Twitter][18]) [came up][15] with the title of this blog post.

  [1]: http://en.wikipedia.org/wiki/SOLID_%28object-oriented_design%29
  [2]: http://cleancoder.posterous.com/
  [3]: http://en.wikipedia.org/wiki/Don%27t_repeat_yourself
  [4]: http://en.wikipedia.org/wiki/KISS_principle
  [5]: http://en.wikipedia.org/wiki/Unit_testing
  [6]: http://php.net/strpbrk
  [7]: http://php.net/strcspn
  [8]: http://www.martinfowler.com/articles/injection.html
  [9]: http://en.wikipedia.org/wiki/GRASP_%28object-oriented_design%29
  [10]: http://chat.stackoverflow.com/rooms/11/php
  [11]: https://github.com/edorian
  [12]: http://twitter.com/#!/__edorian
  [13]: http://chat.stackoverflow.com/transcript/11?m=2182205#2182205
  [14]: http://stackoverflow.com/users/336242/james-butler
  [15]: https://twitter.com/#!/go_oh/status/150187219020300288
  [16]: http://stackoverflow.com/users/208809/gordon
  [17]: https://github.com/gooh
  [18]: http://twitter.com/#!/go_oh