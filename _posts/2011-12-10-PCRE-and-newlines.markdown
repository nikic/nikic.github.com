---
layout: post
title: PCRE and newlines
excerpt: There is a huge number of newline related features in PCRE (regular expressions) that nearly nobody knows about. I want to shed light on some of those.
---
PCRE (this is what you use when you do a `preg_*` call in PHP) has a plethora of newline related
escape sequences and options, but only few know about those. I want to shed light on some of those
options in this post.

`\r\n`, `\n` and `\r`
---------------------

Okay, everyone knows about those escape sequences:

 * `\r\n` is the CRLF newline Windows uses
 * `\n` is the LF newline Unix uses
 * `\r` is the CR newline that old versions of Mac OS used

A common combination is `(?>\r\n|\n|\r)`, which matches any CRLF type newline.

Meet `\R`!
----------

But apart from `\r` and `\n` PCRE also has another character group matching newlines: `\R`. By
default `\R` matches Unicode newlines sequences, but it can be configured using several options:

 * `/(*BSR_ANYCRLF)\R/`: Matches any CRLF type newline sequence and thus is equivalent to the
   `/(?>\r\n|\n|\r)/` regex
 * `/\R/`: Matches any Unicode newline sequence that is in the ASCII range. I.e. it is equivalent to
   `/(?>\r\n|\n|\r|\f|\x0b|\x85)/`. So additionally to the CRLF type newlines it also matches
   FF formfeeds (`\f`), VT vertical tabs (`\x0b`) and the NEL next line character (`\x85`).
 * `/\R/u` (`u` means UTF-8 mode): Matches any Unicode newline sequence including newline characters
   outside the ASCII range, which is equivalent to `/(?>\r\n|\n|\r|\f|\x0b|\x85|\x{2028}|\x{2029})/`.
   This means only two additonal characters are added: The LS line separator (`\x{2028}`) and the
   PS paragraph separator (`\x{2029}`).

Note that `\R` is not special in character classes. So `/[^\R]/u` for example will *not* match any
non-newline character. Instead it will simply match any character which is not `R`.

The magic `.` dot
-----------------

`.` is usally said to be "any character". In a default configuration this is not quite true. `.` is
"any character *which is not a newline*". What exactly a "newline" means here can also be configured
similarly to `\R`:

 * `/(*CRLF)./`: `.` matches everything apart from CRLF (`\r\n`) characters
 * `/(*LF)./`: `.` matches everything apart from LF (`\n`) characters
 * `/(*CR)./`: `.` matches everything apart from CR (`\r`) characters
 * `/(*ANYCRLF)./`: `.` matches everything apart from CRLF style newlines, i.e. everything apart
   from CRLF, LF or CR.
 * `/(*ANY)./`: `.` matches everything apart from Unicode newline sequences (ASCII) [see above]
 * `/(*ANY)./u`: `.` matches everything apart from Unicode newline sequences (any) [see above]
 * `/./`: What this matches depends on how PCRE was compiled, but the default is to behave as if
    `(*LF)` was specified. I.e. unless you compiled PCRE with some other newline configuration `.`
    will match everything apart from `\n`.

To sum up, have a look at the following code:

{% highlight php %}
<?php
var_dump(preg_match('/^a.+b$/',        "a\r\nb"));  // 0 (Newline \n is     contained by \r\n)
var_dump(preg_match('/^a.+b$/',        "a\nb"));    // 0 (Newline \n is     contained by \n)
var_dump(preg_match('/^a.+b$/',        "a\rb"));    // 1 (Newline \n is not contained by \r)
var_dump(preg_match('/(*CR)^a.+b$/',   "a\nb"));    // 1 (Newline \r is not contained by \n)
var_dump(preg_match('/(*CR)^a.+b$/',   "a\rb"));    // 0 (Newline \r is     contained by \r)
$LS = "\xE2\x80\xA8"; // Line Separator in UTF-8
var_dump(preg_match('/(*ANY)^a.+b$/u', "a{$LS}b")); // 0 (Newline LS is     contained by LS)
var_dump(preg_match('/(*ANY)^a.+b$/',  "a{$LS}b")); // 1 (u modifier was not specified, so LS isn't a newline anymore)
{% endhighlight %}

`PCRE_DOTALL` and `\N`
----------------------

PCRE also provides an option which instructs `.` to *really* match any character. This option is
called `PCRE_DOTALL` and can be specified using the `s` modifier in PHP. So `/./s` will match
absolutely any character, including newlines.

But even in `DOTALL` mode you can get the behavior of the "normal" dot: The `\N` escape sequence
behaves the same as `.`, but is not affected by the `s` modifier. So `\N` will always match any
character which is not a newline (where "newline" is again defined by the above options).

`\N`, just like `\R`, looses it's special meaning within character groups.

Whitespace character groups
---------------------------

As a small addendum I would also like to point out what the different whitespace character groups
contain, as this isn't quite clear to most people:

The commonly used `\s` group matches LF (`\n`), CR (`\r`), HT (tab), FF (form feed) and space
characters. So it does *not* contain the VT vertical tab character. The POSIX character group
`[:space:]` on the other hand includes the vertical tab, too.

The `\pZ` Unicode character property for separators does *not* contain the "classic" newlines. But
there are two special, PCRE specific, character properties for that purpose: `p{Xsp}` contains `pZ`
as well as LF, CR and FF. `p{Xps}` additionally contains VT.

Those two Unicode properties are also internally used in `UCP` mode (`UCP` mode makes the normal
`\s` style character groups behave like the Unicode character properties). I.e. `(*UCP)\s` is
equivalent to `\p{Xsp}` and `(*UCP)[:space:]` is equivalent to `\p{Xps}`.

There are also two more character groups for whitespace matching, namely `\v` for vertical
whitespace and `\h` for horizontal. Contrary to the other `\s` style character groups these two
match non-ASCII characters in UTF-8 mode even when not in `UCP` mode. The reason for this is that
they were added only quite late, whereas the others existed pretty much from the beginning. `\v`
matches CR, LF, VT, FF, NEL, LS, PS. `\h` matches HT, space and 17 other horizontal spaces which you
wouldn't normally know. Those contain things like "Six-per-em space" or "Medium mathematical space".