
Author:  Friedrich Weber (fredreichbier)  
Created: 2010-08-18  
Updated: 2010-08-18  
Type:    Guideline  
Status:  Draft  
Fixes:   None  

Coding Style
============

   + Motivation
   + Specification
   + References and footnotes

Motivation
----------

Free software is about collaboration. Code that uses or contains code written by
other developers just looks nicer with a unified coding style. Of course, this
is only a convention. If you have the impression that a certain syntax just works
better in some special case - we won't kill you (probably).

Please try to follow the guideline as well as possible, though. Pretty please!

Specification
-------------

### Indentation And Spaces ###

Use 4 spaces per indentation level. No tabs please.

There should be at least one blank line to separate
grouped function or method definitions.

You don't need to separate variable or field definitions by
blank lines, but feel free to indicate groups of variables
this way.

There should be a space character after the `func` or `Func`
keywords, also after commas and colons:

    doSomething: func ~withInt <T> (abc: Int, callback: Func (Int) -> Bool) {
        ...
    }

### Naming ###

Classes, interfaces, enums and covers should be named using `CapitalizedCamelCase`.

Functions, methods, fields and enum members should be named using `thisKindOfCamelCase`.

Function suffixes should also be named using `thisKindOfCamelCase`.

Non-static global variables should be named using `thisKindOfCamelCase`.

Static global variables should be named using `ALL_CAPS_WITH_UNDERSCORES`.

Local variables and arguments should be named using `thisKindOfCamelCase`.

Generic types should be named using single uppercase characters (probably you won't
need more than `T` and `U` anyway).

### Variable Declarations ###

Use `:=` if it's possible and unambiguous.

References and footnotes
------------------------

None.
