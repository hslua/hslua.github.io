[![Hackage]](https://hackage.haskell.org/package/hslua)

[Hackage]: https://img.shields.io/hackage/v/hslua.svg


Overview
--------

HsLua provides the glue to use Lua from within a Haskell app, or the
other way around. It provides foreign function interace (FFI) bindings,
helper functions, and as well as utilities.

### When to use it

You should give HsLua a try if you

- want a ready-made interface to Lua;
- are looking for a way to use pre-existing Lua libraries with your
  Haskell program; or
- need to expose complex Haskell functions to Lua.

HsLua exposes most of Lua's C API via Haskell functions. It offers
improved type-safety when compared to the raw C functions, while also
translating Lua errors to Haskell exceptions. Furthermore, HsLua
provides type-classes which make interacting with Lua straight-forward
and safe.

### When *not* to use it

HsLua is a bad for applications requiring raw speed. Combining two
garbage collecting language with different error-conventions requires
some acrobatics. HsLua is flexible and (relatively) safe, but this comes
at the price of reduced performance. This doesn't matter for most
use-cases, but don't use HsLua just because Haskell is fast. Sometimes,
coding in C and importing the external functions via the Haskell FFI is
the better way.


<!--
Design
------

- Why no monad transformer? Performance degrades when generalizing from
  IO to MonadIO. The most frequent use would be to add an additional
  state. State variables should be safed in the registry, for example
  via as Userdata.

-->

Internals
---------

### Error Handling

HsLua keeps error handling simple and intuitive for users; all of the
below should be abstracted away, and users should not have to concern
themselves with the details described below.


#### Errors and the Call Stack

The method used for error handling by Lua is build around the `setjmp`
and `longjmp` C functions. This is simple and powerful, but poses some
serious problems when combined with Haskell. The `setjmp` function can
jump upwards on the call-stack, possibly skipping frames belonging to
the Haskell runtime system (RTS). This has the potential to confuse the
RTS, resulting in a program crash. Skipping Haskell frames while
unwinding the stack thus has to be avoided at all costs.

We can call Haskell from Lua which calls Lua again etc. At each language
boundary we have to check for errors and propagate them properly to the
next level in stack. Hslua does this for you, both when returning from
Lua to Haskell, and when calling from Haskell into Lua.

#### HsLua must Translate Errors

Let's say we have this call stack: (stack grows upwards)

    Haskell function
    Lua function
    Haskell program

and we want to report an error in the top-most Haskell function. We
can't use `lua_error` from the Lua C API, because it uses `longjmp`,
which means it skips layers of abstractions, including the Haskell RTS.
There's no way to prevent this `longjmp`. `lua_pcall` sets the jump
target, but even with `lua_pcall` it's not safe. Consider this call
stack:

    Haskell function which calls lua_error
    Lua function, uses pcall
    Haskell program

This program jumps to Lua function, skipping Haskell RTS code that would
run before Haskell function returns. For this reason we can use
`lua_pcall` (`pcall`) only for catching errors from Lua, and even in
that case we need to make sure there are no Haskell calls between the
error-throwing Lua call and our `pcall` call.

#### Error to Exceptions (and back)

To be able to catch exceptions from Haskell functions in Lua, we need to
define a custom error protocol. Currently HsLua does this: `error` has
the same type as Lua's `lua_error`, but instead of calling `lua_error`,
it returns two values: A special error value and an error message. HsLua
exceptions are caught and converted to Lua errors via this function.

All of the above internals should stay hidden most of the time, as all
Haskell functions are wrapped such that error values are transformed
into Lua errors behind the scenes.

At this point our call stack is like this:

    Lua function (Haskell function returned with error, which we caught)
    Haskell program

If we want to further propagate the error message to the Haskell
program, then we can just use Lua's standard `error` function and use
`pcall` on the Haskell side. Note that if we use `error` on the Lua side
and forget to use `pcall` in the calling Haskell function, we would be
starting to skip layers of abstractions and would get a segfault in the
best case. That's why HsLua wraps all API functions that can potentially
fail in custom C functions. Those functions behave idential to the
functions they wrap, but catch all errors and return error codes
instead. This comes with a serious performance penalty, but using
@error@ within Lua should be safe.

The `pcall` function is not wrapped in additional C code but still safe.
The reason it's safe is because the `lua_pcall` C function is calling
the Lua function using Lua C API, and when the called Lua function calls
`error` it `longjmp`s to `lua_pcall` C function, without skipping any
layers of abstraction. `lua_pcall` then returns to Haskell.
