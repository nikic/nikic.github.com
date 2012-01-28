---
layout: post
title: htmlspecialchars() improvements in PHP 5.4
excerpt: There is some nice new stuff for htmlspecialchars() in PHP 5.4, which hasn't yet got the attention it deserves.
---
There has been lots of buzz about many of the new features in PHP 5.4, like the traits support, the
short array syntax and all those other syntax improvements.

But one set of changes that I think is particularly important was largely overlooked: For PHP 5.4
cataphract ([Artefacto][1] on StackOverflow) heroically rewrote large parts of `htmlspecialchars`
thus fixing various quirks and adding some really nice new features.

(The changes discussed here apply not only to [`htmlspecialchars`][2], but also to the related
[`htmlentities`][3] and in parts to [`htmlspecialchars_decode`][4], [`html_entity_decode`][5] and
[`get_html_translation_table`][6].)

Here a quick summary of the most important changes:

 * UTF-8 as the default charset
 * Improved error handling (`ENT_SUBSTITUTE`)
 * Doctype handling (`ENT_HTML401`, ...)

UTF-8 as the default charset
----------------------------

As you hopefully know the third argument for `htmlspecialchars` is the character set. Thing is: Most
people just leave that argument out, thus falling back to the default charset. This default
charset was ISO-8859-1 before PHP 5.4 and as such did not match the UTF-8 encoding most people use.
PHP 5.4 fixes this by making UTF-8 the default.

Improved error handling
-----------------------

Error handling in `htmlspecialchars` before PHP 5.4 was ... uhm, let's call it "unintuitive":

If you passed a string containing an "invalid code unit sequence" (which is Unicode slang for
"not encoded correctly") `htmlspecialchars` would return an empty string. Well, okay, so far so
good. The funny thing was that it additionally would throw an error, **but only if error display
was disabled**. So it would only error if errors are hidden. Nice, innit?

This basically meant that on your development machine you wouldn't see any errors, but on your
production machine the error log would be flooded with them. Awesome.

So, as of PHP 5.4 thankfully *this behavior is gone*. The error will not be generated anymore.

Additionally there are two options that allow you to specify an alternative to just returning an
empty string:

 * `ENT_IGNORE`: This option (which isn't actually new, it was there in PHP 5.3 already) will just
   drop all invalid code unit sequences. This is bad for two reasons: First, you won't notice
   invalid encoding because it'll be simply dropped. Second, this imposes a certain security risk
   (for more info see the [Unicode Security Considerations][7]).
 * `ENT_SUBSTITUTE`: This new alternative option takes a much more sensible approach at the problem:
   Instead of just dropping the code units they will be replaced by a Unicode Replacement Character
   (U+FFFD). So invalid code unit sequences will be replaced by � characters.

Let's have a look at the different behaviors ([demo][8]):

{% highlight php %}
<?php // "\80" is invalid UTF-8 in this context
var_dump(htmlspecialchars("a\x80b"));                 // string(0) ""
var_dump(htmlspecialchars("a\x80b", ENT_IGNORE));     // string(2) "ab"
var_dump(htmlspecialchars("a\x80b", ENT_SUBSTITUTE)); // string(5) "a�b"
{% endhighlight %}

Clearly, you want the last behavior. In your real code it will probably look like this:

{% highlight php %}
<?php

// this goes into the bootstrap (or where appropriate) to make the code
// not throw a notice on PHP 5.3
if (!defined('ENT_SUBSTITUTE')) {
    define('ENT_SUBSTITUTE', 0);          // if you want the empty string behavior on 5.3
    // or
    define('ENT_SUBSTITUTE', ENT_IGNORE); // if you want the char removal behavior on 5.3
                                          // (don't forget about the security issues though!)
}

// don't forget to specify the charset! Otherwise you'll get the old default charset on 5.3.
$escaped = htmlspecialchars($string, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8');
{% endhighlight %}

Doctype handling
----------------

In PHP 5.4 there are four additional flags for specifying the used doctype:

 * `ENT_HTML401` (HTML 4.01) => this is the default
 * `ENT_HTML5` (HTML 5)
 * `ENT_XML1` (XML 1)
 * `ENT_XHTML` (XHTML)

Depending on which doctype you specify `htmlspecialchars` (and the other related functions) will use
different entity tables.

You can see this in the following example ([demo][9]):

{% highlight php %}
<?php
var_dump(htmlspecialchars("'", ENT_HTML401)); // string(6) "&#039;"
var_dump(htmlspecialchars("'", ENT_HTML5));   // string(6) "&apos;"
{% endhighlight %}

So for HTML 5 an `&apos;` entity will be generated, whereas for HTML 4.01 - which does not yet
support `&apos;` - a numerical `&#039;` entity is returned.

The difference becomes more evident when using `htmlentities`, because the differences are larger
there. You can easily see this by having a look at the raw translation tables:

To do this, we can use the `get_html_translation_table` function. Here first an example for the
XML 1 doctype ([demo][10]):

{% highlight php %}
<?php
var_dump(get_html_translation_table(HTML_ENTITIES, ENT_QUOTES | ENT_XML1));
{% endhighlight %}

The result will look like this:

    array(5) {
      ["""]=>
      string(6) "&quot;"
      ["&"]=>
      string(5) "&amp;"
      ["'"]=>
      string(6) "&apos;"
      ["<"]=>
      string(4) "&lt;"
      [">"]=>
      string(4) "&gt;"
    }

This matches our expectations: XML by itself defines only the five basic entities.

Now try the same thing for HTML 5 ([demo][11]) and you'll see something like this:

    array(1510) {
      ["	"]=>
      string(5) "&Tab;"
      ["
    "]=>
      string(9) "&NewLine;"
      ["!"]=>
      string(6) "&excl;"
      ["""]=>
      string(6) "&quot;"
      ["#"]=>
      string(5) "&num;"
      ["$"]=>
      string(8) "&dollar;"
      ["%"]=>
      string(8) "&percnt;"
      ["&"]=>
      string(5) "&amp;"
      ["'"]=>
      string(6) "&apos;"
      // ...
    }

So HTML 5 defines a vast number of entities - 1510 to be precise. You can also try HTML 4.01 and
XHTML; they both define 253 entities.

Also affected by the chosen doctype is another new error handling flag which I did not mention
above: `ENT_DISALLOWED`. This flag will replace characters with a Unicode Replacement Character,
which formally are a valid code unit sequences, but are invalid in the given doctype.

This way you can ensure that the returned string is always well formed regarding encoding (in the
given doctype). I'm not sure though how much sense it makes to use this flag. The browser will
handle invalid characters gracefully anyways, so this seems unnecessary to me (though I'm probably
wrong).

There is other stuff too...
---------------------------

... but I don't want to list everything here. I think the three changes mentioned above are the most
important improvements.

{% highlight php %}
<?php
htmlspecialchars("<\x80The End\xef\xbf\xbf>", ENT_QUOTES | ENT_HTML5 | ENT_DISALLOWED | ENT_SUBSTITUTE, 'UTF-8');
{% endhighlight %}

  [1]: http://stackoverflow.com/users/127724/artefacto
  [2]: http://docs.php.net/htmlspecialchars
  [3]: http://docs.php.net/htmlentities
  [4]: http://docs.php.net/htmlspecialchars_decode
  [5]: http://docs.php.net/html_entity_decode
  [6]: http://docs.php.net/get_html_translation_table
  [7]: http://unicode.org/reports/tr36/#Deletion_of_Noncharacters
  [8]: http://codepad.viper-7.com/kTwelM
  [9]: http://codepad.viper-7.com/mRSquY
  [10]: http://codepad.viper-7.com/ArEfdy
  [11]: http://codepad.viper-7.com/FiYKxv