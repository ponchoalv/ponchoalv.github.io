# Odin C inter-op

A couple of weeks ago I learned that [learning by teaching](https://en.wikipedia.org/wiki/Learning_by_teaching) a thing. I will try to put it in practice via this blog, and this is the reason why I will talk about C inter-op in Odin.

Inter-op between languages is a topic that always interested me and I will try to learn (teach) how it works in different languages. 

## What is Odin?

[Odin](https://odin-lang.org/) is a general-purpose programming language highly opinionated with core values / drivers around simplicity, it was heavily inspired by languages like Pascal, C, Go, etc. The compiler was built on top of [llvm](https://llvm.org/) and doesn't have a garbage collectors. It supports many [calling conventions](https://odin-lang.org/docs/overview/#calling-conventions), this give us a hint that Odin was thought and built with a pragmatic approach.

My understanding is that a calling convention is how the stack is going to be manipulated when calling a function, there are many calling conventions, also, in newer CPU's the calling conventions relies less on the stack and more on the CPU registries, due to performance. Also, calling conventions defines who is responsible for clearing the stack,
the caller or the callee. This is basically the boundary between two languages / programs, and having control over this makes odin a good alternative to inter-op with other languages.

On top of existing calling conventions Odin has their own calling convention that gives more ergonomic to manual memory handling, which basically is the universal `C` ABI + a pointer at the end to the `context`. For now, let's think the `context` as a pointer to a block of memory that functions will use for allocating memory (it is more than that, but this is enough to understand why is useful to have it around.

To recap, things we need to understand when we are doing inter-op between languages are, who is the owner / controlling the memory, how memory is managed (manually, GC, etc) and if the caller knows how the memory looks like for the callee.

## Data types

Odin supports almost [every data type](https://odin-lang.org/docs/overview/#basic-types) you can think of, including maps, dynamic arrays, ring-buffers, etc. makes the language data-oriented, not only that, but it has a whole package for supporting [C types](https://odin-lang.org/docs/overview/#cstring-type) explicitly and convenient functions to cast from one type to the other.

## Show me the code

Alright, this is how you can create a little hello world program in odin that will output `hello world`:

```odin
package main

import "core:fmt"

main :: proc() {
	fmt.println("hello world")
}
```

Store this file as `main.odin` in the root of an empty folder (maybe call `odin-c-inter-op`)

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

If we run our program like this `odin run main.odin -file` it should return:

```
Hi Ponchosberg
```

We successfully called our C library from Odin now :).

I strongly recommend to read this simple [odin official binding to c documentation](https://odin-lang.org/news/binding-to-c/) explains the binding mechanisms and considerations for translating from the types on Odin to C's types.

Finally, this was an exercise for me, to get more familiar to this functionality and in the future I might explore how to do this in other languages (thinking on working with Kotlin native).

You can look at this code and download it from y [Github repository](https://github.com/ponchoalv/simple-odin-foreign), I also included a `Makefile` for `*nix` and a `build.bat` for windows systems.