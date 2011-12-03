---
layout: post
title: Manually installing PEAR on Windows
excerpt: If you ever tried to install PEAR on Windows you probably know what a woeful task it is. This is a short instruction on how to manually install PEAR (without using go-pear.phar).
---
So, you tried installing [PEAR][0] on Windows using their [`go-pear.phar` installer][1] and it - for
some unknown reason - didn't work? Then you still have one options left: Installing it by hand. This
is slightly more cumbersome than just running an installer, but at least it works.

Download PEAR
-------------

First [download the PEAR package][2], extract the `.tgz` file (e.g. using [7-Zip][3]) so you get a
`.tar` file. Extract that file again into some folder (e.g. `PEAR-files/`). If you enter that folder
you should see three entries:

    PEAR-x.y.z/
    package.xml
    package2.xml

Go into the `PEAR-x.y.z/` folder, where `x.y.z` is the PEAR version you downloaded and copy all the
files to the location you want to install PEAR in (I'll use the `D:\htdocs\PEAR` path in this
example).

Write a pear.bat
----------------

Unless you happen to be the rare case where the `pear.bat` just works you will now need to create a
custom `pear.bat` file (it's located in the `script/` directory). Open the file in some text editor
(e.g. [Notepad++][4]) and insert the following:

    @ECHO OFF

    REM The directory we want to put everything in
    SET "PREFIX=D:\htdocs\PEAR"

    REM The PEAR environment variables (using the %PREFIX% defined above)
    SET "PHP_PEAR_INSTALL_DIR=%PREFIX%"
    SET "PHP_PEAR_SYSCONF_DIR=%PREFIX%"
    SET "PHP_PEAR_EXTENSION_DIR=%PREFIX%\ext"
    SET "PHP_PEAR_DOC_DIR=%PREFIX%\docs"
    SET "PHP_PEAR_BIN_DIR=%PREFIX%\scripts"
    SET "PHP_PEAR_DATA_DIR=%PREFIX%\data"
    SET "PHP_PEAR_CFG_DIR=%PREFIX%\cfg"
    SET "PHP_PEAR_WWW_DIR=%PREFIX%\www"
    SET "PHP_PEAR_TEST_DIR=%PREFIX%\tests"
    SET "PHP_PEAR_TEMP_DIR=%PREFIX%\temp"

    REM The PHP binary to use (here a XAMPP binary, yours might be elsewhere)
    SET "PHP_PEAR_PHP_BIN=C:\xampp\php\php.exe"

    "%PHP_PEAR_PHP_BIN%" -C -d date.timezone=UTC -d output_buffering=1 -d safe_mode=0 -d open_basedir="" -d auto_prepend_file="" -d auto_append_file="" -d variables_order=EGPCS -d register_argc_argv="On" -d "include_path='%PHP_PEAR_INSTALL_DIR%'" -f "%PHP_PEAR_BIN_DIR%\pearcmd.php" -- %1 %2 %3 %4 %5 %6 %7 %8 %9

    @ECHO ON

Change the `PREFIX` variable to your PEAR installation directory (i.e. the location you moved the
files to). Also change the `PHP_PEAR_PHP_BIN` variable to the PHP binary you want to use. You can
also adjust the other env variables, though changing some of them will require you to take further
actions. E.g. if you want to change the `BIN_DIR` to `%PREFIX%\bin` you'll need to rename the
`scripts/` folder too.

Download additional packages
----------------------------

If you now open the command line, `cd` into the `scripts/` dir and run `pear` all you will get is
a bunch of `include` errors. This is because `PEAR` requires some more packages to function. You'll
need to install those manually too:

Download the [Console_Getopt][5] package and extract the `.tgz` file just like you extracted the
main PEAR package. Now go into the folder that you have extracted the files into. You'll find a
`package.xml` there and a folder named `Console_Getopt-x.y.z/`. Enter that folder. It'll contain a
yet another folder named `Console/`. Copy this folder into your `PEAR` installation (i.e.
`D:\htdocs\PEAR` in this example).

If you now run `pear` again it should work fine. Run some commands, like `pear update-channels` and
`pear config-set auto_discover 1`. Both should work fine. But if you now run
`pear install --alldeps pear.phpunit.de/PHPUnit` you'll again get a bunch of errors. This is because
PEAR requires two further packages: [Archive_Tar][6] and [Structures_Graph][7]. You can install them
in the same manner as the Console_Getopt package. Only difference is that these packages may also
contain `docs/` and `tests/` directories. Just copy them, too.

If you now try `pear install --alldeps pear.phpunit.de/PHPUnit` again everything should work fine.
PEAR should have created a `PHPUnit/` directory in your installation path and there should be a
`phpunit.bat` in your `scripts/` directory.

Adjusting your include_path
---------------------------

The last step is to add your PEAR installation to the `include_path`. Just open your `php.ini` and
find the line specifying the `include_path`. Replace it with:

    include_path = ".;D:\htdocs\PEAR" ; or whatever your install dir might be

Now you're done :)

 [0]: http://pear.php.net/
 [1]: http://pear.php.net/manual/en/installation.getting.php
 [2]: http://pear.php.net/package/PEAR/download
 [3]: http://www.7-zip.org/
 [4]: http://notepad-plus-plus.org/
 [5]: http://pear.php.net/package/Console_Getopt/download
 [6]: http://pear.php.net/package/Archive_Tar/download
 [7]: http://pear.php.net/package/Structures_Graph/download