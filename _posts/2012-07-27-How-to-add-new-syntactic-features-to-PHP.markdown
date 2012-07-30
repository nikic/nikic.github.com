---
layout: post
title: "How to add new (syntactic) features to PHP"
excerpt: "In this post I'm describing how one can add new syntax to PHP. At the same time, this post can be seen as a general introduction to the workings of the Zend Engine."
---
Several people have recently asked me where you should start if you want to add some new (syntactic) feature to PHP. As
I'm not aware of any existing tutorials on that matter, I'll try to illustrate the whole process in the following. At
the same time this is a general introduction to the workings of the Zend Engine. So upfront: I apologize for this overly
long post.

This post assumes that you already have some basic knowledge of C and also know the fundamental concepts of the PHP
implementation (like zvals). If not, you should read up on them beforehand.

As an example I'll use the addition of an `in` operator which you might already know from other languages like Python.
It works as follows:

    $words = ['hello', 'world', 'foo', 'bar'];
    var_dump('hello' in $words); // true
    var_dump('foo' in $words);   // true
    var_dump('blub' in $words);  // false

    $string = 'PHP is fun!';
    var_dump('PHP' in $string);    // true
    var_dump('Python' in $string); // false

So basically, for arrays the `in` operator is the same as the `in_array` function (but without the needle/haystack
problem) and for strings it's like doing a `false !== strpos($str2, $str1)`.

Prerequisites
-------------

Before we can get going, you'll have to first check out and compile PHP. To do so, you need a few tools. Most of them
are probably already installed on your system, but you may need to install "re2c" and "bison" using the package manager
of your choice. On Ubuntu you'd do this:

    $ sudo apt-get install re2c
    $ sudo apt-get install bison

Next, clone php-src from git and compile it:

    // get source code
    $ git clone http://git.php.net/repository/php-src.git
    $ cd php-src
    // create new branch for in operator
    $ git checkout -b addInOperator
    // build ./configure script
    $ ./buildconf
    // configure PHP in debug mode and with thread safety
    $ ./configure --disable-all --enable-debug --enable-maintainer-zts
    // compile (4 is the number of cores you have)
    $ make -j4

The PHP binary should now be available in `sapi/cli/php`. You can try to do a few things:

    $ sapi/cli/php -v
    $ sapi/cli/php -r 'echo "Hallo World!";'

Now that you have a (hopefully) working PHP compile, we'll take a look at what PHP actually does when it runs a script.

The life of a PHP script
------------------------

To run a script PHP goes through three main phases:

 1. Tokenization
 2. Parsing & Compilation
 3. Execution

In the following I'll explain what exactly is done in each phase, how it is implemented and what we need to change in
order to get the `in` operator working.

Tokenization
------------

In the first phase PHP reads in the source code and breaks it down into smaller units called "tokens". For example the
PHP code `<?php echo "Hello World!";` would be broken down to the following tokens:

    T_OPEN_TAG (<?php )
    T_ECHO (echo)
    T_WHITESPACE ( )
    T_CONSTANT_ENCAPSED_STRING ("Hello World!")
    ';'

As you can see the raw source code was broken down into semantically meaningful tokens. The process of doing so is
referred to as tokenization, lexing or scanning and is implemented in the [`zend_language_scanner.l`][scanner_def] file
of the `Zend/` directory.

If you open the file and scroll down a bit (to somewhere around line 1000), you'll find a large number of token
definitions that look like this:

    <ST_IN_SCRIPTING>"exit" {
        return T_EXIT;
    }

The meaning should be rather obvious: If `exit` is encountered in the source code, the lexer should tag it as `T_EXIT`.
The content between `<` and `>` is the state that the text should be matched in. `ST_IN_SCRIPTING` is the normal state
for PHP code. Some examples of other states are `ST_DOUBLE_QUOTE` (in double quoted string), `ST_HEREDOC` (in heredoc
string), etc.

Another thing that can be done in the scanning routines is specifying a "semantic" value (also called "lower value" or
"lval" for short). Here is an example:

    <ST_IN_SCRIPTING,ST_VAR_OFFSET>{LABEL} {
        zend_copy_value(zendlval, yytext, yyleng);
        zendlval->type = IS_STRING;
        return T_STRING;
    }

`{LABEL}` matches a PHP identifier (it is defined as `[a-zA-Z_\x7f-\xff][a-zA-Z0-9_\x7f-\xff]*`) and the code then
returns the token `T_STRING`. Additionally it copies the text of the token into `zendlval`. So if the lexer encounters
an identifier like `FooBarClass` it'll set `FooBarClass` as the lval. The same is also done for strings, numbers,
variable names, etc.

Luckily the `in` operator does not require in-depth knowledge of the lexer. We just have to add this code snippet
somewhere in the file (analogous to the `exit` token above):

    <ST_IN_SCRIPTING>"in" {
        return T_IN;
    }

Furthermore we have to let the engine know that we added a new token. For this open the
[`zend_language_parser.y`][parser_def] and insert the following line somewhere among its peers:

    %token T_IN "in (T_IN)"

Now you should compile PHP again using `make -j4` (you have to run it in the top level `php-src` folder, not in
`Zend/`). This will generate a new lexer using re2c and compile it. To test that it worked, you can try running
something like this:

    $ sapi/cli/php -r 'in'

This should give you a nice parse error:

    Parse error: syntax error, unexpected 'in' (T_IN) in Command line code on line 1

A last thing we have to do is regenerate the data used by the [tokenizer extension][tokenizer] (which exposes the
internal lexer to userland PHP code). For this you have to `cd` into the `ext/tokenizer` folder and execute
`./tokenizer_data_gen.sh`.

If you run `git diff --stat` now, you will see something like this:

    Zend/zend_language_parser.y       |    1 +
    Zend/zend_language_scanner.c      | 1765 +++++++++++++++++++------------------
    Zend/zend_language_scanner.l      |    4 +
    Zend/zend_language_scanner_defs.h |    2 +-
    ext/tokenizer/tokenizer_data.c    |    4 +-
    5 files changed, 904 insertions(+), 872 deletions(-)

The [`zend_language_scanner.c`][scanner_gen] file is the actual lexer generated using re2c. Because it contains line
number information every change to the lexer will cause a huge diff. So don't worry about that ;)

Parsing & Compilation
---------------------

Now that the source code is broken down in meaningful tokens PHP has to recognize larger structures like "this is an
`if` block" or "you're defining a function there". This process is called parsing and is defined in the
[`zend_language_parser.y`][parser_def] file. Again this is just a definition file and the actual parser is generated
using bison.

To understand how the parser definitions work, let's look at an example:

    class_statement:
            variable_modifiers { CG(access_type) = Z_LVAL($1.u.constant); } class_variable_declaration ';'
        |   class_constant_declaration ';'
        |   trait_use_statement
        |   method_modifiers function is_reference T_STRING { zend_do_begin_function_declaration(&$2, &$4, 1, $3.op_type, &$1 TSRMLS_CC); } '('
               parameter_list ')' method_body { zend_do_abstract_method(&$4, &$1, &$9 TSRMLS_CC); zend_do_end_function_declaration(&$2 TSRMLS_CC); }
    ;

For now, let's leave the parts in curly braces out, so we're left with the following:

    class_statement:
            variable_modifiers class_variable_declaration ';'
        |   class_constant_declaration ';'
        |   trait_use_statement
        |   method_modifiers function is_reference T_STRING '(' parameter_list ')' method_body
    ;

You can read this as:

    A class statement is
            a variable declaration (with access modifier)
        or  a class constant declaration
        or  a trait use statement
        or  a method (with method modifier, optional return-by-ref, method name, parameter list and method body)
    .

To find out what exactly a "method modifier" is you'd then go to the `method_modifier` definition, etc. Should be
fairly straightforward.

So in order to add `in` support to the parser, all you have to do is add a new `expr T_IN expr` rule to
`expr_without_variable`:

    expr_without_variable:
        ...
        |   expr T_IN expr
        ...
    ;

If you now run `make -j4` again, bison will attempt to rebuild the parser, but it will fail with the following rather
obscure error message:

    conflicts: 87 shift/reduce
    /some/path/php-src/Zend/zend_language_parser.y: expected 3 shift/reduce conflicts
    make: *** [/some/path/php-src/Zend/zend_language_parser.c] Error 1

A shift/reduce conflict basically means that the parser doesn't know what to do in some situation. The PHP grammar has
three shift/reduce conflicts by itself (which is expected due to stuff like the dangling elseif/else ambiguity). The
remaining 84 conflicts are caused by the new rule.

The reason is that we didn't specify how `in` should behave around other operators. An example:

    // if you write
    $foo in $bar && $someOtherCond
    // should PHP interpret this as
    ($foo in $bar) && $someOtherCond
    // or as
    $foo in ($bar && $someOtherCond)

The above is called "operator precedence". A related concept is "operator associativity", which determines what happens
when you write `$foo in $bar in $baz`.

In order to fix the shift/reduce conflicts all you have to do is find the following line at the start of the parser and
`T_IN` at the end of it:

    %nonassoc '<' T_IS_SMALLER_OR_EQUAL '>' T_IS_GREATER_OR_EQUAL

What this means is that `in` has the same operator precedence as the `<`-style comparison operators and that the
operator is non-associative. Here are some examples of what this does:

    $foo in $bar && $someOtherCond
    // is interpreted as
    ($foo in $bar) && $someOtherCond
    // because `&&` has lower precedence than `in`

    $foo in ['abc', 'def'] + ['ghi', 'jkl']
    // is interpreted as
    $foo in (['abc', 'def'] + ['ghi', 'jkl'])
    // because `+` has a higher precedence than `in`

    $foo in $bar in $baz
    // will throw a parse error, because `in` is non-associative

If you rerun `make -j4` now everything should work out fine. After that you can try out running something like
`sapi/cli/php -r '"foo" in "bar";'`. This should do nothing, apart from printing a memory leak info:

    [Thu Jul 26 22:33:14 2012]  Script:  '-'
    Zend/zend_language_scanner.l(876) :  Freeing 0xB777E7AC (4 bytes), script=-
    === Total 1 memory leaks detected ===

This is expected because we haven't told the parser yet what it should do when it matches an `in` operator. And this is
where the curly braces come in. What you have to do is replace the existing `expr T_IN expr` rule with the following:

    expr T_IN expr { zend_do_binary_op(ZEND_IN, &$$, &$1, &$3 TSRMLS_CC); }

The part in the curly braces is called a semantic action and is run whenever the parser matches a certain rule (or part
of it). The strange looking `$$`, `$1` and `$3` variables in there are nodes. For example `$1` refers to the first
`expr`, `$3` refers to the second `expr` (`$3` because it is the third element in the rule) and `$$` is the result node.

`zend_do_binary_op` is a compiler instruction. It tells the compiler to emit a `ZEND_IN` opcode that will take `$1` and
`$3` as operands and put the result into `$$`.

Compiler instructions are defined in [`zend_compile.c`][compiler] (with a header entry in
[`zend_compile.h`][compiler_header]). `zend_do_binary_op` for example looks like this:

{% highlight c %}
void zend_do_binary_op(zend_uchar op, znode *result, const znode *op1, const znode *op2 TSRMLS_DC)
{
    zend_op *opline = get_next_op(CG(active_op_array) TSRMLS_CC);

    opline->opcode = op;
    opline->result_type = IS_TMP_VAR;
    opline->result.var = get_temporary_variable(CG(active_op_array));
    SET_NODE(opline->op1, op1);
    SET_NODE(opline->op2, op2);
    GET_NODE(result, opline->result);
}
{% endhighlight %}

The code should be easy to grasp and the next section should help putting it into context. One last thing to mention
here is that in most cases you will have to add your own `zend_do_*` function when adding some new syntax. Adding a new
binary operator is one of the few cases where you don't have to. So if you have to add a new `zend_do_*` function, just
have a look at the existing ones. Most of them are quite simple.

Execution
---------

In the previous section I already mentioned that the compiler is emitting opcodes. Let's look closer at how those
opcodes look like (see [`zend_compile.h`][zend_op]):

{% highlight c %}
struct _zend_op {
    opcode_handler_t handler;
    znode_op op1;
    znode_op op2;
    znode_op result;
    ulong extended_value;
    uint lineno;
    zend_uchar opcode;
    zend_uchar op1_type;
    zend_uchar op2_type;
    zend_uchar result_type;
};
{% endhighlight %}

A short description of what the individual components mean:

 * `opcode`: This is the actual operation that should be executed. This could for example be `ZEND_ADD` or `ZEND_SUB`.
 * `op1`, `op2`, `result`: Every operation can have a maximum of two operands (it can obviously also use just one of
   them, or even none at all) and a result node. The type of the nodes is given by `op1_type`, `op2_type` and
   `result_type`. We'll cover how nodes looks like and what types there are in a minute.
 * `extended_value`: The extended value is used to store flags or some other integer value. E.g. the variable fetching
   instruction uses it to store the variable kind (like `ZEND_FETCH_LOCAL` or `ZEND_FETCH_GLOBAL`).
 * `handler`: To optimize opcode execution this stores the handler function associated with the opcode and the operand
   types. This is determined automatically, so it doesn't have to be set in the compilation code.
 * `lineno`: You know what that means...

There are five basic types that can go into the `*_type` properties:

 * `IS_TMP_VAR`: A temporary variable, usually the result of some expression like `$foo + $bar`. Temporary variables
   cannot be shared, so they don't implement refcounting. They are usually very short lived, so one instruction creates
   them and the next one already frees them again. Temporary variables are usually written as `~n`, so `~0` would be the
   first temporary variable, `~1` the second, etc.
 * `IS_CV`: A compiled variable. To save hash table lookups PHP caches the location of simple variables like `$foo` in
   an array (C array). Furthermore compiled variables allow PHP to optimize the hash table away altogether. CVs are
   denoted using `!n` (`n` here is the offset into the compiled-variable array.)
 * `IS_VAR`: Only simple variables can be turned into CVs. All other kinds of variable accesses, like `$foo['bar']` or
   `$foo->bar` return an `IS_VAR` variable. It basically is just a normal zval (with refcounting and everything). Vars
   are written as `$n`.
 * `IS_CONST`: Constants are literals in the code. For example if you write `"foo"` or `3.141` in your code, those will
   be of type `IS_CONST`. Constants allow for some further optimizations, like reusing zvals for the same value, or
   precalculating hashes.
 * `IS_UNUSED`: Operand not used.

Related to this is how [`znode_op`][znode_op] itself looks like:

{% highlight c %}
typedef union _znode_op {
    zend_uint      constant;
    zend_uint      var;
    zend_uint      num;
    zend_ulong     hash;
    zend_uint      opline_num;
    zend_op       *jmp_addr;
    zval          *zv;
    zend_literal  *literal;
    void          *ptr;
} znode_op;
{% endhighlight %}

As you can see a node is a union, i.e. it can contain one of the elements above (just one!), depending on context. For
example `zv` is used to store `IS_CONST` zvals, `var` is used to store the variable number for `IS_CV`, `IS_VAR` and
`IS_TMP_VAR` variables. The rest is used in various special circumstances. E.g. `jmp_addr` is used with the `JMP*`
instructions (which are required for conditions and loops). Others again are used only during compilation, not during
execution (like `constant`).

So now that we know how individual ops look like, the only remaining question is where those are stored: For every
function (and file) PHP creates a [`zend_op_array`][zend_op_array], which stores the opcodes as well as a lot of other
information. I don't want to go into detail what the individual components are for, you should just know that this
structure exists.

Now let's get back to the implementation of the `in` operator! We already instructed the compiler to emit a `ZEND_IN`
opcode. Now we have to define what this opcode does.

This is done in the [`zend_vm_def.h`][vm_def] file. If you look at it, you'll find that it is full of definitions like
this:

{% highlight c %}
ZEND_VM_HANDLER(1, ZEND_ADD, CONST|TMP|VAR|CV, CONST|TMP|VAR|CV)
{
    USE_OPLINE
    zend_free_op free_op1, free_op2;

    SAVE_OPLINE();
    fast_add_function(&EX_T(opline->result.var).tmp_var,
        GET_OP1_ZVAL_PTR(BP_VAR_R),
        GET_OP2_ZVAL_PTR(BP_VAR_R) TSRMLS_CC);
    FREE_OP1();
    FREE_OP2();
    CHECK_EXCEPTION();
    ZEND_VM_NEXT_OPCODE();
}
{% endhighlight %}

The `ZEND_IN` opcode will look very similar to this, so it's worth understanding what is going on there. I'll go through
it line by line:

{% highlight c %}
// The header defines four things:
//   1. This is the opcode with ID 1
//   2. This opcode is called ZEND_ADD
//   3. This opcode accepts CONST, TMP, VAR and CV as the first operand
//   4. This opcode accepts CONST, TMP, VAR and CV as the second operand
ZEND_VM_HANDLER(1, ZEND_ADD, CONST|TMP|VAR|CV, CONST|TMP|VAR|CV)
{
    // USE_OPLINE means that we want to access the zend_op as `opline`.
    // This is required for all opcodes accessing operands or setting a return value.
    USE_OPLINE
    // For every operand that is accessed a free_op* variable has to be defined.
    // It is used to figure out whether the operand needs freeing.
    zend_free_op free_op1, free_op2;

    // SAVE_OPLINE() actually loads the zend_op into `opline`.
    // USE_OPLINE was only the declaration
    SAVE_OPLINE();
    // Call the fast add function
    fast_add_function(
        // And tell it to put the result into the temporary result variable
        // EX_T here accesses the temporary variable with ID opline->result.var.
        &EX_T(opline->result.var).tmp_var,
        // Fetch the first operand for reading (the R in BP_VAR_R)
        GET_OP1_ZVAL_PTR(BP_VAR_R),
        // Fetch the second operand for reading
        GET_OP2_ZVAL_PTR(BP_VAR_R) TSRMLS_CC);
    // Free both operands (if necessary)
    FREE_OP1();
    FREE_OP2();
    // Check for exceptions. Exceptions can occur virtually everywhere, so one has to check for them in nearly all
    // opcodes. If in doubt, add the check.
    CHECK_EXCEPTION();
    // Go to the next opcode
    ZEND_VM_NEXT_OPCODE();
}
{% endhighlight %}

As you probably noticed there is a lot `UPPERCASE_STUFF` in there. The reason is that `zend_vm_def.h` once again is only
a definition file. The actual Zend VM is generated from it and stored in [`zend_vm_execute.h`][vm_gen] (biiig file). PHP
has three different virtual machine kinds, namely `CALL` (default), `GOTO` and `SWITCH`. Because they all have different
implementation details the definition file uses lots of pseudo-macros like `USE_OPLINE` that are later replaced by some
concrete implementation.

Furthermore the generated VM creates specialized implementations from all possible operand-type permutations. So in the
end there won't be a single `ZEND_ADD` function, but rather different functions for `ZEND_ADD_CONST_CONST`,
`ZEND_ADD_CONST_TMP`, `ZEND_ADD_CONST_VAR`...

Now, in order to implement the `ZEND_IN` opcode, you should add a new opcode definition skeleton at the end of the
`zend_vm_def.h` file:

{% highlight c %}
// 159 is the number of the next free opcode for me. You may need to choose a larger number
ZEND_VM_HANDLER(159, ZEND_IN, CONST|TMP|VAR|CV, CONST|TMP|VAR|CV)
{
    USE_OPLINE
    zend_free_op free_op1, free_op2;
    zval *op1, *op2;

    SAVE_OPLINE();
    op1 = GET_OP1_ZVAL_PTR(BP_VAR_R);
    op2 = GET_OP2_ZVAL_PTR(BP_VAR_R);

    /* TODO */

    FREE_OP1();
    FREE_OP2();
    CHECK_EXCEPTION();
    ZEND_VM_NEXT_OPCODE();
}
{% endhighlight %}

This does nothing more than fetching the operands and discarding them again right away.

In order to generate a new VM you now have to run `php zend_vm_gen.php` from within the `Zend/` directory. (If it gives
you lots of warnings about the `/e` modifier being deprecated, ignore those.) After that go into the top level directory
again and run `make -j4` to recompile.

If everything worked out fine you can now run `sapi/cli/php -r '"foo" in "bar";'` without getting any errors (but it
still won't do anything).

Now, finally, we can implement the actual logic. Let's start with the string case:

{% highlight c %}
if (Z_TYPE_P(op2) == IS_STRING) {
    zval op1_copy;
    int use_copy;

    /* Convert the needle into a string */
    zend_make_printable_zval(op1, &op1_copy, &use_copy);
    if (use_copy) {
        op1 = &op1_copy;
    }

    if (Z_STRLEN_P(op1) == 0) {
        /* For empty needles return true */
        ZVAL_TRUE(&EX_T(opline->result.var).tmp_var);
    } else {
        char *found = zend_memnstr(
            Z_STRVAL_P(op2),                  /* haystack */
            Z_STRVAL_P(op1),                  /* needle */
            Z_STRLEN_P(op1),                  /* needle length */
            Z_STRVAL_P(op2) + Z_STRLEN_P(op2) /* haystack end ptr */
        );

        ZVAL_BOOL(&EX_T(opline->result.var).tmp_var, found != NULL);
    }

    /* Free copy */
    if (use_copy) {
        zval_dtor(&op1_copy);
    }
}
{% endhighlight %}

The hardest part here is actually casting the needle into a string. This is done using `zend_make_printable_zval`. This
function may either have to create a new zval, or not. That's why we pass `op1_copy` and `use_copy` into it. If the
function copied the value we just put it into the `op1` variable (so we don't have to deal with two different variables
everywhere). Furthermore the copy has to be freed at the end (what the last three lines do).

After that everything is simple. We can find out whether the haystack contains the needle using `zend_memnstr`. If the
needle is an empty string we just return `true` directly (because the empty string is part of every string).

Now, if you added the above code in place of the `/* TODO */`, reran `zend_vm_gen.php` and recompiled using `make -j4`,
you will already have a half-working `in` operator:

    $ sapi/cli/php -r 'var_dump("foo" in "bar");'
    bool(false)
    $ sapi/cli/php -r 'var_dump("foo" in "foobar");'
    bool(true)
    $ sapi/cli/php -r 'var_dump("foo" in "hallo foo world");'
    bool(true)
    $ sapi/cli/php -r 'var_dump(2 in "123");'
    bool(true)
    $ sapi/cli/php -r 'var_dump(5 in "123");'
    bool(false)
    $ sapi/cli/php -r 'var_dump("" in "test");'
    bool(true)

Next, we have to implement the array behavior:

{% highlight c %}
else if (Z_TYPE_P(op2) == IS_ARRAY) {
    HashPosition pos;
    zval **value;

    /* Start under the assumption that the value isn't contained */
    ZVAL_FALSE(&EX_T(opline->result.var).tmp_var);

    /* Iterate through the array */
    zend_hash_internal_pointer_reset_ex(Z_ARRVAL_P(op2), &pos);
    while (zend_hash_get_current_data_ex(Z_ARRVAL_P(op2), (void **) &value, &pos) == SUCCESS) {
        zval result;

        /* Compare values using == */
        if (is_equal_function(&result, op1, *value TSRMLS_CC) == SUCCESS && Z_LVAL(result)) {
            ZVAL_TRUE(&EX_T(opline->result.var).tmp_var);
            break;
        }

        zend_hash_move_forward_ex(Z_ARRVAL_P(op2), &pos);
    }
}
{% endhighlight %}

Here the haystack is simply traversed and every values is checked against the needle. We compare using `==`. To compare
using `===` one would have to replace `is_equal_function` with `is_identical_function`.

After rerunning `zend_vm_gen.php` and `make -j4` the `in` operator should be fully operational:

    $ sapi/cli/php -r 'var_dump("test" in []);'
    bool(false)
    $ sapi/cli/php -r 'var_dump("test" in ["foo", "bar"]);'
    bool(false)
    $ sapi/cli/php -r 'var_dump("test" in ["foo", "test", "bar"]);'
    bool(true)
    $ sapi/cli/php -r 'var_dump(0 in ["foo"]);'
    bool(true) // because we're comparing using ==

One last thing to consider is what should happen when the second operator is neither array nor string. I'll just take
the easy way out for this: Throw a warning and return false:

{% highlight c %}
else {
    zend_error(E_WARNING, "Right operand of in has to be either string or array");
    ZVAL_FALSE(&EX_T(opline->result.var).tmp_var);
}
{% endhighlight %}

After rebuilding and recompiling the VM:

    $ sapi/cli/php -r 'var_dump("foo" in new stdClass);'

    Warning: Right operand of in has to be either string or array in Command line code on line 1
    bool(false)

This might not be the best behavior. E.g. one could allow to do things like `2 in 123` or `3.14 in 3.141`. But I'm too
lazy for that right now ;)

Finishing thoughts
------------------

I hope the above helped you understand how to add new features to PHP and what the Zend Engine does when it runs a PHP
script. But even though the article is quite long I covered only small parts of the whole system. So when you want to do
modifications to the ZE the largest part of the job will be reading the existing code. The [cross-reference tool][xref]
helps a lot when browsing through the PHP source code. Apart from that you can always ask questions in the #php.pecl
room on efnet.

After you created an implementation for whatever feature you want, the next step is bringing it up on the [internals
mailing list][internals_list]. People will then look at your feature and decide whether or not it should go in.

Oh, and one last thing: The `in` operator here was just an example. I don't plan on proposing it for inclusion ;)

As always, if you have any further comments or questions, please leave them below.

  [scanner_def]: http://lxr.php.net/xref/PHP_TRUNK/Zend/zend_language_scanner.l
  [scanner_gen]: http://lxr.php.net/xref/PHP_TRUNK/Zend/zend_language_scanner.c
  [parser_def]: http://lxr.php.net/xref/PHP_TRUNK/Zend/zend_language_parser.y
  [compiler]: http://lxr.php.net/xref/PHP_TRUNK/Zend/zend_compile.c
  [compiler_header]: http://lxr.php.net/xref/PHP_TRUNK/Zend/zend_compile.h
  [tokenizer]: http://php.net/tokenizer
  [zend_op]: http://lxr.php.net/xref/PHP_TRUNK/Zend/zend_compile.h#106
  [znode_op]: http://lxr.php.net/xref/PHP_TRUNK/Zend/zend_compile.h#74
  [zend_op_array]: http://lxr.php.net/xref/PHP_TRUNK/Zend/zend_compile.h#_zend_op_array
  [vm_def]: http://lxr.php.net/xref/PHP_TRUNK/Zend/zend_vm_def.h
  [vm_gen]: http://lxr.php.net/xref/PHP_TRUNK/Zend/zend_vm_execute.h
  [xref]: http://lxr.php.net
  [internals_list]: http://php.net/mailing-lists.php