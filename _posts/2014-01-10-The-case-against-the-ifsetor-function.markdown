---
layout: post
title: The case against the ifsetor function
excerpt: The ifsetor function can be used to suppress notices when accessing array indices. This post discusses some of the issues this function has and how to resolve them.
---
Recently igorw wrote a [blog post][igorw_traversal] on how to traverse nested array structures with potentially
non-existing keys without throwing notices. The current "idiomatic" way to do something like this, is to use
`isset()` together with a ternary operator:

{% highlight php startinline %}
    $age = (isset($data['people'][0]['age'])) ? $data['people'][0]['age'] : null;
{% endhighlight %}

The suggested alternative is a [`get_in`][get_in] function, which is used as follows:

{% highlight php startinline %}
    $age = get_in($data, ['people', 0, 'age'], $someDefault);
{% endhighlight %}

Someone on [/r/PHP][reddit_ifsetor] pointed out that there is an alternative approach to this problem, namely the use
of an `ifsetor` function:

{% highlight php startinline %}
    function ifsetor(&$value, $default = null) {
        return isset($value) ? $value : $default;
    }
{% endhighlight %}

The use of this function is very elegant:

{% highlight php startinline %}
    $age = ifsetor($data['people'][0]['age'], $someDefault);
{% endhighlight %}

Note that this function will **not** throw a notice if `$data['people'][0]['age']` (or any index in between) does not
exist, because the `$value` is passed by-reference.

By-reference argument passing
-----------------------------

This seems to come as a surprise to most people, because the code definitely looks like it accesses an undefined index
and ought to throw a notice. Here the magic of references comes in: If you perform a by-reference argument pass (or
assign) PHP will be using a different fetch type for retrieving the array offsets. In this particular case it would
issue a number of "dim w" (dimension write) fetches.

A write-fetch obviously doesn't throw a notice if the assigned index doesn't exist yet (otherwise you wouldn't be able
to create indexes without throwing notices). What's interesting is that the whole thing also work recursively, so none
of the indices in the chain have to exist:

{% highlight php startinline %}
    $array[0][1][2] = 'foobar';
{% endhighlight %}

The above example will not throw a notice if `$array[0][1]` doesn't exist, it won't throw a notice if `$array[0]`
doesn't exist and it even won't throw a notice if the `$array` variable itself doesn't exist.

PHP implements write-fetches by creating the respective offset and initializing it to `null` (if it doesn't yet exist).
This is compatible with nested index assigns because PHP allows silent casts from `null` (and other falsy values) to
arrays. So the following will run without notices:

{% highlight php startinline %}
    $null = null;
    $null[42] = 'foobar';
    var_dump($null); // [42 => 'foobar']
{% endhighlight %}

So what the `$array[0][1][2] = 'foobar'` assignment really does is something along these lines (obviously heavily
simplified):

{% highlight php startinline %}
    $array = null;
    $array[0] = null;           // implicitly converts $array to array
    $array[0][1] = null;        // implicitly converts $array[0] to array
    $array[0][1][2] = 'foobar'; // implicitly converts $array[0][1] to array
{% endhighlight %}

Issues with the ifsetor function
--------------------------------

### Creation of dummy values

Getting back to `ifsetor`, the exact same thing occurs there as well. Before the value is passed to `ifsetor` the
write-fetches will already have created the offset-chain and initialized it to `null`. This means that the function
call will leave behind a `null` value in the index that way passed to it (unless it already existed previously):

{% highlight php startinline %}
    $age = ifsetor($data['people'][0]['age'], $someDefault);
    var_dump($data['people'][0]['age']);
    // NULL, without notice (assuming the index didn't exist beforehand)
{% endhighlight %}

Depending on the context where this occurs, the additional null value might not be of any consequence. On the other hand
it could just as well lead to data corruption, e.g. it could result in a 0 age to be persisted, or something like that.

This is a somewhat subtle issue and quite easy to overlook if you're not familiar with the details of by-reference
passing.

### No notices for nested indices

Another issue is that `ifsetor` doesn't only suppress the notice on the outermost index, but also on all previous
indices and even the array variable itself. As an example, consider the following, slightly changed code:

{% highlight php startinline %}
    $age = ifsetor($date['people'][0]['age'], $someDefault);
{% endhighlight %}

This line contains a typo, namely it incorrectly uses `$date` instead of `$data`. Normally PHP would immediately tell
you about such a typo, but here all notices are suppressed.

This behavior may or may not be what you want. Obviously you'd want a notice for a misspelled `$data`, but it depends
on the context whether a non-existent `$data['people'][0]` should throw a notice or not. Igor's `get_in` function lets
you control which behavior you want:

{% highlight php startinline %}
    $age = get_in($data, ['people', 0, 'age'], $someDefault); // notice on missing $data
    $age = get_in($data['people'], [0, 'age'], $someDefault); // notice on missing $data['people'] as well
    $age = get_in($data['people'][0], ['age'], $someDefault); // notice on missing $data['people'][0] as well
{% endhighlight %}

From my experience you usually only want to suppress the notice on the outermost index. The `$data['people'][0]['age']`
example I've been using throughout the post is somewhat artificial and the real code would likely not involve an
explicit `[0]` access and use a loop instead:

{% highlight php startinline %}
    $people = ifsetor($data['people'], []);
    foreach ($people as $person) {
        $age = ifsetor($person['age']);
        // do something with $age
    }
{% endhighlight %}

See how in both cases something of type `$knownExisting['potentiallyNotExisting']` is fetched? I think that most uses
of the `isset($x) ? $x : $d` pattern are of this form. We'll get back to that at the end of the post.

### Null values treated as non-existing

Recall the definition of the `ifsetor` function:

{% highlight php startinline %}
    function ifsetor(&$value, $default = null) {
        return isset($value) ? $value : $default;
    }
{% endhighlight %}

Another more explicit, but otherwise strictly identical, way of writing the function is this (as `isset` on an existing
variable is just a `null` check):

{% highlight php startinline %}
    function ifsetor(&$value, $default = null) {
        return null !== $value ? $value : $default;
    }
{% endhighlight %}

Note that this function is not capable of distinguishing a `null` value that was actually stored at the index and a
`null` value that was implicitly created by the write-fetch. To `ifsetor` a non-existing offset and an offset containing
`null` are the same.

If `$default` is not null this might result in unexpected behavior:

{% highlight php startinline %}
    $array = [null];
    $result = ifsetor($array[0], 42);
    // $result is now 42, not null, even though $array[0] existed
{% endhighlight %}

Once again, whether or not this is an issue heavily depends on context. In many cases it's okay to treat `null` as
non-existent. In other cases it isn't.

The usual way to avoid this is to use `array_key_exists` instead of `isset` (e.g. the `get_in` function does this), but
there is no way to implement `ifsetor` this way.

### Default is always evaluated

This is an issue that was already mentioned in the comment on reddit: The default value passed to `ifsetor` is always
evaluated, even if it is not needed:

{% highlight php startinline %}
    $value = ifsetor($array['index'], new Foo);
{% endhighlight %}

Here the `new Foo` object is always created, even if `$array['index']` exists. In this case it would just cost one
additional object allocation, but it could also be a more expensive operation (with potential side-effects):

{% highlight php startinline %}
    $value = ifsetor($array['index'], calculateValue());
{% endhighlight %}

This issue exists with both `ifsetor` and `get_in` and can't really be solved in any convenient way. You need to either
write the conditional explicitly or use a (rather verbose) lambda function.

Personally I think that this is actually the least of the issues, because typical default values are simple falsy values
like `null`, `false`, `''` or maybe `[]`. In the cases where you need a complex default value it's probably okay to
write out an if statement.

### By-reference passing often forces a copy

Consider this snippet again:

{% highlight php startinline %}
    $people = ifsetor($data['people'], []);
{% endhighlight %}

Because `ifsetor` accepts the `$value` by reference, there is a good chance that the above call will require a full
copy of the `$data['people']` array if it exists. The reason behind this is that
[PHP's copy-on-write implementation][copy_on_write] forces a zval separation when converting a value to a reference, if
the zval had a refcount>1 beforehand (i.e. if the `$data['people']` zval was used in more than once place).

Once again I think that this is not a particularly large issue, because `ifsetor` would mostly be used on small scalar
types like integers or strings, rather than arrays with a hundred thousand elements, so the copy doesn't matter much.

Still, I think that it's important to be aware of this issue, because in specific cases this minor issue can easily
degenerate to pathological slowdowns. E.g. the Twig templating engine once had [a similar issue][twig_ternary], caused
by lack of copy-on-write support for the ternary operator in ancient versions of PHP ("ancient" obviously means
PHP 5.3 :P)

The ifsetor language construct
------------------------------

As you can see the `ifsetor` function has a lot of issues. I think they can be roughly summarized as "there is way too
much by-ref magic involved".

One way to solve some of the problems is to turn `ifsetor` into a language construct. This feature
[was proposed][ifsetor_rfc] some time ago and declined, the main reason being that you can use the userland
implementation discussed above.

The `ifsetor` language construct would effectively work by a direct expansion to the `isset($x) ? $x : $d` pattern:

{% highlight php startinline %}
    $value = ifsetor($array['index'], $default);
    // expands to
    $value = isset($array['index']) ? $array['index'] : $default;
{% endhighlight %}

Lets take a look at how this would solve some of the issues:

 * **Creation of dummy values**: The `isset()` construct does not use write-fetches. It has its own "is" fetch type,
   which will also suppress notices during the lookup, but won't create any dummy `null` values.
 * **Default is always evaluated**: The ternary operator will only evaluate the branch is actually taken, so the
   default will only be computed if it is used.
 * **By-reference passing often forces a copy**: Since no by-reference pass is involved anymore (and the ternary
   operator uses copy-on-write since PHP 5.4) a copy is no longer necessary.

This leaves only two open issues:

 * **Null values treated as non-existing**: With a language construct this could actually be resolved as well (it's
   hard to put the necessary transformation in terms of PHP code, but internally it's possible), but I'm not sure it
   would make sense, because it makes `ifsetor` inconsistent to `isset`.
 * **No notices for nested indices**: No way around this, at least with that kind of approach.

For me personally the last issue is pretty critical, because I don't like to suppress more errors than strictly
necessary (but well, I could probably live with that.)

What to do?
-----------

After explaining why the `ifsetor` function may not be such a good idea, let's get to what you can use instead. In cases
where you have deeply nested structures with potentially undefined indices all along the path, I'd use Igor's `get_in`
function.

But for the 95% use case, where you only need the notice suppression on the outermost index, why not just go with the
simplest and least-magic solution?

{% highlight php startinline %}
    function array_get(array $array, $index, $default = null) {
        return array_key_exists($index, $array) ? $array[$index] : $default;
    }

    // usage
    $age = array_get($person, 'age', $someDefault);
{% endhighlight %}

I know, I'm stating the obvious here. This is really just `get_in` without the nesting support (thus simplifying its use
for this particular case).

But this still looks rather clumsy. What I really want is the same using [scalar objects][scalar_objects]:

{% highlight php startinline %}
    $age = $person->get('age', $someDefault);
{% endhighlight %}

Yes, this is me calling the `get` "method" on an array. I think this is a very simple and readable solution. But anyway,
that's just a bit of day-dreaming, I have no idea when we'll be introducing scalar objects in PHP (not 5.6 at least).

To finish up this post, let me show you another **very** elegant way of approaching the problem:

{% highlight php startinline %}
    $age = ($_&= $person['age']) ?: $someDefault; // great for code obfuscation
{% endhighlight %}

  [igorw_traversal]: https://igor.io/2014/01/08/functional-library-traversal.html
  [get_in]: https://github.com/igorw/get-in
  [reddit_ifsetor]: http://www.reddit.com/r/PHP/comments/1upjhn/functional_library_traversal/cekfpyj
  [copy_on_write]: http://www.phpinternalsbook.com/zvals/memory_management.html#reference-counting-and-copy-on-write
  [twig_ternary]: http://fabien.potencier.org/article/48/the-php-ternary-operator-fast-or-not
  [ifsetor_rfc]: https://wiki.php.net/rfc/ifsetor
  [scalar_objects]: https://github.com/nikic/scalar_objects
