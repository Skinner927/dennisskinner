---
title: "Writing Good C APIs"
date: 2022-08-07T19:25:19-04:00
#summary: "\"tips & tricks\" for designing strong APIs in cross-platform contexts."
tags: [C, "Software Design"]
draft: true
---

In modern software development we can run CI/CD pipelines to automatically
build projects, scan their binaries for defects, report formatting errors,
warn on usage of potentially dangerous functions, spellchecking, and an
unlimited number of other automated tools to programmatically "yell" at the
programmer to do better. No matter how good your code looks on the inside,
what makes and breaks a library, or even a common function, is its API.

Nobody is an expert in this subject and every project is going to be different.
What I hope to provide are some guidelines and ideas to think about when
designing an API in C. These "tips & tricks" are heavily influenced by my own
personal work designing APIs which work across different operating systems and
architectures.

Don't care what I have to say and just want to see an example? Check out
[SQLite's C API](https://www.sqlite.org/capi3ref.html). It is a great example
of what you should do when creating an API.

## Be consistent

Before I even start, the most important guideline is to always
be consistent. Whatever you choose to do, in whatever way you
choose to do it, ensure you always do it that way. If you're going to choose
to use UTF-32 encoding, all your functions have to accept UTF-32 encoding.
If you use `int` for boolean values in one place, don't start using `bool`
in other places. If you're going to use the terms "create"/"destroy" to
create and destroy contexts, don't later start using "new"/"finish".

## Namespace

When writing C, everything is in the global namespace. If you name a function
`write_to_file()` you're going to get burned by someone else who naively did
the same thing. It may not happen today, it may not happen tomorrow, but when
it does, it's going to take you a very long time to track down what's
happening.

Namespaces start with the project name as a prefix followed by ever increasing
specificity. Usually this looks like module or group, followed by a noun,
followed by a verb, but there are no hard-fast rules.

Let's begin with an example scenario. We're working on a project called
"Great Blue Dog" (who knows what it does). We can use the prefix `gbd_` or
`dog_` or anything else we want -- just be consistent and try to keep it
somewhat unique.

GBD has a function to burry bones. The common naming of this
function would be `gbd_burry_bone()`, but this lends to problems down the line.
If we keep with this "project_verb_noun" pattern our API immediately becomes
difficult to parse. Imagine the following functions exist:

- `gbd_burry_bone()`
- `gbd_chase_cat()`
- `gbd_create_bone()`
- `gbd_find_bone()`
- `gbd_find_cat()`

We can see that just by looking at our API, the `cat` and `bone` functions
are becoming intertwined. It's difficult to find out what functions belong
to `bone` and which to `cat`. This toy example becomes an even bigger problem
when your actual project contains tens to hundreds of functions.

### Namespace in reverse

English has an interesting property (or defect?) in that verbs and adjectives
come before nouns. Don't let this quirk stop you from good namespaces. If we
instead, as outlined above, forget how English works and instead focus on how
our programs are structured, we can make better namespaces. Here are the
same functions reworked with module prefixes.

- `gbd_bone_burry()`
- `gbd_bone_create()`
- `gbd_bone_find()`
- `gbd_cat_chase()`
- `gbd_cat_find()`

With the refactor it's now very easy to see what I can do with a `bone` and
what can be done with a `cat`. This makes using your API much more pleasant
to consumers who are not (and who likely don't care to become) intimately
familiar with your library.

### Namespace everything

Namespaces are not just for functions. Types, constants, enums, even
file names should be properly namespaced. Here's a common pattern I like to
do when creating a `typedef`'d struct:

```c
typedef struct gbd_bone_s {
  // ...
} gbd_bone_t;
```

Say `gbd_bone_create` had a `bone_style` parameter. Use an `enum` and
give it a proper namespace.

```c
typedef enum gbd_bone_style_e {
  GBD_BONE_STYLE_RAWHIDE,
  GBD_BONE_STYLE_RUBBER,
  GBD_BONE_STYLE_BISCUIT
} gbd_bone_style_t
```

(Can you tell I don't have a dog?)

## Use constants

Magic numbers and magic strings are difficult to track down and lead to
errors. If you have a function that expects certain values, define those
values so they can be referenced by name. Not only does this make it easier
for consumers of your library, but it also allows you to later refactor
what an option might be.

A perfectly sane example is the POSIX `open` function:

```c
int open(const char *pathname, int flags);
```

It takes flags such as `O_RDONLY`, `O_WRONLY`, or `O_RDWR` (and others). By
having defined names for each of these flags, not only does it make it easier
for the end-user to resolve what flags to use, it makes their code far more
clear than simply passing in a literal `1` or `2`. These flags are defined
via `#define` preprocessor directive. This is a common pattern when your
flags need to support bitwise operations.

If your flags don't require bitwise combinations, prefer to use an `enum`.
This way you can specify a type for your parameters and it's even easier for
end-users to resolve allowed values.

While `enum` is preferred because it makes it slightly easier to track down
possible values, be aware of your target systems as some older compilers do
not support `enum`.

## Pick a single character encoding

When building APIs that exist across different platforms, be aware that the
default text/character encoding may be different on each platform. Because
of this, strictly declare what character encoding you are going to support.
This is important to do even if only working on a single platform as the
project may potentially grow and having these things defined early will
automatically answer questions in the future.

The most prevalent example in my line of work is text encoding between Linux
and Windows. Windows native Win32 API was built before Unicode was even a
standard and was eventually improved to support UTF-16 via alternative
functions (read: with a crowbar and a sledgehammer). UTF-16 on Windows is
stored as `wchar_t` and regular ASCII is stored as `char`. Linux mostly
supports UTF-8 and does this with the `char` type. Because ASCII is a subset of
UTF-8, this means Linux can support both ASCII and UTF-8 with the single `char`
type. Additionally, the web primarily uses UTF-8 for communication mainly
because ASCII can ride along without issue.

I suggest your external facing APIs always use UTF-8, even on Windows unless
you're writing a Windows-only library and in that case only use `wchar_t`.
Even English uses characters outside of ASCII as Emojis become more engrained
into the lexicon. I am not the only person to believe in this UTF-8 pattern.
The [UTF-8 Everywhere](http://utf8everywhere.org/) group also agrees and I
would highly suggest everyone to read that site from top-to-bottom.

In general, the pattern is to treat all `char` as UTF-8 and on platforms where
passing `char` into functions will not be treated as UTF-8 (Windows) then you
call your conversion function just before the native function and convert any
return values back to `char` immediately.

If this is confusing, and even if it is not, I implore you to read the
[UTF-8 Everywhere](http://utf8everywhere.org/) page and become familiar
with how different types of UTF encodings work.

## Return values

Use a single explicit pattern for conveying success and failure in your
library. This correlates with ["Use&nbsp;constants"](#use-constants) above.

Have you ever called a function similar to the one below?

```c
int mylib_do_something_and_return_status(void);
```

I'm sure you have, and every time it's a question of does it return truthy or
falsey values on success?

This confusion is logical isn't it? Applications return `0` on successful
run which is `false`, but in code we would expect it to return `true` so we can
use it like below:

```c
if (!mylib_do_something_and_return_status()) {
  log("mylib_do_something_and_return_status failed");
  return;
}
```

After all, `0` and `NULL` usually represent failure in other contexts:

```c
char* user_name = get_user_name(ctx);
if (!user_name) {
  log("Failed to get user name");
  return;
}
```

What's worse is when you encounter code like below, and you're not sure which
path is on success, which forces you to look up the docs for that function just
to follow program flow.

```c
if (do_something()) {
  perform_a();
} else {
  perform_b();
}
```

The solution to the problem adds verbosity, but also adds clarity. Create
an enum (or `#define` if you can't use `enum`) with success and errors. The
most important concept is to only define a single "success" value and
everything else are error codes. This way calls to your function can catch
all errors by simply ensuring the returned value is not "success".

```c
typedef enum {
  MYLIB_STATUS_SUCCESS,
  MYLIB_STATUS_ERROR, // non-specific error
  MYLIB_STATUS_ERROR_FILE_READ, // specific error
  MYLIB_STATUS_ERROR_OUT_OF_MEMORY,
  // ...
} mylib_status_t;
```

Then you can define your functions with `mylib_status_t` as their return type.

```c
mylib_status_t mylib_do_something_and_return_status(void);
```

Usage would then be very clear:

```c
if (MYLIB_STATUS_SUCCESS != mylib_do_something_and_return_status()) {
  log("mylib_do_something_and_return_status failed");
  return;
}
```

## Prefer fixed-width integer types

In general, prefer to use
[fixed-width integers](https://en.cppreference.com/w/c/types/integer)
(`int8_t`, `uint16_t`, `int32_t`, etc.) over
[min-width integers](https://en.cppreference.com/w/c/language/arithmetic_types)
(`int`, `long`, `long long`, etc.) and implementation defined integers like
`size_t`, because it makes operations more clear for everyone involved and
provides better interoperability. There is no ambiguity because the widths are
strict and not variable depending on what system is running your code.

This idea can get sticky fast, and there are times where it improves usage to
use regular (min-width) integers or implementation defined such as `size_t`.
These integers are also more portable as fixed-width are optional for C99+
compliance. However, the very large majority of modern compilers support
fixed-width integers. This rule does grow stronger (and in my opinion a
requirement) when dealing with sharing data between different systems and
different languages.

Here's an example where fixed-width can be helpful. Suppose you're writing a
tool in C which listens to instructions over an HTTP API. You're also writing a
complementing client library in Python to appease the unwashed masses `:P`.
Python has no upper (or lower) limit to the size of an integer, but you know
that so you're adding validation to your API calls on the client side (and of
course you're also validating on the API side, but it's nice to alert clients
before hitting the server). Your C function takes an `int` and you tested on
your machine that `printf("%zu\n", sizeof(int));` returned `4` so the max value
is `2,147,483,647` and you double-confirmed this with `INT_MAX`. However, the
C99 specification says an `int`'s minimum width is 16-bytes. This makes that
maximum value `32,767`! Wait a second, isn't that the max value of a `short`?
It is, both `short` and `int` need only be 16-bytes per C99 standard.

## Use inputs that complement outputs

POSIX `write()` is my favorite example of inputs not complementing outputs.
Here's the declaration of `write()`:

```c
ssize_t write(int fd, const void *buf, size_t count);
```

`count` is the number of bytes to write. It is of size `size_t`. The return
value represents the number of bytes written as `ssize_t` which is a signed
version of `size_t`. If `size_t` is the largest integer store then `ssize_t`
would only be able to store a number half the size of `size_t` due to
[two's complement](https://en.wikipedia.org/wiki/Two%27s_complement). This
means you can pass a value into `count` that cannot be returned by `ssize_t`!

Of course, there's additional confusion because `size_t` and `ssize_t` are both
implementation defined so we can't know across every platform what those
min/max can actually be (and it would not be illegal for `size_t` to be 128-bit
and `ssize_t` to be 64-bit which would just add confusion).

To help you sleep at night, the glibc manpage does define what happens when you
pass in a value to `count` which is bigger than `ssize_t`, but the POSIX
definition does not. Either way, if the API used `uint32_t` and `int64_t` (or
even `size_t`/`size_t` with error being `SIZE_MAX`) it would be clear without
having to reference the docs how the function operated.

The objective here is, don't force your end-users to consult the documentation,
try to build APIs that make sense.

## Parameter soup

When you have functions that take more than a few parameters, you can end up
with confusing function calls which are even harder to read without
consulting the documentation.

For example, imagine the function:

```c
int factory_car_new(
  const char* car_name,
  bool has_automatic_transmission,
  uint8_t number_of_axels,
  uint8_t number_of_doors,
  uint8_t number_of_cylinders,
  uint8_t displacement_in_liters,
  uint32_t number_of_cupholders,
  bool has_navigation,
  bool has_apps,
  bool has_heated_front_seats,
  bool has_heated_rear_seats,
);
```

And an example usage may be:

```c
factory_car_new("Freedom", true, 3, 5, 8, 6, 35, false, false, true, false);
```

Maybe I'm wrong, but that sure seems unclear which values you're setting
without referencing the docs or declaration.

An alternative is to create a struct with all the possible parameters. What
is also nice about using a struct is it allows for some of the parameters to
be optional.

```c
typedef struct factory_car_new_args_s {
  bool has_automatic_transmission;
  uint8_t number_of_axels;
  uint8_t number_of_doors;
  uint8_t number_of_cylinders;
  uint8_t displacement_in_liters;
  uint32_t number_of_cupholders;
  bool has_navigation;
  bool has_apps;
  bool has_heated_front_seats;
  bool has_heated_rear_seats;
} factory_car_new_args_t;

int factory_car_new(const char* car_name, factory_car_new_args_t args);
```

And then your improved usage could be:

```c
factory_car_new("Freedom", (factory_car_new_args_t){
    .has_automatic_transmission = true,
    .number_of_axels = 3,
    .number_of_doors = 5,
    .number_of_cylinders = 8,
    .displacement_in_liters = 6,
    .number_of_cupholders = 35,
    .has_navigation = false,
    .has_apps = false,
    .has_heated_front_seats = true,
    .has_heated_rear_seats = false,
});
```

There's more than one way to skin this cat, and it's completely up to the
specific problem you're solving. In general, design your struct to expect all
members to default to `0`/`NULL` because:

- `malloc`'d structs should be initialized with `memset(x, 0, y)`.
- `calloc` is a valid way to allocate structs.
- Structs are often zero initialized with `= {0}`.
- Struct initialization with named members does not require all members to
  be listed (this is good for backwards compatability) and will default
  non-designated members to `0`. This allows for partial initialization:
  ```c
  factory_car_new("Freedom", (factory_car_new_args_t){
      .has_automatic_transmission = true,
      // completely valid where all other members will be 0
  });
  ```

## Globals are (almost always) evil

If you're not brand new to programming you've probably heard this one before,
but has anyone ever explained why not to use globals and what to use instead?

Globals are bad because everyone can read and write to them all at once.
Globals can also become troublesome because your linker could trip up on
incorrect export/import statements and then nobody can write to them (looking
at you `msvc`).

Globals severely limit your library to expand into multithreading and nearly
impossible if you're venturing into multiprocessing/distributed/microservices.
Let me provide some examples.

There are of course instances where globals are perfectly fine, but in general,
they should be used very sparingly and passing values around is preferred.

An example program generates avatar images with geometric shapes. When you
start the program, a command line argument can be passed to set the default
background color of the avatar image. This value is stored in a global variable
and the `avatar_image_generate()` function reads this global to determine the
background color.

Some time passes and your library adds a new feature: users that start with
the letter `G` will always have a green background (bare with me).

The `avatar_image_generate()` function does not take any arguments so to
implement the feature, you quickly swap the global background color to green
and then change it back when it returns. This works fine.

Your little program is getting pretty popular. You need to implement a
work queue so multiple threads can work on generating images. Part of the
feature is that each worker will now have its own default background color.
You're doing the global switch like you did for the green backgrounds, but
some workers are coming out with the wrong color. So you add a lock so that
when one worker has changed the color, other workers don't accidentally try
to generate images, but now your performance is lower than a single thread.
What can we do?

The clear answer is to give `avatar_image_generate()` a color argument, but
how do we store all the colors for the different workers? What about multiple
parameters? This can all be solved with a context.

### Contexts

A context is whatever you'd like, but often it's a struct that is passed around
and it stores things which would normally be thought of as global
configuration. There's nothing fancy about it, but it allows you to have
multiple different configurations at one time. In the example, each worker
thread would have its own context it could pass to each `avatar_*` function.

Contexts can be defined, opaque, or a mix of both. What follows is an example
of a mix of both.

```c
// avatar.h
typedef struct avatar_ctx_private_s avatar_ctx_private_t;

typedef struct avatar_ctx_s {
  char* background_color;
  avatar_ctx_private_t* private;
} avatar_ctx_t;

int avatar_ctx_create(avatar_ctx_t ** inout_ctx);
void avatar_ctx_destroy(avatar_ctx_t ** inout_ctx);
```

```c
// avatar-private.h
// A header file only the library can see and is not exported to consumers

// Here we can declare what struct avatar_ctx_private_s looks like
struct avatar_ctx_private_s {
  char* secret_key;
  // etc.
};
```

Contexts can store anything you'd like to pass around, and it's a perfectly
normal pattern. Context can store pointers to functions or even to other
contexts! Prefer passing around what you would normally consider a global
rather than actually using globals.

