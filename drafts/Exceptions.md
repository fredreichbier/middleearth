
Author:  Friedrich Weber (fredreichbier)  
Created: 2010-08-27  
Updated: 2010-08-27  
Type:    Feature  
Status:  Draft  
Fixes:   None  

Exceptions
==========

    + Motivation
    + Specification

Motivation
----------

ooc always had exceptions. You could even throw them. But throwing an exception would abort the entire application,
and that's not what exceptions are for, so there was a need for an implementation of exception handling in addition
to an exception handling syntax.

That's the approach we have come up with at last. Most of it is not obvious for the user, so there is the
possibility to replace the implementation with a better implementation in the future. If needed.

Specification
-------------

There are some possible mechanisms to implement exceptions at the C level: Either via return codes, via extra
arguments for errors or via the standard C (therefore portable) setjmp/longjmp mechanism.
We chose the latter because it works at acceptable speed with acceptable usability from C code.

### Jumping ###

It's a bit mind boggling at first. There are two functions in the C header `setjmp.h`: `setjmp` and `longjmp` and there
is a type named `jmp_buf`.

`setjmp` can be used to store the current stack and environment (the current "position" in the code) to jump back to
it later; it stores all necessary information for the jump in an opaque memory chunk whose type is `jmp_buf`. When
storing the information, it will return `0`.

Then there is `longjmp` that can be used to jump back to the context stored in a `jmp_buf` by `setjmp`. `longjmp`
takes an integer argument - this value will be returned by `setjmp` after jumping back.

Now for a code example:

    #include <stdio.h>
    #include <setjmp.h>

    jmp_buf buf;

    void jumpBack() {
        printf("<jumpBack>\n");
        longjmp(buf, 123);
        printf("</jumpBack>\n");
    }

    void doSomething() {
        printf("<doSomething>\n");
        jumpBack();
        printf("</doSomething>\n");
    }


    void main() {
        if(setjmp(buf) == 0) {
            doSomething();    
        } else {
            printf("You jumped!\n");
        }
    }

Look at the `setjmp` line: It compares its return value to `0` - it will be `0` if it was invoked to initially
store the context - this will always happen first.
If the return value is not `0`, we had a jump - `longjmp` was invoked. The return value of `setjmp` is the
non-null value passed to `longjmp` then (`123` in this example). The `else` branch is executed then.

Therefore, the program will print:

    <doSomething>
    <jumpBack>
    You jumped!

You see - the lines after `longjmp` are never executed. Same for the part of `doSomething` after the `jumpBack` call.
The flow was just interrupted and the `else` branch was invoked.

Now, just remove the `longjmp` line, the output will be (as expected):

    <doSomething>
    <jumpBack>
    </jumpBack>
    </doSomething>

The flow is not interrupted here.

This mechanism is perfect for exceptions: the `if` branch would contain the contents of a `try` block, the
`else` branch would contain the contents of the `catch` blocks. And that's how we're doing it.

### Back to ooc ###

Since we're going to have multiple nested try-catch blocks and multi-threaded environments, we can't just
use a global variable to store the `jmp_buf` (`JmpBuf` using ooc's naming conventions).
Therefore, ooc uses a thread-local `Stack<StackFrame>` instance to ensure that every thread can have its own exception handling stack.
`StackFrame` is just an auxiliary cover that holds the `JmpBuf`.

So, what are we going to do if we want to catch an exception?

First, we're going to create a `StackFrame` instance and push it to our thread-local context stack. This is
done using `_pushStackFrame`, returning a `StackFrame` instance that is now at the top of the context stack.
Implemented like this:

    exceptionStack := ThreadLocal<Stack<StackFrame>> new()

    _pushStackFrame: inline func -> StackFrame {
        stack: Stack<StackFrame>
        if(!exceptionStack hasValue?()) {
            stack = Stack<StackFrame> new()
            exceptionStack set(stack)
        } else {
            stack = exceptionStack get()
        }
        buf := StackFrame new()
        stack push(buf)
        buf
    }

And that's the code:

    frame := _pushStackFrame()
    
Now, we can already store the context (i.e. call `setJmp`) - that would be done before entering the `try` block:

    if(frame@ buf setJmp() == 0) {
        // try block here
    } else {
        // catch blocks here
    }

In case you didn't notice - that's exactly the same construct as used in the C code above.

So, the `try` block is invoked, code is run, yadda yadda yadda FAIL LET'S RAISE AN EXCEPTION! This functionality
is built into the `Exception throw` method:

    _EXCEPTION: Int = 1

    throw: func {
        _setException(this)
        if(!_hasStackFrame()) {
            print()
            abort()
        } else {
            frame := _popStackFrame()
            frame@ buf longJmp(_EXCEPTION)
        }
    } 

    _setException: inline func (e: Exception) {
        _exception set(e)
    }

    _getException: inline func -> Exception {
        _exception get()
    }    
    
    _popStackFrame: inline func -> StackFrame {
        exceptionStack get() as Stack<StackFrame> pop() as StackFrame
    }

    _hasStackFrame: inline func -> Bool {
        exceptionStack hasValue?() && exceptionStack get() as Stack<StackFrame> size() > 0
    }

First, it "sets" the exception - it says "THAT is the exception that was just thrown". Again, the current
exception is stored in a thread-local variable (called `_exception`, but that's an implementation detail).

Then, it checks if there is a frame on the stack. If there is one, it means that we're surrounded by a
try-catch block somewhere and that we're going to jump in the catch part.
If there is no frame on the stack, it means the exception is not going to be handled and will therefore
print itself (with a backtrace if possible, but that's not covered here), and then crash the program.

In case we're jumping, we're removing the stack frame at the top (the stack frame describing the
context of the outermost try-catch block available) and call `longJmp` on it. This will cause
the machine to jump in the `else` block of the outermost try-catch block where we can
decide what's going to happen with the exception.

A possible user code without any special try-catch syntax looks like this, then:

    second: func {
        "second" println()
        Exception new("HELLO WORLD!") throw()
    }

    first: func {
        second()
        "first" println()
    }

    main: func {
        frame := _pushStackFrame()
        if(frame@ buf setJmp() == 0) {
            first()
        } else {
            "caught: " print()
            _getException() print()
        }
    }

The output (without the backtrace functionality) will look like this:

    second
    caught: [Exception]: HELLO WORLD!

As it can be seen above, we can get the exception that was thrown by calling `_getException`;
so we can get the exception and decide what happens. We can even call `throw` again
to propagate the exception further down the call stack.

### Syntax ###

Now, it certainly works this way - but all the cool kids have got a special try-catch syntax
for exceptions, and so we're having one, too. It's simple, though:

    try {
        // your probably failing code here
    } catch e: OSError {
        // ...
    } catch (i: IOError) {
        // ...
    } catch {
        // ...
    }

You see: First, there is a block with a `try` and then, multiple `catch` blocks follow, they
are handled from the top to the bottom.
A catch keyword can be followed by a VariableDecl-esque exception type matching expression
(with parentheses or without); `catch e: OSError` will make the exception available as `e`
in the catch block (properly casted to an `OSError`).
The last `catch` block is the fall-through block, it will catch all exceptions that aren't
already handled. If you need access to the exception object, you can alternatively use
`catch e: Exception` as the last catch block - all exceptions are subclasses of `Exception`
anyway.

If you want to propagate the exception any further, use `Exception rethrow`.

If there is no default (ie. without an expression) catch block given, the compiler adds
one just calling `rethrow` on the exception object implicitly.

A word on the implementation: The compiler just replaces the try and catch blocks with
code fiddling with the stackframes and `setJmp` directly. The catch blocks are replaced
by a `match` block with types:

    } catch (i: IOError) {
        // catch #1
    } catch o: OSError {
        // catch #2
    }

will be replaced by:

    match (_getException()) {
        case i: IOError => {
            // catch #1
        }
        case o: OSError => {
            // catch #2
        }
    }

Easy thing.

References and footnotes
------------------------

None.
