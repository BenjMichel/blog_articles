
# Transcompilation #

## What is a transpiler ?

A transpiler (also called source-to-source compiler or a transcompiler) is a program that translates a source code into a source code of another language at the same level of abstraction (which makes it different from a compiler whose output is lower level than its input). The input language can be either an existing language like cfront which converts C++ to C or a new language like Coffeescript which is transcompiled to Javascript. The input language may be a superset of the output language which means that any code written in the output language is valid for the input language.

## Why using transcompilation ?

Transcompilation has several advantages :

* Adding new features to an existing language, for instance [Typescript](http://www.typescriptlang.org/) adds static typing and compile-time type checking or [cfront](http://en.wikipedia.org/wiki/Cfront?oldformat=true) which was the original implementation of C++, adds OOP to the C language.

* Being more efficient when coding. [Coffeescript](http://coffeescript.org/) syntax is more concise than JS, for example :

	In coffeescript :
	```
	add = (a, b) -> a+b
	```
In javascript :
	```
	var add = function (a, b) {
		return a + b;
	}
	```
The codebase is more readable and can be more easily be maintained.

* As a compilation phase with its checks is added, you can detect mistakes earlier.

* Encouraging good pratices. Coffeescript compiler automatically converts "==" and "!=" to "===" and "!==" (for non-JS programmers, the two first ones do not check the type unlike the two last ones).

* Porting a codebase to a new language or converting code to a newer version of the language. Python 3 broke compatibility with Python 2 and a [script](https://docs.python.org/2/library/2to3.html) was written.


##Sources and learn more
* https://github.com/jashkenas/coffeescript/wiki/List-of-languages-that-compile-to-JS
* http://en.wikipedia.org/wiki/Source-to-source_compiler?oldformat=true
* http://code.tutsplus.com/articles/should-you-learn-coffeescript--net-23206