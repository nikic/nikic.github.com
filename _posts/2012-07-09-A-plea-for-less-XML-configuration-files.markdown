---
layout: post
title: A plea for less (XML) configuration files
excerpt: Configuration files typically use XML or some other domain specific language. But why? Why not just use the usual programming language instead?
---
I recently tried using [Phing][phing] (a PHP build system) to do some simple release automation. Just creating a PEAR
package and doing a few string replacements here and there.

The result? After several wasted hours I ended up using Phing only for PEAR packaging and doing everything else in a
custom PHP build script.

The reason? Phing uses XML files to configure what it should do during a build. So an excerpt from a build file might
look like this:

{% highlight xml %}
<?xml version="1.0" encoding="UTF-8"?>
<project name="SomeName" default="package" basedir="..">
    <target name="package">
        <property file="build/build.properties" />
        <propertyprompt propertyName="project.version" promptText="Enter project version" />

        <delete dir="${path.results.lib}" />
        <mkdir dir="${path.results.lib}" />

        <copy todir="${path.results.lib}">
            <fileset dir="${path.lib}">
                <include name="**/**" />
            </fileset>
        </copy>

        <!-- do more stuff -->
    </target>

    <!-- more targets -->
</project>
{% endhighlight %}

Could you explain me, why you have to do this? Why do you have to specify everything in an XML configuration file? Why
can't you just do the same thing in the programming language you're using? E.g., why can't I write this instead:

{% highlight php %}
<?php
function package(Phing $phing) {
    include 'properties.php';
    $project->version = $phing->prompt('Enter project version');

    $phing->deleteDir($path->results->lib);
    $phing->createDir($path->results->lib);
    $phing->copyDir($path->lib, $path->results->lib);

    // do more stuff, like replace version strings with $project->version
}
{% endhighlight %}

Don't you think that this is much nicer? Some reasons why I prefer this over XML configs:

 * It uses an environment you're already used to. You don't have to learn how exactly the Phing XML files work. You
   don't need to learn where you can use properties and where you can't. You simply know already.
 * You can use the full power of the programming language. E.g. the main reason why I stopped using Phing was that I
   could not implement some slightly more complex string replacement in it without writing 200 lines of XML. I was able
   to implement those replacements in 5 lines of PHP on the other hand.
 * All the tools you normally use for programming will work. E.g. the IDE will be able to provide autocompletion and
   error analysis. One could use phpDocumentor to document the different build targets and options. Heck, you could even
   unit test the building process, if you really wanted to.

And obviously: XML is a rather verbose language, whereas most programming languages (short of Brainfuck) try to be
concise and readable.

At this point you might say that the above isn't really a configuration file, but rather a program written in XML. I'd
agree with you there, but I think the same also applies to "real" configuration files. There too you can benefit from
the power of the programming language (like putting repeating configuration "patterns" into functions or loops), etc.

Also all this isn't specific to XML. I think that it is *generally* preferable to just use your usual programming
language to write configuration, instead of using some special format like XML.

But don't get me wrong, there certainly are cases where using a special configuration format is the way to go:

 * If the configuration will be used by multiple programming languages. Obviously it doesn't make sense to write PHP
   configs if they have to be used by Python too.
 * If the type of configuration is very specific and would largely benefit from a Domain Specific Language.

In my personal experience neither of those points apply for most of the config files I've encountered.

So, what I ask you for is: If you're creating a config file, consider writing it in your normal programming language
instead of XML or some DSL.

 [phing]: http://www.phing.info/trac/