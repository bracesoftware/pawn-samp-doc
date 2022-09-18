# `#emit`, `__emit` & `@emit`

Between the latest versions of the compiler and the amx_assembly library, there are now three versions of emit - `#emit`, `__emit`, and `@emit`.  They all have subtly different uses and quirks, and these differences are explained here.

## `#emit`

This is the original version, and is used to insert assembly (p-code) in to a script directly, exactly at the point it is used.  For example:

```pawn
Function()
{
    #emit ZERO.pri
    #emit RETN
    return 1;
}
```

This function will be compiled with two return instructions - one from RETN and one from return, and the second one will never be hit.  The generated assembly will look something like:

```asm
PROC
ZERO.pri
RETN
CONST.pri 1
RETN
```

This is the most inflexible version - it just puts exactly what you typed exactly where you typed it.

It is also in a very strange part of the compiler, likely because it was only originally intended for basic debugging of the pawn VM (hence why it was removed in later official releases).  It uses barely any of the standard compiler features:

```pawn
#emit CONST.pri 5 // This works.
#emit CONST.pri -5 // This is a syntax error.
```

Thus you will often see negative numbers written in their full hex representation (for some reason hex does work despite the fact that negative numbers don’t):

```pawn
#emit CONST.pri 0xFFFFFFFB // -5
```

Defines don’t work to abstract that, but const does:

```pawn
#define MINUS_5 0xFFFFFFFB
#emit CONST.pri MINUS_5 // Undefined symbol `MINUS_5`
```

```pawn
pawn Wrote:
const MINUS_5 = -5;
#emit CONST.pri MINUS_5 // Fine.
```

And #if is completely ignored:

```pawn
#if TEST
    #emit CONST.pri 5
#else
    #emit CONST.pri 6
#endif
```

Very unhelpfully generates:

```asm
CONST.pri 5
CONST.pri 6
```

Both branches are used, so you will often see functions with assembly at the end of a file, ommitted with #endinput:

```pawn
#if !TEST
    #endinput
#endif

Func()
{
    #emit CONST.pri 5
}
```

Or in a separate file altogether.

In short #emit is very weird, but everything else has been built from it so there’s a lot to thank it for.

## `__emit`

This is `#emit`, but super-powered.  It is a full expression-level version of `#emit` correctly integrated in to the compiler.  That means you can use full compile-time expressions:

```pawn
__emit(CONST.pri (5 * MAX_PLAYERS));
```

You can use it in defines:

```pawn
#define GetCurrentAddress() __emit(LCTRL 6)
```

And you can use it in expressions:

```pawn
new var = __emit(CONST.pri 6);
```

Where the register pri is always the result of `__emit` returned like a normal expression.  Compared to the old version:

```pawn
new var = 0;
#emit CONST.pri 6
#emit STOR.S.pri var
```

It also adds `.U`, which tries to work out which instruction to use based on the parameter.  For example with `#emit` incrementing a global variable is:

```pawn
#emit INC var
```

While incrementing a local variable is:

```pawn
#emit INC.S var
```

With `__emit` these become:

```pawn
__emit(INC.U var);
```

And the compiler works out which instruction to use based on the scope of var.

There is more that `__emit` can do, like including multiple instructions in one expression:

```pawn
new var = __emit(LOAD.S.pri param, ADD.C 5); // new var = param + 5;
```

But it is still fundamentally the same as #emit in one important way - it is done at compile-time by the compiler.  Everything is inserted in to the AMX at a fixed location (bearing in mind that macros can change this location).

## `@emit`

This is a macro, and purely a run-time operation.  The simplest way to see what it does is to remove the macro itself and look at the underlying function calls.  A context is created, which includes an address in the AMX, and instructions are written to that address one-by-one while the server is running.  This is a very easy way to write self-modifying code:

```pawn
ReturnFive()
{
    // Wrong value!
    return 3;
}

public OnCodeInit()
{
    // Create the context.
    new context[AsmContext];

    // Initialise the context to point to a function.
    AsmInitPtr(context, _:addressof (ReturnFive<>), 4 * cellbytes);
```

This code creates a context, points it to the given function to rewrite it, and makes the buffer big enough to hold four cells.  Note that this code is in OnCodeInit, which is a special callback called before the mode starts, and before the JIT plugin compiles the mode.  All code rewriting must be done before JIT initialisation.  Note also that because OnCodeInit is called so early you can’t use most useful YSI features like hook and foreach - it too is generating its own code at this point.  We then rewrite the function at that address in assembly:

```pawn
    // A function starts with `PROC`.
    AsmEmitProc(context);

    // Then load the correct return value in to `pri`.
    AsmEmitConstPri(context, 5);

    // And end the function.
    AsmEmitRetn(context);

    return 1;
}
```

`@emit` is just a clever macro that wraps all of these confusing function calls.  While they are fundamentally how code is rewritten at run-time, there’s a lot of boilerplate there that obscures what code is being generated.  So if we call the context ctx (this is important as it is hard-coded in to the macros).  The @emit macro is something like:

```pawn
#define @emit%0%1 AsmEmit%0(ctx, %1);
```

It isn’t exactly that, because that’s not a valid macro, but it shows what is happening.  Thus the code becomes:

```pawn
ReturnFive()
{
    // Wrong value!
    return 3;
}

public OnCodeInit()
{
    // Create the context.
    new ctx[AsmContext];

    // Initialise the context to point to a function.
    AsmInitPtr(ctx, _:addressof (ReturnFive<>), 4 * cellbytes);

    // A function starts with `PROC`.
    @emit PROC

    // Then load the correct return value in to `pri`.
    @emit CONST.pri 5

    // And end the function.
    @emit RETN

    return 1;
}
```

So if you see @emit look around for ctx and that’s where the instructions are being written to in memory.  You can also use labels, but if you want to use variables in the generated code, and not to generate the code, you need to explicitly get their address:

```pawn
// return var ? 0 : 1;
@emit LOAD.S.pri ref(var)
@emit JZER.label failure
@emit CONST.pri 0
@emit RETN
@emit failure:
@emit CONST.pri 1
@emit RETN
```
------------------------------------------
This document was originally written by @Y-Less.
