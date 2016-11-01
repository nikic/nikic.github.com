---
layout: post
title: Fast request routing using regular expressions
excerpt: This article describes a number of techniques to improve performance of regular expression based dispatch processes, as used in request routing or lexing.
---
Some time ago I stumbled on the [Pux][pux] routing library, which claims to implement a request router that is many
orders of magnitude faster than the existing solutions. In order to accomplish this, the library makes use of a PHP
extension written in C.

However, after a cursory look at the code I had the strong suspicion that the library was optimizing the wrong parts of
the routing process and I could easily get better performance without resorting to a C extension. This suspicion was
confirmed when I took a look at the benchmarking code and discovered that only the extremely *realistic* case of a
single route was tested.

To investigate the issue further I wrote a small routing library: [FastRoute][fast_route]. This library implements the
dispatch process that I will describe below. To give some perspective upfront, here are the results of a [small
benchmark][bench] against the Pux library:

    1 placeholder  | Pux (no ext) | Pux (ext) | FastRoute
    -----------------------------------------------------
    First route    | 0.17 s       | 0.13 s    | 0.14 s
    Last route     | 2.51 s       | 1.20 s    | 0.49 s
    Unknown route  | 2.34 s       | 1.10 s    | 0.34 s

    9 placeholders | Pux (no ext) | Pux (ext) | FastRoute
    -----------------------------------------------------
    First route    | 0.22 s       | 0.19 s    | 0.20 s
    Last route     | 2.65 s       | 1.78 s    | 0.59 s
    Unknown route  | 2.50 s       | 1.49 s    | 0.40 s

This micro benchmark makes use of one hundred routes and then dispatches against the first of those routes (best case),
the last one (worst case) and an altogether unknown route. This is done in two variations: Once using routes containing
only a single placeholder and then using routes with nine placeholders. The whole process was obviously iterated a few
thousand times.

Before getting to the actual content, let me clarify one last point: While on the surface this is an article about
routing, what I'm really talking about are regular expression based dispatch processes in general. In many ways this is
a continuation of my post on [lexing performance in PHP][lexing_performance].

The routing problem
-------------------

To make sure we're on the same page, lets first define what "routing" refers to. In the most practical form, it is the
process of taking a set of route definitions in a form similar to the following...

```php?start_inline=1
$r->addRoute('GET', '/user/{name}/{id:\d+}', 'handler0');
$r->addRoute('GET', '/user/{id:\d+}', 'handler1');
$r->addRoute('GET', '/user/{name}', 'handler2');
```

...and then dispatch a URI based on them:

```php?start_inline=1
$d->dispatch('GET', '/user/nikic/42');
// => provides 'handler0' and ['name' => 'nikic', 'id' => '42']
```

To bring this to a more abstract level, we'll dispense with the HTTP methods and any specific format for route
definitions. The only thing I'll be considering in this post is the dispatch stage - how the routes are parsed or the
dispatcher data is generated will not be covered.

So, what is the slow part of the routing process? In a massively overengineered system it is likely the general overhead
of instantiating a few dozen objects and calling a few hundred methods. Pux does a great job at reducing this overhead.
However, at a more fundamental level, the slow part of the dispatch process is sequentially going through a list of
dozens or hundreds or thousands of route expressions and matching them against the provided URI. How to make *this*
fast (or at least faster) is the topic of this post.

Combined regular expressions
----------------------------

The basic idea for optimizing this kind of problem is to avoid matching the regular expressions one by one and instead
compile them into one, big regular expression, so you need to perform only one match. Taking the routes from the
last example, this is how such a combined regex looks like:

    Individual regexes:

        ~^/user/([^/]+)/(\d+)$~
        ~^/user/(\d+)$~
        ~^/user/([^/]+)$~

    Combined regex:

        ~^(?:
            /user/([^/]+)/(\d+)
          | /user/(\d+)
          | /user/([^/]+)
        )$~x

The transformation is very simple: Basically you just need to OR all the individual expressions together. When matching
against this expression, how can you find out which of the routes matched? To figure this out, lets look at a sample
`preg_match` output:

```php?start_inline=1
preg_match($regex, '/user/nikic', $matches);
=> [
    "/user/nikic",   # full match
    "", "",          # groups from first route (empty)
    "",              # groups from second route (empty)
    "nikic",         # groups from third route (used!)
]
```

So the trick is to find the first non-empty entry in the `$matches` array (not counting the full match, of course). To
make use of this, you'll need an additional data structure that maps the `$matches` offset to the matched route (or
rather, the information associated with that route):

```php?start_inline=1
[
    1 => ['handler0', ['name', 'id']],
    3 => ['handler1', ['id']],
    4 => ['handler2', ['name']],
]
```

Here's a sample implementation of the whole process:

```php?start_inline=1
public function dispatch($uri) {
    if (!preg_match($this->regex, $uri, $matches)) {
        return [self::NOT_FOUND];
    }

    // find first non-empty match (skipping full match)
    for ($i = 1; '' === $matches[$i]; ++$i);

    list($handler, $varNames) = $this->routeData[$i];

    $vars = [];
    foreach ($varNames as $varName) {
        $vars[$varName] = $matches[$i++];
    }
    return [self::FOUND, $handler, $vars];
}
```

After the first non-empty offset `$i` is found and the associated data was looked up, the placeholder variables can be
populated by continuing to go through the `$matches` array and pairing the values with the variable names.

How well does this simple approach work? Here's a comparison with Pux (with C extension):

    1 placeholder  | Pux (ext) | GPB-NC
    -----------------------------------
    First route    | 0.13 s    | 0.20 s
    Last route     | 1.20 s    | 0.70 s
    Unknown route  | 1.10 s    | 0.16 s

    9 placeholders | Pux (ext) | GPB-NC
    -----------------------------------
    First route    | 0.19 s    | 0.41 s
    Last route     | 1.78 s    | 4.09 s
    Unknown route  | 1.49 s    | 0.30 s

GPB-NC stands for "Group position based, non-chunked" dispatching. The term should become more clear later in the post.
As you can see from the timings this approach provides pretty good performance in the single placeholder case. Of course
it can't beat a C extension implementation that matches correctly on the first route, however it is a good bit faster if
the last route matches (worst case) and does really, really well if the route doesn't match at all.

Things look less sunny if you consider the case with nine placeholders: In the worst case (last route matches) the
performance deteriorates to a point where it is more than twice as slow as the trivial approach. On the other hand the
"unknown route" case is still blazingly fast. How can this be?

The reason behind this (at least I assume so) is the sheer number of capturing groups that are involved in the compiled
regular expression: At 100 routes, with 9 placeholder each, you get 900 capturing groups. If the route is not known,
then `$matches` is not populated and the call is fast. If the first route matches, PCRE populates only the matches
associated with that route (i.e. 9 elements for the capturing groups and another one for the full match). However, if
the last route matches, PCRE must populate not only the matches of the last route, but also those of all preceding
routes (which would be a total of 100 * 9 + 1 = 901 elements).

So what we need to do is reduce the amount of capturing groups that need to be *populated*.

Group number reset
------------------

A little known feature of the PCRE regex syntax is the `(?| ... )` non-capturing group type. The difference between
`(?:` and `(?|` is that the latter will reset the group number in every branch it contains. To understand what this
means lets consider an example:

```php?start_inline=1
preg_match('~(?:(Sat)ur|(Sun))day~', 'Saturday', $matches)
=> ["Saturday", "Sat", ""]   # The last "" is not actually in the $matches array, but that's just
                             # an implementation detail. I'm writing it here to clarify the concept.

preg_match('~(?:(Sat)ur|(Sun))day~', 'Sunday', $matches)
=> ["Sunday", "", "Sun"]

preg_match('~(?|(Sat)ur|(Sun))day~', 'Saturday', $matches)
=> ["Saturday", "Sat"]

preg_match('~(?|(Sat)ur|(Sun))day~', 'Sunday', $matches)
=> ["Sunday", "Sun"]
```

If `(?:` is used PCRE will create a separate entry in `$matches` for the two capturing groups around `Sat` and `Sun`,
even though we know that only one of them can every match (i.e. be non-empty). Here both groups have distinct group
numbers. `(Sat)` is group 1 and `(Sun)` is group 2.

When `(?|` is used both groups share the same group number 1. Whether this corresponds to `(Sat)` or `(Sun)` depends
on which branch of the regular expression is taken.

This should give you an idea on how to solve our "too many capturing groups" problem. Just replace `(?:` with `(?|`:

    ~^(?|
        /user/([^/]+)/(\d+)
      | /user/(\d+)
      | /user/([^/]+)
    )$~x

However, now that the group number is reset we no longer have a means of detecting which of the routes actually matched.
Previously we determined this using the first non-empty offset in `$matches`. Due to the group number reset this will
always be `$matches[1]`, which gives us precious little information.

An idea that might occur to some, is to simply wrap every route in a named group and determine the match based on which
named group contains a value:

    ~^(?|
        (?<route1> /user/([^/]+)/(\d+) )
      | (?<route2> /user/(\d+) )
      | (?<route3> /user/([^/]+) )
    )$~x

However, this is not permitted by PCRE: Internally named groups are implemented by mapping the subpattern name to a
group number and then treating it as an ordinary, non-named group. Thus in the above regular expression `<route1>`,
`<route2>` and `<route3>` would all correspond to the same group number 1 - which doesn't make sense and as such is not
allowed.

A more fruitful approach is to consider the *number* of matched groups. With the three routes from above the first one
would produce a `$matches` array of size 3 (the full match plus two capturing groups), while the two over routes result
in a `$matches` array of size 2.

Right now using the `$matches` length to determine the matched route is ambiguous, but we can easily rectify this by
appending dummy groups:

    ~^(?|
        /user/([^/]+)/(\d+)
      | /user/(\d+)()()
      | /user/([^/]+)()()()
    )$~x

Now the first route has two groups (three `$matches`), the second route three groups (four `$matches`) and the third
route has four groups (five `$matches`). As such the dispatch can be performed using the following lookup table:

```php?start_inline=1
[
    3 => ['handler0', ['name', 'id']],
    4 => ['handler1', ['id']],
    5 => ['handler2', ['name']],
]
```

Here's a sample implementation of this strategy:

```php?start_inline=1
public function dispatch($uri) {
    if (!preg_match($this->regex, $uri, $matches)) {
        return [self::NOT_FOUND];
    }

    list($handler, $varNames) = $this->routeData[count($matches)];

    $vars = [];
    $i = 0;
    foreach ($varNames as $varName) {
        $vars[$varName] = $matches[++$i];
    }
    return [self::FOUND, $handler, $vars];
}
```

Let's see how this compares to the previous approach:

    1 placeholder  | Pux (ext) | GPB-NC | GCB-NC
    --------------------------------------------
    First route    | 0.13 s    | 0.20 s | 0.60 s
    Last route     | 1.20 s    | 0.70 s | 1.06 s
    Unknown route  | 1.10 s    | 0.16 s | 0.56 s

    9 placeholders | Pux (ext) | GPB-NC | GCB-NC
    --------------------------------------------
    First route    | 0.19 s    | 0.41 s | 0.65 s
    Last route     | 1.78 s    | 4.09 s | 0.96 s
    Unknown route  | 1.49 s    | 0.30 s | 0.54 s

GPB is the previous Group *Position* Based approach and GCB the new Group *Count* Based method. Both are still
Non-Chunked. It should be clear that the group count based approach resolves the issue of catastrophic performance quite
nicely (0.96s against 4.09s), however it degrades performance in pretty much all other cases. For example the time for
matching the first route in the single-placeholder case has tripled.

What's the reason for this? My guess would be that the slowdown is caused by the sheer size of the regular expression
and the number of groups it contains (even if only a small fraction of them is populated). To put things into
perspective: For 100 routes we need to generate 99 * 100/2 = 4950 dummy capturing groups. This corresponds to
4950 * 2 = 9900 additional bytes of regular expression, i.e. nearly 10KB.

You can take a look at how the [generated regular expression][small_regex] looks like (for 100 routes with nine
placeholders). Here's the end of the regex:

    |/cv/([^/]+)/([^/]+)/([^/]+)/([^/]+)/([^/]+)/([^/]+)/([^/]+)/([^/]+)/([^/]+)
    ()()()()()()()()()()()()()()()()()()()()()()()()()()()()()()()()()()()()()()
    ()()()()()()()()()()()()()()()()()()()()()()()()()()()()()()()()()()()()()()
    ()()()()()()()()()()()()()()()()()()()()()()())$~

(This is how LISP code looks like.)

Chunked regexes
---------------

As the number of dummy groups increases quadratically with the number of routes, this approach evidently doesn't scale
very well. One way to reduce the amount of dummies is to split the regular expression into two parts: One matching the
first 50 rules, the other matching the second 50. Each regex will need only 49 * 50/2 = 1225 dummy groups, i.e. 2450
will be needed in total, which is a good bit less than 4950. If the routes are split into ten chunks of ten routes each,
every chunk will need 9 * 10/2 = 45 dummy groups, which corresponds to a total of 450.

The implementation for the chunked regime stays about the same, only with an additional `foreach` loop:

```php?start_inline=1
public function dispatch($uri) {
    foreach ($this->regexes as $i => $regex) {
        if (!preg_match($regex, $uri, $matches)) {
            continue;
        }

        list($handler, $varNames) = $this->routeData[$i][count($matches)];

        $vars = [];
        $i = 0;
        foreach ($varNames as $varName) {
            $vars[$varName] = $matches[++$i];
        }
        return [self::FOUND, $handler, $vars];
    }

    return [self::NOT_FOUND];
}
```

Here are the timing results comparing the non-chunked and 10-chunked variants:

    1 placeholder  | GCB-NC | GCB-10C
    ---------------------------------
    First route    | 0.60 s | 0.14 s
    Last route     | 1.06 s | 0.49 s
    Unknown route  | 0.56 s | 0.34 s

    9 placeholders | GCB-NC | GCB-10C
    ---------------------------------
    First route    | 0.65 s | 0.20 s
    Last route     | 0.96 s | 0.59 s
    Unknown route  | 0.54 s | 0.40 s

As you can see the approach using chunks of size 10 beats the non-chunked variants in every case, sometimes being a
factor of two or three faster. The number 10 came up after a few quick measurements with different chunk sizes. However
I didn't test this rigorously and the optimum is likely just somewhere "in the area of" 10.

Remember how the main problem with the group *position* based approach was the excessive number of groups that have to
be populated? That issue should be solvable using chunking as well. The code modifications are about the same for GPB
as they are for GCB, so I won't repeat them. Here are the timing results:

    1 placeholder  | GPB-NC | GPB-10C
    ---------------------------------
    First route    | 0.20 s | 0.13 s
    Last route     | 0.70 s | 0.47 s
    Unknown route  | 0.16 s | 0.31 s

    9 placeholders | GPB-NC | GPB-10C
    ---------------------------------
    First route    | 0.41 s | 0.18 s
    Last route     | 4.09 s | 0.81 s
    Unknown route  | 0.30 s | 0.41 s

Clearly this resolves the issue with the catastrophic many placeholders/last route performance, while making the first
route matches faster as well. Only the "unknown route" cases slow down, because those weren't impacted by the number of
capturing groups in the first place, but are impacted by having to match multiple regular expressions.

Lastly, let's compare group position based and the group count based approach side-by-side, each using chunks of size
10:

    1 placeholder  | Pux (ext) | GPB-10C | GCB-10C
    ----------------------------------------------
    First route    | 0.13 s    | 0.13 s  | 0.14 s
    Last route     | 1.20 s    | 0.47 s  | 0.49 s
    Unknown route  | 1.10 s    | 0.31 s  | 0.34 s

    9 placeholders | Pux (ext) | GPB-10C | GCB-10C
    ----------------------------------------------
    First route    | 0.19 s    | 0.18 s  | 0.20 s
    Last route     | 1.78 s    | 0.81 s  | 0.59 s
    Unknown route  | 1.49 s    | 0.41 s  | 0.40 s

GPB is marginally faster for the single placeholder case, but GCB is a good bit faster for the many placeholders/last
route case. As such, GCB-10C is the dispatch method implemented by [FastRoute][fast_route]. (A GPB-10C implementation is
also present in the code-base, but not used by default.)

Conclusions
-----------

There are a few technical takeaways from this post: The combined regex approach works great for a small number of
regular expressions, but is suboptimal if many regular expressions are involved and disastrous if there are many
capturing groups per rule. Performance can be improved by only combining chunks of approximately ten expressions. This
both improves performance in general and resolves the issue with particularly bad performance in the many-groups case.

The alternative group count based approach doesn't make much of a difference anymore once you start using chunking. It
is marginally worse with fewer capturing groups and somewhat better with many groups. For routing in particular, where
typically more than one placeholder is involved per route, the group count based approach seems preferable. For lexing
on the other hand, where usually only one capturing group is used per token, the position based approach wins.

That much for the technical conclusions on regex dispatch. I'd like to mention two more things though:

Firstly, while writing a PHP extension is obviously a lot of fun, it is unlikely to be particularly beneficial in terms
of performance unless the code involves computations in tight loops. Porting "normal" components like routers to C is
usually a big waste of time. You can get much better results by doing a few small improvements on the algorithmic side.
For the same reason I'm not a big fan of things like Phalcon, i.e. PHP frameworks written as a C extension. You can get
good performance by using an appropriate application design (that is not massively overengineered) and still retain the
flexibility of keeping everything in PHP.

Lastly, I should also mention that for the majority of web applications, especially if they are making use of any of
the big frameworks, routing will not be a bottleneck. The time it takes to route a request will be lost in the vast
complexity of the remaining system. However, there are cases where routing performance *is* quite relevant and which is
why I took the time to write this post. For example a precursor of the routing library described here is being used in
a (closed source) web server written in PHP (by smarter people than myself), which handles something like fifty thousand
requests per second . If you tried to put the Symfony router behind such a server, it would totally cripple your
performance.

  [pux]: https://github.com/c9s/Pux
  [fast_route]: https://github.com/nikic/FastRoute
  [bench]: https://gist.github.com/nikic/9049180
  [lexing_performance]: http://nikic.github.io/2011/10/23/Improving-lexing-performance-in-PHP.html
  [small_regex]: https://gist.github.com/nikic/8464660