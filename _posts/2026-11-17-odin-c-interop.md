# Odin C inter-op

A couple of weeks ago I learned about [learning by teaching](https://en.wikipedia.org/wiki/Learning_by_teaching). And I thought would like to put it in practice via this blog. This is the reason why today I will be writing about C inter-op in Odin.

Inter-op between languages is a topic that always interested me and I will try to learn (teach) how it works in Odin.

## What is Odin?

[Odin](https://odin-lang.org/) is a general-purpose programming language, highly opinionated, with main drivers around simplicity, it was heavily inspired by languages like Pascal, Ca and Go (you can tell). The compiler was built on top of [llvm](https://llvm.org/) and doesn't have a garbage collector. It supports many [calling conventions](https://odin-lang.org/docs/overview/#calling-conventions), giving to the language versatility on top of simplicity (this are two great characteristics to combine in a language).

In my own words, calling convention is how we need to manipulate the stack when calling a function, in newer CPU's, the calling conventions relies less on the stack and more on the CPU registries for performance reasons.

Calling conventions defines who is responsible for clearing the stack, basically reduced to the caller or the callee. They defined the boundary between two languages / programs. Odin give us ways to support the most common calling conventions.

Odin has their own calling convention that gives more ergonomic to manual memory handling, is like a `C` ABI + a pointer at the end to the `context`. For now, let's think that the `context` is a pointer to a block of memory that functions will use for allocating memory.

To recap, when we are doing inter-op between languages we need to pay attention to:

- who is the owner / controlling the memory,
- how memory is managed (manually, GC, etc) and
- if the caller knows how the memory looks like for the callee.

## Data types

Odin supports almost [every data type](https://odin-lang.org/docs/overview/#basic-types) you can think of, including maps, dynamic arrays, ring-buffers, etc. making the language data-oriented, not only that, but it has a whole package for supporting [C types](https://odin-lang.org/docs/overview/#cstring-type) explicitly and convenient functions to cast from one type to the other. Again a pragmatic approach to inter-op with foreign languages.

## Show me the code

Alright, this is how you can create a little hello world program in odin that will output `hello world`:

```odin
package main

import "core:fmt"

main :: proc() {
	fmt.println("hello world")
}
```

Store this file as `main.odin` in the root of an empty folder (maybe call `odin-c-inter-op`). For running the program you would need to run `odin run main.odin -file`.


### First: The library

Let's create a simple C library that will be called by our program later on, we can call it `greetings.c` and store it next to our `main.odin`

```C
#include <stdlib.h>

int pch_str_len(const char *name)
{
	int size = 0;
	while(*name++) size++;
	return size;
}

const char *say_my_name(const char *name)
{
	char *buf = NULL;
	int len = -1;
	if (!name) name = "stranger";

	// We add to the name "Hi ", 3 characters.
	len = 3 + pch_str_len(name);

	if (len == 2) return NULL;

	buf = malloc((size_t)len+1);
	if (!buf) return NULL;

	snprintf(buf, (size_t)len+1, "Hi %s", name);
	return buf;
}

void free_greeting(const char *greeting)
{
	free((void*)greeting);
}

```

This program exposes two different functions `say_my_name` and `free_greeting`, the `free_greeting` could be replaced by the odin's
integration to c functions, but, if we take care on the callee side we can link with a different version of `stdlib` than odin and prevent odd issues, or even better, we can have our own allocator or block of memory in our library this way.

Our library could be dynamically or statically linked, for this example I will use statically linked, I usually prefer this approach when possible, but sometimes is unavoidable to use shared libs.

To build an static library in any `*nix` operating system you could simply run:

```sh
# Compile greetings.o
gcc -c greetings.c

# Link libgreetings.a statically
ar rcs libgreetings.a greetings.o
```

And on windows platform you would need to open a `Developer Command Prompt`, be sure you open it for x64, because odin expect libraries to be compiled (and linked) for x64, also in windows is using RAD linker, which also expect an x64 binary:

```bat
:: Compile grretings.obj
cl -TC -c greetings.c

:: Link greetings.lib statically
lib -nologo greetings.obj -out:greetings.lib
```

From here we can work on using our library in Odin.

### Second: Call our `greetings` library

In Odin to call an external library we need to tell the program how our external functions looks like (Odin doesn't provide any automatic code generation based on .h file, and I think is fine, always simplicity is at its core, but that wouldn't be too hard to implement if needed) and that those external functions are, well, external (foreign). The language gives you the constructs to make the integration fairly easy.

Our library could be dynamically or statically linked, for this example I will use statically linked, I usually prefer this approach when possible, but sometimes is unavoidable to use shared libs.

For using our `greetings` library in odin we need to follow two steps, first we need to do a `foreign import <foreing name> <library filename>` to import the dynamic / static compiled library and secondly we need to tell Odin what functions signatures we are getting from this libraries with the keyboard `foreign`.

For our case it would look like this:

```odin
package main

import "core:fmt"
import "core:strings"


when ODIN_OS == .Windows {
    foreign import greeting_lib "greetings.lib"
} else {
    foreign import greeting_lib "libgreetings.a"
}

@(default_calling_convention = "c")
foreign greeting_lib {
	say_my_name :: proc(name: cstring) -> cstring ---
	free_greeting :: proc(greeting: cstring) ---
}

main :: proc() {
	c_msg := say_my_name("Ponchosberg")
	if c_msg == nil {
		fmt.println("Allocation failed")
		return
	}

	defer free_greeting(c_msg)
	msg := string(c_msg)
	fmt.println(msg)

	defer free_all(context.temp_allocator)
}
```

The `default_calling_convention = c` if basically telling how the `foreign` functions calling should be treated.

The function declaration with a body of `---` it means that the body will be provided by the foreign library (just like C function declaration in header files):
```odin
	say_my_name :: proc(name: cstring) -> cstring ---
```

Once we have our static library compiled we can run the program again by running `odin run main.odin -file` (nothing changed from running the `hello world!`) and it should return:

```
Hi Ponchosberg
```

We successfully called our C library from Odin now :).

I strongly recommend to read this simple [odin official binding to c documentation](https://odin-lang.org/news/binding-to-c/) explains the binding mechanisms and considerations for translating from the types on Odin to C's types.

Finally, this was an exercise for me, to get more familiar to this functionality and in the future I might explore how to do this in other languages (I'm thinking on explaining how this mechanism works in [kotlin-native](https://kotlinlang.org/docs/native-get-started.html), I'm building an [advent of code template](https://github.com/ponchoalv/kaoc) to work with kotlin).

You can get the code we explore in this post [here](https://github.com/ponchoalv/simple-odin-foreign), I also included a `Makefile` for `*nix` and a `build.bat` for windows systems.
