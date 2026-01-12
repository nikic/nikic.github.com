---
layout: special
title: About Me
---
Hi! My name is **Nikita Popov**, but you'll mostly meet me as **nikic** on the internet.
I'm a principal software engineer at [Red Hat](https://www.redhat.com/), working on LLVM, Clang and Rust.

Before that, I worked at [JetBrains](https://www.jetbrains.com/) as a core developer for the PHP project. I studied computer science and physics at the Technical University of Berlin.

Feel free to contact me via [nikic@php.net](mailto:nikic@php.net). Alternatively you can usually find me on the [LLVM Discord](https://discord.gg/xS7Z362).

### Projects

My most popular open-source projects, sorted by stars:

 * [PHP-Parser](https://github.com/nikic/PHP-Parser) -- A PHP parser written in PHP
 * [FastRoute](https://github.com/nikic/FastRoute) -- Fast request router for PHP
 * [scalar_objects](https://github.com/nikic/scalar_objects) -- Extension that adds support for method calls on
   primitive types in PHP
 * [iter](https://github.com/nikic/iter) -- Iteration primitives using generators
 * [php-ast](https://github.com/nikic/php-ast) -- Extension exposing PHP 7 abstract syntax tree
 * [PHP-Fuzzer](https://github.com/nikic/PHP-Fuzzer) -- A fuzzer for PHP libraries

I also maintain the [LLVM compile-time tracker](https://llvm-compile-time-tracker.com/).

### Accepted PHP proposals

**PHP 8.2:**

 * [Deprecate dynamic properties](https://wiki.php.net/rfc/deprecate_dynamic_properties)
 * [Deprecate partially supported callables](https://wiki.php.net/rfc/deprecate_partially_supported_callables)

**PHP 8.1:**

 * [*Readonly properties*](https://wiki.php.net/rfc/readonly_properties_v2)
 * [*First-class callable syntax*](https://wiki.php.net/rfc/first_class_callable_syntax)
 * [New in initializers](https://wiki.php.net/rfc/new_in_initializers)
 * [Array unpacking with string keys](https://wiki.php.net/rfc/array_unpacking_string_keys)
 * [Deprecate passing null to non-nullable arguments of internal functions](https://wiki.php.net/rfc/deprecate_null_to_scalar_internal_arg)
 * [Static variables in inherited methods](https://wiki.php.net/rfc/static_variable_inheritance)
 * [Restrict $GLOBALS usage](https://wiki.php.net/rfc/restrict_globals_usage)
 * [Phasing out Serializable](https://wiki.php.net/rfc/phase_out_serializable)
 * [Deprecations for PHP 8.1](https://wiki.php.net/rfc/deprecations_php_8_1)

**PHP 8.0:**

 * [*Union types*](https://wiki.php.net/rfc/union_types_v2)
 * [*Constructor property promotion*](https://wiki.php.net/rfc/constructor_promotion)
 * [*Named arguments*](https://wiki.php.net/rfc/named_params)
 * [Reclassify engine warnings](https://wiki.php.net/rfc/engine_warnings)
 * [Consistent type errors for internal functions](https://wiki.php.net/rfc/consistent_type_errors)
 * [Always generate fatal error for incompatible method signature](https://wiki.php.net/rfc/lsp_errors)
 * [Stricter type checks for arithmetic/bitwise operators](https://wiki.php.net/rfc/arithmetic_operator_type_checks)
 * [Validation for abstract trait methods](https://wiki.php.net/rfc/abstract_trait_method_validation)
 * [Allow trailing comma in parameter list](https://wiki.php.net/rfc/trailing_comma_in_parameter_list)
 * [Object-based token_get_all() alternative](https://wiki.php.net/rfc/token_as_object)
 * [Allow ::class on objects](https://wiki.php.net/rfc/class_name_literal_on_object)
 * [Static return type](https://wiki.php.net/rfc/static_return_type)
 * [Variable syntax tweaks](https://wiki.php.net/rfc/variable_syntax_tweaks)
 * [Weak maps](https://wiki.php.net/rfc/weak_maps)
 * [Treat namespaced names as single token](https://wiki.php.net/rfc/namespaced_names_as_token)
 * [Saner string to number comparison](https://wiki.php.net/rfc/string_to_number_comparison)
 * [Make sorting stable](https://wiki.php.net/rfc/stable_sorting)

**PHP 7.4:**

 * [*Typed properties*](https://wiki.php.net/rfc/typed_properties_v2)
 * [*Arrow functions*](https://wiki.php.net/rfc/arrow_functions_v2)
 * [Allow throwing exceptions from `__toString()`](https://wiki.php.net/rfc/tostring_exceptions)
 * [Reflection for references](https://wiki.php.net/rfc/reference_reflection)
 * [New custom object serialization mechanism](https://wiki.php.net/rfc/custom_object_serialization)
 * [Deprecations for PHP 7.4](https://wiki.php.net/rfc/deprecations_php_7_4)
 * [Deprecate left-associative ternary operator](https://wiki.php.net/rfc/ternary_associativity)

**PHP 7.3:**

 * [Deprecations for PHP 7.3](https://wiki.php.net/rfc/deprecations_php_7_3)
 * [Deprecate and Remove Case-Insensitive Constants](https://wiki.php.net/rfc/case_insensitive_constant_deprecation)

**PHP 7.2:**

 * [Deprecations for PHP 7.2](https://wiki.php.net/rfc/deprecations_php_7_2)

**PHP 7.1:**

 * [Forbid dynamic calls to scope introspection functions](https://wiki.php.net/rfc/forbid_dynamic_scope_introspection)

**PHP 7.0:**

 * [*Exceptions in the engine*](https://wiki.php.net/rfc/engine_exceptions_for_php7)
 * [*Abstract syntax tree*](https://wiki.php.net/rfc/abstract_syntax_tree)
 * [*Uniform variable syntax*](https://wiki.php.net/rfc/uniform_variable_syntax)
 * [Remove deprecated functionality in PHP 7](https://wiki.php.net/rfc/remove_deprecated_functionality_in_php7)
 * [Reclassify E_STRICT errors](https://wiki.php.net/rfc/reclassify_e_strict)
 * [Remove alternative PHP tags](https://wiki.php.net/rfc/remove_alternative_php_tags)
 * [Remove hexadecimal numeric strings](https://wiki.php.net/rfc/remove_hex_support_in_numeric_strings)

**PHP 5.6:**

 * [*Variadic functions*](https://wiki.php.net/rfc/variadics)
 * [*Argument unpacking*](https://wiki.php.net/rfc/argument_unpacking)
 * [*Operator overloading for GMP*](https://wiki.php.net/rfc/operator_overloading_gmp)

**PHP 5.5:**

 * [*Generators and coroutines*](https://wiki.php.net/rfc/generators)
 * [`empty()` on arbitrary expressions](https://wiki.php.net/rfc/empty_isset_exprs)
 * [Remove `/e` modifier from `preg_replace()`](https://wiki.php.net/rfc/remove_preg_replace_eval_modifier)
 * [Support non-scalar foreach keys](https://wiki.php.net/rfc/foreach-non-scalar-keys)

### Presentations

 * PHP 7: What changed internally? [IPC'15] <br>
   ([slides](https://www.slideshare.net/nikita_ppv/php-7-what-changed-internally))
 * PHP 7: What changed internally? [PHP Barcelona'15] <br>
   ([slides](https://www.slideshare.net/nikita_ppv/php-7-what-changed-internally-php-barcelona-2015),
    [video](https://www.youtube.com/watch?v=M8Ktic5sPlo))
 * PHP 7: What changed internally? [Forum PHP'15] <br>
   ([slides](https://www.slideshare.net/nikita_ppv/php-7-what-changed-internally-forum-php-2015),
    [video](https://www.youtube.com/watch?v=zekEqhaPmag))
 * PHP language trivia [PHPKonf'17] <br>
   ([slides](https://www.slideshare.net/nikita_ppv/php-language-trivia))
 * Static Optimization of PHP bytecode [PHPSC'17] <br>
   ([slides](https://www.slideshare.net/nikita_ppv/static-optimization-of-php-bytecode-phpsc-2017))
 * Static Optimization of PHP bytecode [phpDay'17] <br>
   ([video](https://vimeo.com/237704382))
 * Typed Properties and more: What’s coming in PHP 7.4? [PHP Russia'19] <br>
   ([slides](https://www.slideshare.net/nikita_ppv/typed-properties-and-more-whats-coming-in-php-74),
    [video](https://www.youtube.com/watch?v=teKnckg5x7I))
 * PHP performance trivia [PHP Barcelona'19] <br>
   ([slides](https://www.slideshare.net/nikita_ppv/php-performance-trivia),
    [video](https://www.youtube.com/watch?v=JBWgvUrb-q8))
 * Looking forward to PHP 8 [PHPKonf'19] <br>
   ([slides](pdf/slides_phpkonf19_looking_forward_to_php_8.pdf),
    [video](https://www.youtube.com/watch?v=5H77TnoLLcU))
 * What's new in PHP 8.0? [PHP fwdays'20] <br>
   ([slides](https://www.slideshare.net/nikita_ppv/whats-new-in-php-80),
    [video](https://www.youtube.com/watch?v=NbBRXwu1Md8))
 * What's new in PHP 8.0? [PHPConChina'20] <br>
   ([slides](pdf/slides_phpconchina20_whats_new_in_php_8.0.pdf))
 * Just-In-Time Compiler in PHP 8 [betterCode PHP'20] <br>
   ([slides](https://www.slideshare.net/nikita_ppv/justintime-compiler-in-php-8))
 * What's new in PHP 8.0? [SymfonyWorld'20] <br>
   ([slides](https://www.slideshare.net/nikita_ppv/whats-new-in-php-80-239762987))
 * Opaque pointers are coming [LLVM CGO'22] <br>
   ([slides](https://www.slideshare.net/nikita_ppv/opaque-pointers-are-coming),
    [video](https://www.youtube.com/watch?v=qWHLf31NnNk))
 * A whirlwind tour of the LLVM optimizer [EuroLLVM'23] <br>
   ([slides](https://llvm.org/devmtg/2023-05/slides/Tutorial-May10/01-Popov-AWhirlwindTour-oftheLLVMOptimizer.pdf),
    [video](https://www.youtube.com/watch?v=7GHXDEIMGIY))
 * Rust ❤️ LLVM [LLVM developer meeting'24] <br>
   ([slides](pdf/slides_llvm_dev_meeting24_rust_heart_llvm.pdf),
    [video](https://www.youtube.com/watch?v=Kqz-umsAnk8))
 * LLVM IR: Past, Present and Future [EuroLLVM'25] <br>
   ([slides](pdf/slides_eurollvm25_llvm_ir_past_present_and_future.pdf))

### Theses and Papers

 * Spinor formulation of Lanczos potentials in Riemannian Geometry [Bachelor Physics] <br>
   ([pdf](pdf/thesis_physics_bachelor_lanczos_spinors.pdf))
 * Optimizing PHP Bytecode using Type-Inferred SSA Form [Bachelor CS] <br>
   ([pdf](pdf/thesis_cs_bachelor_php_optimization.pdf))
 * The Tradeoff of Latency, Reliability and Rate for Spinal Codes [Master CS] <br>
   ([pdf](pdf/thesis_cs_master_spinal_codes.pdf))
 * Static Optimization of PHP 7 [CC'17] <br>
   ([pdf](pdf/cc17_static_optimization.pdf),
    [acm](http://dl.acm.org/citation.cfm?id=3033026))

### Miscellaneous

 * [PHP internals book](http://www.phpinternalsbook.com/)
   <br> Free (but incomplete) book on PHP extension development
