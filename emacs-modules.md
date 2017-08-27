---
title: Emacs modules
---
<meta charset="utf-8">

# Emacs modules
{:.no_toc}

* TOC
{:toc}

## Introduction

The GNU Emacs dynamic module API is a C API that allows you to create extension
modules for GNU Emacs written in C or any other language providing C bindings.
This document specifies the interface and behavior of the module subsystem and
the requirements that modules have to fulfill.

Because the module API is a C API, you have to be familiar with C to write
Emacs modules.  Be aware that C is a difficult and unforgiving language; subtle
mistakes tend to result in [undefined
behavior](https://en.wikipedia.org/wiki/Undefined_behavior).  Undefined
behavior is always a bug that you have to find and fix.

All the snippets on this page are subject to the following license:

> Copyright 2017 Google Inc.
>
> Licensed under the Apache License, Version 2.0 (the “License”); you may not
> use this file except in compliance with the License.  You may obtain a copy
> of the License at
>
> <https://www.apache.org/licenses/LICENSE-2.0>
>
> Unless required by applicable law or agreed to in writing, software
> distributed under the License is distributed on an “AS IS” BASIS, WITHOUT
> WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.  See the
> License for the specific language governing permissions and limitations under
> the License.


## Definitions

In this document I’ll use the terms [**undefined
behavior**](https://en.wikipedia.org/wiki/Undefined_behavior) and
[**unspecified behavior**](https://en.wikipedia.org/wiki/Unspecified_behavior)
with the same meanings as in the
[C standard](http://www.open-std.org/jtc1/sc22/wg14/www/docs/n1256.pdf).
Please be aware that the [Emacs Lisp
manual](https://www.gnu.org/software/emacs/manual/html_node/elisp/index.html)
generally uses the word “undefined” where I use “unspecified.”

A **module** is a [library](https://en.wikipedia.org/wiki/Library_(computing))
that GNU Emacs can [dynamically
load](https://en.wikipedia.org/wiki/Dynamic_loading) to provide extension
functions not implemented in Emacs Lisp.  The exact format of module files
depends on the underlying operating system; on GNU/Linux, modules are shared
ELF libraries.

The **GNU Emacs module API** is a set of functions, type definitions, macros,
and enumerators in the header file `emacs-module.h`.  This document describes
its behavior.

A **module function** is a function exported from a module library and
registered with GNU Emacs that provides extension functions to GNU Emacs.

A **module initialization function** is a function exported from a module
library that GNU Emacs calls to initialize the containing module.

An **Emacs value** is the representation of a Lisp object in the module API.
Emacs values are represented using the opaque pointer type alias `emacs_value`.

A **runtime** is an object that Emacs provides to module initialization
functions.  Runtimes are represented using pointers to the structure type
`emacs_runtime`.

A **runtime function** is a function obtained by dereferencing one of the
fields of `emacs_runtime` that have pointer-to-function type.

An **environment** is a context object that modules use to interact with an
Emacs process into which they are loaded.  Environments are represented using
objects of the structure type alias `emacs_env`, or pointers to such objects.

An **environment function** is a function obtained by dereferencing one of the
fields of `emacs_env` that have a pointer-to-function type.  Environment
functions are the primary means of interacting with GNU Emacs.

A **user pointer** is a special kind of Lisp object that wraps an arbitrary
value of type `void *` and a **finalizer function** to clean up (“finalize”)
such a value.


## Common requirements

In this document I’ll use wording similar to
[RFC 2119](https://www.ietf.org/rfc/rfc2119.txt) to describe the requirements
of the module API.  In particular, the words “must” and “mustn’t” denote
absolute requirements that you have to fulfill.  Not fulfilling any of the
requirements described here results in undefined behavior, unless otherwise
noted.

Unless otherwise noted, all pointers passed to module API functions must point
to valid, initialized objects and mustn’t be `NULL`.  Likewise, unless
otherwise noted, pointer-returning environment functions will always return
valid pointers to initialized objects.  If possible, `emacs-module.h` uses
compiler extensions to trigger warnings if the compiler can prove that `NULL`
is incorrectly passed to a module API function.

Modules must use the functions provided by the module API to obtain environment
pointers and value objects; there is no other way to obtain these objects.

Unless otherwise noted, the relation between pointers of the same type passed
to module functions is unspecified.  Modules mustn’t make any assumptions abut
equality or ordering of such pointers.

A module mustn’t access or use runtimes, environments, or values passed to or
obtained in a different module.

Module, initialization, and finalizer functions must either exit the process or
return locally; they mustn’t exit nonlocally (e.g., by using `longjmp` or C++
exceptions).  Functions written in C++ must be declared using the C language
linkage.

Structure types defined by the module API may contain private fields; modules
mustn’t attempt to use or alter these fields.


## Lifetime

### Runtime lifetime

The lifetime of a runtime object is finite.  It corresponds to the C lifetime
of the `emacs_runtime` pointer passed to the module initialization function.
You must not access a runtime object outside its lifetime.


### Environment lifetime

The lifetime of an environment is finite; its beginning and end is described
below, in the sections that describe how modules can obtain pointers to
environment objects.  Modules mustn’t dereference environment pointers or pass
them to module API functions outside of the lifetime of the environments they
represent.


### Value lifetime

Regarding their lifetime, there are two kinds of values: local values and
global values.  Local values are *owned* by a specific environment, and their
lifetime is bound by the lifetime of their owning environment.  The lifetime of
a value begins directly after the function with which it is obtained returns.
The lifetime of a local value ends not before the lifetime of their owning
environment ends; modules mustn’t make any assumptions about the lifetime of
values after their owner’s lifetime has ended.  The lifetime of global values
ends when they are freed (see below) or the Emacs process exits, whichever
comes first.

Modules mustn’t access or use values outside of their lifetime.


## Nested invocations

Multiple invocations to module initialization functions or module functions can
be active at the same time.  Each such invocation receives a unique `emacs_env`
pointer that is different from all other environment pointers that are live at
the same time.


## Threads

The mapping of Emacs Lisp threads to operating system threads is unspecified.
Emacs will never call module initialization functions, module functions, and
user pointer finalizer functions concurrently; this means that at most one such
function is running at a time (unless called from outside of Emacs), and access
to global state doesn’t need synchronization.  However, it’s unspecified in
which operating system thread the functions are called; for example, two nested
invocations of a module function may or may not be executed in the same
thread.

You mustn’t interact with Emacs outside of the current Lisp thread.  Given the
non-concurrency guarantee it’s enough to ensure that you never access the
fields of the structures described in this document from threads that Emacs
hasn’t created.


## Compatibility

### Language compatibility

The Emacs module API is guaranteed to work with all standard C versions
starting with C99 and with all standard C++ versions starting with C++11.  In
practice, it only requires language constructs from C89 or C++98 and some
standard library headers from newer versions, so there’s a good chance that it
works just fine with earlier language versions.


### API compatibility

All documented structure names, structure field names, enumeration names,
enumerator names, enumerator values, and type alias names in the
`emacs-module.h` header fields are stable and will never be changed or removed.
Parameter names are not part of the API.  There might be additional
undocumented names in the header, which are not part of the API and subject to
change at any time.  All toplevel names introduced in `emacs-module.h` begin
with `emacs_` or `EMACS_`.  Emacs may add new names to `emacs-module.h` at any
time; all new toplevel names will also start with `emacs_` or `EMACS_`.
Non-toplevel names such as structure fields or parameters don’t have specific
prefixes.  `emacs-module.h` depends only on headers from the standard C
library.


### ABI compatibility

To allow backwards and forwards compatibility, the following guarantees are
made about all structure types described in this document:

-   Fields are never removed.

-   Fields are never reordered.

-   New fields get only added at the end of structures.

-   Adding new fields will always increase the size of a structure.

-   The first field is always a field named `size` of type `ptrdiff_t`
    containing the actual size of the object, in bytes.  The value of the
    `size` field will always be greater than zero and less than or equal to
    `SIZE_MAX`.

Modules mustn’t access structure fields outside of the object, even if they
could do so using field access (i.e. if the size of a structure object as seen
by the module is larger than the actual size as passed in the `size` field).
To preserve compatibility with older versions of Emacs, modules should check
the `size` field to verify that it is at least as large as expected, and react
accordingly if that is not the case.  To preserve compatibility with future
versions of Emacs, modules should not set a hard upper bound on the `size`
field.  Two different objects of the same structure type will always have the
same dynamic size, i.e., you have to check the `size` member only once per
structure type.


### Version comparison

Before calling runtime or environment functions, you must check whether the
Emacs binary your module is loaded in is new enough.  There are three ways to
do this:

1.  You can compare the static and the dynamic sizes of the `emacs_runtime` and
    `emacs_env` structures to verify that they are as large as you expect.  You
    need to do this in your `module_init` function before accessing any other
    fields of the structures.  The basic pattern looks as follows:

    ```c
    #include <assert.h>
    #include <stddef.h>

    #include <emacs-module.h>

    int
    module_init (struct emacs_runtime *ert)
    {
      assert (ert->size > 0);
      if ((size_t) ert->size < sizeof *ert)
        /* Dynamic size is smaller than static size. */
        return 1;
      emacs_env *env = ert->get_environment (ert);
      assert (env->size > 0);
      if ((size_t) env->size < sizeof *env)
        /* Dynamic size is smaller than static size. */
        return 2;
      /* Continue initialization. */
      return 0;
    }
    ```

    This makes sure that any field you can access is actually present.

2.  You can also compare the dynamic size of the environment structure against
    the fixed sizes of the versioned structures:

    ```c
    #include <assert.h>
    #include <stddef.h>

    #include <emacs-module.h>

    static int emacs_version;

    int
    module_init (struct emacs_runtime *ert)
    {
      assert (ert->size > 0);
      if ((size_t) ert->size < sizeof *ert)
        /* Dynamic size is smaller than static size. */
        return 1;
      emacs_env *env = ert->get_environment (ert);
      assert (env->size > 0);
      if ((size_t) env->size >= sizeof (struct emacs_env_26))
        /* All fields from Emacs 26 are present. */
        emacs_version = 26;
      else if ((size_t) env->size >= sizeof (struct emacs_env_25))
        /* All fields from Emacs 25 are present. */
        emacs_version = 25;
      else
        /* Unknown version. */
        return 2;
      /* Continue initialization. */
      return 0;
    }
    ```

    If you use this option, you must make sure to only access fields that are
    known to be present in the actual Emacs version.

3.  You can also check the presence of individual fields:

    ```c
    #include <assert.h>
    #include <stdbool.h>
    #include <stddef.h>

    #include <emacs-module.h>

    static bool have_intern;
    static bool have_funcall;

    int
    module_init (struct emacs_runtime *ert)
    {
      assert (ert->size > 0);
      if ((size_t) ert->size < sizeof *ert)
        /* Dynamic size is smaller than static size. */
        return 1;
      emacs_env *env = ert->get_environment (ert);
      assert (env->size > 0);
      /* Test whether ‘intern’ field is present. */
      have_intern = ((size_t) env->size
                     >= offsetof (emacs_env, intern) + sizeof env->intern);
      /* Test whether ‘funcall’ field is present. */
      have_funcall = ((size_t) env->size
                      >= offsetof (emacs_env, funcall) + sizeof env->funcall);
      /* More checks. */
      /* Continue initialization. */
      return 0;
    }
    ```

    If you use this option, you must make sure to only access fields that are
    known to be present.

Each of these options has advantages and disadvantages.  From the first to the
third option, both complexity and flexibility increase.  The first option is by
far the simplest one; it’s only a single comparison, and if you use it you can
be sure that you don’t accidentally access a field that’s not present.
However, it’s also the least flexible option: even if you don’t use any field
introduced in later versions of Emacs, your module will still refuse to load if
Emacs is not new enough to contain all the expected fields.  The second option
provides a compromise between complexity and compatibility; it allows you to
stay compatible with older versions of Emacs, but you have to remember to only
access structure fields that you know are present.  The third option is the
most flexible one, but requires enormous amounts of boilerplate code: you need
to check the presence of every single field you want to use.

If you aren’t concerned about staying compatible with old versions of Emacs, I
recommend that you use the first option.  If you want to make your module
available to older versions of Emacs, I recommend the second option.


## Module loading and initialization

Emacs loads modules by calling the `module-load` function.

A module must export a symbol named `plugin_is_GPL_compatible` to report its
GPL compatibility to Emacs; otherwise `module-load` signals an error of type
`module-not-gpl-compatible`.

A module must export a symbol named `emacs_module_init`; otherwise
`module-load` signals and error of type `missing-module-init-function`.

The symbol named `emacs_module_init` must point to a function with the
following signature:

```c
int emacs_module_init (struct emacs_runtime *runtime);
```

Emacs will call this function and pass a pointer to an object of type `struct
emacs_runtime`, which is defined as follows:

```c
struct emacs_runtime
{
  ptrdiff_t size;
  struct /* unspecified */ *private_members;
  emacs_env *(*get_environment) (struct emacs_runtime *runtime);
};
```

The lifetime of the runtime object begins not after the body of the module
initialization function is entered; it ends not before the module
initialization function returns.  Modules mustn’t make any further assumptions
about the lifetime of the runtime object.

The `size` field contains the size of the structure, in bytes.  The
`get_environment` field is a pointer to a function that returns an environment
pointer; module initialization functions may use that function to obtain an
initial environment.  Modules must pass a pointer to the same runtime object to
`get_environment` that has been passed to them.  The lifetime of the
environment returned by the `get_environment` field starts not after the call
to `get_environment` returns and ends not before the module initialization
function ends; modules mustn’t make any further assumption about its lifetime.

Modules must be prepared for any number of invocations of their initialization
function; it is unspecified whether two successful calls to `module-load` with
equivalent module file names will result in one or two invocations of the
initialization function.

After the module initialization function returns, Emacs will perform different
operations depending on the return value and the state of the environment
returned by `get_environment`:

-   If the user has requested a quit using <kbd>C-g</kbd> while the
    initialization function was running, Emacs will ignore the return value and
    the state of the initial environment and quit immediately.

-   Otherwise, if the initialization function has returned a nonzero value,
    `module-load` will signal an error of type `module-init-failed`.

-   Otherwise, if the environment returned by `get_environment` has a nonlocal
    exit pending, `module-load` will exit nonlocally as specified in the
    environment.

-   Otherwise, `module-load` returns `t`.

You might wonder why there are two different ways to report a failure.  The
reason is that there are cases where you can’t use the initial environment to
report errors: for example, if the module received a runtime or environment
structure of unknown size.  In such as case it would be unsafe to attempt to
use the environment structure to signal an error, but returning an integer is
always safe.


## Emacs values

The `emacs_value` type is defined as follows:

```c
typedef struct /* unspecified */ *emacs_value;
```

That is, an `emacs_value` is a pointer to an opaque structure.  Modules mustn’t
make any assumptions about the pointer or its structure; in particular, it is
unspecified whether `emacs_value` pointers point to a valid memory location,
whether `NULL` represents a valid Emacs Lisp object, or whether identical
Emacs Lisp objects are represented by equal pointers or not.


## Environments

The `emacs_env` type is a type alias for the following structure type:

```c
struct emacs_env_26
{
  ptrdiff_t size;
  struct /* unspecified */ *private_members;
  /* Pointers to environment functions. */
}

typedef struct emacs_env_26 emacs_env;
```

The number following `emacs_env_` is the Emacs major version in which the
structure was first defined.  For every Emacs major version, a corresponding
environment structure is available.  The versioned structures “inherit” from
each other in the following sense:

-   A later structure will contain exactly the same fields as an earlier
    structure in exactly the same order.

-   A later structure may contain additional fields after the fields from the
    earlier structure.

The `emacs_env` type alias is always an alias to the newest structure in
`emacs-module.h`.

`size` is the size of the object, in bytes.  It is guaranteed to be the first
field.  The other public fields are collectively called *environment
functions*.  They are described in the following subsections.

The function pointers in an environment structure remain valid as long as the
corresponding `emacs_env` pointer is in scope.  It’s unspecified whether the
some field has the same values in two different `emacs_env` structures.  You
must pass a pointer to the containing structure as the first argument to all
environment functions, for example:

```c
env->intern (env, "nil")
```

For the sake of simplicity, the prototypes below use the syntax for free
functions, not function pointers.  This is just to avoid additional parentheses
and asterisks that make the prototypes less readable.  For instance, the
function

```c
emacs_value intern (emacs_env *env, const char *symbol_name);
```

is actually a function pointer as structure field:

```c
struct emacs_env_25
{
  /* More fields. */
  emacs_value (*intern) (emacs_env *env, const char *symbol_name);
  /* More fields. */
}
```


### Nonlocal exits

Some programming language have the concept of **nonlocal exits**: a function
might not only return normally, but potentially “jump” to some other place in
the code, typically a different function higher up in the call stack.  The key
difference between normal (local) and nonlocal exits is that nonlocal exits can
jump to a position outside of the direct caller of the function; for example,
if a function *f* calls *g* and *g* calls *h*, then *h* might exit nonlocally
by jumping directly back into *f*.  The target of a nonlocal jump is generally
a *dynamic* property of the code, i.e. it’s known only at runtime.  Because a
nonlocal exit affects functions unrelated to the starting point and target of
the jump, there has to be a *global default assumption* whether functions can
exit nonlocally: code either assumes that no function exits nonlocally, or that
potentially all functions exit nonlocally.  Many well-known languages make the
latter assumption; examples are C++, Java, C#, or Python.  Emacs Lisp is also
in the second category; functions can exit nonlocally using `signal` or
`throw`.  Languages in the “nonlocal exit by default” category always provide
language constructs to protect against the effects of nonlocal exits; for
example, C++ has deterministic destructors, and other languages have
`try`–`finally` or similar facilities.  Such **unwind protection** is essential
if you have to assume that nonlocal exits can happen at any time; otherwise, it
would be too difficult to keep data structures consistent, prevent
synchronization primitives from leaking, or clean up resources.

Nonlocal exits are a language feature that can be used for several purposes.
Probably the most well-known one is the use for error reporting, usually called
“exception handling.”  Emacs Lisp uses nonlocal exits for error reporting, but
also for non-erroneous control flow.

The major difficulty when writing dynamic modules is that in the C language
functions are by default assumed to always return normally.  Even though C has
the `setjmp` and `longjmp` functions for nonlocal jumps, it lacks an unwind
protection mechanism, thus nonlocal exits are rare in practice, and most C
codebases assume they don’t happen.  The difficulty arises at the interface
between a language with “nonlocal exit by default” semantics (Emacs Lisp) and a
language with “only normal return by default” semantics (C).  For this reason,
the functions of the module API never exit nonlocally; instead, the API
represents nonlocal exits using the environment-local **pending nonlocal exit
state**.  If a module or environment function wishes to signal a nonlocal exit,
it sets the pending error state using `non_local_exit_signal` or
`non_local_exit_throw`; you can access the pending error state using
`non_local_exit_check` and `non_local_exit_get`.

If a nonlocal exit is pending, calling any environment function other than the
functions used to manage nonlocal exits (i.e. those starting with
`non_local_exit_`) immediately returns an unspecified value without further
processing.  You can make use of this fact to occasionally skip explicit
nonlocal exit checks.

How a function exits is represented using the following enumeration:

```c
enum emacs_funcall_exit
{
  emacs_funcall_exit_return = 0,
  emacs_funcall_exit_signal = 1,
  emacs_funcall_exit_throw = 2
};
```

`emacs_funcall_exit_return` represents a local (normal) exit.
`emacs_funcall_exit_signal` represents an error signal raised by the `signal`
or `error` Lisp functions.  `emacs_funcall_exit_throw` represents a nonlocal
jump to a `catch` construct created by the `throw` Lisp function.


#### `non_local_exit_check`

Module functions can obtain the last function exit type for an environment
using `non_local_exit_check`:

```c
enum emacs_funcall_exit non_local_exit_check (emacs_env *env);
```

`non_local_exit_check` never fails and always returns normally.  If there is no
nonlocal exit pending, it returns the enumerator `emacs_funcall_exit_return`;
otherwise it returns one of the other enumerators.

`non_local_exit_check` is available since GNU Emacs 25.


#### `non_local_exit_get`

For nonlocal exits Emacs stores additional data.  You can retrieve this data
using `non_local_exit_get`:

```c
enum emacs_funcall_exit non_local_exit_get (emacs_env *env,
                                            emacs_value *symbol_or_tag,
                                            emacs_value *data_or_value);
```

Both *symbol_or_tag* and *data_or_value* must be non-`NULL`.  The return value
is the same as for `non_local_exit_check`.  In addition, Emacs fills
`*symbol_or_tag` and `*data_or_value` with additional information depending on
the return value:

-   If the return value is `emacs_funcall_exit_return`, the contents of
    `*symbol_or_tag` and `*data_or_value` after the call are unspecified.

-   If the return value is `emacs_funcall_exit_signal`, Emacs stores the error
    symbol in `*symbol_or_tag` and the error data in `*data_or_value`; that is,
    these values correspond to the two arguments of the `signal` Lisp function.

-   If the return value is `emacs_funcall_exit_throw`, Emacs stores the catch
    tag in `*symbol_or_tag` and the catch value in `*data_or_value`; that is,
    these values correspond to the two arguments of the `throw` Lisp function.

`non_local_exit_get` is available since GNU Emacs 25.


#### `non_local_exit_signal`

```c
void non_local_exit_signal (emacs_env *env, emacs_value symbol,
                            emacs_value data);
```

`non_local_exit_signal` is the module equivalent of the Lisp `signal` function:
it causes Emacs to signal an error of type *symbol* with error data *data*.
*data* should be a list.

`non_local_exit_signal`, like all other environment functions, actually returns
normally when seen as a C function.  Rather, it causes Emacs to signal an error
once you return from the current module function or module initialization
function.  Therefore you should typically return quickly after signaling an
error with this function.  If there was already a nonlocal exit pending when
calling `non_local_exit_signal`, the function does nothing; i.e. it doesn’t
overwrite the error symbol and data.  To do that, you must explicitly call
`non_local_exit_clear` first.

`non_local_exit_signal` is available since GNU Emacs 25.


#### `non_local_exit_throw`

```c
void non_local_exit_throw (emacs_env *env, emacs_value tag, emacs_value value);
```

`non_local_exit_throw` is the module equivalent of the Lisp `throw` function:
it causes Emacs to perform a nonlocal jump to a `catch` block tagged with
*tag*; the catch value will be *value*.

`non_local_exit_throw`, like all other environment functions, actually returns
normally when seen as a C function.  Rather, it causes Emacs to throw to the
catch lock once you return from the current module function or module
initialization function.  Therefore you should typically return quickly after
requesting a jump with this function.  If there was already a nonlocal exit
pending when calling `non_local_exit_throw`, the function does nothing; i.e. it
doesn’t overwrite catch tag and value.  To do that, you must explicitly call
`non_local_exit_clear` first.

`non_local_exit_throw` is available since GNU Emacs 25.


#### `non_local_exit_clear`

```c
void non_local_exit_clear (emacs_env *env);
```

`non_local_exit_clear` resets the pending-error state of *env*.  After calling
`non_local_exit_clear`, `non_local_exit_check` will again return
`emacs_funcall_exit_return`, and Emacs won’t signal an error after returning
from the current module function or module initialization function.  You can
use `non_local_exit_clear` to ignore certain kinds of errors.  You can also
transform errors into different errors by calling `non_local_exit_get`,
`non_local_exit_clear`, and `non_local_exit_signal` in sequence.

`non_local_exit_clear` is available since GNU Emacs 25.


#### How to deal with nonlocal exits properly

The return value of the environment functions doesn’t indicate whether a
nonlocal exit is pending.  The only exception is `copy_string_contents`; for
all other functions you have to call `non_local_exit_check` or
`non_local_exit_get` to find out whether they have returned normally.

The saturating behavior of nonlocal exits gives rise to two error handling
idioms:

1.  You can call `non_local_exit_check` after each and every call to an environment
    function.  That way you can determine with certainty whether the function
    call has exited normally.  This is simple, but requires a lot of
    boilerplate code.  When choosing this option, you might want to wrap the
    environment functions in wrapper functions that call `non_local_exit_check`
    for you, for example:

    ```c
    #include <stdbool.h>
    #include <stdint.h>

    #include <emacs-module.h>

    static bool
    extract_integer (emacs_env *env, emacs_value value, intmax_t *num)
    {
      *num = env->extract_integer (env, value);
      return env->non_local_exit_check (env) == emacs_funcall_exit_return;
    }
    ```

2.  You can call `non_local_exit_check` only before “important” operations.  An
    operation in your code is “important” if it’s a decision based on Emacs
    values, has a side effect, or can take a long time.  For example, in the
    following function you have to insert checks before the `if` statement and
    the `puts` function call:

    ```c
    #include <assert.h>
    #include <stddef.h>
    #include <stdint.h>
    #include <stdio.h>

    #include <emacs-module.h>

    static emacs_value
    test_number_sign (emacs_env *env, ptrdiff_t nargs, emacs_value *args,
                      void *data)
    {
      assert (nargs == 1);
      intmax_t num = env->extract_integer (env, args[0]);
      if (env->non_local_exit_check (env) != emacs_funcall_exit_return)
        return NULL;
      if (num > 0)
        printf ("%jd is positive\n", num);
      else if (num < 0)
        printf ("%jd is negative\n", num);
      else
        printf ("%jd is zero\n", num);
      emacs_value ret = env->make_integer (env, num);
      if (env->non_local_exit_check (env) != emacs_funcall_exit_return)
        return NULL;
      puts ("Success!");
      return ret;
    }
    ```

    If you remove the first check, the program output becomes unpredictable.
    If you had remove the second check, the program prints “Success!” even if
    `make_integer` fails.  In such a simple case this might not seem like a big
    deal, but imagine if instead of `printf` you had added code to delete
    files, send data to the Internet, or started a long-running calculation.
    Therefore you can’t dispense with error checking in all but the most
    trivial cases.  On the other hand, it’s safe to leave out the error
    checking in the following example:

    ```c
    #include <assert.h>
    #include <stddef.h>
    #include <stdint.h>
    #include <string.h>

    #include <emacs-module.h>

    static emacs_value
    locate_config_file (emacs_env *env, ptrdiff_t nargs, emacs_value *args,
                        void *data)
    {
      assert (nargs == 2);
      emacs_value home_dir = args[0];
      emacs_value global_dir = args[1];
      const char *name = "myconfig.conf";
      const size_t name_len = strlen (name);
      assert (name_len <= PTRDIFF_MAX);
      emacs_value list_args[] = {home_dir, global_dir};
      emacs_value locate_args[] = {
        env->make_string (env, name, (ptrdiff_t) name_len),
        env->funcall (env, env->intern (env, "list"), 2, list_args)
      };
      return env->funcall (env, env->intern (env, "locate-file"),
                           2, locate_args);
    }
    ```

    All of the environment functions used in this snippet can exit nonlocally,
    but no nonlocal exit can cause any difference in behavior because there are
    no “important” operations that depend on the outcome of any function.  For
    instance, consider what happens if the `make_string` call and the first
    `intern` call succeed, but the `funcall` to `list` fails: the second
    `intern` and `funcall` combination will just do nothing at all, as if the
    code weren’t there.  This is exactly the behavior you’d get if you inserted
    a `return` conditioned on a `non_local_exit_check` after the first
    `funcall`.

If you’re unsure what to do, or you don’t have yet enough practice with the
module API, then I’d recommend following the first approach and check for
nonlocal exits after each environment function call.  Analyzing whether
leaving out a nonlocal exit check would incur an observable behavior change
can be tricky.  However, there’s one case where the first idiom just adds noise
without making the code simpler: when returning from a module function.  For
example, theoretically you could write

```c
emacs_value nil = env->intern (env, "nil");
if (env->non_local_exit_check (env) != emacs_funcall_exit_return)
  return NULL;
return nil;
```

instead of

```c
return env->intern (env, "nil");
```

but there wouldn’t be any benefit to it: because you are returning from the
module function, there’s no possibility that you could accidentally ignore a
nonlocal exit, and Emacs will check for a nonlocal exit anyway directly after
returning from the function, so you’ve just added a completely pointless check.

If you don’t like the API’s nonlocal exit behavior, you can wrap the
environment functions.  There are a couple of other snippets in this document
that show how to wrap some of them in functions returning `bool` so you don’t
have to call `non_local_exit_check` all the time.  To give a different option,
the following example shows how to wrap a single environment function to get
rid of the nonlocal exit state and the saturating behavior:

```c
#include <stdbool.h>
#include <stdint.h>

#include <emacs-module.h>

struct nonlocal_exit
{
  enum emacs_funcall_exit exit;
  emacs_value symbol_or_tag;
  emacs_value data_or_value;
};

static bool
put_exit (emacs_env *env, struct nonlocal_exit *exit)
{
  exit->exit = env->non_local_exit_get (env, &exit->symbol_or_tag,
                                        &exit->data_or_value);
  env->non_local_exit_clear (env);
  return exit->exit == emacs_funcall_exit_return;
}

static bool
make_integer (emacs_env *env, intmax_t value, emacs_value *result,
              struct nonlocal_exit *nonlocal_exit)
{
  *result = env->make_integer (env, value);
  return put_exit (env, nonlocal_exit);
}
```

Most environment functions can request nonlocal exits.  In particular, most
will use signals to signal errors.  This document calls out explicitly those
functions that never exit nonlocally; you have to assume that all other
functions can exit nonlocally.  Note that even the functions that don’t exit
nonlocally themselves still do nothing and return an unspecified value if a
nonlocal exit was pending when calling them.

This document lists some of the error symbols signaled by environment
functions.  However, it’s not an exhaustive description: environment functions
are free to signal other errors not specified here.  In particular, environment
function will typically signal `memory-full` if they can’t allocate memory, and
`overflow-error` if some numeric cast would overflow the target type.  These
aren’t listed specifically.


### Global references

As explained above, most Emacs values have a short lifetime that ends once
their owning `emacs_env` pointer goes out of scope.  However, occasionally it’s
useful to have values with a longer lifetime:

-   You might want to store some global piece of data that should outlive the
    current function call, similar to Emacs dynamic variables.

-   You have determined that creating some objects over and over again incurs a
    too high CPU cost, so you want to create the object only once.  A good
    example is interning commonly-used symbols such as `car`.

For such use cases the module API provides **global references**.  They are
normal `emacs_value` objects, with one key difference: they are not bound to
the lifetime of any environment.  Rather, you can use them, once created,
whenever *any* environment is active.

Be aware that using global references, like all global state, incurs a
readability cost on your code: with global references, you have to keep track
which parts of your code modify which reference.  You are also responsible for
managing the lifetime of global references, whereas local values go out of
scope manually.  Therefore I recommend to avoid global references as much as
possible and use them only sparingly.


#### `make_global_ref`

```c
emacs_value make_global_ref (emacs_env *env, emacs_value value);
```

`make_global_ref` returns a new global reference for *value*.  *value* can be
any valid local or global reference.  It’s unspecified whether the return value
is equal to *value*.  It’s also unspecified whether two calls to
`make_global_ref` with the same *value* have the same return value.

`make_global_ref` is available since GNU Emacs 25.


#### `free_global_ref`

```c
void free_global_ref (emacs_env *env, emacs_value global_ref);
```

`free_global_ref` frees a global reference previously returned by
`make_global_ref`.  If *global_ref* is a local value or a global reference
that’s already been freed, nothing happens.  Otherwise, the global reference
will no longer be valid after the call.

If two calls to `make_global_ref` have returned the same value and it hasn’t
been freed in the meantime, you also have to call `free_global_ref` twice on
the value; that is, global references are reference-counted.

`free_global_ref` is available since GNU Emacs 25.


### Basic object tests

#### `is_not_nil`

```c
bool is_not_nil (emacs_env *env, emacs_value value);
```

`is_not_nil` returns whether the Lisp object represented by *value* is not
`nil`.  It never exits nonlocally.  There can be multiple different values that
represent `nil`.  It’s unspecified whether a `NULL` value represents `nil` (or
any other valid Lisp object, for that matter).

`is_not_nil` is available since GNU Emacs 25.


#### `eq`

```c
bool eq (emacs_env *env, emacs_value a, emacs_value b);
```

`eq` returns whether *a* and *b* represent the same Lisp object.  It never
exits nonlocally.  Note that `a == b` always implies `env->eq (env, a, b)`,
but the reverse is not true: Two `emacs_value` objects that are different in
the C sense might still represent the same Lisp object, so you must always call
`eq` to check for equality.

`eq` corresponds to the Lisp `eq` function.  For other kinds of equality
comparisons, such as `=`, `eql`, or `equal`, use `intern` and `funcall` to call
the corresponding Lisp function.

`eq` is available since GNU Emacs 25.


#### `type_of`

```c
emacs_value type_of (emacs_env *env, emacs_value value);
```

`type_of` returns the type of *value* as a Lisp symbol.  It corresponds exactly
to the `type-of` Lisp function, which see.

`type_of` is available since GNU Emacs 25.


### Type conversion

The environment functions described in this section convert various values
between C and Emacs.


#### `make_integer`

```c
emacs_value make_integer (emacs_env *env, intmax_t value);
```

`make_integer` creates an Emacs integer object from a C integer value.  If the
*value* can’t be represented as an Emacs integer, Emacs signals an error of
type `overflow-error`.

`make_integer` is available since GNU Emacs 25.


#### `extract_integer`

```c
intmax_t extract_integer (emacs_env *env, emacs_value value);
```

`extract_integer` returns the integral value stored in an Emacs integer object.
If *value* doesn’t represent an integer object, Emacs signals an error of type
`wrong-type-argument`.  If the integer represented by *value* can’t be
represented as `intmax_t`, Emacs signals an error of type `overflow-error`.

`extract_integer` is available since GNU Emacs 25.


#### `make_float`

```c
emacs_value make_float (emacs_env *env, double value);
```

`make_float` creates an Emacs floating-point number from a C floating-point
value.

`make_float` is available since GNU Emacs 25.


#### `extract_float`

```c
double extract_float (emacs_env *env, emacs_value value);
```

`extract_float` returns the value stored in an Emacs floating-point number.  If
*value* doesn’t represent a floating-point object, Emacs signals an error of
type `wrong-type-argument`.

`extract_float` is available since GNU Emacs 25.


#### `make_string`

```c
emacs_value make_string (emacs_env *env, const char *contents,
                         ptrdiff_t length);
```

`make_string` creates a multibyte Lisp string object.  *length* must be
nonnegative.  *contents* must point to an array of at least *length* + 1
characters, and `contents[length]` must be the null character.

If *length* is negative or larger than the maximum allowed Emacs string length,
Emacs raises an `overflow-error` signal.  Otherwise, Emacs treats the memory at
*contents* as the UTF-8 representation of a string.

If the memory block delimited by *contents* and *length* contains a valid UTF-8
string, the return value will be a multibyte Lisp string that contains the same
sequence of Unicode scalar values as represented by *contents*.  Otherwise, the
return value will be a multibyte Lisp string with unspecified contents; in
practice, Emacs will attempt to detect as many valid UTF-8 subsequences in
*contents* as possible and treat the rest as undecodable bytes, but you
shouldn’t rely on any specific behavior in this case.

The returned Lisp string will not contain any text properties.  To create a
string containing text properties, use `funcall` to call functions such as
`propertize`.

`make_string` can’t create strings that contain characters that are not valid
Unicode scalar values.  Such strings are rare, but occur from time to time;
examples are strings with UTF-16 surrogate code points or
strings with extended Emacs characters that don’t correspond to Unicode code
points.  To create such a Lisp string, call e.g. the function `string` and pass
the desired character values as integers.

Because the behavior of `make_string` is unpredictable if *contents* is not a
valid UTF-8 string, you might want to provide a higher-level wrapper function
that checks whether it’s a valid UTF-8 string first, for example:

```c
#include <stdbool.h>
#include <stddef.h>
#include <stdint.h>

#include <unistr.h>  /* from libunistring or Gnulib */

#include <emacs-module.h>

static bool
make_string (emacs_env *env, const char *contents, size_t size,
             emacs_value *result)
{
  if (size > PTRDIFF_MAX)
    {
      env->non_local_exit_signal (env, env->intern (env, "overflow-error"),
                                  env->intern (env, "nil"));
      return false;
    }
  if (u8_check ((const uint8_t *) contents, size) != NULL)
    {
      env->non_local_exit_signal (env, env->intern (env, "wrong-type-argument"),
                                  env->intern (env, "nil"));
      return false;
    }
  *result = env->make_string(env, contents, (ptrdiff_t) size);
  return env->non_local_exit_check (env) == emacs_funcall_exit_return;
}
```

`make_string` is available since GNU Emacs 25.


#### `copy_string_contents`

```c
bool copy_string_contents(emacs_env *env, emacs_value value,
                          char *buffer, ptrdiff_t *size);
```

The function `copy_string_contents` copies the characters in the Lisp string
*value* into *buffer*.  *buffer* may be `NULL`, but *size* must not be `NULL`.

If *value* doesn’t represent a Lisp string, Emacs signals an error of type
`wrong-type-argument`.

If *buffer* is `NULL`, Emacs stores the required size for *buffer* in `*size`
and returns `true`.  The required size includes space for a terminating null
character; it will be at most `SIZE_MAX`.

If *buffer* is not `NULL`, `*size` must be positive, and *buffer* must point to
an array of at least `*size` characters.  If `*size` is nonpositive or less
than the required buffer size (including a terminating null character), Emacs
stores the required size in `*size`, signals an error of type
`args-out-of-range`, and returns `false`.  Otherwise, Emacs copies the UTF-8
representation of the characters contained in *value* to the array that
*buffer* points to and returns `true`.  The contents of *buffer* will include a
terminating null byte at `buffer[*size - 1]`.  If *value* contains only Unicode
scalar values (i.e. it’s either a unibyte string containing only ASCII
characters or a multibyte string containing only characters that are Unicode
scalar values), the string stored in *buffer* will be a valid UTF-8 string
representing the same sequence of scalar values as *value*.  Otherwise, the
contents of *buffer* are unspecified; in practice, Emacs attempts to convert
scalar values to UTF-8 and leaves other bytes alone, but you shouldn’t rely on
any specific behavior in this case.

After returning from `copy_string_contents`, a nonlocal exit is pending if and
only if the return value is `false`.

Emacs strings can contain null characters, and therefore *buffer* may also
contain null characters.  Using `strlen` on *buffer* can result in a length
that’s too short; the actual length will be `*size` − 1.

There’s no environment function to extract string properties.  Use the usual
Emacs functions such as `get-text-property` for that.

To deal with strings that don’t represent sequences of Unicode scalar values,
you can use Emacs functions such as `length` and `aref` to extract the
character values directly.

You might want to wrap `copy_string_contents` in a function that allocates a
buffer of the appropriate size so that you don’t have to call it twice:

```c
#include <assert.h>
#include <stdbool.h>
#include <stddef.h>
#include <stdint.h>
#include <stdlib.h>

#include <emacs-module.h>

static bool
copy_string_contents (emacs_env *env, emacs_value value,
                      char **buffer, size_t *size)
{
  ptrdiff_t buffer_size;
  if (!env->copy_string_contents (env, value, NULL, &buffer_size))
    return false;
  assert (env->non_local_exit_check (env) == emacs_funcall_exit_return);
  assert (buffer_size > 0);
  *buffer = malloc ((size_t) buffer_size);
  if (*buffer == NULL)
    {
      env->non_local_exit_signal (env, env->intern (env, "memory-full"),
                                  env->intern (env, "nil"));
      return false;
    }
  ptrdiff_t old_buffer_size = buffer_size;
  if (!env->copy_string_contents (env, value, *buffer, &buffer_size))
    {
      free (*buffer);
      *buffer = NULL;
      return false;
    }
  assert (env->non_local_exit_check (env) == emacs_funcall_exit_return);
  assert (buffer_size == old_buffer_size);
  *size = (size_t) (buffer_size - 1);
  return true;
}
```

When you use this function, be sure to call `free` on the returned buffer after
use.

If you call `copy_string_contents` passing a Lisp string that only contains
Unicode scalar values and then call `make_string` on the filled buffer, Emacs
will create a string that’s equal (in the sense of `string-equal`) to the
initial string, but text properties are lost.  Likewise, if you call
`make_string` passing a valid UTF-8 string and then call `copy_string_contents`
on the result, Emacs will produce an UTF-8 string that’s byte-by-byte identical
to the initial UTF-8 string.

`copy_string_contents` is available since GNU Emacs 25.


### Interning

#### `intern`

```c
emacs_value intern (emacs_env *env, const char *symbol_name);
```

The function `intern` behaves like the Lisp function `intern`: it looks up
`symbol_name` in the default obarray; if a symbol with that name is already
interned in the obarray, it’s returned, otherwise a new symbol is created and
interned in the obarray.  *symbol_name* must be non-`NULL` and point to a
null-terminated C string.  The string that *symbol_name* points to must contain
only ASCII characters (i.e. characters in the range from 1 to 127); otherwise
it’s unspecified which symbol is looked up and/or interned.

Because the behavior is unpredictable if *symbol_name* is not an ASCII-only
string, you might want to create a higher-level wrapper function for `intern`.
That wrapper function only calls `intern` directly if the symbol name is an
ASCII string and falls back to calling the `intern` Lisp function otherwise:

```c
#include <stdbool.h>
#include <stddef.h>

#include <c-ctype.h>  /* from Gnulib */

#include <emacs-module.h>

static bool
intern (emacs_env *env, const char *name, size_t size, emacs_value *result)
{
  bool simple = true;
  for (size_t i = 0; i < size; ++i)
    if (name[i] == '\0' || !c_isascii (name[i]))
      {
        simple = false;
        break;
      }
  if (simple)
    *result = env->intern (env, name);
  else
    {
      emacs_value string_object;
      /* ‘make_string’ from above. */
      if (!make_string (env, name, size, &string_object))
        return false;
      *result = env->funcall (env, env->intern (env, "intern"),
                              1, &string_object);
    }
  return env->non_local_exit_check (env) == emacs_funcall_exit_return;
}
```


### Function definition

The primary purpose of the module API is to allow you to make C functions
available to Emacs; such functions are called **module functions**.  They have
the following signature:

```c
emacs_value
my_module_function (emacs_env *env, ptrdiff_t nargs,
                    emacs_value *args, void *data)
{
  /* Your code. */
}
```

Within the body of this function, you can use the *env* argument to convert
between Lisp values and C values or interact with Emacs.  The *env* pointer is
unique and different from all other environment pointers that are active at the
same time.  After the module function returns, Emacs will perform different
operations depending on the state of the environment represented by *env*:

-   If the user has requested a quit using <kbd>C-g</kbd> while the module
    function was running, Emacs will ignore both the return value and the state
    of the environment represented by *env* and quit immediately.  Note,
    however, that such quits don’t cause module functions to return; you have
    to actively call `should_quit` if you want to react on user quit requests.

-   Otherwise, if *env* has a nonlocal exit pending, Emacs will ignore the
    return value and exit nonlocally as specified in the environment.  This
    means that in the case of a nonlocal exit you can safely return a dummy
    value such as `NULL` without checking whether it represents a valid Lisp
    object.

-   Otherwise, the return value of the call is the Lisp object represented by
    the module function return value.  In this case, the return value must
    obviously represent a valid Lisp object.  If you don’t have a specific
    value to return, simply return `nil`:

    ```c
    return env->intern (env, "nil");
    ```

    Note that there’s a theoretical chance that the call to `intern` itself
    fails; then Emacs would signal an appropriate error instead of returning
    `nil`.

*nargs* is the number of arguments to the function; it is always nonnegative.
You can further restrict the allowed number of arguments using the *min_arity*
and *max_arity* parameters of `make_function`, which see.  Emacs will never
call a module function with a number of arguments that wouldn’t be allowed by
the arguments passed to `make_function`.  If *nargs* is positive, *args* will
point to an array of at least *nargs* elements: the argument values to the
function.  You must not modify the contents of the *args* array, even though
it’s not declared `const`.  If *nargs* is zero, the value of *args* is
unspecified; that means you mustn’t dereference it.


#### `make_function`

```c
typedef emacs_value (*emacs_subr) (emacs_env *env,
                                   ptrdiff_t nargs, emacs_value *args,
                                   void *data);

emacs_value make_function (emacs_env *env,
                           ptrdiff_t min_arity, ptrdiff_t max_arity,
                           emacs_subr function, const char *documentation,
                           void *data);
```

`make_function` creates an Emacs function from a C function.  This is how you
expose functionality from your module to Emacs.  To use it, you need to define
a module function and pass its address as the *function* argument to
`make_function`.  *min_arity* and *max_arity* must be nonnegative numbers, and
*max_arity* must be greater than or equal to *min_arity*.  Alternatively,
*max_arity* can have the special value `emacs_variadic_function`; in this case
the function accepts an unbounded number of arguments, like functions defined
with `&rest` in Lisp.  The value of `emacs_variadic_function` is a negative
number.  When applied to a function object returned by `make_function`, the
Lisp function `subr-arity` will return
<code>(<var>min_arity</var> . <var>max_arity</var>)</code> if *max_arity* is
nonnegative, or <code>(<var>min_arity</var> . many)</code> if *max_arity* is
`emacs_variadic_function`.

Emacs passes the value of the *data* argument that you give to `make_function`
back to your module function, but doesn’t touch it in any other way.  You can
use *data* to pass additional context to the module function.  If *data* points
to an object, you are responsible to ensure that the object is still live when
Emacs calls the module function.

*documentation* can either be `NULL` or a pointer to a null-terminated string.
If it’s `NULL`, the new function won’t have a documentation string.  If it’s
not `NULL`, Emacs interprets it as an UTF-8 string and uses it as documentation
string for the new function.  If it’s not a valid UTF-8 string, the
documentation string for the new function is unspecified.

The documentation string can end with a special string to specify the argument
names for the function.  See [Documentation Strings of Functions in the Emacs
Lisp reference
manual](https://www.gnu.org/software/emacs/manual/html_node/elisp/Function-Documentation.html)
for the syntax.

The function returned by `make_function` isn’t bound to a symbol.  For the
common case that you want to create a function object and bind it to a symbol
so that Lisp code can call it by name, you might want to add a wrapper function
that combines `make_function` with `defalias`, similar to the `defun`
Lisp function:

```c
#include <stdbool.h>
#include <stddef.h>
#include <string.h>

#include <unistr.h>  /* from libunistring or Gnulib */

#include <emacs-module.h>

typedef emacs_value (*emacs_subr) (emacs_env *env,
                                   ptrdiff_t nargs, emacs_value *args,
                                   void *data);

static bool
defun (emacs_env *env, const char *symbol_name,
       ptrdiff_t min_arity, ptrdiff_t max_arity, emacs_subr function,
       const char *documentation, void *data)
{
  emacs_value symbol;
  /* ‘intern’ from above. */
  if (!intern (env, symbol_name, strlen (symbol_name), &symbol))
    return false;
  if (documentation != NULL
      && u8_check ((const uint8_t *) documentation,
                   strlen (documentation)) != NULL)
    {
      env->non_local_exit_signal (env, env->intern (env, "wrong-type-argument"),
                                  env->intern (env, "nil"));
      return false;
    }
  emacs_value func = env->make_function (env, min_arity, max_arity,
                                         function, documentation, data);
  emacs_value args[] = {symbol, func};
  env->funcall (env, env->intern (env, "defalias"), 2, args);
  return env->non_local_exit_check (env) == emacs_funcall_exit_return;
}
```

`make_function` is less general than `defun` or other Lisp facilities to create
functions.  In particular, it doesn’t support the following types of functions:

-   Interactive functions.  To define such a function, wrap it in another
    function:

    ```c
    #include <stdbool.h>
    #include <stddef.h>
    #include <string.h>

    #include <emacs-module.h>

    typedef emacs_value (*emacs_subr) (emacs_env *env,
                                       ptrdiff_t nargs, emacs_value *args,
                                       void *data);

    static bool
    defun_interactive (emacs_env *env, const char *symbol_name,
                       ptrdiff_t min_arity, ptrdiff_t max_arity,
                       emacs_value interactive, emacs_subr function,
                       const char *documentation, void *data)
    {
      emacs_value symbol;
      /* ‘intern’ from above. */
      if (!intern (env, symbol_name, strlen (symbol_name), &symbol))
        return false;
      /* ‘make_string’ from above. */
      emacs_value doc;
      if (!make_string (env, documentation, strlen (documentation), &doc))
        return false;
      emacs_value func = env->make_function (env, min_arity, max_arity,
                                             function, NULL, data);
      /* Now build up and evaluate the following form:

         (eval '(defun SYMBOL (&rest args)
                  DOCUMENTATION
                  (interactive INTFORM)
                  (apply FUNC ARGS))
               t)  */
      emacs_value list = env->intern (env, "list");
      emacs_value args = env->intern (env, "args");
      emacs_value arglist_elems[] = {env->intern (env, "&rest"), args};
      emacs_value arglist = env->funcall (env, list, 2, arglist_elems);
      emacs_value int_elems[] = {env->intern (env, "interactive"),
                                 interactive};
      emacs_value int_form = env->funcall (env, list, 2, int_elems);
      emacs_value body_elems[] = {env->intern (env, "apply"), func, args};
      emacs_value body = env->funcall (env, list, 3, body_elems);
      emacs_value form_elems[] = {env->intern (env, "defun"), symbol,
                                  arglist, doc, int_form, body};
      emacs_value form = env->funcall (env, list, 6, form_elems);
      emacs_value eval_args[] = {form, env->intern (env, "t")};
      env->funcall (env, env->intern (env, "eval"), 2, eval_args);
      return env->non_local_exit_check (env) == emacs_funcall_exit_return;
    }
    ```

-   Macros or special forms that don’t evaluate their arguments.  To define a
    macro, evaluate a `defmacro` form:

    ```c
    #include <stdbool.h>
    #include <stddef.h>
    #include <string.h>

    #include <emacs-module.h>

    typedef emacs_value (*emacs_subr) (emacs_env *env,
                                       ptrdiff_t nargs, emacs_value *args,
                                       void *data);

    static bool
    defmacro (emacs_env *env, const char *symbol_name,
              ptrdiff_t min_arity, ptrdiff_t max_arity, emacs_subr function,
              const char *documentation, void *data)
    {
      emacs_value symbol;
      /* ‘intern’ from above. */
      if (!intern (env, symbol_name, strlen (symbol_name), &symbol))
        return false;
      /* ‘make_string’ from above. */
      emacs_value doc;
      if (!make_string (env, documentation, strlen (documentation), &doc))
        return false;
      emacs_value func = env->make_function (env, min_arity, max_arity,
                                             function, NULL, data);
      /* Now build up and evaluate the following form:

         (eval '(defmacro SYMBOL (&rest args)
                  DOCUMENTATION
                  nil
                  (apply FUNC args))
               t)  */
      emacs_value list = env->intern (env, "list");
      emacs_value args = env->intern (env, "args");
      emacs_value arglist_elems[] = {env->intern (env, "&rest"), args};
      emacs_value arglist = env->funcall (env, list, 2, arglist_elems);
      emacs_value body_elems[] = {env->intern (env, "apply"), func, args};
      emacs_value body = env->funcall (env, list, 2, body_elems);
      emacs_value form_elems[] = {env->intern (env, "defmacro"), symbol,
                                  arglist, doc, env->intern (env, "nil"),
                                  body};
      emacs_value form = env->funcall (env, list, 6, form_elems);
      emacs_value eval_args[] = {form, env->intern (env, "t")};
      env->funcall (env, env->intern (env, "eval"), 2, eval_args);
      return env->non_local_exit_check (env) == emacs_funcall_exit_return;
    }
    ```

-   Functions with declare forms.  You can get the same effect by applying the
    elements of `defun-declarations-alist` manually:

    ```c
    #include <stdbool.h>
    #include <string.h>

    #include <emacs-module.h>

    static bool
    apply_declaration (emacs_env *env,
                       const char *function_name, emacs_value arglist,
                       const char *property, emacs_value values)
    {
      emacs_value func_symbol;
      /* ‘intern’ from above. */
      if (!intern (env, function_name, strlen (function_name), &func_symbol))
        return false;
      emacs_value prop_symbol;
      if (!intern (env, property, strlen (property), &prop_symbol))
        return false;
      /* Evaluate the following form:

         (eval (apply (cadr (assq PROPERTY defun-declarations-alist))
                      FUNCTION ARGLIST VALUES)
               t)  */
      emacs_value assq_args[] = {
        prop_symbol,
        env->intern (env, "defun-declarations-alist")
      };
      emacs_value element
        = env->funcall (env, env->intern (env, "assq"), 2, assq_args);
      emacs_value declarator
        = env->funcall (env, env->intern (env, "cadr"), 1, &element);
      emacs_value apply_args[] = {declarator, func_symbol, arglist, values};
      emacs_value form
        = env->funcall (env, env->intern (env, "apply"), 4, apply_args);
      emacs_value eval_args[] = {form, env->intern (env, "t")};
      env->funcall (env, env->intern (env, "eval"), 2, eval_args);
      return env->non_local_exit_check (env) == emacs_funcall_exit_return;
    }
    ```

-   Functions with documentation strings that can’t be represented in Unicode
    or contain embedded null characters.  I assume that such functions are
    extremely rare.

The `emacs_subr` type alias is not part of `emacs-module.h`, so you have to
define it yourself if you want it.

`make_function` is available since GNU Emacs 25.


#### `funcall`

```c
emacs_value funcall (emacs_env *env, emacs_value function,
                     ptrdiff_t nargs, emacs_value* args);
```

`funcall` corresponds to the Lisp `funcall` function: it calls any function,
passing it the arguments you provide.  *function* may represent any valid
function, such as Lisp lambdas, C subroutines, or module functions returned by
`make_function`.  It can also be a symbol; Emacs will find its function
definition (like `indirect-function`) and call that.  *nargs* must be
nonnegative.  *args* must point to an array of at least *nargs* elements; Emacs
uses the first *nargs* elements as arguments to *function*.  If *nargs* is
zero, *args* may also be `NULL`.  After `funcall` returns, the contents of the
first *nargs* elements of the array that *args* points to are unspecified;
i.e. if you need the array contents later you have to make a copy before
invoking `funcall`.  `funcall` returns the return value of *function*.  If
*function* exits nonlocally, the return value is unspecified.  Use
`non_local_exit_check` or `non_local_exit_get` to check whether a nonlocal exit
is pending.  You can’t use `funcall` to expand special forms or macros; use
functions such as `eval` or `macroexpand` for that.

For the common case of calling a function though a symbol, you might consider
adding a wrapper function, such as:

```c
#include <stdbool.h>
#include <stddef.h>
#include <stdint.h>
#include <stdlib.h>

#include <emacs-module.h>

static bool
funcall_symbol (emacs_env *env, const char *symbol,
                size_t nargs, const emacs_value* args,
                emacs_value *result)
{
  emacs_value symbol_value;
  /* ‘intern’ from above. */
  if (!intern (env, symbol, &symbol_value))
    return false;
  if (nargs > PTRDIFF_MAX)
    {
      env->non_local_exit_signal (env, env->intern (env, "overflow-error"),
                                  env->intern (env, "nil"));
      return false;
    }
  emacs_value *args_copy;
  if (nargs > 0)
    {
      args_copy = calloc (nargs, sizeof args[0]);
      if (args_copy == NULL)
        {
          env->non_local_exit_signal (env, env->intern (env, "memory-full"),
                                      env->intern (env, "nil"));
          return false;
        }
      for (size_t i = 0; i < nargs; ++i)
        args_copy[i] = args[i];
    }
  else
    args_copy = NULL;
  *result = env->funcall (env, symbol_value, (ptrdiff_t) nargs, args_copy);
  free (args_copy);
  return env->non_local_exit_check (env) == emacs_funcall_exit_return;
}
```

`funcall` is available since GNU Emacs 25.


### Vector access

The module API provides direct access to vector elements without using
`funcall`.


#### `vec_get`

```c
emacs_value vec_get (emacs_env *env, emacs_value vec, ptrdiff_t index);
```

`vec_get` returns the *index*-th element of the vector *vec*.  *index* is
zero-based.  If *vec* is not a Lisp vector, Emacs signals an error of type
`wrong-type-argument`.  If *index* is negative or not less than the number of
elements in *vec*, Emacs signals an error of type `args-out-of-range`.

`vec_get` is available since GNU Emacs 25.


#### `vec_set`

```c
void vec_set (emacs_env *env, emacs_value vec, ptrdiff_t index,
              emacs_value value);
```

`vec_set` sets the *index*-th element of the vector *vec* to *value*.  *index*
is zero-based.  If *vec* is not a Lisp vector, Emacs signals an error of type
`wrong-type-argument`.  If *index* is negative or not less than the number of
elements in *vec*, Emacs signals an error of type `args-out-of-range`.

`vec_set` is available since GNU Emacs 25.


#### `vec_size`

```c
ptrdiff_t vec_size (emacs_env *env, emacs_value vec);
```

`vec_size` returns the number of elements in the vector *vec*.  If *vec* is not
a Lisp vector, Emacs signals an error of type `wrong-type-argument`.

`vec_size` is available since GNU Emacs 25.


### User pointers

When dealing with C code, it’s often useful to be able to store arbitrary C
objects inside Emacs Lisp objects.  For this purpose the module API provides a
unique Lisp datatype called **user pointer**.  A user pointer object
encapsulates a C pointer value and optionally a finalizer function.  Apart from
storing it, Emacs leaves the pointer value alone.  Even though it’s a pointer,
there’s no requirement that it point to valid memory.  If you provide a
finalizer, Emacs will call it when the user pointer object is garbage
collected.  Note that Emacs’s garbage collection is nondeterministic: it might
happen long after an object ceases to be used or not at all.  Therefore you
can’t use user pointer finalizers for finalization that has to be prompt or
deterministic; it’s best to use finalizers only for clean-ups that can be
delayed arbitrarily without bad side effects, such as freeing memory.  If you
store a resource handle in a user pointer that requires deterministic
finalization, you should use a different mechanism such as `unwind-protect`.
Finalizers can’t interact with Emacs in any way; they also can’t fail.

#### `make_user_ptr`

```c
typedef void (*emacs_finalizer) (void *ptr);
emacs_value make_user_ptr (emacs_env *env, emacs_finalizer fin, void *ptr);
```

`make_user_ptr` creates and returns a new user pointer object.  *ptr* is the
pointer value to be embedded in the user pointer; it’s completely arbitrary and
doesn’t need to point to valid memory.  If *fin* is not `NULL`, it must point
to a finalizer function with the following signature:

```c
void fin (void *ptr);
```

When the new user pointer object is being garbage collected, Emacs calls *fin*
with *ptr* as argument.  The finalizer function may contain arbitrary code, but
it must not interact with Emacs in any way or exit nonlocally.  It should
finish as quickly as possible because delaying garbage collection blocks Emacs
completely.

The `emacs_finalizer` type alias is not defined in `emacs-module.h`; if you
want it you have to define it yourself.

`make_user_ptr` is available since GNU Emacs 25.


#### `get_user_ptr`

```c
void *get_user_ptr (emacs_env *env, emacs_value value);
```

`get_user_ptr` returns the user pointer embedded in the user pointer object
represented by *value*; this is the *ptr* value that you have passed to
`make_user_ptr`.  If *value* doesn’t represent a user pointer object, Emacs
signals an error of type `wrong-type-argument`.

`get_user_ptr` is available since GNU Emacs 25.


#### `set_user_ptr`

```c
void set_user_ptr (emacs_env *env, emacs_value value, void *ptr);
```

`set_user_ptr` changes the user pointer wrapped by *value* to *ptr*.  *value*
must be a user pointer object, otherwise Emacs signals an error of type
`wrong-type-argument`.

`set_user_ptr` is available since GNU Emacs 25.


#### `get_user_finalizer`

```c
emacs_finalizer get_user_finalizer (emacs_env *env, emacs_value value);
```

`get_user_finalizer` returns the user pointer finalizer embedded in the user
pointer object represented by *value*; this is the *fin* value that you have
passed to `make_user_ptr`.  If *value* doesn’t have a custom finalizer, Emacs
returns `NULL`.  If *value* doesn’t represent a user pointer object, Emacs
signals an error of type `wrong-type-argument`.

`get_user_finalizer` is available since GNU Emacs 25.


#### `set_user_ptr`

```c
void set_user_finalizer (emacs_env *env, emacs_value value,
                         emacs_finalizer fin);
```

`set_user_finalizer` changes the user pointer finalizer wrapped by *value* to
*fin*.  *value* must be a user pointer object, otherwise Emacs signals an error
of type `wrong-type-argument`.  *fin* can be `NULL` if *value* doesn’t need
custom finalization.

`set_user_ptr` is available since GNU Emacs 25.


### Quitting

#### `should_quit`

```c
bool should_quit (emacs_env *env);
```

Long-running operations block Emacs and make it unresponsive.  To mitigate
this, you should from time to time check whether the user has requested a quit
by hitting <kbd>C-g</kbd>.  To do this, call the `should_quit` function: it
will return `true` if the user wants to quit.  In that case you should return
to Emacs as soon as possible, potentially aborting long-running operations.
When a quit is pending after return from a module function, Emacs quits without
taking the return value or a possible pending nonlocal exit into account.

If you want to run a synchronous operation that could take a long time,
consider running it in a worker thread and calling `should_quit` in a loop, for
example using this helper function:

```c
#include <assert.h>
#include <errno.h>
#include <stdbool.h>
#include <time.h>

#include <pthread.h>

#include <timespec.h>  /* from Gnulib */

#include <emacs-module.h>

static void
assert_timespec (struct timespec time)
{
  assert (time.tv_sec >= 0);
  assert (time.tv_nsec >= 0);
  assert (time.tv_nsec < TIMESPEC_RESOLUTION);
}

/* Run the given operation and check at regular intervals whether the user
   wants to quit.  If the operation completed successfully, return true.
   If the user wants to quit or an error occurred, return false; the caller
   should then return to Emacs as quickly as possible.  */

static bool
run_with_quit (emacs_env *env, void *(*operation)(void *), void *arg,
               struct timespec interval, void **result)
{
  pthread_t thread;
  int status = pthread_create (&thread, NULL, operation, arg);
  if (status != 0)
    {
      emacs_value status_obj = env->make_integer (env, status);
      emacs_value data
        = env->funcall (env, env->intern (env, "list"), 1, &status_obj);
      env->non_local_exit_signal (env, env->intern (env, "pthread-error"),
                                  data);
      return false;
    }
  while (true)
    {
      /* We have to recalculate the timeout in every iteration to account for
         clock jumps.  */
      struct timespec now;
      gettime (&now);
      assert_timespec (now);
      struct timespec timeout = timespec_add (now, interval);
      assert_timespec (timeout);
      /* pthread_timedjoin_np(3) is only available on GNU/Linux.  See
         https://stackoverflow.com/a/11552244/178761 for a portable
         replacement.  */
      status = pthread_timedjoin_np (thread, result, &timeout);
      if (status == ETIMEDOUT)
        {
          if (env->should_quit (env))
            {
              status = pthread_detach (thread);
              assert (status == 0);
              return false;
            }
        }
      else
        {
          assert (status == 0);
          return true;
        }
    }
}
```

`should_quit` is available since GNU Emacs 26.


## C++ compatibility

Emacs modules can be written in C++.  When including `emacs-module.h`, all
definitions will get C language linkage.  Module functions, initialization
functions, and user pointer finalizers must also have C language linkage, and
must not throw C++ exceptions.  Likewise, module API functions won’t throw C++
exceptions.  If possible, `emacs-module.h` attempts to enforce this requirement
by adding `noexcept` to all function prototypes.  Please be aware that throwing
an exception from within a function declared as `noexcept` calls
`std::terminate` and aborts the process.  Therefore you must catch all C++
exceptions before returning control to Emacs, for example using a [Lippincott
function](https://cppsecrets.blogspot.com/2013/12/using-lippincott-function-for.html):

```cpp
#include <cstddef>
#include <exception>
#include <iostream>
#include <stdexcept>

#include <emacs-module.h>

static void
signal_string (emacs_env& env, const char* symbol, const char* what) noexcept
{
  std::size_t length = std::strlen (what);
  emacs_value data;
  if (length <= PTRDIFF_MAX)
    {
      emacs_value what_object
        = env.make_string (&env, what, static_cast<std::ptrdiff_t>(length));
      data = env.funcall (&env, env.intern (&env, "list"), 1, &what_object);
    }
  else
    data = env.intern (&env, "nil");
  env.non_local_exit_signal (&env, env.intern (&env, symbol), data);
}

/* Must be called only while handling an exception. */
static void
translate_exception (emacs_env& env) noexcept try
{
  throw;
}
catch (const std::overflow_error& exc)
{
  signal_string (env, "overflow-error", exc.what ());
}
catch (const std::underflow_error& exc)
{
  signal_string (env, "underflow-error", exc.what ());
}
catch (const std::range_error& exc)
{
  signal_string (env, "range-error", exc.what ());
}
catch (const std::out_of_range& exc)
{
  signal_string (env, "args-out-of-range", exc.what ());
}
catch (const std::bad_alloc& exc)
{
  signal_string (env, "memory-full", exc.what ());
}
/* If you have more exception types that you’d like to treat specially, add
   handlers for them here. */
catch (const std::exception& exc)
{
  signal_string (env, "error", exc.what ());
}
catch (...)
{
  signal_string (env, "error", "unknown error");
}

extern "C"
{
static emacs_value
my_module_function (emacs_env* env, std::ptrdiff_t nargs, emacs_value* args,
                    void* data) noexcept try
{
  /* Here you can throw C++ exceptions freely. */
  std::cout << "Hello world!" << std::endl;
  throw std::range_error ("something bad happened");
}
catch (...)
{
  translate_exception (*env);
  return NULL;
}
}
```

You can also go the other way round, by throwing C++ exceptions whenever
there’s a nonlocal exit:

```cpp
#include <exception>

#include <emacs-module.h>

class nonlocal_exit : public std::exception { };

template <typename T> T
maybe_throw (emacs_env& env, T value)
{
  if (env.non_local_exit_check (&env) == emacs_funcall_exit_return)
    return value;
  else
    throw nonlocal_exit ();
}
```

Now you can wrap arbitrary environment function calls in `maybe_throw` to have
them throw an exception if a nonlocal exit is pending:

```cpp
std::intmax_t i = maybe_throw (env, env.extract_integer (&env, v));
```

Note that the above Lippincott function continues working without modification:
if a nonlocal exit is pending it won’t be overwritten by the call to
`non_local_exit_sigal` in the `catch` clauses.

Another option is to get rid of the saturating behavior by completely
translating nonlocal exits into C++ exceptions, including the auxiliary data:

```cpp
#include <exception>

#include <emacs-module.h>

class emacs_signal : public std::exception
{
public:
  emacs_signal (emacs_value symbol, emacs_value data) noexcept
    : symbol_(symbol), data_(data) { }

  emacs_value symbol () const noexcept { return symbol_; }
  emacs_value data () const noexcept { return data_; }

private:
  emacs_value symbol_;
  emacs_value data_;
};

class emacs_throw : public std::exception
{
public:
  emacs_throw (emacs_value tag, emacs_value value) noexcept
    : tag_(tag), value_(value) { }

  emacs_value tag () const noexcept { return tag_; }
  emacs_value value () const noexcept { return value_; }

private:
  emacs_value tag_;
  emacs_value value_;
};

template <typename T> T
maybe_throw (emacs_env& env, T value)
{
  emacs_value symbol_or_tag;
  emacs_value data_or_value;
  switch (env.non_local_exit_get (&env, &symbol_or_tag, &data_or_value))
    {
    case emacs_funcall_exit_return:
      return value;
    case emacs_funcall_exit_signal:
      env.non_local_exit_clear (&env);
      throw emacs_signal (symbol_or_tag, data_or_value);
    case emacs_funcall_exit_throw:
      env.non_local_exit_clear (&env);
      throw emacs_throw (symbol_or_tag, data_or_value);
    }
}
```

The calls to `non_local_exit_clear` mean that the saturating behavior is gone,
and environment functions wrapped in `maybe_throw` behave like normal C++
functions.  To translate the new exceptions back into nonlocal exits, you have
to handle them in the Lippincott function:

```cpp
static void
translate_exception (emacs_env& env) noexcept try
{
  throw;
}
catch (const emacs_signal& exc)
{
  env.non_local_exit_signal (&env, exc.symbol (), exc.data ());
}
catch (const emacs_throw& exc)
{
  env.non_local_exit_throw (&env, exc.tag (), exc.value ());
}
/* Other handlers as above. */
```


## Caveats and bugs

### Emacs may jump out of arbitrary code on stack overflow

Emacs installs a signal handler for SIGSEGV that attempts to recover from stack
overflows using `longjmp`.  In modules written in C++, this typically causes
undefined behavior; in other modules it will often cause internal data
structures to become silently corrupted.  Therefore you should disable this
behavior in most cases by resetting the signal handler for SIGSEGV to the
default, which will cause Emacs to terminate on stack overflows.


### When using 32-bit pointers, Emacs may jump out of `non_local_error_get`

There is a bug in the current Emacs codebase that can cause
`non_local_error_get` to execute an uncontrollable `longjmp`.  However, this
code path should only get taken in 32-bit processes, so you can prevent it by
failing compilation if pointers are not 64 bits wide.


### Check the size of `emacs_runtime` and `emacs_env` structures

Modules compiled with some version of `emacs-module.h` can be loaded into Emacs
processes using a different version.  The Emacs module structure types
(`emacs_runtime`, `emacs_env`) are generally binary-compatible (fields never
get removed or reordered; adding new fields is guaranteed to increase the
structure size), but you have to check in your initialization function that the
fields you will access are actually present.  The simplest way to achieve this
is to compare the dynamic size (the value of the `size` field) against the
static size, as explained in the example below.


### No sentinel values for nonlocal exits

Except for `copy_string_contents`, you can’t detect whether a module function
has requested a nonlocal exit by only looking at its return value.
Specifically, a module function with a return type of `emacs_value` may legally
return `NULL` whether or not it has returned normally.  You have to use the
function `non_local_exit_check` or `non_local_exit_get` to determine whether
there’s a pending nonlocal exit.


### `emacs_value` objects are not real pointers

The `emacs_value` type is defined as an alias of a structure without
definition.  However, it’s a completely transparent type, and objects of that
type don’t necessarily point to valid memory.  Furthermore, `NULL` may or may
not represent a valid Lisp object.  Therefore, you must never dereference
`emacs_value` objects or assign any meaning to its values.  You should treat
`emacs_value` as a completely opaque handle type that’s only usable as return
or argument type or module environment functions.


### When a nonlocal exit is pending, module functions silently do nothing

There are lots of ways to represent failure modes in code: using C `errno`
values, C++ exceptions, sum types like Haskell’s `Either`, etc.  Some of them,
like C’s `errno` facility, are stateful: functions don’t return errors
directly, but set a global variable that the caller has to check.  However,
Emacs’s nonlocal exit handling approach is quite different from all other known
approaches: nonlocal exits are represented as per-environment state, but
environment functions exhibit *saturating behavior*: once a nonlocal exit is
pending for an environment, all environment functions called for that
environment silently ignore all their arguments (except the environment pointer
itself) and return an unspecified value.  You have to understand the
consequences of this behavior and use the nonlocal exit handling functions
appropriately.


### Strings passed to `make_string` must be null-terminated

Even though you have to pass the length of the string explicitly to
`make_string`, the string must still be null-terminated.  This is unlike all
other C APIs, which either take a null-terminated string or a pointer and a
length.


### Module functions may not modify the contents of the *args* array

Module functions receive their arguments in the *args* parameter.  That
parameter is defined as `emacs_value *args`.  Even though it’s not defined as a
pointer to `const`, module functions must not modify the array.


## Example

This is an example that shows how to work around some of the caveats described
above.  It refuses to compile if not in 64-bit mode, checks the sizes of the
runtime and environment structures, and resets the signal handler for
`SIGSEGV`.

```c
#include <assert.h>
#include <limits.h>
#include <signal.h>
#include <stdalign.h>
#include <stddef.h>
#include <stdint.h>

#include <verify.h>  /* from Gnulib */

#include <emacs-module.h>

/* The next three static assertions check that pointers are 64 bits and
   properly aligned.  This avoids a bug that can cause non_local_exit_get to
   exit nonlocally by failing compilation if the bug is possible. */
verify (CHAR_BIT == 8);
verify (sizeof (emacs_value) == 8);
verify (alignof (emacs_value) == 8);

/* The actual initialization function.  It’s called in a safe regime where all
   members of env are accessible and nonlocal exits are no longer possible. */
static void initialize_module (emacs_env *env);

extern int
emacs_module_init (struct emacs_runtime *ert)
{
  /* Fail if Emacs is too old. */
  assert (ert->size > 0);
  if ((size_t) ert->size < sizeof *ert)
    return 1;
  emacs_env *env = ert->get_environment(ert);
  assert (env->size > 0);
  if ((size_t) env->size < sizeof *env)
    return 2;
  /* Prevent Emacs’s dangerous stack overflow recovery. */
  if (signal (SIGSEGV, SIG_DFL) == SIG_ERR)
    return 3;
  /* From this point on we are reasonably safe and can call the actual
     initialization routine. */
  initialize_module (env);
  /* initialize_module can still use env->non_local_exit_signal to signal
     errors during initialization.  These will cause Emacs to signal even if we
     return 0 here. */
  return 0;
}
```


## Module assertions

You can pass a command-line option `-module-assertions` (or
`--module-assertions`) to the Emacs binary.  If you supply this option, Emacs
will perform some additional checks to find violations of the requirements
described in this document.  A violation (that would otherwise lead to
undefined behavior) causes Emacs to print an error message and then abort the
process.  You should use this command-line option while you are developing or
debugging a module; it can detect misuses of the module API that would
otherwise be hard to detect manually.  Module assertions can’t detect all
specification violations related to modules, so not triggering module
assertions is not a proof that a module is bug-free.  Module assertions slow
down the interaction between Emacs and modules significantly; therefore you
shouldn’t enable them in production.

Module assertions are a new type of assertions only for modules; they are
different from Lisp assertions (the Lisp `cl-assert` macro), C assertions (the
C `assert` macro), and Emacs source-level assertions (the C `eassert` and
`eassume` macros).


## History

The module API as presented here was designed by Daniel Colascione and
primarily implemented by Aurélien Aptel.  You can find the [original design
document](https://lists.gnu.org/archive/html/emacs-devel/2015-02/msg00960.html)
with some rationales in the `emacs-devel` archives.  The current implementation
differs from the original design in several ways:

-   In the original design, environments are thread-local so that calling an
    environment function from a different thread than the one owning the
    environment would be undefined behavior.  In the current implementation,
    you can call environment function from any thread as long as it’s the
    current Emacs Lisp thread; the thread in which the environment was created
    doesn’t matter.

-   In the original design, calling most environment functions is undefined
    behavior if a nonlocal exit is pending.  In the current implementation,
    nonlocal exits saturate: environment functions will ignore their arguments
    and do nothing if a nonlocal exit is pending.

-   The original design treats `NULL` as sentinel value for a nonlocal exit.
    The current implementation has no sentinel values.

-   Instead of a Boolean value to represent nonlocal exits, the current
    implementation uses a ternary enumeration to deal with `throw` jumps.  This
    unfortunately makes checking for nonlocal exits more verbose, but has the
    advantage that no special marker object for `throw` is required.

-   `make_function` allows specifying a documentation string and an additional
    data argument.

-   Instead of `int64_t`, the current implementation uses `intmax_t`.  In
    theory this reduces ABI compatibility because `intmax_t` is supposed to be
    a variable-width integer type; in practice [ABI compatibility appears to be
    important enough to keep it
    fixed](https://gcc.gnu.org/bugzilla/show_bug.cgi?id=49595#c2).

-   Instead of `size_t`, the current implementation uses `ptrdiff_t`.  This has
    advantages and disadvantages: on the one hand, `size_t` is more common in
    existing C APIs so that additional type casts and range checks are
    required; on the other hand, unsigned integer types have come to be seen
    more negatively in recent years due to their behavior on overflow.

-   `copy_string_contents` signals an error if provided with a too short buffer
    in the current implementation.

-   Some functions have slightly different names.

-   The original design doesn’t contain the vector access functions or
    `should_quit`.

<!-- Local Variables: -->
<!-- mode: gfm -->
<!-- ispell-local-dictionary: "en_US" -->
<!-- fill-column: 79 -->
<!-- End: -->
