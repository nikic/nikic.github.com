---
layout: post
title: PHP solves problems. Oh, and you can program with it too!
excerpt: PHP is a great language to start programming. And once you started, PHP is also good for "real" programming. So what's the problem?
---
I'd like to point your attention to one particular comment on [Jeff Atwood's new "PHP sucks!" blog
post][phpSingularity], a comment by Adam Victor Nazareth Brandizzi:

> I am a Java programmer at work (could be no more corporate-y) and a Python developer in my projects (could be no more
> hipster) but I admire PHP and its ability of solving problems. It grows because, some times, some poor soul wants to
> create an [online encyclopaedia][wikipedia], or some teacher needs an [online teaching platform][moodle], or someone
> wants to write a [blog][wordpress]. Those people do not want to learn to program, they want to **solve problems**.

I think this comment really nailed it. I think *this* is the most important reason for PHP's success.

People come to PHP because they have some problem and they need to solve it. This is what PHP really shines at. You
can simply take your static HTML website, add a simple `<?php include 'counter.php'; ?>` in there, and ... be done!

From there you start writing simple scripts, learn how to process forms, how to talk to the database, etc. After some
time you start using object oriented programming and maybe make use of some framework.

That's actually pretty much how I got into programming.

With most other languages it is the other way around. With them you first study computer science for five years and then
you go out into the world to find some problem you can solve. (You could say that PHP is a programmer-producing
language, whereas most other languages are programmer-consuming.)

At this point you may ask: Well, once you learned programming, why not switch to another language? Simple: PHP has its
quirks, but it is not **that** bad. Sure, people always try to tell you that, but it just isn't true. Most of the things
people criticize about PHP are really non-issues practically. Like the inconsistent needle/haystack order. That's always
at the top of things that are wrong with PHP. But in reality it really doesn't matter. Sure, it would be nice if the
order was consistent, but my IDE does a very good job at reminding me how to do it correctly.

To summarize:

 * PHP is a great language to start programming!
 * Once you started, PHP is also good for "real" programming (you know, object orientation and stuff).
 * PHP is not as bad as they say. There are issues, like with every language, but they rarely cause problem in practice.

To add to that, I also noticed that most of the PHP bashers have a ten-year-old image of the language. E.g. consider
this quote from Jeff's post:

> What's depressing is not that PHP is horribly designed. Does anyone even dispute that PHP is the worst designed
> mainstream "language" to blight our craft in decades? What's truly depressing is that **so little has changed**.

This statement is outrageous. PHP has changed a lot in recent years, but most people seem to remember it as the shitty
language with terrible OOP support that PHP 4 was. Well, I got news for you: That language is dead for nearly a decade
now. PHP 5 has very good object orientation support, which is actually really similar to Java. PHP 5.3 additionally
added namespace and lambda function support to the mix. (And PHP 5.5 *will* add a lot of awesome new features.) And you
tell me nothing changed?

End of rant.

  [phpSingularity]: http://www.codinghorror.com/blog/2012/06/the-php-singularity.html
  [wikipedia]: http://en.wikipedia.org/wiki/Main_Page
  [moodle]: http://moodle.org/
  [wordpress]: http://wordpress.org/