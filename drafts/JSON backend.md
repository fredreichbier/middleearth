
Author:  Friedrich Weber (fredreichbier)  
Created: 2010-08-09  
Updated: 2010-08-09  
Type:    Extension  
Status:  Draft  
Fixes:   None  

JSON backend for ooc compilers
==============================

   + Motivation

Motivation
----------

Some people (like me) would be very pleased to be able to use ooc code from other languages
like Python. However, this would require writing bindings, and nobody likes writing bindings.
To simplify this process, it would be nice if the ooc compiler could give information about
the API of ooc modules as well as information what the generated C code looks like, so that
one could possibly write automatic bindings generators, based on the compiler's information.

Also, the effort required to write API documentation generators would be reduced significantly
if the compiler can give information about in-code documentation and the inner structure
of ooc code.

Please note that the JSON backend does *not* dump the whole ooc AST - it only contains
information about the publicly accessible interface of the modules, no information
about the actual code.

To guarantee interoperability and simplicity, JSON was chosen as the output format.

rock contains a full implementation of this specification.

Specification
-------------

The JSON backend creates one plain text file containing JSON data for each sourcefile
involved in the build process, i.e. each sourcefile that the compiler would generate
C code from in order to build the executable.

The filename for a module is generated following this scheme:

    {{ outpath }}/{{ sourcepath }}/{{ package }}/{{ module }}.json

 + `outpath`: the desired output directory for the generated files (rock-specific: defaults
   to `rock_tmp`, can be can changed via the `-outpath` command line option)
 + `sourcepath`: the source path the module was found in
 + `package`: one or several nested subdirectories (for example: `io/native/`)
 + `module`: the module name without the trailing `.ooc`

To enable the JSON backend in rock, pass the `-backend=json` command line option.

For a detailed specification of JSON, please refer to the [JSON][] specification.

### Root structure ###

The root structure of the JSON output is a JSON object, having the following members, all
of them required and guaranteed to be present:

 + `path` (string): the absolute module path as it was imported (e.g. `"io/File"`)
 + `entities` (array): an array of members
 + `globalImports` (array of strings): the globally imported modules as an array of strings
   (e.g. `["io/File", "os/Process"])
 + `namespacedImports` (object of arrays of strings): all created namespaces with imported modules
   (e.g. `{"IO": ["io/File", "io/FileReader"]}` for `import io/[File, FileReader] into IO`)
 + `uses` (array): an array of "used" usefiles (e.g. `["foo"]` for `use foo`)

### Members ###

Members are described as arrays-of-arrays rather than objects to preserve the order their definition
in the ooc code.

A member consists of an array containing its name as a string and all versioned entities.

For example, if there is a member named `"foo"` that has two different versions, it builds the
following structure:

    ["foo", entity1, entity2]

The entities contain further information about the different versions.

### Entities ###

An entity describes a certain object in the ooc world, e.g. a class, a global variable, a method as
a JSON object.

The root structure contains only the *top-level* entities, ie. classes, covers, global variables,
functions, but no methods or fields.

Every entity has a `tag` member that contains an identifier that can be used to refer to the entity.
However, it's not really unique, because versioned entities (i.e. two versions of the same entity,
done by using the `version` statement in the ooc code) have the same tag since they cannot coexist anyway.

Furthermore, all entities have a `type` member specifying the entity's type, which is one of:

 + `"function"`
 + `"method"`
 + `"globalVariable"`
 + `"field"`
 + `"class"`
 + `"cover"`
 + `"enum"`
 + `"enumElement"`
 + `"operator"`
 + `"interface"`
 + `"interfaceImpl"`

Also, the `doc` member will contain the documentation given by the user via an oocdoc tag or an empty string.

### Tags ###

Tags are used to identify entities, but also to specify types, especially pointer and reference types, and
to describe version specs.

There is a special mini-language for tags:

    tag: modifier "(" parameters ")" 
       | identifier
    parameters: tag { "," tag }
    identifier: [a-zA-Z0-9_ ]+
    modifier: [a-zA-Z0-9_]+

Note that there should not be a whitespace after commas, so `test(a,b)` is better than `test(a, b)`.

#### Entity tags ####

Described in detail later.

 + `interfaceImpl(interface,for)`
 + `method(className,name)`
 + `field(className,name)`
 + `enumElement(enumName,name)`
 + `operator(name,generics(...)?,arguments(...)?,return(...)?`

#### Type tags ####
 
 + `pointer(type)`: describes a pointer type (e.g. `pointer(Int)`, `pointer(pointer(Char))`)
 + `reference(type)`: describes a reference type
 + `multi(type)`: describes a tuple type as used for multi-return functions (e.g. `multi(pointer(Int),String)`)
 + `Func(generics(...)?,arguments(...)?,return(...)?)`: describes a function type.  
    All three parameters after the name contain several type tags (`generics`, `arguments`) or
    one type tag (`return`), for example:
    `generics(T,U)`, `arguments(String,pointer(Int))`,`return(multi(Int,reference(String)))`  
    If one of these parameters would have no parameters anyway, it can be omitted, for example:
    `Func(arguments(String))`.

#### Version spec tags ####

To identify a certain version specification as given to the version statement, the following
modifiers are used (each with its ooc equivalent)

 + `and(spec1,spec2)`: equivalent to `spec1 && spec2`
 + `or(spec1,spec2)`: equivalent to `spec1 || spec2`
 + `not(spec)`: equivalent to `!spec`

`spec` can be another tag or a version name.

Example:

    version(!(gc && (win32 || win64)))
    
    =>

    "not(and(gc,or(win32,win64)))"

### All entities for you ###

#### function ####

 + `tag`: The name of the function, without modifiers.
 + `type`
 + `doc`
 + `unmangled`
 + `name` (string): Although the name is identical to the tag, it contains the name of the function.
    It also contains the suffix (if given), separated by a "~" char, but no whitespace.
    So, a `doSomething: func ~string` would have the name `"doSomething~string"`.
 + `modifiers` (array of strings): An (possibly empty) array of (string) modifiers. Possible modifiers:  
    `const`, `static`, `final`, `inline`, `proto`
 + `genericTypes` (array of strings): All generic types names or an empty array.
 + `extern` (boolean or string): Either `true` (if it's an extern function, but not aliased) or a string
   containing the extern name of the function (if it's an extern function with a name), or `false`.
 + `returnType` (null or string): Either the return value tag or null if the function has no return type.
 + `arguments` (array of arrays): A list of three-element lists consisting of `[name, argument tag, modifiers or null]`.

If a function has varargs, the last argument is named `"..."` with the type `""` (empty string).

#### method ####

A method entity has the same members as a `function` entity, but its tag looks like this:

    method(className,name)

where `className` is the name (and the tag) of the owner class / cover / interface and name
is the method's name including the suffix.

#### globalVariable ####

 + `tag`: The name of the global variable, without modifiers.
 + `type`
 + `doc`
 + `name`
 + `unmangled`
 + `modifiers` (array of strings): a (possibly empty) list of modifiers. Possible modifiers: `const`, `static`.
 + `value` (string): the variable value (if known, e.g. for constants), otherwise null.
 + `varType` (string): the tag of the variable type.
 + `extern` (string or boolean): Either true, false or a string.
 + `propertyData` (object or null): A propertyData object if it's a property, otherwise null.

##### propertyData #####

An object holding information about the property. Can be present in `field` and `globalVariable` entities.

 + `hasGetter` (boolean): Does it have a getter? (custom or default)
 + `hasSetter` (boolean): Does it have a setter? (custom or default)
 + `fullGetterName` (string or null): full getter function name, or null if it doesn't have one.
 + `fullSetterName` (string or null): full setter function name, or null if it doesn't have one.

### field ###

Like a `globalVariable` entity, but with a `field` tag:

    field(className,name)

where `className` is the name (and the tag) of the owner class / cover / interface and name
is the field's name.

### class ###
 
 + `tag`
 + `type`
 + `doc`
 + `name` (string)
 + `genericTypes` (array of strings)
 + `extends` (string or null): name of the class extending, or null, if it doesn't have a supertype.
 + `members` (array): see the members section.
 + `abstract` (boolean): abstract? true or false.
 + `final` (boolean): a boolean describing if it's final or not.

### cover ###

 + `tag`: Name of the cover.
 + `type`
 + `doc`
 + `name` (string)
 + `from` (string or null): the type we're covering from or null if there is none.
 
### enum ###

 + `tag`: Name of the enum.
 + `type`
 + `doc`
 + `name` (string)
 + `incrementOper` (string): Increment operator. Either `"+"` or `"*"`.
 + `incrementStep` (number): Increment step, defaults to `1`.
 + `elements` (array of 2-element-arrays): An array of arrays `[name, element]` where
   `element` is of the type `enumElement`.

### enumElement ###
 
 + `tag`: `enumElement(enumName,elementName)`. You understand?
 + `type`
 + `doc`: always an empty string here, rock doesn't support it yet.
 + `name` (string): name of the enum element.
 + `extern` (string or null): its extern name or null.
 + `value` (number): its value.

### operator ###

 + `tag`: see the operator tag.
 + `type`
 + `doc`: always an empty string here, rock doesn't support oocdocs on operator definitions yet.
 + `symbol` (string): the ooc operator symbol, like `"+"` or `">>="`
 + `name` (string): its symbol name like `"PLUS"`. Used in operator tags.
 + `function` (object): just an ordinary entity describing the generated function.

### interface ###

 + `tag`: see the interface tag.
 + `type`
 + `doc`
 + `name`
 + `members`: see the members section.

### interfaceImpl ###

 + `tag`
 + `type`
 + `doc`: always an empty string here. No implementation. Too poor.
 + `interface` (string): tag of the implemented interface.
 + `for` (string): tag of the target class or cover.
 
References and footnotes
------------------------

None.

[JSON]: http://json.org
