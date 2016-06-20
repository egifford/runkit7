Runkit7: Runkit extension fork for PHP 7
========================================

For all those things you.... probably shouldn't have been doing anyway....
__Now with partial support for PHP7!__ (This extension isn't production ready yet).

[Building and installing runkit in unix (64-bit PHP7 only)](#building-and-installing-runkit7-in-unix-php7)

The following contributions are welcome:

-	Pull requests with  PHP5 -> PHP7 code migration of functions
-	New test cases for features that no longer work in PHP7, or code crashing runkit.
-	Issues for PHP language features that worked in PHP5, but no longer work in PHP7, 
	for the implemented methods (`runkit_function_*` and `runkit_method_*`)

Pull requests with fixes, documentation, and additional tests for PHP7 are welcome.

Most of the runkit tests for method manipulation and function manipulation are passing.
Other methods and corresponding tests are disabled/skipped because changes to php internals in php7 made them impractical.

---------------------
## PHP7 SPECIFIC DETAILS

### Bugs in PHP7 runkit

-	There are still segmentation faults when manipulating internal functions
	(when you renaming/redefining/(copying?) them, under certain conditions).
-	There are reference counting bugs. 
	2 calls to `emalloc` have been temporarily replaced with calls to `pemalloc`
	so that I could execute tests.
-	There may be a few remaining logic errors after migrating the code to PHP7.
	The PHP7 Zend API code
-	I still need to fix bugs in the way runkit's extension shutdown is done.
	Importantly, runkit still needs to be cleaned up first (i.e. before every other extension) (To do this, I need to implement the PHP7 version of `php_runkit_hash_move_to_front`)

### Implemented APIs for PHP7
#### Implemented APIs for PHP7 (buggy):

-	`runkit_function_*`: Most tests are passing. There are some bugs related to renaming internal functions.
-	`runkit_method_*`: Most tests are passing. Same comment as `runkit_function_*`
-	`runkit_zval_inspect`: Partly passing, and needs to be rewritten because of PHP7's zval changes.
-	`runkit_constant_add` works. Other constant manipulation functions don't work yet for constants within the same file.
-	Runkit superglobals.

#### Unsupported APIs for PHP7:
(These functions will be missing)

-	`runkit_import`
	Disabled because of bugs related to properties
-	`runkit_class_adopt` and `runkit_class_emancipate`
	Disabled because of bugs related to properties
-	`runkit_lint*`
	Not yet compilable.
-	`runkit_constant_*` : `runkit_constant_add` works reliably, other methods don't.
	This works better when the constants are declared in a different file.
-	`runkit_default_property_*`
	Disabled because of bugs related to properties

	`runkit_default_property_add` has been removed in php7 - it requires `realloc`ing a different zval to add a property to the property table
	That would break a lot of things

#### Reasons for disabling property manipulation

1. The property manipulation code still has bugs, and the "offset" used is in bytes as of php7, but still treated as an index in this code.
2. As of php7's new zval layout, The only way to "add" a default property would be to realloc() every single one
   of the zend_objects that are instances of that class (to make room for another property).
   This would break php internals and possibly extensions.
   A possible other way way would be to change the API to "runkit_default_property_modify($className, $propertyName, $value, $flags = TODO)"
   (with a precondition $propertyName already existed)
   The old way properties of objects was stored was as a pointer to an array.
   In php7, it's part of zend_object itself, similar to what is described in https://gcc.gnu.org/onlinedocs/gcc/Zero-Length.html (1-length, with an UNDEF value at the end)
3. It should be possible to meet many uses by modifying constructors with runkit_method_move and runkit_method_add,
   or using ReflectionProperty for fetching.
   https://secure.php.net/manual/en/reflectionproperty.setaccessible.php (sets accessibility only for ReflectionProperty)
   https://secure.php.net/manual/en/reflectionproperty.setvalue.php
   https://secure.php.net/manual/en/reflectionproperty.getvalue.php
	

### USEFUL LINKS
For those unfamiliar with PHP5 extension writing:
-	[PHP Internals book](http://www.phpinternalsbook.com/index.html) - Describes how to write extensions *for PHP7*
-	[Upgrading PHP extensions from PHP5 to NG](https://wiki.php.net/phpng-upgrading)
-	[PHPNG Implementation Details](https://wiki.php.net/phpng-int)


The representation of internal values(`zval`s) has changed between PHP5 and PHP7, along with the way refcounting is done.

-	https://nikic.github.io/2015/05/05/Internal-value-representation-in-PHP-7-part-1.html
-	https://nikic.github.io/2015/06/19/Internal-value-representation-in-PHP-7-part-2.html

This now uses `zend_string`.
I changed the code to use `zend_string` wherever possible to be consistent.
This is not strictly necessary.

Notes on `HashTable`s

-	https://nikic.github.io/2014/12/22/PHPs-new-hashtable-implementation.html
-	HashTables no longer use linked lists. They use an array of `Bucket`s instead, and use collision chaining.
	(TODO: implement php_runkit_hash_move_to_front)
-	The new versions of `zend_hash_*` take `zend_string` pointers instead of pairs of `char* 
-	Most `zend_hash_*` now have equivalent `zend_hash_str_*` methods.
	(If I remember correctly, `zend_hash_str_*` methods now taken `strlen` as the length instead of `strlen+1`)
-	To add/retrieve pointers from a `zend_hash`, there are now `zend_hash_*_ptr` methods.
	Depending on the table being used, these may call destructor functions when pointers are removed.

Changes to the internal representation of `HashTable`s require a lot of code changes.

Notes on the new implementation of `HashTable`s:

Miscellaneous notes on differences betwen PHP5 and PHP7
-	zend opcode, opline, and zend_functions have changed in PHP7.
-	Stack frame layout has changed.
-	Reflection data structures changed.
-	And so on: https://wiki.php.net/phpng-upgrading (Upgrading extensions from PHP5 to PHP7)
-	https://github.com/php/php-src/blob/PHP-7.0.0/UPGRADING - Describes changes to PHP that can be seen by PHP programmers. (E.g. backwards incompatible changes, deprecated functionality, new language features, etc.)

### FURTHER WORK

Things to do in the near future:

-   Fix bugs related to edge cases of function and method manipulation
-   See if constant manipulation in the same file can be fixed, e.g. by recompiling functions using those constants.
    It was broken because php7 compiler inlines the constants automatically in the generated opcodes.

Things to do after that:

-   Replace property manipulation with runkit_default_property_modify
-   Support and test php 7.1 constant visibility
-   See if `runkit\_lint` can be implemented

UPSTREAM DOCUMENTATION
======================

(runkit7 is forked from https://github.com/zenovich/runkit)

---------------------
Feel free to support Dmitry Zenovich via PayPal (dzenovich@gmail.com) if Runkit serves you.
By making donation you invest in the project's future, helping it to be compatible with current PHP versions
and to have less bugs and more features.

[![paypal](https://www.paypalobjects.com/en_US/i/btn/btn_donateCC_LG.gif)](https://www.paypal.com/cgi-bin/webscr?cmd=_s-xclick&hosted_button_id=P2WY8LBB2YGMQ)

---------------------

Runkit has three groups of features outlined below:


## CUSTOM SUPERGLOBALS
A new .ini entry `runkit.superglobal` is defined which may be specified as a simple variable, or list of simple variables to be registered as
superglobals.  runkit.superglobal is defined as PHP_INI_SYSTEM and must be set in the system-wide php.ini.

Example:

php.ini:
```ini
runkit.superglobal=foo,bar
```

test.php:
```php
function testme() {
  echo "Foo is $foo\n";
  echo "Bar is $bar\n";
  echo "Baz is $baz\n";
}
$foo = 1;
$bar = 2;
$baz = 3;

testme();
```

Outputs:
```
Foo is 1
Bar is 2
Baz is
```

Compatability: PHP 7.0 or greater


## USER DEFINED FUNCTION AND CLASS MANIPULATION
__NOTE: Only a subset of the APIs have been ported to PHP7. Some of these APIs have segmentation faults in corner cases__

Userdefined functions and userdefined methods may now be renamed, delete, and redefined using the API described at http://www.php.net/runkit

Examples for these functions may also be found in the tests folder.

Compatability: PHP7 (Partial, buggy)


## SANDBOXING (Not yet supported)
With the introduction of TSRM based subinterpreter support a running PHP script may now generate a new thread and interactively switch contexts back and
forth between it.  THIS FEATURE DOES NOT PROVIDE FULL SCRIPT THREADING.  This feature only allows you to run processes in a subinterpreter optionally
with additional security.

First, create an instance of the Runkit_Sandbox object:

```php
$php = new Runkit_Sandbox();
```

To read and write variables in this subinterpreter, just access the properties of the object:

```php
$php->foo = 'bar';
$php->baz = 'boom';
```

Individual functions may also be called (executed within the newly created scope):

```php
$php->session_start();
```

Or you can execute a block of arbitrary code:

```php
$php->eval('echo "The value of foo is $foo\n";');
```

In this example, $foo will be interpolated as 'bar' since that's what you set it to earlier.

Certain INI Options which are ordinarily only modifiable in the system php.ini may be passed during instantiation and changed for your subinterpreter as
well, these options are passed as an associative array to the Runkit_Sandbox constructor and include the following:

safe_mode			safe_mode may only be turned on for a Runkit_Sandbox interpreter using this option.  It cannot be turned off, doing so would
					circumvent the setting specified by your system administrator in the system php.ini.

open_basedir		Like safe_mode, you can only use this setting to make things more restrictive.

allow_url_fopen		In keeping with safe_mode, this can only be turned off (more restrictive than global environment)

disable_functions	Any function names specified in this coma-delimited list will be disabled IN ADDITION TO already disabled functions.

disable_classes		Like disable_functions, this list is in addition to already disabled classes.

Sandboxing is ONLY AVAILABLE in PHP 5.1 (release version, or snapshot dated after April 28th, 2005) when thread safety has been enabled.  To enable
thread safety, just make sure that --enable-maintainer-zts is specified on your ./configure line.  This doesn't necessarily mean that your SAPI will use
PHP in a threaded manner, just that PHP is prepared to behave that way.  If you're building for Apache2-Worker then you're already built for thread
safety.

If you wish/need to use PHP 5.0.x, or a cvs snapshot of 5.1 which predates April 28th, you can apply the tsrm_5.0.diff patch included in this package:

```sh
cd /path/to/php-5.0.x/
cat /path/to/runkit/tsrm_5.0.diff | patch -p0
```

Then just rebuild using the --enable-maintainer-zts option specified above.


runkit_lint() and runkit_lint_file() also exist as a simpler approach to verifying the syntactic legality of passed code within an isolated environment.

## BUILDING AND INSTALLING RUNKIT(7) IN 32-bit systems (not yet supported)
__NOTE__: This probably won't work properly with 32-bit builds of PHP.
The Zend VM's implementation for 32-bit PHP is different from the 64-bit VMs.

TODO: 

## BUILDING AND INSTALLING RUNKIT(7) IN UNIX (PHP7)
```
git clone https://github.com/TysonAndre/runkit7.git
cd runkit
phpize
# Need to disable sandboxing and superglobals for PHP7 until they are implemented.
./configure --disable-runkit-sandbox --disable-runkit-super
make
make test
# If you know how to uninstall this:
# sudo make install
```

Current Build Status
--------------------

Roughly 16 failing tests, 93 skipped tests, 79 passing tests.

[![Build Status](https://secure.travis-ci.org/runkit7/runkit7.png?branch=master)](http://travis-ci.org/runkit7/runkit7)
