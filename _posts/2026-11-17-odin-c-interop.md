# Odin C inter-op

A couple of weeks ago I learned that [learning by teaching](https://en.wikipedia.org/wiki/Learning_by_teaching) a thing. I will try to put it in practice via this blog, and this is the reason why I will talk about C inter-op in Odin.

Inter-op between languages is a topic that always interested me and I will try to learn (teach) how it works in different languages. 

## What is Odin?

[Odin](https://odin-lang.org/) is a general-purpose programming language highly opinionated with core values / drivers around simplicity, it was heavily inspired by languages like Pascal, C, Go, etc. The compiler was built on top of [llvm](https://llvm.org/) and doesn't have a garbage collectors. It supports many [calling conventions](https://odin-lang.org/docs/overview/#calling-conventions), this give us a hint that Odin was thought and built with a pragmatic approach.

My understanding is that a calling convention is how the stack is going to be manipulated when calling a function, there are many calling conventions, also, in newer CPU's the calling conventions relies less on the stack and more on the CPU registries, due to performance. Also, calling conventions defines who is responsible for clearing the stack,
the caller or the callee. This is basically the boundary between two languages / programs, and having control over this makes odin a good alternative to inter-op with other languages.

On top of existing calling conventions Odin has their own calling convention that gives more ergonomic to manual memory handling, which basically is the universal `C` ABI + a pointer at the end to the `context`. For now, let's think the `context` as a pointer to a block of memory that functions will use for allocating memory (it is more than that, but this is enough to understand why is useful to have it around.

To recap, things we need to understand when we are doing inter-op between languages are, who is the owner / controlling the memory, how memory is managed (manually, gc, etc) and if the caller knows how the memory looks like for the callee.

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

Let's create a simple C library that will be called by our program later on.

```C
#include <stdlib.h>

// name should always be a valid null terminated c_string
// we do not support buffers (we don't take the length as argument)
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

// We should expose a way to call free from odin (I think)
void free_greeting(const char *greeting)
{
	free((void*)greeting);
}

```

