---
layout: post
title: Improving lexing performance in PHP
excerpt: Some thoughts on improving lexing performance in PHP by compiling the individual token regexes into one big super-regex.
---
[Lexing][1] isn't something you would normally do in PHP, simply because other languages like C can
easily outperform PHP by several orders of magnitude. Still it sometimes is desirable to write
lexers in PHP, e.g. for tokenizing templates, doccomments and other [DSLs][2]. So I want to share
some thoughts on how to write fast lexers in PHP.

Lexing CSV files
----------------

PHP already provides functions for parsing CSV [files][3] and [strings][4]. I will use CSV lexing as
an example anyways, simply because it's so simple. Here an example of a CSV line (as PHP implements
it):

    Field,Another Field,"comma -> , <- comma","quote -> \" <- quote"

If we define the tokens in terms of regular expressions, they would look roughly like this:

{% highlight php %}
<?php
$tokenMap = array(
    '~[^",\r\n]+~A'                     => T_PLAIN_FIELD,
    '~"[^"\\\\]*(?:\\\\.[^"\\\\]*)*"~A' => T_QUOTED_FIELD,
    '~,~A'                              => T_FIELD_SEPARATOR,
    '~\r\n?|\n~A'                       => T_LINE_SEPARATOR,
);
{% endhighlight %}

Looping through the regexes
---------------------------

The most obvious approach to lexing in PHP is to simply loop through the regexes and match them
against the current position, until there are no characters left:

{% highlight php %}
<?php
function lex($string, array $tokenMap) {
    $tokens = array();

    $offset = 0; // current offset in string
    while (isset($string[$offset])) { // loop as long as we aren't at the end of the string
        foreach ($tokenMap as $regex => $token) {
            if (preg_match($regex, $string, $matches, null, $offset)) {
                $tokens[] = array(
                    $token,      // token ID      (e.g. T_FIELD_SEPARATOR)
                    $matches[0], // token content (e.g. ,)
                );
                $offset += strlen($matches[0]);
                continue 2; // continue the outer while loop
            }
        }

        throw new LexingException(sprintf('Unexpected character "%s"', $string[$offset]));
    }

    return $tokens;
}
{% endhighlight %}

The drawback of this code is fairly obvious: On every new offset one needs to iterate through all
regexes and match them one at the time. This maybe isn't such a problem in this case as we only have
four regexes, but when lexing some real language one usually deals with hundreds of them.

Compiling into a single regex
-----------------------------

The solution is to compile all regexes into a single big one. Our above regex would be converted to:

{% highlight php %}
<?php
$regex = '~
    ([^",\r\n]+)
  | ("[^"\\\\]*(?:\\\\.[^"\\\\]*)*")
  | (,)
  | (\r\n?|\n)
~xA';
{% endhighlight %}

But how can we use this? How can we know which of the four subregexes matched the string? Simple: As
all the subregexes are enclosed in parenthesis they are capturing groups. So our `$matches` array
will contain the matched string at position `[0]` and at the position of the matched subregex. All
other groups will be empty.

An example: If the regex matched the `,` subregex, we would get this `$matches` array:

{% highlight php %}
<?php
array(
    0 => ',',
    1 => '',
    2 => '',
    3 => ',',
    4 => '',
);
{% endhighlight %}

From this we know that the 3rd subregex matched, as it is the one which has a value.

The resulting code for an abstract lexer class would be:

{% highlight php %}
<?php
class Lexer {
    protected $regex;
    protected $offsetToToken;

    public function __construct(array $tokenMap) {
        $this->regex = '((' . implode(')|(', array_keys($tokenMap)) . '))A';
        $this->offsetToToken = array_values($tokenMap);
    }

    public function lex($string) {
        $tokens = array();

        $offset = 0;
        while (isset($string[$offset])) {
            if (!preg_match($this->regex, $string, $matches, null, $offset)) {
                throw new LexingException(sprintf('Unexpected character "%s"', $string[$offset]));
            }

            // find the first non-empty element (but skipping $matches[0]) using a quick for loop
            for ($i = 1; '' === $matches[$i]; ++$i);

            $tokens[] = array($matches[0], $this->offsetToToken[$i - 1]);

            $offset += strlen($matches[0]);
        }

        return $tokens;
    }
}
{% endhighlight %}

How much does this really change?
---------------------------------

I am seeing approximately 30% performance improvement for the average case ([online demo][5]). But
this value varies with different regexes and input. If the number of regexes increases the
performance improvement is bigger. Additionally in the compiled-regex variant the order of the
regexed has less influence on the execution time ([online demo][6]). I.e. it is not that important
to put the more probable regexes first and the less probable onces last. (But it still is important,
only less important!)

  [1]: http://en.wikipedia.org/wiki/Lexical_analysis
  [2]: http://en.wikipedia.org/wiki/Domain-specific_language
  [3]: http://php.net/fgetcsv
  [4]: http://php.net/str_getcsv
  [5]: http://codepad.viper-7.com/B5Arj5
  [6]: http://codepad.viper-7.com/hng1i0