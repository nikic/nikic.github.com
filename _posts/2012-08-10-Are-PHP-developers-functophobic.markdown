---
layout: post
title: Are PHP developers functophobic?
excerpt: "PHP developers don't seem to like normal functions much. I think that this is related to the one-to-one class to file mapping that PHP has inherited from Java."
---
There is this one thing that I noticed recently and that concerns me: **PHP devs don't use functions.**

Now, that was overly general, so let me clarify: PHP developers who have reached a certain degree of sophistication
basically stop using plain functions - instead everything goes all classes and methods. At least that's the observation
I made when looking at various open-source libraries and frameworks. The only type of function you'll find in any of the
"high-quality" libs are anonymous functions. But that's pretty much it.

If you look at other languages, the landscape is different. For example Python code will to a large part be class
definitions, but there will always be normal functions in between.

So in PHP functions are basically used in two cases: If the programmer doesn't yet use OOP or if the script isn't worth
using OOP for. The notion of "helper function" or something like that does not seem to exist.

This somewhat concerns me because I think that sticking everything into classes isn't really the right thing to do. This
kind of behavior usually leads to Class-Oriented Programming, where you are basically just using classes for the sake of
using them. (Reminder: Using classes does *not* mean your code is object-oriented!)

I also think that the reluctance to define helper functions is what leads to the push for more and more functions in the
PHP core. E.g. I recently have seen a few people asking for some kind of `array_rand_value` function. This function
would basically be the same as `array_rand`, but it would return a value instead of a key. So it could be defined in
userland code in two lines:

{% highlight php %}
<?php
function array_rand_value(array $array) {
    if (empty($array)) return null;
    return $array[array_rand($array)];
}
{% endhighlight %}

This is a tiny small function that you can easily define yourself. But then arises what I think is the main problem:
Where should you put it? How do I incorporate it with the autoloader?

PHP to large parts inherited the Java OOP model and one thing that came with it is having a one-to-one mapping between
classes and files. PHP does not enforce this, but it is a very common convention. PHP's autoloading support and the
PSR-0 standard are emphasizing this further.

Other languages similar to PHP do not have this convention. For example in Python it is very normal to define multiple
related classes and functions in one file. In Python a file is rather a module, i.e. a group of related components. In
such a setup it is obviously much easier to define small functions in between.

Generally I think that the one-to-one file mapping so common in PHP is problematic. Apart from making it hard to use
functions it also adds an additional cost to creating small classes:

Object oriented programming done right usually leads to a large number of small classes. This is great for code reuse,
maintainability and testing.

But in PHP you have to create a new file for every one of those classes. And this is really getting in my way. I'd have
no problem batch-defining ten tiny classes in one file. But I feel like creating ten distinct files for them is
counterproductive (and hampers maintainability).

Another tangentially related "standard" practice I feel badly about is the excessive use of doccomments. In most cases
phpdoc comments just comment the obvious - at the same increasing the code size by a factor of two or three. I have no
problem with doccomments *where necessary*, but in most cases (at least with well designed code) the behavior should be
obvious from the method and parameter names.

So, basically, what I'm thinking about here is being more "compact": Instead of defining a file for every class, put
related (smaller) classes into one file. Instead of defining a class for everything, just use a function. Instead of
cluttering the code with lots of useless doccomments, just leave them out unless really necessary.

But it's just a thought ;) Maybe I got it all wrong.