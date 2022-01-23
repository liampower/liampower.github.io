---
title: "Safer C programming on Windows in 2020"
date: 2020-08-13T21:37:05+01:00
draft: true
summary: Quick guide on how to write safer C and C++ on Windows in 2020
---

I love programming in C and C++ &mdash; even with the rise of Rust, it's still hard to
find a language that offers the sheer power and control of C. Recently, the software
industry has started to realise an uncomfortable truth: C and C++ programs often
contain a *shocking* number of (often severe) security vulnerabilities due to unsafe
memory practices. Microsoft and Google have both come out with data that shows up to
70% of CVEs can be attributed to memory bugs. What is going on? Are C/++ programmers
so incompetent they just create bugs left, right and centre? No. 

So how do we fix this? Clearly we cannot going this way; software is only becoming
*more* entwined with our lives, not less. Do *you* want someone to root your phone
with a malformed JPEG? One solution is to abandon memory-unsafe languages entirely,
an approach that has been enthuasiastically pushed by a group of people perogatively
termed "the Rust task force". Perhaps in fifty years all software will be written
in memory safe languages, switching cold-turkey to Rust today simply is not practical
for the vast majority of software. 


## UBSan
This is one of Clang's runtime bug detection tools. You link your program with the
UBSan framework and it will report at runtime things like integer overflow, division
by zero and null pointer dereferences. For example, take the following code:

```c
// test.c
#include <stdlib.h>

int main(int argc, const char** const argv)
{
    int* ptr = NULL;
    *ptr = 0xABCD; // Undefined behaviour here
    
    return EXIT_SUCCESS;
}
```

If we compile our program using the clang UBSan library, like so...
```bash
$ clang --fsanitize=undefined test.c -o test.exe
```
...and then run the program, we receieve the following message:

```bash
test.c:6:5: runtime error: store to null pointer of type 'int'
SUMMARY: UndefinedBehaviorSanitizer: undefined-behavior test.c:6:5 in
```
Voila! A previously unhandled null pointer dereference is caught and reported.
UBSan can even report the exact source location the fault occurred at.

## Clang-tidy
Clang-tidy is another tool from the LLVM project. Clang-tidy is a static analyser
that can apply a wide range of checks to your code before it runs. Especially
impressive, this tool can detect null dereferences, uninitialised memory and
heap memory leaks before your program runs. Consider the following program:

```c
#include <stdlib.h>

int main(int argc, const char** const argv)
{
    int* data = malloc(100*sizeof(int));

    return EXIT_SUCCESS;
}
```

Running the following command produces the following output
```bash
$ clang-tidy test.c

test.c:12:13: warning: Array access (from variable 'data')
results in a null pointer dereference
    data[0] = 1;
            ^
test.c:5:5: note: 'data' initialized to a null pointer value
    int* data = NULL;
    ^
test.c:7:9: note: Assuming the condition is false
    if (rand() < 10)
        ^
test.c:7:5: note: Taking false branch
    if (rand() < 10)
    ^
test.c:12:13: note: Array access (from variable 'data') results in a
null pointer dereference
    data[0] = 1;
            ^
test.c:14:5: warning: Potential leak of memory pointed to by 'data'
    return EXIT_SUCCESS;
    ^
test.c:7:9: note: Assuming the condition is true
    if (rand() < 10)
        ^
test.c:7:5: note: Taking true branch
    if (rand() < 10)
    ^
test.c:9:16: note: Memory is allocated
        data = malloc(100*sizeof(int));
               ^
test.c:14:5: note: Potential leak of memory pointed to by 'data'
    return EXIT_SUCCESS;

```
