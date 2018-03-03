---
layout: post
title: Understanding PHP's internal function definitions (PHP's Source Code for PHP Developers - Part 2)
excerpt: The second part of the "PHP's Source Code for PHP Developers" series, covering how to find functions in the PHP source code and how they are structured.
---
Welcome to the second part of the "PHP's Source Code For PHP Developers" series.

In the [previous part][1] ircmaxell explained where you can find the PHP source code and how it is basically structured
and also gave a small introduction to C (as that's the language PHP is written in). If you missed that post, you
probably should read it before starting with this one.

What we'll cover in this article is locating the definitions of internal functions in the PHP codebase, as well as
understanding them.

How to find function definitions
--------------------------------

For a start, let's try to find out how the `strpos` function is defined.

The first thing to try, is to go to the [PHP 5.4 source code root][2] and type `strpos` into the search box at the top
of the page. [The result][3] will be a huge listing of `strpos` occurrences in the PHP source code.

As this doesn't really help us much, we use a little trick: Instead of searching for just `strpos`, we [search for
`"PHP_FUNCTION strpos"`][4] instead (don't forget the quotes, they are important).

Now we are left with only two entries:

    /PHP_5_4/ext/standard/
        php_string.h 48   PHP_FUNCTION(strpos);
        string.c     1789 PHP_FUNCTION(strpos)

First thing to notice is that both occurrences are in the `ext/standard` folder. This is exactly where one would expect
to find them, as the `strpos` function (together with pretty much all other string, array and file functions) is part of
the `standard` extension.

Now open both links in new tabs and see what code hides behind them.

You'll find that the first link leads you to the [`php_string.h` file][5], which is full of code looking like this:

    // ...
    PHP_FUNCTION(strpos);
    PHP_FUNCTION(stripos);
    PHP_FUNCTION(strrpos);
    PHP_FUNCTION(strripos);
    PHP_FUNCTION(strrchr);
    PHP_FUNCTION(substr);
    // ...

This is exactly how a typical header file (a file ending in `.h`) looks like: A plain list of functions which are
defined elsewhere. We aren't really interested in this, as we already know what we're looking for.

The second link is much more interesting: It leads to the [`string.c` file][6], which contains the actual source code of
the function.

Before I'll walk you through the code step by step, I'd recommend you to try and understand the function by
yourself. It's a really simple function and most things should be clear even if you don't know the exact details.

The skeleton of a PHP function
------------------------------

All PHP functions share the same basic structure. At the top there are a few variable declarations, then there is a
`zend_parse_parameters` call, then comes the main logic, with `RETURN_***` and `php_error_docref` calls intermixed.

So, let's start with the variable declarations:

    zval *needle;
    char *haystack;
    char *found = NULL;
    char  needle_char[2];
    long  offset = 0;
    int   haystack_len;

The first line declares `needle` as being a pointer to a `zval`. A `zval` is PHP's internal representation of an
arbitrary PHP value. How exactly it looks will be the subject of the next post.

The second line declares `haystack` as a pointer to a character. At this point you'll have to remember that in C, arrays
are represented by pointers to their first value. I.e. the `haystack` will point to the first character of the
`$haystack` string you passed in. Then `haystack + 1` will point to the second character, `haystack + 2` to the third,
and so on. So one could read in the whole string by always incrementing the pointer by one.

The problem arising here is that PHP has to know when the string ends. Otherwise it would always keep incrementing the
pointer without ever stopping. In order to deal with this, PHP also stores an explicit length, here in the
`haystack_len` variable.

The last declaration of interest to us at this point is the `offset` variable, which will be used to store the third
parameter of the function: the offset to start searching at. It is declared as a `long`, which is an integer datatype,
just like `int`. The difference between those two is not of importance here, but you should know that PHP integers are
stored in `long`s and string lengths are stored in `int`s.

Now let's look at the next three lines:

    if (zend_parse_parameters(ZEND_NUM_ARGS() TSRMLS_CC, "sz|l", &haystack, &haystack_len, &needle, &offset) == FAILURE) {
        return;
    }

What these lines basically do, is take the parameters that were passed to the function and put them into the variables which were declared above.

The first argument to the function is the number of arguments passed. This number is provided by the `ZEND_NUM_ARGS()`
macro.

The next argument is the `TSRMLS_CC` macro, which is kind of an idiosyncrasy of PHP. You'll find this strange macro
scattered across pretty much the whole PHP code base. It is part of the Thread Safe Resource Mananger (TSRM), which
ensures that PHP doesn't mix up variables between multiple threads. This is unimportant to us, so whenever you see
`TSRMLS_CC` (or `TSRMLS_DC`) in the code, just ignore it. (A strangeness which you might have noticed, is that there is
no comma before this "argument". This has to do with the fact that depending on whether or not you are using a
thread-safe build, the macro will either evaluate to nothing or to `, tsrm_ls`. So basically the comma is part of the
macro.)

Now comes the important stuff: The `"sz|l"` string specifies which parameters the function accepts:

    s  // first parameter is a *s*tring
    z  // second parameter is a *z*val (an arbitrary value)
    |  // the following parameters (here just one) are optional
    l  // third parameter is a *l*ong (an integer)

There are more type specifiers than `s`, `z` and `l`, but most should be clear from the character. For example `b` is a
**b**oolean, `d` is a **d**ouble (floating point number), `a` is an **a**rray, `f` is a callback (**f**unction) and `o`
is an **o**bject.

The remaining arguments `&haystack, &haystack_len, &needle, &offset` specify the variables to put the values of the arguments into.
As you can see, they are all passed by reference (`&`), which means that not the variables themselves are passed, but
pointers to them.

After this call `haystack` will contain the haystack string, `haystack_len` the length of that string, `needle` the
needle value and `offset` the starting offset.

Additionally the function is checked for `FAILURE` (which happens if you try to pass invalid arguments to the function,
e.g an array to a string parameter). In this case `zend_parse_parameters` will throw a warning and the code of the
function just `return`s (which will eventually return `null` to the userland PHP code).

So after the parameters are parsed, the main function body starts:

    if (offset < 0 || offset > haystack_len) {
        php_error_docref(NULL TSRMLS_CC, E_WARNING, "Offset not contained in string");
        RETURN_FALSE;
    }

What this code does is pretty obvious. If the offset is out of bounds an `E_WARNING` level error is thrown through
`php_error_docref` and then false is returned using the `RETURN_FALSE` macro.

`php_error_docref` is the error function you'll mainly find in extensions (i.e. the `ext` folder). The name comes from
the fact that it emits a reference to the documentation in the error message (you know, the one that never works...).
Additionally there is the `zend_error` function, which is mainly used by the Zend Engine, but also occurs in extension
code from time to time.

Both functions use [`sprintf`][7]-like formatting, thus error messages can contain placeholders, which are then filled
using the following arguments. Here is an example:

    php_error_docref(NULL TSRMLS_CC, E_WARNING, "Failed to write %d bytes to %s", Z_STRLEN_PP(tmp), filename);
    // %d is filled with Z_STRLEN_PP(tmp)
    // %s is filled with filename

Let's proceed in the code:

    if (Z_TYPE_P(needle) == IS_STRING) {
        if (!Z_STRLEN_P(needle)) {
            php_error_docref(NULL TSRMLS_CC, E_WARNING, "Empty delimiter");
            RETURN_FALSE;
        }

        found = php_memnstr(haystack + offset,
                            Z_STRVAL_P(needle),
                            Z_STRLEN_P(needle),
                            haystack + haystack_len);
    }

The first five lines should be clear: This branch is only executed if the `needle` is a string and an error is thrown if
it is empty. Then comes the interesting part: `php_memnstr` is called, which is the function doing the main work. As
always you can click on the function name to see its source code.

`php_memnstr` returns the pointer to the first occurrence of the needle in the haystack (that's why the `found` variable
is declared as `char *`, i.e. a pointer to character). From this the offset can be easily computed by subtracting the
two pointers, as can be seen at the end of the function:

     RETURN_LONG(found - haystack);

Finally, let's look at the branch which is taken when the `needle` is not a string:

    else {
        if (php_needle_char(needle, needle_char TSRMLS_CC) != SUCCESS) {
            RETURN_FALSE;
        }
        needle_char[1] = 0;

        found = php_memnstr(haystack + offset,
                            needle_char,
                            1,
                            haystack + haystack_len);
    }

I'll just quote what this does from the manual: "If needle is not a string, it is converted to an integer and applied as
the ordinal value of a character." This basically means that instead of writing `strpos($str, 'A')` you could also write
`strpos($str, 65)`, because the ordinal value of `A` is `65`.

If you look up at the variable declarations, you'll see that `needle_char` is declared as `char needle_char[2]`, i.e. a
string with two characters. `php_needle_char` will put the actual character (in our example the `A`) into
`needle_char[0]`. Then the `strpos` code will set `needle_char[1]` to `0`. The reason behind this is that in C, strings
are zero-terminated, i.e. the last character is set to NUL (the character with the ordinal value `0`). In the context of
PHP this doesn't make much sense, as PHP stores an explicit length for all strings (so it does not need zero-termination
to find the end of a string), but this still is done in order to ensure compatibility with the C functions used
internally by PHP.

Zend functions
--------------

I'm getting tired of `strpos`, so lets try to find another function: `strlen`. We'll do this using our usual approach:

Starting from the [PHP 5.4 source code root][2] try to search for `strlen`.

You'll see lots of unrelated uses of the function, so instead search for `"PHP_FUNCTION strlen"`. While doing so, you'll
notice something strange though: There won't be any results.

The reason is that `strlen` is one of the few functions, which is not defined by an extension, but by the Zend Engine
itself. In such cases the function is not defined as `PHP_FUNCTION(strlen)`, but as `ZEND_FUNCTION(strlen)`. Thus we
also have to [search for `"ZEND_FUNCTION strlen"`][8] instead.

As we already know, we have to click on the entry without a semicolon `;` at the end to get to the source code. This
leads us to the following definition in [`Zend/zend_builtin_functions.c`][9]:

    ZEND_FUNCTION(strlen)
    {
        char *s1;
        int s1_len;

        if (zend_parse_parameters(ZEND_NUM_ARGS() TSRMLS_CC, "s", &s1, &s1_len) == FAILURE) {
            return;
        }

        RETVAL_LONG(s1_len);
    }

I don't think that I have to further comment on this, as the function is so simple.

Methods
-------

We'll cover how classes and objects work in more detail in a different post, but as a small peek ahead: You can search
for class methods by typing `ClassName::methodName` into the search. As an example, try to
[search for `SplFixedArray::getSize`][10].

In the next part
----------------

The next part will again be published on [ircmaxell's blog][11]. It will cover what `zval`s are, how they work and how
they are used in the source code (all those `Z_***` macros...)

  [1]: https://blog.ircmaxell.com/2012/03/phps-source-code-for-php-developers.html
  [2]: https://lxr.room11.org/xref/php-src@5.6
  [3]: https://lxr.room11.org/search?project=php-src@5.6&q=strpos
  [4]: https://lxr.room11.org/search?project=php-src@5.6&q="PHP_FUNCTION+strpos"
  [5]: https://lxr.room11.org/xref/php-src@5.6/ext/standard/php_string.h#48
  [6]: https://lxr.room11.org/xref/php-src@5.6/ext/standard/string.c#1789
  [7]: https://php.net/sprintf
  [8]: https://lxr.room11.org/search?project=php-src@5.6&q="ZEND_FUNCTION+strlen"
  [9]: https://lxr.room11.org/xref/php-src@5.6/Zend/zend_builtin_functions.c#478
  [10]: https://lxr.room11.org/search?project=php-src@5.6&q=SplFixedArray::getSize
  [11]: https://blog.ircmaxell.com/
