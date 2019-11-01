---
layout: post
title: "A thread safe ISAAC-rand"
date: 2018-03-02
categories:
---

# A thread safe ISAAC-rand
ISAAC-rand is not thread safe because it uses a global context to keep track of the random state, meaning that when the context is seeded in a threaded environment, its state is likely to be clobbered by another thread.  A lockless solution to this issue can be achieved with a few relatively simple changes.

Looking into ISAAC's source code (`ISAAC-rand.c`), we notice that the random context is actually scoped quite locally, shared only between `seed_random` and `random_num`:

```c
randctx R;

void seed_random(char* term, int length)
{
    memset(R.randrsl, 0, sizeof(R.randrsl));
    strncpy((char *)(R.randrsl), term, length);
    randinit(&R, TRUE);
}

short random_num(short max)
{
    return rand(&R) % max;
}
```

Studying the above snippet, we can simply create a local version of `R` in `seed_random`, and modify the function to return the indicated context.  In this way, any context can be maintained unharmed, and localized to its own function.

To facilitate this change, the `random_num` function should also be modified to take a local version of the context.  The most straightforward way of accomplishing this is to change the function's signature to take the random context in form of an argument;

```c
randctx seed_random(char* term, int length)
{
    randctx R;
    memset(R.randrsl, 0, sizeof(R.randrsl));
    strncpy((char *)(R.randrsl), term, length);
    randinit(&R, TRUE);
    return R;
}

short random_num(randctx *R, short max)
{
    return rand(R) % max;
}
```

Note that this changes the program's original design; it is now required that the context be initialized with a seed before returning random numbers.  To get around this, a solution may be to create a basic function, perhaps named `init_random`, that returns a `randctx`.

Additionally, we of course have to change the way that the random number generator is used.  Below are two implementations: before (top), and after (bottom).

```c
// Before.
void example_function(char* term)
{
    ...
    seed_random(term, WORDLEN);
    short rand = random_num(SIGNATURE_LEN);
    ...
}

// After.
typedef struct randctx randctx;

...
void example_function(char* term)
{
    ...
    randctx R = seed_random(term, WORDLEN);
    short rand = random_num(&R, SIGNATURE_LEN);
    ...
}
```

Depending on how you've setup your code, you'll need to add a type definition for `randctx` so that the compiler will recognize it, as seen above.
